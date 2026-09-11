# Getting the Latest Changes from Master into Your Feature Branch

Keeping a feature branch current with `master` (or `main`) is a routine task with several approaches — direct pull, fetch-then-merge, rebase, and the traditional branch-switching dance. This guide covers each, when to use it, how to preview incoming changes, and how to handle the conflicts that come up.

## Option 1: Direct Pull from Master (fewest steps)

```bash
# From your feature branch
git pull origin master
```

Fetches `master` from the remote and merges it into your current branch in one command — no branch switching. This creates a merge commit if your branch and `master` have both moved on.

One subtlety: this merges the *remote* `master` straight into your branch and leaves your **local** `master` ref untouched. If you rely on a current local `master` for other work, that's the reason to prefer Option 4, which updates local `master` first.

## Option 2: Fetch, Then Merge (more control)

```bash
git fetch origin master
git merge origin/master
```

Splitting the steps lets you inspect what's incoming before you merge. `git fetch` updates your `origin/master` remote-tracking ref without touching your working tree, so nothing changes until you run `git merge`.

## Option 3: Rebase Instead of Merge (linear history)

```bash
# From your feature branch
git pull --rebase origin master
```

Replays your branch's commits on top of the latest `master`, producing a linear history with no merge commit. Equivalent explicit form:

```bash
git fetch origin master
git rebase origin/master
```

Rebasing rewrites your branch's commit hashes. That's fine for a branch only you work on, but avoid it on branches others have already pulled unless the team agrees.

## Option 4: Traditional Branch-Switching (works, but extra steps)

```bash
git checkout master
git pull
git checkout Daniel_work
git merge master
```

This updates your local `master` first, then merges it into your feature branch. It works, but the direct pull in Option 1 achieves the same result without the checkout round-trip. It's mainly useful when you also want your local `master` kept current.

## Preview Incoming Changes First

Before merging or rebasing, see what's coming:

```bash
# Update remote-tracking refs without merging
git fetch origin

# Commits on master that you don't have yet
git log HEAD..origin/master --oneline

# Commits you have that master doesn't (your work)
git log origin/master..HEAD --oneline

# File-level diff of what would merge in
git diff HEAD...origin/master
```

## Merge vs Rebase — Which to Use

| | Merge | Rebase |
|---|-------|--------|
| History | Non-linear, keeps a merge commit | Linear, no merge commit |
| Commit hashes | Preserved | Rewritten |
| Safe on shared branches | Yes | Risky — avoid if others pulled it |
| Conflict handling | Resolved once, in one commit | May resolve per replayed commit |
| Context | Shows when integration happened | Reads as if built on latest master |

Rule of thumb: **rebase** for a clean, private feature branch you'll open a PR from; **merge** when the branch is shared or you want to preserve the exact integration history.

## Handling Merge Conflicts

If your changes overlap with updates on `master`, Git pauses and marks the conflicts:

```bash
# See which files conflict
git status

# Edit the files to resolve conflict markers (<<<<<<<, =======, >>>>>>>),
# then stage them
git add <resolved-file>
```

Finish the operation:

```bash
# If you merged
git commit

# If you rebased
git rebase --continue
```

Bail out and return to the pre-operation state if things go wrong:

```bash
# Abort a merge
git merge --abort

# Abort a rebase
git rebase --abort
```

## Tips

- The direct `git pull origin master` from your branch saves you the checkout dance in Option 4.
- On newer repos the default branch is `main` — substitute it for `master` throughout.
- Set `git config --global pull.rebase true` to make `git pull` rebase by default instead of merging.
- Rebasing a branch you've already pushed means you'll need `git push --force-with-lease` (safer than `--force`) to update the remote.
- Update frequently. Small, regular integrations produce fewer and smaller conflicts than one big merge at the end.
