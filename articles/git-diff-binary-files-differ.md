# Fixing "Binary files differ" in git diff

`git diff` sometimes reports that files differ but shows no line-by-line changes — just `Binary files a/file and b/file differ`. This happens when Git classifies the file as binary, either by content detection or a `.gitattributes` rule. This guide explains why Git makes that call, how to see the real diff now, and how to fix the underlying cause for good.

## The Symptom

```text
$ git diff
diff --git a/notes.md b/notes.md
index 1a2b3c4..5d6e7f8 100644
Binary files a/notes.md and b/notes.md differ
```

The change is real, but Git refuses to render a textual diff because it thinks the file is binary.

## Quick Ways to See the Diff Now

These are workarounds — they show the diff without changing how the file is stored.

```bash
# Force Git to diff the content as text
git diff --text

# Ignore carriage-return differences at end of line (CRLF vs LF noise)
git diff --ignore-cr-at-eol

# Just list which files changed
git diff --name-only

# High-level summary (mode changes, renames, etc.)
git diff --summary

# Open the change in your configured GUI diff tool
git difftool
```

## Why Git Thinks a Text File Is Binary

Git's binary detection is not the same as the `file` command. Git treats content as binary when any of these hold:

1. **A NUL byte (`0x00`) appears** in the scanned portion of the file (Git checks the first several KB).
2. **A `.gitattributes` rule marks it binary**, such as `*.md binary` or a blanket `* binary`.
3. **The blob content looks non-text** by Git's heuristic (unusual control characters, encoding artifacts).

So even a file that `file` reports as "ASCII text" can trip Git's detection if an editor or bad encoding introduced NUL bytes or control characters, or if an overly broad attribute rule is in place.

## Diagnose the Cause

### Check for NUL bytes or stray control characters

```bash
# Look for NUL (00) bytes in the file
hexdump -C notes.md | grep ' 00 '

# grep also flags binary content
grep -qP '\x00' notes.md && echo "contains NUL bytes"

# Show the detected encoding
file notes.md
```

### Check what attributes Git is applying

```bash
# See the current attributes file
cat .gitattributes

# Ask Git exactly how it classifies a specific path
git check-attr -a notes.md

# Example output when a rule forces binary:
# notes.md: binary: set
```

`git check-attr` is the fastest way to confirm whether an attribute rule — not the content — is the culprit.

## Fix 1: A .gitattributes Rule Is Forcing Binary

The most common cause is an overly broad rule. A line like `* binary` or `* binary eol=lf` tells Git to treat **every file** in the repo as binary, text files included.

Replace it with auto-detection:

```bash
# Overwrite with a sane cross-platform default
echo "* text=auto eol=lf" > .gitattributes

# Re-apply attributes to all tracked files
git add --renormalize .

# The diff should now render as text
git diff
```

For a single file type or path instead of a global rule:

```bash
# Treat all Markdown as text
echo "*.md text" >> .gitattributes

# Or a single file
echo "notes.md text" >> .gitattributes

git add .gitattributes
git add --renormalize .
git diff
```

`git add --renormalize .` rewrites the stored blobs according to the new attributes so future diffs are clean — do this after any `.gitattributes` change.

## Fix 2: The File Actually Contains NUL Bytes

If diagnosis shows real NUL bytes (not just an attribute rule), the file content itself is the problem — often from a save in the wrong encoding (e.g., UTF-16) or a corrupt write.

```bash
# Strip NUL bytes into a clean copy
tr -d '\000' < notes.md > notes.clean.md
mv notes.clean.md notes.md

# Or, if it was saved as UTF-16, convert to UTF-8
iconv -f UTF-16 -t UTF-8 notes.md -o notes.utf8.md
mv notes.utf8.md notes.md

git add notes.md
git diff --cached
```

Marking such a file as `text` in `.gitattributes` only hides the symptom — clean the content so editors and tooling behave correctly.

## The Recommended .gitattributes Baseline

For a cross-platform repository, this is the safe default:

```text
* text=auto eol=lf
```

What each part means:

- `*` — applies to all files.
- `text=auto` — let Git auto-detect text vs binary instead of forcing either.
- `eol=lf` — normalize line endings to LF in the working tree.

Avoid `* binary eol=lf`: the `binary` macro expands to `-text -diff`, which disables text handling and diffs for everything.

## Line Endings by Operating System

| OS | Line ending | Attribute |
|----|-------------|-----------|
| Linux / macOS | LF (`\n`) | `eol=lf` |
| Windows | CRLF (`\r\n`) | `eol=crlf` |

Storing LF in the repository with `text=auto eol=lf` keeps history consistent across the team regardless of contributor OS. If a specific file must keep CRLF (some Windows tooling requires it), scope that with a targeted rule like `*.bat text eol=crlf`.

## Summary

1. Confirm the cause with `git check-attr -a <file>` and a NUL-byte check (`hexdump`/`grep`).
2. If a `.gitattributes` rule forces binary, replace `* binary` with `* text=auto eol=lf`.
3. Run `git add --renormalize .` after any attribute change so stored blobs match.
4. If the content truly has NUL bytes, clean or re-encode the file rather than masking it.
5. Use `git diff --text` as a temporary way to view the diff while you sort out the root cause.
