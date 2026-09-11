# Undoing a Pushed Commit: revert vs reset

Once a commit is pushed, undoing it splits into two philosophies: **revert**, which records a new commit that cancels the change and keeps history intact, and **reset**, which removes the commit and rewrites history. The right choice depends on whether the branch is shared. This guide covers both, the safer force-push option, and how to pick.

## The Decision in One Line

- **Shared branch** (others may have pulled it): use `git revert` — it never rewrites history.
- **Private branch** (only you): `git reset` is fine, and gives cleaner history.

Rewriting history on a branch other people have already pulled forces everyone to reconcile their copies, which is the mess `revert` avoids.

## Option 1: git revert (safe for shared branches)

Creates a **new** commit that undoes the changes of a previous one. History is preserved — the bad commit stays, followed by its inverse.

```bash
# Revert a specific commit, opening an editor for the message
git revert <commit-hash>

# Push normally — no force needed
git push origin <branch-name>
```

Useful variants:

```bash
# Revert without opening the editor (use default message)
git revert --no-edit <commit-hash>

# Revert a range of commits (newest to oldest)
git revert --no-edit <oldest-hash>^..<newest-hash>

# Stage the revert but don't commit yet (combine several into one)
git revert --no-commit <commit-hash>
git commit -m "Revert problematic changes"
```

Reverting a **merge commit** needs `-m` to pick which parent to keep as mainline:

```bash
git revert -m 1 <merge-commit-hash>
```

## Option 2: git reset --hard (private branches only)

Removes the commit entirely by moving the branch pointer back, then rewrites the remote with a force push.

```bash
# Move the branch to the commit BEFORE the bad one
git reset --hard <commit-hash>~1

# Overwrite the remote branch (rewrites history)
git push --force origin <branch-name>
```

`<commit-hash>~1` means "the parent of that commit," so the bad commit is dropped. To simply drop the most recent commit:

```bash
git reset --hard HEAD~1
git push --force origin <branch-name>
```

Beware: `--hard` also discards any uncommitted changes in your working tree.

## Option 3: git push --force-with-lease (safer force push)

Prefer this over plain `--force`. It refuses to overwrite the remote if someone else pushed since your last fetch, protecting you from clobbering their work.

```bash
git reset --hard <commit-hash>~1
git push --force-with-lease origin <branch-name>
```

If it's rejected, someone pushed in the meantime — fetch, review, and reconcile before trying again rather than reaching for `--force`.

## Choosing the Reset Mode

`git reset` has three modes that differ in what they do to your changes:

| Mode | Branch pointer | Staging area | Working tree | Use when |
|------|:-------------:|:------------:|:------------:|----------|
| `--soft` | moved | kept | kept | Re-commit differently; changes stay staged |
| `--mixed` (default) | moved | reset | kept | Unstage but keep edits in the working tree |
| `--hard` | moved | reset | **discarded** | Throw the changes away entirely |

```bash
# Undo the last commit but keep its changes staged for a new commit
git reset --soft HEAD~1

# Undo the last commit and unstage, keeping the file edits
git reset --mixed HEAD~1
```

## revert vs reset at a Glance

| | `git revert` | `git reset --hard` |
|---|-------------|--------------------|
| History | Preserved (adds a commit) | Rewritten (removes commits) |
| Push | Normal `git push` | Requires `--force` / `--force-with-lease` |
| Safe on shared branches | Yes | No |
| Undoes | One or more specified commits | Everything back to a point |
| Recoverable | Trivially (revert the revert) | Via reflog, until GC |

## Recovering If You Reset Too Far

A `reset` isn't truly gone right away — the reflog remembers where the branch was:

```bash
# Find the commit you reset away from
git reflog

# Point the branch back at it
git reset --hard <commit-hash>
```

## Summary

- Public/shared branch → `git revert <hash>` then a normal push. History intact, no force.
- Private branch → `git reset --hard <hash>~1` then `git push --force-with-lease`.
- Always prefer `--force-with-lease` over `--force`.
- Pick the reset mode (`--soft`/`--mixed`/`--hard`) based on whether you want to keep the changes.
- Reset in error? `git reflog` plus `git reset --hard <hash>` gets you back.
