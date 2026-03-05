# Design: Remplacement Playwright par Agent Browser

**Date:** 2026-03-05
**Statut:** Validé

## Contexte

Le plugin eclipse-tools utilise actuellement le serveur MCP `@playwright/mcp` pour l'automatisation navigateur (pré-check UX dans le pipeline ideation). L'objectif est de remplacer Playwright MCP par [agent-browser](https://github.com/vercel-labs/agent-browser) de Vercel, un CLI headless en Rust optimisé pour les agents IA.

## Décision

**Approche A — Skill officiel** : Récupérer le skill officiel agent-browser, l'intégrer dans le plugin, adapter les skills existants.

## Changements

### 1. Intégration du skill agent-browser

- Récupérer le `SKILL.md` officiel de `vercel-labs/agent-browser`
- Le placer dans `skills/agent-browser/SKILL.md`
- Ce skill enseigne à Claude le workflow : `open` → `snapshot -i` → interact via refs (`@e1`, `@e2`) → `close`

### 2. Configuration MCP

- Supprimer l'entrée `playwright` de `.mcp.json`
- Pas de remplacement MCP — agent-browser s'utilise via Bash (CLI)

### 3. Adaptation de `skills/ideation/SKILL.md`

La section "Playwright Pre-check" (lignes 69-83) sera réécrite pour utiliser des commandes CLI agent-browser :

```bash
agent-browser open <url>
agent-browser screenshot
agent-browser snapshot -i -c
agent-browser close
```

La logique reste identique : détecter un serveur de dev → naviguer → capturer screenshots + snapshot accessibility → passer au UX agent → ne jamais bloquer le pipeline si ça échoue.

### 4. Adaptation de `agents/ideation-ux.md`

- Renommer "Playwright observations" → "Browser observations"
- Le type d'evidence `"playwright_observation"` → `"browser_observation"`

### 5. Contexte d'agent autonome

Le skill agent-browser sera enrichi pour refléter les capacités d'un agent installé sur le serveur :

- **Découverte d'URL** : Lire les configs du projet (`package.json`, `.env`, docker-compose, nginx) pour trouver l'URL automatiquement
- **Création de comptes/permissions** : Utiliser le CLI du projet ou accéder à la DB (`psql`, `sqlite3`, Prisma CLI, etc.) pour créer des utilisateurs de test et attribuer des rôles
- **Connaissance des vues** : Explorer les routes (fichiers routes, `pages/`, `app/`) pour naviguer directement aux bonnes pages
- **Manipulation du contexte** : Seeder des données via CLI/DB, configurer des feature flags, manipuler l'état de l'application pour les conditions exactes du test
- **Workflow complet** : Préparer le contexte (DB/CLI) → ouvrir le navigateur → naviguer → vérifier visuellement → nettoyer

## Fichiers impactés

| Fichier | Action |
|---------|--------|
| `.mcp.json` | Supprimer entrée `playwright` |
| `skills/agent-browser/SKILL.md` | Créer (skill officiel + enrichissement agent autonome) |
| `skills/ideation/SKILL.md` | Réécrire section Playwright Pre-check |
| `agents/ideation-ux.md` | Renommer références Playwright → Browser |

## Pas de changements sur

- Les autres agents ideation (code, quality, docs, perf, security)
- Les autres skills (remember, next, design, plan, build, finish, competitor, roadmap)
- `settings.json`, `hooks/hooks.json`, `plugin.json`
