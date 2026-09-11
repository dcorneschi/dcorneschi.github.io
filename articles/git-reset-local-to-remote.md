# Resetting a Local Branch to Match the Remote

Sometimes local work goes sideways and you just want your branch to look **exactly** like the remote again — discarding local commits, uncommitted edits, and stray files. The tool for that is `git reset --hard` against the remote-tracking branch, optionally followed by `git clean`. This is destructive and unrecoverable for uncommitted work, so this guide also covers how to do it safely.

## The Core Command

```bash
git fetch origin
git reset --hard origin/master
```

What each step does:

1. `git fetch origin` — updates your remote-tracking refs so `origin/master` reflects the remote's current state (it does **not** change your files).
2. `git reset --hard origin/master` — moves your current branch to `origin/master` and forces the index and working tree to match, throwing away local commits and uncommitted changes.

If your default branch is `main`:

```bash
git fetch origin
git reset --hard origin/main
```

## Also Remove Untracked Files

`git reset --hard` restores tracked files but leaves **untracked** files (new files, build output) in place. To get a truly pristine checkout, add `git clean`:

```bash
git fetch origin
git reset --hard origin/master
git clean -fd          # remove untracked files and directories
```

Preview first — `git clean` deletions are not recoverable:

```bash
git clean -nd          # dry run: list what would be removed
```

Add `-x` (`git clean -fdx`) only if you also want `.gitignore`-d files gone, which typically includes local config and secrets.

## Reset the Current Branch Without Naming It

To reset whatever branch you're on to its own upstream, use the `@{u}` shorthand:

```bash
git fetch
git reset --hard @{u}      # @{u} = this branch's configured upstream
```

Handy when the branch isn't `master`/`main` but tracks a remote branch.

## Preview What You're About to Lose

Because this is destructive, check what will be discarded before running it:

```bash
git fetch origin

# Local commits that would be dropped
git log --oneline origin/master..HEAD

# Uncommitted changes that would be lost
git status
git diff
```

## Safer Alternatives

### Keep a backup branch first

Reset is far less scary if you park your current state on a branch beforehand:

```bash
git branch backup-before-reset
git fetch origin
git reset --hard origin/master
# recover later if needed with: git reset --hard backup-before-reset
```

### Stash instead of discarding

If you might want the uncommitted changes back:

```bash
git stash push -u          # save tracked + untracked changes
git fetch origin
git reset --hard origin/master
# bring them back onto the clean base:
git stash pop
```

## Recovering After a Reset

Committed work isn't truly gone immediately — the reflog remembers where your branch pointed:

```bash
# Find the commit you were on before the reset
git reflog

# Restore the branch to it
git reset --hard <commit-hash>
```

This works for **committed** state only. Uncommitted changes and files removed by `git clean` are unrecoverable — which is why the stash/backup steps above matter.

## reset --hard vs the Gentler Modes

If you don't actually want to throw everything away, a softer reset may be what you need:

| Command | Effect |
|---------|--------|
| `git reset --hard origin/master` | Match remote exactly; discard all local changes |
| `git reset --soft origin/master` | Move branch pointer only; keep changes staged |
| `git reset --mixed origin/master` | Move pointer, unstage; keep working-tree edits |
| `git restore --source origin/master <file>` | Reset a single file, not the whole branch |

## Summary

- `git fetch origin && git reset --hard origin/master` makes your branch identical to the remote.
- Add `git clean -fd` (preview with `-nd`) to also drop untracked files.
- Use `@{u}` to reset the current branch to its own upstream.
- This destroys uncommitted work permanently — stash or create a `backup` branch first if there's any doubt.
- Reset committed state back via `git reflog`; `clean`-ed and uncommitted changes can't be recovered.
