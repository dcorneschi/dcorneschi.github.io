# Removing Untracked Files with git clean

`git clean` deletes untracked files from your working tree — build artifacts, generated output, stray files from a checkout. It's genuinely destructive: cleaned files are **not** recoverable through Git because they were never committed. This guide covers a safe workflow, every relevant flag, and the guardrails worth knowing before you run it.

## Always Preview First

```bash
# Dry run — list what WOULD be deleted, delete nothing
git clean -n

# Same thing, long form
git clean --dry-run
```

Make previewing a habit. Because cleaned files bypass the reflog and object store, there's no `git` command to bring them back.

## Remove Untracked Files

```bash
# Delete untracked files in the current directory and below
git clean -f
```

`-f` (force) is required by default — Git refuses to delete otherwise unless `clean.requireForce` is set to `false`.

## Remove Untracked Files and Directories

```bash
git clean -fd
```

Plain `git clean -f` skips untracked *directories*. Add `-d` to remove them too.

## Include Ignored Files

```bash
# Untracked files, directories, AND .gitignore-d files
git clean -fdx

# Remove ONLY ignored files, keep other untracked files
git clean -fdX
```

Note the case distinction:

- `-x` removes ignored files **in addition to** other untracked files.
- `-X` (capital) removes **only** ignored files, leaving other untracked files alone.

`-fdx` is the "nuke it back to a pristine checkout" option — it wipes local config, `.env` files, and editor artifacts that live in `.gitignore`. Use with care.

## Flags Reference

| Flag | Meaning |
|------|---------|
| `-n` / `--dry-run` | Show what would be deleted, delete nothing |
| `-f` / `--force` | Actually perform the deletion (required by default) |
| `-d` | Include untracked directories |
| `-x` | Also remove ignored files |
| `-X` | Remove **only** ignored files |
| `-i` / `--interactive` | Choose interactively what to remove |
| `-e <pattern>` | Exclude paths matching the pattern from cleaning |

## Interactive Mode

When you want to review and pick items rather than delete in bulk:

```bash
git clean -di
```

This opens a menu where you can list, filter by pattern, select items, or confirm each deletion — a safer middle ground between a dry run and a blanket `-fd`.

## Cleaning a Specific Path or Excluding Files

```bash
# Clean only within a subdirectory
git clean -fd path/to/dir

# Clean but keep files matching a pattern
git clean -fdx -e "*.env" -e ".vscode/"
```

`-e` adds exclude patterns on top of your `.gitignore`, letting you wipe most untracked content while preserving specific files.

## Recommended Workflow

```bash
# 1. Preview everything that would go
git clean -nd

# 2. If it looks right, remove untracked files and directories
git clean -fd

# 3. Only if you also want ignored files gone (pristine tree)
git clean -fdx
```

## git clean vs git reset vs git checkout

These operate on different sets of changes — knowing which is which prevents accidents:

| Command | Acts on | Recoverable? |
|---------|---------|--------------|
| `git clean` | **Untracked** files | No — files were never in Git |
| `git restore` / `git checkout -- <file>` | **Tracked** files with uncommitted edits | The discarded edits are lost, but the committed version remains |
| `git reset` | Staging area / branch pointer | Usually, via reflog |

A common full reset to a clean committed state combines two commands:

```bash
# Discard tracked-file modifications...
git reset --hard

# ...then remove untracked files and directories
git clean -fd
```

## Cautions

- **Not undoable via Git.** Untracked files removed by `git clean` are gone. Preview with `-n` first, every time.
- `-x` deletes ignored files too, which often includes secrets (`.env`), local config, and dependency folders. Double-check before using it.
- Run from the repository root (or pass an explicit path) so you clean the scope you intend — `git clean` is relative to your current directory.
- Set `clean.requireForce = false` only if you fully understand the risk; leaving `-f` mandatory is a useful safety net.
