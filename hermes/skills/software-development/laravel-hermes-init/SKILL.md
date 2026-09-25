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

In Hermes, an `AGENTS.md` inside the project directory auto-attaches as **Subdirectory context** as soon as the session's cwd is inside that project — check whether you already received it before reading it again, and treat it as authoritative if present. If you never saw it, `search_files` for it explicitly; do not proceed on the assumption that it does not exist.

### 3. Verify MCP Tools

Test each configured MCP server with a lightweight call:

| Server | Test call | Expected |
|---|---|---|
| Laravel Boost | `application_info` | JSON with php_version, laravel_version, packages |
| Context7 | `query_docs` with `/laravel/docs` | Markdown doc excerpt |
| GitHub MCP | `search_repositories` with repo name | JSON with matching repos |
| CodeGraph | CLI: `codegraph explore "<symbol>"` | Symbol relationships |

**Server absent from `tool_search`:** MCP servers are registered once, at session start. If a configured server is missing from the `available_sources` list, fixing its config/binary now will NOT make it appear this session — it registers on the next session start. Confirm the server itself works (handshake below), then use its CLI equivalent for the rest of this session and expect the MCP tools next session. Re-run `tool_search` once after any fix rather than assuming.

**Stdio handshake to verify a server independently of Hermes** (isolates "config broken" from "transient death"):

```bash
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | timeout 20 sh -c '<the exact command+args from ~/.hermes/config.yaml>'
```

A `result.serverInfo` line means the server is healthy — the failure was transport-side.

**Laravel Boost intermittent crash:** the first MCP call may lose its stdio subprocess. The error tells you not to replay it blindly — verify the external state first (`php artisan boost:mcp` handshake as above), then retry the tool call exactly once.

**MCP tools are local tools** — each must be called individually via `tool_call`. Cannot batch multiple local MCP tools in one call (only connectors can batch).

**Fall back to the equivalent CLI the moment an MCP call fails, don't keep retrying.** Two failure shapes seen in practice, both with a working CLI answer:

- `tool_call` returns `calls is not valid JSON: Expecting ',' delimiter` — the argument payload is being mangled, often by long bodies or non-ASCII text. Shortening the payload may work; if it keeps failing, switch tools rather than shrinking the message further.
- `MCPError: Authentication Failed` — the server's token is stale even though the CLI is authenticated. Confirm with `gh auth status`, then drive the operation with `gh` (e.g. `gh pr create --repo <owner>/<repo> --base <base> --head <fork-user>:<branch> --title-file`/`--body-file`).

Rule: verify the fallback actually produced the artifact (`gh pr view <n> --json url,state,mergeable`) before reporting success. Cross-repo PRs need the fork qualified as `head: <owner>:<branch>`; for a long body, write it to a scratch file and pass `--body-file` rather than inlining it.

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

**The local branch name and the remote branch it publishes to are often different** (e.g. a work branch configured to publish onto the fork's `beta`). Read the real destination instead of assuming:

```bash
git config branch.<current-branch>.remote   # which remote
git config branch.<current-branch>.merge    # which remote branch (refs/heads/...)
```

Then push explicitly to that ref — a plain `git push` targets a same-named remote branch and errors when the configured merge ref differs:
```bash
git push <remote> HEAD:<branch-from-merge-config>
```

Before syncing, confirm the three positions (`HEAD`, fork branch, canonical branch) with `git rev-parse` and `git ls-remote <canonical-url> refs/heads/<branch>`; if they already agree there is nothing to pull or push — report "in sync" instead of manufacturing a commit.

## Pitfalls

- **Pip vs npm CodeGraph:** Running `pip install codegraph` installs a completely different tool. The MCP config expects `@colbymchenry/codegraph` from npm. If `codegraph explore` returns usage text about matplotlib or D3.js instead of symbol data, you installed the wrong one — `pip uninstall codegraph && npm install -g @colbymchenry/codegraph`.
- **CodeGraph not in PATH:** After npm install, the binary lands at the npm global prefix (check with `npm config get prefix`). The MCP server config in `~/.hermes/config.yaml` must match this path.
- **MCP server crash on first call:** Laravel Boost's stdio subprocess occasionally dies. This is transient — retry the call. If it persists across multiple retries, the PHP artisan process may need a restart.
- **Never batch local MCP tools:** `tool_call` with multiple local (non-connector) tool entries is rejected. Call each MCP tool individually.
- **`git remote -v` is ground truth:** Never assume which remote is 'origin' vs 'upstream'. Some forks rename remotes differently.
- **A shared branch may be behind the canonical one after you fetch it.** When a fork branch tracks (or is published to) an upstream branch, compare with `git rev-list --left-right --count <canonical-sha>...HEAD` — a non-zero left count means work is missing even though the local tree looks clean. Fast-forward with `git merge --ff-only`, after stashing any dirty file only if the stash is truly redundant (compare the stashed blob against the target commit first, then drop it).

## Reporting While Long Commands Run

Full test suites and Playwright runs take minutes. Start them in the background, then either continue work that does not depend on the result or report progress on a short interval — do not go silent until completion. The user should never have to ask what is happening. State the real numbers (`N passed`, `0 failed`, plus any pre-existing risky/skipped count) and never round a partially-verified run up to a pass.