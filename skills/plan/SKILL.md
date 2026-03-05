---
name: plan
description: >
  Create a TDD implementation plan from a design document. Produces bite-sized tasks with
  exact file paths, test commands, allowed_paths scope contracts, and definitions of done.
  Detects the project's test runner and generates stack-specific commands.
  No code "novels" — each task is one RED-GREEN-COMMIT cycle.
disable-model-invocation: true
argument-hint: "[ECL-IDEA-NNNN or ECL-FEAT-NNNN]"
---

# Eclipse Implementation Plan

You create a strict TDD implementation plan from a design document.

## HARD RULES

- Every task = exactly one RED → GREEN → COMMIT cycle
- No task touches more than 5 files
- No "implement the feature" mega-tasks — split into atomic units
- Each task's test MUST be runnable independently
- Plan is a recipe, not a novel — exact code, exact commands, no prose

## Step 1: Resolve Target Item

Accepts both ECL-IDEA and ECL-FEAT items.

1. ECL-* ID → fetch from Mem0
2. Text → match in Mem0 (status: designed)
3. No args → smart default (first `designed` item by priority, non-ambiguity rule applies)

Verify the item has status "designed" and a `design_ref`.
If no design_ref → "This item needs a design first. Run /eclipse-tools:design [id]"

## Step 2: Load Context

1. Read the design document (`design_ref` path)
2. Read ECL-CTX from Mem0 for tech stack info
3. If ECL-CTX missing or incomplete, detect:
   - Test runner: vitest, jest, pytest, cargo test, go test, mocha, ava
   - Test command: `npx vitest run`, `npx jest`, `pytest -v`, `cargo test`, `go test ./...`
   - Package manager: npm, pnpm, yarn, pip, cargo
   - Store detection results back in ECL-CTX

## Step 3: Generate Tasks

For EACH task:

```json
{
  "task_id": "task-NNN",
  "title": "Short imperative title",
  "test_file": "exact/path/to/test.ts",
  "test_command": "npx vitest run exact/path/to/test.ts",
  "impl_files": ["exact/path/to/impl.ts"],
  "allowed_paths": ["exact/path/to/impl.ts", "exact/path/to/test.ts"],
  "definition_of_done": [
    "Specific verifiable criterion 1",
    "Specific verifiable criterion 2"
  ],
  "commit_message": "feat(scope): short description",
  "tdd_artifact_ref": "docs/tdd/[idea-id]/task-NNN.json"
}
```

### Task Generation Rules

- `allowed_paths` MUST list specific files, NOT directories. Max 5 files.
- `test_command` MUST be the exact command adapted to the detected test runner
- `definition_of_done` MUST be verifiable assertions (pass/fail), not vague goals
- `tdd_artifact_ref` MUST follow the template `docs/tdd/[idea-id]/task-NNN.json`
- `commit_message` follows conventional commits: `feat|fix|refactor|test(scope): description`
- Each task's test MUST fail before implementation (true RED phase)

### Task Ordering

- Dependencies first: if task B imports from task A's impl, A comes first
- Tests before implementation: if a shared test helper is needed, create it in task 1
- Infrastructure before features: DB migrations, config, types come first

## Step 4: Write Plan Document

Save to `docs/plans/YYYY-MM-DD-<topic>-plan.md`

### Plan Header

```markdown
# [Title] Implementation Plan

**Item:** [ECL-IDEA-NNNN or ECL-FEAT-NNNN]
**Design:** [path to design doc]
**Date:** YYYY-MM-DD
**Tasks:** [N]
**Test runner:** [detected runner]
**Test command pattern:** [pattern]
```

### Task Format

For each task:

````markdown
### Task N: [Title]

**Scope contract:**
- allowed_paths: [`file1`, `file2`]
- definition_of_done:
  1. [criterion]
  2. [criterion]
- commit_message: `feat(scope): description`
- tdd_artifact: `docs/tdd/[idea-id]/task-NNN.json`

**Step 1: Write the failing test**

```typescript
// exact/path/to/test.ts
import { thing } from '../impl';

describe('thing', () => {
  it('should do X', () => {
    expect(thing(input)).toBe(expected);
  });
});
```

**Step 2: Run test to verify it fails**

Run: `npx vitest run exact/path/to/test.ts`
Expected: FAIL with "Cannot find module" or "thing is not a function"

**Step 3: Write minimal implementation**

```typescript
// exact/path/to/impl.ts
export function thing(input: Type): ReturnType {
  return expected;
}
```

**Step 4: Run test to verify it passes**

Run: `npx vitest run exact/path/to/test.ts`
Expected: PASS — 1 test passed

**Step 5: Commit**

```bash
git add exact/path/to/test.ts exact/path/to/impl.ts
git commit -m "feat(scope): description"
```
````

## Step 5: Update Mem0

- Status → "planned"
- `plan_ref` → path to plan document
- `updated_at`, `last_action_at` → now

## Step 6: Chain

```
Plan saved with [N] tasks to [path].

Two execution options:
1. Subagent-Driven (this session) — /eclipse-tools:build [id]
2. Later — /eclipse-tools:next will remind you

Which approach?
```

## Error Handling

- No design_ref → guide to /eclipse-tools:design
- Test runner not detected → ask user: "Which test runner does this project use?"
- Design doc missing or corrupted → error, ask to re-run /eclipse-tools:design
- Mem0 write failure → warn, plan doc is still saved locally
