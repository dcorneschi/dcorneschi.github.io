# Syncing a Branch Across Multiple Computers

Working on the same branch from more than one machine — a laptop and a desktop, say — is smooth as long as you follow one habit: **pull before you start, push when you finish.** Skip the pull and your next push gets rejected because the branch moved on the other machine. This guide covers the workflow, why `git pull --rebase` keeps the history clean for a solo branch, and how to handle the conflicts that occasionally come up.

## The Core Workflow

### Before starting work

```bash
git pull origin feature-branch
```

This brings in whatever you pushed from the other computer, so you're building on the latest state.

### After making changes

```bash
git add .
git commit -m "Your commit message"
git push origin feature-branch
```

## Why Pull First

If you skip the pull, your local branch is behind the remote, and the push is rejected:

```text
! [rejected]        feature-branch -> feature-branch (fetch first)
```

You'd then have to pull anyway — except now with local commits in play, which is exactly when merge conflicts appear. Pulling *first* avoids that.

## git pull vs git pull --rebase

Both integrate the remote changes; they differ in the shape of the resulting history.

### Default pull (merge)

```bash
git pull origin feature-branch
```

Creates a merge commit joining the remote and local lines:

```text
Remote:  A---B---C
Local:   A---D---E
Result:  A---D---E---M   (merge commit M)
              \     /
               B---C
```

### pull --rebase

```bash
git pull --rebase origin feature-branch
```

Replays your local commits on top of the remote ones for a linear history:

```text
Remote:  A---B---C
Local:   A---D---E
Result:  A---B---C---D'---E'   (linear)
```

### Merge vs rebase at a glance

| | Merge (default) | Rebase |
|---|-----------------|--------|
| Merge commit | Yes | No |
| History | Preserves exact timing | Linear, easier to read |
| Commit hashes | Unchanged | Rewritten (D→D′) |
| Best for | Shared branches | Personal branches, cross-machine sync |

For a solo branch you're syncing between your own computers, **rebase** is usually the better fit — it avoids a trail of merge commits and keeps `git log` readable.

### When rebase is (and isn't) appropriate

Good for:

- Personal branches synced across your own machines.
- Keeping a feature branch's history clean before it's shared.

Avoid for:

- Commits others have already pulled and built on.
- `main`/`master` or public branches with multiple contributors.

Because rebase rewrites hashes, rebasing commits other people already have forces them to reconcile — fine for a branch only you touch, disruptive otherwise.

## Making Rebase the Default

```bash
# Just this branch
git config branch.feature-branch.rebase true

# All branches, globally
git config --global pull.rebase true
```

With that set, a bare `git pull` rebases instead of merging.

## Handling Conflicts During a Rebase

```bash
# After editing the conflicted files:
git add <conflicted-files>
git rebase --continue

# Or bail out and return to the pre-rebase state:
git rebase --abort
```

Conflicts during rebase are resolved per replayed commit, so you may go through `--continue` more than once on a longer branch.

## Set Up Branch Tracking

So plain `git pull`/`git push` target the right remote branch:

```bash
# When you first push a new branch
git push -u origin feature-branch

# For an existing local branch
git branch --set-upstream-to=origin/feature-branch
```

After this, `git pull` and `git push` work with no arguments.

## Pro Tips

- **Pull first, always** — especially the first thing when you sit down at the other computer.
- **Use `--rebase`** on a personal cross-machine branch for a clean, linear history.
- **Commit before switching machines** so nothing is left uncommitted and stranded; a `git stash` works for quick, unfinished work but commits are safer.
- Set upstream tracking once so the argument-free commands just work.

## Quick One-Liner

```bash
git pull --rebase && git add . && git commit -m "message" && git push
```

## Summary

- The habit: `git pull` before work, `git add`/`commit`/`push` after.
- Skipping the pull gets your push rejected — pulling first avoids conflict pileups.
- `git pull --rebase` keeps a solo cross-machine branch linear; make it the default with `pull.rebase true`.
- Reserve rebase for personal/unshared branches; on shared branches, prefer a normal merge pull.
