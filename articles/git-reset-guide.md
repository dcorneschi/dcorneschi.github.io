# git reset Guide: soft, mixed, and hard

`git reset` moves the current branch pointer to another commit and, depending on the mode, optionally updates the staging area and working tree. It's the workhorse for undoing local commits and unstaging files — and, in `--hard` form, one of the few Git commands that permanently destroys uncommitted work. This guide covers the three modes, file-level resets, recovery via reflog, and how to reset safely.

> For how `reset` compares with `revert`, `checkout`, and `restore`, see [Undoing Changes in Git: reset vs checkout vs revert](articles/git-undo-reset-checkout-revert.md).

## The Three Modes

```bash
git reset --soft  <commit>   # move HEAD; keep staging area and working tree
git reset --mixed <commit>   # move HEAD; reset staging area, keep working tree (DEFAULT)
git reset --hard  <commit>   # move HEAD; reset staging area AND working tree
```

Visualized, resetting `HEAD -> C3 -> C2 -> C1` back to `C1`:

```text
--soft   HEAD -> C1   staging: unchanged   working tree: unchanged
--mixed  HEAD -> C1   staging: cleared     working tree: unchanged
--hard   HEAD -> C1   staging: cleared     working tree: cleared
```

| Mode | HEAD | Staging area | Working tree |
|------|:----:|:------------:|:------------:|
| `--soft` | moves | unchanged | unchanged |
| `--mixed` (default) | moves | reset | unchanged |
| `--hard` | moves | reset | **reset (destructive)** |

## Resetting Commits

```bash
# Undo the last commit
git reset HEAD~1              # mixed: keep changes, unstaged
git reset --soft HEAD~1       # keep changes staged
git reset --hard HEAD~1       # discard the changes entirely

# Reset to a specific commit
git reset abc1234             # mixed
git reset --soft abc1234
git reset --hard abc1234

# Reset several commits at once
git reset --soft HEAD~3       # undo last 3, keep everything staged
git reset --hard HEAD~5       # undo last 5, discard everything
```

## Unstaging (the staging area)

A `--mixed` reset with a path only touches the index, which is how you unstage:

```bash
git reset <file>                 # unstage a file
git reset HEAD <file>            # same, explicit
git reset                        # unstage everything
git reset HEAD -- file1 file2    # unstage specific files
git reset -- "*.js"              # unstage by pathspec
git reset -- src/                # unstage a directory
git reset -p                     # interactively unstage hunks (--patch)
```

On Git 2.23+, `git restore --staged <file>` does the same thing with clearer intent.

## File-Level Reset to a Commit

```bash
# Unstage a file (reset its index entry to HEAD)
git reset HEAD -- <file>

# Reset a file's staged content to an older commit's version
git reset abc1234 -- config.json
git reset abc1234 -- src/*.js
```

Note this updates the **index** entry for the file; it doesn't overwrite the working-tree copy the way `git restore --source abc1234 <file>` does.

## Common Use Cases

### Redo the last commit message

```bash
git reset --soft HEAD~1
git commit -m "Better commit message"
```

### Squash several commits into one

```bash
git reset --soft HEAD~3       # undo 3 commits, keep all changes staged
git commit -m "Combined commit"
```

### Sync a local branch to the remote exactly

```bash
git fetch origin
git reset --hard origin/main  # local becomes identical to origin/main
git clean -fd                 # also remove untracked files, if you want a pristine tree
```

### Split a commit into pieces

```bash
git reset --soft HEAD~1       # undo the commit, keep it all staged
git reset HEAD <unwanted-file>  # unstage the part you want separate
git commit -m "First piece"
git add <unwanted-file>
git commit -m "Second piece"
```

## Destructive Warning

These permanently discard uncommitted work — there is no undo for changes that were never committed:

```bash
git reset --hard <commit>     # loses working-tree changes
git reset --hard HEAD~1       # loses the last commit's changes
git reset --hard origin/main  # loses local commits and changes
```

`--hard` also **rewrites history**, so never use it on commits already pushed to a shared branch — use [`git revert`](articles/git-undo-pushed-commit.md) there instead.

## Reset Safely

```bash
# Shelve changes first
git stash

# ...or bookmark the current state before a hard reset
git branch backup-$(date +%Y%m%d-%H%M%S)
git tag backup-before-reset

# Then reset; recover from the backup if needed
git reset --hard <commit>
```

Prefer `--soft`/`--mixed` when you want to keep the work, and check `git status` before any reset.

## Recovering from a Bad Reset

Reset moves the branch pointer, and the previous position lives in the reflog until garbage collection:

```bash
git reflog
# abc1234 HEAD@{0}: reset: moving to HEAD~1
# def5678 HEAD@{1}: commit: Add new feature   <- the commit you dropped
# ghi9012 HEAD@{2}: commit: Fix bug

# Put the branch back...
git reset --hard def5678

# ...or recover onto a new branch
git branch recovery def5678
```

This recovers **committed** state. Changes discarded by `git reset --hard` that were never committed cannot be recovered.

## Troubleshooting

**"Cannot reset — you have uncommitted changes"**

```bash
git stash
git reset <commit>
git stash pop
```

Or commit them to a throwaway commit first, then reset.

## Quick Reference

| Command | Purpose |
|---------|---------|
| `git reset HEAD~1` | Undo last commit, keep changes unstaged |
| `git reset --soft HEAD~1` | Undo last commit, keep changes staged |
| `git reset --hard HEAD~1` | Undo last commit, discard changes |
| `git reset <file>` | Unstage a file |
| `git reset` | Unstage everything |
| `git reset abc1234 -- <file>` | Reset a file's staged content to a commit |
| `git reset --hard origin/main` | Match the remote exactly (destructive) |
| `git reflog` | Find commits dropped by a reset |

Reset rewrites history — safe locally, dangerous on shared branches.
