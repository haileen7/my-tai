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

- **The session starts inside the project directory** when `terminal.cwd` points at it (`hermes config set terminal.cwd /path/to/project`). Verify with `pwd`; do not re-`cd` into a directory you are already in.
- **Superpowers skills are the process default, not an option.** Load the matching one BEFORE acting: `brainstorming` for "let's build X", `systematic-debugging` for "fix this bug", `test-driven-development` for behavior changes, `requesting-code-review` before commit, `verification-before-completion` before reporting done. Announce the skill, then follow it. The `using-superpowers` bootstrap is already loaded — never re-load it, and ignore its body if you were dispatched as a subagent.
- Never clone the repo again. `cd` into the existing local repository.
- Never modify existing Git remotes.
- Never guess or hardcode branch names — read them from `git branch --show-current`.
- Never switch branches unless explicitly instructed.
- AGENTS.md (or equivalent) is authoritative project context — read it first, keep it loaded.
- All code changes commit and push to the current branch. Never push to shared branches (main, beta) without explicit instruction.
- **CodeGraph first for every structural question.** "Where is X", "which files/lines deal with Y", "how does this flow work", "what touches this before I change it" go to `codegraph_explore` (MCP) or `codegraph query|explore` (CLI) as the FIRST call — before `search_files`, before `read_file`. Reading files with grep to answer a structure question leaves an available, indexed graph idle and is the thing the user objects to. Structural question = explore; then `read_file` only for files explore did not show or for sections it truncated. Reserve text search for literal strings (see Pitfalls).

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

**Start with the built-in probe** — it reports transport, auth, and tool discovery per server in one shot, and separates "binary/config broken" from "tool call failed" without hand-rolling JSON-RPC:

```bash
hermes mcp list      # which servers are configured and enabled
hermes mcp test <name>   # ✓ Connected + tools discovered, or the failing stage
```

Then confirm each server with a live, cheap call:

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
# Prefer the configured remote when one actually points at the canonical repo.
git fetch <canonical-remote> <canonical-branch>
git rev-list --count HEAD..<canonical-remote>/<canonical-branch>   # behind
git rev-list --count <canonical-remote>/<canonical-branch>..HEAD   # ahead
```

When no remote points at the canonical repo, fetch by URL into `FETCH_HEAD` instead of adding a remote (adding one violates the standing rules):

```bash
git fetch <canonical-url> <canonical-branch>
git rev-list --count HEAD..FETCH_HEAD      # behind
git log --oneline -1 FETCH_HEAD            # what you are comparing against
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

**A fast-forward that lands a dependency bump leaves the installed tree stale.**
`git merge` only moves tracked files — it never runs a package manager, so
`composer.lock` and `vendor/` (or `package-lock.json` and `node_modules/`) can
now contradict the manifest you just pulled. After any merge touching a
manifest, confirm reality before trusting a build or an MCP probe:

```bash
composer show <package>            # installed version
composer validate --with-dependencies
npm ls <package> --depth=0        # or: search_files target='files' over node_modules
```

An MCP/CLI probe that reports a version older than the new constraint means the
manifest moved but the install did not — run the installer, then re-probe. Do not
report the merged dependency as active until `composer show` / `npm ls` agrees
with the new constraint.

### 7. Prove the Graph After a Sync

A fast-forward that lands new classes leaves a stale `.codegraph/` index, and the
new code is exactly what you will be asked about next. Re-index, then resolve one
of the new symbols to confirm the graph sees the pull:

```bash
codegraph sync                                  # or: codegraph init
codegraph query "<NewClassName>"                 # query = symbol + its references
codegraph explore "<NewClassName>"               # explore = relationships/call paths
```

`query` answers "where is this symbol used" (definitions, imports, call sites);
`explore` answers "what does this touch". Use `query` for a quick post-sync proof,
`explore` before changing a symbol's contract. Doing this unprompted also makes
tool usage visible instead of leaving it for the user to ask about.

## Pitfalls

- **Do not repeat a `cd` that already succeeded.** The terminal session's cwd
  persists between calls, so the first `cd <dir>` moves you in and the second
  `cd <dir>` fails with "No such file or directory" — the shell is *already*
  there, the path is not missing. Either omit the `cd` on later calls or pass an
  absolute path / the `workdir` parameter.

- **Staged changes already in `git status` at session start are residue, not your work.** A `D`/`M` in the left column is a previous session's staging area. Read it before syncing: `git diff --cached --stat` plus `git diff` tells you whether the index holds a real decision or leftover bookkeeping. Fold it into the sync (commit it, or restore it) rather than letting it silently block the next merge.

- **Pip vs npm CodeGraph:** Running `pip install codegraph` installs a completely different tool. The MCP config expects `@colbymchenry/codegraph` from npm. If `codegraph explore` returns usage text about matplotlib or D3.js instead of symbol data, you installed the wrong one — `pip uninstall codegraph && npm install -g @colbymchenry/codegraph`.
- **CodeGraph not in PATH:** After npm install, the binary lands at the npm global prefix (check with `npm config get prefix`). The MCP server config in `~/.hermes/config.yaml` must match this path. The MCP subprocess gets the absolute path, but your interactive shell may not have that directory on PATH — a `codegraph: command not found` from the terminal while `hermes mcp test codegraph` connects is exactly this. Prefix the session with `export PATH="$HOME/.npm-global/bin:$PATH"` or call the absolute path.
- **`codegraph init` is not instant on a large repo.** It walks the whole tree and builds the SQLite graph; run it with `background=true, notify=true` and do other work while it finishes rather than blocking the turn on it.
- **MCP server crash on first call:** Laravel Boost's stdio subprocess occasionally dies. This is transient — retry the call. If it persists across multiple retries, the PHP artisan process may need a restart.
- **Never batch local MCP tools:** `tool_call` with multiple local (non-connector) tool entries is rejected. Call each MCP tool individually.
- **`git remote -v` is ground truth:** Never assume which remote is 'origin' vs 'upstream'. Some forks rename remotes differently.
- **Per-instance agent notes are not repo content.** Bootstrapping tools drop scratch files in the project dir (`.hermes.md` and friends) and they show up as untracked noise. When the rule is "commit every change", gitignore these beside the existing agent dirs rather than committing them — they carry this instance's paths, not project knowledge. Commit the `.gitignore` line; leave the file untracked. **But check whether canonical upstream has since made the same call** (`git log canonical/<base> -- <path>`): if it deleted the file and ignored it, take upstream's version of that decision and do not re-add it locally.
- **A shared branch may be behind the canonical one after you fetch it.** When a fork branch tracks (or is published to) an upstream branch, compare with `git rev-list --left-right --count <canonical-sha>...HEAD` — a non-zero left count means work is missing even though the local tree looks clean. Fast-forward with `git merge --ff-only`, after stashing any dirty file only if the stash is truly redundant (compare the stashed blob against the target commit first, then drop it).

- **CodeGraph indexes symbols, not string literals — pair it with a text sweep.** Route paths (`/it/wireless`), Blade `link=`/`href` attributes, permission names as strings (`'bw'`, `'view_hr_dashboard'`), config keys and env vars do not resolve as symbols, so `codegraph explore` legitimately returns nothing for them. Do not read that as "nothing to see": follow the graph call with a literal sweep of the candidate strings. The two answer different halves of the same question.

- **Access-control audits need FIVE checks, not three — the two extra ones are where the real bugs hide.** Beyond the Blade gate / route middleware / seeder declaration, also check: (4) **ancestor gates** — a menu item is pruned by any enclosing `@canany`, so an item can carry the right permission and still be unreachable (add the child's permission to the parent's `@canany`); and (5) **middleware nesting** — `Route::middleware(['a','b'])`, or a route nested inside another permission group, requires **both**, while one `role_or_permission:a|b` means either. Read the resolved stack with `php artisan route:list --path=X -v` instead of reasoning about the source. Also remember Spatie's `hasAnyPermission()` answers `false` (no exception) for a permission name that is not in the DB, so an unseeded permission silently 403s every non-admin — a gate that "looks fine" in code and denies everyone in production.
- **When auditing access control, verify the same permission at all THREE layers and cross-check the sets.** For each link, compare (1) the Blade gate (`@can`/`@canany` around the `<x-menu-item>`), (2) the route middleware in `routes/web.php` (`role_or_permission:`), and (3) the permission's declaration in the seeder. Mismatches are the bugs, and they come in three shapes worth naming in the report: a link whose gate is looser or stricter than its route (user sees it, gets 403 — or the reverse, never sees a page they may open); a declared-but-never-referenced permission (dead) and a referenced-but-never-declared one (a gate that silently passes/fails for everyone); and a view (whole submenu or single route) reachable by every authenticated user because neither gate nor middleware exists. Also list routes with no menu link at all. Distinguish genuine inconsistency from deliberate design before calling it a bug — an accepted pattern (e.g. a feature gated by the broader permission that already covers it) is not a finding.

## Reporting While Long Commands Run

Full test suites and Playwright runs take minutes. Start them in the background, then either continue work that does not depend on the result or report progress on a short interval — do not go silent until completion. The user should never have to ask what is happening. State the real numbers (`N passed`, `0 failed`, plus any pre-existing risky/skipped count) and never round a partially-verified run up to a pass.

A `wait` on a background process is capped by a configured timeout and will be
released early with the process still running — that release is not a failure and
not the end of your turn. Reply with what is still pending and let the completion
notification arrive on its own; do not re-issue the wait in a loop.

**Verify seeder/table names instead of guessing them.** Seeders report row counts
in prose, and a guessed singular table name throws `relation "x" does not exist`
that aborts the rest of a counting loop, hiding the tables that would have
worked. Get the real names from `php artisan db:show` and count those.