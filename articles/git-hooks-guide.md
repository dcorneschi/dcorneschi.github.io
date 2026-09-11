# Git Hooks Guide

Git hooks are scripts Git runs automatically at specific points in its workflow — before a commit, before a push, after a merge, and so on. They let you automate tasks and enforce policies: linting, running tests, blocking secrets, formatting commit messages. This guide covers how hooks work, the ones you'll use most, working examples, sharing them across a team, and their key limitation.

## How Hooks Work

Every repository has a `.git/hooks/` directory. A hook is simply an executable script named after the event it handles (`pre-commit`, `pre-push`, etc.) with no file extension. When the matching Git event fires, Git runs the script.

- Fresh repos ship `*.sample` files (e.g. `pre-commit.sample`) — examples that are inactive until you rename and enable them.
- **Exit code matters** for "pre" hooks: a non-zero exit **aborts** the operation (the commit or push doesn't happen). Zero lets it proceed.
- Hooks run with the repository root as the working directory.

## Client-Side vs Server-Side Hooks

- **Client-side** hooks run on your machine around local actions: `pre-commit`, `prepare-commit-msg`, `commit-msg`, `post-commit`, `pre-push`, `post-merge`, `post-checkout`.
- **Server-side** hooks run on the remote when it receives a push: `pre-receive`, `update`, `post-receive` — used for centralized policy enforcement that can't be bypassed by a contributor.

This guide focuses on client-side hooks, which are what most workflows use.

## Common Hooks

| Hook | Fires | Typical use |
|------|-------|-------------|
| `pre-commit` | Before the commit is created | Lint, format, scan for secrets — abort on failure |
| `prepare-commit-msg` | Before the message editor opens | Insert templates, branch name, or issue number |
| `commit-msg` | After the message is entered | Validate message format (e.g. Conventional Commits) |
| `post-commit` | After the commit succeeds | Notifications, docs updates |
| `pre-push` | Before pushing to a remote | Run the test suite, check branch policy — abort on failure |
| `post-merge` | After a merge (incl. `pull`) | Reinstall dependencies when a lockfile changed |

## Examples

### pre-commit — lint staged files

```bash
#!/bin/sh
# .git/hooks/pre-commit
echo "Running pre-commit checks..."

# Lint only staged JavaScript files
files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.js$')
if [ -n "$files" ]; then
    echo "$files" | xargs npx eslint || {
        echo "ESLint failed. Fix errors before committing."
        exit 1
    }
fi

echo "Pre-commit checks passed."
```

The `--diff-filter=ACM` limits to Added/Copied/Modified files so you don't lint deletions.

### pre-push — run tests

```bash
#!/bin/sh
# .git/hooks/pre-push
echo "Running tests before push..."

npm test || {
    echo "Tests failed. Push aborted."
    exit 1
}

echo "All tests passed."
```

### prepare-commit-msg — prefix the branch name

```bash
#!/bin/sh
# .git/hooks/prepare-commit-msg — $1 is the path to the commit message file
branch=$(git branch --show-current)

if [ -n "$branch" ] && ! grep -q "$branch" "$1"; then
    sed -i.bak -e "1s/^/[$branch] /" "$1" && rm -f "$1.bak"
fi
```

### post-commit — notify

```bash
#!/bin/sh
# .git/hooks/post-commit
hash=$(git rev-parse --short HEAD)
msg=$(git log -1 --pretty=%B)
echo "Committed $hash: $msg"
# ...forward to Slack, email, etc.
```

## Setting Up a Hook

```bash
# 1. Create or enable the hook (drop the .sample suffix if present)
mv .git/hooks/pre-commit.sample .git/hooks/pre-commit

# 2. Make it executable
chmod +x .git/hooks/pre-commit

# 3. Trigger the matching action to test it
git commit -m "test"
```

Bypass a client-side hook for a single command when you genuinely need to:

```bash
git commit --no-verify -m "wip"   # skip pre-commit and commit-msg
git push --no-verify              # skip pre-push
```

## Sharing Hooks Across a Team

The catch with `.git/hooks/` is that it is **not** committed or cloned — each person sets hooks up individually, and they're easy to forget. Two ways to share them:

**Commit a hooks directory and point Git at it:**

```bash
# Keep hooks in a tracked directory, e.g. .githooks/
git config core.hooksPath .githooks
```

Everyone still runs the `git config` line once (or you script it), but the hook scripts themselves live in the repo.

**Use a hook manager** that wires everything up on install:

- **pre-commit** — a language-agnostic framework configured via a committed `.pre-commit-config.yaml`; contributors run `pre-commit install` once.
- **Husky** — the common choice in the npm/Node ecosystem.
- **Lefthook** — a fast, parallel manager that works across ecosystems.

See [Blocking Commits with Trailing Whitespace](articles/git-block-trailing-whitespace-commits.md) for a concrete pre-commit hook plus the `pre-commit` framework in action.

## Notes and Gotchas

- **Local by default.** Hooks aren't pushed or pulled. Use `core.hooksPath` or a manager to share them.
- **Not a security boundary.** Anyone can skip a client-side hook with `--no-verify`; enforce hard requirements server-side or in CI.
- **Keep them fast.** A slow `pre-commit`/`pre-push` frustrates contributors — run only what's needed (e.g. lint staged files, not the whole tree).
- **Any executable works.** Hooks can be shell, Python, Node, etc. — the shebang line decides the interpreter; the file just needs the execute bit.

## Summary

- Hooks are executable scripts in `.git/hooks/` (or `core.hooksPath`) that fire on Git events.
- "Pre" hooks abort the action on a non-zero exit — use them for linting, tests, and policy checks.
- They're local to each clone; share via `core.hooksPath` or a manager (pre-commit, Husky, Lefthook).
- They're bypassable (`--no-verify`), so treat CI/server-side hooks as the real enforcement layer.
