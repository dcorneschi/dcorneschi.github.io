# GitHub CLI (gh) Cheatsheet

The `gh` command-line tool brings GitHub to your terminal — authentication, repositories, pull requests, issues, releases, Actions, Gists, and API access. This reference covers the commands you reach for most.

## Installation and Authentication

### Install

```bash
# macOS (Homebrew)
brew install gh

# Debian/Ubuntu
sudo apt install gh

# Fedora/RHEL
sudo dnf install gh

# Arch
sudo pacman -S github-cli

# Windows (winget)
winget install --id GitHub.cli

# Check version
gh --version
```

### Authenticate

```bash
# Interactive login (prompts for browser or token)
gh auth login

# Login with a token from stdin
echo "$GH_TOKEN" | gh auth login --with-token

# Login to a GitHub Enterprise host
gh auth login --hostname github.example.com

# Show current auth status
gh auth status

# Refresh credentials / add scopes
gh auth refresh -s repo,read:org

# Print the active token (useful for scripts)
gh auth token

# Log out
gh auth logout
```

### Configuration

```bash
# Set default git protocol to SSH
gh config set git_protocol ssh

# Set your preferred editor
gh config set editor "code --wait"

# List all config
gh config list

# Set up gh as a git credential helper
gh auth setup-git
```

## Repositories

```bash
# Create a new repo (interactive)
gh repo create

# Create a private repo from the current directory and push
gh repo create my-project --private --source=. --push

# Create a public repo with a description
gh repo create my-org/my-project --public --description "My project"

# Clone a repo
gh repo clone owner/repo

# View a repo (opens details in terminal)
gh repo view owner/repo

# Open a repo in the browser
gh repo view owner/repo --web

# Fork a repo and clone it
gh repo fork owner/repo --clone

# List your repos
gh repo list

# List another user's or org's repos
gh repo list owner --limit 50

# Rename the current repo
gh repo rename new-name

# Set repo default (used by pr/issue commands in multi-remote setups)
gh repo set-default owner/repo

# Delete a repo (requires the delete_repo scope)
gh repo delete owner/repo
```

## Pull Requests

```bash
# Create a PR from the current branch (interactive)
gh pr create

# Create a PR with title and body, targeting main
gh pr create --base main --title "Add feature" --body "Description here"

# Create a draft PR and fill from commits
gh pr create --draft --fill

# List open PRs
gh pr list

# List PRs by author, label, or state
gh pr list --author "@me" --label bug --state open

# View a PR (number, URL, or branch)
gh pr view 42
gh pr view --web

# Check out a PR branch locally
gh pr checkout 42

# See the diff
gh pr diff 42

# Review a PR
gh pr review 42 --approve
gh pr review 42 --request-changes --body "Please fix X"
gh pr review 42 --comment --body "Looks good overall"

# See CI/check status for a PR
gh pr checks 42

# Merge a PR
gh pr merge 42 --squash --delete-branch
gh pr merge 42 --merge
gh pr merge 42 --rebase --auto

# Mark a draft PR ready for review
gh pr ready 42

# Close or reopen a PR
gh pr close 42
gh pr reopen 42

# Add reviewers, assignees, or labels
gh pr edit 42 --add-reviewer octocat --add-label "needs-review"
```

## Issues

```bash
# Create an issue (interactive)
gh issue create

# Create with title, body, labels, and assignee
gh issue create --title "Bug: crash on start" --body "Steps..." --label bug --assignee "@me"

# List issues
gh issue list

# Filter issues
gh issue list --state open --label bug --assignee "@me"

# View an issue
gh issue view 17
gh issue view 17 --web

# Comment on an issue
gh issue comment 17 --body "Working on this now"

# Close or reopen
gh issue close 17
gh issue reopen 17

# Edit labels/assignees
gh issue edit 17 --add-label "priority" --add-assignee octocat

# Transfer an issue to another repo
gh issue transfer 17 owner/other-repo
```

## GitHub Actions

```bash
# List recent workflow runs
gh run list

# List runs for a specific workflow
gh run list --workflow ci.yml

# View a run (interactive selection or by ID)
gh run view
gh run view 1234567890

# View logs for a run
gh run view 1234567890 --log

# View only failed step logs
gh run view 1234567890 --log-failed

# Watch a run until it completes
gh run watch

# Re-run a workflow (all jobs or just failed)
gh run rerun 1234567890
gh run rerun 1234567890 --failed

# Cancel a run
gh run cancel 1234567890

# List workflows
gh workflow list

# Manually trigger a workflow_dispatch workflow
gh workflow run deploy.yml --ref main -f environment=prod

# Enable or disable a workflow
gh workflow enable deploy.yml
gh workflow disable deploy.yml
```

## Releases

```bash
# List releases
gh release list

# Create a release with notes
gh release create v1.2.0 --title "v1.2.0" --notes "Changelog here"

# Auto-generate release notes from commits
gh release create v1.2.0 --generate-notes

# Create a release and upload build artifacts
gh release create v1.2.0 ./dist/app-linux ./dist/app-macos

# Upload assets to an existing release
gh release upload v1.2.0 ./dist/extra-binary

# View a release
gh release view v1.2.0

# Download release assets
gh release download v1.2.0 --pattern "*.tar.gz"

# Delete a release
gh release delete v1.2.0 --cleanup-tag
```

## Gists

```bash
# Create a gist from a file
gh gist create notes.md

# Create a public gist
gh gist create --public script.sh

# Create from stdin
echo "hello" | gh gist create --filename hello.txt

# List your gists
gh gist list

# View a gist
gh gist view <gist-id>

# Edit or clone a gist
gh gist edit <gist-id>
gh gist clone <gist-id>
```

## API Access

```bash
# GET a resource
gh api /repos/owner/repo

# GET the authenticated user
gh api /user

# Use a jq-style filter on the response
gh api /repos/owner/repo --jq '.stargazers_count'

# Paginate through all results
gh api /repos/owner/repo/issues --paginate

# POST with fields
gh api /repos/owner/repo/issues -f title="Bug" -f body="Details"

# Use a specific HTTP method
gh api --method DELETE /repos/owner/repo/issues/comments/123

# Call a GraphQL query
gh api graphql -f query='{ viewer { login } }'
```

## Aliases and Extensions

```bash
# Create a shortcut alias
gh alias set prs 'pr list --author "@me"'

# Use it
gh prs

# List aliases
gh alias list

# Browse and install extensions
gh extension browse
gh extension install owner/gh-extension-name

# List and upgrade extensions
gh extension list
gh extension upgrade --all
```

## Handy One-Liners

```bash
# Open the current repo in the browser
gh browse

# Open a specific file/line on the web
gh browse src/main.go:42

# Check out your most recent PR
gh pr checkout $(gh pr list --author "@me" --limit 1 --json number --jq '.[0].number')

# Approve and merge in one flow
gh pr review --approve && gh pr merge --squash --delete-branch

# Get the default branch name
gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'

# List open PRs as clean text (number + title)
gh pr list --json number,title --jq '.[] | "\(.number)\t\(.title)"'

# Watch the latest run and rerun failed jobs if it fails
gh run watch || gh run rerun --failed
```

## Tips

- Most `pr` and `issue` commands accept a number, a URL, or (for PRs) a branch name.
- `@me` is a special value that resolves to the authenticated user in `--author` and `--assignee` filters.
- Add `--json <fields>` plus `--jq <filter>` to any list/view command for scriptable, structured output — run the command with `--json` and no value to see the available fields.
- Set `GH_TOKEN` (or `GITHUB_TOKEN`) in CI environments to authenticate non-interactively without `gh auth login`.
- Use `gh repo set-default` once per clone when a repo has multiple remotes so `gh` knows which one to target.
