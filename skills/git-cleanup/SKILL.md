---
name: git-cleanup
description: Use when the user wants to tidy up a git repo's local branches — invoked via "/git-cleanup", "clean up branches", "prune branches", or after merging PRs and wanting to remove stale local refs. Prunes remote-tracking refs, deletes merged local branches, and flags shared branches before deleting.
---

# Git Cleanup

## Overview

Tidy a git repository's local branch state. Fetches with prune, summarizes branch status, flags any branch that contains collaborator commits, deletes merged local branches, and reports what was cleaned.

Works in any repo — safe by default. Destructive actions are confirmed before running.

## When to Use

- User types `/git-cleanup` or says "clean up branches", "prune branches", "clean up git"
- After merging PRs: user wants to remove stale local branches
- User complains about long `git branch` output / "too many branches"
- Out-of-scope: rewriting history, deleting remote branches on the server, force-pushing, resolving conflicts

## Safety Posture

- **Never force-delete** (`-D`) without explicit user approval for each branch.
- **Never touch remote branches** on the server (no `git push --delete`) unless the user explicitly asks.
- **Never delete the current branch, `main`, `master`, or `develop`.**
- **Always confirm** before deleting shared branches (branches containing commits by other authors).
- **Never run `git clean -fd`** as part of this flow — that targets working-tree files, not branches.

## Procedure

### 1. Fetch and prune

```bash
git fetch --all --prune
git branch -vv
git branch -a --format='%(refname:short) %(upstream:track)'
```

`--prune` removes remote-tracking refs whose upstream was deleted on the server — this covers the "remove remote-tracking refs for deleted remotes" step.

### 2. Summarize

Report in this shape:

- **Current branch:** name + ahead/behind vs upstream
- **Recently merged (local):** branches fully merged into `main`/`master`/default
- **Gone upstream:** branches whose upstream is `[gone]` in `branch -vv` — these are the squash-merge / PR-merged candidates
- **Ahead/behind:** branches with unpushed work or that need a pull
- **Stale:** local branches with no recent commit activity (optional — only flag if asked)

Useful one-liners:

```bash
# Branches fully merged into the default branch
git branch --merged "$(git symbolic-ref refs/remotes/origin/HEAD --short | sed 's|origin/||')"

# Branches whose upstream is gone (typical after PR merge / squash)
git branch -vv | awk '/: gone]/{print $1}'
```

### 3. Flag shared branches

Before deleting any candidate, check whether it contains commits by other authors:

```bash
git log --format='%ae' main..<branch> | sort -u
```

If the output contains any author other than the current user (`git config user.email`), **stop and ask** before deleting that specific branch. Call it out by name in the confirmation prompt.

### 4. Delete merged local branches

Use `-d` (safe delete — refuses if unmerged):

```bash
git branch -d <branch>
```

For branches that are `[gone]` upstream but `-d` refuses (common with squash-merged PRs, because the local SHA was never merged literally):

1. Confirm with the user that this branch was squash-merged via PR.
2. Use `git branch -D <branch>` **only** after explicit user confirmation for that branch.

Don't batch `-D` across a set. Confirm per branch, or get blanket approval for an explicit list.

### 5. Post-prune check

```bash
git remote prune origin
git branch -vv
```

(Usually a no-op after step 1, but confirms the final state.)

### 6. Report

Print a short summary:

- Branches deleted (list)
- Branches skipped and why (unmerged, shared, protected)
- Remote-tracking refs pruned
- Any follow-ups the user should consider (e.g., "`feature/x` has unpushed commits — push or delete manually")

## Quick Reference

| Situation | Command | Notes |
|-----------|---------|-------|
| Sync + prune remote refs | `git fetch --all --prune` | Run first |
| See tracking state | `git branch -vv` | `[gone]` = upstream deleted |
| List merged branches | `git branch --merged <default>` | Safe candidates for `-d` |
| Safe delete | `git branch -d <branch>` | Refuses if unmerged |
| Force delete | `git branch -D <branch>` | Only after per-branch confirmation |
| Check authors on a branch | `git log --format='%ae' main..<branch> \| sort -u` | Shared if >1 author |
| Prune remote-tracking only | `git remote prune origin` | Subset of `fetch --prune` |

## Common Mistakes

- **Deleting `[gone]` branches without checking for unpushed work.** `[gone]` means upstream is gone, not that local commits are merged. Always inspect `git log <default>..<branch>` first.
- **Using `-D` as the default.** `-d` is safe; `-D` is force. Default to `-d` and escalate only with explicit approval.
- **Forgetting squash-merge case.** GitHub/GitLab squash-merges create a new commit on `main` with a different SHA. Local branch appears unmerged to `git branch --merged` even though the PR shipped. Use the `[gone]` upstream signal + user confirmation.
- **Deleting `main`/`master`/`develop` or the current branch.** Hard-exclude these from any candidate list.
- **Silently deleting branches with collaborator commits.** Always flag and ask first.
- **Running `git push --prune` or `git push --delete` as part of cleanup.** Out of scope unless the user explicitly requested remote deletion.
