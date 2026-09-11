# Git Stash Guide

`git stash` temporarily shelves your uncommitted changes so you can switch context — fix an urgent bug, pull updates, or move work to another branch — then bring them back later. This guide covers the full workflow: saving, listing, inspecting, applying, handling conflicts, the partial/selective options, and recovering a dropped stash.

## What Stash Does

Stash pushes your working changes onto a LIFO stack and reverts the working tree to `HEAD`:

```text
Working directory  --git stash-->  stash stack (LIFO)
stash stack  --git stash pop-->  working directory
```

**Stashed by default:** modified tracked files and staged changes.
**Not stashed by default:** untracked (never-added) files and ignored files — use `-u`/`-a` for those.

## Saving Changes

```bash
git stash                     # stash tracked changes (staged + unstaged)
git stash push                # same, modern syntax
git stash push -m "message"   # stash with a description (preferred)
git stash save "message"      # older form (deprecated)
```

### Include untracked / ignored files

```bash
git stash -u                  # include untracked files (--include-untracked)
git stash -a                  # include untracked AND ignored (--all)
git stash push -u -m "with untracked"
```

### Stash specific files or hunks

```bash
git stash push -- src/app.js src/util.js   # only these files
git stash push -- "*.css"                   # pathspec match
git stash push -- src/                      # a directory
git stash push -p                           # interactively pick hunks (--patch)
```

### Staged vs unstaged only

```bash
git stash push --staged       # only staged changes (Git 2.35+); short: -S
git stash push --keep-index   # stash everything but LEAVE staged changes in the tree; short: -k
```

`--keep-index` is the tool for testing exactly what you've staged before committing.

## Listing and Inspecting

```bash
git stash list                # all stashes
# stash@{0}: WIP on main: abc1234 Last commit
# stash@{1}: On feature: def5678 Another commit

git stash show                # stat summary of the latest stash
git stash show -p             # full diff
git stash show --name-only    # just file names
git stash show -p 'stash@{1}' # diff of a specific stash
```

Reference syntax: `stash@{0}` is the most recent, `stash@{1}` the next, and so on (0-indexed).

## Applying: pop vs apply

```bash
git stash pop                 # apply the latest stash AND remove it from the stack
git stash apply               # apply but KEEP it on the stack
git stash pop 'stash@{2}'     # apply a specific stash and drop it
git stash apply 'stash@{2}'   # apply a specific stash, keep it
```

Use `apply` when you're unsure — it keeps the stash as a backup until you confirm everything's fine, then drop it manually.

## Removing Stashes

```bash
git stash drop                # remove the latest stash
git stash drop 'stash@{2}'    # remove a specific stash
git stash clear               # remove ALL stashes (irreversible — review first)
```

## Handling Conflicts

If `git stash pop` hits a conflict, the stash is **not** dropped — it's preserved so you don't lose it:

```bash
git stash pop
# resolve the conflict markers, then:
git add <resolved-files>
git stash drop                # drop it manually once resolved
```

The safer pattern is `git stash apply`, resolve, verify, then `git stash drop`.

## Create a Branch from a Stash

When a stash conflicts because the base moved, branch off the commit it was made from instead of fighting the merge:

```bash
git stash branch <branch-name>            # from the latest stash
git stash branch fix 'stash@{2}'          # from a specific stash
```

This checks out the commit where the stash was created, makes a new branch, applies the stash, and drops it if it applied cleanly.

## Common Workflows

```bash
# Quick context switch to fix an urgent bug
git stash push -m "feature: halfway done"
git switch main
# ...fix, commit, push...
git switch feature-branch
git stash pop

# Test only staged changes
git add <files>
git stash push --keep-index    # shelve unstaged, keep staged
# run tests, then:
git stash pop

# Pull with local changes in the way
git stash
git pull --rebase
git stash pop

# Move work started on the wrong branch
git stash
git switch -c correct-branch
git stash pop
```

## Export a Stash as a Patch

```bash
git stash show -p 'stash@{0}' > my-changes.patch
git apply my-changes.patch     # re-apply later, even elsewhere
```

## Recovering a Dropped Stash

A dropped stash becomes a dangling commit that survives until garbage collection — find and re-apply it:

```bash
# List dangling commits (stash entries among them)
git fsck --no-reflog | awk '/dangling commit/ {print $3}'

# Inspect a candidate, then apply it
git stash apply <commit-hash>
```

`git reflog` may also still list `refs/stash` history if the drop was recent.

## Troubleshooting

| Message | Cause / fix |
|---------|-------------|
| "No local changes to save" | Nothing tracked is modified; if you only have untracked files, use `git stash -u` |
| Conflict when popping | Stash is preserved — resolve, `git add`, then `git stash drop` |
| "Cannot apply to a dirty working tree" | Commit or stash current changes first, or `git reset --hard HEAD` before applying |

## Best Practices

- **Describe every stash:** `git stash push -m "auth validation WIP"` — a list of unlabeled `WIP` entries is useless later.
- **Prefer `apply` over `pop`** when conflicts are possible, so the stash stays as a safety net.
- **Add `-u`** when switching contexts so new files come along.
- **Keep the stack short** — apply or drop promptly.
- **Don't use stash as long-term storage.** It's short-lived scratch space; for anything you care about, commit to a branch — commits are far safer than a stash you might `clear`.

## Quick Reference

| Command | Purpose |
|---------|---------|
| `git stash` | Stash tracked changes |
| `git stash -u` / `-a` | Include untracked / everything |
| `git stash push -m "msg"` | Stash with a description |
| `git stash push -p` | Interactively stash hunks |
| `git stash push --staged` / `--keep-index` | Only staged / keep staged |
| `git stash list` | List stashes |
| `git stash show -p` | Show a stash's diff |
| `git stash pop` / `apply` | Apply and remove / apply and keep |
| `git stash drop` / `clear` | Remove one / remove all |
| `git stash branch <name>` | Create a branch from a stash |

Stash is for short-term shelving — for anything important, commit to a branch instead.
