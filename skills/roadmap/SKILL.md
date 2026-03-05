---
name: roadmap
description: >
  Generate a product roadmap with MoSCoW-prioritized features, phases, and milestones.
  Optionally includes competitor analysis via --with-competitor flag.
  Results stored in Mem0 as ECL-FEAT items (not ECL-IDEA).
disable-model-invocation: true
argument-hint: "[--with-competitor]"
---

# Eclipse Roadmap Pipeline

You orchestrate the roadmap generation workflow in up to 3 phases.

## Phase 1: Discovery

1. Dispatch `roadmap-discovery` agent with the project root path
2. Agent analyzes project files → returns discovery JSON
3. Store/update ECL-CTX in Mem0 with tech stack info from discovery
4. If agent returns invalid JSON → log error, STOP ("Discovery failed, cannot generate roadmap")

## Phase 2: Competitor Analysis (optional)

**Only if** $ARGUMENTS contains "--with-competitor":

1. Reuse the `competitor-researcher` agent (same agent as `/eclipse-tools:competitor` — no duplication)
2. Pass discovery context to the agent (project name, type, audience)
3. Pass list of already-analyzed competitors from Mem0 (if any)
4. Agent returns competitor JSON
5. Store ECL-COMP items in Mem0 (same mapping logic as competitor skill)
6. Generate derived ECL-IDEA items from high-opportunity gaps (same logic as competitor skill)

**If agent fails or returns invalid JSON:** log warning, continue to Phase 3 without competitor data. Never block the roadmap.

**If --with-competitor NOT specified:** skip this phase entirely.

## Phase 3: Feature Generation

1. Dispatch `roadmap-features` agent with:
   - Discovery JSON from Phase 1
   - Competitor JSON from Phase 2 (if available, otherwise omit)
   - List of existing Mem0 items for this project_id (for dedup)

2. Prompt:

```
Generate a feature roadmap for this project.

Discovery context:
[discovery JSON]

[If available:]
Competitor analysis:
[competitor JSON]

Existing items (avoid duplicates):
[Mem0 summary: fingerprints + ids + status + titles]

Return a single JSON object with vision, phases, milestones, and features.
Every feature must have testable acceptance_criteria.
```

3. Validate agent output:
   - Must be valid JSON
   - Must have `phases[]` and `features[]`
   - Must have at least 3 features
   - If invalid → log error, display what we have, suggest re-running

## Phase 4: Persist

For each feature in agent output:

1. Generate `ECL-FEAT-{NNNN}` (sequential from highest existing)
2. Map agent's local IDs (feature-1 → ECL-FEAT-0001)
3. Store with full schema:
   ```json
   {
     "schema_version": 2,
     "project_id": "[project_id]",
     "id": "ECL-FEAT-0001",
     "fingerprint": "sha256(normalize(title))[:16]",
     "title": "[from agent]",
     "description": "[from agent]",
     "type": "roadmap_feature",
     "source": "roadmap",
     "status": "draft",
     "priority": "[must/should/could/wont → mapped to critical/high/medium/low]",
     "complexity": "[from agent]",
     "impact": "[from agent]",
     "phase": "[phase_id]",
     "milestone": "[milestone_id]",
     "dependencies": "[mapped to ECL-FEAT-* IDs]",
     "acceptance_criteria": "[from agent, must be testable]",
     "competitor_insight_ids": "[mapped to ECL-COMP-* if applicable]",
     "evidence": [
       {
         "type": "discovery",
         "observation": "[why this feature matters, from discovery context]"
       }
     ],
     "derived_from": null,
     "design_ref": null,
     "plan_ref": null,
     "created_at": "[now]",
     "updated_at": "[now]",
     "last_action_at": "[now]"
   }
   ```

4. Run deduplication before each write (exact fingerprint + Jaccard)
5. Priority mapping: must→critical, should→high, could→medium, wont→low

## Phase 5: Display

```
Roadmap Generated — [project_name]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Vision: [one_liner from discovery]
Audience: [primary_persona]
Competitor insights: [yes/no + count if yes]

Phase 1: Foundation / MVP
  Milestone: [title]
    MUST  [ECL-FEAT-0001] [feature title]
    MUST  [ECL-FEAT-0002] [feature title]

Phase 2: Enhancement
  Milestone: [title]
    SHOULD [ECL-FEAT-0003] [feature title]
    SHOULD [ECL-FEAT-0004] [feature title]

Phase 3: Growth
  Milestone: [title]
    COULD  [ECL-FEAT-0005] [feature title]

Phase 4: Vision
    WONT   [ECL-FEAT-0006] [feature title] (future)

Total: [N] features ([M] MUST, [O] SHOULD, [P] COULD, [Q] WONT)
Saved as drafts in Mem0. Use /eclipse-tools:next to triage.
```

## Mode Degrade (sans Mem0)

If Mem0 unavailable:
1. Run discovery and feature generation normally
2. Display the full roadmap to user
3. Warn: "Mem0 unavailable — roadmap not persisted."
4. Do NOT attempt any Mem0 writes

## Important: ECL-FEAT vs ECL-IDEA

- Features from roadmap are stored as **ECL-FEAT** (not ECL-IDEA)
- They appear in `/eclipse-tools:next` dashboard alongside ECL-IDEA items
- They follow the same lifecycle (draft → accepted → designed → planned → building → done)
- The `next` skill displays both types in the same dashboard

## TODO

- Ensure `next` skill handles ECL-FEAT items in its dashboard queries (not just ECL-IDEA)
