# Blocking Commits with Trailing Whitespace

Trailing spaces and blank-line-at-EOF noise clutter diffs and reviews. You can make Git *reject* commits that introduce them, so the rule is enforced automatically rather than relying on discipline. This guide covers Git's built-in whitespace config, a pre-commit hook (the reliable approach), the cross-team `pre-commit` framework, and editor settings that prevent the problem at the source.

> This is the *enforcement* side. For detecting and fixing whitespace that's already there (`git diff --check`, `git rebase --whitespace=fix`, `.gitattributes`), see [Detecting and Fixing Whitespace Errors in Git](articles/git-whitespace-errors-check-fix.md).

## Method 1: Git's Built-in Whitespace Config

```bash
# This repo only
git config core.whitespace trailing-space,space-before-tab
git config apply.whitespace error

# Or globally
git config --global core.whitespace trailing-space,space-before-tab
git config --global apply.whitespace error
```

This makes `git apply` and patch operations flag or reject whitespace errors. The catch: it does **not** block a normal `git commit` on its own — it governs patch application. To reject commits you need a hook (Method 2), though `core.whitespace` still defines *what counts* as an error for `git diff --check`, which the hook uses.

## Method 2: A Pre-Commit Hook (recommended)

A `pre-commit` hook runs before each commit and can abort it by exiting non-zero. This is the most reliable per-repo enforcement.

### Simple version

```bash
#!/bin/sh
# .git/hooks/pre-commit — reject staged trailing whitespace

if git rev-parse --verify HEAD >/dev/null 2>&1; then
    against=HEAD
else
    # Initial commit: diff against the empty tree
    against=$(git hash-object -t tree /dev/null)
fi

# --check exits non-zero on whitespace errors, blocking the commit
exec git diff-index --check --cached $against --
```

### Version with friendlier output

```bash
#!/bin/sh
# .git/hooks/pre-commit — reject trailing whitespace, with guidance

echo "Checking for trailing whitespace..."

if git rev-parse --verify HEAD >/dev/null 2>&1; then
    against=HEAD
else
    against=$(git hash-object -t tree /dev/null)
fi

offenders=$(git diff-index --check --cached $against -- 2>&1 \
    | grep "trailing whitespace" | cut -d: -f1 | sort -u)

if [ -n "$offenders" ]; then
    echo "Commit rejected — trailing whitespace in:"
    echo "$offenders" | sed 's/^/  - /'
    echo ""
    echo "Fix with:  git diff --check     (see the issues)"
    echo "           sed -i 's/[[:space:]]*$//' <file>   (strip them)"
    exit 1
fi

echo "No trailing whitespace found."
exit 0
```

### Install it

```bash
# Save the script as .git/hooks/pre-commit, then:
chmod +x .git/hooks/pre-commit
```

`git diff-index --check --cached` inspects **staged** content: it exits 1 (blocking the commit) when it finds trailing whitespace or other `core.whitespace` errors, and 0 otherwise.

> Note: `.git/hooks/` is not committed or shared. Each clone needs the hook installed, which is why teams often prefer Method 3 or a tracked hooks directory (`git config core.hooksPath .githooks`).

## Method 3: The pre-commit Framework (best for teams)

The [`pre-commit`](https://pre-commit.com) framework stores hook config *in the repo*, so everyone gets the same checks after a one-line install.

```bash
pip install pre-commit          # or: brew install pre-commit
```

Create `.pre-commit-config.yaml` at the repo root:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-merge-conflict
      - id: check-yaml
      - id: check-json
```

Then activate it in each clone:

```bash
pre-commit install

# Optionally run against the whole repo once
pre-commit run --all-files
```

Because the config is committed, `pre-commit install` is the only per-clone step — the checks themselves travel with the repo.

## Testing the Hook

```bash
printf 'line with trailing spaces   \n' > test-file.txt
git add test-file.txt
git commit -m "test"
# → the commit is rejected, pointing at test-file.txt
```

Clean up and retry once the trailing spaces are removed.

## Cleaning Up Existing Files

```bash
# Strip trailing whitespace from Markdown files (GNU sed / Linux)
find . -name "*.md" -exec sed -i 's/[[:space:]]*$//' {} \;

# macOS/BSD sed needs an argument after -i
find . -name "*.md" -exec sed -i '' 's/[[:space:]]*$//' {} \;
```

## Prevent It at the Source: Editor Settings

Hooks are the backstop; trimming on save means offenders rarely reach the hook.

**VS Code** (`settings.json`):

```json
{
  "files.trimTrailingWhitespace": true,
  "files.trimFinalNewlines": true
}
```

**Vim** (`.vimrc`):

```vim
autocmd BufWritePre * :%s/\s\+$//e
```

**Sublime Text** (settings):

```json
{
  "trim_trailing_white_space_on_save": true
}
```

## Summary

- `core.whitespace` defines what counts as an error but doesn't block commits by itself — it governs `git apply` and feeds `git diff --check`.
- A `pre-commit` hook running `git diff-index --check --cached` reliably rejects offending commits per repo (remember hooks aren't shared).
- The `pre-commit` framework commits the config so a whole team gets the same checks with one `pre-commit install`.
- Enable trim-on-save in your editor so trailing whitespace rarely gets created in the first place.
