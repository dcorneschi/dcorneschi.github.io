# Viewing Unpushed Commits in Git

Before pushing — or when you just want to know how far ahead of the remote you are — it's useful to list the commits that exist locally but aren't on the remote yet. Git's range syntax and the `@{u}` upstream shorthand make this quick. This guide covers viewing, diffing, counting, and saving your unpushed work.

## View Unpushed Commits

```bash
# Commits on your local branch not yet on origin/main
git log origin/main..HEAD
```

Substitute your branch name for `main` as needed (`master`, `develop`, etc.). The range `origin/main..HEAD` means "commits reachable from HEAD but not from origin/main" — exactly your unpushed work.

### Compact view

```bash
git log origin/main..HEAD --oneline
```

### See the actual changes

```bash
git diff origin/main..HEAD
```

### Works for any branch — the upstream shorthand

```bash
git log @{u}..HEAD
```

`@{u}` (or `@{upstream}`) resolves to whatever upstream your current branch tracks, so this works regardless of branch name — as long as an upstream is configured. Even shorter, `git log @{push}..` targets the branch you'd push to.

### Quick list with messages — git cherry

```bash
git cherry -v
```

`git cherry -v` lists unpushed commits with their subjects, comparing against the upstream. Empty output means you're all caught up. Unlike the range commands, it compares by **patch content**, so a commit already applied upstream under a different hash is marked `-` (equivalent) rather than `+` (still unique) — useful after a rebase or cherry-pick.

## Make Sure You're Comparing Against Current Data

`origin/main` is a **remote-tracking ref** — a local cache of where the remote was at your last fetch. If it's stale, your "unpushed" list may be inaccurate. Refresh it first without merging:

```bash
git fetch origin
git log origin/main..HEAD --oneline
```

`git fetch` updates the tracking refs only; it doesn't touch your working tree or current branch.

## Count and Summarize

```bash
# How many commits am I ahead?
git rev-list --count origin/main..HEAD

# Ahead/behind in one line (behind<TAB>ahead)
git rev-list --left-right --count origin/main...HEAD

# Files changed across unpushed commits
git diff --stat origin/main..HEAD

# Just the file names
git diff --name-only origin/main..HEAD
```

`git status` also reports this against your upstream — look for "Your branch is ahead of 'origin/main' by N commits."

## Check Across All Branches

To see unpushed commits on **every** local branch at once:

```bash
# Commits on any local branch not on any remote
git log --branches --not --remotes --oneline

# Branch-by-branch ahead/behind summary
git for-each-ref --format='%(refname:short) %(upstream:track)' refs/heads
```

The second command prints something like `feature [ahead 3]` for each branch that tracks an upstream.

## Save to a File

### Commit log

```bash
git log origin/main..HEAD > unpushed-commits.txt
```

### Compact log

```bash
git log origin/main..HEAD --oneline > unpushed-commits.txt
```

### The changes (diff)

```bash
git diff origin/main..HEAD > unpushed-changes.txt
```

### Both at once

```bash
git log origin/main..HEAD > unpushed-commits.txt && \
git diff origin/main..HEAD > unpushed-changes.txt
```

### As a portable patch

If you want to *apply* the unpushed work elsewhere rather than just read it, export it as patches instead of a plain diff:

```bash
# One patch file per unpushed commit, with full metadata
git format-patch origin/main
```

## Quick Reference

| Command | Shows |
|---------|-------|
| `git log origin/main..HEAD` | Unpushed commits (full) |
| `git log origin/main..HEAD --oneline` | Unpushed commits (compact) |
| `git log @{u}..HEAD` | Unpushed commits vs tracked upstream |
| `git cherry -v` | Unpushed commits with messages (patch-equivalent) |
| `git diff origin/main..HEAD` | Actual code changes not yet pushed |
| `git rev-list --count @{u}..HEAD` | Number of commits ahead |
| `git log --branches --not --remotes` | Unpushed commits across all branches |
| `git format-patch origin/main` | Unpushed commits as patch files |

## Notes

- Run `git fetch` first for an accurate comparison — remote-tracking refs are only as current as your last fetch.
- `@{u}` requires an upstream; set one with `git push -u origin HEAD` (or `git branch --set-upstream-to`).
- Remember the dot difference: two dots (`..`) for the commit list, three dots (`...`) for symmetric ahead/behind counts.
