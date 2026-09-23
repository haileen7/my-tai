---
name: laravel-hermes-init
description: "Set up Hermes session for Laravel projects with MCP tools."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [laravel, mcp, codegraph, initialization, development-workflow]
    category: software-development
---

# Laravel + Hermes Session Initialization

## When to Use

At the start of every Hermes session working on a Laravel project that has MCP tools configured (Laravel Boost, Context7, GitHub MCP, CodeGraph). Triggers: entering a Laravel project directory, user says 'init' or 'setup session', or after a fresh session start on a known Laravel repo.

Procedure for starting a Hermes Agent session on a Laravel project that uses MCP tools (Laravel Boost, Context7, GitHub MCP, CodeGraph). Apply at the start of every session — not on user request.

## Standing Rules

- Never clone the repo again. `cd` into the existing local repository.
- Never modify existing Git remotes.
- Never guess or hardcode branch names — read them from `git branch --show-current`.
- Never switch branches unless explicitly instructed.
- AGENTS.md (or equivalent) is authoritative project context — read it first, keep it loaded.
- All code changes commit and push to the current branch. Never push to shared branches (main, beta) without explicit instruction.

## Procedure

### 1. Git Inspection

```bash
git remote -v
git branch --show-current
git status --short
```

Record: remote names, current branch, any uncommitted changes. Never assume origin/upstream semantics — always read remotes.

### 2. Read Project Instructions

Read `AGENTS.md` (or `CLAUDE.md` / `.hermes.md`). These files define project-specific gotchas, conventions, and tool usage. Keep their instructions in context for the entire session.

### 3. Verify MCP Tools

Test each configured MCP server with a lightweight call:

| Server | Test call | Expected |
|---|---|---|
| Laravel Boost | `application_info` | JSON with php_version, laravel_version, packages |
| Context7 | `query_docs` with `/laravel/docs` | Markdown doc excerpt |
| GitHub MCP | `search_repositories` with repo name | JSON with matching repos |
| CodeGraph | CLI: `codegraph explore "<symbol>"` | Symbol relationships |

**Laravel Boost intermittent crash:** First MCP call may lose its stdio subprocess. Retry once — this is a known transient issue, not a config problem.

**MCP tools are local tools** — each must be called individually via `tool_call`. Cannot batch multiple local MCP tools in one call (only connectors can batch).

### 4. Install CodeGraph if Missing

CodeGraph provides semantic code intelligence (symbol resolution, call paths, blast radius). AGENTS.md often mandates its use before grep/read_file for structural questions.

**Correct npm package:** `@colbymchenry/codegraph`

```bash
npm install -g @colbymchenry/codegraph
```

**Verify path matches MCP config:**
```bash
cat ~/.hermes/config.yaml | grep -A 5 'codegraph'
```
The MCP config specifies the binary path. If npm installs to a different location, the MCP server will fail to start. Common paths:
- npm global: `~/.npm-global/bin/codegraph`
- pip: `~/.local/bin/codegraph` (WRONG — this is a different, unrelated package)

**Do NOT install the pip `codegraph` package** — it's a different tool (code metrics/graphing) that conflicts by name. Always use the npm `@colbymchenry/codegraph`.

### 5. Initialize CodeGraph Index

```bash
cd /path/to/project && codegraph init
```

This scans and indexes the codebase (typically <10s for medium projects). Index is stored locally in `.codegraph/`.

**Sync after edits:**
```bash
codegraph sync
```
Run this if files were edited while no index was running (e.g., between sessions).

**Verify:**
```bash
codegraph status .
codegraph explore "<known class or method>"
```

### 6. Sync with Upstream

```bash
git fetch upstream
git log --oneline upstream/beta..HEAD   # commits ahead of beta
git log --oneline HEAD..upstream/beta   # commits behind beta
```

Push current branch to its configured remote after any work:
```bash
git push origin HEAD
```

## Pitfalls

- **Pip vs npm CodeGraph:** Running `pip install codegraph` installs a completely different tool. The MCP config expects `@colbymchenry/codegraph` from npm. If `codegraph explore` returns usage text about matplotlib or D3.js instead of symbol data, you installed the wrong one — `pip uninstall codegraph && npm install -g @colbymchenry/codegraph`.
- **CodeGraph not in PATH:** After npm install, the binary lands at the npm global prefix (check with `npm config get prefix`). The MCP server config in `~/.hermes/config.yaml` must match this path.
- **MCP server crash on first call:** Laravel Boost's stdio subprocess occasionally dies. This is transient — retry the call. If it persists across multiple retries, the PHP artisan process may need a restart.
- **Never batch local MCP tools:** `tool_call` with multiple local (non-connector) tool entries is rejected. Call each MCP tool individually.
- **`git remote -v` is ground truth:** Never assume which remote is 'origin' vs 'upstream'. Some forks rename remotes differently.