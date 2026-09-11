# Viewing a File's Change History and Blame in Git

When you need to know how a file evolved — every commit that touched it, what each commit changed, or who last modified a specific line — Git's `log` and `blame` commands answer those questions. This guide covers file history, custom formatting, following renames, per-line blame, and how to dig from a suspicious line back to the commit and full change behind it.

## View a File's Commit History

```bash
# Every commit that touched the file
git log <filename>

# With the diff introduced by each commit
git log -p <filename>

# Compact, one line per commit
git log --oneline <filename>
```

`git log <filename>` limits history to commits that changed that path — the everyday way to answer "what happened to this file?"

## Custom Pretty Formatting

A format string gives you exactly the columns you want:

```bash
# hash - author, date : subject   (short YYYY-MM-DD date)
git log --pretty=format:"%h - %an, %ad : %s" --date=short <filename>

# Relative time, e.g. "2 days ago"
git log --pretty=format:"%h - %an, %ad : %s" --date=relative <filename>

# ISO 8601 timestamps
git log --pretty=format:"%h - %an, %ad : %s" --date=iso <filename>

# Show both author date and committer date
git log --pretty=format:"%h - %an, %ad (committed: %cd) : %s" --date=short <filename>
```

Common placeholders:

| Placeholder | Meaning |
|-------------|---------|
| `%h` | Abbreviated commit hash |
| `%an` | Author name |
| `%ae` | Author email |
| `%ad` | Author date (when the change was originally made) |
| `%cd` | Committer date (when it landed in the repo) |
| `%s` | Commit subject |

Author date vs committer date differ after operations like rebase or cherry-pick, which is why showing both can be revealing.

## Follow a File Through Renames

```bash
git log --follow <filename>
```

Plain `git log` stops at the point a file was renamed. `--follow` continues tracing history across the rename, so you see the file's full lineage. Combine it freely with formatting:

```bash
git log --follow --oneline <filename>
```

## Visualize the History

```bash
git log --graph --pretty=format:"%h %an %ad - %s" --date=short <filename>

# Stat of what changed each commit (files + line counts)
git log --stat <filename>
```

## Filter File History

Narrow the log to find a specific change faster:

```bash
# Commits by a specific author
git log --author="Jane" <filename>

# Commits within a date range
git log --since="2026-01-01" --until="2026-06-30" <filename>

# Commits whose message matches a pattern
git log --grep="refactor" <filename>

# Commits that added or removed a given string in the file ("pickaxe")
git log -S "functionName" <filename>

# Commits changing lines matching a regex
git log -G "regex" <filename>
```

The pickaxe (`-S`) is the fastest way to find when a particular piece of code was introduced or deleted.

## See Who Changed Each Line — git blame

```bash
# Annotate every line with its last-changing commit and author
git blame <filename>

# Show author email and short dates
git blame -e --date=short <filename>

# Blame only a line range (e.g. lines 40–60)
git blame -L 40,60 <filename>
```

Each blame line shows the commit hash, author, timestamp, and the line content — pointing you at the commit responsible.

### See Past a Refactor with Blame

Blame can be fooled by code that merely moved. These flags look through movement to the real origin:

```bash
# Detect lines moved or copied within the same file
git blame -M <filename>

# Detect lines moved or copied from other files
git blame -C <filename>

# Ignore whitespace-only changes when assigning blame
git blame -w <filename>
```

To skip noisy formatting commits entirely, list them in a file and pass it:

```bash
git blame --ignore-revs-file .git-blame-ignore-revs <filename>
```

## From a Line to the Full Story

A typical investigation chains blame into log and show:

```bash
# 1. Find the commit that last touched the line
git blame -L 42,42 path/to/file

# 2. Read that commit's full change
git show <commit-hash>

# 3. See what the file looked like at that commit
git show <commit-hash>:path/to/file
```

## Examples

```bash
# History of cluster.yaml with a clean format
git log --pretty=format:"%h - %an, %ad : %s" --date=short cluster.yaml

# Who modified each line of kubeconfig
git blame kubeconfig

# When was "apiVersion" last changed in a manifest?
git log -S "apiVersion" --oneline deployment.yaml
```

## Quick Reference

| Command | Shows |
|---------|-------|
| `git log <file>` | Commits that changed the file |
| `git log -p <file>` | Those commits with diffs |
| `git log --follow <file>` | History across renames |
| `git log -S "text" <file>` | When a string was added/removed |
| `git blame <file>` | Last commit/author per line |
| `git blame -L 40,60 <file>` | Blame for a line range |
| `git show <hash>:<file>` | The file's contents at a commit |
