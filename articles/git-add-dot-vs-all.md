# Staging Changes in Git: git add . vs -A vs -u

`git add .` and `git add -A` both stage new, modified, and deleted files — but they differ in *scope* depending on where you run them. This guide explains the real difference, clears up a common misconception, and gives you a decision guide plus the other staging modes worth knowing.

## The Short Answer

| Command | New files | Modified files | Deleted files | Scope |
|---------|-----------|----------------|---------------|-------|
| `git add .` | Yes | Yes | Yes | Current directory and below |
| `git add -A` | Yes | Yes | Yes | Entire working tree |
| `git add -u` | No | Yes | Yes | Tracked files (respects pathspec) |

The key distinction is **scope**, not file type. In modern Git, both stage additions, modifications, and deletions — the difference is *which part of the tree* they look at.

## git add .

`git add .` stages all changes in the current directory and its subdirectories:

```bash
# Stage everything under the current directory
git add .
```

- Includes new (untracked), modified, and deleted files.
- The `.` is a pathspec — it means "here and below". If you run it from a subdirectory, changes elsewhere in the repo are left unstaged.
- More predictable when you only want to touch the area you're working in.

```bash
# From repo root — stages the whole tree
cd ~/project
git add .

# From a subdirectory — stages only src/ and below
cd ~/project/src
git add .        # changes in ~/project/docs are NOT staged
```

## git add -A (--all)

`git add -A` (long form `git add --all`) stages all changes across the entire working tree:

```bash
# Stage every change in the repository, regardless of cwd
git add -A
```

- Includes new, modified, and deleted files.
- Ignores your current directory — it always considers the whole repository.
- Equivalent to `git add --all` and, when run from the repo root with no pathspec, produces the same result as `git add .`.

```bash
# From a subdirectory — still stages the ENTIRE repo
cd ~/project/src
git add -A       # changes in ~/project/docs ARE staged
```

## The Version Gotcha (Why the "Difference" Is Often Misstated)

A widespread explanation says `git add .` skips file deletions while `git add -A` includes them. That was true in **Git before 2.0**, but it no longer is.

- **Git 1.x:** `git add .` staged new and modified files but *not* deletions. You needed `-A` or `-u` to stage removals.
- **Git 2.0 and later:** `git add .` stages new, modified, *and* deleted files within its pathspec, just like `-A`.

So on any current Git, the only real difference between `git add .` and `git add -A` is the **directory scope** described above. Check your version with:

```bash
git --version
```

## git add -u (--update)

A related mode that's easy to confuse with the two above:

```bash
# Stage modifications and deletions of ALREADY-TRACKED files only
git add -u
```

- Stages modified and deleted files.
- **Does not** stage new (untracked) files.
- Useful when you want to record edits and removals but not accidentally add new files or build artifacts.

## Side-by-Side Behavior

Assume you're at the repository root with these changes:

- `new.txt` — a brand new untracked file
- `edited.txt` — a tracked file you modified
- `gone.txt` — a tracked file you deleted

| Command | Stages `new.txt` | Stages `edited.txt` | Stages `gone.txt` |
|---------|:----------------:|:-------------------:|:-----------------:|
| `git add .` | Yes | Yes | Yes |
| `git add -A` | Yes | Yes | Yes |
| `git add -u` | No | Yes | Yes |

Run from a subdirectory instead, and `git add .` limits itself to that subtree while `git add -A` and `git add -u` still consider the whole repo (with `-u` limited to tracked files).

## Preview Before You Stage

Check what would be affected before committing to it:

```bash
# See working-tree status
git status

# Dry run — show what add would stage without staging it
git add -A --dry-run
git add . --dry-run

# Review the actual line changes not yet staged
git diff

# Review what is already staged
git diff --staged
```

## Which Should You Use?

- **`git add .`** — Best default. It's predictable and scopes to where you're working, which avoids sweeping up unrelated changes elsewhere in the repo.
- **`git add -A`** — Use when you deliberately want every change across the whole repository staged, no matter your current directory.
- **`git add -u`** — Use when you want to stage edits and deletions of tracked files but leave untracked files alone.
- **Specific paths** — When in doubt, stage explicit files (`git add src/app.js`) for the cleanest, most intentional commits.

```bash
# Stage a single file
git add path/to/file.txt

# Stage multiple specific files
git add file1.txt dir/file2.txt

# Interactively choose hunks to stage
git add -p
```

## Notes and Gotchas

- All of these respect `.gitignore` — ignored files are not staged unless you force them with `git add -f`.
- `git add -p` (patch mode) is the tool of choice when a file has several changes and you only want to commit some of them.
- If you staged too much, unstage with `git restore --staged <file>` (Git 2.23+) or `git reset HEAD <file>` on older versions.
- Empty directories are never staged — Git only tracks files.
