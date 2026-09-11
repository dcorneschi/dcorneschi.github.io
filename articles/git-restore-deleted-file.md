# Restoring a Deleted File from a Git Repository

Recovering a deleted file depends on *when* it was deleted: is the deletion still uncommitted, or was it committed (and maybe pushed) some time ago? This guide walks each case — from the quick "I just deleted it" fix to hunting down a file removed dozens of commits back — and explains how to find the deleting commit when you don't remember the path.

> For the broader `git restore` command reference, see [Restoring Files with git restore](articles/git-restore-files.md).

## Case 1: Deleted but Not Yet Committed

If you deleted the file (with `rm` or `git rm`) but haven't committed the deletion, the file still lives in the last commit — just restore it from HEAD.

```bash
# Modern (Git 2.23+)
git restore path/to/file

# If the deletion was staged (e.g. git rm), restore staging + working tree
git restore --staged --worktree path/to/file

# Older equivalent
git checkout HEAD -- path/to/file
```

Check what's pending first if unsure:

```bash
git status            # shows "deleted: path/to/file"
```

## Case 2: Deleted in a Past Commit

Once the deletion is committed, HEAD no longer has the file, so you must restore it from **before** the deleting commit.

### Step 1 — Find the commit that deleted it

```bash
git log --diff-filter=D --summary -- path/to/file
```

- `--diff-filter=D` limits the log to commits that **deleted** files.
- `--summary` prints the `delete mode ...` lines so you can confirm.

The top result is the commit that removed the file. Note its hash.

### Step 2 — Restore from the commit just before deletion

```bash
git restore --source=<deleting-commit>^ -- path/to/file

# Older equivalent
git checkout <deleting-commit>^ -- path/to/file
```

The `^` means "the parent of that commit" — the last state that still contained the file. (The deleting commit itself doesn't have it.)

### Step 3 — Commit the recovery

The restored file is staged; commit it to bring it back for good:

```bash
git commit -m "Restore path/to/file"
```

If the deletion was already pushed, a normal `git push` publishes the recovery — no force needed, since you're adding a new commit rather than rewriting history.

## Finding the File When You Don't Know the Path

```bash
# Search every deletion across all branches for a filename
git log --diff-filter=D --summary --all | grep <filename>

# List all files ever deleted (with the commit before each)
git log --diff-filter=D --name-only --pretty=format:'%h %s'
```

Once you have the path and deleting commit, use the Case 2 restore command.

## Restore an Entire Deleted Directory

Point the same commands at the directory path:

```bash
git restore --source=<deleting-commit>^ -- path/to/dir/
# or
git checkout <deleting-commit>^ -- path/to/dir/
```

## Inspect Before Restoring

To see the file's contents at a commit without touching your working tree:

```bash
# Print the file as it was in the parent of the deleting commit
git show <deleting-commit>^:path/to/file

# Save it under a different name to compare
git show <deleting-commit>^:path/to/file > recovered.txt
```

## Which Command for Which Situation

| Situation | Command |
|-----------|---------|
| Deleted, not committed | `git restore path/to/file` |
| Deletion was staged (`git rm`) | `git restore --staged --worktree path/to/file` |
| Deleted in a past commit | `git restore --source=<commit>^ -- path/to/file` |
| Just want to view it | `git show <commit>^:path/to/file` |
| Whole deleting commit was bad, undo it | `git revert <commit>` |

`git revert <deleting-commit>` is an alternative for Case 2 when the deleting commit *only* removed files — it creates a new commit that reinstates them. Prefer the targeted restore when the deleting commit also made other changes you want to keep.

## Notes

- **The `^` is essential** for committed deletions — restoring from the deleting commit itself gives you "no file," since that commit is where it's already gone.
- **The `--` separator** tells Git the argument is a path, not a branch — always use it for deleted-file restores.
- **Recovery is a normal add + commit** — it never rewrites history, so it's safe on shared branches and needs no force push.
- If the file was **never committed at all**, Git can't recover it (nothing was ever tracked) — check your editor's local history or filesystem trash instead.

## Summary

- Not committed yet → `git restore path/to/file` (or `--staged --worktree` if the removal was staged).
- Committed → find the deleting commit with `git log --diff-filter=D --summary -- path`, then `git restore --source=<commit>^ -- path` and commit.
- Unknown path → `git log --diff-filter=D --summary --all | grep <name>`.
- View without restoring → `git show <commit>^:path`.
- Restoring is additive and safe to push; only a file that was never tracked is truly unrecoverable via Git.
