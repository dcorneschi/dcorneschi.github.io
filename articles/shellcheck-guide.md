# ShellCheck Guide

ShellCheck is a static analysis tool for shell scripts. It catches common bugs, style issues, and pitfalls in bash/sh/dash/ksh scripts — things that are syntactically valid but almost certainly wrong.

## Installation

```bash
# Ubuntu / Debian
sudo apt install shellcheck

# RHEL / Rocky / AlmaLinux (EPEL required)
sudo dnf install epel-release
sudo dnf install ShellCheck

# macOS (Homebrew)
brew install shellcheck

# Arch Linux
sudo pacman -S shellcheck

# From binary (any Linux)
VERSION="0.10.0"
curl -fsSL "https://github.com/koalaman/shellcheck/releases/download/v${VERSION}/shellcheck-v${VERSION}.linux.x86_64.tar.xz" | tar -xJf -
sudo cp shellcheck-v${VERSION}/shellcheck /usr/local/bin/
shellcheck --version

# Docker
docker run --rm -v "$PWD:/mnt" koalaman/shellcheck:stable script.sh
```

## Basic Usage

```bash
# Check a single script
shellcheck script.sh

# Check multiple scripts
shellcheck *.sh
shellcheck scripts/*.sh lib/*.sh

# Check with specific shell dialect
shellcheck -s bash script.sh
shellcheck -s sh script.sh
shellcheck -s dash script.sh
shellcheck -s ksh script.sh

# Check from stdin
echo 'echo $foo' | shellcheck -s bash -

# Recursive check (find all shell scripts)
find . -name "*.sh" -exec shellcheck {} +

# Or using fd
fd -e sh -x shellcheck
```

## Output Formats

```bash
# Default (human-readable with caret markers)
shellcheck script.sh

# GCC format (file:line:col: severity: message)
shellcheck -f gcc script.sh

# JSON output (for automation)
shellcheck -f json script.sh

# JSON1 (simplified JSON, one object per finding)
shellcheck -f json1 script.sh

# checkstyle XML (for CI tools like Jenkins)
shellcheck -f checkstyle script.sh

# diff format (shows suggested fixes)
shellcheck -f diff script.sh

# TTY (colored, default when output is a terminal)
shellcheck -f tty script.sh

# Quiet (only exit code, no output)
shellcheck -f quiet script.sh
```

### Apply Fixes Automatically

```bash
# Generate diff and apply
shellcheck -f diff script.sh | patch -p1

# Or review the diff first
shellcheck -f diff script.sh | less
```

## Severity Levels

| Level | Flag | Description |
|-------|------|-------------|
| error | `-S error` | Definite bugs (will cause failures) |
| warning | `-S warning` | Likely issues (probably wrong) |
| info | `-S info` | Style suggestions (could be better) |
| style | `-S style` | Pure style/cosmetic |

```bash
# Only show errors and warnings (skip info/style)
shellcheck -S warning script.sh

# Only show errors
shellcheck -S error script.sh
```

## Common Warnings and Fixes

### SC2086 — Double-quote to prevent globbing and word splitting

The most common warning. Unquoted variables undergo word splitting and glob expansion.

```bash
# BAD — SC2086
name="hello world"
echo $name        # Splits into two words
rm $file          # Dangerous if file contains spaces or globs

# GOOD
echo "$name"
rm "$file"
```

### SC2046 — Quote command substitution to prevent splitting

```bash
# BAD — SC2046
files=$(find . -name "*.txt")
rm $files          # Word splitting on filenames with spaces

# GOOD
while IFS= read -r -d '' file; do
    rm "$file"
done < <(find . -name "*.txt" -print0)
```

### SC2006 — Use $(...) instead of legacy backticks

```bash
# BAD — SC2006
date=`date +%Y-%m-%d`

# GOOD
date=$(date +%Y-%m-%d)
```

### SC2015 — Note that A && B || C is not if-then-else

```bash
# BAD — SC2015 (C runs if B fails, not just if A fails)
[ -f file ] && cat file || echo "missing"

# GOOD
if [ -f file ]; then
    cat file
else
    echo "missing"
fi
```

### SC2034 — Variable appears unused

```bash
# SC2034 — often a false positive for sourced scripts
MY_VAR="value"  # Warning if not used in same file

# Fix: export it or use a directive
export MY_VAR="value"
# Or: # shellcheck disable=SC2034
```

### SC2155 — Declare and assign separately to avoid masking return values

```bash
# BAD — SC2155 (local masks the exit code of command)
local output=$(command_that_might_fail)

# GOOD
local output
output=$(command_that_might_fail)
```

### SC2164 — Use cd ... || exit in case cd fails

```bash
# BAD — SC2164
cd /some/path
rm -rf *          # Dangerous if cd failed!

# GOOD
cd /some/path || exit 1
rm -rf *

# Or
cd /some/path || { echo "Failed to cd"; exit 1; }
```

### SC2162 — read without -r mangles backslashes

```bash
# BAD — SC2162
read input

# GOOD
read -r input
```

### SC2181 — Check exit code directly with if cmd

```bash
# BAD — SC2181 (less readable)
command
if [ $? -eq 0 ]; then
    echo "success"
fi

# GOOD
if command; then
    echo "success"
fi
```

### SC2129 — Consider using { cmd1; cmd2; } >> file

```bash
# BAD — SC2129 (multiple redirections to same file)
echo "line1" >> file.txt
echo "line2" >> file.txt
echo "line3" >> file.txt

# GOOD
{
    echo "line1"
    echo "line2"
    echo "line3"
} >> file.txt
```

### SC2236 — Use -z/-n instead of ! -n/! -z

```bash
# BAD — SC2236
if [ ! -z "$var" ]; then

# GOOD
if [ -n "$var" ]; then
```

### SC2009 — Consider using pgrep instead of grep ps

```bash
# BAD — SC2009
ps aux | grep nginx | grep -v grep

# GOOD
pgrep -f nginx
```

### SC2012 — Use find instead of ls to better handle non-alphanumeric filenames

```bash
# BAD — SC2012
for f in $(ls *.txt); do

# GOOD
for f in *.txt; do
# Or
find . -name "*.txt" -exec ... \;
```

### SC2143 — Use grep -q instead of comparing output

```bash
# BAD — SC2143
if [ "$(grep -c error log.txt)" -gt 0 ]; then

# GOOD
if grep -q error log.txt; then
```

### SC2029 — Expansion happens locally, not on remote

```bash
# BAD — SC2029 (variable expands locally before SSH)
ssh server "echo $REMOTE_VAR"

# GOOD (single quotes prevent local expansion)
ssh server 'echo $REMOTE_VAR'

# Or escape the dollar sign
ssh server "echo \$REMOTE_VAR"
```

### SC2091 — Remove surrounding $() to avoid executing output as command

```bash
# BAD — SC2091
$(grep pattern file)   # Executes the output as a command!

# GOOD
grep pattern file      # Just run the command
result=$(grep pattern file)  # Or capture output
```

## Disabling Warnings

### Inline Directive (Next Line)

```bash
# shellcheck disable=SC2086
echo $unquoted_var_on_purpose
```

### Inline Directive (Same Line)

```bash
echo $var  # shellcheck disable=SC2086
```

### Disable for Entire Block

```bash
# shellcheck disable=SC2086,SC2046
{
    echo $var1
    echo $var2
    files=$(find . -name "*.log")
    rm $files
}
```

### Disable for Entire File (Top of File)

```bash
#!/bin/bash
# shellcheck disable=SC2086,SC2034
# This script uses unquoted variables intentionally

echo $PATH
UNUSED_VAR="needed by sourcing script"
```

### Enable Specific Check Only

```bash
# shellcheck enable=require-variable-braces
echo "$var"   # OK
echo "$var"   # Would warn without braces: use ${var}
```

## Directives Reference

| Directive | Effect |
|-----------|--------|
| `# shellcheck disable=SCxxxx` | Suppress specific warning |
| `# shellcheck disable=SCxxxx,SCyyyy` | Suppress multiple warnings |
| `# shellcheck source=path/to/file` | Specify sourced file for analysis |
| `# shellcheck source-path=dir` | Set search path for sourced files |
| `# shellcheck shell=bash` | Override shell detection |
| `# shellcheck enable=name` | Enable optional check |
| `# shellcheck disable=all` | Suppress all warnings (use sparingly) |

### Source Directives

```bash
# Tell shellcheck where sourced files are
# shellcheck source=lib/common.sh
source "$SCRIPT_DIR/lib/common.sh"

# Or use source-path for multiple files
# shellcheck source-path=lib
source "$SCRIPT_DIR/lib/utils.sh"
source "$SCRIPT_DIR/lib/config.sh"
```

## Optional Checks

These aren't enabled by default but can be turned on:

```bash
# Require braces around all variables
# shellcheck enable=require-variable-braces
echo "${var}"  # Required
echo "$var"    # Would warn

# Require double brackets in tests
# shellcheck enable=require-double-brackets
[[ -f file ]]  # Required
[ -f file ]    # Would warn

# Deprecate which command
# shellcheck enable=deprecate-which
command -v cmd  # Preferred
which cmd       # Would warn
```

```bash
# Enable optional checks from command line
shellcheck --enable=require-variable-braces,require-double-brackets script.sh
```

## CI/CD Integration

### GitHub Actions

```yaml
name: ShellCheck
on: [push, pull_request]

jobs:
  shellcheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run ShellCheck
        uses: ludeeus/action-shellcheck@master
        with:
          severity: warning
          scandir: './scripts'
```

### GitLab CI

```yaml
shellcheck:
  image: koalaman/shellcheck-alpine:stable
  stage: lint
  script:
    - find . -name "*.sh" -exec shellcheck -S warning {} +
  allow_failure: false
```

### Pre-commit Hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/koalaman/shellcheck-precommit
    rev: v0.10.0
    hooks:
      - id: shellcheck
        args: ['-S', 'warning']
```

### Makefile

```makefile
.PHONY: lint lint-shell

lint-shell:
	@echo "Running ShellCheck..."
	@find . -name "*.sh" -not -path "./.git/*" -exec shellcheck -S warning {} +

lint: lint-shell
```

### Jenkins

```groovy
stage('ShellCheck') {
    steps {
        sh 'find . -name "*.sh" -exec shellcheck -f checkstyle {} + > shellcheck-report.xml || true'
        recordIssues tools: [checkStyle(pattern: 'shellcheck-report.xml')]
    }
}
```

## .shellcheckrc (Project Configuration)

Create `.shellcheckrc` in your project root to set defaults:

```bash
# .shellcheckrc

# Default shell
shell=bash

# Disable globally for this project
disable=SC2034,SC2059

# Enable optional checks
enable=require-variable-braces

# Set source path
source-path=lib:scripts

# External sources (don't warn about unresolvable sources)
external-sources=true
```

ShellCheck looks for `.shellcheckrc` in the script's directory, parent directories, and `~/.shellcheckrc`.

## Editor Integration

| Editor | Plugin/Extension |
|--------|-----------------|
| VS Code | `timonwong.shellcheck` (ShellCheck extension) |
| Vim/Neovim | ALE, Syntastic, or native LSP with `bash-language-server` |
| Emacs | Flycheck with `flycheck-shellcheck` |
| Sublime Text | SublimeLinter-shellcheck |
| IntelliJ/JetBrains | Built-in Shell Script plugin |

## Common Patterns

### Script with Full Linting

```bash
#!/bin/bash
# shellcheck disable=SC1091  # Don't follow sourced files that aren't available
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# shellcheck source=lib/common.sh
source "${SCRIPT_DIR}/lib/common.sh"

main() {
    local env="${1:-dev}"
    local config_file="${SCRIPT_DIR}/configs/${env}.conf"

    if [[ ! -f "$config_file" ]]; then
        echo "Error: Config file not found: $config_file" >&2
        exit 1
    fi

    # shellcheck source=/dev/null  # Config file varies at runtime
    source "$config_file"

    echo "Deploying to ${env} environment"
}

main "$@"
```

### Intentionally Unquoted (With Explanation)

```bash
# Word splitting is intentional here — flags should split
# shellcheck disable=SC2086
docker run $DOCKER_FLAGS --name myapp myimage:latest

# Or use arrays (preferred, no shellcheck warning)
docker_flags=(-d --restart=always --network=host)
docker run "${docker_flags[@]}" --name myapp myimage:latest
```

### Working with Arrays

```bash
# ShellCheck-compliant array patterns
files=()
while IFS= read -r -d '' file; do
    files+=("$file")
done < <(find . -name "*.txt" -print0)

# Process array
for file in "${files[@]}"; do
    echo "Processing: $file"
done
```

## Exit Codes

| Exit Code | Meaning |
|-----------|---------|
| 0 | No issues found (at the specified severity) |
| 1 | Issues found |
| 2 | ShellCheck itself encountered an error |
| 3 | Invalid arguments |
| 4 | File not found |

```bash
# Use in scripts
shellcheck -S warning script.sh
if [ $? -eq 0 ]; then
    echo "No issues found"
else
    echo "Issues detected"
fi

# Or directly
if shellcheck -S error script.sh; then
    echo "No errors"
fi
```

## Quick Reference

```bash
# Check a script
shellcheck script.sh

# Specific severity
shellcheck -S warning script.sh

# Specific shell
shellcheck -s bash script.sh

# Exclude specific warnings
shellcheck -e SC2086,SC2034 script.sh

# Enable optional checks
shellcheck --enable=require-variable-braces script.sh

# Output format for CI
shellcheck -f gcc script.sh

# Auto-fix (generate patch)
shellcheck -f diff script.sh | patch -p1

# Check all scripts recursively
find . -name "*.sh" -exec shellcheck {} +

# Docker (no install needed)
docker run --rm -v "$PWD:/mnt" koalaman/shellcheck:stable /mnt/script.sh
```

## Most Important Rules to Know

| Code | Summary | Impact |
|------|---------|--------|
| SC2086 | Quote your variables | Word splitting, glob expansion, security |
| SC2046 | Quote command substitution | Same as above for `$(...)` |
| SC2006 | Use `$(...)` not backticks | Readability, nesting support |
| SC2155 | Declare and assign separately | Masked exit codes |
| SC2164 | `cd` can fail — handle it | Running commands in wrong directory |
| SC2015 | `A && B \|\| C` isn't if-then-else | Unexpected C execution |
| SC2029 | SSH expansion happens locally | Remote scripts get wrong values |
| SC2162 | Use `read -r` | Backslash mangling |
| SC2181 | Check exit code directly | Readability |
| SC2012 | Don't parse `ls` | Broken on special filenames |
