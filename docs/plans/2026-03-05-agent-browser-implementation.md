# Replace Playwright with Agent Browser — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace Playwright MCP with Vercel's agent-browser CLI skill in the eclipse-tools plugin.

**Architecture:** Remove the Playwright MCP server, add the official agent-browser skill (enriched with autonomous agent capabilities), and update all references in ideation skills/agents.

**Tech Stack:** agent-browser CLI (Rust/Node.js), Bash for execution

---

### Task 1: Create the agent-browser skill

**Files:**
- Create: `skills/agent-browser/SKILL.md`
- Create: `skills/agent-browser/references/commands.md`
- Create: `skills/agent-browser/references/authentication.md`
- Create: `skills/agent-browser/references/snapshot-refs.md`
- Create: `skills/agent-browser/references/session-management.md`
- Create: `skills/agent-browser/references/profiling.md`
- Create: `skills/agent-browser/references/proxy-support.md`
- Create: `skills/agent-browser/references/video-recording.md`
- Create: `skills/agent-browser/templates/form-automation.sh`
- Create: `skills/agent-browser/templates/authenticated-session.sh`
- Create: `skills/agent-browser/templates/capture-workflow.sh`

**Step 1: Fetch and install the official skill**

Run:
```bash
cd /Users/andrelauret/Documents/05_Ressources/eclipse-tools
npx skills add vercel-labs/agent-browser
```

This should create `skills/agent-browser/SKILL.md` and associated reference/template files.

If `npx skills add` fails (not all environments support it), manually create the files by copying from the repo content fetched during design.

**Step 2: Verify the skill was created**

Run: `ls -la skills/agent-browser/`
Expected: SKILL.md exists along with references/ and templates/ directories.

**Step 3: Enrich SKILL.md with autonomous agent context**

Add the following section at the end of `skills/agent-browser/SKILL.md`, before any closing content:

```markdown
## Autonomous Agent Context (Eclipse-Tools)

As an agent installed on the server, you have full access to the project context beyond just the browser:

### URL Discovery
- Read project configs (`package.json` scripts, `.env`, `docker-compose.yml`, nginx configs) to find dev/staging URLs automatically
- Check ECL-CTX in Mem0 for `dev_url` if available

### Account and Permission Setup
- Use project CLI tools or access the DB directly (`psql`, `sqlite3`, Prisma CLI, Django manage.py, Rails console, etc.) to create test users and assign roles/permissions needed for your test scenario
- Read auth configs to understand available roles and permission models

### View Discovery
- Explore route files (`pages/`, `app/`, router configs) to know which views exist
- Navigate directly to the right pages without guessing

### Context Manipulation
- Seed test data via CLI/DB to set up the exact state needed
- Configure feature flags, environment variables, or app settings
- Clean up test data after verification

### Full Test Workflow
1. **Prepare context** — Create users, seed data, configure flags (via CLI/DB)
2. **Open browser** — `agent-browser open <discovered-url>`
3. **Navigate and verify** — Snapshot, interact, screenshot
4. **Clean up** — Remove test data, close browser
```

**Step 4: Commit**

```bash
git add skills/agent-browser/
git commit -m "feat: add agent-browser skill from vercel-labs/agent-browser"
```

---

### Task 2: Remove Playwright MCP from configuration

**Files:**
- Modify: `.mcp.json:7-9`

**Step 1: Remove the playwright entry from .mcp.json**

Edit `.mcp.json` to remove the `playwright` server entry. The file should go from:

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
      "command": "uvx",
      "args": ["mem0-mcp-server"],
      "env": {
        "MEM0_API_KEY": "${MEM0_API_KEY}"
      }
    }
  }
}
```

To:

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    },
    "mem0": {
      "command": "uvx",
      "args": ["mem0-mcp-server"],
      "env": {
        "MEM0_API_KEY": "${MEM0_API_KEY}"
      }
    }
  }
}
```

**Step 2: Commit**

```bash
git add .mcp.json
git commit -m "chore: remove Playwright MCP server, replaced by agent-browser CLI"
```

---

### Task 3: Update ideation skill to use agent-browser

**Files:**
- Modify: `skills/ideation/SKILL.md:69-83`

**Step 1: Rewrite the Playwright Pre-check section**

Replace lines 69-83 of `skills/ideation/SKILL.md` (the "Playwright Pre-check" section) with:

```markdown
## Browser Pre-check (for UX agent)

Before dispatching agents, check if a dev server is running:

1. Read ECL-CTX for `dev_url` — if set, use it
2. Else, read package.json `scripts` for "dev"/"start"/"serve" — extract port
3. Attempt to navigate to the detected URL using agent-browser CLI:
   ```bash
   agent-browser open <detected-url> && agent-browser wait --load networkidle
   ```
4. If successful:
   - Take screenshots: `agent-browser screenshot --full`
   - Capture accessibility snapshot: `agent-browser snapshot -i -c`
   - Pass screenshots and snapshot observations to the UX agent as extra context
   - Close browser: `agent-browser close`
5. If failed or no URL detected:
   - Log: "No running app detected — UX agent will use static analysis only"
   - Pass no browser context to UX agent
6. **NEVER block the pipeline** if agent-browser fails. Catch all errors, log, continue.
```

**Step 2: Update the Dispatch Phase agent prompt reference**

Check lines 95-108 in `skills/ideation/SKILL.md`. The prompt template for UX agent says `[For UX agent only: Playwright observations: ...]`. Change to:

```
[For UX agent only: Browser observations: ...]
```

**Step 3: Verify no other Playwright references remain in ideation skill**

Run: `grep -i playwright skills/ideation/SKILL.md`
Expected: No output (zero matches).

**Step 4: Commit**

```bash
git add skills/ideation/SKILL.md
git commit -m "feat: replace Playwright MCP with agent-browser CLI in ideation skill"
```

---

### Task 4: Update ideation-ux agent

**Files:**
- Modify: `agents/ideation-ux.md:5-6`
- Modify: `agents/ideation-ux.md:58-61`
- Modify: `agents/ideation-ux.md:96`

**Step 1: Update the agent description**

Line 5-6 currently says:
```
  Playwright live audit is handled by the ideation orchestrator skill, not this agent.
```

Change to:
```
  Browser live audit (via agent-browser) is handled by the ideation orchestrator skill, not this agent.
```

**Step 2: Update the Playwright Context section header and content**

Lines 58-61 currently say:
```markdown
## Playwright Context (if provided)

If the orchestrator passes you screenshot observations or live audit findings, incorporate them as additional evidence with `"type": "playwright_observation"`.
```

Change to:
```markdown
## Browser Context (if provided)

If the orchestrator passes you screenshot observations or live audit findings, incorporate them as additional evidence with `"type": "browser_observation"`.
```

**Step 3: Update the process step**

Line 96 currently says:
```
7. Incorporate any Playwright observations if provided
```

Change to:
```
7. Incorporate any browser observations if provided
```

**Step 4: Verify no Playwright references remain**

Run: `grep -i playwright agents/ideation-ux.md`
Expected: No output (zero matches).

**Step 5: Commit**

```bash
git add agents/ideation-ux.md
git commit -m "feat: rename Playwright references to browser in ideation-ux agent"
```

---

### Task 5: Final verification

**Step 1: Grep entire project for remaining Playwright references**

Run: `grep -ri playwright --include='*.md' --include='*.json' .`
Expected: No output (zero matches across the entire project).

**Step 2: Verify .mcp.json is valid JSON**

Run: `cat .mcp.json | python3 -m json.tool`
Expected: Valid JSON output with only `context7` and `mem0` servers.

**Step 3: Verify git status is clean**

Run: `git status`
Expected: Clean working tree with all changes committed.

**Step 4: Final commit (if any stragglers)**

Only if needed:
```bash
git add -A
git commit -m "chore: complete Playwright to agent-browser migration"
```
