# Git Shell Functions and Aliases

Wrapping common Git sequences in shell functions saves keystrokes and enforces habits — one command to add-commit-push, spin up a repo, or clean merged branches. Drop these into your `~/.bashrc` or `~/.zshrc`, reload the shell, and they're available everywhere. This guide collects a practical set, grouped by purpose, with notes on what each does.

> These are shell wrappers around Git. For Git's own built-in aliases (`git config alias.*`), see the [Git Cheatsheet](articles/git-cheatsheet.md).

## Installing

Add the functions to your shell startup file and reload:

```bash
# Append to ~/.bashrc (bash) or ~/.zshrc (zsh), then:
source ~/.bashrc      # or: source ~/.zshrc
```

Each function checks for required arguments where it matters and prints a usage hint instead of running a broken command.

## Repository Setup

```bash
# Initialize a repo, first commit, add remote, and push — in one shot
# Usage: gitinit "First commit" git@github.com:user/repo.git
gitinit() {
    git init
    git add .
    git commit -m "$1"
    git branch -M main
    git remote add origin "$2"
    git push -u origin main
}
```

## Commit and Push

```bash
# Add all + commit (with a usage guard)
# Usage: gac "message"
gac() {
    [ -z "$1" ] && { echo "Usage: gac 'commit message'"; return 1; }
    git add -A && git commit -m "$1"
}

# Add all + commit + push to origin/main
# Usage: gpush "message"
gpush() {
    [ -z "$1" ] && { echo "Usage: gpush 'commit message'"; return 1; }
    git add -A && git commit -m "$1" && git push origin main
}

# Add all + commit + push to the current branch's upstream
# Usage: gcp "message"
gcp() {
    [ -z "$1" ] && { echo "Usage: gcp 'commit message'"; return 1; }
    git add -A && git commit -m "$1" && git push
}

# Amend the last commit without changing its message
gamend() {
    git commit --amend --no-edit
}

# Undo the last commit but keep the changes staged
gundo() {
    git reset --soft HEAD~1
}
```

> Note: `git add -A` (or `git add .`) stages everything, which is convenient but can sweep up files you didn't mean to commit. Glance at `git status` first, or stage specific files, when precision matters.

## Status, Log, and Diff

```bash
# Compact status with branch/ahead-behind info
gst() {
    git status -sb
}

# Last N commits as a graph (default 5)
# Usage: glg [count]
glg() {
    git log --oneline --graph --decorate -n "${1:-5}"
}

# Full branch tree across all refs
gtree() {
    git log --graph --oneline --all --decorate
}

# Files changed in the last commit
glast() {
    git diff --name-status HEAD~1 HEAD
}

# Diff of what's currently staged
gds() {
    git diff --staged
}

# Find commits by message across all refs
# Usage: gfind "search term"
gfind() {
    [ -z "$1" ] && { echo "Usage: gfind 'search term'"; return 1; }
    git log --all --grep="$1" --oneline
}

# Contributors ranked by commit count
gcontrib() {
    git shortlog -sn
}
```

## Branches

```bash
# Create and switch to a new branch
# Usage: gcb branch-name
gcb() {
    [ -z "$1" ] && { echo "Usage: gcb branch-name"; return 1; }
    git checkout -b "$1"
}

# Push the current branch to origin
gpo() {
    git push origin "$(git branch --show-current)"
}

# Pull with rebase (linear history)
gpr() {
    git pull --rebase
}

# Switch to the default branch and pull the latest
gup() {
    local main_branch
    main_branch=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
    git checkout "${main_branch:-main}" && git pull
}

# Delete local branches already merged (skips main/master/develop)
gclean() {
    git branch --merged | grep -v '\*\|main\|master\|develop' | xargs -r git branch -d
}

# Sync a fork's main with its upstream
gsync() {
    git fetch upstream && git checkout main && git merge upstream/main
}
```

## Stash

```bash
# Stash, optionally with a message
# Usage: gss ["message"]
gss() {
    if [ -z "$1" ]; then git stash; else git stash push -m "$1"; fi
}

gsa() { git stash apply; }   # apply the most recent stash
gsl() { git stash list; }    # list all stashes
```

## Rebase and Fixups

```bash
# Interactive rebase of the last N commits (default 3)
# Usage: greb [count]
greb() {
    git rebase -i HEAD~"${1:-3}"
}

# Create a fixup commit targeting a specific commit
# (later: git rebase -i --autosquash to fold it in)
# Usage: gfix <commit-hash>
gfix() {
    [ -z "$1" ] && { echo "Usage: gfix <commit-hash>"; return 1; }
    git commit --fixup="$1"
}
```

`gfix` pairs with an autosquash rebase — after creating fixup commits, run `git rebase -i --autosquash <base>` and Git orders and marks them to merge into their targets automatically.

## Function vs Git Alias — Which to Use

- **Git aliases** (`git config --global alias.co checkout`) extend the `git` command itself: `git co`. Best for simple command substitutions.
- **Shell functions** (above) can run *multiple* commands, take positional arguments, add usage checks, and combine Git with other tools. Best for multi-step workflows like add-commit-push.

## Notes and Cautions

- **Load order:** put these in `~/.bashrc`/`~/.zshrc` (or a sourced file like `~/.git-functions.sh`) so they load in every shell.
- **Name clashes:** short names like `gst`, `gup`, `gcp` may collide with existing aliases (e.g. Oh My Zsh's Git plugin already defines many `g*` aliases). Check with `type gst` before defining, and rename as needed.
- **`add -A` is broad:** the commit/push helpers stage everything; prefer explicit staging for anything sensitive.
- **`gpush` hardcodes `main`:** it always pushes to `origin main`; use `gcp`/`gpo` when you're on a different branch.

## Summary

- Shell functions turn multi-step Git sequences into one command with argument checks.
- Group them by purpose (setup, commit/push, branches, stash, rebase) and source them from your shell rc file.
- Prefer functions over Git aliases when a workflow needs multiple commands or arguments; use Git aliases for simple one-command shortcuts.
- Watch for name collisions with existing Git-plugin aliases, and be mindful that `add -A` stages everything.
