---
name: design
description: >
  Brainstorm and design a solution for an accepted idea or feature.
  Explores the codebase, asks clarifying questions, proposes approaches, produces a design document
  with scope, acceptance criteria, and testing strategy. Updates Mem0 status to "designed".
disable-model-invocation: true
argument-hint: "[ECL-IDEA-NNNN or ECL-FEAT-NNNN or text description]"
---

# Eclipse Design

You brainstorm and design an implementation approach for a chosen idea or feature.

## Step 1: Resolve Target Item

Accepts both ECL-IDEA and ECL-FEAT items. They share the same lifecycle.

1. If $ARGUMENTS is an ECL-* ID → fetch from Mem0 directly
2. If $ARGUMENTS is text → search Mem0 for matching items:
   - Filter by project_id + status in (accepted, draft) — exclude rejected, done, abandoned, deferred by default
   - Score by token similarity
   - If 1 match > 80% → use directly
   - If 2-3 matches > 50% → propose top 3, ask user to choose
   - If 0 matches → "No matching idea found. Create one with /eclipse-tools:ideation?"
3. If no $ARGUMENTS → smart default:
   - Find first `accepted` item by priority (critical > high > medium > low), then oldest
   - If 1 candidate → use it, announce which
   - If 2+ candidates → list top 3, ask user to choose (non-ambiguity rule)
   - If 0 → "No accepted ideas. Run /eclipse-tools:next to triage drafts, or /eclipse-tools:ideation to generate ideas."

## Step 2: Explore Codebase

1. Read the item's evidence files (from `evidence[].file_path`)
2. Read surrounding code to understand context, patterns, dependencies
3. Identify existing patterns the design should follow (naming, architecture, error handling)
4. Check for related items in Mem0 that might create conflicts or synergies

## Step 3: Ask Clarifying Questions

- One question at a time
- Prefer multiple choice when possible (2-4 options)
- Focus on: purpose, constraints, success criteria, trade-offs
- Do NOT ask if you can infer from code or evidence — only ask for genuinely ambiguous decisions

## Step 4: Propose 2-3 Approaches

For each approach:
- Description (2-3 sentences)
- Trade-offs (pros/cons)
- Affected files (exact paths)
- Estimated effort (small/medium/large)
- Lead with your recommendation and explain why

## Step 5: Write Design Document

After user validates an approach, save to `docs/plans/YYYY-MM-DD-<topic>-design.md`.

The design document MUST include these sections:

### Required Sections

```markdown
# [Title] Design

**Item:** [ECL-IDEA-NNNN or ECL-FEAT-NNNN]
**Date:** YYYY-MM-DD
**Status:** designed

## Goal
[One sentence: what problem does this solve?]

## Approach
[Selected approach description, 2-5 sentences]

## Scope
### Files to create
- `exact/path/to/new-file.ts` — [purpose]

### Files to modify
- `exact/path/to/existing.ts:NN-MM` — [what changes]

### Files explicitly OUT of scope
- `path/to/untouched.ts` — [why not touched]

## Acceptance Criteria
[Testable, verifiable criteria — each must be pass/fail]
1. [Criterion 1]
2. [Criterion 2]
3. ...

## Testing Strategy
### Unit tests
- [What to test, which test file]

### Integration tests (if applicable)
- [What to test end-to-end]

### Manual verification (if applicable)
- [Steps to manually verify]

## Error Handling
[How errors are handled in the new code]

## Dependencies
[External packages, internal modules, ECL-* items that must be done first]
```

### Anti-Bloat Rules

- Do NOT include architecture diagrams for small changes
- Do NOT add sections that would be empty — omit them
- Scale the document to the complexity of the change:
  - Small (1-3 files): 1-2 pages
  - Medium (3-10 files): 2-4 pages
  - Large (10+ files): full document

## Step 6: Update Mem0

- Status → "designed"
- `design_ref` → path to design document
- `updated_at`, `last_action_at` → now

## Step 7: Chain

"Design saved to `[path]`. Run /eclipse-tools:plan to create the implementation plan?"

## Error Handling

- Item not found in Mem0 → guide user to /eclipse-tools:ideation or /eclipse-tools:next
- Item already designed → ask: "This item already has a design. Overwrite or view existing?"
- Mem0 write failure → display warning, design doc is still saved locally
