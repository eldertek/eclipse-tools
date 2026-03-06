# Bug Hunter Design

**Skill:** `/eclipse-tools:bug-hunter`
**Date:** 2026-03-06
**Status:** designed

## Goal

Skill de chasse aux bugs autonome et generique. Un agent monolithique explore le code, les tests et l'UI (via agent-browser) pour trouver UN bug concret et reproductible, produit un packet de preuve, attend validation explicite, corrige, prouve le fix, commit, puis relance la chasse automatiquement. Memoire Mem0 complete (`ECL-BUG`) avec strategie "chercher la ou on a le moins cherche".

## Decisions cles

| Question | Choix |
|----------|-------|
| Scope | 100% generique, detection auto du stack |
| Types de bugs | Code + Tests + Browser (agent-browser) |
| Modele d'interaction | Hunt -> Packet -> Gate -> Fix -> Proof (1 bug a la fois) |
| Persistance | Mem0 complet: ECL-BUG avec fingerprint, dedup, lifecycle |
| Strategie de chasse | Cherche la ou il a le moins cherche (heatmap Mem0) |
| Preuves | Video pour browser, screenshots + logs pour code/tests |
| Couverture | Bugs logiques + incoherences metier + securite exploitable |
| Relation agents existants | Complementaire: consulte ECL-IDEA/ECL-BUG pour eviter doublons |
| Structure agents | Un seul agent monolithique (bug-hunter) |
| Nommage | `/eclipse-tools:bug-hunter`, agent `bug-hunter`, type `ECL-BUG` |

## Architecture

```
/eclipse-tools:bug-hunter [$ARGUMENTS]
        |
        v
   skills/bug-hunter/SKILL.md    (orchestrateur)
        |
        v
   agents/bug-hunter.md          (agent monolithique)
   Outils: Read, Grep, Glob, Bash, WebSearch
   Bash scope: test commands + agent-browser:*
        |
        v
   Mem0: ECL-BUG-NNNN
   (visible dans /eclipse-tools:next)
```

### Fichiers a creer

- `skills/bug-hunter/SKILL.md` -- skill orchestrateur
- `agents/bug-hunter.md` -- agent monolithique

### Fichiers a modifier

- `skills/remember/SKILL.md` -- ajouter type BUG dans ID Generation
- `skills/next/SKILL.md` -- ajouter ECL-BUG dans le dashboard

## Lifecycle ECL-BUG

```
found -> validated -> fixing -> fixed -> verified -> done
  |         |          |        |         |
  +-> rejected    +-> blocked  +-> wont_fix
                      |
                      +-> fixing (retry)
```

| Status | Transition | Declencheur |
|--------|-----------|-------------|
| `found` | Agent produit le bug packet | Automatique |
| `validated` | User dit "je valide ce bug" | Explicite |
| `rejected` | User dit "faux positif" | Explicite |
| `fixing` | Agent commence la correction | Automatique apres validation |
| `blocked` | Fix echoue apres 3 tentatives | Automatique |
| `fixed` | Tests passent, code corrige | Automatique |
| `verified` | Preuve post-fix produite | Automatique |
| `done` | User dit "je valide le fix" -> commit | Explicite |
| `wont_fix` | User reconnait le bug mais ne veut pas corriger | Explicite |

Gates explicites:
1. `found -> validated` : "je valide ce bug"
2. `verified -> done` : "je valide le fix" (declenche commit + relance)

## Schema ECL-BUG (Mem0)

```json
{
  "schema_version": 2,
  "project_id": "[calculated]",
  "id": "ECL-BUG-0001",
  "fingerprint": "sha256(normalize(title))[:16]",
  "title": "Short bug title",
  "description": "What's wrong and why",
  "severity": "critical|high|medium|low",
  "category": "logic|security|race_condition|edge_case|inconsistency|ui|validation|error_handling",
  "status": "found",
  "cause": {
    "file": "src/api/checkout.ts",
    "line": 42,
    "explanation": "Why this is the root cause"
  },
  "repro_steps": [
    "Step 1: ...",
    "Step 2: ..."
  ],
  "expected": "What should happen",
  "observed": "What actually happens",
  "impact": "Who is blocked / what data is wrong",
  "discovery_layer": "code|tests|browser",
  "discovery_zone": "src/api/",
  "evidence": [
    {
      "type": "file|screenshot|video|test_output|log|network",
      "path": "path or URL",
      "snippet": "relevant excerpt",
      "observation": "what this proves"
    }
  ],
  "fix": {
    "files_changed": [],
    "commit_sha": null,
    "commit_message": null
  },
  "hunt_session": {
    "started_at": "ISO8601",
    "zones_explored": ["src/api/", "src/auth/"],
    "layers_used": ["code", "tests", "browser"]
  },
  "created_at": "ISO8601",
  "updated_at": "ISO8601",
  "last_action_at": "ISO8601"
}
```

## Workflow du Skill

### Phase 1: Preparation

1. Calculer project_id (algorithme du remember skill)
2. Charger ECL-BUG + ECL-IDEA existants depuis Mem0
3. Calculer la heatmap des zones explorees:
   - Chaque ECL-BUG: zones_explored[] -> +1 point par zone
   - Chaque ECL-IDEA (security, code_quality): evidence[].path -> +0.5 point par zone
   - Zones = repertoires de premier/second niveau du projet
   - Zones avec score le plus bas = priorite de chasse
   - Zones jamais explorees = priorite maximale
4. Detecter stack: test runner, test command, framework, dev URL

### Phase 2: Dispatch Agent

Passer au bug-hunter agent:
- `zones_least_explored[]` (top 5)
- `known_issues[]` (fingerprints + titres ECL-BUG et ECL-IDEA existants)
- `stack_context` (test runner, URLs, framework)
- `focus` ($ARGUMENTS si fourni, override les zones)

Attendre resultat: `BUG_FOUND` ou `NO_BUG`.

Si `NO_BUG`: afficher zones explorees, proposer relance.

### Phase 3: Bug Packet

1. Dedup: verifier fingerprint + Jaccard contre existants
2. Stocker ECL-BUG en Mem0 (status: found)
3. Afficher le packet formate:

```
Bug Trouve -- ECL-BUG-NNNN
----------------------------

Severite: [HIGH]
Categorie: [validation]
Decouvert via: [browser]

1. Etapes de reproduction
   1. ...
   2. ...

2. Attendu vs Observe
   Attendu: ...
   Observe: ...

3. Impact metier
   ...

4. Cause probable
   fichier:ligne -- explication

5. Preuves
   - Screenshot: docs/bugs/ECL-BUG-NNNN/repro-01.png
   - Test output: ...

6. Video de repro (si browser)
   docs/bugs/ECL-BUG-NNNN/repro.webm

----------------------------
-> "je valide ce bug" | "faux positif" | "wont fix"
```

4. GATE: attendre reponse user

### Phase 4: Fix

1. Status -> fixing
2. Agent corrige (max 3 tentatives)
3. Lance les tests pour verifier
4. Si browser bug: re-navigue pour verifier
5. Si blocked apres 3 -> status:blocked, STOP
6. Si OK -> status:fixed

### Phase 5: Preuve Post-Fix

1. Browser bug: video du meme parcours (fix OK)
2. Code/test bug: test output + screenshots
3. Diff avant/apres si pertinent
4. Ajouter evidence[] dans ECL-BUG
5. Status -> verified
6. Afficher comparaison AVANT / APRES
7. GATE: attendre reponse user
   - "je valide le fix" -> Phase 6
   - "pas bon" -> status:fixing, re-Phase 4
   - "abandon" -> status:wont_fix

### Phase 6: Commit & Relance

1. `git add` (fichiers modifies uniquement)
2. `git commit -m "fix(scope): [title]"`
3. Stocker commit_sha dans ECL-BUG
4. Status -> done
5. Afficher resume
6. Relance automatique -> retour Phase 1 (user peut dire "stop")

## Agent bug-hunter.md

### Inputs

- `zones_least_explored[]` -- top 5 zones a prioriser
- `known_issues[]` -- fingerprints + titres ECL-BUG et ECL-IDEA existants
- `stack_context` -- test runner, test command, framework, dev URL
- `focus` -- $ARGUMENTS (optionnel, override les zones)

### Process

#### Phase A: Reconnaissance

1. Lire structure du projet (Glob)
2. Identifier zones cibles (focus > zones_least_explored)
3. Identifier entry points: routes, formulaires, API handlers, middleware
4. Lire git log --oneline -20 pour fichiers recemment modifies

#### Phase B: Chasse Statique (code)

1. **Bugs logiques**: conditions inversees, off-by-one, comparaisons laches, variables shadowed, return manquant, null/undefined non gere, promesses non-awaited, race conditions
2. **Incoherences metier**: enums dupliques qui ne matchent pas, validation client/serveur incoherente, routes sans handler, permissions verifiees a un endroit mais pas un autre
3. **Securite exploitable**: injection (SQL, commande, XSS), auth bypass, IDOR, secrets hardcodes
4. **Error handling**: try/catch qui avale l'erreur, erreurs async non catchees, messages d'erreur qui leak des infos

Comparer chaque trouvaille avec known_issues[]. Skip si deja connu.

#### Phase C: Chasse par Tests

1. Lancer la test suite complete
2. Si tests FAIL: analyser si vrai bug ou flaky
3. Si tous PASS: identifier zones non couvertes, assertions faibles

#### Phase D: Chasse Browser (si applicable)

Conditions: le projet a une UI ET une URL est detectable.

1. Detecter URL de dev (.env, package.json scripts, fallback localhost)
2. Ouvrir agent-browser, snapshot
3. Explorer parcours a risque: formulaires, flows multi-etapes, pages dynamiques, pages d'erreur
4. Tester edge cases: champs vides, valeurs extremes, double-clic, back button, refresh mid-flow
5. Si bug trouve: enregistrer video + screenshots annotes

#### Phase E: Rapport

Retourner JSON structure:
- `BUG_FOUND` + bug object complet + hunt_session
- `NO_BUG` + hunt_session (zones explorees, observations)

### Regles non-negociables

- UN seul bug par invocation
- JAMAIS corriger, seulement trouver et documenter
- JAMAIS fabriquer des preuves
- cause.file et cause.line sont OBLIGATOIRES
- Toujours fermer agent-browser en fin de session
- Si agent-browser echoue -> continuer avec code/tests

## Stockage des preuves

```
docs/bugs/
  ECL-BUG-0001/
    repro.webm          (video repro -- browser bugs)
    repro-01.png         (screenshot annote)
    repro-02.png         (screenshot annote)
    fix.webm             (video post-fix -- browser bugs)
    fix-01.png           (screenshot post-fix)
    test-output.txt      (sortie test -- code/test bugs)
```

## Modifications aux skills existants

### remember/SKILL.md

Ajouter BUG dans ID Generation:
```
Format: ECL-{TYPE}-{NNNN}
- TYPE: IDEA, COMP, FEAT, CTX, BUG
```

### next/SKILL.md

Step 2: inclure ECL-BUG dans les queries Mem0.

Step 3 (dashboard): ajouter section bugs entre "EN COURS" et "A VALIDER":
```
BUGS
  [ECL-BUG-0001] [title] -- [severity] -- [status]
```

Step 4 (suggest action): ajouter dans la priority chain:
- 1.5: found bug exists -> "Bug trouve: [id] [title]. Valider ou rejeter ?"
- 1.6: validated bug exists -> "Corriger [id]: [title]"
- 1.7: verified bug exists -> "Valider le fix de [id]: [title]"
