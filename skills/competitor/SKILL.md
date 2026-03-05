---
name: competitor
description: >
  Analyze competitors for the current project. Uses WebSearch to find alternatives,
  user complaints, and market gaps. Results stored in Mem0 with full source traceability.
  Generates derived ideas (ECL-IDEA) from high-opportunity gaps.
disable-model-invocation: true
argument-hint: "[sector or project description]"
---

# Eclipse Competitor Analysis

You orchestrate a competitor research workflow with full source traceability.

## Step 1: Load Project Context

1. Search Mem0 for ECL-CTX with this project_id
2. If found: use project_name, tech_stack, target audience as context
3. If not found: detect from project files (package.json, README, etc.)
4. If $ARGUMENTS provided: use as sector/description override
5. If no context available at all: ask user ONE question: "Describe your project in one sentence"

## Step 2: Check Existing Competitor Data

Search Mem0 for ECL-COMP items with this project_id.
Build a summary of already-analyzed competitors to pass to agent (avoid re-researching).

If Mem0 unavailable: proceed with empty context, log warning.

## Step 3: Dispatch Agent

Launch `competitor-researcher` agent with:
- Project context (name, type, audience, stack)
- List of already-analyzed competitor names (if any)
- Sector/description override (if $ARGUMENTS provided)

Prompt:

```
Research competitors for this project:
- Project: [name/description]
- Type: [project_type]
- Target audience: [audience]
- Already analyzed (skip these): [list]

[If $ARGUMENTS]: Focus on the sector: $ARGUMENTS

Return a single JSON object with competitors, market_gaps, and research_metadata.
```

## Step 4: Store in Mem0

Map agent's local IDs to global ECL-* IDs:

For each competitor in agent output:
1. Generate `ECL-COMP-{NNNN}` (sequential from highest existing)
2. Store full competitor object with:
   - `schema_version: 2`
   - `project_id`
   - `sources[]` preserved as-is (with real URLs and accessed_at)
   - `claims[]` preserved with local IDs (claim-001, etc.)
   - `market_gaps[]` if attached to this competitor
3. Build a mapping: `competitor-1 → ECL-COMP-0001`, etc.

For standalone market_gaps: store within the most relevant ECL-COMP entry.

## Step 5: Generate Derived Ideas

For each market gap with `opportunity_size: "high"`:

1. Create an ECL-IDEA with:
   ```json
   {
     "schema_version": 2,
     "project_id": "[project_id]",
     "id": "ECL-IDEA-{NNNN}",
     "title": "[suggested_feature from gap]",
     "description": "[gap description + opportunity]",
     "type": "code_improvement",
     "source": "competitor",
     "status": "draft",
     "priority": null,
     "derived_from": {
       "type": "competitor_claim",
       "competitor_insight_id": "ECL-COMP-{mapped}",
       "claim_id": "claim-{from agent}",
       "gap_id": "gap-{from agent}"
     },
     "evidence": [
       {
         "type": "url",
         "url": "[source url from claim]",
         "snippet": "[source snippet]",
         "observation": "[why this is an opportunity]"
       }
     ]
   }
   ```

2. Run deduplication BEFORE writing:
   - Exact fingerprint against existing Mem0 items
   - Jaccard near-duplicate check
   - If duplicate → skip, log
   - If near-dup → flag for user decision

## Step 6: Display Results

```
Competitor Analysis Complete
━━━━━━━━━━━━━━━━━━━━━━━━━━━

Competitors analyzed: [N]
  [ECL-COMP-0001] [name] — [relevance] — [claims count] pain points
  [ECL-COMP-0002] [name] — [relevance] — [claims count] pain points
  ...

Pain points found: [total claims]
Market gaps identified: [total gaps]
Derived ideas generated: [N] (saved as drafts)

Top Gaps:
  1. [gap description] — opportunity: [high/medium] — affects [N] competitors
  2. [gap description] — opportunity: [high/medium] — affects [N] competitors

Sources consulted: [count]
Limitations: [list if any]
```

If near-duplicates flagged:
```
Near-duplicates detected:
  ECL-IDEA-{new} "[title]" ~ ECL-IDEA-{existing} "[title]" (Jaccard: 0.7)
  → Merge / Keep both / Skip ?
```

Then: "Use /eclipse-tools:next to triage derived ideas, or /eclipse-tools:roadmap --with-competitor to generate a roadmap."

## Mode Degrade (sans Mem0)

If Mem0 unavailable:
1. Display the full analysis to user (console output)
2. Display derived ideas inline
3. Warn: "Mem0 unavailable — analysis not persisted. Set MEM0_API_KEY to enable."
4. Do NOT attempt any Mem0 writes
