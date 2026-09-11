# Editing a Commit Message

Changing a commit's message is easy for the latest commit and only slightly more involved for older ones. The method depends on two things: whether the commit is the most recent, and whether it's already been pushed. This guide covers `git commit --amend` for the tip, interactive rebase `reword` for older commits, handling uncommitted changes with `--autostash`, and when a force push is required.

## Latest Commit, Not Pushed

```bash
git commit --amend -m "New commit message"
```

`--amend` rewrites the most recent commit with the new message. Omit `-m` to open your editor instead:

```bash
git commit --amend
```

### Related amend options

```bash
# Add staged changes to the last commit WITHOUT changing its message
git add forgotten-file.txt
git commit --amend --no-edit

# Change the message AND reset the author to your current identity
git commit --amend --reset-author -m "New message"
```

`--no-edit` is handy for folding in a file you forgot to stage; `--reset-author` updates the recorded author/email (useful when the commit was made under the wrong identity).

## Older Commit, Not Pushed

Use interactive rebase and `reword` the target commit:

```bash
git rebase -i --autostash HEAD~N
```

Replace `N` with how many commits back the target is. In the editor, change `pick` to `r` (or `reword`) on that commit, save, and Git opens a second editor for the new message. Push normally when ready:

```bash
git push
```

No force push is needed here because the rewritten history hasn't left your machine yet.

## Already Pushed Commit

The steps are the same, but rewriting a commit that's already on the remote changes its hash, so you must force-push afterward — using `--force-with-lease` to avoid clobbering others' work.

### Recommended: with --autostash

```bash
git rebase -i --autostash HEAD~N
# reword the target commit, then:
git push --force-with-lease
```

### Manual stash equivalent

```bash
git stash
git rebase -i HEAD~N
# after the rebase completes:
git push --force-with-lease
git stash pop
```

`--autostash` does the stash/pop automatically around the rebase, which is why it's the cleaner option when you have uncommitted changes.

## The Rebase Steps in Detail

1. The editor opens with the commits in the range, oldest at the top:

   ```text
   pick abc1234 Some old message
   pick def5678 Another commit
   pick ghi9012 Latest commit
   ```

2. Change `pick` to `r` (or `reword`) on the commit you want to edit:

   ```text
   r abc1234 Some old message
   pick def5678 Another commit
   pick ghi9012 Latest commit
   ```

3. Save and close. Git replays the commits and, at the reworded one, opens another editor — write the new message, save, and close.

4. If the commit was pushed, force-push:

   ```bash
   git push --force-with-lease
   ```

## Editing Multiple Messages at Once

Mark more than one commit with `r` in the same rebase to reword several in one pass — Git opens a message editor for each in turn.

## Notes and Cautions

- **`--force-with-lease` over `--force`.** It refuses to push if the remote moved since your last fetch, protecting teammates' commits. Plain `--force` overwrites unconditionally.
- **Only reword shared history with care.** Rewriting pushed commits forces everyone else to reconcile; coordinate on active shared branches.
- **`--autostash` handles uncommitted work.** It stashes before the rebase and re-applies after, so you don't have to stash manually.
- **Overshot or messed up the rebase?** `git rebase --abort` returns to the pre-rebase state; afterward, `git reflog` can recover the previous commit positions.
- **Amended the wrong commit and lost the old message?** The original stays in the reflog (30 days by default). Read its message without moving the branch, then amend again and paste it back:

  ```bash
  git reflog                       # find the entry, e.g. HEAD@{1}
  git show -s --format=%B HEAD@{1} # print its full message
  git commit --amend               # paste the recovered message
  ```
- **Amending is a rewrite too.** `git commit --amend` on an already-pushed tip also needs `git push --force-with-lease`.

## Summary

- Latest commit, local → `git commit --amend -m "…"`.
- Older commit → `git rebase -i --autostash HEAD~N`, mark it `r`, edit the message.
- Already pushed → same, then `git push --force-with-lease`.
- Use `--autostash` to carry uncommitted changes through the rebase; use `--force-with-lease`, never blind `--force`.

## Source

- [How to Change a Git Commit Message, Even After Pushing](https://linuxize.com/post/change-git-commit-message) — content was rephrased for compliance with licensing restrictions.
