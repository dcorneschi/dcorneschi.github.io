# Homebrew Cheatsheet

Homebrew is the missing package manager for macOS (and Linux). It installs command-line tools, applications, and fonts from the terminal.

## Installation

```bash
# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Verify installation
brew --version

# Uninstall Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/uninstall.sh)"
```

## Package Management

### Install and Remove

```bash
# Install a formula (CLI tool)
brew install wget
brew install git node python3

# Install a cask (GUI application)
brew install --cask firefox
brew install --cask visual-studio-code
brew install --cask iterm2

# Reinstall
brew reinstall wget

# Uninstall
brew uninstall wget
brew uninstall --cask firefox

# Remove package and all dependencies not used by other packages
brew uninstall --zap --cask firefox
```

### Search

```bash
# Search for packages
brew search git
brew search --cask chrome

# Search with regex
brew search /^git/

# Show package info
brew info wget
brew info --cask firefox

# Open package homepage in browser
brew home wget

# Show package dependencies
brew deps wget
brew deps --tree wget
```

### List Installed

```bash
# List all installed formulae
brew list

# List installed casks
brew list --cask

# List installed formulae only
brew list --formula

# List with versions
brew list --versions

# List top-level packages (not installed as dependencies)
brew leaves

# List packages that depend on a formula
brew uses --installed wget
```

## Updating

```bash
# Update Homebrew itself and all formulae definitions
brew update

# Upgrade all installed packages
brew upgrade

# Upgrade specific package
brew upgrade wget

# Upgrade casks
brew upgrade --cask

# See what would be upgraded
brew outdated

# Preview what would be upgraded (dry run)
brew upgrade --dry-run

# See outdated casks
brew outdated --cask

# Pin a formula (prevent upgrades)
brew pin node@18

# Unpin
brew unpin node@18

# List pinned packages
brew list --pinned
```

## Cleanup

```bash
# Remove old versions of installed packages
brew cleanup

# See what would be cleaned
brew cleanup -n

# Remove all cache files
brew cleanup -s

# Remove specific package's old versions
brew cleanup wget

# Remove cache older than default (120 days)
brew cleanup --prune=30

# Remove dependencies no longer needed by any installed formula
brew autoremove
```

## Diagnostics

```bash
# Check system for potential problems
brew doctor

# Show Homebrew configuration
brew config

# Show where a formula is installed
brew --prefix wget

# Show Homebrew prefix
brew --prefix

# Show cellar path
brew --cellar

# Show cache path
brew --cache

# Show install history for a package
brew log wget

# Create symlinks for a formula
brew link wget

# Remove symlinks
brew unlink wget

# Install from a local formula file
brew install ./formula.rb
```

## Services

```bash
# List all services
brew services list

# Start a service
brew services start postgresql@15
brew services start redis

# Stop a service
brew services stop postgresql@15

# Restart a service
brew services restart nginx

# Run once (don't register with launchctl)
brew services run postgresql@15

# Stop all services
brew services stop --all

# Remove stale services (stopped/errored)
brew services cleanup
```

## Taps (Third-Party Repositories)

```bash
# Add a tap
brew tap hashicorp/tap
brew tap homebrew/cask-fonts

# List taps
brew tap

# Show tap information
brew tap-info hashicorp/tap

# Remove a tap
brew untap hashicorp/tap

# Install from a specific tap
brew install hashicorp/tap/terraform
```

## Bundle (Brewfile)

### Create Brewfile

```bash
# Dump all installed packages to a Brewfile
brew bundle dump

# Dump with descriptions
brew bundle dump --describe

# Dump to specific file
brew bundle dump --file=~/Brewfile
```

### Example Brewfile

```ruby
# Taps
tap "hashicorp/tap"
tap "homebrew/cask-fonts"

# CLI tools
brew "git"
brew "wget"
brew "jq"
brew "awscli"
brew "kubectl"
brew "terraform"
brew "node@18"

# Applications
cask "firefox"
cask "visual-studio-code"
cask "iterm2"
cask "docker"
cask "1password"

# Fonts
cask "font-jetbrains-mono"

# Mac App Store (requires mas)
mas "Slack", id: 803453959
```

### Install from Brewfile

```bash
# Install everything in Brewfile
brew bundle

# Install from specific file
brew bundle --file=~/Brewfile

# Check what's missing (without installing)
brew bundle check

# List everything in Brewfile
brew bundle list

# Remove packages not in Brewfile
brew bundle cleanup
brew bundle cleanup --force
```

## Versions

```bash
# Install specific version
brew install node@18
brew install python@3.11

# Switch between versions
brew unlink node@20 && brew link node@18

# List available versions
brew search node

# Show installed version
brew list --versions node
```

## Cask-Specific Commands

```bash
# Show app info
brew info --cask firefox

# Show where cask is installed
brew --caskroom

# Force reinstall (useful when app auto-updated)
brew reinstall --cask firefox

# List cask artifacts (what gets installed)
brew info --cask --json=v2 firefox | jq '.casks[0].artifacts'
```

## Formulae vs Casks

| Type | Installs | Command |
|------|----------|---------|
| Formula | CLI tools, libraries | `brew install git` |
| Cask | macOS GUI apps (.app) | `brew install --cask firefox` |

## Common Formulae

```bash
# Development
brew install git node python3 go rust

# DevOps
brew install awscli kubectl helm terraform ansible docker-compose

# Utilities
brew install wget curl jq yq bat exa fd ripgrep fzf tmux htop

# Networking
brew install nmap mtr iperf3 wrk

# Databases
brew install postgresql@15 redis mysql sqlite
```

## Common Casks

```bash
# Browsers
brew install --cask firefox google-chrome

# Development
brew install --cask visual-studio-code iterm2 docker postman

# Productivity
brew install --cask 1password rectangle raycast obsidian

# Communication
brew install --cask slack zoom discord

# Utilities
brew install --cask the-unarchiver appcleaner
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `brew doctor` reports issues | Follow the suggested fixes in output |
| Permission errors | `sudo chown -R $(whoami) $(brew --prefix)/*` |
| Formula conflicts | `brew unlink old && brew link new` |
| Cask already installed | `brew reinstall --cask app-name` |
| Outdated Xcode CLT | `xcode-select --install` |
| Slow `brew update` | Check network; `brew update --auto-update` disables auto-update |
| Package not found | `brew update` first, then try again |

```bash
# Reset Homebrew if things go wrong
brew update-reset

# Force remove a formula and ignore dependencies
brew uninstall --ignore-dependencies wget

# Clear download cache
rm -rf $(brew --cache)
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `HOMEBREW_NO_AUTO_UPDATE` | Set to 1 to disable auto-update before install |
| `HOMEBREW_CLEANUP_MAX_AGE_DAYS` | Days before cache is cleaned (default: 120) |
| `HOMEBREW_NO_ANALYTICS` | Set to 1 to disable analytics |
| `HOMEBREW_CASK_OPTS` | Default options for cask commands |
| `HOMEBREW_PREFIX` | Homebrew install prefix |

```bash
# Disable auto-update (add to ~/.zshrc)
export HOMEBREW_NO_AUTO_UPDATE=1

# Disable analytics
brew analytics off
```

## Tips

- Run `brew doctor` periodically to catch issues early
- Use `brew bundle` with a Brewfile to replicate your setup on a new machine
- Pin critical tools (`brew pin node@18`) to prevent accidental upgrades
- Use `brew leaves` to see what you explicitly installed vs auto-dependencies
- `brew cleanup` regularly to free disk space
- Set `HOMEBREW_NO_AUTO_UPDATE=1` if `brew install` feels slow (it updates first by default)
