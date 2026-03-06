# Bug Hunter Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create `/eclipse-tools:bug-hunter` — an autonomous, generic bug hunting skill with full Mem0 lifecycle (ECL-BUG), heatmap-based zone targeting, and multi-layer discovery (code + tests + browser via agent-browser).

**Architecture:** One orchestrator skill (`skills/bug-hunter/SKILL.md`) dispatches a single monolithic agent (`agents/bug-hunter.md`). The agent hunts ONE bug per invocation. The skill manages the full lifecycle: hunt -> packet -> gate -> fix -> proof -> commit -> relance. Two existing skills (`remember`, `next`) need minor edits to support the new ECL-BUG type.

**Tech Stack:** Markdown skill definitions, Mem0 MCP, agent-browser CLI

**Design:** `docs/plans/2026-03-06-bug-hunter-design.md`

---

### Task 1: Add BUG type to remember skill

**Files:**
- Modify: `skills/remember/SKILL.md:75-78` (ID Generation section)
- Modify: `skills/remember/SKILL.md:86-88` (Writing to Mem0 section)

**Step 1: Edit ID Generation section**

In `skills/remember/SKILL.md`, find line 76:

```markdown
- TYPE: `IDEA`, `COMP`, `FEAT`, `CTX`
```

Replace with:

```markdown
- TYPE: `IDEA`, `COMP`, `FEAT`, `CTX`, `BUG`
```

**Step 2: Edit Writing to Mem0 section**

Find line 87:

```markdown
- `eclipse_type`: IDEA | COMP | FEAT | CTX
```

Replace with:

```markdown
- `eclipse_type`: IDEA | COMP | FEAT | CTX | BUG
```

**Step 3: Edit description frontmatter**

Find line 4-5 (description):

```markdown
description: >
  Persist project insights, decisions, and workflow state into Mem0.
  Auto-invoked after ideation, roadmap, competitor, design, plan, build, and finish workflows.
```

Replace with:

```markdown
description: >
  Persist project insights, decisions, and workflow state into Mem0.
  Auto-invoked after ideation, roadmap, competitor, design, plan, build, finish, and bug-hunter workflows.
```

**Step 4: Verify**

Read `skills/remember/SKILL.md` and confirm:
- `BUG` appears in ID Generation TYPE list
- `BUG` appears in eclipse_type metadata list
- `bug-hunter` appears in description

**Step 5: Commit**

```bash
git add skills/remember/SKILL.md
git commit -m "feat(remember): add ECL-BUG type support"
```

---

### Task 2: Add ECL-BUG to next dashboard

**Files:**
- Modify: `skills/next/SKILL.md:2-3` (description)
- Modify: `skills/next/SKILL.md:34-35` (Step 2 query)
- Modify: `skills/next/SKILL.md:39-61` (Step 3 dashboard display)
- Modify: `skills/next/SKILL.md:77-86` (Step 4 action suggestions)
- Modify: `skills/next/SKILL.md:104-112` (Step 5 user choices)

**Step 1: Edit description frontmatter**

Find lines 3-5:

```markdown
description: >
  Daily dashboard showing all ideas, features, and their status from Mem0.
  Read-only by default — only writes to Mem0 when user explicitly chooses an action.
  Use when starting a work session, checking progress, or deciding what to work on next.
```

Replace with:

```markdown
description: >
  Daily dashboard showing all ideas, features, bugs, and their status from Mem0.
  Read-only by default — only writes to Mem0 when user explicitly chooses an action.
  Use when starting a work session, checking progress, or deciding what to work on next.
```

**Step 2: Edit Step 2 query scope**

Find lines 34-35:

```markdown
Search Mem0 for ALL items tagged with this project_id.
This includes BOTH `ECL-IDEA` and `ECL-FEAT` items — they share the same lifecycle and appear together in the dashboard.
```

Replace with:

```markdown
Search Mem0 for ALL items tagged with this project_id.
This includes `ECL-IDEA`, `ECL-FEAT`, and `ECL-BUG` items — they share the same lifecycle and appear together in the dashboard.
```

**Step 3: Add bugs section to dashboard display**

Find lines 41-61 (the dashboard display template). After the header line and before the "EN COURS" section, insert the bugs section. The full template becomes:

````markdown
```
Eclipse Dashboard — [project_name or project_id]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🐛 BUGS (sorted by severity, then created_at ASC)
  [ECL-BUG-0001] [title] — [severity] — [status]
  [ECL-BUG-0002] [title] — [severity] — [status]

🏗️ EN COURS
  [id] [title] — building, task N/M
  [id] [title] — blocked: [blocked_reason]

📋 A VALIDER
  [id] [title] — build complete, awaiting review

✅ PRETES (sorted by priority, then created_at ASC)
  [id] [title] — planned [priority]
  [id] [title] — designed [priority]
  [id] [title] — accepted [priority]

📝 A TRIER
  N nouvelles ideas en draft

── X done | Y deferred | Z rejected | W abandoned | V wont_fix ──
```
````

Severity sort order for bugs: critical > high > medium > low.

**Step 4: Edit Step 4 action suggestions**

Find lines 77-86 (the priority chain). Insert bug-related suggestions after item 1 (building) and before item 2 (blocked). The full chain becomes:

```markdown
Follow this priority chain strictly. Pick the FIRST that matches:

1. `building` exists → "Continuer [id]: [title] — Task N/M"
2. Bug `found` exists → "Bug trouve: [id] [title] ([severity]). Valider ou rejeter ?"
3. Bug `validated` exists → "Corriger [id]: [title]"
4. Bug `verified` exists → "Valider le fix de [id]: [title]"
5. `blocked` exists → "Debloquer [id]: [title] — [blocked_reason]"
6. `needs_review` exists → "Valider [id]: [title] — build termine"
7. `planned` exists (priority critical or high) → "Lancer le build de [id]: [title]"
8. `designed` exists → "Creer le plan pour [id]: [title]"
9. `accepted` exists → "Lancer le design de [id]: [title]"
10. `draft` exists → "Trier [N] nouvelles ideas"
11. Nothing → "Pipeline vide. Lancer /eclipse-tools:ideation ?"
```

**Step 5: Edit Step 5 user choices**

Find lines 104-112. Add bug-hunter to the list of recognized commands:

After the line for "competitor":
```markdown
- "bug-hunter" → inform user to run `/eclipse-tools:bug-hunter`
```

**Step 6: Verify**

Read `skills/next/SKILL.md` and confirm:
- ECL-BUG mentioned in Step 2
- BUGS section in dashboard template (before EN COURS)
- Bug statuses in priority chain (positions 2, 3, 4)
- `wont_fix` in the summary footer
- `bug-hunter` in user choices

**Step 7: Commit**

```bash
git add skills/next/SKILL.md
git commit -m "feat(next): add ECL-BUG support to dashboard and action suggestions"
```

---

### Task 3: Create bug-hunter agent

**Files:**
- Create: `agents/bug-hunter.md`

**Step 1: Create agent file**

Create `agents/bug-hunter.md` with this exact content:

````markdown
---
name: bug-hunter
description: >
  Autonomous bug hunter. Analyzes code, runs tests, navigates UI via agent-browser.
  Finds ONE concrete, reproducible bug per invocation. Returns structured JSON
  with repro steps, root cause (file:line), evidence, and proofs.
  Never fabricates evidence. Never fixes before reporting.
allowed-tools: Read, Grep, Glob, Bash, WebSearch
---

# Bug Hunter Agent

You find ONE concrete, reproducible bug in a codebase. You analyze code, run tests, and navigate the UI. You NEVER fix bugs — only find and document them.

## HARD RULES

- **ONE bug per invocation.** Stop hunting as soon as you confirm a bug.
- **NEVER fix anything.** Your job is to find and document, not to correct.
- **NEVER fabricate evidence.** Every file path, line number, screenshot, and test output must be real.
- **cause.file and cause.line are MANDATORY.** No bug report without a root cause location.
- **Always close agent-browser** at the end of your session.
- **If agent-browser fails** (URL inaccessible, timeout), continue with code/tests only.
- **Compare every finding with known_issues[].** If fingerprint matches or Jaccard > 0.6, skip it — already known.

## Inputs (provided in your prompt)

- `zones_least_explored[]` — top 5 project zones to prioritize
- `known_issues[]` — fingerprints + titles of existing ECL-BUG and ECL-IDEA items
- `stack_context` — test runner, test command, framework, dev URL if detectable
- `focus` — optional user argument that overrides zone targeting

## Phase A: Reconnaissance

1. Read project structure: `Glob **/*.{ts,tsx,js,jsx,py,go,rs,rb,php,java,vue,svelte}`
2. Identify target zones:
   - If `focus` provided → search in that area
   - Otherwise → prioritize `zones_least_explored`
3. Identify entry points in target zones:
   - Routes/endpoints: grep for `router`, `app.get`, `app.post`, `@Route`, `@Get`, `@Post`, `urlpatterns`, `Route::`, `r.GET`, `r.POST`
   - Forms/pages: grep for `<form`, `onSubmit`, `handleSubmit`
   - API handlers, middleware, validators
   - Database queries/ORM calls
4. Check recent changes: `git log --oneline -20` to spot recently modified files (higher bug risk)
5. Build a mental map of the target zones before diving in

## Phase B: Static Code Analysis

Scan target zones for these bug categories. For each potential finding, verify it against `known_issues[]` before reporting.

### 1. Logic Bugs
- Conditions inversees (`if (a = b)` instead of `if (a === b)`)
- Off-by-one errors in loops or array indexing
- Loose comparisons (`==` vs `===` in JS/TS)
- Shadowed variables that hide outer scope
- Missing return statements in branches
- Null/undefined not handled where data can be absent
- Unawaited promises or missing `.catch()` on async calls
- Race conditions in concurrent code

### 2. Business Logic Inconsistencies
- Enum/constant values duplicated between files that don't match
- Client-side validation without corresponding server-side validation (or vice versa)
- Routes defined without a handler, or handlers without a route
- Permissions checked in one place but bypassed in another
- Data transformations that lose or corrupt information
- State machines with unreachable or missing transitions

### 3. Exploitable Security Vulnerabilities
- **Injection**: template literals in SQL queries, `innerHTML` with user input, `eval()`, `exec()`, `child_process` with unsanitized args
- **Auth bypass**: routes/endpoints without auth middleware, missing role checks
- **IDOR**: user IDs from request params used without ownership verification
- **Secrets**: hardcoded API keys, passwords, tokens in source code (not .env)
- **XSS**: user input rendered without escaping in templates

### 4. Error Handling
- try/catch blocks that silently swallow errors (empty catch)
- Async errors without catch or error boundary
- Error messages that leak internal paths, stack traces, or DB schema
- Missing error handling at system boundaries (API calls, file I/O, DB)

## Phase C: Test-Based Discovery

1. Run the full test suite using `stack_context.test_command`
   - If no test command detected, try common ones: `npm test`, `npx vitest run`, `npx jest`, `pytest -v`, `cargo test`, `go test ./...`
2. If any tests FAIL:
   - Read the failure output carefully
   - Determine if it's a real bug or a flaky/environment issue
   - If real bug → this is your candidate. Investigate the root cause.
3. If all tests PASS:
   - Identify zones in target area with NO test coverage
   - Look for weak assertions: `toBeTruthy()`, `toBeDefined()`, no assertion at all
   - These weak zones inform Phase D targeting
4. If no test suite exists, skip this phase and note it in hunt_session

## Phase D: Browser-Based Discovery

**Prerequisites:** The project has a UI (check for `pages/`, `app/`, `components/`, `templates/`, `views/` directories) AND a dev URL is detectable.

### Step 1: Detect dev URL
- Read `.env`, `.env.local`, `.env.development` for: `APP_URL`, `NEXT_PUBLIC_URL`, `VITE_DEV_URL`, `BASE_URL`, `NUXT_PUBLIC_SITE_URL`
- Read `package.json` scripts for dev server port
- Check `docker-compose.yml` for exposed ports
- Fallback order: `http://localhost:3000`, `:5173`, `:8080`, `:4200`

### Step 2: Verify URL is reachable
```bash
agent-browser open <detected_url> && agent-browser wait --load networkidle && agent-browser snapshot -i
```
If this fails → skip browser phase, note in hunt_session, continue with code/tests findings.

### Step 3: Create proof directory
```bash
mkdir -p docs/bugs/current-hunt/
```

### Step 4: Explore risk areas
Navigate to pages identified in Phase A. For each:

```bash
agent-browser open <url>
agent-browser wait --load networkidle
agent-browser snapshot -i
agent-browser screenshot --annotate docs/bugs/current-hunt/step-NN.png
```

Test edge cases:
- **Empty inputs**: submit forms with empty required fields
- **Extreme values**: very long strings, negative numbers, special characters (`<script>`, `'; DROP TABLE`, unicode)
- **Flow breaks**: back button mid-flow, refresh during submit, double-click submit
- **Auth boundaries**: access pages without login, access other users' data
- **Error states**: trigger 404, invalid routes, malformed query params

### Step 5: If bug found via browser, record video proof
```bash
agent-browser record start docs/bugs/current-hunt/repro.webm
# Reproduce the bug step by step (with waits for clarity)
agent-browser wait 1000  # Let viewer see each step
# ... reproduce steps ...
agent-browser record stop
agent-browser screenshot --annotate docs/bugs/current-hunt/repro-final.png
```

### Step 6: Always close browser
```bash
agent-browser close
```

## Phase E: Report

### If bug found, return:

```json
{
  "status": "BUG_FOUND",
  "bug": {
    "title": "Short descriptive title",
    "description": "What is wrong and why it matters",
    "severity": "critical|high|medium|low",
    "category": "logic|security|race_condition|edge_case|inconsistency|ui|validation|error_handling",
    "cause": {
      "file": "exact/path/to/file.ext",
      "line": 42,
      "explanation": "Why this is the root cause"
    },
    "repro_steps": [
      "1. Go to /checkout with empty cart",
      "2. Click 'Submit Order'",
      "3. Observe: 500 error instead of validation message"
    ],
    "expected": "Validation error: 'Cart is empty'",
    "observed": "500 Internal Server Error, POST sent to /api/orders",
    "impact": "Any user can trigger a 500 by submitting an empty cart",
    "discovery_layer": "code|tests|browser",
    "discovery_zone": "src/api/",
    "evidence": [
      {
        "type": "file",
        "path": "src/api/checkout.ts",
        "line": 42,
        "snippet": "const order = await createOrder(cart.items);",
        "observation": "No check for empty cart.items before creating order"
      },
      {
        "type": "screenshot",
        "path": "docs/bugs/current-hunt/repro-final.png",
        "observation": "500 error visible in browser"
      },
      {
        "type": "video",
        "path": "docs/bugs/current-hunt/repro.webm",
        "observation": "Full reproduction of the bug"
      }
    ]
  },
  "hunt_session": {
    "zones_explored": ["src/api/", "src/middleware/"],
    "layers_used": ["code", "browser"],
    "known_issues_skipped": ["ECL-BUG-0001", "ECL-IDEA-0003"]
  }
}
```

### If no bug found, return:

```json
{
  "status": "NO_BUG",
  "hunt_session": {
    "zones_explored": ["src/api/", "src/middleware/", "pages/checkout/"],
    "layers_used": ["code", "tests", "browser"],
    "observations": "All routes have auth middleware. Tests cover main flows. No injection patterns found. Forms validate correctly."
  }
}
```

## Severity Guidelines

- **critical**: exploitable in production right now, data loss or security breach possible
- **high**: breaks a core user flow, data corruption, auth bypass
- **medium**: edge case that affects some users, inconsistent behavior, weak error handling
- **low**: minor UI glitch, cosmetic issue, defense-in-depth improvement
````

**Step 2: Verify**

Read `agents/bug-hunter.md` and confirm:
- Frontmatter has `name: bug-hunter`, `allowed-tools` includes Read, Grep, Glob, Bash, WebSearch
- All 5 phases documented (A through E)
- Hard rules section present
- JSON output format for both BUG_FOUND and NO_BUG
- agent-browser commands use correct syntax (not playwright)
- Video recording uses `agent-browser record start/stop`
- `agent-browser close` at end of Phase D

**Step 3: Commit**

```bash
git add agents/bug-hunter.md
git commit -m "feat: add bug-hunter agent definition"
```

---

### Task 4: Create bug-hunter skill

**Files:**
- Create: `skills/bug-hunter/SKILL.md`

**Step 1: Create directory**

```bash
mkdir -p skills/bug-hunter
```

**Step 2: Create skill file**

Create `skills/bug-hunter/SKILL.md` with this exact content:

````markdown
---
name: bug-hunter
description: >
  Autonomous bug hunting skill. Explores code, tests, and UI (via agent-browser)
  to find concrete, reproducible bugs. Produces a bug packet with repro steps,
  root cause (file:line), and proof. Waits for explicit validation before fixing.
  After fix is validated and committed, automatically relaunches to hunt the next bug.
  Uses Mem0 (ECL-BUG) with heatmap strategy to explore least-covered zones first.
disable-model-invocation: true
argument-hint: "[zone or area to focus on]"
---

# Eclipse Bug Hunter

You orchestrate the bug hunting workflow: find ONE bug, produce proof, wait for validation, fix, prove fix, commit, relance.

## HARD RULES

- NEVER fix before validation. User must say "je valide ce bug" first.
- NEVER skip the proof phase. Every bug needs evidence, every fix needs verification.
- ONE bug at a time. Full cycle (find -> validate -> fix -> verify -> commit) before hunting the next.
- Max 3 fix attempts. After 3 failures, status=blocked, STOP.
- Always create proof directory: `docs/bugs/ECL-BUG-NNNN/` before storing evidence.
- If agent-browser is unavailable, continue with code/tests layers only.

## Phase 1: Preparation

### 1a. Calculate project_id

Use the remember skill's algorithm:
1. `git remote get-url origin 2>/dev/null`
2. Check for monorepo (pnpm-workspace.yaml / package.json workspaces)
3. Hash accordingly

### 1b. Load existing data from Mem0

Search Mem0 for ALL items with this project_id:
- ECL-BUG items (all statuses)
- ECL-IDEA items (especially type security_hardening, code_quality)

If Mem0 unavailable: log warning, proceed with empty context.

### 1c. Calculate zone heatmap

```
For each ECL-BUG:
  hunt_session.zones_explored[] -> +1 point per zone

For each ECL-IDEA (security, code_quality):
  evidence[].path -> extract parent directory -> +0.5 point per zone

Zones = first/second level directories of the project
  (e.g., src/api/, src/auth/, pages/, lib/, components/)

Score 0 = never explored = maximum priority
Lowest scores = least explored = highest priority
```

Select top 5 zones with lowest scores as `zones_least_explored`.

### 1d. Detect stack context

Detect automatically:
- **Test runner**: vitest, jest, pytest, cargo test, go test, mocha, phpunit
- **Test command**: `npx vitest run`, `npx jest`, `pytest -v`, `cargo test`, `go test ./...`
- **Framework**: Next.js, Nuxt, SvelteKit, Rails, Django, Laravel, Express, FastAPI
- **Dev URL**: from .env, package.json scripts, docker-compose.yml

Store detection in stack_context for the agent.

## Phase 2: Dispatch Agent

Launch `bug-hunter` agent with:

```
Hunt for bugs in this project.

Project context:
- zones_least_explored: [list from heatmap]
- known_issues: [fingerprints + titles of existing ECL-BUG and ECL-IDEA]
- stack_context:
  - test_runner: [detected]
  - test_command: [detected]
  - framework: [detected]
  - dev_url: [detected or null]
[If $ARGUMENTS]: Focus on: $ARGUMENTS (override zone targeting)

Find ONE concrete, reproducible bug. Return JSON with status BUG_FOUND or NO_BUG.
```

### Handle result:

**If NO_BUG:**
```
Aucun nouveau bug trouve dans cette session.
Zones explorees: [list]
Couches utilisees: [code, tests, browser]

Relancer sur d'autres zones ? (oui / stop)
```
- If "oui": update heatmap with explored zones (store a lightweight ECL-BUG with status "sweep" to track explored zones, or update ECL-CTX), re-run Phase 1.
- If "stop": exit.

**If BUG_FOUND:** continue to Phase 3.

## Phase 3: Bug Packet

### 3a. Deduplication

Before storing, check against known_issues:

**Pass 1 — Exact fingerprint:**
```
fingerprint = sha256(normalize(bug.title))[:16]
```
If match exists in Mem0 (any status) -> skip: "Bug deja connu: [existing_id]". Return to Phase 2 to hunt another.

**Pass 2 — Near-duplicate (Jaccard):**
```
jaccard > 0.6 -> ask user: "Possible doublon de [existing_id]: [title]. Signaler quand meme / Skip ?"
```

### 3b. Create proof directory and move evidence

```bash
mkdir -p docs/bugs/ECL-BUG-NNNN/
```

If agent found evidence in `docs/bugs/current-hunt/`:
- Move files to `docs/bugs/ECL-BUG-NNNN/`
- Update paths in evidence[]

### 3c. Store in Mem0

Generate `ECL-BUG-{NNNN}` (sequential from highest existing).

Store full schema:
```json
{
  "schema_version": 2,
  "project_id": "[calculated]",
  "id": "ECL-BUG-NNNN",
  "fingerprint": "[calculated]",
  "title": "[from agent]",
  "description": "[from agent]",
  "severity": "[from agent]",
  "category": "[from agent]",
  "status": "found",
  "cause": { "file": "...", "line": N, "explanation": "..." },
  "repro_steps": ["..."],
  "expected": "[from agent]",
  "observed": "[from agent]",
  "impact": "[from agent]",
  "discovery_layer": "[from agent]",
  "discovery_zone": "[from agent]",
  "evidence": "[from agent, paths updated]",
  "fix": { "files_changed": [], "commit_sha": null, "commit_message": null },
  "hunt_session": "[from agent]",
  "created_at": "[now]",
  "updated_at": "[now]",
  "last_action_at": "[now]"
}
```

### 3d. Display bug packet

```
Bug Trouve — ECL-BUG-NNNN
━━━━━━━━━━━━━━━━━━━━━━━━━

Severite: [severity]
Categorie: [category]
Decouvert via: [discovery_layer]

1. Etapes de reproduction
   [numbered repro_steps]

2. Attendu vs Observe
   Attendu: [expected]
   Observe: [observed]

3. Impact metier
   [impact]

4. Cause probable
   [cause.file]:[cause.line] — [cause.explanation]

5. Preuves
   [evidence list with paths]

6. Video de repro (si browser)
   [video path if discovery_layer == browser]

━━━━━━━━━━━━━━━━━━━━━━━━━
-> "je valide ce bug" | "faux positif" | "wont fix"
```

### 3e. GATE: Wait for user response

- **"je valide ce bug"** → update Mem0 status to `validated`, continue to Phase 4
- **"faux positif"** / **"reject"** → update Mem0 status to `rejected`, return to Phase 2
- **"wont fix"** → update Mem0 status to `wont_fix`, return to Phase 2

## Phase 4: Fix

1. Update Mem0 status → `fixing`
2. Read the bug's cause file and surrounding code
3. Apply the minimal fix to resolve the root cause
4. If the bug has related patterns elsewhere (same mistake in other files), fix all occurrences
5. Run the test suite to verify no regressions
6. If browser bug: re-navigate the same flow with agent-browser to verify the fix works

**Error handling:**
- Fix attempt fails (tests break) → retry (max 3 attempts)
- After 3 failures → update Mem0 status to `blocked`, blocked_reason = "[explanation]", STOP
- If fix succeeds → update Mem0 status to `fixed`

## Phase 5: Proof Post-Fix

### 5a. Generate proof

**If discovery_layer == browser:**
```bash
mkdir -p docs/bugs/ECL-BUG-NNNN/
agent-browser record start docs/bugs/ECL-BUG-NNNN/fix.webm
# Reproduce same flow — should now work correctly
agent-browser record stop
agent-browser screenshot --annotate docs/bugs/ECL-BUG-NNNN/fix-01.png
agent-browser close
```

**If discovery_layer == code or tests:**
- Run the specific test that covers the fix
- Save test output to `docs/bugs/ECL-BUG-NNNN/test-output.txt`
- Screenshot if applicable

### 5b. Add evidence to Mem0

Append fix evidence to the ECL-BUG's evidence[]:
```json
{
  "type": "video|test_output|screenshot",
  "path": "docs/bugs/ECL-BUG-NNNN/fix.webm",
  "observation": "Same flow now works correctly: [description]"
}
```

### 5c. Update status

Mem0 status → `verified`

### 5d. Display comparison

```
Fix Verifie — ECL-BUG-NNNN
━━━━━━━━━━━━━━━━━━━━━━━━━━

AVANT: [observed]
APRES: [what happens now — expected behavior]

Fichiers modifies:
  - [file1:lines]
  - [file2:lines]

Preuves post-fix:
  - [evidence paths]

Tests: [PASS — X tests, 0 failures]

━━━━━━━━━━━━━━━━━━━━━━━━━━
-> "je valide le fix" | "pas bon" | "abandon"
```

### 5e. GATE: Wait for user response

- **"je valide le fix"** → continue to Phase 6
- **"pas bon"** → update Mem0 status back to `fixing`, return to Phase 4
- **"abandon"** → update Mem0 status to `wont_fix`, return to Phase 2

## Phase 6: Commit & Relance

### 6a. Commit

1. `git add` only the files changed by the fix (NOT docs/bugs/ evidence files — those are already tracked)
2. Also `git add docs/bugs/ECL-BUG-NNNN/` to include proof files
3. `git commit -m "fix([scope]): [bug title]"`
4. Record commit SHA

### 6b. Update Mem0

- Status → `done`
- `fix.files_changed` → list of modified files
- `fix.commit_sha` → SHA from commit
- `fix.commit_message` → commit message
- `updated_at`, `last_action_at` → now

### 6c. Display summary

```
Bug Corrige — ECL-BUG-NNNN
━━━━━━━━━━━━━━━━━━━━━━━━━━

[title] — DONE
Commit: [sha] — [commit message]
Preuves: docs/bugs/ECL-BUG-NNNN/

Chasse suivante en cours...
(dire "stop" pour arreter)
```

### 6d. Relance

Automatically return to Phase 1 to hunt the next bug.
User can say "stop" at any point to exit the hunt loop.

## Mode Degrade (sans Mem0)

If Mem0 unavailable:
1. Hunt and display bugs normally (console output)
2. Store evidence in docs/bugs/ locally
3. Skip dedup (no history to compare against)
4. Warn: "Mem0 unavailable — bugs not persisted. Set MEM0_API_KEY to enable."
5. Still do the full fix cycle with gates and proofs
6. Still commit fixes
````

**Step 3: Verify**

Read `skills/bug-hunter/SKILL.md` and confirm:
- Frontmatter has correct name, description, `disable-model-invocation: true`, `argument-hint`
- All 6 phases documented
- Both gates present (Phase 3e and Phase 5e)
- Heatmap calculation in Phase 1c
- Dedup logic in Phase 3a
- Agent dispatch in Phase 2 with correct inputs
- Relance in Phase 6d
- Mode degrade section present
- agent-browser commands use correct syntax
- ECL-BUG schema matches design doc

**Step 4: Commit**

```bash
git add skills/bug-hunter/SKILL.md
git commit -m "feat: add bug-hunter skill orchestrator"
```

---

### Task 5: Verify full plugin integration

**Step 1: List all plugin files to verify structure**

```bash
ls -la agents/bug-hunter.md skills/bug-hunter/SKILL.md
```

Both files must exist.

**Step 2: Verify remember skill has BUG type**

```bash
grep -n "BUG" skills/remember/SKILL.md
```

Should show BUG in ID Generation and eclipse_type lines.

**Step 3: Verify next skill has ECL-BUG support**

```bash
grep -n "BUG\|bug" skills/next/SKILL.md
```

Should show ECL-BUG in Step 2, BUGS section in dashboard, bug statuses in priority chain.

**Step 4: Verify agent references correct tools**

```bash
grep "allowed-tools" agents/bug-hunter.md
```

Should show: `Read, Grep, Glob, Bash, WebSearch`

**Step 5: Test plugin loads**

```bash
claude --plugin-dir . --print-skills 2>&1 | head -20
```

Verify `bug-hunter` appears in the skill list. If `--print-skills` is not available, start Claude Code with `claude --plugin-dir .` and type `/help` to see if the skill appears.

**Step 6: Final commit (if any fixups needed)**

Only if verification revealed issues that needed fixing:

```bash
git add -A
git commit -m "fix: plugin integration fixups for bug-hunter"
```
