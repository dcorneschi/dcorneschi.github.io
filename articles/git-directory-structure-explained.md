# The .git Directory Structure Explained

When you run `git init`, Git creates a single hidden `.git` directory at the root of your project. Everything Git knows about your repository — its configuration, its history, its branches, its hooks — lives inside it. Delete it and you're left with plain files; back it up and you've backed up the entire repository. This guide walks through the anatomy of `.git`, explaining what each top-level file and folder is for.

## Creating the Directory

The `.git` directory appears the moment you initialize a repository:

```bash
git init myproject
cd myproject
ls -a .git
```

```text
.git/
├── config
├── description
├── HEAD
├── hooks/
├── info/
├── objects/
└── refs/
```

That's the standard skeleton Git lays down for a fresh repository. Each entry has a distinct job.

## Top-Level Overview

| Entry | Type | Purpose |
|-------|------|---------|
| `config` | File | Repository-specific configuration |
| `description` | File | Repo description (used by GitWeb only) |
| `HEAD` | File | Points to the branch/commit you're on |
| `hooks/` | Folder | Scripts that run on Git events |
| `info/` | Folder | Repo metadata, such as a local ignore file |
| `objects/` | Folder | The object store — all commits, trees, and blobs |
| `refs/` | Folder | Pointers to commits (branches and tags) |

## config

The `config` file holds settings that apply to this repository only. It's plain INI-style text and is where things like your remotes, branch tracking, and repo-local user settings are stored.

```bash
cat .git/config
```

```ini
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
[remote "origin"]
	url = git@github.com:user/repo.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
	remote = origin
	merge = refs/heads/main
```

Git resolves configuration in layers — system, then global (`~/.gitconfig`), then this repo-local `config`, with the most specific level winning:

```bash
git config --local user.email "me@example.com"     # writes to .git/config
git config --list --show-origin                    # see every level and its source
```

## description

The `description` file contains a one-line description of the repository. It's used **only** by GitWeb (Git's built-in web viewer) and is ignored by GitHub, GitLab, and everyday Git commands. For most repositories you can safely ignore it.

```bash
cat .git/description
# Unnamed repository; edit this file 'description' to name the repository.
```

## HEAD

`HEAD` is Git's "you are here" pointer — it records which branch or commit you currently have checked out. Normally it holds a symbolic reference to a branch:

```bash
cat .git/HEAD
# ref: refs/heads/main
```

When you check out a raw commit or tag, HEAD becomes *detached* and stores a commit hash directly instead of a `ref:` line. Nearly every command (`commit`, `reset`, `diff`, `log`) operates relative to HEAD.

## hooks/

The `hooks/` directory holds scripts that Git runs automatically at specific points in its workflow — before a commit, after a merge, before a push, and so on. On a fresh repo it's populated with `.sample` files that are inert until you remove the suffix and make them executable.

```bash
ls .git/hooks
```

```text
applypatch-msg.sample     pre-commit.sample
commit-msg.sample         pre-push.sample
post-update.sample        pre-rebase.sample
prepare-commit-msg.sample update.sample
```

```bash
# Activate a hook by removing the .sample suffix
mv .git/hooks/pre-commit.sample .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
```

Hooks are commonly used to run linters, enforce commit-message formats, or block bad pushes.

## info/

The `info/` directory stores repository metadata that doesn't belong in tracked files. Its most useful member is `exclude` — a local ignore list that works like `.gitignore` but is **not** committed or shared:

```bash
cat .git/info/exclude
```

```text
# git ls-files --others --exclude-from=.git/info/exclude
# Lines that start with '#' are comments.
*.log
scratch/
```

Use `.git/info/exclude` for patterns personal to your working copy — things you don't want to impose on everyone via the shared `.gitignore`.

## objects/

The `objects/` directory is the heart of Git: the **object store** where all your content actually lives. Every commit, every directory tree, and every file snapshot is stored here as a compressed, content-addressed object named by its SHA hash.

```bash
ls .git/objects
```

```text
info/
pack/
3a/
5f/
...
```

Git recognizes four object types:

| Object | Represents |
|--------|-----------|
| Blob | The contents of a single file |
| Tree | A directory listing (names → blobs/trees) |
| Commit | A snapshot pointing to a tree, plus metadata |
| Tag | An annotated tag pointing to an object |

Loose objects are stored in subdirectories named after the first two characters of their hash; over time Git compresses many of them into `.pack` files under `objects/pack/` to save space.

```bash
# Peek at any object by its hash
git cat-file -t 3a5f1c...   # show the type
git cat-file -p 3a5f1c...   # pretty-print the contents
```

## refs/

The `refs/` directory holds pointers to commits — the human-friendly names layered over the object store's hashes. Branches live under `refs/heads/`, tags under `refs/tags/`, and remote-tracking branches under `refs/remotes/`.

```bash
find .git/refs -type f
```

```text
.git/refs/heads/main
.git/refs/heads/feature-login
.git/refs/tags/v1.0.0
```

Each ref file simply contains the commit hash it points to:

```bash
cat .git/refs/heads/main
# 51dc6ecb327578cca503abba4a56e8c18f3835e1
```

So a branch is nothing more than a movable pointer to a commit. When you commit on `main`, Git writes a new object into `objects/` and updates `refs/heads/main` to point at it.

## How the Pieces Fit Together

The files and folders form a chain from "where you are" down to the actual content:

```text
HEAD                    →  refs/heads/main   →  a commit in objects/
(you are here)             (branch pointer)     (the snapshot)
```

```bash
cat .git/HEAD                       # ref: refs/heads/main
cat .git/refs/heads/main            # 51dc6ec... (the commit hash)
git cat-file -p 51dc6ec             # the commit object in objects/
```

- **config / description** configure and label the repository.
- **HEAD / refs** track *where* you are and *what* your branches and tags point to.
- **objects** stores *the actual data* — every version of every file.
- **hooks / info** extend and customize local behavior.

## Quick Reference

| Path | What it holds |
|------|---------------|
| `.git/config` | Repo-local configuration and remotes |
| `.git/description` | GitWeb-only repo description |
| `.git/HEAD` | The currently checked-out branch or commit |
| `.git/hooks/` | Event-triggered scripts (`pre-commit`, `pre-push`, ...) |
| `.git/info/exclude` | Local, uncommitted ignore rules |
| `.git/objects/` | All commits, trees, blobs, and packs |
| `.git/refs/heads/` | Local branch pointers |
| `.git/refs/tags/` | Tag pointers |
| `.git/refs/remotes/` | Remote-tracking branch pointers |

```bash
git rev-parse --git-dir      # print the path to the .git directory
git count-objects -vH        # summarize the object store's size
```

The `.git` directory is the entire repository — the working files beside it are just a convenient checkout. Once you can name what each entry does, commands that seemed to touch invisible internals become concrete: `git commit` writes to `objects/` and bumps a file in `refs/`, `git checkout` rewrites `HEAD`, and `git config` edits one plain text file.
