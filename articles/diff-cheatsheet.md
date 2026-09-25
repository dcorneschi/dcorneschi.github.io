# diff Cheatsheet

`diff` compares two files (or directory trees) line by line and reports what changed. Its output can be read by humans or fed to `patch` to reproduce the changes elsewhere — which is why the **unified** format (`-u`) is the lingua franca of patches and code review. This cheatsheet covers the formats, the whitespace/case options, directory comparison, and how to read the output.

For applying diffs as patches in a repo, see [Creating and Applying Git Patch Files](articles/git-create-apply-patches.md); for stream editing, the [sed Cheatsheet](articles/sed-cheatsheet.md).

## Basic Usage

```sh
diff file1 file2          # default (ed-style) output
diff -u file1 file2       # unified format — the one you usually want
diff -c file1 file2       # context format
diff -y file1 file2       # side-by-side columns
```

`diff` reports how to change `file1` **into** `file2`: the first file is the "old"/`-`/`<` side, the second is the "new"/`+`/`>` side. Order matters.

## Output Formats

```sh
diff -u file1 file2       # unified: - / + lines, compact (patches, code review)
diff -c file1 file2       # context: !/-/+ with surrounding lines
diff -y file1 file2       # side-by-side, | marks differing lines
diff -n file1 file2       # RCS format
diff -e file1 file2       # ed script
```

Unified is the default choice: it's compact, `patch`-compatible, and what Git and review tools use.

## Ignoring Differences

```sh
diff -i file1 file2       # ignore case
diff -w file1 file2       # ignore ALL whitespace
diff -b file1 file2       # ignore changes in amount of whitespace
diff -B file1 file2       # ignore blank-line-only changes
diff -E file1 file2       # ignore tab expansion differences
diff -Z file1 file2       # ignore trailing whitespace

# combine them
diff -iw file1 file2      # ignore case and whitespace together
```

`-w` is the heavy hammer (ignores whitespace entirely); `-b` is gentler (only ignores *changes in amount*). Reach for `-b` when indentation style differs but structure shouldn't.

## Reporting Only (Quiet)

```sh
diff -q file1 file2       # print only "Files X and Y differ" (or nothing)
diff -s file1 file2       # also report when files are identical
```

`-q` (aka `--brief`) is ideal in scripts — combine with the exit code (below) rather than parsing output.

## Directory Comparison

```sh
diff -r dir1 dir2                    # recurse into subdirectories
diff -rq dir1 dir2                   # recursive, names of differing files only
diff -ru dir1 dir2                   # recursive unified (a directory patch)
diff -r --exclude='*.log' dir1 dir2  # skip files matching a glob
diff -rN dir1 dir2                   # treat missing files as empty (see below)
```

`diff -rq` is the fastest way to see *which* files differ between two trees without the line-level detail.

## Context Amount

By default unified/context diffs show 3 lines of surrounding context. Change it:

```sh
diff -U 5 file1 file2      # unified with 5 lines of context
diff -C 5 file1 file2      # context format with 5 lines
diff -U 0 file1 file2      # no context — changed lines only
```

## Advanced Options

```sh
diff -N old new            # treat an absent file as empty (needed for new files in patches)
diff -a file1 file2        # force text comparison (diff files it thinks are binary)
diff -x PATTERN -r a b     # exclude files matching PATTERN (repeatable)
diff --exclude-from=FILE -r a b   # exclusion patterns from a file
```

> Use `-N` when generating a patch that **adds or removes whole files** — without it, `diff` skips files that exist on only one side, and `patch` won't create/delete them.

## Reading the Output

### Unified format (-u)

```text
--- file1    2023-01-01 12:00:00
+++ file2    2023-01-01 12:30:00
@@ -1,3 +1,3 @@
 line1
-old line
+new line
 line3
```

- `---` / `+++` — the old and new files.
- `@@ -1,3 +1,3 @@` — the **hunk header**: old file lines 1–3, new file lines 1–3.
- `-` — line removed from the first file · `+` — line added in the second · a leading space — unchanged context.

### Default (ed-style) format

```text
1c1
< old line
---
> new line
```

- `1c1` — line 1 **c**hanged to line 1. Other codes: `a` (added), `d` (deleted), e.g. `3a4` or `2,4d1`.
- `<` — content from the first file · `>` — content from the second.

## Making and Applying Patches

Unified diff is the standard patch format:

```sh
diff -u original.txt modified.txt > changes.patch   # create a patch
patch original.txt < changes.patch                  # apply it
patch -R original.txt < changes.patch               # reverse (undo) it

# a whole-tree patch
diff -ruN old_dir new_dir > tree.patch
patch -p1 < tree.patch                              # apply from inside the tree
```

`patch -p1` strips one leading path component, which matches the `a/`…`b/` prefixes Git-style patches use. See [Creating and Applying Git Patch Files](articles/git-create-apply-patches.md) for the Git workflow.

## Exit Codes

`diff`'s exit status is how scripts test for differences — don't parse the output:

| Code | Meaning |
|------|---------|
| `0` | Files are identical |
| `1` | Files differ |
| `2` | Trouble (e.g. a file couldn't be read) |

```sh
if diff -q a b >/dev/null; then echo "same"; else echo "different"; fi
```

## Notes and Gotchas

- **Argument order matters:** `diff old new` describes turning `old` into `new`. Swap them and every `+`/`-` flips.
- **`-w` vs `-b`:** `-w` ignores whitespace entirely; `-b` only ignores *changes in amount*. Pick deliberately.
- **`-N` for patches that add/remove files** — otherwise one-sided files are silently skipped.
- **Exit code 1 is not an error** — it just means "they differ". Only `2` indicates a real problem, which trips up `set -e` scripts (guard with `|| true` or test explicitly).
- **`diff` is line-oriented.** For word- or character-level diffs use `git diff --word-diff`, `wdiff`, or `dwdiff`. For three-way merges, use `diff3` or `git merge-file`.

## Quick Reference

| Option | Description |
|--------|-------------|
| `-u` | Unified format (patches, review) |
| `-c` | Context format |
| `-y` | Side-by-side (`-W N` sets width) |
| `-r` | Recurse into directories |
| `-q` | Brief — only report *if* files differ |
| `-s` | Report identical files too |
| `-i` | Ignore case |
| `-w` | Ignore all whitespace |
| `-b` | Ignore whitespace *amount* changes |
| `-B` | Ignore blank-line changes |
| `-N` | Treat missing files as empty |
| `-a` | Force text comparison |
| `-x PAT` | Exclude files matching PAT |
| `-U N` / `-C N` | N lines of context |

```sh
diff -u a b > changes.patch          # create a patch
diff -ruN old/ new/ > tree.patch     # whole-tree patch
diff -rq dir1 dir2                   # which files differ
diff -iw a b                         # ignore case + whitespace
diff -y -W 120 a b                   # side-by-side, 120 cols
diff -q a b && echo same             # scriptable identity check
```

For related material, see [Creating and Applying Git Patch Files](articles/git-create-apply-patches.md) and the [sed Cheatsheet](articles/sed-cheatsheet.md).
