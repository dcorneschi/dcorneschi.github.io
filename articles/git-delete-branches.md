# Deleting Git Branches: Local, Remote, and Cleanup

A practical reference for removing Git branches — safe vs force deletes, removing remote branches, pruning stale tracking references, bulk cleanup of merged branches, and the errors you'll hit along the way.

## Delete a Local Branch

```bash
# Safe delete — only succeeds if the branch is fully merged
git branch -d branch-name

# Force delete — removes the branch even if it isn't merged
git branch -D branch-name
```

`-d` refuses to delete a branch that has commits not merged into its upstream or the current branch, which protects you from losing work. `-D` is shorthand for `--delete --force` and skips that check.

## Delete a Remote Branch

```bash
# Modern syntax
git push origin --delete branch-name

# Older colon syntax (still works — "push nothing to this ref")
git push origin :branch-name
```

Deleting the remote branch does not remove your local copy, and vice versa — local and remote deletions are independent operations.

## Delete Both Local and Remote

```bash
git branch -d branch-name && git push origin --delete branch-name
```

The `&&` runs the remote delete only if the local delete succeeds. Swap in `-D` if you intend to force the local delete regardless of merge state.

## List Branches

```bash
# All branches — local and remote-tracking
git branch -a

# Local branches only
git branch

# Remote-tracking branches only
git branch -r

# Show last commit and upstream for each branch
git branch -vv
```

## Clean Up Stale Remote-Tracking References

When a branch is deleted on the remote, your local remote-tracking references (like `origin/feature-x`) don't disappear automatically. Prune them:

```bash
# Fetch updates and remove stale remote-tracking refs
git fetch --prune

# Prune without fetching
git remote prune origin

# Preview what a prune would remove (no changes made)
git remote prune origin --dry-run
```

To make pruning automatic on every fetch, set it once:

```bash
git config --global fetch.prune true
```

## Common Workflow

```bash
# 1. Switch away from the branch you want to delete
git switch main        # or: git checkout main

# 2. Delete the local branch
git branch -d feature-branch

# 3. Delete the remote branch
git push origin --delete feature-branch

# 4. Clean up stale remote-tracking references
git fetch --prune
```

You cannot delete the branch you're currently on — step 1 avoids the "cannot delete branch checked out at ..." error.

## Delete Multiple Branches

```bash
# Multiple local branches
git branch -d branch1 branch2 branch3

# Multiple remote branches
git push origin --delete branch1 branch2 branch3
```

## Delete All Merged Branches

Remove every local branch already merged into the current branch, skipping the current branch and common long-lived branches:

```bash
git branch --merged | grep -v '\*' | grep -vE '^\s*(main|master|develop)$' | xargs -r -n 1 git branch -d
```

- `git branch --merged` lists branches whose tips are reachable from HEAD.
- `grep -v '\*'` drops the current branch (marked with `*`).
- `grep -vE '(main|master|develop)'` protects long-lived branches.
- `xargs -r -n 1 git branch -d` deletes them one at a time (`-r` skips running if the list is empty).

Run the pipeline without the final `xargs` first to preview exactly what will be deleted:

```bash
git branch --merged | grep -v '\*' | grep -vE '^\s*(main|master|develop)$'
```

## Rename Instead of Delete

If you meant to rename rather than remove:

```bash
# Rename the current branch
git branch -m new-name

# Rename a specific branch
git branch -m old-name new-name
```

## Recovering a Deleted Branch

A deleted branch is just a removed pointer — the commits usually survive until garbage collection. Recover with the reflog:

```bash
# Find the tip commit of the deleted branch
git reflog

# Recreate the branch at that commit
git branch recovered-branch <commit-sha>
```

## Troubleshooting

**Error: "The branch 'x' is not fully merged"**
- The branch has commits not merged anywhere Git can see. Use `-D` to force delete, or merge the branch first if you want to keep the work.

**Error: "Cannot delete branch 'x' checked out at ..."**
- You're on the branch you're trying to delete. Switch away first with `git switch main`.

**Error: "remote ref does not exist"**
- The remote branch was already deleted. Run `git fetch --prune` to sync your local references.

**Stale `origin/*` branches still showing after remote deletion**
- Run `git fetch --prune` or `git remote prune origin`. Enable `fetch.prune=true` to automate it.
