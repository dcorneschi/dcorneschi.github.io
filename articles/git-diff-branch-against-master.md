# Comparing Your Branch Against Master with git diff

`git diff` answers "what's different?" — but *which* difference depends on exactly how you invoke it. Comparing against master with a bare name, two dots, or three dots gives you different results, and mixing in staged/unstaged state adds more variants. This guide lays out what each form compares, how to save the output, and how to revert files back to master's version.

## What Each Diff Compares

| Command | What it shows |
|---------|---------------|
| `git diff master` | Everything different from master (committed **and** uncommitted) |
| `git diff master..HEAD` | Only your committed changes vs master |
| `git diff master...HEAD` | Your committed changes since the branches diverged (from the merge base) |
| `git diff` | Only unstaged changes in the working tree |
| `git diff --cached` | Only staged changes (after `git add`) |

Quick guide to intent:

- **What did my branch introduce (commits only)?** → `git diff master..HEAD` (or `...` from the merge base).
- **What haven't I committed yet?** → `git diff` (unstaged) or `git diff --cached` (staged).
- **Full picture, commits + uncommitted, vs master?** → `git diff master`.

## Seeing the Changes

### All changes vs master

```bash
git diff master
```

Shows every line-by-line difference between your working tree and master — staged and unstaged modifications to tracked files included.

```diff
diff --git a/src/app.js b/src/app.js
index 3a4b5c6..7d8e9f0 100644
--- a/src/app.js
+++ b/src/app.js
@@ -10,6 +10,8 @@ function init() {
   const config = loadConfig();
+  const logger = setupLogger();
+  logger.info('App starting');
   startServer(config);
 }
```

Lines prefixed `+` are additions, `-` deletions; the `@@` header marks the affected line numbers.

### Committed changes only

```bash
git diff master..HEAD
```

Compares master's tip to your latest commit, ignoring anything uncommitted in your working tree.

### Only your branch's work (from the merge base)

```bash
git diff master...HEAD
```

The three-dot form diffs from the **merge base** — where your branch diverged — so it shows only what *you* changed, even if master moved on afterward. For "what does my PR add?", this is usually the most accurate.

## Pairing with the Commit Log

```bash
# Which commits are on your branch but not master
git log master..HEAD

# Same, with each commit's full diff
git log -p master..HEAD
```

`git log -p` is the most complete view: every commit message followed by its patch.

## Filtered and Summary Views

```bash
# Just the names of changed files
git diff master --name-only

# File names with a status letter (M/A/D/R)
git diff master --name-status

# Compact per-file insertion/deletion summary
git diff master --stat

# Limit the diff to one file
git diff master -- src/app.js

# Compare any two branches (not just master)
git diff branch-a..branch-b
```

`--name-status` letters: `M` modified, `A` added, `D` deleted, `R` renamed.

## Staged vs Unstaged

```bash
# Unstaged working-tree changes
git diff

# Staged changes (what your next commit will contain)
git diff --cached      # --staged is a synonym
```

## Saving Diff Output to a File

```bash
# A plain diff
git diff master > diff.txt

# Commit log, then the full diff appended
git log master..HEAD > changes.txt && git diff master >> changes.txt

# The most complete export: every commit with its patch
git log -p master..HEAD > changes.txt
```

If your goal is to *apply* these changes elsewhere rather than just read them, prefer a proper patch (`git format-patch master`) over a redirected diff.

## Reverting Files to Master's State

To discard your local changes to specific files and restore master's version:

```bash
# One or more files
git checkout master -- path/to/file1 path/to/file2

# A single file
git checkout master -- package.json
```

This overwrites those files in your working tree with master's version **and stages them**, leaving your other changes untouched. On Git 2.23+ the clearer equivalent is:

```bash
git restore --source master --staged --worktree package.json
```

Because this discards uncommitted work in the named files, double-check with `git diff master -- <file>` before running it — there's no undo for the overwritten changes.

## Two Dots vs Three Dots (the recurring gotcha)

For `git diff`, the dots mean:

- `master..HEAD` (two dots) — diff between the two tips directly.
- `master...HEAD` (three dots) — diff from the **merge base** to HEAD (your branch's own changes).

Note this is the opposite emphasis from `git log`, where three dots means the *symmetric* set of commits unique to either side. When in doubt for a branch review diff, three dots (`...`) against master is the safe choice.

## Quick Reference

| Command | Shows |
|---------|-------|
| `git diff master` | All changes (committed + uncommitted) vs master |
| `git diff master..HEAD` | Committed changes vs master |
| `git diff master...HEAD` | Your branch's changes from the merge base |
| `git diff` / `git diff --cached` | Unstaged / staged changes |
| `git diff master --stat` | Per-file change summary |
| `git diff master --name-only` | Changed file names |
| `git log -p master..HEAD` | Commits with diffs |
| `git checkout master -- <file>` | Restore a file to master's version |
