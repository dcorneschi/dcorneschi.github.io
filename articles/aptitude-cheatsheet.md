# Aptitude Cheatsheet

Aptitude is a high-level package manager for Debian/Ubuntu systems. It provides a text-based UI and a command-line interface with advanced dependency resolution.

## Installation

```bash
# Aptitude is not installed by default on modern Ubuntu
sudo apt install -y aptitude

# Show help
aptitude help
```

## Basic Package Operations

### Install Packages

```bash
# Install a package
sudo aptitude install nginx

# Install multiple packages
sudo aptitude install nginx php mysql-server

# Install a specific version
sudo aptitude install nginx=1.18.0-0ubuntu1

# Install without prompting
sudo aptitude install -y nginx

# Install all packages matching a pattern (e.g., containing "minic")
sudo aptitude install "~nminic"

# Simulate install (dry run)
sudo aptitude install -s nginx
```

### Download Without Installing

```bash
# Download package .deb file without installing
sudo aptitude download nginx
```

### Remove Packages

```bash
# Remove package (keep config files)
sudo aptitude remove nginx

# Remove package and config files (purge)
sudo aptitude purge nginx

# Remove package and its unused dependencies
sudo aptitude remove --purge-unused nginx
```

### Update and Upgrade

```bash
# Update package lists
sudo aptitude update

# Upgrade installed packages (safe, no removals)
sudo aptitude safe-upgrade

# Full upgrade (may remove packages to resolve conflicts)
sudo aptitude full-upgrade

# Dist-upgrade (upgrades all, installs/removes as necessary)
sudo aptitude dist-upgrade

# Simulate upgrade
sudo aptitude safe-upgrade -s
```

## Search and Information

### Search Packages

```bash
# Search by name
aptitude search nginx

# Search by description
aptitude search ~dnginx

# Search installed packages
aptitude search ~i

# Search installed packages matching pattern
aptitude search '~i nginx'

# Search packages not installed
aptitude search '~n nginx !~i'

# Search by maintainer
aptitude search '~m canonical'

# Search broken packages
aptitude search ~b

# Search packages on hold
aptitude search ~ahold

# Search automatically installed packages
aptitude search '~M'

# Search manually installed packages
aptitude search '~i !~M'

# Search obsolete packages (no longer in repos)
aptitude search ~o
```

### Package Information

```bash
# Show package details
aptitude show nginx

# Show full package details (verbose)
aptitude show -v nginx

# Show package dependencies
aptitude depends nginx

# Show reverse dependencies (what depends on this package)
aptitude rdepends nginx

# Show why a package is installed
aptitude why nginx

# Show why a package is NOT installed
aptitude why-not nginx
```

### List Packages

```bash
# List all installed packages
aptitude search '~i' --display-format '%p %v'

# List upgradable packages
aptitude search '~U'

# List packages by size (largest first)
aptitude search '~i' --sort '~installsize'

# List automatically installed packages
aptitude search '~i ~M'

# List manually installed packages
aptitude search '~i !~M'
```

## Dependency Management

### Hold and Unhold Packages

```bash
# Hold a package (prevent upgrades)
sudo aptitude hold nginx

# Unhold a package
sudo aptitude unhold nginx

# List held packages
aptitude search '~ahold'
```

### Mark Packages

```bash
# Mark as automatically installed
sudo aptitude markauto nginx

# Mark as manually installed
sudo aptitude unmarkauto nginx

# Remove unused automatically installed packages
sudo aptitude autoclean
```

### Resolve Dependencies

```bash
# Fix broken dependencies
sudo aptitude install -f

# Show dependency tree
aptitude depends --show-all nginx

# Show why a package would be removed
aptitude why-not nginx
```

## Clean Up

```bash
# Remove downloaded .deb files from cache
sudo aptitude clean

# Remove old downloaded .deb files (keep current)
sudo aptitude autoclean

# Remove unused packages (auto-installed, no longer needed)
sudo aptitude purge '~c'

# Remove packages with residual config files
sudo aptitude purge '~c'

# Remove all packages that were auto-installed and are now orphaned
sudo aptitude purge '~o'
```

## Interactive Mode (TUI)

```bash
# Launch interactive text UI
sudo aptitude

# Key bindings in TUI:
# q         - Quit / go back
# ?         - Help
# u         - Update package lists
# U         - Mark all upgradable
# g         - Preview / apply changes
# /         - Search
# n         - Find next match
# +         - Mark for install/upgrade
# -         - Mark for removal
# _         - Mark for purge
# =         - Hold at current version
# :         - Mark as auto-installed
# L         - Change display limit (filter)
# C         - View changelog
# d         - View dependencies
# i         - Cycle through info views
# Enter     - Expand/collapse group
```

## Search Patterns Reference

Aptitude uses a powerful pattern language for searches:

| Pattern | Meaning |
|---------|---------|
| `~n<name>` | Package name matches |
| `~d<text>` | Description contains text |
| `~i` | Installed |
| `!~i` | Not installed |
| `~M` | Automatically installed |
| `!~M` | Manually installed |
| `~U` | Upgradable |
| `~o` | Obsolete (not in any repo) |
| `~b` | Broken |
| `~c` | Removed but has config files |
| `~ahold` | On hold |
| `~v<ver>` | Version matches |
| `~A<archive>` | From specific archive (stable, testing) |
| `~O<origin>` | From specific origin (Ubuntu, Debian) |
| `~s<section>` | In section (admin, libs, net) |
| `~p<priority>` | Has priority (required, important, standard, optional, extra) |
| `~D<dep>` | Depends on package |
| `~R<dep>` | Reverse-depends (depended on by) |
| `~m<maint>` | Maintainer matches |
| `~G<tag>` | Has debtag |
| `~T` | Tagged as a task |
| `~E` | Essential package |
| `?and(p1, p2)` | Both patterns match |
| `?or(p1, p2)` | Either pattern matches |
| `?not(p)` | Pattern does not match |

### Pattern Examples

```bash
# Find installed packages from a specific section
aptitude search '~i ~snet'

# Find large installed packages (> 10MB)
aptitude search '~i ~S installsize > 10000'

# Find packages that depend on python3
aptitude search '~D python3'

# Find installed packages with no reverse dependencies
aptitude search '~i !~R ~i'

# Find packages from a specific PPA/origin
aptitude search '~O "LP-PPA-"'
```

## Output Formatting

```bash
# Custom display format
aptitude search '~i' --display-format '%p %v %I'

# Format placeholders:
# %p - package name
# %v - current version
# %V - candidate version
# %I - installed size
# %d - description (short)
# %D - description (long)
# %s - section
# %M - auto-installed flag (A = auto)
# %t - archive (stable, testing, etc.)

# Examples
aptitude search '~i' --display-format '%p - %v (%I)'
aptitude search '~U' --display-format '%p: %v -> %V'
```

## Aptitude vs apt/apt-get

| Feature | aptitude | apt/apt-get |
|---------|----------|-------------|
| Interactive TUI | Yes | No |
| Dependency resolver | Advanced (multiple solutions) | Basic |
| Pattern search | Yes (powerful) | Limited |
| Why installed | `aptitude why` | No equivalent |
| Action logging | Built-in | Via apt history log |
| Hold packages | `aptitude hold` | `apt-mark hold` |
| Auto/manual marking | Built-in | `apt-mark` |
| Recommends handling | Configurable | Global only |
| Conflict resolution | Offers alternatives | Abort or force |
| Speed | Slower (more analysis) | Faster |

## Configuration

```bash
# Aptitude config file
cat /etc/apt/apt.conf.d/

# User config
~/.aptitude/config

# Common options
echo 'Aptitude::Recommends-Important "false";' | sudo tee /etc/apt/apt.conf.d/99aptitude-no-recommends

# Disable auto-install of recommends
sudo aptitude install --without-recommends nginx

# Always show full resolver output
# Add to ~/.aptitude/config:
# Aptitude::ProblemResolver::Show-Steps "true";
```

## Common Workflows

### Find and Remove Orphaned Packages

```bash
# Find packages no longer in any repository
aptitude search '~o'

# Remove them
sudo aptitude purge '~o'
```

### Find Packages with Residual Configs

```bash
# Find removed packages with leftover config files
aptitude search '~c'

# Purge them all
sudo aptitude purge '~c'
```

### List Manually Installed Packages (Rebuild System)

```bash
# Export list of manually installed packages
aptitude search '~i !~M' -F '%p' | sort > manual-packages.txt

# Re-install on another system
cat manual-packages.txt | xargs sudo aptitude install -y
```

### Downgrade a Package

```bash
# List available versions
aptitude versions nginx

# Install specific older version
sudo aptitude install nginx=1.18.0-0ubuntu1

# Hold to prevent re-upgrade
sudo aptitude hold nginx
```

## Package Integrity and Status

### dselect (Package Status)

```bash
# View package status database (/var/lib/dpkg/status)
dselect

# dselect provides an ncurses interface for:
# - Viewing all package states
# - Selecting packages for install/removal
# - Resolving dependencies interactively
```

### debsums (Verify Installed Packages)

```bash
# Install debsums
sudo aptitude install debsums

# Verify MD5 sums of all installed packages
sudo debsums

# Check a specific package
sudo debsums nginx

# Show only files that failed verification
sudo debsums -c

# Show only changed config files
sudo debsums -ce

# Generate missing md5sums files
sudo debsums --generate=missing
```

## Quick Reference

| Action | Command |
|--------|---------|
| Install | `sudo aptitude install <pkg>` |
| Remove | `sudo aptitude remove <pkg>` |
| Purge | `sudo aptitude purge <pkg>` |
| Update lists | `sudo aptitude update` |
| Safe upgrade | `sudo aptitude safe-upgrade` |
| Full upgrade | `sudo aptitude full-upgrade` |
| Search | `aptitude search <pattern>` |
| Show info | `aptitude show <pkg>` |
| Hold | `sudo aptitude hold <pkg>` |
| Unhold | `sudo aptitude unhold <pkg>` |
| Why installed | `aptitude why <pkg>` |
| Dry run | `sudo aptitude install -s <pkg>` |
| Clean cache | `sudo aptitude clean` |
| TUI mode | `sudo aptitude` |
