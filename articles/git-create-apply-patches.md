# Creating and Applying Git Patch Files

Patch files let you capture changes as portable text — to save work in progress, share a fix without pushing, or email a series of commits. Git offers two families of patch: `git diff` (plain change patches) and `git format-patch` (patches carrying full commit metadata). This guide covers creating both, every useful option, and how to apply them.

## diff vs format-patch: Pick the Right Tool

| | `git diff` | `git format-patch` |
|---|-----------|--------------------|
| Contains | Raw line changes only | Commit(s) with author, date, message |
| Output | One file (stdout redirect) | One file **per commit** |
| Applied with | `git apply` | `git am` |
| Best for | WIP, quick change sharing | Sharing commits, mailing patch series |

## Creating Patches with git diff

### Unstaged changes

```bash
git diff > changes.patch
```

### Staged changes

```bash
git diff --cached > staged-changes.patch   # --staged is a synonym
```

### Everything (staged + unstaged) against HEAD

```bash
git diff HEAD > all-changes.patch
```

### Between branches

```bash
git diff main..feature-branch > branch-diff.patch
```

### Specific files

```bash
# One file
git diff path/to/file.txt > file-changes.patch

# Several files
git diff file1.txt file2.txt > multiple-files.patch
```

## Creating Patches with git format-patch

These produce mailbox-format files (one per commit, numbered) that preserve authorship and commit messages.

```bash
# The most recent commit
git format-patch -1 HEAD

# The last 3 commits (creates 0001-*.patch, 0002-*.patch, 0003-*.patch)
git format-patch -3

# A range of commits
git format-patch <from-commit>..<to-commit>

# Every commit on this branch not on main
git format-patch main

# All commits by a specific author, numbered
git format-patch --author="John Doe" -n
```

Handy output options:

```bash
# Write patches into a directory
git format-patch -3 -o patches/

# Emit a single combined file instead of one per commit
git format-patch main --stdout > series.patch

# Add a cover-letter (0000-*.patch) summarizing the series
git format-patch -3 --cover-letter
```

## Diff Formatting Options

These apply to `git diff` (and many to `format-patch`):

```bash
# More context lines around each change (default is 3)
git diff -U10 > patch-with-context.patch

# Include binary file changes (otherwise shown as "Binary files differ")
git diff --binary > binary-patch.patch

# Just a summary of changed files and line counts
git diff --stat > stats-patch.patch

# Drop the a/ and b/ path prefixes
git diff --no-prefix > no-prefix.patch

# Ignore whitespace-only changes
git diff -w > ignore-whitespace.patch
```

`--binary` matters: a normal `git diff` patch cannot recreate binary changes, so include it when the change touches images or other non-text files.

## Applying Patches

### Patches made with git diff → git apply

```bash
# Apply to the working tree
git apply changes.patch

# Apply AND stage the changes in one step
git apply --index changes.patch

# Apply in reverse (undo a previously applied patch)
git apply -R changes.patch
```

### Patches made with git format-patch → git am

```bash
# Apply and re-create the commit(s) with original metadata
git am series.patch

# Apply every numbered patch in a directory
git am patches/*.patch
```

`git am` recreates commits (author, date, message intact); `git apply` only changes files without committing.

### Test Before Applying

```bash
# Does this patch apply cleanly? (changes nothing)
git apply --check changes.patch

# Show how many lines/files it would touch
git apply --stat changes.patch

# Verbose per-file report
git apply --summary changes.patch
```

### When a Patch Doesn't Apply Cleanly

```bash
# Apply what applies, leave .rej files for the conflicts
git apply --reject changes.patch

# Try harder to locate context that shifted
git apply -3 changes.patch        # 3-way merge using blob info

# For git am, resolve conflicts then continue — or bail out
git am --continue
git am --abort
```

The `-p` level controls how many leading path components to strip, which matters when the patch was made from a different directory depth:

```bash
git apply -p0 no-prefix.patch     # patch has no a/ b/ prefixes
```

## Practical Examples

```bash
# Save timestamped work in progress
git diff > "wip-$(date +%Y%m%d).patch"

# Patch of just the last commit
git format-patch -1 HEAD

# Difference between two branches, custom name
git diff main..develop > feature-differences.patch

# Email a 3-commit series with a cover letter
git format-patch -3 --cover-letter -o outgoing/
```

## Best Practices

- Use `git format-patch` when the recipient should get your commits (metadata preserved); use `git diff` for quick, informal change sharing.
- Always run `git apply --check` before applying a patch you didn't create.
- Add `--binary` whenever the change includes non-text files.
- Give patches descriptive names so their contents are obvious later.
- Generate and apply patches from the repository root so path prefixes line up; otherwise adjust with `-p`.
