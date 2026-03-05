# Eclipse Tools — Design Document

> Plugin Claude Code pour l'analyse, l'ideation, la planification et l'execution autonome de projets logiciels.

**Date :** 2026-03-05
**Version :** 1.0.0
**Architecture :** Skills orchestrateurs + Agents workers + Mem0 (source de verite)

---

## 1. Principes non-negociables

1. **Mem0 = source de verite** — Tout est scope par `project_id`, versionne, dedup par fingerprint.
2. **TDD enforceable** — Preuve de test FAIL avant impl, preuve de test PASS apres. Reviewers refusent si pas de trace.
3. **next = idempotent** — Lecture seule. Zero ecriture Mem0 sauf action explicite de l'utilisateur.
4. **Tracabilite stricte** — Chaque idea a `evidence[]`, chaque claim competitor a `sources[]`.
5. **Scope contract en build** — `allowed_paths` + "definition of done" par task. 1 commit atomique. Zero refactor hors scope.
6. **Smart defaults** — Chaque skill fonctionne sans argument en reprenant le dernier item pertinent.
7. **Sorties humain + machine** — Resume lisible pour l'utilisateur, JSON strict pour Mem0.
8. **Fallbacks gracieux** — Playwright tente si app running, sinon audit statique. Context7 confirme, ne genere pas.

---

## 2. Scoping Mem0

### 2.1 Project ID

Identifiant stable par projet, calcule a l'initialisation :

```
project_id = sha256(git_remote_url || absolute_repo_path + package_name)[:12]
```

Exemples :
- `a3f7c2e91b4d` (repo avec remote)
- `8e2f1a6c3d9b` (repo local sans remote)

Toutes les operations Mem0 sont prefixees par `project_id`. Aucun cross-project.

### 2.2 Schema versioning

Chaque item stocke dans Mem0 a un champ `schema_version`.

```
schema_version: 2  →  format actuel
schema_version: 1  →  ancien format (migration automatique)
absent             →  traiter comme v1, migrer au prochain write
```

Migration v1 → v2 :
- Ajouter `fingerprint` (recalculer depuis title normalise)
- Ajouter `evidence: []` si absent
- Ajouter `last_action_at = updated_at`
- Ajouter `schema_version: 2`

---

## 3. Schema Mem0

### 3.1 Idea

```json
{
  "schema_version": 2,
  "project_id": "a3f7c2e91b4d",
  "id": "ECL-IDEA-0001",
  "fingerprint": "sha256_of_normalized_title_first_16_chars",
  "title": "Ajouter caching Redis sur les API endpoints",
  "description": "Les endpoints /api/users et /api/products font des queries SQL a chaque requete. Un cache Redis avec TTL 5min reduirait la charge DB de ~60%.",
  "type": "performance_optimization",
  "source": "ideation",
  "status": "draft",
  "priority": null,
  "evidence": [
    {
      "type": "file",
      "path": "src/api/users.ts",
      "line": 42,
      "snippet": "const users = await db.query('SELECT * FROM users')",
      "observation": "Query SQL sans cache, executee a chaque requete"
    },
    {
      "type": "metric",
      "value": "~200ms per request",
      "observation": "Latence mesuree sur endpoint /api/users"
    }
  ],
  "rejection_reason": null,
  "defer_reason": null,
  "blocked_reason": null,
  "waiting_on": null,
  "design_ref": null,
  "plan_ref": null,
  "derived_from": null,
  "created_at": "2026-03-05T10:30:00Z",
  "updated_at": "2026-03-05T10:30:00Z",
  "last_action_at": "2026-03-05T10:30:00Z"
}
```

### 3.2 Status lifecycle

```
                    ┌─────────┐
                    │  draft  │
                    └────┬────┘
                         │ tri (accept/reject/defer)
              ┌──────────┼──────────┐
              v          v          v
         ┌────────┐ ┌────────┐ ┌────────┐
         │accepted│ │rejected│ │deferred│
         └───┬────┘ └────────┘ └────────┘
             │ design
             v
        ┌────────┐
        │designed│
        └───┬────┘
            │ plan
            v
        ┌───────┐
        │planned│
        └───┬───┘
            │ build
            v
       ┌────────┐     ┌───────┐
       │building│────>│blocked│
       └───┬────┘     └───────┘
           │ build complete
           v
     ┌────────────┐
     │needs_review│
     └─────┬──────┘
           │ review pass
           v
        ┌────┐
        │done│
        └────┘

  (any status) ──> abandoned (arret definitif, different de rejected)
```

**Definitions :**

| Status | Signification |
|--------|---------------|
| `draft` | Generee par ideation/roadmap, non triee |
| `accepted` | Utilisateur a valide, prete pour design |
| `rejected` | Utilisateur a refuse (avec `rejection_reason`) |
| `deferred` | Report (avec `defer_reason`) |
| `designed` | Design doc produit (`design_ref` rempli) |
| `planned` | Plan TDD produit (`plan_ref` rempli) |
| `building` | Execution en cours par subagents |
| `blocked` | Bloque (`blocked_reason` + `waiting_on`) |
| `needs_review` | Build termine, validation fonctionnelle pas faite |
| `done` | Termine, merge, deploye |
| `abandoned` | Arret definitif post-implementation (different de rejected pre-implementation) |

### 3.3 Competitor Insight

```json
{
  "schema_version": 2,
  "project_id": "a3f7c2e91b4d",
  "id": "ECL-COMP-0001",
  "competitor_name": "Linear",
  "competitor_url": "https://linear.app",
  "relevance": "high",
  "sources": [
    {
      "title": "Linear complaints on Reddit",
      "url": "https://reddit.com/r/projectmanagement/comments/abc123",
      "snippet": "Linear's search is terrible, can never find old issues",
      "accessed_at": "2026-03-05T11:00:00Z"
    }
  ],
  "claims": [
    {
      "id": "claim-001",
      "claim": "Linear's search ne supporte pas la recherche full-text dans les commentaires",
      "severity": "high",
      "source_ids": ["https://reddit.com/r/projectmanagement/comments/abc123"],
      "opportunity": "Implementer une recherche full-text couvrant issues + commentaires + attachments"
    }
  ],
  "market_gaps": [
    {
      "id": "gap-001",
      "description": "Aucun outil ne propose de recherche semantique dans les issues",
      "opportunity_size": "high",
      "suggested_feature": "Recherche semantique AI-powered",
      "affected_competitors": ["Linear", "Jira", "Asana"]
    }
  ],
  "created_at": "2026-03-05T11:00:00Z",
  "updated_at": "2026-03-05T11:00:00Z"
}
```

### 3.4 Idea derivee de competitor

```json
{
  "schema_version": 2,
  "project_id": "a3f7c2e91b4d",
  "id": "ECL-IDEA-0042",
  "fingerprint": "...",
  "title": "Recherche full-text dans les issues et commentaires",
  "type": "code_improvement",
  "source": "competitor",
  "status": "draft",
  "derived_from": {
    "type": "competitor_claim",
    "competitor_insight_id": "ECL-COMP-0001",
    "claim_id": "claim-001",
    "gap_id": "gap-001"
  },
  "evidence": [
    {
      "type": "url",
      "url": "https://reddit.com/r/projectmanagement/comments/abc123",
      "snippet": "Linear's search is terrible...",
      "observation": "Pain point confirme par 47 upvotes"
    }
  ]
}
```

### 3.5 Roadmap Feature

```json
{
  "schema_version": 2,
  "project_id": "a3f7c2e91b4d",
  "id": "ECL-FEAT-0001",
  "fingerprint": "...",
  "title": "Authentication JWT avec refresh tokens",
  "description": "...",
  "priority": "must",
  "complexity": "medium",
  "impact": "high",
  "phase": "phase-1-mvp",
  "milestone": "milestone-1-1-auth",
  "dependencies": [],
  "acceptance_criteria": [
    "Login retourne access + refresh token",
    "Refresh endpoint renouvelle l'access token",
    "Tokens expires sont rejetes avec 401"
  ],
  "status": "draft",
  "evidence": [
    {
      "type": "file",
      "path": "src/middleware/auth.ts",
      "observation": "Auth actuelle = basic auth sans expiration"
    }
  ],
  "derived_from": null,
  "design_ref": null,
  "plan_ref": null,
  "created_at": "2026-03-05T12:00:00Z",
  "updated_at": "2026-03-05T12:00:00Z",
  "last_action_at": "2026-03-05T12:00:00Z"
}
```

### 3.6 Project Context (stocke une seule fois, mis a jour)

```json
{
  "schema_version": 2,
  "project_id": "a3f7c2e91b4d",
  "id": "ECL-CTX-0001",
  "project_name": "my-saas-app",
  "tech_stack": {
    "language": "TypeScript",
    "framework": "Next.js 14",
    "test_runner": "vitest",
    "test_command": "npx vitest run",
    "package_manager": "pnpm",
    "database": "PostgreSQL + Prisma"
  },
  "detected_patterns": {
    "api_style": "REST + tRPC",
    "component_style": "React Server Components",
    "state_management": "Zustand"
  },
  "last_ideation_at": "2026-03-05T10:30:00Z",
  "last_roadmap_at": null,
  "last_competitor_at": null,
  "updated_at": "2026-03-05T10:30:00Z"
}
```

---

## 4. Deduplication

Avant d'ecrire une idea dans Mem0, calculer le fingerprint :

```
fingerprint = sha256(normalize(title))[:16]

normalize(text) = text.toLowerCase().trim().replace(/\s+/g, ' ')
```

Algorithme :
1. Calculer fingerprint de la nouvelle idea
2. Chercher dans Mem0 : meme `project_id` + meme `fingerprint`
3. Si match exact → skip (log "duplicate detected")
4. Si aucun match → creer
5. Les ideas `rejected` et `done` avec meme fingerprint bloquent aussi la creation

---

## 5. Regles du dashboard `/eclipse-tools:next`

### 5.1 Idempotence

`next` est **lecture seule**. Il lit Mem0, affiche un dashboard, propose des actions. Aucune ecriture Mem0 tant que l'utilisateur n'a pas choisi.

### 5.2 Ordre d'affichage

```
Section 1: EN COURS (priorite absolue)
  building  → tries par last_action_at DESC
  blocked   → tries par last_action_at DESC (avec raison affichee)

Section 2: A VALIDER
  needs_review → tries par last_action_at DESC

Section 3: PRETES
  planned   → tries par priority (critical > high > medium > low), puis created_at ASC
  designed  → meme tri
  accepted  → meme tri

Section 4: A TRIER
  draft     → tries par created_at DESC (plus recentes en premier)

Section 5: STATS (une ligne)
  "X done | Y deferred | Z rejected | W abandoned"
```

### 5.3 Suggestions contextuelles

Apres le dashboard, `next` propose **une seule action recommandee** :

```
SI building exists:
  → "Continuer [id]: [title] — Task N/M"

SINON SI needs_review exists:
  → "Valider [id]: [title] — build termine, en attente de review"

SINON SI planned exists (priority critical ou high):
  → "Lancer le build de [id]: [title]"

SINON SI designed exists:
  → "Creer le plan pour [id]: [title]"

SINON SI accepted exists:
  → "Lancer le design de [id]: [title]"

SINON SI draft exists:
  → "Trier N nouvelles ideas"

SINON:
  → "Aucune idea en attente. Lancer /eclipse-tools:ideation ?"
```

### 5.4 Actions disponibles

L'utilisateur peut repondre :
- Un chiffre/id → lance l'action sur cet item
- "trier" → entre en mode tri des drafts
- "ideation" → lance une nouvelle ideation
- "roadmap" → lance le roadmap
- "rien" → quitte sans ecriture

---

## 6. TDD — Garde-fous obligatoires

### 6.1 Dans `plan`

Le skill `plan` doit :

1. **Detecter le test runner** depuis `ECL-CTX` dans Mem0 (ou le detecter : package.json, pyproject.toml, Cargo.toml)
2. **Produire des commandes exactes** adaptees a la stack :
   - `npx vitest run src/api/__tests__/cache.test.ts` (pas juste "run tests")
   - `pytest tests/api/test_cache.py::test_cache_hit -v`
   - `cargo test cache::tests::test_cache_hit -- --nocapture`
3. **Chaque task contient** :
   - `test_file`: chemin exact du fichier test
   - `test_command`: commande exacte
   - `impl_files`: chemins exacts des fichiers a modifier
   - `allowed_paths`: liste blanche des fichiers touchables
   - `definition_of_done`: criteres precis
   - `commit_message`: message de commit standard

### 6.2 Dans `build`

Pour chaque task, le subagent `implementer` doit :

```
Phase 1 — RED
  1. Ecrire le test
  2. Executer test_command
  3. Capturer l'output
  4. VERIFIER que le test FAIL
  → Si le test PASS deja → STOP, signaler au reviewer

Phase 2 — GREEN
  5. Ecrire l'implementation minimale
  6. Executer test_command
  7. Capturer l'output
  8. VERIFIER que le test PASS
  → Si le test FAIL encore → iterer (max 3 tentatives)

Phase 3 — COMMIT
  9. git add [allowed_paths seulement]
  10. git commit -m "[commit_message]"
```

### 6.3 TDD Reviewer (integre dans spec-reviewer)

Le `spec-reviewer` a une checklist TDD obligatoire :

```
[ ] Au moins 1 fichier test modifie/cree dans ce commit
[ ] Le test a ete ecrit AVANT l'implementation (verifiable via git diff order ou trace)
[ ] Trace de test run FAIL presente (output capture)
[ ] Trace de test run PASS presente (output capture)
[ ] Aucun fichier hors allowed_paths modifie
[ ] commit_message conforme au standard
[ ] definition_of_done satisfait
```

**Si un check echoue → REFUSE.** L'implementer doit corriger avant re-review.

---

## 7. Playwright — Fallback gracieux

L'agent `ideation-ux` suit cet algorithme :

```
1. Verifier si une app est running (check common ports: 3000, 5173, 8080, 4200)
2. SI app detectee:
   → Utiliser Playwright MCP pour naviguer, screenshoter, auditer
   → Verifier accessibilite live (ARIA, contrast, keyboard nav)
   → Capturer les screenshots comme evidence
3. SINON:
   → Log "No running app detected, falling back to static audit"
   → Audit statique du code :
     - Grep pour aria-*, role=, alt=, tabIndex
     - Analyser les CSS pour contrast ratios
     - Checker les patterns UI (loading states, error boundaries, responsive)
     - Verifier les imports d'accessibilite (headlessui, radix, etc.)
   → Produire les ideas avec evidence[] = fichiers/lignes, pas screenshots
4. JAMAIS bloquer le pipeline ideation si Playwright echoue
```

---

## 8. Context7 — Usage strict

Context7 est utilise **uniquement pour confirmer**, jamais pour generer du contenu generique.

Regles :
1. L'agent detecte d'abord un pattern dans le code (ex: `useEffect` sans cleanup)
2. L'agent identifie la lib concernee (ex: React 19)
3. **Puis** il interroge Context7 : "React 19 useEffect cleanup best practices"
4. Si Context7 confirme le probleme → l'idea cite Context7 comme evidence
5. Si Context7 ne confirme pas → l'idea est droppee ou marquee "unconfirmed"

Chaque evidence Context7 doit inclure :
```json
{
  "type": "context7",
  "library": "react",
  "version": "19.x",
  "query": "useEffect cleanup on unmount",
  "finding": "React 19 docs confirm cleanup function required for subscriptions",
  "observation": "Le composant UserDashboard.tsx:34 manque le cleanup"
}
```

---

## 9. Scope contract en build

### 9.1 Par task

Chaque task dans le plan a :

```json
{
  "task_id": "task-003",
  "title": "Add Redis cache to /api/users endpoint",
  "allowed_paths": [
    "src/api/users.ts",
    "src/lib/cache.ts",
    "src/api/__tests__/users.cache.test.ts"
  ],
  "definition_of_done": [
    "GET /api/users retourne les donnees depuis Redis si cache hit",
    "Cache TTL = 300s configurable",
    "Cache miss → query DB → populate cache → return",
    "Test couvre hit, miss, et expiration"
  ],
  "commit_message": "feat(api): add Redis cache to /api/users endpoint"
}
```

### 9.2 Regles reviewers

Le `code-quality-reviewer` :
- **REFUSE** tout changement hors `allowed_paths`
- **REFUSE** tout refactor qui n'est pas dans `definition_of_done`
- **NE SUGGERE PAS** de refactors supplementaires
- Se limite a : bugs, security issues, et code quality dans le scope

---

## 10. Smart defaults

| Skill | Sans argument | Avec argument |
|-------|---------------|---------------|
| `ideation` | Analyse le projet courant | `ideation "focus security"` filtre les types |
| `roadmap` | Analyse le projet courant | `roadmap --with-competitor` |
| `competitor` | Utilise le contexte projet Mem0 | `competitor "SaaS project management"` |
| `next` | Dashboard complet | N/A |
| `design` | Reprend le premier `accepted` par priorite | `design ECL-IDEA-0042` ou `design "cache redis"` → match Mem0, propose 3 candidats |
| `plan` | Reprend le dernier `designed` du project_id | `plan ECL-IDEA-0042` |
| `build` | Reprend le dernier `planned` du project_id | `build ECL-IDEA-0042` |
| `finish` | Reprend le dernier `building` ou `needs_review` | `finish ECL-IDEA-0042` |

### Matching texte libre

Quand l'utilisateur passe du texte libre au lieu d'un ID :

```
1. Normaliser l'input
2. Chercher dans Mem0 : project_id + status compatible
3. Scorer par similarite (mots communs dans title + description)
4. Si 1 match > 80% → utiliser directement
5. Si 2-3 matches > 50% → proposer les 3 candidats
6. Si 0 match → "Aucune idea trouvee. Voulez-vous en creer une ?"
```

---

## 11. Sorties humain + machine

Chaque workflow produit deux outputs :

### 11.1 Resume lisible (affiche a l'utilisateur)

```
Ideation Complete — 12 ideas generees
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 # | Type         | Titre                              | Effort  | Evidence
---+--------------+------------------------------------+---------+----------
 1 | performance  | Cache Redis API endpoints          | medium  | 3 files
 2 | security     | Rate limiting sur /api/auth         | small   | 2 files
 3 | code_quality | Extract validation utils            | trivial | 5 files
 ...

Sauvegarde dans Mem0 : 12 items (3 dedup skipped)
→ Voulez-vous trier maintenant ?
```

### 11.2 JSON strict (stocke dans Mem0)

Chaque idea individuelle selon le schema section 3.1.
Le resume global n'est PAS stocke — il est recalculable depuis les items individuels.

---

## 12. Structure plugin finale

```
eclipse-tools/
├── .claude-plugin/
│   └── plugin.json
├── .mcp.json                          # Context7 + Playwright + Mem0
├── skills/
│   ├── ideation/SKILL.md              # 6 agents → drafts Mem0
│   ├── roadmap/SKILL.md               # Discovery → Features → drafts Mem0
│   ├── competitor/SKILL.md            # WebSearch → insights Mem0
│   ├── next/SKILL.md                  # Dashboard idempotent
│   ├── design/SKILL.md               # Brainstorming → design doc → designed
│   ├── plan/SKILL.md                  # Design → plan TDD strict → planned
│   ├── build/SKILL.md                # Subagent TDD + scope contract → building/done
│   ├── finish/SKILL.md               # Tests finaux, merge/PR
│   └── remember/SKILL.md             # Auto, persist Mem0
├── agents/
│   ├── ideation-code-improvements.md
│   ├── ideation-code-quality.md
│   ├── ideation-documentation.md
│   ├── ideation-performance.md
│   ├── ideation-security.md
│   ├── ideation-ux.md                 # Playwright avec fallback statique
│   ├── roadmap-discovery.md
│   ├── roadmap-features.md
│   ├── competitor-researcher.md       # Sources tracees, claims lies
│   ├── implementer.md                 # TDD strict : RED → GREEN → COMMIT
│   ├── spec-reviewer.md              # Checklist TDD obligatoire
│   └── code-quality-reviewer.md      # Scope contract enforce
├── hooks/
│   └── hooks.json
└── settings.json
```

---

## 13. MCPs

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    },
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    },
    "mem0": {
      "command": "npx",
      "args": ["-y", "mem0-mcp"],
      "env": {
        "MEM0_API_KEY": "${MEM0_API_KEY}"
      }
    }
  }
}
```

---

## 14. Flow complet

```
Jour 1:
  /eclipse-tools:ideation
    → 6 agents paralleles analysent le code
    → Context7 confirme les recommandations
    → 15 ideas generees, 3 dedup → 12 stockees en draft dans Mem0
    → "12 ideas. Trier maintenant ?"
    → Tri : 5 accepted, 4 rejected (avec raisons), 3 deferred

Jour 2:
  /eclipse-tools:next
    → Dashboard lecture seule :
      Accepted: 5 ideas (2 high, 2 medium, 1 low)
      Draft: 0
      Done: 0
    → Suggestion : "Lancer le design de ECL-IDEA-0003 (Cache Redis, HIGH)"
    → User : "oui"

  /eclipse-tools:design ECL-IDEA-0003
    → Explore le code, questions une par une
    → Propose 3 approches : Redis standalone / Redis + invalidation / In-memory LRU
    → User valide Redis standalone
    → Ecrit docs/plans/2026-03-06-redis-cache-design.md
    → Mem0 : status → designed, design_ref rempli

  /eclipse-tools:plan
    → Reprend automatiquement ECL-IDEA-0003 (dernier designed)
    → Detecte stack : TypeScript, Vitest, pnpm
    → Produit 5 tasks TDD avec :
      - test_file, test_command (npx vitest run ...)
      - impl_files, allowed_paths
      - definition_of_done, commit_message
    → Ecrit docs/plans/2026-03-06-redis-cache-plan.md
    → Mem0 : status → planned

  /eclipse-tools:build
    → Reprend automatiquement ECL-IDEA-0003 (dernier planned)
    → Mem0 : status → building
    → Pour chaque task :
      1. Dispatch implementer subagent
         → RED : ecrit test, run → FAIL (output capture)
         → GREEN : ecrit impl, run → PASS (output capture)
         → COMMIT : git add [allowed_paths] && git commit
      2. Dispatch spec-reviewer
         → Checklist TDD : test modifie? FAIL trace? PASS trace? scope ok?
         → Si echec → implementer corrige → re-review
      3. Dispatch code-quality-reviewer
         → Qualite dans le scope uniquement
         → Si echec → implementer corrige → re-review
      4. Task done
    → Mem0 : status → needs_review

  /eclipse-tools:finish
    → Reprend automatiquement ECL-IDEA-0003 (needs_review)
    → Run test suite complete
    → Verifie output PASS
    → Propose : merge / PR / cleanup
    → Mem0 : status → done

Jour 3:
  /eclipse-tools:next
    → "1 done | 4 accepted restantes. Lancer design de ECL-IDEA-0007 ?"
```

---

## 15. Durcissements operationnels

### 15.1 Onboarding Mem0

Le plugin doit fonctionner meme sans Mem0 configure.

```
AU LANCEMENT (chaque skill) :
  1. Verifier si MEM0_API_KEY est set (env var)
  2. SI absent :
     → next : fonctionne en mode "read-only local"
       (affiche "Mem0 non configure. Fonctionnement degrade.")
     → skills d'ecriture (ideation/roadmap/competitor/design/plan/build/finish) :
       → Proposer : "Mem0 non configure. Configurer maintenant ?"
       → SI oui : guider l'utilisateur vers https://app.mem0.ai/
         puis stocker la key via `claude config set`
       → SI non : fonctionner sans persistance (output console uniquement)
  3. JAMAIS ecrire la key dans :
     - Un fichier versionne (.env commite, plugin.json, etc.)
     - Mem0 lui-meme
     - Les logs ou outputs console
  4. Stockage recommande : variable d'environnement shell (.zshrc / .bashrc)
     ou settings Claude Code (scope user)
```

### 15.2 Artefacts TDD (preuve verifiable)

Chaque task executee par `build` produit un artefact local :

```
docs/tdd/<idea-id>/
  ├── task-001.json
  ├── task-002.json
  └── task-003.json
```

Schema d'un artefact TDD :

```json
{
  "task_id": "task-003",
  "idea_id": "ECL-IDEA-0003",
  "timestamp": "2026-03-06T14:32:00Z",
  "red": {
    "command": "npx vitest run src/api/__tests__/users.cache.test.ts",
    "exit_code": 1,
    "output_tail": "FAIL src/api/__tests__/users.cache.test.ts\n  x cache hit returns cached data\n  x cache miss queries DB\n\nTests: 2 failed, 2 total",
    "test_file": "src/api/__tests__/users.cache.test.ts"
  },
  "green": {
    "command": "npx vitest run src/api/__tests__/users.cache.test.ts",
    "exit_code": 0,
    "output_tail": "PASS src/api/__tests__/users.cache.test.ts\n  ✓ cache hit returns cached data (3ms)\n  ✓ cache miss queries DB (12ms)\n\nTests: 2 passed, 2 total",
    "impl_files": ["src/api/users.ts", "src/lib/cache.ts"]
  },
  "commit": {
    "sha": "a1b2c3d",
    "message": "feat(api): add Redis cache to /api/users endpoint",
    "changed_files": [
      "src/api/users.ts",
      "src/lib/cache.ts",
      "src/api/__tests__/users.cache.test.ts"
    ]
  }
}
```

**Regle absolue : pas d'artefact = spec-reviewer refuse.**

Mem0 stocke la reference `tdd_artifact_ref: "docs/tdd/ECL-IDEA-0003/task-003.json"`, pas le contenu.

### 15.3 Near-duplicate detection

En plus du fingerprint exact (section 4), ajouter une detection near-duplicate :

```
Algorithme en 2 passes :

Passe 1 — Exact (existant)
  fingerprint = sha256(normalize(title))[:16]
  Si match exact → skip

Passe 2 — Near-duplicate (nouveau)
  fingerprint2 = sha256(normalize(title + " " + type))[:16]
  Si match fingerprint2 → "possible duplicate: ECL-IDEA-0012"

  Token overlap (Jaccard) :
    tokens_new = set(normalize(title).split())
    tokens_existing = set(normalize(existing_title).split())
    jaccard = |intersection| / |union|
    Si jaccard > 0.6 → "possible duplicate: ECL-IDEA-0012"

  Action : NE PAS bloquer automatiquement.
    → Proposer : "Possible duplicata de ECL-IDEA-0012. Merger / Garder les deux / Skip ?"
```

### 15.4 Project ID monorepo

Revision du calcul de `project_id` pour supporter les monorepos :

```
SI monorepo detecte (workspaces dans package.json ou pnpm-workspace.yaml) :
  project_id = sha256(git_remote_url + "/" + workspace_relative_path)[:12]
  Stocker workspace_root dans ECL-CTX

SINON :
  project_id = sha256(git_remote_url || absolute_repo_path)[:12]

Exemples monorepo :
  apps/frontend/ → sha256("git@github.com:org/mono.git/apps/frontend")[:12]
  apps/backend/  → sha256("git@github.com:org/mono.git/apps/backend")[:12]
```

Ajout dans ECL-CTX :

```json
{
  "workspace_root": "apps/frontend",
  "monorepo": true,
  "monorepo_manager": "pnpm"
}
```

### 15.5 Playwright : detection app running amelioree

Remplacer le check naif de ports par un flow fiable :

```
1. SI ECL-CTX contient `dev_url` :
   → Utiliser directement (ex: "http://localhost:3000")
   → Tenter Playwright navigate → si OK, continuer

2. SINON, detecter depuis package.json/scripts :
   → Chercher script "dev", "start", "serve"
   → Extraire le port si present (ex: "vite --port 5173")
   → Proposer : "Lancer `pnpm dev` pour activer l'audit UX live ?"
   → SI user accepte : lancer le serveur, attendre ready, puis Playwright
   → SI user refuse : fallback statique

3. SINON (pas de script dev) :
   → Fallback statique directement
   → Log "No dev server detected, using static audit"

4. JAMAIS bloquer le pipeline si Playwright echoue
   → Catch toute erreur Playwright → fallback statique
```

Stocker `dev_url` dans ECL-CTX apres detection reussie pour les prochains runs.

### 15.6 Scope contract : anti drive-by formatting

Regles supplementaires pour les reviewers :

```
Le code-quality-reviewer REFUSE si :

1. Un fichier change sur >50 lignes sans etre dans impl_files
   (signe de reformatage automatique)

2. allowed_paths contient un dossier entier (ex: "src/")
   → Le plan doit lister des fichiers precis
   → Max 5 fichiers par task (sinon splitter la task)

3. Un fichier hors allowed_paths est modifie
   (meme si c'est "juste un import")

4. Des changements cosmetiques (whitespace, formatting, reorder imports)
   sur des fichiers qui ne sont pas dans impl_files

Exception : le fichier test lui-meme peut changer librement
(car TDD implique iteration sur le test).
```

### 15.7 Tri rapide des drafts dans next

Quand l'utilisateur choisit "trier" dans le dashboard, activer un mode tri 1-touche :

```
Mode tri rapide — 4 drafts a trier
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[1/4] ECL-IDEA-0013: "Migrer Jest vers Vitest"
      Type: code_quality | Evidence: 3 files
      → (A)ccept  (D)efer  (R)eject  (S)kip

> A high

[2/4] ECL-IDEA-0014: "Ajouter rate limiting"
      Type: security | Evidence: 2 files
      → (A)ccept  (D)efer  (R)eject  (S)kip

> R "on utilise deja un WAF"

[3/4] ECL-IDEA-0015: "Lazy loading images"
      Type: performance | Evidence: 1 file
      → (A)ccept  (D)efer  (R)eject  (S)kip

> D "post-v2"

[4/4] ECL-IDEA-0016: "Documenter l'API REST"
      Type: documentation | Evidence: 4 files
      → (A)ccept  (D)efer  (R)eject  (S)kip

> S

Tri termine : 1 accepted, 1 rejected, 1 deferred, 1 skipped
```

Regles :
- `A` ou `A high` / `A critical` / `A medium` / `A low` (defaut: medium)
- `R "raison"` (raison obligatoire)
- `D "raison"` (raison obligatoire)
- `S` = skip, reste en draft pour plus tard
- Ecriture Mem0 seulement apres que l'utilisateur confirme le tri complet
  (ou apres chaque reponse — au choix de l'utilisateur)

### 15.8 Smart defaults : regle de non-ambiguite

Quand un smart default doit choisir automatiquement un item :

```
SI exactement 1 candidat viable :
  → Utiliser directement
  → Afficher : "Reprise automatique de ECL-IDEA-0003 (seul item designed)"

SI 2+ candidats viables :
  → NE PAS auto-choisir
  → Proposer les 3 premiers :
    "Plusieurs items disponibles :
     1. ECL-IDEA-0003 — Cache Redis (HIGH, designed 2026-03-06)
     2. ECL-IDEA-0007 — Auth JWT (HIGH, designed 2026-03-05)
     3. ECL-IDEA-0011 — Rate limit (MEDIUM, designed 2026-03-06)
     Lequel ?"

SI 0 candidat :
  → "Aucun item [status] trouve. Lancer /eclipse-tools:[skill precedent] d'abord ?"
```
