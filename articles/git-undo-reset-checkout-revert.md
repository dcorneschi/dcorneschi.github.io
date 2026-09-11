# Undoing Changes in Git: reset vs checkout vs revert

Undoing work in Git comes down to three commands that people constantly mix up: `git reset`, `git checkout` (or its modern split `restore`/`switch`), and `git revert`. They operate on different things and carry very different risk. This guide explains each, when to use it, and gives a decision tree so you pick the right one — especially the golden rule: **if you've pushed it, revert; if it's still local, reset.**

## Git's Three Trees

Every undo touches one or more of these:

1. **Working directory** — your actual files on disk.
2. **Staging area (index)** — what's staged for the next commit.
3. **Commit history** — the committed snapshots.

Which trees a command changes is exactly what separates `reset`, `checkout`, and `revert`.

## Quick Reference

| Command | Scope | Safety | Use case | History |
|---------|-------|--------|----------|---------|
| `git reset` | Commit / file | Unsafe on shared branches | Undo local commits, unstage files | Rewrites |
| `git checkout` | Commit / file | Can lose uncommitted changes | Switch branches, restore files | No |
| `git revert` | Commit only | Safe on shared branches | Undo pushed commits | Adds a new commit |

## 1. git reset — Rewind Locally

`reset` moves the branch pointer back and optionally changes the index and working tree. Its three modes differ only in how far that change reaches.

| Mode | Branch pointer | Staging area | Working tree |
|------|:-------------:|:------------:|:------------:|
| `--soft` | moved | kept | kept |
| `--mixed` (default) | moved | reset | kept |
| `--hard` | moved | reset | **discarded** |

```bash
# Undo the last commit, keep changes staged (e.g. to amend the commit)
git reset --soft HEAD~1

# Undo and unstage, keep the edits in the working tree (re-group commits)
git reset HEAD~1            # --mixed is the default

# Undo and throw everything away — destructive
git reset --hard HEAD~1
```

### Unstaging a file

```bash
git add config.json
git reset config.json       # unstage, keep the edit
```

`--hard` discards uncommitted work permanently, so reserve it for changes you truly don't want. For the full menu of removing a local commit, see [Removing a Local Commit: reset, revert, and rebase](articles/git-remove-local-commit.md).

## 2. git checkout — Switch and Restore

`checkout` is overloaded: it switches branches *and* restores files. Git 2.23+ split those jobs into `switch` and `restore` (clearer, and preferred).

### Switch branches

```bash
git checkout main           # switch to an existing branch
git checkout -b feature-new # create and switch
# modern equivalents:
git switch main
git switch -c feature-new
```

### View an old commit (detached HEAD)

```bash
git checkout abc123         # detached HEAD — good for inspecting, not committing
git switch main             # return to a branch
```

### Restore a file

```bash
git checkout -- index.html          # discard working-tree changes (old syntax)
git restore index.html              # modern equivalent
git checkout abc123 -- server.js    # restore a file from a specific commit
git restore --source abc123 server.js
```

For the full file-restore picture, see [Restoring Files with git restore](articles/git-restore-files.md).

## 3. git revert — Safe Public Undo

`revert` creates a **new** commit that inverts an earlier one. Nothing is rewritten, so it's the safe choice on shared branches.

```bash
# Undo a specific commit with a new inverse commit
git revert abc123

# Push normally — no force needed
git push origin main
```

Range and batching:

```bash
git revert abc123..xyz789   # revert a range (exclusive of the first)
git revert -n abc123        # stage the revert without committing, to combine changes
```

For the pushed-vs-private decision and force-push details, see [Undoing a Pushed Commit: revert vs reset](articles/git-undo-pushed-commit.md); to cancel a revert itself, see [Canceling a git revert](articles/git-cancel-revert.md).

## Real-World Scenarios

**Committed to the wrong branch (not pushed):**

```bash
git branch feature-branch    # bookmark the commit
git reset --hard HEAD~1      # remove it from the current branch
git switch feature-branch    # continue there
```

**Undo the last commit but keep the work:**

```bash
git reset HEAD~1             # keep changes unstaged
git reset --soft HEAD~1      # keep changes staged
```

**Pushed bad code:**

```bash
# Not: git reset --hard + force push  (breaks teammates)
git revert HEAD
git push origin main
```

**Discard all uncommitted changes:**

```bash
git reset --hard HEAD        # everything
git restore broken-file.js   # a single file
```

**Test an earlier state, then come back:**

```bash
git checkout abc123
# ...test...
git switch main
```

## Decision Tree

```text
Have you already pushed/shared the commits?
├─ YES → git revert            (new inverse commit, safe for teams)
└─ NO  → Keep your changes?
         ├─ YES → git reset --soft / --mixed
         └─ NO  → git reset --hard   (permanently deletes changes)

Working with a file, not a commit?
├─ Unstage it       → git restore --staged <file>   (or git reset <file>)
└─ Discard edits    → git restore <file>             (or git checkout -- <file>)
```

## Common Mistakes

- **Resetting a shared branch and force-pushing.** Rewriting pushed history forces everyone to reconcile — use `git revert` instead.
- **Reaching for `checkout` when you mean `restore`.** On Git 2.23+, `git restore <file>` (discard) and `git restore --staged <file>` (unstage) are unambiguous; `git switch` handles branches.
- **`git reset --hard` without thinking.** It deletes uncommitted work with no prompt; if you overshoot a commit reset, `git reflog` can recover the committed state.

## Modern Command Map (Git 2.23+)

| Old | New |
|-----|-----|
| `git checkout <branch>` | `git switch <branch>` |
| `git checkout -b <branch>` | `git switch -c <branch>` |
| `git checkout -- <file>` | `git restore <file>` |
| `git reset <file>` (unstage) | `git restore --staged <file>` |

## Summary

- **`git reset`** rewinds commits locally — `--soft`/`--mixed`/`--hard` control how much is kept vs discarded.
- **`git checkout`** (now `switch`/`restore`) navigates branches and restores files; it doesn't rewrite history.
- **`git revert`** safely undoes pushed commits by adding an inverse commit.
- Golden rule: pushed → `revert`; local → `reset` is fine. Recover overshoots with `git reflog`.
