---
name: git-workflow
description: "Git pitfalls: nested repos, staging, identity."
version: 1.0.0
author: Sydney
license: MIT
metadata:
  hermes:
    tags: [git, workflow, pitfalls, staging, nested-repos, commit]
    category: software-development
---

# Git Workflow

Pitfalls and procedures for everyday git operations that fall outside standard
`gh` CLI workflows (for gh-specific flows see the `github` skill).

## Standing Rules

- Always check `git status` and `git remote -v` before staging or pushing.
- Configure per-repo identity before first commit if global config is absent.
- Never assume a branch name — read it from `git branch --show-current`.

## Pitfalls

### Syncing a fork branch against a canonical upstream

When the local remote is a fork and the real base is elsewhere, add the
canonical repo as a **named remote-tracking ref** rather than a permanent remote,
so no existing remote config is modified:

```bash
git fetch https://github.com/<owner>/<repo>.git <base-branch>:refs/remotes/canonical/<base-branch>
git rev-list --left-right --count HEAD...canonical/<base-branch>   # ahead<TAB>behind
git log --oneline HEAD..canonical/<base-branch>                    # incoming
git merge --ff-only canonical/<base-branch>
```

Fetch the URL separately (`git ls-remote <url> <branch>`) to confirm the
upstream SHA before merging — a fork's own tracking branch can be stale or
diverged, and comparing against it answers the wrong question.

**A local untracked file that upstream now tracks will abort the fast-forward**
with `untracked working tree files would be overwritten by merge`. This happens
when a boot-time file (`AGENTS.md`, `.hermes.md`, editor settings) is written
locally on every session start and later committed upstream. Delete the local copy,
merge, let the tracked version land, then diff the two to confirm nothing local
was lost. Do not resolve it by adding the path to `.git/info/exclude` — that
guarantees a permanent divergence from upstream.

### Nested `.git` directories when copying content

When `cp -r` (or similar) a directory that contains its own `.git` into another repo,
`git add` detects the nested `.git` and stages only a submodule reference (mode 160000),
not the actual files. Clones of the outer repo will not contain the copied content.

**Fix — remove nested `.git` before staging:**

```bash
cp -r /source/dir target/inside/repo
rm -rf target/inside/repo/.git
git add target/inside/repo/
```

If already staged as submodule:

```bash
git rm -r --cached target/inside/repo/
rm -rf target/inside/repo/.git
git add target/inside/repo/
git commit -m "Add dir contents (fix nested repo)"
```

### Missing git identity on fresh clones

New clones may lack both global and per-repo `user.name`/`user.email`.
Commits fail with `empty ident name`.

**Fix — set per-repo config before first commit:**

```bash
git config user.email "user@users.noreply.github.com"
git config user.name "username"
```

Detect from `gh auth status` or set manually.

## Verification

- `git status` shows no unexpected submodule entries.
- `git diff --cached --stat` shows actual file additions, not just mode changes.
- Commit and push succeed without warnings about embedded repos.
