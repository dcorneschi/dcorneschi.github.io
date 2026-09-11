# Changing a Commit Message After Pushing (When No One Has Pulled)

You pushed a commit, then noticed a typo or a vague message. Fixing it means rewriting the commit — which changes its hash — and force-pushing the new version to the remote. This is safe **only** when no one else has pulled the branch yet, which is exactly the situation this guide assumes. If others may have the old commit, see the caution at the end.

> For the full picture of editing messages (older commits, uncommitted changes, `--autostash`), see [Editing a Commit Message](articles/git-edit-commit-message.md).

## Why "No One Has Pulled" Matters

Rewording a commit doesn't edit it in place — Git creates a *new* commit with a new SHA and moves the branch to point at it. The old commit is orphaned. Pushing that requires overwriting the remote branch (a force push).

- **No one has pulled** → nobody has the old commit, so replacing it on the remote affects only you. Safe.
- **Someone has pulled** → they still hold the old hash; your force push makes their history diverge, and they'll have to reconcile. Avoid — use `git revert` or coordinate.

## Before You Start

- The branch with the commit is checked out (`git switch <branch>`).
- You have a clean working tree, or you've stashed uncommitted changes (see the gotcha below).
- You've confirmed no one else has pulled the branch since your last push — ask teammates or check the remote's activity if unsure.

## Fixing the Most Recent Commit

This is the common case. Amend the message, then force-push:

```bash
# 1. Rewrite the last commit's message
git commit --amend -m "Correct, clearer message"

# 2. Overwrite the remote branch
git push --force-with-lease
```

`git commit --amend` with no `-m` opens your editor instead, if you prefer to write a longer message:

```bash
git commit --amend
```

### Gotcha: amend also folds in staged changes

`git commit --amend` doesn't only touch the message — it adds whatever is currently **staged** into the amended commit. If you have unrelated work staged and only want to change the message, stash first so nothing extra sneaks in:

```bash
git stash          # shelve uncommitted work
git commit --amend -m "Correct message"
git stash pop      # bring it back
```

### Prefer --force-with-lease over --force

Even when you believe no one has pulled, use `--force-with-lease`. It only overwrites the remote if it still matches what you last fetched — so if someone *did* push in the meantime, the push is rejected instead of silently destroying their work.

```bash
git push --force-with-lease          # safe force
git push --force                     # overwrites unconditionally — avoid
```

If `--force-with-lease` is rejected, don't reach for `--force` — fetch, check what changed, and reconsider whether it's really safe.

## Fixing an Older Commit's Message

If the commit isn't the latest, use an interactive rebase to `reword` it, then force-push:

```bash
git rebase -i HEAD~N          # N = how many commits back the target is
```

In the editor, change `pick` to `r` (or `reword`) on the target line, save, and Git opens a second editor for the new message:

```text
pick   abc1234 Some earlier commit
r      def5678 Message to fix        <- change pick -> r
pick   ghi9012 Latest commit
```

Then push the rewritten history:

```bash
git push --force-with-lease
```

## If Someone Has Already Pulled

If it turns out a teammate already pulled the old commit, **don't** amend and force-push — that splits their history from the remote and causes merge headaches. Instead, for a mere message fix, the least disruptive option is usually to leave the commit alone and move on; the wording lives in history but nothing breaks.

If the message genuinely needs correcting on the record, either:

- **Add a follow-up commit** noting the correction (e.g. "Note: previous commit should read '…'"), or
- **Coordinate a rewrite** — amend and force-push, then have everyone who pulled reset their local branch to the new history. Disruptive, so reserve it for messages that actually matter (misleading, security-sensitive, etc.).

## Verify Before and After

```bash
# Before: see the message you're about to change
git log -1 --pretty=%B          # last commit's message
git log --oneline -5            # or a short list

# After amending/rewording, confirm the new message and that you're ahead of the remote
git log --oneline -1
git status                      # "ahead of 'origin/...' by 1 commit" until you push
```

## If the Force Push Is Rejected

```
! [rejected]  main -> main (stale info)
```

`--force-with-lease` refused because the remote moved since your last fetch — meaning someone else pushed (and quite possibly pulled). Stop and investigate:

```bash
git fetch origin
git log --oneline origin/main -5   # see what landed
```

If others now have history built on the branch, don't force-push. Leave the original commit and add a follow-up, or coordinate a rewrite with the team.

## Summary

- Rewording a pushed commit rewrites its hash and needs a force push — safe only while no one else has the branch.
- Latest commit: `git commit --amend -m "…"` then `git push --force-with-lease`.
- Older commit: `git rebase -i HEAD~N`, mark it `r`, then `git push --force-with-lease`.
- Always use `--force-with-lease`, never blind `--force`; if it's rejected, someone pushed — reassess rather than forcing.

## Source

- [How to Change a Git Commit Message After Push (When No One Has Pulled)](https://www.codegenes.net/blog/changing-git-commit-message-after-push-given-that-no-one-pulled-from-remote/) — content was rephrased for compliance with licensing restrictions.
