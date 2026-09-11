# Ignoring Command Errors in Bash with || true

When a script runs under `set -e` (exit on error), any command that returns a non-zero exit status aborts the whole script. That's usually what you want — but some commands are *expected* to fail sometimes, and that failure is harmless. The `|| true` idiom lets a single command fail without taking the script down with it.

> `set -e` makes Bash exit as soon as any command returns non-zero. `command || true` forces the overall exit status to 0, so the script keeps going even when `command` fails.

## The Problem

Consider adding a Helm repository in a script that uses `set -e`. The first run succeeds; a second run fails because the repo already exists, and `set -e` kills the script:

```bash
set -e
helm repo add autoscaler https://kubernetes.github.io/autoscaler
# First run:  "autoscaler" has been added to your repositories
# Second run: Error: repository name (autoscaler) already exists
#             -> exit code 1 -> set -e aborts the script here
```

That "error" isn't a real problem — the repo is present, which is all we wanted.

## The Fix: || true

`||` runs the right-hand command only if the left-hand one fails. Since `true` always succeeds, the combined status is always 0:

```bash
set -e
helm repo add autoscaler https://kubernetes.github.io/autoscaler || true
# Second run still prints the error, but the script continues
```

`|| :` is an equivalent shorthand — `:` is a builtin that does nothing and returns 0:

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler || :
```

## Controlling the Noise

`|| true` swallows the *exit status*, but the error message still prints to stderr. Combine it with redirection when the message is just noise:

```bash
# Suppress stderr only (keep normal output)
helm repo add autoscaler https://kubernetes.github.io/autoscaler 2>/dev/null || true

# Suppress both stdout and stderr
helm repo add autoscaler https://kubernetes.github.io/autoscaler >/dev/null 2>&1 || true
```

## Alternative: Check First (Idempotent)

`|| true` hides *all* failures, including unexpected ones. When you want to ignore only the "already exists" case and still catch genuine errors, test for the condition explicitly:

```bash
if ! helm repo list 2>/dev/null | grep -q '^autoscaler'; then
    helm repo add autoscaler https://kubernetes.github.io/autoscaler
fi
```

This is more precise: a real failure (bad URL, network error) on the `helm repo add` will still trip `set -e`, because you only skip the command when the repo genuinely exists.

## Pitfalls

### It Masks Real Failures

`|| true` makes the command *always* succeed — so a typo, a permissions error, or a network outage all pass silently. Use it only where failure is genuinely acceptable, not as a blanket "make the error go away" tool.

### Capturing the Real Exit Code

If you need to react differently depending on why a command failed, don't use `|| true`. Capture the status and branch on it:

```bash
set +e
helm repo add autoscaler https://kubernetes.github.io/autoscaler
status=$?
set -e

if [ "$status" -ne 0 ]; then
    echo "repo add returned $status (continuing anyway)" >&2
fi
```

### Interaction with pipefail

Under `set -o pipefail`, a pipeline fails if *any* stage fails. Put `|| true` on the whole pipeline, not just the last command, or the earlier failure can still surface:

```bash
set -eo pipefail
# Guard the entire pipeline:
{ some_command | grep something; } || true
```

### It Only Neutralizes That One Command

`|| true` binds to the single command (or `{ ...; }` group) it's attached to. Later commands in the script still run under `set -e` as normal.

## When to Use || true

- Commands that fail harmlessly on repeat runs (adding a repo, creating an existing resource).
- Optional steps that shouldn't stop the script.
- Cleanup commands that may have nothing to clean (`rm`, `kill`, `docker rm`).
- CI/CD steps where an expected non-zero is not an error.

## When Not to Use || true

- Critical operations that must succeed for the script to be correct.
- Commands where a non-zero status signals a real problem you should handle.
- Cases where you need to branch on *why* something failed — capture `$?` instead.

## Summary

| Technique | Effect |
|-----------|--------|
| `cmd \|\| true` | Ignore failure, keep going; error message still prints |
| `cmd \|\| :` | Same as `\|\| true` (`:` is the no-op builtin) |
| `cmd 2>/dev/null \|\| true` | Ignore failure and hide the error message |
| `if ! check; then cmd; fi` | Run only when needed; still catches real failures |
| capture `$?` | Ignore failure but inspect/branch on the exact status |

`|| true` makes scripts idempotent and resilient against *expected* failures. Reach for the check-first pattern or an explicit exit-code check when you care about telling expected failures apart from real ones.
