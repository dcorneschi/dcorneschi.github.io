# Sharing a Branch: Push, Then Fetch vs Pull

Handing your work to a teammate is a two-sided flow: you commit and push your branch, and they retrieve it. The retrieving side is where people trip up — `git fetch` and `git pull` are not interchangeable, and using the wrong one at the wrong time causes surprise merges or errors. This guide walks the whole handoff and explains when each command fits.

## Your Side: Commit and Push

### If you have uncommitted changes

```bash
git status                 # see what's changed
git add .                  # stage everything (or name specific files)
git commit -m "Your commit message"
```

### Push the branch

```bash
# Existing branch that already tracks a remote
git push origin <branch-name>

# First push of a new branch — also sets up tracking
git push -u origin <branch-name>
```

The `-u` (`--set-upstream`) on the first push links your local branch to the remote one, so later you can just run `git push` and `git pull` with no arguments. See [git push vs git push origin HEAD](articles/git-push-vs-push-origin-head.md) for the details of what gets pushed where.

## Their Side: Get Your Changes

### First time getting the branch

```bash
git fetch origin              # download the new branch info (nothing merged)
git switch <branch-name>      # check it out (git checkout <branch-name> also works)
```

After the fetch, checking out the branch name creates a local branch tracking `origin/<branch-name>`, already up to date.

### Already on the branch, just want updates

```bash
git switch <branch-name>      # make sure you're on it
git pull origin <branch-name> # fetch AND merge the new commits
```

## fetch vs pull — the Key Distinction

This is the crux of the whole workflow:

| | `git fetch` | `git pull` |
|---|-------------|------------|
| What it does | Downloads remote commits/branches | Fetches **and** merges into your current branch |
| Touches your working tree | No | Yes |
| Can cause conflicts | No | Yes |
| Best for | Getting a branch for the first time; inspecting before merging | Updating a branch you're actively on |

- **`git fetch`** is always safe — it updates your remote-tracking refs (`origin/*`) without changing your files. Nothing merges until you say so.
- **`git pull`** is `fetch` + `merge` in one step. Convenient for updating the branch you're on, but it can produce merge conflicts and will act on whatever branch you currently have checked out — which is why it surprises people who run it on the wrong branch.

## Recommended Workflow

```bash
# First time getting a teammate's branch
git fetch origin
git switch <branch-name>       # up to date, nothing to merge

# Updating a branch you already have
git switch <branch-name>
git pull                        # or: git pull --rebase for linear history
```

For keeping a feature branch current with `main`/`master` (the more involved case with merge-vs-rebase choices), see [Getting the Latest Changes from Master into Your Feature Branch](articles/git-update-feature-branch-from-master.md).

## Inspect Before You Merge

The advantage of `fetch` is that you can review incoming changes before integrating:

```bash
git fetch origin

# What did the remote branch gain that you don't have?
git log HEAD..origin/<branch-name> --oneline

# See the actual diff
git diff HEAD..origin/<branch-name>

# Happy with it? Merge (or rebase) it in
git merge origin/<branch-name>
```

## Notes

- Prefer `git add <files>` over `git add .` when you want tight, intentional commits — `.` sweeps in everything under the current directory.
- `git pull --rebase` replays your local commits on top of the fetched ones for a linear history instead of a merge commit.
- If `git pull` errors with "not currently on a branch" or acts unexpectedly, confirm which branch you're on with `git status` first.
- After a teammate deletes a remote branch, run `git fetch --prune` to clear the stale `origin/*` reference.

## Summary

- **Push:** `git add`/`git commit`, then `git push -u origin <branch>` the first time, `git push` after.
- **Retrieve first time:** `git fetch origin` then `git switch <branch>` — safe, no merge.
- **Update an existing branch:** `git pull` while on it — fetches and merges.
- `fetch` never touches your files; `pull` merges into the current branch, so know which branch you're on before running it.
