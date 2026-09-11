# Canceling a git revert

"Cancel a revert" can mean three different things depending on where you are: aborting a revert that's stuck mid-conflict, undoing a revert commit you just made, or reverting the revert to bring the original changes back. Each has a different command, and the right one depends on whether the revert has been pushed. This guide walks through all three.

## First, Identify Your Situation

| Situation | What you want | Command |
|-----------|---------------|---------|
| Revert paused on a conflict | Cancel the in-progress operation | `git revert --abort` |
| Revert committed, not pushed | Remove the revert commit | `git reset --hard HEAD~1` |
| Revert committed and pushed | Bring the original changes back safely | `git revert <revert-hash>` |

## 1. Abort a Revert In Progress (conflict state)

When `git revert` hits a conflict it pauses and waits for you. To cancel the whole thing and return to the pre-revert state:

```bash
git revert --abort
```

This restores your working tree and index to exactly where they were before the revert started. Related states you may see:

```bash
git revert --continue    # finish after resolving conflicts (not cancel)
git revert --skip        # skip the current commit in a multi-commit revert
```

## 2. Undo a Completed Revert (not yet pushed)

If the revert already produced a commit and you haven't pushed it, remove that commit by moving the branch back one:

```bash
# See recent history to confirm the revert is the tip
git log --oneline -5

# Drop the revert commit
git reset --hard HEAD~1
```

Choose the reset mode based on whether you want to keep the revert's changes staged:

```bash
git reset --soft HEAD~1    # remove the commit, keep changes staged
git reset HEAD~1           # (--mixed, default) keep changes, unstaged
git reset --hard HEAD~1    # remove the commit and discard its changes
```

`--hard` is destructive — it discards uncommitted work as well. Use it only when you're sure.

## 3. Revert the Revert (already pushed)

If the revert is on a shared branch, **don't** rewrite history with reset. Instead, revert the revert — a new commit that re-applies the original changes:

```bash
# Find the revert commit's hash
git log --oneline

# Undo it with another revert
git revert <revert-commit-hash>
```

This is history-safe: no force push, nothing others have pulled gets rewritten. It's the correct way to "bring back" changes that a pushed revert removed.

## Recovering After a Reset

If you reset and then change your mind, the reflog still knows where the branch was:

```bash
git reflog                     # find the pre-reset position
git reset --hard <commit-hash> # restore it
```

This recovers committed state; changes discarded by `git reset --hard` that were never committed are gone.

## Which to Use

- **Mid-conflict** → `git revert --abort`. This is the most common "cancel" case.
- **Committed but local** → `git reset` (`--soft`/`--mixed` to keep the work, `--hard` to drop it).
- **Committed and pushed** → `git revert <revert-hash>` to avoid rewriting shared history.

For the broader picture of undoing commits (revert vs reset on pushed vs private branches), see [Undoing a Pushed Commit: revert vs reset](articles/git-undo-pushed-commit.md).

## Summary

- `git revert --abort` cancels a revert stuck on a conflict.
- `git reset --hard HEAD~1` removes a revert commit you haven't pushed (pick the mode to keep or drop changes).
- `git revert <revert-hash>` re-applies the original changes when the revert is already shared — no history rewrite.
- Overshot a reset? `git reflog` + `git reset --hard <hash>` gets committed state back.
