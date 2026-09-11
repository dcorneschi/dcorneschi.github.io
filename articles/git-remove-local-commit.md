# Removing a Local Commit: reset, revert, and rebase

You just committed something you didn't mean to — wrong files, a typo in the message, or a change you want to redo. If the commit hasn't been pushed to a shared branch, you have several ways to undo it, differing in whether your changes are kept, discarded, or reversed with a new commit. This guide walks through each and when to use it.

> If the commit is **already pushed to a shared branch**, prefer `git revert` and see [Undoing a Pushed Commit: revert vs reset](articles/git-undo-pushed-commit.md), which covers the force-push considerations.

## 1. Soft Reset — keep changes staged

```bash
git reset --soft HEAD~1
```

Moves the branch pointer back one commit but leaves your changes **staged**, ready to recommit. Ideal for redoing the commit message or combining with more edits:

```bash
git reset --soft HEAD~1
git commit -m "Better message"
```

## 2. Mixed Reset — keep changes unstaged

```bash
git reset --mixed HEAD~1      # --mixed is the default
git reset HEAD~1              # same thing
```

Undoes the commit **and** unstages the changes, leaving the edits in your working tree. Use it when you want to re-select what to stage before committing again.

## 3. Hard Reset — discard everything

```bash
git reset --hard HEAD~1
```

Removes the commit and **throws away** its changes from the index and working tree. Fast, but the uncommitted result is unrecoverable — use only when you're certain you don't want the changes.

## 4. Revert — undo with a new commit

```bash
git revert HEAD
```

Creates a **new** commit that reverses the target commit's changes, leaving history intact. This is the safe choice for shared branches because it doesn't rewrite anything. No force push needed.

## 5. Interactive Rebase — surgical edits across commits

```bash
git rebase -i HEAD~2
```

Opens an editor listing the last N commits, where you can `drop` (remove), `squash`/`fixup` (combine), `reword` (rename), `edit`, or reorder them. Use it when the fix involves more than just the most recent commit — e.g. deleting a commit from the middle or squashing three into one.

```text
pick   a1b2c3d Add feature
drop   d4e5f6a Debug logging   # delete this commit
squash 7g8h9i0 Fix typo        # fold into the one above
```

## Choosing the Right Method

| Method | History | Your changes | Best for |
|--------|---------|--------------|----------|
| `reset --soft` | Rewritten | Kept, staged | Redo the commit or its message |
| `reset --mixed` | Rewritten | Kept, unstaged | Re-pick what to stage |
| `reset --hard` | Rewritten | **Discarded** | Throw the commit away entirely |
| `revert` | Preserved (adds commit) | Reversed by new commit | Commits on shared/pushed branches |
| `rebase -i` | Rewritten | Per your choices | Editing/removing multiple commits |

Rules of thumb:

- **Local, not pushed** → `reset` (`--soft`/`--mixed` to keep work, `--hard` to drop it).
- **Pushed or shared** → `revert`, to avoid rewriting history others have.
- **Multiple or non-latest commits** → interactive `rebase`.

## Undoing the Undo

`reset` and `rebase` move branch pointers, and the old position lives in the reflog until garbage collection:

```bash
# Find where the branch was before you reset/rebased
git reflog

# Point the branch back at it
git reset --hard <commit-hash>
```

This recovers **committed** state. Changes discarded by `git reset --hard` (that were never committed) can't be recovered — which is why the softer resets are safer defaults.

## Summary

- Keep the work: `git reset --soft HEAD~1` (staged) or `git reset --mixed HEAD~1` (unstaged).
- Drop it entirely: `git reset --hard HEAD~1`.
- Already shared: `git revert HEAD` — no history rewrite.
- Multiple/older commits: `git rebase -i HEAD~N`.
- Overshot? `git reflog` + `git reset --hard <hash>` gets committed state back.
