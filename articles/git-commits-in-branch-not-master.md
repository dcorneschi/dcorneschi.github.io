# Showing Commits in Your Branch That Aren't in Master

Before opening a pull request or rebasing, it's useful to see exactly which commits your branch adds on top of `master` (or `main`). Git's range syntax makes this a one-liner — but the two-dot and three-dot forms mean different things, and mixing them up is a common source of confusion. This guide covers the commands and what each range actually selects.

## The Quick Answer

```bash
# Commits on your branch but not on master
git log --oneline master..HEAD
```

`master..HEAD` reads as "reachable from HEAD but not from master" — precisely the work your branch adds.

## Listing the Commits

```bash
# Full commit log for the range
git log master..HEAD

# Concise one-line-per-commit summary (most common)
git log --oneline master..HEAD

# Include changed-file stats per commit
git log --stat master..HEAD

# HEAD is implied, so this is equivalent to the first command
git log master..
```

## Seeing the Actual Code Changes

Here's the key distinction between the dot forms:

```bash
# THREE dots: the diff introduced by your branch since it diverged
git diff master...HEAD
```

- `git diff master...HEAD` (three dots) diffs from the **merge base** — the point where the branches diverged — to HEAD. This shows only *your* changes, ignoring anything that landed on master after you branched.
- `git diff master..HEAD` (two dots, or just `git diff master HEAD`) diffs the two tips directly, so it also reflects master-only changes as reversed differences. Usually not what you want for "what did my branch change."

For reviewing your branch's contribution, prefer the three-dot form.

## Two Dots vs Three Dots — The Gotcha

The dots mean **opposite things** depending on whether you're using `git log` or `git diff`:

| Form | `git log` | `git diff` |
|------|-----------|------------|
| `master..HEAD` (two dots) | Commits on HEAD not on master | Direct tip-to-tip diff |
| `master...HEAD` (three dots) | Commits on **either** side but not both (symmetric) | Diff from the **merge base** to HEAD |

So for commits, two dots is what you want; for a diff of your branch's work, three dots is what you want. It's the reverse of what many people assume.

## cherry: Which Commits Are Already Upstream

```bash
# Show commits, marking which are already in master
git cherry -v master
```

`git cherry` compares by patch content, not just commit hash. Each line is prefixed:

- `+` — the commit is **not** yet in master (still unique to your branch).
- `-` — an equivalent change **is** already in master (e.g., cherry-picked or applied separately).

This is more accurate than a plain log when commits were cherry-picked, because it detects equivalent patches even under different hashes.

## Visualizing the Divergence

```bash
# Symmetric view — commits unique to each side, as a graph
git log --oneline --graph master...HEAD

# Include all refs for full context
git log --oneline --graph --decorate --all
```

The three-dot range in `git log` (symmetric difference) shows commits unique to *both* branches, which is handy for seeing how far each side has moved apart. Add `--left-right` to mark which side each commit belongs to:

```bash
git log --oneline --left-right master...HEAD
# <  commit only on master (left)
# >  commit only on HEAD (right)
```

## Counting and Naming

```bash
# How many commits ahead of master is your branch?
git rev-list --count master..HEAD

# Just the files your branch touched
git diff --name-only master...HEAD

# Ahead/behind counts in one line
git rev-list --left-right --count master...HEAD
# Output: "<behind>   <ahead>"
```

## Comparing Against the Remote

If you want your branch measured against the remote's master rather than a possibly-stale local copy, fetch first and use the remote-tracking ref:

```bash
git fetch origin
git log --oneline origin/master..HEAD
git diff origin/master...HEAD
```

## Summary

- **List your commits:** `git log --oneline master..HEAD` (two dots).
- **Diff your branch's work:** `git diff master...HEAD` (three dots, from the merge base).
- **Check what's already upstream:** `git cherry -v master`.
- **Count how far ahead:** `git rev-list --count master..HEAD`.
- Remember the dots flip meaning between `log` and `diff` — two dots for the commit list, three dots for the branch diff.
