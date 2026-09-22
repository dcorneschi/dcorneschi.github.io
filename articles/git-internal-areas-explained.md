# Git's Internal Areas: Working Directory, Staging, Commit History, and the Local Repository

When you run `git init`, Git creates a hidden `.git` directory that quietly tracks everything about your project. Understanding the areas Git manages — the working directory, the staging area, the commit history, and the local repository as a whole — makes the everyday commands (`add`, `commit`, `status`) stop feeling like magic. This guide walks through each area, where it physically lives, and how a change flows from an edited file to a permanent commit.

## The Big Picture

A Git project moves changes through three main states before they become permanent history:

```text
Working Directory  →  Staging Area (Index)  →  Commit History
   (your files)         (git add)               (git commit)
```

All of this is coordinated by the **local repository** — the `.git` directory sitting at the root of your project.

```text
your-project/
├── .git/            ← the local repository (Git's data lives here)
│   ├── index        ← the staging area (created after first git add)
│   ├── objects/     ← the commit history and all snapshots
│   ├── refs/        ← branch and tag pointers
│   └── HEAD         ← which commit you're on
├── src/
└── README.md        ← working directory files
```

## Working Directory

The working directory is simply the set of files and folders you see and edit — your project as it exists on disk, outside the `.git` directory. This is where you write code, create files, and make changes.

Files in the working directory are in one of a few states:

| State | Meaning |
|-------|---------|
| Untracked | Git isn't watching this file yet |
| Modified | Tracked file changed since the last commit |
| Staged | Change has been added to the staging area |
| Unmodified | Matches the last commit exactly |

```bash
git status          # see what's modified, staged, or untracked
git diff            # see unstaged changes in the working directory
```

Editing a file only changes the working directory. Git doesn't record anything permanent until you move that change forward.

## Staging Area (the Index)

The staging area — also called the **index** — is where you assemble the exact set of changes that will go into your next commit. It sits between the working directory and the commit history, letting you decide precisely what to include.

You add changes to the staging area with `git add`:

```bash
git add README.md       # stage a single file
git add .                # stage everything in the current directory
```

### The index File

The staging area is stored in a single file: `.git/index`.

Here's a detail that trips people up: **the index file doesn't exist until you stage something for the first time.** In a freshly initialized project where you haven't run `git add` on anything yet, you won't find an `index` file inside `.git`. The moment you add your first file to the staging area, Git creates it.

```bash
git init rainbow
cd rainbow
ls .git/index            # no such file yet

echo "hello" > colors.txt
git add colors.txt
ls .git/index            # now it exists
```

This is why the staging area is sometimes described lazily as "the index file" — the file is the physical form of the staging area.

## Commit History

A **commit** in Git is one version of a project — think of it as a snapshot, a standalone version that references all the files belonging to it at that moment. Every time you commit, that snapshot is saved into the commit history.

```bash
git commit -m "Add colors file"
```

### Commit Hashes

Every commit has a **commit hash** (sometimes called a commit ID): a unique 40-character string of letters and numbers that acts as the commit's name and lets you refer to it.

```text
51dc6ecb327578cca503abba4a56e8c18f3835e1
```

In practice you only need the first seven characters to refer to a commit, because that prefix is almost always unique within a repository:

```text
51dc6ec
```

```bash
git log --oneline        # shows short hashes + messages
git show 51dc6ec         # inspect a commit by its short hash
```

### Where the History Lives

The commit history is represented by the `objects` directory inside `.git`:

```text
.git/objects/
```

Every commit you make is saved here. Understanding the history in depth means diving into Git's internals — a complex topic you don't need for learning the basics. For everyday use, the key takeaway is simple: **each commit is a saved snapshot, and all of them are stored in the commit history.**

## Local Repository

The **local repository** is the whole package: it's the `.git` directory that Git creates and maintains at the root of your project. It contains all four pieces working together:

```text
Local Repository (.git/)
├── Staging Area   → .git/index
├── Commit History → .git/objects/
├── Branch/Tag refs→ .git/refs/
└── HEAD pointer   → .git/HEAD
```

The working directory is your project files on disk; the local repository is Git's private bookkeeping alongside them. Together they let you track, stage, commit, and travel through the full history of your project — all locally, with no server required.

## How a Change Flows Through the Areas

Putting it all together, here's the lifecycle of a single edit:

```text
1. Edit a file            → change lives in the Working Directory
2. git add <file>         → change moves to the Staging Area (.git/index)
3. git commit -m "..."    → snapshot saved to Commit History (.git/objects)
```

```bash
# 1. Make a change in the working directory
echo "blue" >> colors.txt

# 2. Stage it
git add colors.txt

# 3. Commit it into history
git commit -m "Add blue to colors"

# Inspect the result
git log --oneline -1
# 51dc6ec Add blue to colors
```

## Quick Reference

| Area | Physical Location | Populated By | Purpose |
|------|-------------------|--------------|---------|
| Working Directory | Your project files on disk | Editing files | Where you make changes |
| Staging Area (Index) | `.git/index` | `git add` | Assemble the next commit |
| Commit History | `.git/objects/` | `git commit` | Permanent snapshots |
| Local Repository | `.git/` | `git init` / clone | Contains all of the above |

| Command | What it does |
|---------|--------------|
| `git init` | Create the local repository (`.git`) |
| `git status` | Show working directory and staging state |
| `git diff` | Show unstaged working-directory changes |
| `git add <file>` | Move a change into the staging area |
| `git commit -m "..."` | Save the staged snapshot to history |
| `git log --oneline` | List commits with short hashes |
| `git show <hash>` | Inspect a specific commit |

Once you can picture where each change lives — the working directory, the index, and the object store inside `.git` — the core Git workflow becomes a simple story of moving a snapshot forward one deliberate step at a time.
