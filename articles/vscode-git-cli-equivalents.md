# VS Code Git Actions and Their Git CLI Equivalents

VS Code's built-in Git support (the **Source Control** panel, the **Command Palette**, and status-bar controls) drives the same `git` commands you'd run in a terminal. This maps the common VS Code actions to their CLI equivalents — handy for learning what the UI does under the hood, and for scripting the same operations.

> Open Source Control with **Ctrl/Cmd+Shift+G**. Git commands in the Command Palette are prefixed with **Git:** (open it with **Ctrl/Cmd+Shift+P**). VS Code runs the same `git` binary on your `PATH`, so the CLI equivalents below are exactly what it invokes.

## Repository Setup

| VS Code action | Git CLI |
|----------------|---------|
| **Initialize Repository** (Source Control) / `Git: Initialize Repository` | `git init` |
| **Clone Repository** / `Git: Clone` | `git clone <url>` |
| Open a folder that's already a repo | (nothing — VS Code detects `.git`) |
| `Git: Add Remote` | `git remote add <name> <url>` |

## Staging and Committing

| VS Code action | Git CLI |
|----------------|---------|
| Hover a file → **+** (Stage Changes) | `git add <file>` |
| **+** on the *Changes* group header (Stage All) | `git add -A` |
| Hover a file → **−** (Unstage Changes) | `git restore --staged <file>` (or `git reset HEAD <file>`) |
| Stage a selected block → **Stage Selected Ranges** | `git add -p <file>` (interactive hunks) |
| Type a message → **✓ Commit** | `git commit -m "message"` |
| **Commit** with nothing staged (stages all tracked first, if enabled) | `git commit -am "message"` |
| `Git: Commit Staged (Amend)` / **Commit** → Amend | `git commit --amend` |
| Discard a file's changes (**↩ / trash icon**) | `git restore <file>` (or `git checkout -- <file>`) |
| Discard **all** changes | `git restore .` |

> VS Code's plain **Commit** button commits **staged** changes. If nothing is staged, it can be configured (setting `git.enableSmartCommit`) to stage and commit all tracked changes — the equivalent of `git commit -a`.

## Branches

| VS Code action | Git CLI |
|----------------|---------|
| Status bar branch name → **Create new branch** / `Git: Create Branch` | `git switch -c <branch>` (or `git checkout -b <branch>`) |
| Status bar branch name → pick a branch / `Git: Checkout to` | `git switch <branch>` (or `git checkout <branch>`) |
| `Git: Create Branch From` | `git switch -c <branch> <start-point>` |
| `Git: Rename Branch` | `git branch -m <newname>` |
| `Git: Delete Branch` | `git branch -d <branch>` (`-D` to force) |
| `Git: Merge Branch` | `git merge <branch>` |
| `Git: Rebase Branch` | `git rebase <branch>` |

## Syncing with a Remote

| VS Code action | Git CLI |
|----------------|---------|
| **Sync Changes** (status bar ↻) | `git pull` then `git push` |
| `Git: Pull` | `git pull` |
| `Git: Push` | `git push` |
| Push a brand-new branch / `Git: Push to...` | `git push -u origin <branch>` |
| `Git: Fetch` | `git fetch` |
| `Git: Fetch (Prune)` | `git fetch --prune` |
| `Git: Pull (Rebase)` | `git pull --rebase` |
| **Publish Branch** (for a local-only branch) | `git push -u origin <branch>` |

> The status-bar **Sync** button does a pull **and** a push. The number next to it (e.g. `↓2 ↑1`) shows commits to pull and to push relative to the upstream.

## Viewing History and Differences

| VS Code action | Git CLI |
|----------------|---------|
| Click a changed file (opens the diff editor) | `git diff <file>` (unstaged) / `git diff --staged <file>` |
| **Timeline** view (bottom of Explorer) | `git log --follow <file>` |
| `Git: View History` (or the Git Graph / built-in graph) | `git log --oneline --graph --decorate` |
| Hover in the editor gutter → line blame (with GitLens) | `git blame <file>` |
| Compare with another branch | `git diff <branch>..<branch>` |

## Stashing

| VS Code action | Git CLI |
|----------------|---------|
| `Git: Stash` | `git stash` |
| `Git: Stash (Include Untracked)` | `git stash -u` |
| `Git: Pop Stash` | `git stash pop` |
| `Git: Apply Stash` | `git stash apply` |
| `Git: Drop Stash` | `git stash drop` |
| `Git: Stash with Message` | `git stash push -m "message"` |

## Undoing Things

| VS Code action | Git CLI |
|----------------|---------|
| `Git: Undo Last Commit` (keeps changes, unstaged) | `git reset --soft HEAD~1` (VS Code uses a mixed-style undo, keeping working changes) |
| Discard a file (Source Control) | `git restore <file>` |
| Revert a commit (`Git: Revert` / from a graph view) | `git revert <commit>` |
| `Git: Checkout to` a previous commit (detached) | `git checkout <commit>` |

> VS Code's **Undo Last Commit** removes the commit but keeps your changes in the working tree (like `git reset --soft`/mixed), so you can re-commit. It does **not** discard your edits.

## Tags

| VS Code action | Git CLI |
|----------------|---------|
| `Git: Create Tag` | `git tag <name>` (or `git tag -a <name> -m "msg"`) |
| `Git: Delete Tag` | `git tag -d <name>` |
| Push tags (`Git: Push (Follow Tags)`) | `git push --follow-tags` (or `git push --tags`) |

## Merge Conflicts

| VS Code action | Git CLI |
|----------------|---------|
| Conflict markers → **Accept Current / Incoming / Both** in the editor | edit the file, then `git add <file>` |
| **Complete Merge** after resolving | `git commit` (finishes the merge) |
| `Git: Abort Merge` | `git merge --abort` |
| 3-way **Merge Editor** (Resolve in Merge Editor) | manual resolve + `git add` |

## Handy Command Palette Entries

```text
Git: Clone
Git: Initialize Repository
Git: Checkout to...
Git: Create Branch...
Git: Merge Branch...
Git: Rebase Branch...
Git: Pull
Git: Push
Git: Sync
Git: Fetch
Git: Stash
Git: Pop Stash
Git: Show Git Output          # opens the Output panel showing the actual git commands run
```

> **See exactly what VS Code runs:** open the **Output** panel (**View → Output**) and select **Git** from the dropdown (or run `Git: Show Git Output`). Every underlying `git` command and its result is logged there — the definitive way to learn the CLI equivalent of any UI action.

## Notes

- VS Code uses whatever `git` is on your `PATH`; set it explicitly with the `git.path` setting if needed.
- Some actions above (line blame, rich history graph, interactive rebase UI) come from the popular **GitLens** and **Git Graph** extensions rather than the built-in Git support, but they still map to the same CLI commands.
- `git.enableSmartCommit`, `git.confirmSync`, and `git.autofetch` change how the Commit and Sync buttons behave — worth reviewing if the UI's behavior surprises you.

## Related

- [SSH ControlMaster](articles/ssh-controlmaster.md) — speeding up Git-over-SSH with connection reuse
- [SSH Managing Multiple Keys](articles/ssh-managing-multiple-keys.md) — per-service keys for GitHub and other Git remotes
