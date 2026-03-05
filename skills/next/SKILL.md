---
name: next
description: >
  Daily dashboard showing all ideas, features, and their status from Mem0.
  Read-only by default — only writes to Mem0 when user explicitly chooses an action.
  Use when starting a work session, checking progress, or deciding what to work on next.
disable-model-invocation: true
---

# Eclipse Dashboard

You are the daily entry point for eclipse-tools. Show the user what's in their pipeline and suggest the next action.

## CRITICAL: Idempotence

This skill is **READ-ONLY**. You MUST NOT write to Mem0 unless the user explicitly chooses an action (accept, reject, defer, continue, etc.). Displaying the dashboard = zero Mem0 writes. This is non-negotiable.

## Step 1: Check Mem0 Availability

1. Attempt to search Mem0 for project context
2. If Mem0 unavailable:
   - Display: "Mem0 not configured. Set MEM0_API_KEY environment variable and restart."
   - Display: "Get your key at https://app.mem0.ai/"
   - STOP here. Do not display an empty dashboard.
3. If Mem0 available: continue

## Step 2: Calculate Project ID

Use the remember skill's algorithm:
1. `git remote get-url origin 2>/dev/null`
2. Check for monorepo (pnpm-workspace.yaml / package.json workspaces)
3. Hash accordingly

Search Mem0 for ALL items tagged with this project_id.

## Step 3: Display Dashboard

Organize items by status in this exact order. Only show sections that have items.

```
Eclipse Dashboard — [project_name or project_id]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

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

── X done | Y deferred | Z rejected | W abandoned ──
```

Priority sort order: critical > high > medium > low > null.
Within same priority: oldest first (created_at ASC).

If NO items found at all:
```
Eclipse Dashboard — [project_id]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Pipeline vide. Aucune idea trouvee.
→ Lancer /eclipse-tools:ideation pour analyser le projet
```

## Step 4: Suggest ONE Action

Follow this priority chain strictly. Pick the FIRST that matches:

1. `building` exists → "Continuer [id]: [title] — Task N/M"
2. `blocked` exists → "Debloquer [id]: [title] — [blocked_reason]"
3. `needs_review` exists → "Valider [id]: [title] — build termine"
4. `planned` exists (priority critical or high) → "Lancer le build de [id]: [title]"
5. `designed` exists → "Creer le plan pour [id]: [title]"
6. `accepted` exists → "Lancer le design de [id]: [title]"
7. `draft` exists → "Trier [N] nouvelles ideas"
8. Nothing → "Pipeline vide. Lancer /eclipse-tools:ideation ?"

### Non-Ambiguity Rule

When the suggestion involves choosing an item and multiple candidates exist:
- **Exactly 1 candidate** → use it directly, display which one
- **2+ candidates** → list top 3, ask user to choose. NEVER auto-select.

Example with multiple candidates:
```
Plusieurs items planned :
  1. ECL-IDEA-0003 — Cache Redis (HIGH, planned 2026-03-06)
  2. ECL-IDEA-0007 — Auth JWT (HIGH, planned 2026-03-05)
  3. ECL-IDEA-0011 — Rate limit (MEDIUM, planned 2026-03-06)
Lequel ?
```

## Step 5: Wait for User Choice

The user can respond with:
- An ID (e.g., "ECL-IDEA-0003") → launch the appropriate next action for that item
- A number (e.g., "1") → same, using the dashboard numbering
- "trier" → enter rapid triage mode (see below)
- "ideation" → inform user to run `/eclipse-tools:ideation`
- "roadmap" → inform user to run `/eclipse-tools:roadmap`
- "competitor" → inform user to run `/eclipse-tools:competitor`
- "rien" / "quit" / "q" → exit. ZERO Mem0 writes.

## Rapid Triage Mode

When user chooses "trier", present each draft one by one:

```
Mode tri rapide — N ideas a trier
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[1/N] ECL-IDEA-0013: "Migrer Jest vers Vitest"
      Type: code_quality | Effort: small | Evidence: 3 files
      → (A)ccept [priority]  (D)efer "reason"  (R)eject "reason"  (S)kip
```

Interaction rules:
- `A` → accept with priority medium (default)
- `A high` / `A critical` / `A low` → accept with specified priority
- `R "raison"` → reject. Reason is REQUIRED. If user says just `R`, ask for reason.
- `D "raison"` → defer. Reason is REQUIRED. If user says just `D`, ask for reason.
- `S` → skip, item stays as draft for later

**Mem0 writes during triage:** Write each decision to Mem0 immediately after the user responds for that item. This is the ONE exception to "no writes" — the user has explicitly chosen an action.

After all items triaged (or user says "stop"):
```
Tri termine: X accepted, Y rejected, Z deferred, W skipped
```

## Error Handling

- Mem0 read failure → display warning, show what data is available, continue
- Empty project → guide user to run ideation
- Corrupted item in Mem0 → skip it, log warning, continue with other items
