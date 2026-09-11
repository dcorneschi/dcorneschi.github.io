# Fixing Shell Script Execute Permissions Across Windows and Linux

Shell scripts authored or edited on Windows arrive on Linux without the execute bit (`+x`), so `./script.sh` fails with "Permission denied." The fix has two parts: setting the executable bit **in a way Git records**, and normalizing line endings so the script actually runs. This guide covers both, plus how to make the fix stick for everyone who clones the repo.

## Why It Happens

Windows filesystems don't carry the Unix execute permission, so scripts created there land on Linux as non-executable. Two separate problems often show up together:

1. **Missing execute bit** — `chmod +x` locally fixes it, but unless Git records the bit, teammates hit the same issue.
2. **CRLF line endings** — Windows editors save `\r\n`; a `#!/bin/bash\r` shebang makes Linux fail with a confusing `bad interpreter` or `^M` error even when the file *is* executable.

Solving it properly means addressing both.

## Option 1: chmod (local quick fix)

```bash
# One directory
chmod +x scripts/eks/*.sh

# All .sh files, recursively
find scripts -name "*.sh" -type f -exec chmod +x {} \;
```

This fixes your local checkout only. If `core.fileMode` is true, Git will notice the mode change and you can commit it — but the cleaner way to record it is Option 2.

## Option 2: Record the Executable Bit in Git

Git stores a file's executable bit as part of its mode (`100644` non-exec vs `100755` exec). Set it directly in the index with `--chmod` so it's committed regardless of your local filesystem:

```bash
# Mark specific scripts executable in Git's index
git add --chmod=+x scripts/eks/*.sh
git commit -m "Make scripts executable"

# All shell scripts in the repo
find scripts -name "*.sh" -type f -exec git add --chmod=+x {} \;
git commit -m "Make all scripts executable"
```

`git add --chmod=+x` works even on Windows where `chmod` has no effect — it edits the stored mode, so everyone who clones gets an executable script.

### A note on core.fileMode

```bash
# Track file-permission changes
git config core.fileMode true

# Ignore them (useful on filesystems that misreport modes)
git config core.fileMode false
```

Setting `core.fileMode false` is handy when a filesystem (or a Windows mount) reports spurious permission changes that clutter `git status`. Prefer `git add --chmod` over relying on `core.fileMode` to *set* the bit.

## Option 3: Normalize Line Endings with .gitattributes

Even an executable script fails if it has CRLF endings. Force LF for shell scripts via `.gitattributes` at the repo root:

```bash
echo "*.sh text eol=lf" >> .gitattributes
git add .gitattributes
git commit -m "Force LF line endings for shell scripts"
```

`text eol=lf` guarantees `.sh` files are stored and checked out with Unix line endings on every platform. To apply it to files already committed with CRLF, re-normalize:

```bash
git add --renormalize .
git commit -m "Renormalize line endings"
```

## Option 4: A Setup Script

For repos where recording the bit isn't practical, ship a helper that fixes permissions after clone:

```bash
#!/bin/bash
# make-executable.sh — make all shell scripts executable
find scripts -name "*.sh" -type f -exec chmod +x {} \;
echo "All shell scripts are now executable"
```

```bash
chmod +x make-executable.sh
./make-executable.sh
```

## Verifying

```bash
# Check the filesystem permission — look for the x bits
ls -l scripts/eks/eks-check-addons-version.sh
# -rwxr-xr-x  ...   <- the x means executable

# Check what Git has recorded (100755 = executable, 100644 = not)
git ls-files -s scripts/eks/eks-check-addons-version.sh

# Confirm the file has no CRLF endings
file scripts/eks/eks-check-addons-version.sh
# ...ASCII text  (good)  vs  ...with CRLF line terminators (bad)
```

`git ls-files -s` is the authoritative check — it shows the mode Git will hand to everyone who clones, independent of your local filesystem.

## Best Practice for a Cross-Platform Repo

1. Add `.gitattributes` with `*.sh text eol=lf` so scripts always get LF endings.
2. Record the executable bit in Git: `find scripts -name "*.sh" -type f -exec git add --chmod=+x {} \;`
3. Commit both changes together.
4. Verify with `git ls-files -s` that scripts show mode `100755`.
5. Note in the README that scripts are executable, so contributors don't re-break the bit.

This combination — LF endings recorded in `.gitattributes` plus the executable bit stored in Git — makes scripts work on Windows, macOS, and Linux without per-clone fixups.
