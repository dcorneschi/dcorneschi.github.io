# Detecting and Fixing Whitespace Errors in Git

Trailing spaces, blank lines at end of file, and stray tabs are easy to introduce and annoying in diffs and code review. Git can both **detect** these whitespace errors and **fix** them across a range of commits. This guide covers the two core commands, what counts as a whitespace error, and how to configure Git to flag or block them automatically.

## The Two Core Commands

```bash
# Report whitespace errors in your changes relative to main
git diff main --check

# Rewrite your commits on top of main, fixing whitespace as it replays them
git rebase --whitespace=fix main
```

- `git diff --check` **reports** problems without changing anything.
- `git rebase --whitespace=fix` **fixes** them by reapplying each commit with the whitespace corrected.

The typical flow is: check first, then fix.

## Checking for Whitespace Errors

```bash
# Check your working-tree changes against main
git diff main --check

# Check staged changes only
git diff --cached --check

# Check unstaged changes in the working tree
git diff --check
```

Output points at the exact file and line, for example:

```text
src/app.js:42: trailing whitespace.
+    const x = 1;   
```

A non-empty result means there are issues to fix; the command exits non-zero, which makes it useful in scripts and CI.

## What Counts as a Whitespace Error

Git's default `core.whitespace` checks are:

| Error | Meaning |
|-------|---------|
| `blank-at-eol` | Trailing whitespace at the end of a line |
| `blank-at-eof` | Blank line(s) at the end of a file |
| `space-before-tab` | A space appearing before a tab in the indentation |

Optional (off by default) checks you can enable:

| Error | Meaning |
|-------|---------|
| `indent-with-non-tab` | Line indented with spaces instead of a tab |
| `tab-in-indent` | Tab used in indentation (opposite preference) |
| `tabwidth=<n>` | Sets how wide a tab is for the above checks |

Configure which ones apply:

```bash
# Example: flag trailing whitespace and space-before-tab, treat tabs as 4 wide
git config --global core.whitespace "blank-at-eol,space-before-tab,tabwidth=4"
```

## Fixing Whitespace Errors

### Across a range of commits (rebase)

```bash
git rebase --whitespace=fix main
```

This replays every commit from `main..HEAD`, stripping the whitespace errors as it goes. Because it rewrites commit hashes, treat it like any rebase: fine on a private branch, coordinate first if others have pulled it. If the branch was already pushed, update the remote with:

```bash
git push --force-with-lease
```

### Just the current changes (no rebase)

To clean the working tree without rewriting history, re-apply your own diff through `git apply`, which honors `--whitespace=fix`:

```bash
git diff | git apply --whitespace=fix -
```

Or fix a single file directly with standard tools:

```bash
# Strip trailing whitespace from a file (macOS/BSD sed)
sed -i '' 's/[[:space:]]*$//' path/to/file

# GNU sed (Linux)
sed -i 's/[[:space:]]*$//' path/to/file
```

## Prevent Whitespace Errors on Apply and Commit

### Reject or warn when applying patches

```bash
# Refuse to apply a patch that introduces whitespace errors
git apply --whitespace=error patch.diff

# Warn but still apply
git apply --whitespace=warn patch.diff
```

`git config apply.whitespace fix` makes `git apply` fix whitespace automatically by default.

### Block bad commits with the sample pre-commit hook

Git ships a sample hook that runs `git diff-index --check` before each commit. Enable it:

```bash
mv .git/hooks/pre-commit.sample .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
```

With it active, commits containing whitespace errors are rejected until you clean them up.

## Highlighting Whitespace in Diffs

Make errors visible whenever you read a diff:

```bash
# Color whitespace errors in diff output
git config --global color.diff.whitespace "red reverse"

# Emphasize whitespace changes in the diff itself
git diff --ws-error-highlight=all main
```

## Summary

- **Detect:** `git diff main --check` (also `--cached` or plain `--check`).
- **Fix a range:** `git rebase --whitespace=fix main`, then `git push --force-with-lease` if already pushed.
- **Fix current changes only:** `git diff | git apply --whitespace=fix -`.
- **Prevent:** enable the sample `pre-commit` hook and/or set `apply.whitespace`.
- Tune what counts via `core.whitespace`.
