# Fixing "git apply" Whitespace Errors

`git apply` often fails with `trailing whitespace` errors and refuses to apply a patch. This happens because Git's `apply.whitespace` policy treats whitespace errors (trailing spaces, blank-line-at-EOF, space-before-tab) as fatal by default in many setups. This guide shows how to apply the patch anyway, how to fix the root cause, and how to find and strip the offending whitespace.

> Related: [Detecting and Fixing Whitespace Errors in Git](articles/git-whitespace-errors-check-fix.md) and [Creating and Applying Git Patch Files](articles/git-create-apply-patches.md).

## Why It Happens

`git apply` inspects the patch for whitespace errors and, depending on the `apply.whitespace` setting (or the `--whitespace` flag), it can **warn**, **error out**, or **fix** them. When it errors, the patch is rejected before anything is applied — so the fix is either to relax that behavior or clean the patch.

## Quick Fixes

### Auto-fix whitespace (recommended)

```bash
git apply --whitespace=fix your-patch.patch
```

Applies the patch and strips the trailing whitespace as it goes — you get the change without importing the whitespace problems.

### Warn but apply

```bash
git apply --whitespace=warn your-patch.patch
```

Prints warnings but applies the patch as-is.

### Ignore whitespace differences entirely

```bash
git apply --whitespace=nowarn your-patch.patch   # silent
git apply --ignore-whitespace your-patch.patch   # ignore WS when matching context
```

`--ignore-whitespace` also helps when the patch fails to apply because context lines differ only in whitespace.

## Change the Default Behavior

```bash
# Stop treating whitespace as an error on future applies
git config --global apply.whitespace nowarn

# Or make fixing the default
git config --global apply.whitespace fix

# Check the current setting
git config --get apply.whitespace
```

Values: `nowarn` (ignore), `warn` (report, still apply), `error` (report and fail), `fix` (correct and apply).

## Clean the Patch File Manually

If you'd rather sanitize the patch itself:

```bash
# Strip trailing whitespace from every line
sed 's/[[:space:]]*$//' your-patch.patch > clean.patch
git apply clean.patch

# Also strip carriage returns (CRLF patches)
tr -d '\r' < your-patch.patch | sed 's/[[:space:]]*$//' > clean.patch
git apply clean.patch
```

Be aware: blindly stripping trailing whitespace from a patch can alter intended content in rare cases (e.g. a patch that legitimately adds trailing spaces). `--whitespace=fix` is usually safer because Git only touches the added lines.

## Finding Trailing Whitespace

### On the command line

```bash
cat -A file.txt                         # trailing spaces show as `$` at line ends
grep -n '[[:space:]]$' file.txt         # list lines (with numbers) that have trailing WS
awk '/[[:space:]]$/ {print NR": "$0}' file.txt
git diff --check                        # Git's own whitespace-error report
```

### In editors

**VS Code** — Settings → "Render Whitespace" → `all` or `trailing`; search with regex `\s+$`.

**Vim / Neovim**

```vim
:set list listchars=trail:·,tab:→\ ,eol:¬
/\s\+$                " search for trailing whitespace
:match ErrorMsg /\s\+$/   " highlight it
```

## Removing Trailing Whitespace

```bash
# A single file (GNU sed / Linux)
sed -i 's/[[:space:]]*$//' file.txt

# macOS / BSD sed needs an argument after -i
sed -i '' 's/[[:space:]]*$//' file.txt

# All matching files, recursively
find . -name "*.txt" -exec sed -i 's/[[:space:]]*$//' {} \;
```

## Which Approach to Use

- **Just get the patch applied cleanly** → `git apply --whitespace=fix` (best default — keeps your tree clean without importing the errors).
- **Patch fails on context whitespace** → `git apply --ignore-whitespace`.
- **You control the patch and want it clean at the source** → strip it with `sed`, then apply.
- **Tired of the errors on every apply** → set `apply.whitespace` (`fix` or `nowarn`) globally.

## Summary

- The error comes from Git's `apply.whitespace` policy rejecting whitespace errors.
- `--whitespace=fix` applies the patch and strips trailing whitespace — the recommended one-off fix.
- `--ignore-whitespace` helps when context lines differ only in whitespace.
- Set `apply.whitespace` to change the default; clean patches with `sed`/`tr` if you prefer.
- Find offenders with `git diff --check`, `cat -A`, or `grep '[[:space:]]$'`.
