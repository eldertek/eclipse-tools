---
name: build
description: >
  Execute a TDD implementation plan task-by-task using subagents.
  Each task: implementer (RED-GREEN-COMMIT) → spec-reviewer → code-quality-reviewer.
  Strict Mem0 transitions, max 3 review cycles, mandatory TDD artifacts.
disable-model-invocation: true
argument-hint: "[ECL-IDEA-NNNN or ECL-FEAT-NNNN]"
---

# Eclipse Build Pipeline

You execute an implementation plan using subagent-driven TDD development.

## HARD RULES

- Mem0 transitions are STRICT: planned → building → (blocked | needs_review)
- You CANNOT skip from planned to done — must go through building
- Max 3 review cycles per task. After 3 REFUSED → status=blocked, STOP
- Every task MUST produce a TDD artifact. No artifact = task not done.
- Create `docs/tdd/[idea-id]/` directory BEFORE executing any task

## Step 1: Resolve Target Item

Accepts both ECL-IDEA and ECL-FEAT items.

1. ECL-* ID → fetch from Mem0 (status: planned)
2. Text → match in Mem0
3. No args → smart default (first `planned` by priority, non-ambiguity rule)

Verify:
- Status is "planned"
- `plan_ref` exists and points to a readable file
- If status is not "planned" → "This item has status [status]. Expected: planned. Run /eclipse-tools:plan first?"

## Step 2: Load Plan

1. Read the plan document (`plan_ref` path)
2. Extract all tasks with full spec:
   - task_id, title, test_file, test_command, impl_files
   - allowed_paths, definition_of_done, commit_message, tdd_artifact_ref
3. Count total tasks (M)

## Step 3: Prepare TDD Directory

```bash
mkdir -p docs/tdd/[idea-id]/
```

This MUST happen before any task execution. If the directory already exists (resumed build), that's fine.

## Step 4: Update Mem0 → building

- Status → "building"
- `updated_at`, `last_action_at` → now
- This is the ONLY transition at this point. Do NOT set needs_review or done yet.

## Step 5: Execute Tasks (Sequential)

For EACH task in order (task 1, task 2, ... task M):

### 5a. Announce

```
Task [N/M]: [title]
━━━━━━━━━━━━━━━━━
```

### 5b. Dispatch Implementer

Launch `implementer` agent with:
- Full task spec from plan
- Idea ID for artifact naming
- Task ID

Wait for result. Handle each case:

- `"PASS"` → continue to 5c
- `"FAIL"` → update Mem0: status="blocked", blocked_reason="Task [N] failed: [reason]". Display error. STOP.
- `"ALREADY_PASSING"` → log warning, skip RED phase verification in review but still require GREEN + COMMIT. Continue to 5c.
- `"QUESTION"` → display question to user, get answer, re-dispatch implementer with answer. Max 2 questions per task.

### 5c. Dispatch Spec Reviewer

Launch `spec-reviewer` agent with:
- Task spec (allowed_paths, definition_of_done, commit_message, impl_files, tdd_artifact_ref)
- TDD artifact path (from implementer result)
- Git commit SHA (from implementer result)

**Review cycle (max 3):**

1. If `"APPROVED"` → continue to 5d
2. If `"REFUSED"`:
   - Display refused checks to user
   - Re-dispatch implementer with fix instructions (include REFUSED reasons)
   - Re-dispatch spec-reviewer (cycle 2)
   - If refused again → cycle 3
   - If refused after cycle 3 → update Mem0: status="blocked", blocked_reason="Spec review failed after 3 cycles: [reasons]". STOP.

### 5d. Dispatch Code Quality Reviewer

Launch `code-quality-reviewer` agent with:
- allowed_paths, impl_files, test_file from task spec
- Git commit SHA

**Review cycle (max 3, shared counter with spec review):**

1. If `"APPROVED"` → task complete
2. If `"ISSUES"` with blocking items (bug/security):
   - Re-dispatch implementer to fix blocking issues
   - Re-review (respect max 3 total cycles for this task)
   - If still blocked after max cycles → status="blocked", STOP
3. If `"APPROVED"` or only non-blocking suggestions → task complete

### 5e. Report Task Progress

```
Task [N/M] complete: [title]
  TDD: RED ✓ → GREEN ✓ → COMMIT ✓
  Spec review: APPROVED (cycle [N])
  Quality review: APPROVED
  Artifact: docs/tdd/[idea-id]/task-NNN.json
```

## Step 6: All Tasks Complete

1. Verify artifact count: count files in `docs/tdd/[idea-id]/` — must equal total tasks (M)
2. If count mismatch → log warning: "Expected [M] artifacts, found [count]. Some tasks may be incomplete."
3. Update Mem0: status → "needs_review"
4. Display summary:

```
Build Complete — [idea title]
━━━━━━━━━━━━━━━━━━━━━━━━━━━

Tasks: [M/M] complete
TDD artifacts: docs/tdd/[idea-id]/ ([count] artifacts)
Commits: [list of SHAs with messages]
Review cycles used: [total across all tasks]

Status: needs_review
Ready for final validation. Run /eclipse-tools:finish [id]
```

## Error Handling

- Plan file missing → "Plan not found at [path]. Re-run /eclipse-tools:plan?"
- Mem0 write failure during status transition → warn but continue execution
- Agent crash/timeout → treat as FAIL, set blocked
- Git conflict during commit → set blocked with reason "merge conflict"

## Resuming a Blocked Build

If the item has status "blocked":
1. Display the blocked_reason
2. Ask: "Fix the issue and resume? (yes / abandon / restart from task N)"
3. If resume → set status back to "building", continue from the failed task
4. If abandon → set status to "abandoned"
5. If restart from N → reset to "building", start from task N
