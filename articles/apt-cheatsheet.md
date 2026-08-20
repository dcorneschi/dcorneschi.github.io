# apt Cheatsheet

`apt` is the high-level package manager for Debian-based systems (Ubuntu, Debian, Linux Mint). It wraps `dpkg` and adds dependency resolution, repository management, and package downloading. The `apt` command (introduced in Ubuntu 14.04 / Debian 8) provides a cleaner CLI than the older `apt-get` and `apt-cache` pair.

## Update & Upgrade

```bash
# Update package index (fetch latest package lists)
sudo apt update

# Upgrade all installed packages (safe — won't remove anything)
sudo apt upgrade

# Upgrade with automatic conflict resolution (may remove packages)
sudo apt full-upgrade

# Update + upgrade in one shot
sudo apt update && sudo apt upgrade -y

# Upgrade a single package
sudo apt install --only-upgrade package-name

# Simulate upgrade (show what would change without doing it)
apt upgrade --simulate
apt upgrade -s

# Show what would be upgraded
apt list --upgradeable

# Distribution upgrade (major release, e.g., 22.04 → 24.04)
sudo do-release-upgrade
sudo do-release-upgrade -d    # Upgrade to development release
```

## Install Packages

```bash
# Install a package
sudo apt install package-name

# Install multiple packages
sudo apt install pkg1 pkg2 pkg3

# Install a specific version
sudo apt install package-name=1.2.3-1

# Install without prompting
sudo apt install -y package-name

# Install from a .deb file (with dependency resolution)
sudo apt install ./package.deb

# Install without recommended packages
sudo apt install --no-install-recommends package-name

# Install with suggested packages
sudo apt install --install-suggests package-name

# Reinstall a package (refresh all its files)
sudo apt install --reinstall package-name

# Install and mark as manually installed
sudo apt install package-name
apt-mark manual package-name

# Download without installing
apt download package-name

# Download source package
apt source package-name
apt source --download-only package-name
```

## Remove Packages

```bash
# Remove a package (keep config files)
sudo apt remove package-name

# Remove and delete config files (purge)
sudo apt purge package-name

# Remove unused dependencies (auto-installed, no longer needed)
sudo apt autoremove

# Purge + autoremove
sudo apt purge package-name && sudo apt autoremove --purge

# Remove all packages matching pattern
sudo apt remove 'libfoo*'

# Simulate removal (see what would happen)
apt remove --simulate package-name

# Simulate autoremove (--dry-run is an alias for --simulate)
apt autoremove --dry-run
```

## Search & Info

```bash
# Search for packages by name/description
apt search keyword
apt search "web server"

# Search only in package names
apt search --names-only nginx

# Show package details
apt show package-name

# Show all available versions
apt list --all-versions package-name

# List installed packages
apt list --installed

# List upgradeable packages
apt list --upgradeable

# Show package dependencies
apt depends package-name

# Show reverse dependencies (what depends on this)
apt rdepends package-name

# Show package policy (versions, priorities, repos)
apt policy package-name

# Show which repository provides a package
apt-cache policy package-name

# Show package changelog
apt changelog package-name
```

## apt-cache Commands

```bash
# Search packages
apt-cache search keyword
apt-cache search --names-only keyword

# Show package info
apt-cache show package-name

# Show detailed package graph info (dependencies, reverse deps, versions)
apt-cache showpkg package-name

# Show package dependencies
apt-cache depends package-name

# Show recursive dependencies
apt-cache depends --recurse --no-recommends package-name

# Show reverse dependencies
apt-cache rdepends package-name

# Show only installed reverse deps
apt-cache rdepends --installed package-name

# Show package statistics
apt-cache stats

# Show policy (sources and priorities)
apt-cache policy
apt-cache policy package-name

# Show unmet dependencies
apt-cache unmet

# List package names matching pattern
apt-cache pkgnames nginx

# Count all available packages in the cache
apt-cache pkgnames | wc -l

# Show policy with wildcard
apt-cache policy open-vm-tools*

# Show source package info
apt-cache showsrc package-name

# Dump entire package cache (debugging)
apt-cache dump | head -50
```

## Check if a Package is in a Repo

### rmadison (Query Archive Directly)

```bash
# Install rmadison (part of devscripts)
sudo apt install devscripts

# Query the Ubuntu archive for all published versions
rmadison snapd

# Output shows versions across all suites/pockets:
#  snapd | 2.45.1+20.04 | focal          | source, amd64, arm64, ...
#  snapd | 2.70+20.04   | focal-updates  | source, amd64, arm64, ...
#  snapd | 2.70+20.04   | focal-security | source, amd64, arm64, ...

# Query a specific suite
rmadison -s noble snapd

# Query for a specific architecture
rmadison -a amd64 snapd
```

`rmadison` is the most direct method — it queries the archive server-side regardless of local apt state.

### apt-get changelog

```bash
# View changelog (sometimes has entries when release notes don't)
apt-get changelog package-name
```

### Web Resources

```bash
# Launchpad Package Tracker (authoritative source)
# https://launchpad.net/ubuntu/+source/<package-name>

# Ubuntu Packages Search
# https://packages.ubuntu.com/<package-name>

# For a specific release:
# https://packages.ubuntu.com/<codename>/<package-name>
```

### Typical Verification Workflow

```bash
# 1. Confirm package is in the archive and which pocket
rmadison package-name

# 2. Confirm your system can see it
sudo apt update && apt-cache policy package-name

# 3. Check Launchpad for build/publish timestamps (web)
# https://launchpad.net/ubuntu/+source/<package-name>
```

## apt-get Commands

```bash
# Update package lists
sudo apt-get update

# Upgrade packages
sudo apt-get upgrade
sudo apt-get dist-upgrade    # Equivalent to apt full-upgrade

# Install
sudo apt-get install package-name
sudo apt-get install -y package-name
sudo apt-get install --no-install-recommends package-name
sudo apt-get install package-name=version

# Remove
sudo apt-get remove package-name
sudo apt-get purge package-name
sudo apt-get autoremove

# Fix broken dependencies
sudo apt-get install -f
sudo apt-get --fix-broken install

# Download package (doesn't install)
apt-get download package-name

# Download only, placing it in /var/cache/apt/archives (not current dir)
sudo apt-get -d install package-name

# Download source
apt-get source package-name

# Build dependencies for a source package
sudo apt-get build-dep package-name

# Clean package cache
sudo apt-get clean          # Remove all cached .debs
sudo apt-get autoclean      # Remove only obsolete cached .debs

# Simulate (dry-run)
apt-get install -s package-name
apt-get upgrade -s

# Check for broken packages
sudo apt-get check

# Re-read package selections from dpkg
sudo apt-get dselect-upgrade
```

## Repository Management

```bash
# List configured repositories
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/
apt-cache policy

# List all configured deb lines (one-liner)
grep -rhE '^deb' /etc/apt/sources.list* /etc/apt/sources.list.d/

# Add a PPA (Ubuntu)
sudo add-apt-repository ppa:user/ppa-name
sudo apt update

# Remove a PPA
sudo add-apt-repository --remove ppa:user/ppa-name

# Add or remove a repository component (e.g., universe, multiverse)
sudo add-apt-repository universe
sudo add-apt-repository -r universe

# Edit sources.list using $EDITOR
sudo apt edit-sources

# Add a repository manually (modern .sources format - Ubuntu 24.04+)
cat <<'EOF' | sudo tee /etc/apt/sources.list.d/example.sources
Types: deb
URIs: https://repo.example.com/apt
Suites: noble
Components: main
Signed-By: /etc/apt/keyrings/example.gpg
EOF

# Add a repository manually (legacy one-line format)
echo "deb [signed-by=/etc/apt/keyrings/example.gpg] https://repo.example.com/apt $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/example.list

# Add GPG key for a repository
curl -fsSL https://repo.example.com/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/example.gpg

# List configured keys
apt-key list                          # Deprecated
ls /etc/apt/keyrings/                 # Modern approach
ls /etc/apt/trusted.gpg.d/           # System trusted keys

# Show repository file for a package
apt-cache policy package-name | grep -A1 "500\|990"
```

### sources.list Format

```
# Legacy format
deb [options] uri suite component1 [component2] [...]
deb-src [options] uri suite component1 [component2] [...]

# Examples
deb http://archive.ubuntu.com/ubuntu noble main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu noble main restricted

# With signed-by (recommended)
deb [signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable
```

## Package Holds & Pins

```bash
# Hold a package (prevent upgrades)
sudo apt-mark hold package-name

# Unhold a package
sudo apt-mark unhold package-name

# Show held packages
apt-mark showhold

# Show held packages (alternative via dpkg)
dpkg -l | grep "^hi"

# Unhold all held packages at once
sudo apt-mark unhold $(apt-mark showhold)

# Mark as manually installed (won't be autoremoved)
sudo apt-mark manual package-name

# Mark as auto-installed (eligible for autoremove)
sudo apt-mark auto package-name

# Show manually installed packages
apt-mark showmanual

# Show auto-installed packages
apt-mark showauto
```

### APT Pinning (Version Control)

```bash
# Pin a package to a specific version
cat <<'EOF' | sudo tee /etc/apt/preferences.d/pin-nginx
Package: nginx
Pin: version 1.18.*
Pin-Priority: 1001
EOF

# Pin all packages from a specific repository
cat <<'EOF' | sudo tee /etc/apt/preferences.d/pin-repo
Package: *
Pin: origin "packages.example.com"
Pin-Priority: 100
EOF

# Pin to prevent a package from being installed
cat <<'EOF' | sudo tee /etc/apt/preferences.d/block-snap
Package: snapd
Pin: release *
Pin-Priority: -1
EOF

# Show effective pin/priority for a package
apt-cache policy package-name
```

| Priority | Effect |
|----------|--------|
| `1001+` | Force install (even downgrade) |
| `990` | Install if newer than installed |
| `500` | Default priority |
| `100` | Install only if nothing else available |
| `1` | Install only if explicitly requested |
| `-1` | Never install |

## Cache Management

```bash
# Show cache size
du -sh /var/cache/apt/archives/

# Remove all cached .deb files
sudo apt clean

# Remove only obsolete cached .deb files (old versions)
sudo apt autoclean

# List cached .deb files
ls /var/cache/apt/archives/*.deb | wc -l

# Show apt cache statistics
apt-cache stats
```

## Fix Broken Packages

```bash
# Fix broken dependencies
sudo apt --fix-broken install
sudo apt-get install -f

# Configure pending packages
sudo dpkg --configure -a

# Full repair sequence
sudo dpkg --configure -a
sudo apt --fix-broken install
sudo apt update
sudo apt upgrade

# Clear lock files (only if apt crashed)
sudo rm -f /var/lib/apt/lists/lock
sudo rm -f /var/lib/dpkg/lock*
sudo rm -f /var/cache/apt/archives/lock
sudo dpkg --configure -a

# Rebuild package cache
sudo apt update --fix-missing

# Reinstall all packages from a specific source (nuclear option)
sudo apt install --reinstall $(dpkg -l | grep '^ii' | awk '{print $2}')
```

## Proxy & Network

```bash
# Set proxy for apt (temporary)
sudo apt -o Acquire::http::Proxy="http://proxy:3128" update

# Set proxy permanently
cat <<'EOF' | sudo tee /etc/apt/apt.conf.d/90proxy
Acquire::http::Proxy "http://proxy:3128";
Acquire::https::Proxy "http://proxy:3128";
EOF

# Disable proxy for specific hosts
cat <<'EOF' | sudo tee /etc/apt/apt.conf.d/91noproxy
Acquire::http::Proxy::security.ubuntu.com "DIRECT";
EOF

# Force IPv4 only
sudo apt -o Acquire::ForceIPv4=true update

# Make IPv4 permanent
echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4
```

## Unattended / Non-Interactive

```bash
# Install without any prompts
sudo DEBIAN_FRONTEND=noninteractive apt install -y package-name

# Upgrade without prompts (keep old configs)
sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y \
  -o Dpkg::Options::="--force-confold" \
  -o Dpkg::Options::="--force-confdef"

# Preseed answers for package configuration
echo "postfix postfix/main_mailer_type select Internet Site" | sudo debconf-set-selections
sudo DEBIAN_FRONTEND=noninteractive apt install -y postfix

# Show debconf questions for a package
sudo debconf-show package-name

# List all stored debconf values
sudo debconf-get-selections

# Select which frontend debconf uses (dialog, readline, noninteractive, etc.)
sudo dpkg-reconfigure debconf

# List which packages use debconf
grep -E 'Package:|Depends:.*debconf' /var/lib/dpkg/status
```

## apt Configuration

```bash
# Show all apt configuration
apt-config dump

# Show specific setting
apt-config dump | grep -i proxy

# Configuration file locations (in priority order)
# /etc/apt/apt.conf              (main config, rarely used)
# /etc/apt/apt.conf.d/*.conf     (drop-in configs, preferred)

# Useful config options
cat <<'EOF' | sudo tee /etc/apt/apt.conf.d/99local
// Don't install recommended packages by default
APT::Install-Recommends "false";

// Don't install suggested packages
APT::Install-Suggests "false";

// Keep downloaded .debs after install
Binary::apt::APT::Keep-Downloaded-Packages "true";

// Auto-clean old cache entries
APT::Periodic::AutocleanInterval "7";
EOF
```

## Unattended Upgrades

```bash
# Install unattended-upgrades
sudo apt install unattended-upgrades

# Enable automatic security updates
sudo dpkg-reconfigure -plow unattended-upgrades

# Configuration file
cat /etc/apt/apt.conf.d/50unattended-upgrades

# Test run (dry-run)
sudo unattended-upgrade --dry-run --debug

# Force run now
sudo unattended-upgrade

# Check logs
cat /var/log/unattended-upgrades/unattended-upgrades.log
```

## One-Liners

```bash
# Update + upgrade + autoremove + clean in one shot
sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove --purge -y && sudo apt clean

# List top 20 largest installed packages
dpkg-query -Wf '${Installed-Size}\t${Package}\n' | sort -rn | head -20

# Find manually installed packages (not auto-dependencies)
apt-mark showmanual | sort

# List packages from a specific PPA or repository
apt list --installed 2>/dev/null | grep "ppa\|docker\|repo-name"

# Show packages that were recently upgraded
grep "upgrade" /var/log/apt/history.log | tail -5

# List last 10 packages installed (from dpkg log, including rotated logs)
zgrep 'install ' /var/log/dpkg.log* | sort | cut -f1,2,4 -d' ' | tail

# Full list of installed packages with installation date
zgrep 'install ' /var/log/dpkg.log* | sort | cut -f1,2,4 -d' '

# Find which package provides a command
apt-file search --regexp 'bin/command$'

# Install build essentials for compiling
sudo apt install build-essential

# List all available versions of a package
apt-cache madison package-name

# Show all packages in a section
apt-cache search --names-only "^linux-image"

# Downgrade a package to a specific version
sudo apt install package-name=1.2.3-1
sudo apt-mark hold package-name

# Find packages with security updates available
apt list --upgradeable 2>/dev/null | grep -i security

# Show all versions of upgradeable packages
apt list --upgradeable -a

# Filter upgradeable by security and updates repos
apt list --upgradeable 2>/dev/null | grep -E "(security|updates)"

# Check which security repositories are configured
apt-cache policy | grep -A1 security

# View security-related changelog entries for a package
apt-get changelog package-name | grep -i security

# Search for packages with security updates (aptitude)
aptitude search '~U~Asecurity'

# Show upgradeable packages with versions (requires apt-show-versions)
sudo apt install apt-show-versions
apt-show-versions -u

# Count installed packages
apt list --installed 2>/dev/null | wc -l

# Show only manually installed packages with versions
apt-mark showmanual | xargs dpkg-query -f '${Package}\t${Version}\n' -W 2>/dev/null

# Find broken packages
sudo apt-get check 2>&1 | grep -v "^$"

# Download all dependencies for a package (offline install prep)
apt-get download $(apt-cache depends --recurse --no-recommends --no-suggests \
  --no-conflicts --no-breaks --no-replaces --no-enhances package-name | \
  grep "^\w" | sort -u)

# Show package install/remove history
cat /var/log/apt/history.log | grep -E "^(Start-Date|Commandline|Install|Remove|Purge)"

# Find duplicate packages (multiple architectures)
dpkg -l | awk '/^ii/{print $2}' | sed 's/:.*$//' | sort | uniq -d

# Purge all deleted packages (configs remaining on disk)
sudo apt purge $(dpkg -l | grep '^rc' | awk '{print $2}')

# Purge deleted kernel packages specifically
sudo apt purge $(dpkg -l | grep '^rc' | awk '{print $2}' | grep 'linux-')

# Show package version (quiet, one-line)
apt -qq list package-name

# List packages not available in any repository (locally installed .debs)
apt list --installed 2>/dev/null | grep "local"

# Show size of each repository's package list
wc -l /var/lib/apt/lists/*Packages 2>/dev/null | sort -n

# Find what would be autoremoved
apt autoremove --simulate 2>/dev/null | grep "^Remv"

# Clone installed packages to another machine
apt-mark showmanual | ssh user@remote 'xargs sudo apt install -y'

# Find packages installed from backports
apt list --installed 2>/dev/null | grep backport
```

## Tips & Tricks

### Speed Up Downloads

```bash
# Use a faster mirror (generate sources.list)
# For Ubuntu: https://launchpad.net/ubuntu/+archivemirrors
# For Debian: https://www.debian.org/mirror/list

# Use apt-fast (parallel downloads)
sudo add-apt-repository ppa:apt-fast/stable
sudo apt update && sudo apt install -y apt-fast
apt-fast install package-name

# Increase download concurrency (native)
echo 'APT::Acquire::Queue-Mode "access";' | sudo tee /etc/apt/apt.conf.d/99parallel
```

### Avoid Kernel Upgrade Prompts

```bash
# Keep current kernel config without prompting
echo 'DPkg::options { "--force-confold"; }' | sudo tee /etc/apt/apt.conf.d/90keepconf
```

### Install Specific Architecture

```bash
# Add i386 architecture
sudo dpkg --add-architecture i386
sudo apt update

# Install 32-bit package
sudo apt install package-name:i386

# Remove architecture
sudo dpkg --remove-architecture i386
```

### Work with Source Packages

```bash
# Enable source repositories
sudo sed -i 's/^# deb-src/deb-src/' /etc/apt/sources.list
sudo apt update

# Download source and build dependencies
apt source package-name
sudo apt build-dep package-name

# Build the package
cd package-name-*/
dpkg-buildpackage -us -uc -b
```

### Auto-Install on Missing File Access

```bash
# auto-apt automatically installs packages when a missing file is accessed
# NOTE: Only available on Ubuntu 16.04 and older — removed from 18.04+
sudo apt install auto-apt
sudo auto-apt update

# Run a command — if it tries to access a file from an uninstalled package,
# auto-apt will install that package on the fly
auto-apt run ./configure
```

### Apt History & Logging

```bash
# Show apt command history
cat /var/log/apt/history.log

# Show compressed older history
zcat /var/log/apt/history.log.*.gz

# Show dpkg-level log
cat /var/log/dpkg.log

# Show what was installed on a specific date
grep "2024-06-15" /var/log/apt/history.log

# Show last 5 apt operations
grep "^Start-Date" /var/log/apt/history.log | tail -5
```

### Offline Package Installation

```bash
# On online machine: download package + all dependencies
mkdir offline-pkgs && cd offline-pkgs
apt-get download $(apt-cache depends --recurse --no-recommends --no-suggests \
  --no-conflicts --no-breaks --no-replaces --no-enhances package-name | \
  grep "^\w" | sort -u)

# Transfer to offline machine, then install
sudo dpkg -i *.deb
sudo apt --fix-broken install    # Fix any ordering issues
```

### Safe Upgrades in Production

```bash
# Check what would change
apt upgrade --simulate

# Upgrade only security patches
sudo unattended-upgrade --dry-run
sudo unattended-upgrade

# Upgrade with logging and confirmation
sudo apt upgrade 2>&1 | tee /tmp/upgrade-$(date +%F).log

# Rollback: show what was upgraded
grep "Upgrade:" /var/log/apt/history.log | tail -1

# Downgrade a problematic package
sudo apt install package-name=previous.version
```

### Troubleshooting Hash Sum Mismatch

```bash
# Clear apt lists and re-download
sudo rm -rf /var/lib/apt/lists/*
sudo apt clean
sudo apt update

# If persists, check for proxy/CDN issues
sudo apt -o Acquire::https::No-Cache=True update
sudo apt -o Acquire::http::No-Cache=True update
```

### Flush Repository Cache Completely

```bash
# 1. Temporarily move away all source lists
sudo mv /etc/apt/sources.list.d/*.list /tmp/

# 2. Run apt update (clears references to removed repos)
sudo apt update

# 3. Move source lists back
sudo mv /tmp/*.list /etc/apt/sources.list.d/

# 4. Update again
sudo apt update
```

### apt vs apt-get vs apt-cache

| Feature | `apt` | `apt-get` / `apt-cache` |
|---------|-------|-------------------------|
| User-friendly output | Yes (progress bars, colors) | No |
| Scripting-stable | No (output may change) | Yes |
| Combined search + info | `apt search`, `apt show` | `apt-cache search`, `apt-cache show` |
| Tab completion | Better | Basic |
| `upgrade` | `apt upgrade` | `apt-get upgrade --with-new-pkgs` |
| `full-upgrade` | `apt full-upgrade` | `apt-get dist-upgrade` |
| Recommended for | Interactive use | Scripts and automation |

> **For scripts and automation, use `apt-get` and `apt-cache`** — their output format is stable across releases. `apt` is designed for human interaction and its output format can change.

## Shell Functions & Aliases

```bash
# Update function (update + upgrade)
function update() {
    sudo apt update
    sudo apt -y upgrade
}

# One-line alias
alias update_os='sudo apt update && sudo apt -y upgrade'

# Full maintenance alias
alias apt-maintain='sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove --purge -y && sudo apt clean'
```

## Important Files

| File | Purpose |
|------|---------|
| `/etc/apt/sources.list` | Main repository list |
| `/etc/apt/sources.list.d/` | Additional repository files (`.list` or `.sources`) |
| `/etc/apt/apt.conf` | Main APT configuration (rarely used directly) |
| `/etc/apt/apt.conf.d/` | Drop-in configuration files |
| `/etc/apt/preferences` | Package pinning rules |
| `/etc/apt/preferences.d/` | Drop-in pinning files |
| `/etc/apt/keyrings/` | GPG keys for repositories (modern) |
| `/etc/apt/trusted.gpg.d/` | Trusted GPG keys (legacy location) |
| `/etc/apt/auth.conf.d/` | Repository authentication credentials |
| `/var/lib/apt/lists/` | Downloaded package indexes |
| `/var/cache/apt/archives/` | Downloaded .deb files |
| `/var/log/apt/history.log` | APT command history |
| `/var/log/apt/term.log` | Terminal output from APT operations |
| `/var/log/dpkg.log` | Low-level dpkg operations log |
| `/var/lib/apt/extended_states` | Auto-installed package tracking |
| `/var/lib/dpkg/status` | dpkg database (all package states) |
| `/var/lib/dpkg/info/<pkg>.postinst` | Package post-install script |
| `/var/lib/dpkg/info/<pkg>.preinst` | Package pre-install script |
| `/etc/apt/apt.conf.d/01autoremove` | Autoremove configuration (kernel patterns, etc.) |

## Common Errors & Fixes

| Error | Fix |
|-------|-----|
| `Could not get lock /var/lib/dpkg/lock` | Wait for other apt process, or `sudo rm /var/lib/dpkg/lock*` then `sudo dpkg --configure -a` |
| `Hash Sum mismatch` | `sudo rm -rf /var/lib/apt/lists/* && sudo apt update` |
| `The following packages have unmet dependencies` | `sudo apt --fix-broken install` |
| `E: Unable to locate package` | `sudo apt update` first; check spelling and repository |
| `NO_PUBKEY XXXXXXXX` | `sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys XXXXXXXX` or add to `/etc/apt/keyrings/` |
| `dpkg was interrupted` | `sudo dpkg --configure -a` |
| `Held packages` blocking upgrade | `sudo apt-mark unhold package-name` |
| `Sub-process /usr/bin/dpkg returned error` | `sudo dpkg --configure -a && sudo apt --fix-broken install` |
| Release file expired | Check system clock: `date`; or add `Acquire::Check-Valid-Until "false";` |

## "Packages Have Been Kept Back"

This happens when upgrading a package requires installing new dependencies or removing existing ones. `apt upgrade` won't do that by default.

```bash
# Fix: upgrade with new packages allowed
sudo apt-get --with-new-pkgs upgrade

# Or upgrade specific kept-back packages
sudo apt install --only-upgrade package1 package2

# Show which packages are held back
apt upgrade --simulate 2>/dev/null | grep "kept back"
```

> **Tip:** Use `--only-upgrade` to avoid marking the package as manually installed. This preserves auto-removal eligibility for its dependencies.

## APT Debugging

### HTTP/HTTPS Debug

```bash
# Debug HTTP acquire operations
sudo apt-get update -o Debug::Acquire::http=true 2>&1 | tee apt-debug.log

# Debug HTTPS acquire operations
sudo apt-get update -o Debug::Acquire::https=true 2>&1 | tee apt-debug-https.log

# Debug all acquire operations
sudo apt-get update -o Debug::Acquire=true 2>&1 | tee apt-debug-full.log

# Debug with network and netrc details
sudo apt-get update -o Debug::Acquire::netrc=true -o Debug::Acquire::http=true 2>&1 | tee apt-debug-network.log

# Verbose worker output (low-level package acquisition)
sudo apt-get update -o Debug::pkgAcquire=true -o Debug::pkgAcquire::Worker=true 2>&1 | tee apt-debug-verbose.log

# Full debug — all options combined
sudo apt-get update \
  -o Debug::Acquire=true \
  -o Debug::Acquire::http=true \
  -o Debug::Acquire::https=true \
  -o Debug::pkgAcquire=true \
  2>&1 | tee apt-debug-complete.log
```

### Fix Common Network Issues

```bash
# Disable HTTP pipelining (fixes NOSPLIT / ClearSigned errors behind proxies)
sudo apt-get update -o Acquire::http::Pipeline-Depth=0

# Debug with pipeline disabled
sudo apt-get update -o Acquire::http::Pipeline-Depth=0 -o Debug::Acquire::http=true 2>&1 | tee apt-debug-no-pipeline.log

# Test with custom proxy
sudo apt-get update \
  -o Acquire::http::proxy="http://proxy-host:port/" \
  -o Debug::Acquire::http=true

# Increase timeout and retries
sudo apt-get update \
  -o Acquire::http::Timeout=120 \
  -o Acquire::Retries=5

# Disable SSL verification (testing only — NOT for production)
sudo apt-get update \
  -o Acquire::https::Verify-Peer=false \
  -o Acquire::https::Verify-Host=false
```

### Test a Single Repository (Bypass sources.list)

```bash
# Isolate a single repo for testing
sudo apt-get update \
  -o Dir::Etc::sourcelist=/dev/null \
  -o Dir::Etc::sourceparts=/dev/null \
  -o APT::Get::List-Cleanup=0 \
  -o Debug::Acquire::http=true
```

### Compare with curl

```bash
# Test the same URL that apt is failing on
curl -v http://archive.ubuntu.com/ubuntu/dists/noble/InRelease

# Test with explicit proxy
curl -v -x http://proxy-host:port http://archive.ubuntu.com/ubuntu/dists/noble/InRelease
```

### Analyze Debug Logs

```bash
# Look for NOSPLIT / hash errors
grep -i "nosplit\|clearsigned.*not.*valid\|hash.*mismatch" apt-debug*.log

# Look for proxy authentication issues
grep -i "407\|authentication\|unauthorized" apt-debug*.log

# Look for SSL/TLS issues
grep -i "ssl\|tls\|certificate\|verify" apt-debug*.log

# Look for network connectivity issues
grep -i "timeout\|connection.*failed\|network.*unreachable" apt-debug*.log

# Watch live debug output
tail -f apt-debug.log
```

### Make Debug Options Permanent

```bash
# Save debug settings to a drop-in file (useful for persistent troubleshooting)
cat <<'EOF' | sudo tee /etc/apt/apt.conf.d/99debug
Debug::Acquire::http "true";
Debug::Acquire::https "true";
Acquire::http::Pipeline-Depth "0";
EOF

# Remove when done
sudo rm /etc/apt/apt.conf.d/99debug
```

## Quick Reference

```bash
# Update & Upgrade
apt update                        # Refresh package index
apt upgrade                       # Upgrade installed packages
apt full-upgrade                  # Upgrade with conflict resolution

# Install & Remove
apt install pkg                   # Install
apt install ./pkg.deb             # Install local .deb
apt remove pkg                    # Remove (keep configs)
apt purge pkg                     # Remove + delete configs
apt autoremove                    # Remove unused dependencies

# Search & Info
apt search keyword                # Search packages
apt show pkg                      # Package details
apt list --installed              # List installed
apt list --upgradeable            # List upgradeable
apt depends pkg                   # Show dependencies
apt rdepends pkg                  # Show reverse deps
apt policy pkg                    # Show versions/sources

# Maintenance
apt clean                         # Clear package cache
apt autoclean                     # Clear obsolete cache
apt --fix-broken install          # Fix broken deps

# Holds
apt-mark hold pkg                 # Prevent upgrades
apt-mark unhold pkg               # Allow upgrades
apt-mark showhold                 # List held packages
apt-mark showmanual               # List manually installed
```
