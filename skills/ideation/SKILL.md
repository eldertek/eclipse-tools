---
name: ideation
description: >
  Run a comprehensive ideation analysis on the current project. Launches 6 specialized agents in parallel
  to find code improvements, quality issues, documentation gaps, performance optimizations, security
  vulnerabilities, and UI/UX issues. Results are stored in Mem0 as draft ideas for triage.
argument-hint: "[focus area]"
---

# Eclipse Ideation Pipeline

You orchestrate a comprehensive analysis of the current project by dispatching 6 specialized agents in parallel.

## Pre-flight

### 1. Detect Project Context

Read project files to identify the tech stack:
- `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, or similar
- Identify: language, framework, test runner, test command, package manager, database

Store/update this as ECL-CTX in Mem0 (if available). Schema:

```json
{
  "schema_version": 2,
  "project_id": "[calculated]",
  "id": "ECL-CTX-0001",
  "project_name": "[from package.json name or directory]",
  "tech_stack": {
    "language": "TypeScript",
    "framework": "Next.js 14",
    "test_runner": "vitest",
    "test_command": "npx vitest run",
    "package_manager": "pnpm",
    "database": "PostgreSQL + Prisma"
  },
  "dev_url": null,
  "workspace_root": null,
  "monorepo": false
}
```

### 2. Check Mem0 for Existing Items

Search Mem0 for all items with this project_id. Build a compact summary for agents:

```
Existing items (do NOT re-suggest these):
- ECL-IDEA-0001 [draft] fingerprint:abc123 "Add Redis caching"
- ECL-IDEA-0002 [rejected] fingerprint:def456 "Migrate to Bun"
- ECL-IDEA-0003 [done] fingerprint:ghi789 "Fix N+1 queries"
```

This summary is passed to every agent so they can avoid duplicates.
If Mem0 is unavailable, pass an empty summary and proceed without dedup context.

### 3. Filter Agents (if $ARGUMENTS provided)

If user passed a focus area, only launch matching agents:
- "security" → ideation-security only
- "perf" or "performance" → ideation-performance only
- "quality" or "code" → ideation-code-improvements + ideation-code-quality
- "docs" or "documentation" → ideation-documentation only
- "ux" or "ui" → ideation-ux only
- No arguments or "all" → launch all 6

## Browser Pre-check (for UX agent)

Before dispatching agents, check if a dev server is running:

1. Read ECL-CTX for `dev_url` — if set, use it
2. Else, read package.json `scripts` for "dev"/"start"/"serve" — extract port
3. Attempt to navigate to the detected URL using agent-browser CLI:
   ```bash
   agent-browser open <detected-url> && agent-browser wait --load networkidle
   ```
4. If successful:
   - Take screenshots: `agent-browser screenshot --full`
   - Capture accessibility snapshot: `agent-browser snapshot -i -c`
   - Pass screenshots and snapshot observations to the UX agent as extra context
   - Close browser: `agent-browser close`
5. If failed or no URL detected:
   - Log: "No running app detected — UX agent will use static analysis only"
   - Pass no browser context to UX agent
6. **NEVER block the pipeline** if agent-browser fails. Catch all errors, log, continue.

## Dispatch Phase

Launch ALL selected agents **in PARALLEL** using the Agent tool.

Each agent receives:
- The project root path
- The Mem0 summary (existing items for dedup avoidance)
- The ECL-CTX tech stack info
- For UX agent: Browser observations if available

Prompt template for each agent:

```
Analyze the project at [root_path] for [agent specialty].

Tech stack: [from ECL-CTX]

Existing Mem0 items (do NOT re-suggest):
[compact summary]

[For UX agent only: Browser observations: ...]

Return your findings as a strict JSON array. Return [] if nothing found.
```

Agents to dispatch:
1. `ideation-code-improvements` — "code improvement opportunities"
2. `ideation-code-quality` — "code quality issues"
3. `ideation-documentation` — "documentation gaps"
4. `ideation-performance` — "performance optimizations"
5. `ideation-security` — "security vulnerabilities"
6. `ideation-ux` — "UI/UX and accessibility improvements"

## Aggregation Phase

1. Collect JSON arrays from all agents
2. If an agent returned invalid JSON, log a warning and skip it (do not fail the pipeline)
3. Merge all arrays into a single list
4. Run deduplication (using remember skill patterns):
   - **Pass 1:** Exact fingerprint match against Mem0 items → skip
   - **Pass 2:** Exact fingerprint match across current batch (inter-agent dedup) → skip
   - **Pass 3:** Jaccard near-duplicate (>0.6) → flag for user decision later
5. Assign IDs: `ECL-IDEA-{NNNN}` (sequential from highest existing in Mem0)
6. Set all items to `status: "draft"`
7. Add `schema_version: 2`, `project_id`, `source: "ideation"`, timestamps

## Persist Phase

For each deduplicated idea:
1. Store in Mem0 with full schema
2. Track counts: total generated, exact duplicates skipped, near-duplicates flagged

If Mem0 unavailable:
- Display all ideas to user anyway (console output)
- Warn: "Mem0 unavailable — ideas not persisted. Set MEM0_API_KEY to enable persistence."

## Output Phase

Display summary table:

```
Ideation Complete — [total] ideas generated
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 # | Type            | Title                              | Effort  | Evidence
---+-----------------+------------------------------------+---------+----------
 1 | performance     | Cache Redis API endpoints          | medium  | 3 files
 2 | security        | Rate limiting sur /api/auth        | small   | 2 files
 3 | code_quality    | Extract validation utils           | trivial | 5 files
 ...

Saved to Mem0: [saved] items ([skipped] duplicates skipped, [flagged] near-duplicates flagged)
```

If near-duplicates were flagged:
```
Near-duplicates detected:
  ECL-IDEA-0015 "Add Redis cache" ~ ECL-IDEA-0001 "Redis caching for API" (Jaccard: 0.71)
  → Merge / Keep both / Skip ?
```

Then ask: "Voulez-vous trier maintenant ? (oui / non / /eclipse-tools:next plus tard)"

## Rapid Triage (if user says yes)

Enter the same triage mode as `/eclipse-tools:next`:

```
[1/N] ECL-IDEA-0013: "Title"
      Type: [type] | Effort: [effort] | Evidence: [count] files
      → (A)ccept [priority]  (D)efer "reason"  (R)eject "reason"  (S)kip
```

Rules:
- `A` or `A high` / `A critical` / `A medium` / `A low` — default: medium
- `R "reason"` — reason REQUIRED
- `D "reason"` — reason REQUIRED
- `S` — stays draft
- Write each decision to Mem0 immediately

After triage:
```
Tri termine: X accepted, Y rejected, Z deferred, W skipped
→ Lancer /eclipse-tools:design sur une idea acceptee ?
```
