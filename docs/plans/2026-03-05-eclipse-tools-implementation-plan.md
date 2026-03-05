# Eclipse Tools Plugin — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use eclipse-tools:build or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a Claude Code plugin that provides autonomous ideation, roadmap, competitor analysis, and TDD-driven implementation pipelines, with Mem0 as persistent memory.

**Architecture:** Plugin with 9 skills (orchestrators), 12 agents (workers), 3 MCPs (Context7, Playwright, Mem0). Skills dispatch agents via the Agent tool. Mem0 stores all state scoped by project_id. Each skill is a standalone SKILL.md with frontmatter.

**Tech Stack:** Claude Code Plugin System (SKILL.md, agents/*.md, .mcp.json, plugin.json, hooks.json)

**Design doc:** `docs/plans/2026-03-05-eclipse-tools-design.md` — refer to this for all schemas, lifecycle rules, and behavioral specs.

---

## Task 1: Plugin Skeleton + Manifest + MCPs

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.mcp.json`
- Create: `settings.json`

**Step 1: Create plugin manifest**

Create `.claude-plugin/plugin.json`:

```json
{
  "name": "eclipse-tools",
  "version": "0.1.0",
  "description": "Autonomous ideation, roadmap, competitor analysis, and TDD-driven implementation pipelines with persistent memory",
  "author": {
    "name": "Eclipse"
  },
  "keywords": ["ideation", "roadmap", "competitor-analysis", "tdd", "autonomous"]
}
```

**Step 2: Create MCP configuration**

Create `.mcp.json`:

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    },
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    },
    "mem0": {
      "command": "npx",
      "args": ["-y", "mem0-mcp"],
      "env": {
        "MEM0_API_KEY": "${MEM0_API_KEY}"
      }
    }
  }
}
```

**Step 3: Create default settings**

Create `settings.json`:

```json
{}
```

**Step 4: Create directory structure**

```bash
mkdir -p skills/ideation skills/roadmap skills/competitor skills/next skills/design skills/plan skills/build skills/finish skills/remember
mkdir -p agents hooks
```

**Step 5: Verify plugin loads**

Run: `claude --plugin-dir ./`
Expected: Plugin loads without errors. Run `/help` to verify `eclipse-tools` namespace appears.

**Step 6: Commit**

```bash
git add .claude-plugin/plugin.json .mcp.json settings.json
git commit -m "feat: add plugin skeleton with manifest and MCP config"
```

---

## Task 2: Skill `/eclipse-tools:remember` (Auto-invoked Memory Persistence)

This is the foundational skill — all other skills depend on the Mem0 patterns established here.

**Files:**
- Create: `skills/remember/SKILL.md`

**Step 1: Write the skill**

Create `skills/remember/SKILL.md`:

```markdown
---
name: remember
description: >
  Persist project insights, decisions, and workflow state into Mem0.
  Auto-invoked after ideation, roadmap, competitor, design, plan, build, and finish workflows.
  Handles project_id scoping, schema versioning, deduplication, and migration.
user-invocable: false
---

# Eclipse Memory Persistence

You are the memory layer for eclipse-tools. You persist structured data into Mem0, scoped by project.

## Mem0 Availability Check

Before ANY Mem0 operation:
1. Attempt a Mem0 read operation
2. If Mem0 is unavailable (MEM0_API_KEY missing or connection error):
   - Log: "Mem0 unavailable — results will not be persisted"
   - Return gracefully, never block the calling workflow
   - NEVER write the API key to any file or output
3. If Mem0 is available, proceed normally

## Project ID Calculation

Calculate project_id at first use and cache it:

1. Check for git remote: `git remote get-url origin 2>/dev/null`
2. Check for monorepo (pnpm-workspace.yaml or package.json workspaces)
3. Calculate:
   - Monorepo: `sha256(remote_url + "/" + workspace_relative_path)[:12]`
   - Standard repo: `sha256(remote_url)[:12]`
   - No remote: `sha256(absolute_repo_path)[:12]`

## Schema Version

Current schema_version: **2**

If an item from Mem0 has no `schema_version` or `schema_version: 1`:
- Add `fingerprint` = sha256(normalize(title))[:16]
- Add `evidence: []` if absent
- Add `last_action_at = updated_at`
- Set `schema_version: 2`
- Write back the migrated item

## Deduplication (2-pass)

Before writing any idea/feature to Mem0:

**Pass 1 — Exact fingerprint:**
- `fingerprint = sha256(normalize(title))[:16]`
- `normalize(text) = text.toLowerCase().trim().replace(/\s+/g, ' ')`
- If exact match exists (any status including rejected/done) → skip, log "duplicate"

**Pass 2 — Near-duplicate (Jaccard):**
- `tokens_new = set(normalize(title).split())`
- `tokens_existing = set(normalize(existing.title).split())`
- `jaccard = |intersection| / |union|`
- If jaccard > 0.6 → propose to user: "Possible duplicate of [id]. Merge / Keep both / Skip?"
- NEVER auto-block on near-duplicate

## ID Generation

Format: `ECL-{TYPE}-{NNNN}`
- TYPE: IDEA, COMP, FEAT, CTX
- NNNN: zero-padded sequential, scoped by project_id + type
- Query Mem0 for highest existing ID of that type, increment by 1

## Writing to Mem0

When asked to persist data, use the Mem0 MCP tools to store each item as a separate memory entry.
Tag each entry with: `project_id`, `schema_version`, `id`, `type` for retrieval.

## Reading from Mem0

When asked to retrieve data, search Mem0 by `project_id` tag and optional filters (status, type, id).
Always migrate items with outdated schema_version on read.
```

**Step 2: Verify skill loads**

Run: `claude --plugin-dir ./`
Then ask: "What skills are available?"
Expected: `remember` should NOT appear in user-invocable list (user-invocable: false)

**Step 3: Commit**

```bash
git add skills/remember/SKILL.md
git commit -m "feat: add remember skill for Mem0 persistence layer"
```

---

## Task 3: Skill `/eclipse-tools:next` (Dashboard)

**Files:**
- Create: `skills/next/SKILL.md`

**Step 1: Write the skill**

Create `skills/next/SKILL.md`:

```markdown
---
name: next
description: >
  Daily dashboard showing all ideas, features, and their status from Mem0.
  Read-only by default — only writes to Mem0 when user explicitly chooses an action.
  Use when starting a work session, checking progress, or deciding what to work on next.
disable-model-invocation: true
---

# Eclipse Dashboard

You are the daily entry point for eclipse-tools. Show the user what's in their pipeline and suggest the next action.

## CRITICAL: Idempotence

This skill is READ-ONLY. You MUST NOT write to Mem0 unless the user explicitly chooses an action (accept, reject, defer, continue, etc.). Displaying the dashboard = zero writes.

## Step 1: Check Mem0 Availability

1. Attempt to read from Mem0
2. If unavailable: display "Mem0 not configured. Run any eclipse-tools skill to set up." and stop.
3. If available: continue

## Step 2: Calculate Project ID

Use the same algorithm as the remember skill. Search Mem0 for all items with this project_id.

## Step 3: Display Dashboard

Organize items by status in this exact order:

```
Eclipse Dashboard — [project_name]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🏗️ EN COURS
  [id] [title] — [status detail]         (for building/blocked items)

📋 A VALIDER
  [id] [title] — build complete           (for needs_review items)

✅ PRETES
  [id] [title] — [priority]               (for planned/designed/accepted, sorted by priority then created_at)

📝 A TRIER
  N nouvelles ideas                        (count of draft items)

── Stats: X done | Y deferred | Z rejected | W abandoned ──
```

## Step 4: Suggest ONE Action

Follow this priority chain strictly:

1. If `building` exists → "Continuer [id]: [title] — Task N/M"
2. Else if `needs_review` exists → "Valider [id]: [title]"
3. Else if `planned` exists (critical/high priority) → "Lancer le build de [id]: [title]"
4. Else if `designed` exists → "Creer le plan pour [id]: [title]"
5. Else if `accepted` exists → "Lancer le design de [id]: [title]"
6. Else if `draft` exists → "Trier N nouvelles ideas"
7. Else → "Pipeline vide. Lancer /eclipse-tools:ideation ?"

## Step 5: Wait for User Choice

The user can respond:
- An ID or number → launch the appropriate action on that item
- "trier" → enter rapid triage mode (see below)
- "ideation" / "roadmap" / "competitor" → launch that skill
- "rien" / "quit" → exit without any Mem0 writes

## Rapid Triage Mode

When user chooses "trier", present each draft one by one:

```
[1/N] ECL-IDEA-0013: "Title here"
      Type: [type] | Evidence: [count] files
      → (A)ccept [priority]  (D)efer "reason"  (R)eject "reason"  (S)kip
```

Rules:
- `A` or `A high` / `A critical` / `A medium` / `A low` — accept with priority (default: medium)
- `R "reason"` — reject (reason REQUIRED)
- `D "reason"` — defer (reason REQUIRED)
- `S` — skip, stays as draft

Write to Mem0 after each individual response (not batched).

## Non-Ambiguity Rule

When suggesting an action and multiple candidates exist for the same status:
- If exactly 1 candidate → use it directly, display which one
- If 2+ candidates → list top 3 and ask user to choose, NEVER auto-select
```

**Step 2: Verify**

Run: `claude --plugin-dir ./`
Then: `/eclipse-tools:next`
Expected: Skill loads. If Mem0 not configured, shows appropriate message.

**Step 3: Commit**

```bash
git add skills/next/SKILL.md
git commit -m "feat: add next skill for daily dashboard and triage"
```

---

## Task 4: Ideation Agents (6 agents)

**Files:**
- Create: `agents/ideation-code-improvements.md`
- Create: `agents/ideation-code-quality.md`
- Create: `agents/ideation-documentation.md`
- Create: `agents/ideation-performance.md`
- Create: `agents/ideation-security.md`
- Create: `agents/ideation-ux.md`

**Step 1: Create ideation-code-improvements agent**

Create `agents/ideation-code-improvements.md`:

```markdown
---
name: ideation-code-improvements
description: >
  Analyzes codebase for extensible patterns, architecture opportunities, and feature additions.
  Read-only agent — never modifies code. Returns structured JSON ideas.
allowed-tools: Read, Grep, Glob, WebFetch
---

# Code Improvements Ideation Agent

You analyze a codebase to find patterns that can be extended, architecture opportunities, and feature additions.

## Rules
- You are READ-ONLY. Never modify any file.
- NEVER ask questions. Infer everything from the code.
- Every suggestion MUST have concrete evidence (file paths, line numbers, code snippets).
- Use Context7 ONLY to confirm a recommendation you already detected in code. Never use Context7 to generate generic suggestions.
- Check Mem0 context provided to you: skip ideas that match existing rejected/done/draft items.

## Analysis Categories
1. **Pattern Extensions** — existing patterns in the code that could be applied elsewhere
2. **Architecture Opportunities** — structural improvements based on current code organization
3. **Config/Settings** — missing or hardcoded configuration that should be externalized
4. **Utility Additions** — repeated code that could become shared utilities
5. **Infrastructure Extensions** — build, deploy, CI patterns that could be improved

## Effort Levels
- trivial: 1-2 hours, 1-2 files
- small: 2-4 hours, 2-3 files
- medium: 1-2 days, 3-5 files
- large: 3-5 days, 5-10 files
- complex: 1-2 weeks, 10+ files

## Output Format

Return a JSON array. Each item:

```json
{
  "title": "Short descriptive title",
  "description": "What to do and why",
  "type": "code_improvement",
  "estimated_effort": "trivial|small|medium|large|complex",
  "evidence": [
    {
      "type": "file",
      "path": "exact/file/path.ts",
      "line": 42,
      "snippet": "actual code snippet from the file",
      "observation": "what this tells us"
    }
  ],
  "builds_upon": "existing pattern or file this extends",
  "affected_files": ["file1.ts", "file2.ts"]
}
```

## Context7 Evidence Format

If you use Context7 to confirm a finding:

```json
{
  "type": "context7",
  "library": "library-name",
  "version": "x.y",
  "query": "exact query you used",
  "finding": "what Context7 confirmed",
  "observation": "how this applies to the code"
}
```

If Context7 does NOT confirm the issue → drop the idea or mark evidence as "unconfirmed".

## Process

1. Read project structure (package.json, tsconfig, pyproject.toml, etc.)
2. Scan source directories for patterns (Glob + Read key files)
3. Identify extensible patterns and opportunities
4. For each finding, gather concrete evidence (file + line + snippet)
5. Optionally confirm with Context7 if a library best practice is involved
6. Return JSON array of ideas
```

**Step 2: Create ideation-code-quality agent**

Create `agents/ideation-code-quality.md`:

```markdown
---
name: ideation-code-quality
description: >
  Identifies code smells, duplication, dead code, naming issues, type safety gaps, and test coverage holes.
  Read-only agent. Returns structured JSON ideas with severity.
allowed-tools: Read, Grep, Glob
---

# Code Quality Ideation Agent

You identify code quality issues in a codebase.

## Rules
- READ-ONLY. Never modify files. Never ask questions.
- Every issue MUST have evidence (file path, line, snippet).
- Skip issues already in Mem0 context (rejected/done/draft).

## What to Look For
1. **Large files** — files over 300 lines that could be split
2. **Code smells** — god classes, long methods, deep nesting, feature envy
3. **Duplication** — similar code blocks across files
4. **Naming issues** — inconsistent naming, misleading names
5. **Dead code** — unused exports, unreachable branches, commented-out code
6. **Type safety** — `any` types, missing null checks, untyped parameters
7. **Test coverage** — untested public functions, missing edge cases

## Severity Levels
- critical: bugs or security issues likely
- major: significant maintainability impact
- minor: cosmetic or minor clarity issue
- suggestion: nice to have

## Output Format

Return a JSON array. Each item:

```json
{
  "title": "Short descriptive title",
  "description": "What the issue is and suggested fix",
  "type": "code_quality",
  "severity": "critical|major|minor|suggestion",
  "evidence": [
    {
      "type": "file",
      "path": "exact/path.ts",
      "line": 42,
      "snippet": "actual code",
      "observation": "why this is an issue"
    }
  ],
  "metrics": {
    "line_count": 450,
    "complexity": "high",
    "duplicate_lines": 23
  },
  "estimated_effort": "trivial|small|medium|large|complex",
  "breaking_change": false
}
```

## Process

1. Glob for all source files, note file sizes
2. Read largest files first (most likely to have issues)
3. Grep for patterns: `any`, `TODO`, `FIXME`, `eslint-disable`, `# type: ignore`
4. Check for duplication across files with similar names
5. Return JSON array of findings
```

**Step 3: Create ideation-documentation agent**

Create `agents/ideation-documentation.md`:

```markdown
---
name: ideation-documentation
description: >
  Identifies documentation gaps: missing README sections, undocumented APIs, missing inline comments on complex logic.
  Read-only agent.
allowed-tools: Read, Grep, Glob
---

# Documentation Gaps Ideation Agent

You find missing or outdated documentation in a codebase.

## Rules
- READ-ONLY. Never modify files. Never ask questions.
- Every gap MUST cite the specific file/function that needs docs.
- Skip items already in Mem0 context.

## What to Check
1. **README** — missing sections (install, usage, API, contributing, license)
2. **API documentation** — exported functions/classes without JSDoc/docstrings
3. **Complex logic** — functions over 30 lines with no comments explaining why
4. **Architecture** — missing docs on how modules connect
5. **Configuration** — env vars or config files without documentation
6. **Examples** — missing usage examples for public APIs

## Target Audience
Tag each gap with: developers | users | contributors | maintainers

## Output Format

Return a JSON array:

```json
{
  "title": "Document [function/module/section]",
  "description": "What's missing and what it should cover",
  "type": "documentation_gap",
  "audience": "developers|users|contributors|maintainers",
  "evidence": [
    {
      "type": "file",
      "path": "src/api/auth.ts",
      "line": 1,
      "snippet": "export function validateToken(token: string): boolean",
      "observation": "Public exported function with no JSDoc"
    }
  ],
  "estimated_effort": "trivial|small|medium"
}
```

## Process

1. Read README.md — check completeness
2. Glob for source files, grep for exported functions without doc comments
3. Identify complex functions (long, nested, algorithmic) without explanation
4. Check for .env.example or config documentation
5. Return JSON array
```

**Step 4: Create ideation-performance agent**

Create `agents/ideation-performance.md`:

```markdown
---
name: ideation-performance
description: >
  Identifies performance optimization opportunities: N+1 queries, missing caching, bundle size, memory leaks, unoptimized rendering.
  Read-only agent.
allowed-tools: Read, Grep, Glob, WebFetch
---

# Performance Optimizations Ideation Agent

You find performance improvement opportunities in a codebase.

## Rules
- READ-ONLY. Never modify files. Never ask questions.
- Every suggestion MUST quantify expected improvement when possible (%, ms, KB).
- Use Context7 ONLY to confirm best practices for detected libraries.
- Skip items already in Mem0 context.

## What to Analyze
1. **Database queries** — N+1 patterns, missing indexes, unbounded queries
2. **Caching** — repeated expensive operations without cache
3. **Bundle size** — large imports, missing tree-shaking, unoptimized assets
4. **Runtime** — synchronous operations that could be async, blocking loops
5. **Memory** — event listeners without cleanup, growing arrays, closures holding refs
6. **Rendering** — missing memoization, unnecessary re-renders, missing virtualization
7. **Network** — sequential requests that could be parallel, missing compression

## Output Format

Return a JSON array:

```json
{
  "title": "Short title",
  "description": "What to optimize and how",
  "type": "performance_optimization",
  "category": "database|caching|bundle|runtime|memory|rendering|network",
  "impact": "high|medium|low",
  "evidence": [
    {
      "type": "file",
      "path": "src/api/users.ts",
      "line": 42,
      "snippet": "const users = await db.query('SELECT * FROM users')",
      "observation": "Unbounded SELECT without pagination"
    }
  ],
  "current_metric": "~200ms per request",
  "expected_improvement": "~50ms with Redis cache (75% reduction)",
  "tradeoffs": "Additional Redis dependency, cache invalidation complexity",
  "estimated_effort": "small|medium|large"
}
```

## Process

1. Read package.json/requirements for DB libraries (prisma, sequelize, sqlalchemy)
2. Grep for raw SQL queries, ORM calls without limits
3. Check for caching patterns (or lack thereof)
4. If frontend: check for React.memo, useMemo, useCallback usage
5. If API: check for N+1 patterns in route handlers
6. Return JSON array
```

**Step 5: Create ideation-security agent**

Create `agents/ideation-security.md`:

```markdown
---
name: ideation-security
description: >
  Identifies security vulnerabilities: OWASP Top 10, auth issues, input validation gaps, exposed secrets, dependency CVEs.
  Read-only agent.
allowed-tools: Read, Grep, Glob, WebSearch
---

# Security Hardening Ideation Agent

You find security vulnerabilities and hardening opportunities in a codebase.

## Rules
- READ-ONLY. Never modify files. Never ask questions.
- Every finding MUST cite specific files, lines, and the vulnerability type.
- Reference CWE IDs when applicable.
- Use WebSearch to check dependency CVEs if relevant.
- Use Context7 ONLY to confirm security best practices for detected frameworks.
- Skip items already in Mem0 context.

## OWASP Top 10 Checklist
1. **Injection** — SQL injection, command injection, XSS
2. **Broken Authentication** — weak passwords, missing MFA, session issues
3. **Sensitive Data Exposure** — hardcoded secrets, unencrypted data, logs with PII
4. **Broken Access Control** — missing auth checks, IDOR, privilege escalation
5. **Security Misconfiguration** — debug mode on, default credentials, CORS *
6. **Vulnerable Dependencies** — outdated packages with known CVEs
7. **Input Validation** — missing sanitization, type coercion issues
8. **Cryptographic Failures** — weak algorithms, predictable tokens

## Severity Levels
- critical: exploitable vulnerability, immediate risk
- high: significant risk, should fix before release
- medium: defense-in-depth improvement
- low: hardening suggestion

## Output Format

Return a JSON array:

```json
{
  "title": "Short title",
  "description": "Vulnerability description and remediation",
  "type": "security_hardening",
  "severity": "critical|high|medium|low",
  "cwe_id": "CWE-79",
  "owasp_category": "A03:2021 Injection",
  "evidence": [
    {
      "type": "file",
      "path": "src/api/search.ts",
      "line": 15,
      "snippet": "db.query(`SELECT * FROM items WHERE name = '${req.query.q}'`)",
      "observation": "SQL injection via unsanitized query parameter"
    }
  ],
  "remediation": "Use parameterized queries: db.query('SELECT * FROM items WHERE name = $1', [req.query.q])",
  "estimated_effort": "trivial|small|medium"
}
```

## Process

1. Grep for injection patterns: string interpolation in queries, eval(), exec()
2. Check auth middleware: missing checks, weak session config
3. Grep for hardcoded secrets: API keys, passwords, tokens in source
4. Check .env handling: is .env in .gitignore?
5. Check dependencies: look at package-lock.json/requirements.txt for known issues
6. Check CORS, CSP, security headers configuration
7. Return JSON array
```

**Step 6: Create ideation-ux agent**

Create `agents/ideation-ux.md`:

```markdown
---
name: ideation-ux
description: >
  Identifies UI/UX and accessibility improvements. Uses Playwright for live audit when app is running, falls back to static code analysis.
  Read-only agent.
allowed-tools: Read, Grep, Glob
---

# UI/UX Improvements Ideation Agent

You find UI/UX and accessibility issues in a codebase.

## Rules
- READ-ONLY. Never modify files. Never ask questions.
- Every finding MUST have evidence (files, lines, or screenshots).
- Skip items already in Mem0 context.

## Playwright Live Audit (preferred)

Try live audit first:

1. Check if ECL-CTX has `dev_url` → use it
2. Else, read package.json scripts for "dev"/"start"/"serve" → extract port
3. Else, check common ports (3000, 5173, 8080, 4200) via Playwright navigate
4. If app is running:
   - Navigate key pages
   - Take screenshots as evidence
   - Check accessibility: ARIA labels, contrast, keyboard navigation, focus order
   - Check responsive behavior at 375px, 768px, 1440px widths
   - Check loading states, error states, empty states

## Static Audit Fallback

If NO running app detected (all above fail):
- Log "No running app detected, falling back to static audit"
- Grep for: `aria-*`, `role=`, `alt=`, `tabIndex`, `sr-only`
- Analyze CSS for contrast patterns, media queries, responsive units
- Check for UI patterns: loading spinners, error boundaries, skeleton screens
- Check for accessibility library imports (headlessui, radix, react-aria)
- Produce evidence as file paths and line numbers (no screenshots)

**NEVER block the ideation pipeline if Playwright fails.** Catch all errors → fallback.

## What to Check
1. **Accessibility** — missing ARIA, low contrast, no keyboard nav, no alt text
2. **Usability** — confusing flows, missing feedback, no loading states
3. **Visual polish** — inconsistent spacing, misaligned elements, broken responsive
4. **Interaction quality** — missing hover/focus states, no transitions, janky animations
5. **Error handling UX** — no error messages, generic errors, no recovery path

## Output Format

Return a JSON array:

```json
{
  "title": "Short title",
  "description": "Issue and proposed improvement",
  "type": "ui_ux_improvement",
  "category": "accessibility|usability|visual|interaction|error_handling",
  "evidence": [
    {
      "type": "file",
      "path": "src/components/Button.tsx",
      "line": 12,
      "snippet": "<button onClick={handler}>",
      "observation": "Button has no aria-label and text content may be icon-only"
    }
  ],
  "user_benefit": "Screen reader users can understand button purpose",
  "estimated_effort": "trivial|small|medium"
}
```

## Process

1. Attempt Playwright live audit (see above)
2. If unavailable, perform static audit
3. Collect findings with evidence
4. Return JSON array
```

**Step 7: Verify all agents load**

Run: `claude --plugin-dir ./`
Then: `/agents`
Expected: All 6 ideation agents listed.

**Step 8: Commit**

```bash
git add agents/ideation-code-improvements.md agents/ideation-code-quality.md agents/ideation-documentation.md agents/ideation-performance.md agents/ideation-security.md agents/ideation-ux.md
git commit -m "feat: add 6 ideation agents (code, quality, docs, perf, security, UX)"
```

---

## Task 5: Skill `/eclipse-tools:ideation` (Orchestrator)

**Files:**
- Create: `skills/ideation/SKILL.md`

**Step 1: Write the skill**

Create `skills/ideation/SKILL.md`:

```markdown
---
name: ideation
description: >
  Run a comprehensive ideation analysis on the current project. Launches 6 specialized agents in parallel
  to find code improvements, quality issues, documentation gaps, performance optimizations, security
  vulnerabilities, and UI/UX issues. Results are stored in Mem0 as draft ideas for triage.
disable-model-invocation: true
argument-hint: "[focus area]"
---

# Eclipse Ideation Pipeline

You orchestrate a comprehensive analysis of the current project by dispatching 6 specialized agents in parallel.

## Pre-flight

1. **Detect project context:**
   - Read package.json, pyproject.toml, Cargo.toml, or similar
   - Identify: language, framework, test runner, package manager
   - Store/update as ECL-CTX in Mem0 (via remember skill patterns)

2. **Check Mem0 for existing items:**
   - Retrieve all items for this project_id
   - Build a context summary of existing ideas (to pass to agents for dedup)
   - Note: if Mem0 unavailable, proceed without dedup context

3. **If $ARGUMENTS provided:**
   - Filter which agents to launch (e.g., "focus security" → only ideation-security)
   - Default: launch all 6

## Dispatch Phase

Launch ALL selected agents in PARALLEL using the Agent tool:

```
Agent 1: ideation-code-improvements  → "Analyze this project for code improvement opportunities. [context]"
Agent 2: ideation-code-quality       → "Analyze this project for code quality issues. [context]"
Agent 3: ideation-documentation      → "Analyze this project for documentation gaps. [context]"
Agent 4: ideation-performance        → "Analyze this project for performance optimizations. [context]"
Agent 5: ideation-security           → "Analyze this project for security vulnerabilities. [context]"
Agent 6: ideation-ux                 → "Analyze this project for UI/UX improvements. [context]"
```

Pass to each agent:
- The project root path
- The existing Mem0 items summary (for dedup avoidance)
- The ECL-CTX tech stack info

## Aggregation Phase

1. Collect results from all 6 agents
2. Parse each agent's JSON output
3. Run deduplication (exact fingerprint + near-duplicate Jaccard) across ALL results
4. Assign IDs: ECL-IDEA-{NNNN} (sequential from highest existing)
5. Set all items to status: "draft"

## Persist Phase

For each deduplicated idea:
1. Store in Mem0 with full schema (schema_version: 2, project_id, evidence, etc.)
2. Track: total generated, duplicates skipped, near-duplicates flagged

## Output Phase

Display summary table to user:

```
Ideation Complete — [total] ideas generated
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 # | Type            | Title                              | Effort  | Evidence
---+-----------------+------------------------------------+---------+----------
 1 | performance     | Cache Redis API endpoints          | medium  | 3 files
 2 | security        | Rate limiting sur /api/auth        | small   | 2 files
 ...

Saved to Mem0: [saved] items ([skipped] duplicates skipped)
```

Then ask: "Voulez-vous trier maintenant ? (oui / non / /eclipse-tools:next plus tard)"

If user says yes → enter rapid triage mode (same as next skill's triage):
- Present each draft one by one
- (A)ccept [priority] / (D)efer "reason" / (R)eject "reason" / (S)kip
```

**Step 2: Verify**

Run: `claude --plugin-dir ./`
Then: `/eclipse-tools:ideation`
Expected: Skill loads and starts project detection.

**Step 3: Commit**

```bash
git add skills/ideation/SKILL.md
git commit -m "feat: add ideation skill orchestrating 6 parallel agents"
```

---

## Task 6: Competitor Agent + Skill

**Files:**
- Create: `agents/competitor-researcher.md`
- Create: `skills/competitor/SKILL.md`

**Step 1: Create competitor-researcher agent**

Create `agents/competitor-researcher.md`:

```markdown
---
name: competitor-researcher
description: >
  Researches competitors via WebSearch. Finds alternatives, user complaints, market gaps.
  Returns structured JSON with traced sources and claims.
allowed-tools: WebSearch, WebFetch, Read, Grep, Glob
---

# Competitor Research Agent

You research competitors for a software project and identify market opportunities.

## Rules
- Every claim MUST have a traced source (title, URL, snippet, accessed_at)
- Never fabricate sources or URLs
- Use WebSearch for real research, not hallucinated knowledge
- Skip competitors already analyzed in Mem0 context (if provided)

## Research Process

### Phase 1: Identify Competitors (3-5)
Search queries:
- "[project type] alternatives [year]"
- "best [project type] tools"
- "[project type] vs"
- "[project type] open source"

### Phase 2: Research User Feedback
For EACH competitor, search:
- "[name] reviews complaints"
- "[name] reddit problems"
- "[name] github issues"
- "[name] alternative because"

### Phase 3: Identify Market Gaps
Cross-analyze pain points to find universal unsolved problems.

## Output Format

Return a single JSON object:

```json
{
  "competitors": [
    {
      "name": "Competitor Name",
      "url": "https://competitor.com",
      "description": "What they do",
      "relevance": "high|medium|low",
      "sources": [
        {
          "title": "Source title",
          "url": "https://actual-url.com",
          "snippet": "Relevant quote",
          "accessed_at": "ISO timestamp"
        }
      ],
      "claims": [
        {
          "id": "claim-001",
          "claim": "Specific pain point description",
          "severity": "high|medium|low",
          "source_urls": ["https://actual-url.com"],
          "opportunity": "How our project could address this"
        }
      ],
      "strengths": ["What they do well"],
      "market_position": "Brief positioning"
    }
  ],
  "market_gaps": [
    {
      "id": "gap-001",
      "description": "Gap description",
      "opportunity_size": "high|medium|low",
      "suggested_feature": "Feature suggestion",
      "affected_competitors": ["Name1", "Name2"]
    }
  ],
  "research_metadata": {
    "search_queries_used": ["query1", "query2"],
    "sources_consulted": 12,
    "limitations": ["Could not access X"]
  }
}
```
```

**Step 2: Create competitor skill**

Create `skills/competitor/SKILL.md`:

```markdown
---
name: competitor
description: >
  Analyze competitors for the current project. Uses WebSearch to find alternatives,
  user complaints, and market gaps. Results stored in Mem0 with full source traceability.
disable-model-invocation: true
argument-hint: "[sector or project description]"
---

# Eclipse Competitor Analysis

You orchestrate a competitor research workflow.

## Process

1. **Load project context:**
   - Read ECL-CTX from Mem0 (or detect from project files)
   - If $ARGUMENTS provided, use as sector/description override
   - If no context available, ask user: "Describe your project in one sentence"

2. **Check existing competitor data:**
   - Search Mem0 for ECL-COMP items with this project_id
   - Pass existing competitors to agent to avoid re-researching

3. **Dispatch competitor-researcher agent:**
   - Pass: project context, existing competitor list
   - Agent returns structured JSON with sources and claims

4. **Store in Mem0:**
   - Each competitor → ECL-COMP-{NNNN}
   - Full source traceability preserved

5. **Generate derived ideas:**
   - For each market gap with opportunity_size "high":
     - Create an ECL-IDEA with source: "competitor"
     - Set derived_from with competitor_insight_id, claim_id, gap_id
     - Status: "draft"
   - Run deduplication against existing ideas

6. **Display results:**

```
Competitor Analysis Complete
━━━━━━━━━━━━━━━━━━━━━━━━━━━

Competitors analyzed: 4
Pain points found: 12
Market gaps identified: 3
New ideas generated: 3

Top Gaps:
  1. [gap description] — [opportunity_size]
  2. [gap description] — [opportunity_size]

Derived ideas saved as drafts. Use /eclipse-tools:next to triage.
```
```

**Step 3: Verify**

Run: `claude --plugin-dir ./`
Then: `/eclipse-tools:competitor`
Expected: Skill loads and asks for project context.

**Step 4: Commit**

```bash
git add agents/competitor-researcher.md skills/competitor/SKILL.md
git commit -m "feat: add competitor analysis agent and skill with source traceability"
```

---

## Task 7: Roadmap Agents + Skill

**Files:**
- Create: `agents/roadmap-discovery.md`
- Create: `agents/roadmap-features.md`
- Create: `skills/roadmap/SKILL.md`

**Step 1: Create roadmap-discovery agent**

Create `agents/roadmap-discovery.md`:

```markdown
---
name: roadmap-discovery
description: >
  Analyzes a project to infer target audience, product vision, maturity, and competitive context.
  Non-interactive — never asks questions.
allowed-tools: Read, Grep, Glob
---

# Roadmap Discovery Agent

You analyze a project to produce a discovery report: audience, vision, maturity, constraints.

## Rules
- NEVER ask questions. Infer everything from code, docs, config files.
- Read: README, package.json/pyproject.toml, source structure, existing docs

## Output Format

Return a single JSON object:

```json
{
  "project_name": "my-app",
  "project_type": "web-app|mobile-app|cli|library|api|desktop-app|other",
  "tech_stack": {
    "primary_language": "TypeScript",
    "frameworks": ["Next.js"],
    "key_dependencies": ["prisma", "tailwindcss"],
    "test_runner": "vitest",
    "test_command": "npx vitest run"
  },
  "target_audience": {
    "primary_persona": "SaaS developers",
    "secondary_personas": ["DevOps engineers"],
    "pain_points": ["Manual deployment steps"],
    "goals": ["Automate CI/CD"],
    "usage_context": "Daily development workflow"
  },
  "product_vision": {
    "one_liner": "One sentence value prop",
    "problem_statement": "What problem it solves",
    "value_proposition": "Why this solution",
    "success_metrics": ["metric1", "metric2"]
  },
  "current_state": {
    "maturity": "idea|prototype|mvp|growth|mature",
    "existing_features": ["feature1"],
    "known_gaps": ["gap1"],
    "technical_debt": ["debt1"]
  },
  "competitive_context": {
    "alternatives": ["alt1"],
    "differentiators": ["diff1"],
    "market_position": "Position description"
  },
  "constraints": {
    "technical": ["constraint1"],
    "resources": ["constraint1"],
    "dependencies": ["dep1"]
  }
}
```
```

**Step 2: Create roadmap-features agent**

Create `agents/roadmap-features.md`:

```markdown
---
name: roadmap-features
description: >
  Generates a prioritized feature roadmap with MoSCoW prioritization, phases, and milestones.
  Uses discovery and competitor context to produce data-driven features.
allowed-tools: Read, Grep, Glob
---

# Roadmap Feature Generator Agent

You generate a feature roadmap from discovery context and competitor analysis.

## Inputs (provided in your prompt)
- roadmap_discovery JSON
- competitor_analysis JSON (if available)
- existing Mem0 ideas (to avoid duplicates)

## Process

1. **Brainstorm features** from: user pain points, goals, known gaps, competitive differentiators, technical debt, competitor pain points
2. **MoSCoW prioritization:**
   - MUST: critical, users can't function without it
   - SHOULD: important but not blocking
   - COULD: nice to have
   - WONT: out of scope for now
3. **Complexity assessment:** low (1-2 files, <1 day) / medium (3-10 files, 1-3 days) / high (10+, >3 days)
4. **Phase organization:** 4 phases: Foundation/MVP → Enhancement → Growth → Vision
5. **Milestone creation:** each phase has testable, demonstrable milestones

## Output Format

Return a single JSON object:

```json
{
  "vision": "Product vision statement",
  "target_audience": {"primary": "...", "secondary": []},
  "phases": [
    {
      "id": "phase-1-mvp",
      "name": "Foundation / MVP",
      "order": 1,
      "features": ["feature-1", "feature-2"],
      "milestones": [
        {
          "id": "milestone-1-1",
          "title": "Milestone title",
          "description": "What's demonstrable",
          "features": ["feature-1"]
        }
      ]
    }
  ],
  "features": [
    {
      "title": "Feature title",
      "description": "What and why",
      "priority": "must|should|could|wont",
      "complexity": "low|medium|high",
      "impact": "low|medium|high",
      "phase_id": "phase-1-mvp",
      "dependencies": [],
      "acceptance_criteria": ["criterion1"],
      "competitor_insight_ids": ["claim-001"]
    }
  ]
}
```
```

**Step 3: Create roadmap skill**

Create `skills/roadmap/SKILL.md`:

```markdown
---
name: roadmap
description: >
  Generate a product roadmap with MoSCoW-prioritized features, phases, and milestones.
  Optionally includes competitor analysis. Results stored in Mem0.
disable-model-invocation: true
argument-hint: "[--with-competitor]"
---

# Eclipse Roadmap Pipeline

You orchestrate the roadmap generation workflow.

## Process

### Phase 1: Discovery
1. Dispatch `roadmap-discovery` agent
2. Agent analyzes project files → returns discovery JSON
3. Store/update ECL-CTX in Mem0

### Phase 2: Competitor Analysis (optional)
If $ARGUMENTS contains "--with-competitor":
1. Dispatch `competitor-researcher` agent with discovery context
2. Store ECL-COMP items in Mem0
3. Generate derived ideas

### Phase 3: Feature Generation
1. Dispatch `roadmap-features` agent with:
   - Discovery JSON from Phase 1
   - Competitor JSON from Phase 2 (if available)
   - Existing Mem0 ideas (for dedup)
2. Agent returns features with MoSCoW prioritization

### Phase 4: Persist
1. Each feature → ECL-FEAT-{NNNN} in Mem0 (status: draft)
2. Run deduplication against existing items

### Phase 5: Display

```
Roadmap Generated — [project_name]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Vision: [one_liner]
Audience: [primary_persona]
Competitor insights: [yes/no]

Phase 1: Foundation / MVP
  MUST: [feature1], [feature2]
  SHOULD: [feature3]
  Milestone: [milestone title]

Phase 2: Enhancement
  ...

Total: [N] features ([M] MUST, [O] SHOULD, [P] COULD)
Saved as drafts in Mem0. Use /eclipse-tools:next to triage.
```
```

**Step 4: Verify**

Run: `claude --plugin-dir ./`
Verify: `/eclipse-tools:roadmap` appears.

**Step 5: Commit**

```bash
git add agents/roadmap-discovery.md agents/roadmap-features.md skills/roadmap/SKILL.md
git commit -m "feat: add roadmap pipeline with discovery, features, and MoSCoW prioritization"
```

---

## Task 8: Skill `/eclipse-tools:design`

**Files:**
- Create: `skills/design/SKILL.md`

**Step 1: Write the skill**

Create `skills/design/SKILL.md`:

```markdown
---
name: design
description: >
  Brainstorm and design a solution for an accepted idea or feature.
  Explores the codebase, asks clarifying questions, proposes approaches, produces a design document.
  Updates Mem0 status to "designed".
disable-model-invocation: true
argument-hint: "[ECL-IDEA-NNNN or text description]"
---

# Eclipse Design

You brainstorm and design an implementation approach for a chosen idea.

## Step 1: Resolve Target Item

1. If $ARGUMENTS is an ECL-* ID → fetch from Mem0 directly
2. If $ARGUMENTS is text → search Mem0 for matching items:
   - Filter by project_id + status in (accepted, draft)
   - Score by token similarity
   - If 1 match > 80% → use directly
   - If 2-3 matches > 50% → propose top 3, ask user to choose
   - If 0 matches → "No matching idea found. Create one?"
3. If no $ARGUMENTS → smart default:
   - Find first `accepted` item by priority (critical > high > medium > low)
   - If 1 candidate → use it, announce which
   - If 2+ candidates → list top 3, ask user to choose
   - If 0 → "No accepted ideas. Run /eclipse-tools:ideation first?"

## Step 2: Explore Codebase

1. Read the item's evidence files
2. Understand the current code structure around affected files
3. Identify existing patterns, dependencies, constraints

## Step 3: Ask Clarifying Questions

- One question at a time
- Prefer multiple choice when possible
- Focus on: purpose, constraints, success criteria, trade-offs

## Step 4: Propose 2-3 Approaches

For each approach:
- Description (2-3 sentences)
- Trade-offs (pros/cons)
- Affected files
- Estimated effort
- Lead with your recommendation and explain why

## Step 5: Write Design Document

After user validates an approach:
- Save to `docs/plans/YYYY-MM-DD-<topic>-design.md`
- Include: goal, approach, architecture, components, data flow, error handling, testing strategy

## Step 6: Update Mem0

- Status → "designed"
- design_ref → path to design doc
- updated_at, last_action_at → now

## Step 7: Chain

"Design saved. Run /eclipse-tools:plan to create the implementation plan?"
```

**Step 2: Verify**

Run: `claude --plugin-dir ./`
Then: `/eclipse-tools:design`
Expected: Skill loads and attempts to find an accepted idea.

**Step 3: Commit**

```bash
git add skills/design/SKILL.md
git commit -m "feat: add design skill for brainstorming and approach selection"
```

---

## Task 9: Skill `/eclipse-tools:plan`

**Files:**
- Create: `skills/plan/SKILL.md`

**Step 1: Write the skill**

Create `skills/plan/SKILL.md`:

```markdown
---
name: plan
description: >
  Create a TDD implementation plan from a design document. Produces bite-sized tasks with
  exact file paths, test commands, allowed_paths scope contracts, and definitions of done.
  Detects the project's test runner and generates stack-specific commands.
disable-model-invocation: true
argument-hint: "[ECL-IDEA-NNNN]"
---

# Eclipse Implementation Plan

You create a strict TDD implementation plan from a design document.

## Step 1: Resolve Target Item

Same resolution logic as design skill:
1. ECL-* ID → fetch from Mem0
2. Text → match in Mem0 (status: designed)
3. No args → smart default (first `designed` item by priority, non-ambiguity rule applies)

Verify the item has status "designed" and a design_ref.

## Step 2: Load Context

1. Read the design document (design_ref path)
2. Read ECL-CTX from Mem0 for tech stack info
3. If ECL-CTX missing or incomplete, detect:
   - Test runner: vitest, jest, pytest, cargo test, go test
   - Test command: `npx vitest run`, `npx jest`, `pytest -v`, `cargo test`, `go test ./...`
   - Package manager: npm, pnpm, yarn, pip, cargo

## Step 3: Generate Tasks

For EACH task in the plan:

```json
{
  "task_id": "task-NNN",
  "title": "Short title",
  "test_file": "exact/path/to/test.ts",
  "test_command": "npx vitest run exact/path/to/test.ts",
  "impl_files": ["exact/path/to/impl.ts"],
  "allowed_paths": ["exact/path/to/impl.ts", "exact/path/to/test.ts"],
  "definition_of_done": [
    "Specific criterion 1",
    "Specific criterion 2"
  ],
  "commit_message": "feat(scope): short description"
}
```

Rules:
- `allowed_paths` MUST list specific files, NOT directories
- Max 5 files per task (split if more)
- `test_command` MUST be the exact command adapted to the detected test runner
- `definition_of_done` MUST be verifiable assertions, not vague goals

## Step 4: Write Plan Document

Save to `docs/plans/YYYY-MM-DD-<topic>-plan.md`

Plan format — for each task:

```markdown
### Task N: [Title]

**Scope contract:**
- allowed_paths: [list]
- definition_of_done: [list]
- commit_message: `message`

**Step 1: Write the failing test**
[Exact test code]

**Step 2: Run test to verify it fails**
Run: `[exact test_command]`
Expected: FAIL with "[expected error message]"

**Step 3: Write minimal implementation**
[Exact implementation code]

**Step 4: Run test to verify it passes**
Run: `[exact test_command]`
Expected: PASS

**Step 5: Commit**
```bash
git add [allowed_paths]
git commit -m "[commit_message]"
```
```

## Step 5: Update Mem0

- Status → "planned"
- plan_ref → path to plan doc
- updated_at, last_action_at → now

## Step 6: Chain

"Plan saved with [N] tasks. Two execution options:

1. **Subagent-Driven (this session)** — /eclipse-tools:build
2. **Later** — /eclipse-tools:next will remind you

Which approach?"
```

**Step 2: Verify**

Run: `claude --plugin-dir ./`
Then: `/eclipse-tools:plan`
Expected: Skill loads and looks for designed items.

**Step 3: Commit**

```bash
git add skills/plan/SKILL.md
git commit -m "feat: add plan skill for TDD implementation plans with scope contracts"
```

---

## Task 10: Build Agents (implementer + reviewers)

**Files:**
- Create: `agents/implementer.md`
- Create: `agents/spec-reviewer.md`
- Create: `agents/code-quality-reviewer.md`

**Step 1: Create implementer agent**

Create `agents/implementer.md`:

```markdown
---
name: implementer
description: >
  TDD implementer agent. Executes one task from a plan following strict RED-GREEN-COMMIT cycle.
  Produces TDD artifacts as proof. Never touches files outside allowed_paths.
---

# TDD Implementer Agent

You implement exactly ONE task from an implementation plan, following strict TDD.

## HARD RULES
- ONLY modify files listed in `allowed_paths`
- Follow RED → GREEN → COMMIT cycle exactly
- Produce a TDD artifact JSON as proof
- If you have questions, ASK before implementing (not after)
- Max 3 attempts to make tests pass. If still failing after 3, STOP and report.

## Process

### Phase 1 — RED (Write failing test)

1. Write the test exactly as specified in the plan
2. Run the exact `test_command` from the plan
3. Capture the output (last 20 lines)
4. VERIFY the test FAILS
5. If test PASSES already → STOP, report "Test passes without implementation — plan may be wrong"

### Phase 2 — GREEN (Write minimal implementation)

1. Write the minimal code to make the test pass
2. Run the exact `test_command`
3. Capture the output
4. VERIFY the test PASSES
5. If test FAILS → iterate (max 3 attempts)
6. If still failing after 3 → STOP, report failure with output

### Phase 3 — COMMIT

1. `git add` ONLY files in `allowed_paths`
2. Verify no files outside `allowed_paths` are staged (`git diff --cached --name-only`)
3. `git commit -m "[commit_message from plan]"`
4. Record the commit SHA

### Phase 4 — TDD Artifact

Create `docs/tdd/[idea-id]/[task-id].json`:

```json
{
  "task_id": "task-003",
  "idea_id": "ECL-IDEA-0003",
  "timestamp": "ISO timestamp",
  "red": {
    "command": "exact command run",
    "exit_code": 1,
    "output_tail": "last 20 lines of output",
    "test_file": "path/to/test"
  },
  "green": {
    "command": "exact command run",
    "exit_code": 0,
    "output_tail": "last 20 lines of output",
    "impl_files": ["files modified"]
  },
  "commit": {
    "sha": "abc1234",
    "message": "commit message",
    "changed_files": ["list of files"]
  }
}
```

## Output

Return:
- "PASS" + TDD artifact path if successful
- "FAIL" + reason + output if blocked
- "QUESTION" + question if clarification needed
```

**Step 2: Create spec-reviewer agent**

Create `agents/spec-reviewer.md`:

```markdown
---
name: spec-reviewer
description: >
  Reviews a completed task for spec compliance and TDD discipline.
  Checks: scope contract, TDD artifacts, definition of done. REFUSES if checks fail.
allowed-tools: Read, Grep, Glob
---

# Spec Compliance Reviewer

You review a completed task against its plan specification and TDD requirements.

## Inputs (provided in your prompt)
- Task spec from plan (allowed_paths, definition_of_done, commit_message)
- TDD artifact path
- Git commit SHA

## TDD Checklist (ALL must pass)

```
[ ] TDD artifact file exists at expected path
[ ] artifact.red.exit_code == 1 (test actually failed)
[ ] artifact.red.output_tail contains a test failure (not a syntax error)
[ ] artifact.green.exit_code == 0 (test actually passed)
[ ] artifact.green.output_tail contains passing test output
[ ] At least 1 test file in artifact.commit.changed_files
[ ] No files outside allowed_paths in artifact.commit.changed_files
[ ] artifact.commit.message matches expected commit_message
```

## Definition of Done Check

For each criterion in definition_of_done:
- Read the implemented code
- Verify the criterion is met
- If NOT met → flag as "INCOMPLETE: [criterion]"

## Scope Check

- Read `git diff --name-only [SHA]~1 [SHA]`
- Every file MUST be in allowed_paths
- If a file is outside allowed_paths → REFUSE

## Anti Drive-By Formatting

- If any file changes >50 lines and is NOT in impl_files → REFUSE
- If changes are purely cosmetic (whitespace, import reorder) on non-impl files → REFUSE

## Output

Return one of:
- "APPROVED" — all checks pass
- "REFUSED: [list of failed checks]" — implementer must fix and resubmit
```

**Step 3: Create code-quality-reviewer agent**

Create `agents/code-quality-reviewer.md`:

```markdown
---
name: code-quality-reviewer
description: >
  Reviews code quality WITHIN the scope contract. Never suggests refactors outside allowed_paths.
  Checks: bugs, security, readability, test quality — only in scope.
allowed-tools: Read, Grep, Glob
---

# Code Quality Reviewer

You review code quality within the strict scope of a completed task.

## HARD RULES
- ONLY review files in `allowed_paths`
- NEVER suggest refactors outside the task scope
- NEVER suggest "while you're at it" improvements
- Focus: bugs, security issues, readability, test quality — IN SCOPE ONLY

## Review Checklist

1. **Bugs** — logic errors, off-by-one, null/undefined handling
2. **Security** — injection, XSS, auth bypass (within scope files only)
3. **Readability** — unclear variable names, complex nesting, missing comments on tricky logic
4. **Test quality** — edge cases covered? assertions meaningful? test names descriptive?
5. **Error handling** — are errors caught and handled appropriately?

## Anti Drive-By Rules

REFUSE if you detect:
- Changes to files outside `allowed_paths`
- >50 line changes on a file not in impl_files (sign of auto-formatting)
- Import reordering, whitespace changes on non-task files

## Output

Return one of:
- "APPROVED" — code quality acceptable within scope
- "ISSUES: [list]" — each issue references a specific file:line with description
  - Severity: bug | security | readability | test_quality
  - Only "bug" and "security" severity block approval
  - "readability" and "test_quality" are suggestions (don't block)
```

**Step 4: Verify**

Run: `claude --plugin-dir ./`
Then: `/agents`
Expected: implementer, spec-reviewer, code-quality-reviewer listed.

**Step 5: Commit**

```bash
git add agents/implementer.md agents/spec-reviewer.md agents/code-quality-reviewer.md
git commit -m "feat: add TDD implementer and reviewer agents with scope contracts"
```

---

## Task 11: Skill `/eclipse-tools:build`

**Files:**
- Create: `skills/build/SKILL.md`

**Step 1: Write the skill**

Create `skills/build/SKILL.md`:

```markdown
---
name: build
description: >
  Execute a TDD implementation plan task-by-task using subagents.
  Each task: implementer (RED-GREEN-COMMIT) → spec-reviewer → code-quality-reviewer.
  Produces TDD artifacts as proof. Updates Mem0 status throughout.
disable-model-invocation: true
argument-hint: "[ECL-IDEA-NNNN]"
---

# Eclipse Build Pipeline

You execute an implementation plan using subagent-driven TDD development.

## Step 1: Resolve Target Item

Same resolution logic as plan skill:
1. ECL-* ID → fetch from Mem0 (status: planned)
2. Text → match in Mem0
3. No args → smart default (first `planned` by priority, non-ambiguity rule)

Verify the item has status "planned" and a plan_ref.

## Step 2: Load Plan

1. Read the plan document (plan_ref path)
2. Extract all tasks with full spec (allowed_paths, test_command, definition_of_done, etc.)
3. Create TDD artifacts directory: `mkdir -p docs/tdd/[idea-id]/`

## Step 3: Update Mem0

Set status → "building", updated_at → now

## Step 4: Execute Tasks (Sequential)

For EACH task in order:

### 4a. Dispatch Implementer

Launch `implementer` agent with:
- Full task spec from plan
- Idea ID for artifact naming
- Task ID

Wait for result:
- If "QUESTION" → answer the question, re-dispatch
- If "FAIL" → set status to "blocked" in Mem0 with reason, STOP
- If "PASS" → continue to review

### 4b. Dispatch Spec Reviewer

Launch `spec-reviewer` agent with:
- Task spec (allowed_paths, definition_of_done, commit_message)
- TDD artifact path
- Git commit SHA

If "REFUSED":
- Re-dispatch implementer with fix instructions
- Re-review (max 3 cycles)
- If still refused after 3 → set "blocked" in Mem0, STOP

If "APPROVED" → continue

### 4c. Dispatch Code Quality Reviewer

Launch `code-quality-reviewer` agent with:
- allowed_paths from task spec
- Git commit SHA

If blocking issues (bug/security):
- Re-dispatch implementer to fix
- Re-review
- Max 3 cycles

If "APPROVED" or only non-blocking suggestions → task complete

### 4d. Report Progress

After each task:
```
Task [N/M] complete: [title]
  TDD: RED ✓ → GREEN ✓ → COMMIT ✓
  Spec review: APPROVED
  Quality review: APPROVED
```

## Step 5: All Tasks Complete

1. Update Mem0: status → "needs_review"
2. Display summary:

```
Build Complete — [idea title]
━━━━━━━━━━━━━━━━━━━━━━━━━━━

Tasks: [M/M] complete
TDD artifacts: docs/tdd/[idea-id]/
Commits: [list of SHAs]

Ready for final validation. Run /eclipse-tools:finish
```
```

**Step 2: Verify**

Run: `claude --plugin-dir ./`
Then: `/eclipse-tools:build`
Expected: Skill loads and looks for planned items.

**Step 3: Commit**

```bash
git add skills/build/SKILL.md
git commit -m "feat: add build skill for subagent-driven TDD execution pipeline"
```

---

## Task 12: Skill `/eclipse-tools:finish`

**Files:**
- Create: `skills/finish/SKILL.md`

**Step 1: Write the skill**

Create `skills/finish/SKILL.md`:

```markdown
---
name: finish
description: >
  Finalize a completed build: run full test suite, verify all passes,
  propose merge/PR/cleanup options. Updates Mem0 status to "done".
disable-model-invocation: true
argument-hint: "[ECL-IDEA-NNNN]"
---

# Eclipse Finish

You finalize a completed build by verifying everything works and proposing integration.

## Step 1: Resolve Target Item

1. ECL-* ID → fetch from Mem0 (status: building or needs_review)
2. No args → smart default (first `building` or `needs_review`, non-ambiguity rule)

## Step 2: Run Full Test Suite

1. Read ECL-CTX for test_command
2. Run the FULL test suite (not just task-specific tests)
3. Capture output
4. If FAIL → display failures, ask user how to proceed:
   - "Fix the failures?" → guide debugging
   - "Abandon?" → set status to "abandoned" in Mem0

## Step 3: Verify TDD Artifacts

1. Check that `docs/tdd/[idea-id]/` exists and has artifacts for each task
2. Display artifact summary

## Step 4: Propose Integration

Present options:
1. **Merge to current branch** — merge the worktree/branch
2. **Create PR** — create a pull request with summary from idea + design + plan
3. **Keep as-is** — leave the branch for manual review
4. **Abandon** — discard the work, set status to "abandoned"

## Step 5: Update Mem0

Based on user choice:
- Merge/PR → status: "done"
- Keep → status: "needs_review" (unchanged)
- Abandon → status: "abandoned"

Set updated_at, last_action_at → now.

## Step 6: Summary

```
[idea title] — DONE
━━━━━━━━━━━━━━━━━━

Status: [done/needs_review/abandoned]
Tests: [X passed, 0 failed]
TDD artifacts: docs/tdd/[idea-id]/
Commits: [N] commits
[PR URL if created]

Pipeline complete. Run /eclipse-tools:next for your next task.
```
```

**Step 2: Verify**

Run: `claude --plugin-dir ./`
Then: `/eclipse-tools:finish`
Expected: Skill loads.

**Step 3: Commit**

```bash
git add skills/finish/SKILL.md
git commit -m "feat: add finish skill for build validation and integration"
```

---

## Task 13: Hooks

**Files:**
- Create: `hooks/hooks.json`

**Step 1: Create hooks config**

Create `hooks/hooks.json`:

```json
{
  "hooks": {}
}
```

Note: We start with empty hooks. Hooks can be added later for:
- SessionEnd: auto-persist session insights to Mem0
- PostToolUse on Write/Edit: validate files against scope contract during build

**Step 2: Commit**

```bash
git add hooks/hooks.json
git commit -m "feat: add hooks skeleton"
```

---

## Task 14: Final Verification + README

**Files:**
- Verify: all files present
- Test: full plugin load

**Step 1: Verify structure**

```bash
find . -type f -not -path './.git/*' | sort
```

Expected output:
```
./.claude-plugin/plugin.json
./.mcp.json
./agents/code-quality-reviewer.md
./agents/competitor-researcher.md
./agents/ideation-code-improvements.md
./agents/ideation-code-quality.md
./agents/ideation-documentation.md
./agents/ideation-performance.md
./agents/ideation-security.md
./agents/ideation-ux.md
./agents/implementer.md
./agents/roadmap-discovery.md
./agents/roadmap-features.md
./agents/spec-reviewer.md
./docs/plans/2026-03-05-eclipse-tools-design.md
./docs/plans/2026-03-05-eclipse-tools-implementation-plan.md
./hooks/hooks.json
./settings.json
./skills/build/SKILL.md
./skills/competitor/SKILL.md
./skills/design/SKILL.md
./skills/finish/SKILL.md
./skills/ideation/SKILL.md
./skills/next/SKILL.md
./skills/plan/SKILL.md
./skills/remember/SKILL.md
./skills/roadmap/SKILL.md
```

**Step 2: Test plugin load**

Run: `claude --plugin-dir ./`
Run: `/help`
Expected: All 8 user-invocable skills listed under `eclipse-tools:` namespace:
- eclipse-tools:ideation
- eclipse-tools:roadmap
- eclipse-tools:competitor
- eclipse-tools:next
- eclipse-tools:design
- eclipse-tools:plan
- eclipse-tools:build
- eclipse-tools:finish

(remember is not user-invocable)

Run: `/agents`
Expected: All 12 agents listed.

**Step 3: Commit**

```bash
git add docs/plans/2026-03-05-eclipse-tools-implementation-plan.md
git commit -m "docs: add implementation plan"
```
