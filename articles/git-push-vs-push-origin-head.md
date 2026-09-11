# git push vs git push origin HEAD

Both commands push your current branch, but they differ in how Git decides *what* to push and *where*. `git push` relies on configured upstream tracking; `git push origin HEAD` is explicit and works even when no upstream is set. This guide explains each, the `push.default` setting behind the difference, and when to reach for which.

## git push

```bash
git push
```

With no arguments, `git push` uses your `push.default` configuration to decide the target. On Git 2.0+ the default is `simple`:

- It pushes the **current branch** to its configured **upstream** branch of the same name.
- It requires an upstream to already be set.
- If no upstream is configured, it fails with a hint to run `git push -u origin <branch>`.

So `git push` is convenient for established branches that already track a remote, but it does nothing useful on a brand-new local branch until you set tracking.

## git push origin HEAD

```bash
git push origin HEAD
```

This is explicit about both the remote and the ref:

- `origin` names the remote directly.
- `HEAD` resolves to whatever branch you're currently on.
- Git creates a same-named branch on `origin` if it doesn't exist yet.
- It works whether or not an upstream is configured.

Because `HEAD` expands to the current branch name, `git push origin HEAD` is equivalent to `git push origin <current-branch>` without you typing the name — handy and safe against typos.

## Key Differences

| Aspect | `git push` | `git push origin HEAD` |
|--------|-----------|------------------------|
| Upstream required | Yes | No |
| Remote specified | No — uses configured upstream | Yes (`origin`) |
| Creates missing remote branch | Only via upstream setup | Yes |
| Works on a fresh branch | No (until `-u` is set) | Yes |
| Sets up tracking | Only with `-u` | No (unless you add `-u`) |

## The push.default Setting

`git push`'s behavior is governed by `push.default`:

```bash
# See the current value
git config push.default
```

| Value | Behavior of bare `git push` |
|-------|-----------------------------|
| `simple` (default 2.0+) | Push current branch to same-named upstream; refuse if names differ |
| `current` | Push current branch to a same-named branch on the remote, **creating it if needed** |
| `upstream` | Push to the upstream branch even if named differently |
| `matching` (old default) | Push **all** branches with a matching name on the remote |
| `nothing` | Refuse to push without an explicit refspec |

Notably, setting `push.default = current` makes a bare `git push` behave much like `git push origin HEAD` — it'll create the remote branch on first push:

```bash
git config --global push.default current
```

## Setting Upstream Tracking

To make future bare `git push` (and `git pull`) work on a new branch, set the upstream once:

```bash
# Push and set the upstream in one go
git push -u origin HEAD

# Equivalent explicit form
git push --set-upstream origin my-branch
```

After this, plain `git push` and `git pull` target the tracked branch. You can also let Git set upstreams automatically on every first push:

```bash
git config --global push.autoSetupRemote true
```

With that enabled, a bare `git push` on a new branch creates and tracks the remote branch automatically — closing most of the gap between the two commands.

## Practical Scenarios

```bash
# New local branch, no upstream yet
git push                 # fails: no upstream configured
git push origin HEAD     # works: creates the branch on origin
git push -u origin HEAD  # works AND sets tracking for next time

# Existing branch that already tracks origin
git push                 # works
git push origin HEAD     # also works — same result
```

## When to Use Each

Use `git push` when:

- The branch already has an upstream.
- You want to rely on configured tracking for a consistent team workflow.

Use `git push origin HEAD` when:

- Pushing a new branch for the first time.
- You want to be explicit about the remote.
- No upstream is configured.
- You're writing scripts or automation where predictability matters.

## Pro Tip: an Alias

```bash
git config --global alias.ph 'push origin HEAD'
```

Then `git ph` pushes the current branch to `origin` in any situation. For first-time pushes that also set tracking:

```bash
git config --global alias.phu 'push -u origin HEAD'
```

## Summary

- `git push` = "push to my configured upstream" — convenient, but needs tracking set up first.
- `git push origin HEAD` = "push my current branch to origin explicitly" — works everywhere, creates the remote branch if missing.
- Set `push.autoSetupRemote true` (or use `git push -u origin HEAD` once) to make bare `git push` just work on new branches.
