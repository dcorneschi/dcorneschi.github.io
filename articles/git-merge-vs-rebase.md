# Git Merge vs Rebase

Merge and rebase both integrate changes from one branch into another, but they produce very different history and carry very different risk. Merge preserves the exact timeline and adds a merge commit; rebase rewrites your commits onto a new base for a clean, linear history. This guide compares them visually and practically, and gives clear rules for when to use each.

## Visual Comparison

Starting point — `feature` branched off `main`, both have moved on:

```text
main:      A---B---C
            \
feature:     D---E---F
```

After `git merge main` (into feature) — a merge commit `M` joins the two lines:

```text
main:      A---B---C-------M
            \             /
feature:     D---E---F---/
```

After `git rebase main` — `feature`'s commits are replayed on top of `C` as new commits (`D'`, `E'`, `F'`):

```text
main:      A---B---C
                    \
feature:             D'---E'---F'
```

The rebase result has no merge commit and reads as if the work started from the latest `main`. Note `D'/E'/F'` are **new commits with new hashes** — the originals are rewritten.

## Key Differences

| Aspect | Merge | Rebase |
|--------|-------|--------|
| History | Preserves the original timeline | Rewrites it into a linear line |
| Merge commit | Yes | No |
| Commit hashes | Unchanged | Rewritten |
| Conflicts | Resolved once | May recur per replayed commit |
| Traceability | Shows when branches integrated | Loses that context |
| Shared branches | Safe | Dangerous — rewrites shared history |

## Merge — Pros and Cons

**Pros**

- Preserves complete history and integration context.
- Safe for shared/public branches — nothing is rewritten.
- Shows exactly when a feature was merged.
- Non-destructive and easy to undo.
- Conflicts are resolved once, in the merge commit.

**Cons**

- Adds merge commits, which some find noisy.
- Harder to read as a linear progression.
- Many merges can produce a tangled graph.

## Rebase — Pros and Cons

**Pros**

- Clean, linear history that's easy to follow.
- No extra merge commits.
- Makes the work read as if built on the latest `main`.
- Nicer diffs for code review.

**Cons**

- Rewrites commit history — dangerous on shared branches.
- The same conflict can resurface across several replayed commits.
- Loses the record of when the work actually happened.
- Easier to get wrong if you're not comfortable with Git.

## When to Use Which

**Prefer merge when:**

- Working on shared/public branches or `main`/`develop`.
- Multiple developers touch the same branch.
- You want to preserve exact history and integration points.
- On a long-running feature branch.

**Prefer rebase when:**

- The branch is private and only you touch it.
- You want clean, linear history before opening a PR.
- Preparing small, focused changes for review.

## The Golden Rule

**Never rebase commits that have been pushed and that others may have based work on.** Rewriting shared history forces everyone else to reconcile diverged copies. If commits live only on your local, private branch, rebase freely; once they're shared, use merge (or coordinate explicitly).

Because rebase rewrites hashes, updating an already-pushed branch after a rebase requires a force push — use `--force-with-lease`, never a blind `--force`:

```bash
git push --force-with-lease
```

## Best of Both Worlds

A common, effective workflow: **rebase to tidy your private branch, then merge into main.**

```bash
# 1. Clean up your private feature branch on the latest main
git switch feature-branch
git rebase main

# 2. Integrate with a merge (a fast-forward, or --no-ff for an explicit merge commit)
git switch main
git merge feature-branch
```

This gives you clean, linear feature commits while still recording the integration. Many hosts automate it: GitHub's "Rebase and merge" and "Squash and merge" options apply the same idea in the PR UI.

## Workflow Examples

### Safe merge workflow

```bash
git switch main
git pull origin main
git switch feature-branch
git merge main            # bring main into feature, resolve conflicts once
git switch main
git merge feature-branch  # integrate feature into main
```

### Rebase workflow (private branches only)

```bash
git switch feature-branch
git rebase main           # replay your commits on the latest main
# resolve conflicts as they come up, then:
git rebase --continue
git switch main
git merge feature-branch  # fast-forward merge
```

## Team Guidance

- **Small teams:** rebase private branches, merge (or squash-merge) into main.
- **Large teams:** favor merge for safety; avoid rebasing shared branches.
- **Open source:** merge external contributions; maintainers may rebase for cleanup.

When in doubt, **merge** — it's safer and more forgiving, and the extra merge commits are honest history rather than clutter.

## Summary

- Merge preserves history and is safe on shared branches; rebase produces linear history but rewrites commits.
- Never rebase shared/pushed commits; rebase only private branches.
- A rebase-then-merge flow gives clean feature commits plus a recorded integration.
- Updating a pushed branch after rebasing needs `git push --force-with-lease`.

Related: [Getting the Latest Changes from Master into Your Feature Branch](articles/git-update-feature-branch-from-master.md) · [Aborting and Investigating Merge Conflicts](articles/git-abort-investigate-merge-conflicts.md) · [Syncing a Branch Across Multiple Computers](articles/git-sync-branch-across-computers.md)
