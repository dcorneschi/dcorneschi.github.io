# Normalizing Line Endings with .gitattributes

Mixed CRLF/LF line endings cause phantom diffs, broken shell scripts, and "the whole file changed" noise when a Windows and a Linux contributor touch the same repo. A committed `.gitattributes` file fixes this once for everyone by telling Git how to normalize text and which files to leave alone as binary. This guide explains the syntax, a solid starter config, and how to apply it to an existing repo.

## Why .gitattributes Beats core.autocrlf

You *can* set line-ending behavior per machine with `git config core.autocrlf`, but that relies on every contributor configuring it correctly and consistently. `.gitattributes` lives **in the repo**, is committed, and applies to everyone automatically — it's the reliable, team-wide way to enforce line endings. When both are set, `.gitattributes` wins.

## A Solid Starter .gitattributes

Place this at the repository root:

```gitattributes
# Ensure all text files use LF line endings in the repo
* text=auto eol=lf

# Explicitly declare text files you want normalized
*.js    text eol=lf
*.json  text eol=lf
*.md    text eol=lf
*.py    text eol=lf
*.sh    text eol=lf
*.txt   text eol=lf
*.xml   text eol=lf
*.yml   text eol=lf
*.yaml  text eol=lf

# Files that are truly binary and must never be modified
*.png   binary
*.jpg   binary
*.jpeg  binary
*.gif   binary
*.ico   binary
*.mov   binary
*.mp4   binary
*.mp3   binary
*.pdf   binary
*.zip   binary
```

## What Each Directive Means

| Directive | Effect |
|-----------|--------|
| `text=auto` | Git auto-detects text vs binary and normalizes text to LF in the repo |
| `text eol=lf` | Force LF in the working tree too (not just storage) |
| `text eol=crlf` | Force CRLF in the working tree (for files that require it) |
| `binary` | Shorthand for `-text -diff` — never modify or diff as text |
| `-text` | Treat as binary; no line-ending conversion |

- **`* text=auto`** is the safety net: for any file, let Git decide and normalize text. Stored line endings become LF.
- **`eol=lf`** additionally pins the *checked-out* endings to LF, so even Windows checkouts get LF on disk — important for shell scripts and Docker files that break with CRLF.
- **Explicit `*.ext text eol=lf`** lines make intent obvious and override auto-detection for that type.
- **`binary`** protects images, archives, and media from being corrupted by line-ending or encoding conversion.

## eol=lf vs eol=crlf

Most projects want LF everywhere. Use `eol=crlf` only for files that Windows tooling requires with carriage returns:

```gitattributes
*        text=auto eol=lf
*.bat    text eol=crlf     # Windows batch files
*.ps1    text eol=crlf     # PowerShell, if your tooling needs CRLF
```

## Applying It to an Existing Repo

Adding `.gitattributes` doesn't retroactively fix files already committed with CRLF. Re-normalize once:

```bash
# 1. Add and commit the .gitattributes file
git add .gitattributes
git commit -m "Add .gitattributes for line-ending normalization"

# 2. Re-normalize all tracked files to the new rules
git add --renormalize .

# 3. Commit the (possibly large) normalization change on its own
git commit -m "Normalize line endings"
```

Doing the renormalize as a **separate commit** keeps it from tangling with real code changes, which makes review and `git blame` cleaner.

## Verifying

```bash
# See how Git classifies a specific file
git check-attr -a path/to/file
# e.g. -> path/to/file: text: auto   eol: lf

# Confirm a file has no CRLF endings
file path/to/script.sh
# ...ASCII text            (good)
# ...with CRLF line terminators  (needs renormalizing)
```

`git check-attr -a <file>` is the authoritative way to see which attributes actually apply to a path.

### List EOLs and Bulk-Convert CRLF Files

`git ls-files --eol` reports the line endings Git sees for every tracked file:

```bash
# Columns: index-eol  working-tree-eol  attr  filename
git ls-files --eol

# Convert any tracked files that still have CRLF to LF (needs dos2unix)
for f in $(git ls-files --eol | grep 'w/crlf' | awk '{print $NF}'); do
  dos2unix "$f"
done
```

After converting, commit the result — pair it with `git add --renormalize .` so the stored blobs match your `.gitattributes` rules.

You can also check the active `core.autocrlf` setting, which governs conversion when no attribute applies:

```bash
git config --global core.autocrlf   # global setting
git config core.autocrlf            # repo-specific
```

`core.autocrlf` values: `false` (no conversion), `input` (CRLF→LF on commit only), `true` (CRLF→LF on commit, LF→CRLF on checkout). A committed `.gitattributes` overrides it.

## Related Gotchas

- If `git diff` shows **"Binary files differ"** with no visible changes, an overly broad rule like `* binary` may be forcing text files to binary — see [Fixing "Binary files differ" in git diff](articles/git-diff-binary-files-differ.md).
- Shell scripts also need the **executable bit** recorded in Git alongside LF endings to run on Linux — see [Fixing Shell Script Execute Permissions Across Windows and Linux](articles/git-shell-script-executable-permissions.md).

## Summary

- Commit a `.gitattributes` with `* text=auto eol=lf` so line endings are normalized for everyone, independent of each machine's `core.autocrlf`.
- Add explicit `*.ext text eol=lf` lines for clarity and `*.ext binary` for media/archives.
- Use `eol=crlf` only for files that genuinely need CRLF (e.g. `.bat`).
- Run `git add --renormalize .` in a dedicated commit to fix files already committed with CRLF.
- Verify with `git check-attr -a <file>`.
