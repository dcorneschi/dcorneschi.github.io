# Understanding HEAD in Git

`HEAD` is Git's "you are here" pointer — it tells Git which commit you're currently working on. Almost every everyday command operates relative to it. This guide explains what HEAD is, where it lives, the difference between attached and detached states, and the relative-reference syntax (`HEAD~`, `HEAD^`) you use to navigate history.

## What HEAD Is

HEAD is a pointer to the commit you currently have checked out. Usually it points to a branch name, and that branch points to a commit:

```text
HEAD → main → commit abc1234
```

So "where am I?" resolves through HEAD to a branch to a specific commit.

## Where HEAD Lives

HEAD is stored in the `.git/HEAD` file:

```bash
cat .git/HEAD
```

Attached to a branch, it holds a symbolic reference:

```text
ref: refs/heads/main
```

Detached, it holds a raw commit hash instead:

```text
9f8e7d6c5b4a3210fedcba9876543210abcdef12
```

## Attached HEAD (Normal State)

When HEAD is attached to a branch, each new commit moves **both** HEAD and the branch forward together:

```text
Before commit:   HEAD → main → C3 → C2 → C1
After commit:    HEAD → main → C4 → C3 → C2 → C1
```

```bash
git log --oneline -1
# abc1234 (HEAD -> main) Latest commit message
```

The `HEAD -> main` in the output is Git telling you HEAD is attached to `main`.

## Detached HEAD State

A detached HEAD points **directly at a commit** rather than a branch. This happens when you check out a specific commit, tag, or remote-tracking ref:

```bash
git checkout abc1234    # detached at a commit
git checkout v1.0.0     # detached at a tag
```

```text
Attached:   HEAD → main → C3
Detached:   HEAD → C3          (no branch involved)
```

Commits you make while detached belong to no branch. If you switch away without saving them, they become unreachable and are eventually garbage-collected.

### Saving Work from a Detached HEAD

```bash
# Before switching away — put your position on a branch
git switch -c my-new-branch     # or: git checkout -b my-new-branch

# Already switched away? Recover via the reflog
git reflog                      # find the commit hash
git branch recovery abc1234     # create a branch pointing at it
```

Modern Git prints a warning and the commit hash when you enter detached HEAD, precisely so you can recover it later with `git reflog`.

## Relative References

You rarely type full hashes — HEAD-relative syntax is how you point at nearby commits.

| Syntax | Meaning |
|--------|---------|
| `HEAD` | The current commit |
| `HEAD~` / `HEAD~1` | Parent (one commit back) |
| `HEAD~2` | Grandparent (two back) |
| `HEAD~n` | n commits back along the first-parent line |
| `HEAD^` | First parent (same as `HEAD~1` in linear history) |
| `HEAD^2` | Second parent (only meaningful on a merge commit) |

```bash
git show HEAD          # current commit
git show HEAD~1        # previous commit
git show HEAD~3        # three commits ago
git diff HEAD~2..HEAD  # changes over the last two commits
```

## HEAD~ vs HEAD^ — the Distinction

The two look similar but answer different questions:

- `~` walks **backward** along the first-parent lineage: `HEAD~3` is three commits back.
- `^` **selects which parent** at a single commit: `HEAD^1` is the first parent, `HEAD^2` the second.

For linear history with no merges, `HEAD~1` and `HEAD^` are identical. They differ at merge commits, which have two parents:

```text
        C5 (HEAD -> main, merge commit)
       /  \
      C4    C3 (feature)
```

```bash
HEAD^1     # first parent  — the branch you were on (C4)
HEAD^2     # second parent — the branch you merged in (C3)
HEAD^^^    # same as HEAD~3
HEAD^2~3   # second parent, then three back from there
```

## HEAD vs Branch vs Commit

```text
HEAD    = where you are right now
Branch  = a named pointer to a commit (advances with new commits)
Commit  = an immutable snapshot at a point in time (never moves)
```

```bash
git log --oneline -3
# abc1234 (HEAD -> feature-login) Add login form
# def5678 Add auth module
# ghi9012 (main) Initial setup
```

Here HEAD is on `feature-login` (at `abc1234`), while `main` still points at `ghi9012`.

## Common Uses of HEAD

```bash
# Undo the last commit, keep changes staged
git reset --soft HEAD~1

# Undo the last commit, discard the changes
git reset --hard HEAD~1

# Show all uncommitted changes vs the last commit
git diff HEAD

# Modify the commit HEAD points to
git commit --amend

# Create a branch at the current position
git branch new-feature HEAD

# Apply the commit from three positions back
git cherry-pick HEAD~3
```

## Inspecting Where HEAD Is

```bash
git rev-parse HEAD           # full commit hash
git rev-parse --short HEAD   # short hash
git log --oneline -1         # hash + message
git symbolic-ref HEAD        # the branch HEAD is attached to (errors if detached)
```

`git symbolic-ref HEAD` is a clean way to test attachment: it prints `refs/heads/<branch>` when attached and fails when detached.

## Practical Scenarios

```bash
# See what changed since the branch started (committed changes)
git diff main..HEAD

# Undo the last two commits but keep the code
git reset --soft HEAD~2

# Escape a detached HEAD back to a branch
git checkout -        # return to the previous branch
git switch main       # or go to a named branch

# Find where HEAD has been (recover lost work)
git reflog
```

## Quick Reference

| Command | What it does |
|---------|--------------|
| `cat .git/HEAD` | See where HEAD points |
| `git log --oneline -1` | Show the current HEAD commit |
| `git rev-parse HEAD` | Get HEAD's commit hash |
| `git symbolic-ref HEAD` | Show the attached branch (or error if detached) |
| `git show HEAD` | Show HEAD commit details |
| `git diff HEAD` | Uncommitted changes vs HEAD |
| `git reset HEAD~1` | Move HEAD back one commit |
| `git switch -c name` | Save a detached HEAD to a branch |
| `git reflog` | Find where HEAD has been |

HEAD is simply "where you are now" — internalize that, and the reset, diff, and rebase commands that reference it stop feeling mysterious.
