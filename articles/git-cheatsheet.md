# Git Cheatsheet

A broad reference of everyday Git commands and workflows — setup, branching, remotes, history, undoing changes, stashing, merging/rebasing, tags, and a set of end-to-end workflow examples. For deep dives on individual topics, the Git section of this site has dedicated articles (linked where relevant).

## Setup and Configuration

```bash
# Initialize a repository
git init
git init my-project

# Identity
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global core.editor "code --wait"
git config --global init.defaultBranch main

# Per-repo overrides
git config user.name "Work Name"
git config user.email "work@company.com"

# Inspect config
git config --list
git config --global --list
git config user.name
```

## Cloning

```bash
git clone https://github.com/user/repo.git
git clone git@github.com:user/repo.git          # SSH
git clone https://github.com/user/repo.git my-folder   # custom directory
git clone -b develop https://github.com/user/repo.git  # a specific branch
git clone --depth 1 https://github.com/user/repo.git   # shallow (latest commit)
```

See [Git Clone Methods and Options](articles/git-clone-methods.md) for transports, submodules, mirrors, and auth.

## Basic Workflow

```bash
# Status
git status
git status -s                # short format

# Stage
git add file.txt
git add .                    # everything under the current dir
git add *.js                 # matching files
git add -A                   # all changes, incl. deletions, repo-wide
git add -u                   # only tracked (modified/deleted) files

# Commit
git commit -m "Add new feature"
git commit -am "Add and commit tracked files"
git commit --amend -m "New message"

# Push
git push
git push origin main
git push -u origin feature-branch    # set upstream on first push

# Stop tracking a file but keep it locally (e.g. a committed .env)
git rm --cached .env                 # then add it to .gitignore
git rm -r --cached path/to/dir       # same, for a directory

# Inspect the file list
git ls-files                         # all tracked files
git ls-files --deleted               # tracked files deleted from the working tree
git status --ignored                 # include ignored files in status
```

Related: [git add . vs git add -A](articles/git-add-dot-vs-all.md) · [Editing a Commit Message](articles/git-edit-commit-message.md) · [git push vs git push origin HEAD](articles/git-push-vs-push-origin-head.md)

## Branching

```bash
# List
git branch                   # local
git branch -r                # remote
git branch -a                # all
git branch -vv               # with tracking info
git branch --show-current

# Create / switch
git branch feature-login
git checkout -b feature-login    # create + switch
git switch -c feature-login      # modern equivalent
git switch main                  # switch (modern)
git checkout main
git switch -                     # switch back to the previous branch (git checkout -)

# Branch from a specific base (inherits that base's full history)
git checkout -b new-branch existing-branch   # from another branch
git checkout -b new-branch abc1234            # from a specific commit
git switch -c new-branch existing-branch      # modern equivalent

# Switch to the previous branch
git switch -                     # git checkout - / git checkout @{-1}

# Check out a remote branch (create a local tracking branch)
git switch feature/api                          # auto-creates a tracking branch (2.23+)
git checkout --track origin/feature/api         # explicit; -t is the short form
git switch -c local-name origin/feature/api     # custom local name

# Force / quiet switch
git switch -f main               # discard local changes when switching (git checkout -f)
git switch -q main               # suppress output

# Inspect an old commit or tag (detached HEAD)
git checkout abc1234             # a commit
git checkout HEAD~2              # relative to HEAD
git checkout v1.2.0              # a tag
git switch -                     # return to your branch

# Rename
git branch -m old-name new-name
git branch -m new-name           # rename current branch

# Delete
git branch -d feature-login                    # safe (merged only)
git branch -D feature-login                    # force
git push origin --delete feature-login         # remote branch

# Merged / unmerged
git branch --merged
git branch --no-merged

# Bulk: delete all local branches except the current one
git branch | grep -v '^\*' | xargs git branch -D

# Bulk: delete all remote branches under backup/
git branch -r | grep 'origin/backup/' | sed 's|origin/||' | xargs git push origin --delete
```

See [Deleting Git Branches](articles/git-delete-branches.md) and [Listing Git Branches by Author](articles/git-list-branches-by-author.md).

## Remotes

```bash
# View / manage
git remote -v
git remote show origin
git remote add origin https://github.com/user/repo.git
git remote add upstream https://github.com/original/repo.git
git remote set-url origin https://github.com/user/new-repo.git
git remote remove origin

# Fetch / pull
git fetch
git fetch --all
git pull origin main
git pull --rebase origin main

# Sync a fork from upstream
git fetch upstream
git switch main
git merge upstream/main

# Push
git push origin main
git push -u origin feature-branch
git push --all
git push --tags
git push --force-with-lease          # safer than --force

# Prune stale remote-tracking refs
git remote prune origin
git fetch --prune
```

Related: [Sharing a Branch: Push, Then Fetch vs Pull](articles/git-share-branch-push-fetch-pull.md)

## History and Inspection

```bash
# Log
git log
git log --oneline
git log --oneline --graph --decorate --all
git log -n 5
git log --author="John Doe"
git log --since="2 weeks ago"
git log --until="2023-01-01"
git log --grep="bug fix"     # search messages
git log -S "function_name"   # pickaxe: search code additions/removals
git log main..HEAD           # commits unique to your branch (not in main)
git log --pretty=format:"%h - %an, %ar : %s"   # custom format

# File history
git log -- file.txt
git log -p -- file.txt       # with patches
git log -L 15,20:file.txt    # history of a line range
git log --name-only          # commits with the changed file names
git log --name-status        # file names + status (A/M/D)

# Show / blame
git show HEAD
git show abc1234
git show HEAD:file.txt        # file at a commit
git blame file.txt
git blame -L 10,20 file.txt
```

Related: [Viewing a File's Change History and Blame](articles/git-file-history-blame.md)

## Unpushed Commits

```bash
git fetch                                # refresh remote-tracking refs first
git log origin/main..HEAD --oneline      # commits not yet pushed
git log @{u}..HEAD --oneline             # same, via the upstream shorthand
git cherry -v                            # unpushed commits with messages (patch-equivalent)
git cherry -v origin/main                # compared to a specific ref

# Only my commits ahead of the remote
git log --author="$(git config user.name)" origin/main..HEAD --oneline
git log --no-merges --author="$(git config user.name)" origin/main..HEAD --oneline

# Upstream / tracking info
git status                               # ahead/behind summary
git branch -vv                           # branches with their upstreams
git rev-parse --abbrev-ref @{u}          # name of the current branch's upstream
```

Related: [Viewing Unpushed Commits in Git](articles/git-view-unpushed-commits.md)

## Viewing Changes

```bash
git diff                     # unstaged
git diff --staged            # staged (aka --cached)
git diff HEAD                # all changes (staged + unstaged) vs last commit
git diff HEAD~1              # vs previous commit
git diff main..feature       # between branches
git diff HEAD..origin/main   # local vs remote branch
git diff --name-only         # names only
git diff main --name-only    # changed file names vs a branch
git diff --stat              # summary with per-file +/- counts

# A single file across commits (-- separates paths from refs)
git diff HEAD~1 -- file.txt          # file vs the previous commit
git diff HEAD~3 -- file.txt          # file vs 3 commits ago
git diff abc1234 def5678 -- file.txt # file between two commits
git show abc1234:path/to/file        # a file's full contents at a commit
```

Related: [Comparing Your Branch Against Master with git diff](articles/git-diff-branch-against-master.md)

## Undoing Changes

```bash
# Discard working-tree changes
git restore file.txt
git restore .
git checkout -- file.txt     # older syntax

# Restore a file from another commit or branch
git checkout HEAD~3 -- deleted-file.txt   # recover a file removed earlier
git checkout main -- config.json          # grab a file from another branch
git restore --source main config.json     # modern equivalent

# Unstage
git restore --staged file.txt
git reset HEAD file.txt       # older syntax

# Reset one file to a specific commit's version
git reset abc1234 -- config.json

# Undo commits
git reset --soft HEAD~1       # keep changes staged
git reset --mixed HEAD~1      # keep changes, unstaged (default)
git reset --hard HEAD~1       # discard changes
git reset --hard origin/main  # match remote exactly

# Undo with a new commit (safe on shared branches)
git revert HEAD
git revert abc1234
git revert HEAD~3..HEAD       # a range
git revert abc1234 def5678      # several specific commits
git revert --no-commit abc1234  # stage the revert without committing
git revert --no-edit abc1234    # use the default message, skip the editor
git revert -m 1 <merge-hash>    # revert a merge commit (keep parent 1)
git revert --continue           # after resolving revert conflicts
git revert --abort              # cancel a revert in progress

# Recover from a bad reset — reflog remembers where HEAD was
git reflog
git reset --hard <lost-commit-hash>

# Delete a commit from the REMOTE (rewrites history — coordinate with your team)
git reset --hard HEAD~1                       # drop the last local commit (HEAD~2 for two, etc.)
git push origin <branch> --force-with-lease   # then overwrite the remote
# Middle commit? drop its line in an interactive rebase, then force-push:
git rebase -i <commit-before-the-one-to-drop>
git push origin <branch> --force-with-lease
# Note: `git reset --hard origin/main` only resets LOCAL — it deletes nothing on the remote.
```

Deep dives: [Undoing Changes in Git: reset vs checkout vs revert](articles/git-undo-reset-checkout-revert.md) · [Removing a Local Commit](articles/git-remove-local-commit.md) · [Undoing a Pushed Commit](articles/git-undo-pushed-commit.md) · [Restoring Files with git restore](articles/git-restore-files.md)

## Stashing

```bash
git stash
git stash push -m "Work in progress"   # (git stash save is deprecated)
git stash -u                            # include untracked
git stash push -- file.txt              # specific files
git stash list
git stash show -p                       # show patch
git stash apply                         # apply, keep on stack
git stash apply 'stash@{1}'
git stash pop                           # apply and remove
git stash drop 'stash@{0}'
git stash clear
git stash branch new-feature 'stash@{1}'
```

Full guide: [Git Stash Guide](articles/git-stash-guide.md)

## Merging and Rebasing

```bash
# Merge
git switch main
git merge feature-branch
git merge --no-ff feature-branch     # force a merge commit
git merge --squash feature-branch    # collapse into staged changes
git merge --abort

# Merge favoring one side automatically
git merge -X ours feature-branch     # prefer our side on conflicts
git merge -X theirs feature-branch   # prefer their side on conflicts
git merge --no-commit --no-ff feature-branch   # stage the merge to review first

# Cherry-pick
git cherry-pick abc1234
git cherry-pick abc1234..def5678     # a range
git cherry-pick --continue           # after resolving conflicts
git cherry-pick --abort              # bail out

# Rebase
git rebase main
git rebase -i HEAD~3                  # interactive
git rebase --continue
git rebase --skip
git rebase --abort
```

Conflict resolution during a merge or rebase:

```bash
# List the conflicted files
git diff --name-only --diff-filter=U
git status

# Resolve by taking one whole side
git checkout --ours  path/to/file    # keep our version
git checkout --theirs path/to/file   # keep their version

# Inspect the three versions of a conflicted file
git show :1:path/to/file             # common ancestor
git show :2:path/to/file             # ours
git show :3:path/to/file             # theirs

# 1. Edit the conflicted files to resolve markers, then stage them
git add conflicted-file.txt
# 2. Continue
git merge --continue     # or: git rebase --continue / git cherry-pick --continue
# ...or bail out
git merge --abort        # or: git rebase --abort / git cherry-pick --abort

# Reuse recorded resolutions so repeated conflicts auto-resolve
git config --global rerere.enabled true
```

Related: [Aborting and Investigating Merge Conflicts](articles/git-abort-investigate-merge-conflicts.md) · [Getting the Latest Changes from Master into Your Feature Branch](articles/git-update-feature-branch-from-master.md)

## Tags

```bash
# Create
git tag v1.0.0                              # lightweight
git tag -a v1.0.0 -m "Release 1.0.0"        # annotated
git tag -a v1.0.0 abc1234 -m "Release 1.0.0"  # tag a specific commit

# List / show
git tag
git tag -l "v1.*"
git show v1.0.0

# Push / delete
git push origin v1.0.0
git push origin --tags
git tag -d v1.0.0
git push origin --delete v1.0.0
```

## Advanced

```bash
# Search tracked content
git grep "function_name"
git grep -n "TODO"
git grep -i "bug"

# Submodules
git submodule add https://github.com/user/repo.git lib/repo
git submodule update --init --recursive
git submodule update --remote
git submodule deinit lib/repo && git rm lib/repo

# Worktrees
git worktree add ../hotfix hotfix-branch
git worktree list
git worktree remove ../hotfix

# Bisect (binary search for a bad commit)
git bisect start
git bisect bad
git bisect good v1.0.0
# ...test each step, mark good/bad, then:
git bisect reset
```

## Cleaning Up and Maintenance

```bash
# Remove untracked files
git clean -n                 # dry run
git clean -f                 # files
git clean -fd                # files + directories

# Nuke local state to match remote exactly (destructive)
git reset --hard origin/main && git clean -fd

# Prune stale remote branches
git remote prune origin

# Garbage collection
git gc
git gc --aggressive

# Pull every Git repo in the current directory
for i in */.git; do (cd "$(dirname "$i")" && git pull); done
```

Related: [Removing Untracked Files with git clean](articles/git-clean-untracked-files.md) · [Resetting a Local Branch to Match the Remote](articles/git-reset-local-to-remote.md)

## Handy Aliases

```bash
git config --global alias.st "status -s"
git config --global alias.co checkout
git config --global alias.br "branch -vv"
git config --global alias.cm "commit -m"
git config --global alias.ca "commit -am"
git config --global alias.unstage "reset HEAD --"
git config --global alias.last "log -1 HEAD"
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.undo "reset --soft HEAD~1"
# Push the current branch and set upstream
git config --global alias.pc '!git push -u origin $(git branch --show-current)'
```

## Workflow Examples

### Feature branch

```bash
git switch main
git pull origin main
git switch -c feature/user-authentication
# ...work...
git add .
git commit -m "Add user login form"
git push -u origin feature/user-authentication
# open a PR/MR, get it reviewed and merged, then clean up:
git switch main
git pull origin main
git branch -d feature/user-authentication
git push origin --delete feature/user-authentication
```

### Hotfix

```bash
git switch main
git pull origin main
git switch -c hotfix/critical-bug
git commit -am "Fix critical vulnerability"
git push -u origin hotfix/critical-bug
# merge to main, tag, and back-merge to develop
git switch main && git merge hotfix/critical-bug && git push origin main
git switch develop && git merge hotfix/critical-bug && git push origin develop
git branch -d hotfix/critical-bug
git push origin --delete hotfix/critical-bug
```

### Release

```bash
git switch develop
git pull origin develop
git switch -c release/v1.2.0
git commit -am "Bump version to 1.2.0"
git push -u origin release/v1.2.0
# merge to main and tag
git switch main && git merge release/v1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"
git push origin main --tags
# back-merge to develop, then clean up
git switch develop && git merge release/v1.2.0 && git push origin develop
git branch -d release/v1.2.0 && git push origin --delete release/v1.2.0
```

## Emergency Recipes

```bash
# Committed to the wrong branch (not pushed)
git branch correct-branch      # bookmark the commit
git reset --hard HEAD~1        # remove it here
git switch correct-branch      # continue there

# Undo the last push — safe on shared branches
git revert HEAD && git push origin main
# ...dangerous alternative if nobody else pulled it
git reset --hard HEAD~1 && git push --force-with-lease origin main

# Recover a "lost" commit (detached HEAD, bad reset, etc.)
git reflog
git switch -c recovered <commit-hash>

# Restore an accidentally deleted file
git checkout HEAD -- file.txt
git log --oneline -- file.txt      # find the deleting commit
git checkout <commit>^ -- file.txt # grab it from just before deletion
```

Related: [Understanding HEAD in Git](articles/git-head-explained.md)

## Best Practices

- Make small, focused commits with clear, present-tense messages (`feat: ...`, `fix: ...`, `docs: ...`).
- Use consistent branch names (`feature/...`, `bugfix/...`, `hotfix/...`).
- Pull (or fetch) before you push; review with `git diff --staged` before committing.
- Keep `main`/`master` stable; develop in branches and rebase feature branches before merging.
- Maintain a `.gitignore` (`node_modules/`, `.env`, `*.log`, `dist/`).
- Be deliberate with destructive commands — `git reset --hard`, `git clean -fd`, and `git push --force`.

## Source

- [Baeldung — Git series](https://www.baeldung.com/ops/git-series)
