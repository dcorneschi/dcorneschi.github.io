# Restoring a Repository to a Past Commit's State

"Take my files back to how they looked at commit X" sounds like one operation, but Git offers several — `git checkout`, `git restore`, `git revert`, and `git reset --hard` — and they differ in a subtle, important way: **whether they delete files that were added *after* the target commit.** Getting this wrong leaves stray files behind or destroys work unexpectedly. This guide compares the approaches and shows how to reach an exact past state.

## The Core Gotcha

Restoring *tracked* files to an old version does **not** remove files that didn't exist back then — those are untracked relative to the old commit and stay in your working tree. Reaching a truly identical state usually needs a cleanup step.

| Command | Deletes files added after the target commit? |
|---------|:--------------------------------------------:|
| `git checkout X -- .` | No — only rewrites tracked files |
| `git restore --source=X .` | No — same limitation |
| `git restore --source=X . && git clean -fd` | Yes — `clean` removes the extras |
| `git revert X` | Yes — if X is the commit that added them |
| `git reset --hard X` | Yes — automatically |

## Restore Specific Files or Directories

To pull individual files/dirs from a commit into your working tree (they land **staged**):

```bash
# Specific files
git checkout <commit> -- file1.txt file2.txt

# A whole directory
git checkout <commit> -- path/to/dir/

# Modern equivalent (Git 2.23+)
git restore --source=<commit> file1.txt file2.txt

# From N commits ago
git checkout HEAD~3 -- file1.txt

# Then commit
git commit -m "Restore files from <commit>"
```

Find the commit first with:

```bash
git --no-pager log --oneline
```

The `--` separates the commit reference from the file paths, so Git doesn't mistake a filename for a branch.

## Restore the Whole Tree to a Commit (staying on your branch)

```bash
# Overwrite all TRACKED files with their state at <commit>
git checkout <commit> -- .
# or the modern form
git restore --source=<commit> .
```

This stages the differences and keeps you on your current branch — HEAD doesn't move. But remember the gotcha: files added *after* `<commit>` are untracked-to-it and remain. To get an exact match, add a clean:

```bash
git restore --source=<commit> .
git clean -fd            # preview first with: git clean -nd
git commit -m "Restore working tree to <commit>"
```

The result: history moves *forward* with a new commit whose contents match the old state — nothing is rewritten, so it's safe on shared branches.

## Output a File Without Touching the Working Tree

`git show` prints a file's content at a commit — handy to inspect before restoring, or to redirect into place:

```bash
git show <commit>:path/to/file          # print to stdout
git show <commit>:path/to/file > file   # overwrite the working copy
```

## restore vs revert vs reset --hard

These answer three different questions:

- **`git restore --source=X .`** — "make my files look like X." A blanket overwrite of tracked files; doesn't commit; doesn't move HEAD; leaves newer files unless you `git clean`.
- **`git revert X`** — "undo what commit X did." Surgical: reverses only X's changes and records a new commit. Safe on shared branches. If X added files, the revert removes them.
- **`git reset --hard X`** — "move my branch to X and match it exactly." Moves HEAD, resets index and working tree, and deletes files added after X automatically. **Rewrites history** — dangerous once pushed.

### Do they ever produce the same result?

In a simple linear history, undoing the most recent commit gives the same end state either way:

```text
77e1847 (files: daniel)
   |
e36305b (files: daniel, adinuta, pisu)   <- HEAD
```

Both get you back to just `daniel`:

```bash
# Blanket restore
git restore --source=77e1847 .
git clean -fd
git commit -m "Restore to 77e1847"

# Surgical revert
git revert e36305b
```

They diverge across multiple commits:

```text
A --- B --- C --- D   <- HEAD
```

- `git restore --source=A .` matches state A (effectively undoing B, C, and D together).
- `git revert D` undoes only D, keeping B's and C's changes.

## Reaching an Exact State (moving HEAD)

When you want the branch itself to point at the old commit and the tree to match precisely:

```bash
git reset --hard 77e1847     # HEAD -> 77e1847, tree matches, newer files removed
```

This is the cleanest way to an exact state, but it **rewrites history** — only do it on private branches, or coordinate and force-push (`git push --force-with-lease`) if the branch was shared. To sync to the remote's state instead of a local commit:

```bash
git fetch origin
git reset --hard origin/main
git clean -fdx               # preview with -nxd first
```

## Choosing an Approach

- **A few files from an old commit** → `git checkout <commit> -- <files>` or `git restore --source=<commit> <files>`.
- **Match a past state, keep history intact** → `git restore --source=<commit> .` + `git clean -fd`, then commit. Safe on shared branches.
- **Undo one specific commit on a shared branch** → `git revert <commit>`.
- **Reset the branch to exactly an old commit (private branch)** → `git reset --hard <commit>`.
- **Just inspect a file at a commit** → `git show <commit>:<file>`.

## Related

- [Restoring Files with git restore](articles/git-restore-files.md)
- [git reset Guide: soft, mixed, and hard](articles/git-reset-guide.md)
- [Undoing Changes in Git: reset vs checkout vs revert](articles/git-undo-reset-checkout-revert.md)
- [Removing Untracked Files with git clean](articles/git-clean-untracked-files.md)
