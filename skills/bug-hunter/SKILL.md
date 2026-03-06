---
name: bug-hunter
description: >
  Autonomous bug hunting skill. Explores code, tests, and UI (via agent-browser)
  to find concrete, reproducible bugs. Produces a bug packet with repro steps,
  root cause (file:line), and proof. Waits for explicit validation before fixing.
  After fix is validated and committed, automatically relaunches to hunt the next bug.
  Uses Mem0 (ECL-BUG) with heatmap strategy to explore least-covered zones first.
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
