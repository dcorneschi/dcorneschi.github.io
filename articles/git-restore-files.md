# Restoring Files with git restore

`git restore` is the modern, purpose-built command for undoing changes to files — discarding uncommitted edits, unstaging, pulling a file from another commit, or recovering one that was deleted. It splits the overloaded old `git checkout` into a clearer tool. This guide covers each use, including how to hunt down and restore a file removed in a past commit.

## Discard Uncommitted Changes to a File

```bash
# Revert a file to its last committed (HEAD) state
git restore <file>

# Multiple files
git restore file1.txt src/file2.js

# Everything in the working tree
git restore .
```

This overwrites your working-tree changes with the committed version — the discarded edits are **not recoverable**, so be sure before running it.

## Restore a File from a Specific Commit

```bash
git restore --source <commit-hash> <file>

# Also works with branch names, tags, or relative refs
git restore --source main <file>
git restore --source HEAD~3 <file>
```

`--source` (short `-s`) chooses where the content comes from, instead of the default `HEAD`.

## Unstage a File (keep working-tree changes)

```bash
# Remove from the index but leave your edits intact
git restore --staged <file>
```

By default `git restore` acts on the **working tree**. Add `--staged` to act on the index instead, or both together to reset a file completely:

```bash
# Unstage AND discard working-tree changes (back to HEAD entirely)
git restore --staged --worktree <file>
# short: git restore -SW <file>
```

| Flags | Acts on | Effect |
|-------|---------|--------|
| `git restore <file>` | Working tree | Discard uncommitted edits |
| `git restore --staged <file>` | Index | Unstage, keep working-tree edits |
| `git restore -SW <file>` | Both | Reset the file entirely to HEAD |

## Restore a Deleted File

### 1. Find the commit that deleted it

```bash
git log --diff-filter=D --summary -- path/to/file
```

`--diff-filter=D` limits the log to commits that **deleted** files, and `--summary` shows the delete entries.

### 2. Restore from the commit just before deletion

```bash
git restore --source <commit-hash>^ -- path/to/file
```

The `^` means "the parent of that commit" — you grab the file as it existed right *before* it was removed. (The deleting commit itself no longer contains the file.)

### Don't remember the exact path?

```bash
# Search all deletions across all branches
git log --diff-filter=D --summary --all | grep <filename>
```

Once you spot the path and commit, use the restore command above.

## The -- Separator

The bare `--` before a path tells Git "everything after this is a file path, not a branch or option." Use it when a filename could be mistaken for a ref or when restoring deleted files:

```bash
git restore --source <hash>^ -- path/to/file
```

## restore vs checkout vs reset

`git restore` was introduced (Git 2.23) to disentangle jobs the old commands overloaded:

| Task | Modern | Older equivalent |
|------|--------|------------------|
| Discard file changes | `git restore <file>` | `git checkout -- <file>` |
| Restore file from a commit | `git restore -s <ref> <file>` | `git checkout <ref> -- <file>` |
| Unstage a file | `git restore --staged <file>` | `git reset HEAD <file>` |
| Switch branches | `git switch <branch>` | `git checkout <branch>` |

Both old and new commands still work; `restore`/`switch` are just clearer about intent, which reduces the classic "did I just switch branches or discard my file?" ambiguity of `git checkout`.

## Notes

- `git restore <file>` discards uncommitted work permanently — check `git diff <file>` first if unsure.
- `--source`/`-s` picks the source commit; default is `HEAD`.
- Default target is the working tree; `--staged`/`-S` targets the index; combine for both.
- For deleted files, restore from `<deleting-commit>^` — the parent — since the deleting commit no longer has the file.
- `--diff-filter=D` is the key to finding when and where a file was removed.
