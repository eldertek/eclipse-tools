---
name: remember
description: >
  Persist project insights, decisions, and workflow state into Mem0.
  Auto-invoked after ideation, roadmap, competitor, design, plan, build, finish, and bug-hunter workflows.
  Handles project_id scoping, schema versioning, deduplication, and migration.
user-invocable: false
---

# Eclipse Memory Persistence

You are the memory layer for eclipse-tools. You persist structured data into Mem0, scoped by project.

## Mem0 Availability Check

Before ANY Mem0 operation:
1. Attempt a Mem0 read operation (search for project context)
2. If Mem0 is unavailable (MEM0_API_KEY missing or connection error):
   - Log to user: "Mem0 unavailable — results will not be persisted"
   - Return gracefully, NEVER block the calling workflow
   - NEVER write the API key to any file, log, or output
3. If Mem0 is available, proceed normally

## Project ID Calculation

Calculate project_id at first use:

1. Run: `git remote get-url origin 2>/dev/null`
2. Check for monorepo:
   - Look for `pnpm-workspace.yaml` or `workspaces` key in root `package.json`
   - If monorepo: identify current workspace relative path
3. Calculate:
   - **Monorepo:** `sha256(remote_url + "/" + workspace_relative_path)[:12]`
   - **Standard repo with remote:** `sha256(remote_url)[:12]`
   - **No remote:** `sha256(absolute_repo_path)[:12]`
4. Cache the project_id for the duration of the session

## Schema Version

Current schema_version: **2**

When reading an item from Mem0:
- If `schema_version` is absent or `1`:
  - Add `fingerprint` = sha256(normalize(title))[:16]
  - Add `evidence: []` if absent
  - Add `last_action_at = updated_at` if absent
  - Set `schema_version: 2`
  - Write the migrated item back to Mem0

## Deduplication (2-pass)

Before writing any idea/feature to Mem0:

**Pass 1 — Exact fingerprint:**
```
fingerprint = sha256(normalize(title))[:16]
normalize(text) = text.toLowerCase().trim().replaceAll(/\s+/g, ' ')
```
- Search Mem0 for same `project_id` + same `fingerprint`
- If match exists (any status including rejected/done) → skip, log "duplicate detected: [existing_id]"

**Pass 2 — Near-duplicate (Jaccard token overlap):**
```
tokens_new = set(normalize(new_title).split(' '))
For each existing item:
  tokens_existing = set(normalize(existing.title).split(' '))
  jaccard = |intersection(tokens_new, tokens_existing)| / |union(tokens_new, tokens_existing)|
  If jaccard > 0.6 → flag as near-duplicate
```
- If near-duplicate found → ask user: "Possible duplicate of [existing_id]: [existing_title]. Merge / Keep both / Skip?"
- NEVER auto-block creation on near-duplicate — always ask

## ID Generation

Format: `ECL-{TYPE}-{NNNN}`
- TYPE: `IDEA`, `COMP`, `FEAT`, `CTX`, `BUG`
- NNNN: zero-padded 4 digits, sequential
- To find next ID: search Mem0 for all items with same project_id + type prefix, find highest NNNN, increment by 1
- If no existing items: start at 0001

## Writing to Mem0

Use the Mem0 MCP tools. Each item = one memory entry.
Tag every entry with metadata for retrieval:
- `project_id`: the calculated project ID
- `eclipse_type`: IDEA | COMP | FEAT | CTX | BUG
- `eclipse_id`: the full ECL-* ID
- `schema_version`: 2

## Reading from Mem0

Search by `project_id` tag. Optional additional filters:
- By type: filter on `eclipse_type` tag
- By status: filter on `status` field in content
- By ID: filter on `eclipse_id` tag

Always check `schema_version` on read and migrate if needed.

## Error Handling

- Mem0 connection timeout → log warning, return empty results, never block
- Mem0 write failure → log warning, inform user "Failed to persist [id]", continue workflow
- Invalid JSON from Mem0 → log warning, skip item, continue
- NEVER throw an error that would block the calling skill
