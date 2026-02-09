---
name: git-worktree-vibe-flow
description: Mandatory worktree development SOP (GitHub PR). Always follow for branch/worktree/commit/push/PR lifecycle and parallel vibe coding.
---

# Git Worktree Vibe Flow (GitHub PR)

Operate with a stable control worktree (repo root) + one task worktree per task.

Defaults:
- Control worktree stays on base and always up to date.
- All edits/tests happen in task worktrees.
- Auto-commit + auto-push + auto-create PR.
- After PR merge: auto-delete remote branch + remove worktree + delete local branch.

## Safety rules (non-negotiable)

- Never run destructive/history-rewriting commands: `git reset --hard`, `git push --force`, interactive rebase.
- Never commit or print secrets. Never stage `.env*`/secret files.
- Never remove a worktree unless it is clean.
- Do not change global user config (git config, package manager global config) unless the user explicitly asks.

## Conventions

- Control worktree: the current directory.
- Worktree storage (outside repo): `<repo_parent>/.worktrees/<repoName>/<branchPath>`.
- Branch naming (preferred):
  - `feature/<slug>` or `fix/<slug>`
  - `vibe/<YYYYMMDD>-<slug>`
  - `slug`: lowercase letters/digits/hyphen, keep short.

## Control worktree contract (base always latest)

Every session start (from control worktree):
- `git fetch --prune origin`
- Determine base from `origin/HEAD`: `git rev-parse --abbrev-ref origin/HEAD` (e.g. `origin/main`)
- `git switch <base>`
- `git pull --ff-only origin <base>`

If control worktree is dirty: do not develop there; create/reuse a task worktree instead.

## When to parallelize

Create a new worktree when tasks can progress independently or the current worktree is not clean. Avoid parallel worktrees if tasks touch the same core files.

## Create/reuse task worktree

- Reuse: `git worktree list --porcelain` (find `branch refs/heads/<branch>`)
- Create (from control):
  - Ensure parent dir exists: `<repo_parent>/.worktrees/<repoName>/`
  - `git worktree add -b <branch> <worktreePath> origin/<base>`
- Verify (in task worktree): `git status` + `git branch --show-current`

## Bootstrap: symlink local secrets/config (whitelist only)

Goal: keep ONE canonical local secret/config in repo root; each worktree symlinks to it (no copying).

Whitelist (repo-root files only):
- `.env`
- `.env.local`
- `.env.*.local` (e.g. `.env.development.local`)
- `.envrc` (direnv)
- `config.local.*`, `settings.local.*`, `*.local.json`, `*.local.yml`, `*.local.yaml`, `*.local.toml`
- `.npmrc` (treat as sensitive; never print; never commit)

Rules:
- Link only if (exists in control repo root) AND (ignored by git) AND (not tracked).
- Never overwrite a real file in the task worktree.

Never link build outputs or dependency dirs (do NOT share across branches):
- `node_modules/`, `.venv/`, `dist/`, `build/`, `.next/`, `target/`, `out/`.

## Bootstrap: install dependencies (per worktree)

Always install dependencies inside the task worktree.

Prefer existing lockfiles/tools (do not migrate by default):
- Node: `pnpm-lock.yaml` -> `pnpm install`; `package-lock.json` -> `npm ci`; `yarn.lock` -> `yarn install --frozen-lockfile`
- Python: `uv.lock` -> `uv sync`; `poetry.lock` -> `poetry install`; else venv + pip per repo docs

Always remind about global-cache acceleration (recommendation, not an automatic migration):
- If current setup lacks lockfile or installs are slow/unreproducible, recommend adopting a cached + reproducible flow (e.g. Node: pnpm; Python: uv/poetry). Only migrate if the user explicitly asks.

## Work: commit, push, PR (default automatic)

In task worktree:
1) Make changes + run minimal relevant checks.
2) Guardrail: staged files must NOT include `.env*`, `.npmrc`, `*.local.*`.
3) Commit (match repo subject style; inspect `git log -10 --pretty=format:%s`).
4) Push: `git push -u origin <branch>`
5) PR: `gh pr create --base <base> --head <branch> --title <title> --body <body>`

If `gh` is not available:
- Still push the branch.
- Provide the exact compare URL template and ask the user to create/merge the PR manually.

PR body should include:
- What/why, how to test, risks/follow-ups.

## Track PR to completion; auto cleanup after merge

After PR is created:
- Record PR URL/number.
- Check status: `gh pr view <pr> --json merged,mergedAt,state,headRefName,baseRefName,url`

When PR is merged:
1) Delete remote branch: `git push origin --delete <branch>` (note if rejected, continue).
2) Remove worktree (must be clean): `git worktree remove <worktreePath>`
3) Delete local branch: `git branch -d <branch>`
4) `git worktree prune`

If PR is closed/unmerged:
- Do not delete remote branch automatically.
- Ask the user whether to keep branch/worktree for later or clean up.

## Failure modes and what to do

- Worktree add fails because branch is checked out elsewhere:
  - Find the existing worktree via `git worktree list --porcelain` and reuse it.
- Worktree remove blocked (unclean):
  - Stop and ask: commit, stash, or discard (never discard automatically).
- Secrets accidentally staged:
  - Unstage immediately and add to `.gitignore` if missing; do not proceed until clean.
