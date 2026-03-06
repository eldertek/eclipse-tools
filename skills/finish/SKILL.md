---
name: finish
description: >
  Finalize a completed build: run full test suite, verify TDD artifacts (1 per task),
  prepare merge/PR text. Updates Mem0 status to "done".
argument-hint: "[ECL-IDEA-NNNN or ECL-FEAT-NNNN]"
---

# Eclipse Finish

You finalize a completed build by verifying everything works and proposing integration.

## Step 1: Resolve Target Item

Accepts both ECL-IDEA and ECL-FEAT items.

1. ECL-* ID → fetch from Mem0 (status: building or needs_review)
2. No args → smart default (first `needs_review`, then `building`, non-ambiguity rule)

If no matching item → "No items ready to finish. Run /eclipse-tools:next for status."

## Step 2: Detect Test Command

1. Read ECL-CTX from Mem0 for `test_command`
2. If ECL-CTX has a `test_command_full` field → use it (project-level full suite command)
3. If not, detect from project:
   - Check `package.json` scripts for `test` key → `npm test` or `pnpm test`
   - Check for `vitest.config.*` → `npx vitest run`
   - Check for `jest.config.*` → `npx jest`
   - Check for `pytest.ini` / `pyproject.toml [tool.pytest]` → `pytest -v`
   - Check for `Cargo.toml` → `cargo test`
   - Check for `go.mod` → `go test ./...`
4. If detection fails → ask user: "What command runs your full test suite?"

Store the detected command as `test_command_full` in ECL-CTX for future use.

## Step 3: Run Full Test Suite

1. Run the detected full test command
2. Capture full output
3. Parse results:
   - Total tests, passed, failed, skipped
   - If any FAIL:
     ```
     Test suite: FAIL
     [N] failed tests:
       - [test name]: [failure reason]

     Options:
       1. Fix the failures (debug mode)
       2. Ignore and continue (mark as needs_review)
       3. Abandon the build
     ```
   - If user chooses "fix" → guide debugging, re-run tests after fix
   - If user chooses "ignore" → keep status as needs_review, warn
   - If user chooses "abandon" → status="abandoned" in Mem0, STOP

4. If all PASS → continue

## Step 4: Verify TDD Artifacts

1. Read the plan document to get total task count (M)
2. List files in `docs/tdd/[idea-id]/`
3. Count artifact files (*.json)
4. Verify: artifact count == M
5. If mismatch:
   ```
   TDD Artifacts: WARNING
   Expected: [M] artifacts (one per task)
   Found: [count]
   Missing: task-003.json, task-007.json

   Continue anyway? (yes / investigate)
   ```
6. For each artifact, quick-verify:
   - `red.exit_code != 0`
   - `green.exit_code == 0`
   - `commit.sha` is present
   - `scope_check.all_in_scope == true`

Display summary:
```
TDD Artifacts: [count]/[M] verified
  All RED phases failed correctly: ✓
  All GREEN phases passed: ✓
  All commits in scope: ✓
```

## Step 5: Prepare Integration Text

Generate a summary from the idea + design + plan + build artifacts:

```markdown
## [ECL-IDEA-NNNN] [Title]

**Type:** [type from idea]
**Priority:** [priority]
**Complexity:** [from design/plan]

### Summary
[Description from the idea/design]

### Changes
[List of all files changed, from TDD artifacts commit.changed_files, deduplicated]

### Test Coverage
- [M] tasks implemented with TDD
- [total tests] tests added
- Full test suite: PASS ([X] tests)

### TDD Proof
Artifacts: `docs/tdd/[idea-id]/`
```

## Step 6: Propose Integration

Present options:

```
Build verified. Integration options:

1. Create PR — prepare PR with the summary above (text only, you push)
2. Merge to current branch — direct merge
3. Keep as-is — leave for manual review (status stays needs_review)
4. Abandon — discard the work (status → abandoned)
```

### Option 1: Create PR
- Display the PR title and body text for the user to use
- Include: summary, changes list, test results, TDD proof link
- Do NOT automatically create the PR — prepare the text, user decides when to push/create
- If user confirms → help them create the PR with `gh pr create`

### Option 2: Merge
- Merge the feature branch if applicable
- If on main/master → warn and confirm

### Option 3: Keep
- Status stays "needs_review"
- "Run /eclipse-tools:finish again when ready"

### Option 4: Abandon
- Status → "abandoned"
- Artifacts preserved for reference

## Step 7: Update Mem0

Based on user choice:
- PR created or Merged → status: "done"
- Keep → status: "needs_review" (unchanged)
- Abandon → status: "abandoned"

Set `updated_at`, `last_action_at` → now.

## Step 8: Summary

```
[idea title] — [STATUS]
━━━━━━━━━━━━━━━━━━━━━━

Status: [done / needs_review / abandoned]
Tests: [X passed, Y failed, Z skipped]
TDD artifacts: docs/tdd/[idea-id]/ ([count] files)
Commits: [N] commits
[PR URL if created]

Pipeline complete. Run /eclipse-tools:next for your next task.
```
