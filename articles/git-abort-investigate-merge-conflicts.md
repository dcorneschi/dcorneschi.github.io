# Aborting and Investigating Merge Conflicts

When a merge stops with conflicts, you have two reasonable moves: abort and step back to understand what's going on, or inspect the conflicts in place before deciding. This guide covers aborting cleanly, investigating the difference between the branches, and reading the conflict "stages" Git exposes during a merge.

## Abort the Merge

```bash
git merge --abort
```

This cancels the in-progress merge and returns your working tree and index to exactly the state they were in before you started. It's the safest reset when a merge surprises you.

If `--abort` complains (rare, usually from a pre-existing dirty state), the lower-level equivalent is:

```bash
git reset --merge
```

## Confirm You're Actually Mid-Merge

```bash
git status
```

During a conflicted merge, `git status` shows `You have unmerged paths` and lists files as `both modified`. After a successful `--abort`, it returns to a clean (or your prior) state. The presence of a `.git/MERGE_HEAD` file is the definitive signal that a merge is in progress.

## Investigate the Branches

Once aborted, figure out why the branches diverged before trying again.

```bash
# Where you are and which branches exist
git branch -a

# Visualize how the branches diverged
git log --oneline --graph --decorate --all

# Commits that differ between two branches
git log --oneline main..feature-branch    # on feature, not main
git log --oneline feature-branch..main     # on main, not feature

# Full content diff between the branches
git diff main..feature-branch

# Just the names of files that differ
git diff --name-only main..feature-branch
```

The two-dot range (`a..b`) is the quick way to answer "what's on one branch that isn't on the other."

## Preview a Merge Without Committing

To find out whether a merge *will* conflict before you commit to it:

```bash
# Attempt the merge but don't auto-commit or fast-forward
git merge --no-commit --no-ff feature-branch

# Inspect the result, then back out
git merge --abort
```

You can also spot likely conflicts ahead of time by checking which files both sides touched:

```bash
git diff --name-only main...feature-branch
```

## Inspect Conflicts Without Aborting

If you'd rather look before deciding, examine the conflict in place:

```bash
# Which files are conflicted
git status

# Show only the files with conflicts
git diff --name-only --diff-filter=U

# See the conflicting hunks (combined diff)
git diff
```

### Read the Three Conflict Stages

During a conflict, Git keeps three versions of each conflicted file in the index, addressable by stage number:

```bash
git show :1:path/to/file    # stage 1 — common ancestor (merge base)
git show :2:path/to/file    # stage 2 — "ours" (current branch)
git show :3:path/to/file    # stage 3 — "theirs" (branch being merged)
```

Comparing these tells you exactly what each side changed relative to the shared starting point, which is often clearer than reading the inline conflict markers.

```bash
# What "ours" changed since the ancestor
git diff :1:path/to/file :2:path/to/file

# What "theirs" changed since the ancestor
git diff :1:path/to/file :3:path/to/file
```

### Diff Against One Side at a Time

```bash
# Show the conflict relative to our version
git diff --ours path/to/file

# ...relative to their version
git diff --theirs path/to/file

# ...relative to the merge base
git diff --base path/to/file
```

## Resolve In Place Instead of Aborting

If investigation shows the conflict is straightforward, you can just fix it:

```bash
# Take our version of a file wholesale
git checkout --ours path/to/file

# Or take their version
git checkout --theirs path/to/file

# Then stage and continue
git add path/to/file
git commit          # completes the merge
```

Use `git merge --continue` (or `git commit`) once all conflicts are staged.

## Recommended Workflow

```bash
# 1. A merge conflicted — step back
git merge --abort

# 2. Understand how the branches diverged
git log --oneline --graph --decorate --all
git diff --name-only main..feature-branch

# 3. Re-attempt the merge now that you know what to expect
git merge feature-branch
```

## Notes

- `git merge --abort` only works while a merge is in progress; there's nothing to abort once it's committed (use `git reset --hard ORIG_HEAD` to undo a just-completed merge instead).
- The same investigation commands apply to rebase conflicts — swap `git rebase --abort` for the abort step.
- `ORIG_HEAD` points at where HEAD was before the merge started, which is handy for comparisons or recovery.
- Commit or stash unrelated work before merging; a clean working tree makes both aborting and investigating far simpler.
