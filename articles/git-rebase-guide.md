# git rebase Guide

`git rebase main` replays your branch's commits on top of the latest `main`, producing a clean, linear history with no merge commit. It's the go-to for tidying a private feature branch before it merges. This guide covers what rebase actually does step by step, interactive rebase, resolving conflicts, and the one rule you must not break.

> For a side-by-side comparison with merging, see [Git Merge vs Rebase](articles/git-merge-vs-rebase.md).

## How It Works

Before rebase — `feature` diverged from `main` after commit A:

```text
main:      A---B---C
            \
feature:     D---E---F
```

After `git rebase main` — your commits are replayed on top of `C` as new commits:

```text
main:      A---B---C
                    \
feature:             D'---E'---F'
```

## Step by Step

Running `git rebase main` from your feature branch:

1. **Finds the common ancestor** of your branch and `main` (commit A).
2. **Sets aside** your commits since that point (D, E, F).
3. **Moves your branch tip** to the latest `main` (commit C).
4. **Replays your commits** one at a time on top of C.
5. **Creates new commits** (D′, E′, F′) — same changes, but **new hashes**.

That last point is why rebase "rewrites history": the original D/E/F are replaced, not reused.

## How It Differs from Merge

- **Merge** adds a merge commit, preserves the original timeline, and shows when branches diverged and rejoined — the graph looks like a tree.
- **Rebase** adds no merge commit, rewrites your commits onto a new base, and yields a linear history that reads as if you started from the latest `main`.

## When to Use Rebase

**Good for:**

- Tidying a feature branch before merging to `main`.
- Keeping a clean, linear history.
- Making your work read as if built on the latest code.

**Avoid when:**

- The branch is shared/public and others may have it.
- The commits are already pushed and others are building on them.
- You want to preserve the exact timeline of when work happened.

## Common Commands

```bash
git rebase main            # rebase the current branch onto main
git rebase -i main         # interactive — edit, squash, reorder, drop commits
git rebase --continue      # proceed after resolving a conflict
git rebase --skip          # skip the current commit
git rebase --abort         # cancel and return to the pre-rebase state
```

## Interactive Rebase

```bash
git rebase -i main
```

Git opens an editor listing the commits to be replayed:

```text
pick d1a2b3c Add login feature
pick e4f5g6h Fix login bug
pick h7i8j9k Update tests
```

Change `pick` to control each commit:

- `reword` — keep the commit, edit its message.
- `squash` / `fixup` — fold into the previous commit (`fixup` also discards the message).
- `edit` — pause at this commit to amend its contents.
- `drop` — remove the commit entirely.
- Reorder the lines to reorder the commits.

Save and close, and Git applies your plan. A common use is squashing several WIP commits into one clean commit before a PR.

## Resolving Conflicts During a Rebase

Because commits are replayed one at a time, a conflict can surface at each step:

```bash
# Git pauses on the conflicting commit and lists the files
git status

# Edit the files to resolve the conflict markers, then stage them
git add <file>

# Continue to the next commit
git rebase --continue
```

Repeat until all commits are replayed. If it goes wrong, `git rebase --abort` returns you to exactly where you started. See [Aborting and Investigating Merge Conflicts](articles/git-abort-investigate-merge-conflicts.md) for deeper conflict tooling.

> Tip: enable `git config --global rerere.enabled true` so Git remembers how you resolved a conflict and re-applies it automatically if the same one recurs during the rebase.

## Updating a Pushed Branch After Rebasing

Rebasing changes commit hashes, so if the branch was already pushed, a normal push is rejected. Update the remote with a lease-protected force push (never a blind `--force`):

```bash
git push --force-with-lease
```

`--force-with-lease` refuses to overwrite if someone else pushed since your last fetch, which protects teammates' work.

## The Golden Rule

**Never rebase commits that have been pushed to a shared repository and that others may be working on.** Rewriting shared history forces everyone else to reconcile diverged copies and is a common source of team pain. Rebase freely on private, local-only branches; once commits are shared, prefer merge (or coordinate explicitly before force-pushing).

## Summary

- `git rebase main` replays your commits onto the latest `main` as new commits, giving linear history with no merge commit.
- Use `git rebase -i` to reword, squash, reorder, or drop commits before sharing.
- Resolve conflicts per replayed commit: fix, `git add`, `git rebase --continue`; `--abort` bails out safely.
- After rebasing a pushed branch, update it with `git push --force-with-lease`.
- Only rebase private history — never shared, pushed commits others depend on.
