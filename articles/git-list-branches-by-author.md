# Listing Git Branches by Author

Finding "whose branches are these?" is a common cleanup and review task. Git has no concept of a branch *creator* — a branch is just a pointer — so these techniques key off the **author of each branch's tip commit**. That's usually what you want, with one caveat covered below. This guide shows the reliable ways to filter branches by author, locally and on the remote.

## An Important Caveat First

Git does **not** record who created a branch. What you can filter on is the author (or committer) of the commit a branch currently points to. That's a good proxy most of the time, but note:

- If someone else made the latest commit on a branch you started, it'll show under *their* name.
- After a merge or rebase, the tip author may change.

With that understood, here are the methods.

## Method 1: for-each-ref (recommended)

```bash
# Local branches whose tip commit is by "Author Name"
git for-each-ref --format='%(refname:short) %(authorname)' refs/heads | grep "Author Name"
```

`git for-each-ref` reads the branch tip's author directly, making it fast and reliable. Replace `Author Name` with the name (or a fragment of it).

## Method 2: Include Remote Branches

```bash
git for-each-ref --format='%(refname:short) %(authorname)' refs/heads refs/remotes | grep "Author Name"
```

Adding `refs/remotes` widens the search to remote-tracking branches. Run `git fetch --all` first so those refs are current.

## Method 3: Add Dates for Sorting and Cleanup

```bash
# Include the tip commit date
git for-each-ref --format='%(refname:short) %(authorname) %(committerdate:short)' \
  refs/heads refs/remotes | grep "Author Name"

# Sort by most-recent activity
git for-each-ref --sort=-committerdate \
  --format='%(committerdate:short) %(authorname) %(refname:short)' refs/heads | grep "Author Name"
```

Sorting by `committerdate` is handy for spotting stale branches to prune.

## Method 4: Filter with the Author Email

Names can collide or vary; email is often more precise:

```bash
git for-each-ref --format='%(refname:short) %(authoremail)' refs/heads refs/remotes \
  | grep "user@example.com"
```

## Method 5: Colorized Output

```bash
git for-each-ref \
  --format='%(color:green)%(refname:short)%(color:reset) by %(color:blue)%(authorname)%(color:reset) on %(committerdate:short)' \
  refs/heads refs/remotes | grep "Author Name"
```

## Method 6: By Commits, Not Just the Tip

The `for-each-ref` methods only inspect each branch's **tip**. If you want branches that contain *any* commit by an author (even if it's not the latest), check each branch's log:

```bash
git branch -a --format='%(refname:short)' | while read br; do
  if git log -1 --author="Author Name" "$br" >/dev/null 2>&1 \
     && [ -n "$(git log --author='Author Name' -1 --pretty=%H "$br")" ]; then
    echo "$br"
  fi
done
```

Simpler, if you just want to confirm a branch's tip is by someone:

```bash
git branch -a --format='%(refname:short)' \
  | xargs -I {} sh -c 'git log -1 --author="Author Name" {} >/dev/null 2>&1 && echo {}'
```

## Tips

- **Partial names work:** `grep "John"` instead of the full name.
- **Case-insensitive:** `grep -i "john"`.
- **Show only branch names:** drop the extra `%(...)` fields from the format string.
- **Fetch first** for accurate remote results: `git fetch --all --prune`.
- **Prefer email** (`%(authoremail)`) when multiple people share a display name.

## Quick Reference

| Command | Lists |
|---------|-------|
| `git for-each-ref --format='%(refname:short) %(authorname)' refs/heads \| grep "Name"` | Local branches by tip author |
| `... refs/heads refs/remotes \| grep "Name"` | Local + remote branches |
| `--format='... %(authoremail)'` | Filter by email instead of name |
| `--sort=-committerdate` | Order by most recent activity |
| `git branch -a --format='%(refname:short)' \| xargs -I {} sh -c '...'` | Branches whose tip is by an author |
