# dnf / yum Cheatsheet

Package management on RHEL 7–10 using `yum` (RHEL 7) and `dnf` (RHEL 8+). Both share nearly identical syntax — `dnf` is the successor to `yum` with improved dependency resolution and performance. On RHEL 8+, `yum` is a symlink to `dnf`.

## Quick Reference

| Action | Command |
|--------|---------|
| Install a package | `dnf install httpd` |
| Remove a package | `dnf remove httpd` |
| Update all packages | `dnf update` |
| Update a single package | `dnf update httpd` |
| Search packages | `dnf search nginx` |
| Show package info | `dnf info httpd` |
| List installed packages | `dnf list installed` |
| List available packages | `dnf list available` |
| Check for updates | `dnf check-update` |
| Clean cache | `dnf clean all` |
| Show package dependencies | `dnf deplist httpd` |
| Which package provides a file | `dnf provides /usr/bin/curl` |

---

## Install and Remove

```bash
# Install a package
dnf install httpd

# Install multiple packages
dnf install httpd php mariadb-server

# Install without confirmation
dnf install -y httpd

# Install a local RPM
dnf install ./package.rpm

# Install from a URL
dnf install https://example.com/package.rpm

# Reinstall a package
dnf reinstall httpd

# Remove a package
dnf remove httpd

# Remove with dependencies (autoremove orphaned packages)
dnf autoremove

# Remove a package without removing dependencies
rpm -e --nodeps package_name

# Install a specific version
dnf install httpd-2.4.57-5.el9
```

---

## Update and Upgrade

```bash
# Check for available updates
dnf check-update

# Update all packages
dnf update

# Update a single package
dnf update httpd

# Update and remove obsolete packages
dnf upgrade

# Security updates only
dnf update --security

# List available security updates
dnf updateinfo list security

# Show advisory details
dnf updateinfo info RHSA-2024:1234

# Exclude a package from update
dnf update --exclude=kernel*

# Update to a specific version
dnf update httpd-2.4.57-5.el9
```

---

## Search and Info

```bash
# Search by name and description
dnf search nginx

# Show package details
dnf info httpd

# Show installed package details
dnf info --installed httpd

# Which package provides a file
dnf provides /etc/httpd/conf/httpd.conf
dnf provides "*/fstab"
dnf provides "*/httpd.conf"

# Which package provides a library
dnf provides libstdc++.so.5

# List packages providing a specific dependency
dnf repoquery --whatprovides 'perl(Net::HTTP)'

# List files in a package
rpm -ql httpd

# List files in a package (not installed)
dnf repoquery -l httpd
repoquery --list httpd

# List config files for a package
rpm -qc httpd

# List documentation files
rpm -qd httpd

# Show changelog
dnf changelog httpd
rpm -q --changelog httpd | head -40
```

---

## List Packages

```bash
# List all installed packages
dnf list installed

# List all available packages
dnf list available

# List available packages from a specific repo
dnf list available | grep epel
repoquery -qa --repoid=epel

# List all packages from a specific repo (disable all others)
dnf --disablerepo="*" --enablerepo="epel" list available

# List installed and available versions
dnf list httpd

# List all packages (installed + available)
dnf list all

# List recently installed
dnf list recent

# List packages by pattern
dnf list "php*"

# Show all versions of a package (duplicates)
dnf list php-gd --showduplicates

# List upgradable packages
dnf list upgrades

# List installed packages sorted by size
rpm -qa --queryformat '%{SIZE} %{NAME}\n' | sort -rn | head -20

# List all installed plugins
dnf list installed | grep plugin
```

---

## Repository Management

```bash
# List enabled repos
dnf repolist

# List all repos (enabled and disabled)
dnf repolist all

# Enable a repo
dnf config-manager --enable epel

# Disable a repo
dnf config-manager --disable epel

# Add a repo from URL
dnf config-manager --add-repo https://example.com/repo.repo

# Install from a specific repo
dnf install --repo=epel htop

# Temporarily enable a disabled repo
dnf install --enablerepo=epel-testing htop

# Temporarily disable a repo
dnf update --disablerepo=epel

# Show repo info
dnf repoinfo epel

# Refresh repo metadata
dnf makecache

# Clean all cached data
dnf clean all

# Clean only metadata
dnf clean metadata

# Clean only packages
dnf clean packages
```

### Repository Configuration Files

```bash
# Repo config location
/etc/yum.repos.d/*.repo

# Example repo file
cat /etc/yum.repos.d/example.repo
```

```ini
[example-repo]
name=Example Repository
baseurl=https://example.com/rhel/$releasever/$basearch/
enabled=1
gpgcheck=1
gpgkey=https://example.com/RPM-GPG-KEY-example
```

---

## Groups and Modules

### Package Groups

```bash
# List available groups
dnf group list

# List all groups (including hidden)
dnf group list --hidden

# Show group info
dnf group info "Development Tools"

# Install a group
dnf group install "Development Tools"
dnf groupinstall "Server with GUI"

# Remove a group
dnf group remove "Development Tools"

# Update a group
dnf group update "Development Tools"

# List installed groups
dnf group list --installed
```

### Module Streams (RHEL 8+)

```bash
# List all modules
dnf module list

# List enabled modules
dnf module list --enabled

# List installed modules
dnf module list --installed

# List versions of a specific module
dnf module list php

# Show module info
dnf module info php

# Show packages installed by profiles of a module
dnf module info --profile php:8.1

# Find which module provides a package
dnf module provides php-fpm

# Enable a module stream (without installing)
dnf module enable php:8.1

# Install a module with default profile
dnf module install php:8.1

# Install a module with specific profile
dnf module install php:8.1/devel

# Switch module stream
dnf module reset php
dnf module enable php:8.2
dnf module install php:8.2

# Disable a module and remove its packages
dnf module remove php
dnf module disable php

# Reset a module (remove stream lock)
dnf module reset php
```

---

## History and Rollback

The history database is stored in:
- RHEL 7: `/var/lib/yum/history/`
- RHEL 8+: `/var/lib/dnf/history/`

```bash
# Show transaction history
dnf history

# List all transactions
dnf history list all

# Show details of a transaction
dnf history info 15

# Show packages affected in a transaction
dnf history info 15 --verbose

# Undo a transaction (rollback)
dnf history undo 15

# Redo a transaction
dnf history redo 15

# Rollback to a previous state (undo all transactions after this point)
dnf history rollback 12

# Show history for a specific package
dnf history list firefox

# Show details about all transactions for a package
dnf history info firefox

# Show the version of packages for all transactions
yum history packages-list firefox

# Clear all history (yum on RHEL 7)
yum history new
```

---

## Dependencies

```bash
# Show package dependencies (first level)
dnf deplist httpd

# Show reverse dependencies (what requires this package)
dnf repoquery --whatrequires httpd

# Show all requires for a package
dnf repoquery --requires httpd

# Show provides
dnf repoquery --provides httpd

# Display the entire tree of dependencies
repoquery --tree-requires bash

# Show duplicates of a package
repoquery php-gd --show-duplicates

# Check for broken dependencies
dnf check

# Install missing dependencies for a local RPM
dnf install ./package.rpm

# Skip broken packages during update
dnf update --skip-broken

# Download package without installing
dnf download httpd
dnf download --resolve httpd    # Include dependencies
dnf download --destdir=/tmp httpd

# Download with yumdownloader (RHEL 7)
yumdownloader --destdir /tmp kernel
```

---

## Package Locking (Versionlock)

Prevent specific packages from being updated:

```bash
# Install the plugin
dnf install dnf-plugin-versionlock     # RHEL 8+
yum install yum-plugin-versionlock     # RHEL 7

# Lock a package at current version
dnf versionlock add httpd

# List locked packages
dnf versionlock list

# Remove a lock
dnf versionlock delete httpd

# Clear all locks
dnf versionlock clear
```

---

## Caching and Offline Use

```bash
# Cache packages for offline use
dnf makecache

# Keep downloaded packages in cache
# Edit /etc/dnf/dnf.conf:
# keepcache=1

# Download packages only (no install)
dnf install --downloadonly --downloaddir=/tmp/packages httpd

# Install from local cache
dnf install --cacheonly httpd

# Create a local repo from downloaded RPMs
dnf install createrepo_c
createrepo_c /path/to/rpms/
```

---

## DNF Configuration

### /etc/dnf/dnf.conf (RHEL 8+) or /etc/yum.conf (RHEL 7)

```ini
[main]
gpgcheck=1
installonly_limit=3
clean_requirements_on_remove=True
best=True
skip_if_unavailable=False
keepcache=0
max_parallel_downloads=10
defaultyes=True
```

| Directive | Description |
|-----------|-------------|
| `gpgcheck` | Verify GPG signatures (1=yes, 0=no) |
| `installonly_limit` | Number of kernel versions to keep |
| `clean_requirements_on_remove` | Remove orphaned deps on package removal |
| `best` | Always install the newest version |
| `skip_if_unavailable` | Skip unavailable repos instead of erroring |
| `keepcache` | Keep downloaded RPMs in cache |
| `max_parallel_downloads` | Concurrent downloads (dnf only) |
| `defaultyes` | Default to yes on prompts |
| `exclude` | Packages to exclude globally |
| `proxy` | HTTP proxy URL |
| `proxy_username` | Proxy authentication username |
| `proxy_password` | Proxy authentication password |

---

## Kernel Management

```bash
# List installed kernels
dnf list installed kernel

# Remove old kernels (keep last 2)
dnf remove $(dnf repoquery --installonly --latest-limit=-2 -q)

# Set how many kernels to keep
# Edit /etc/dnf/dnf.conf: installonly_limit=3

# Install a specific kernel version
dnf install kernel-5.14.0-362.el9

# Show running kernel
uname -r

# Show default boot kernel
grubby --default-kernel
```

---

## GPG Keys

```bash
# Import a GPG key
rpm --import https://example.com/RPM-GPG-KEY-example

# List imported keys
rpm -qa gpg-pubkey*

# Show key details
rpm -qi gpg-pubkey-12345678-abcdef01

# Remove a key
rpm -e gpg-pubkey-12345678-abcdef01

# Disable GPG check temporarily
dnf install --nogpgcheck package_name
```

---

## Useful One-Liners

```bash
# Show 10 largest installed packages
rpm -qa --queryformat '%{SIZE}\t%{NAME}-%{VERSION}\n' | sort -rn | head -10

# List packages installed from a specific repo
dnf list installed | grep @epel

# Count installed packages
rpm -qa | wc -l

# List packages installed today
rpm -qa --queryformat '%{INSTALLTIME} %{NAME}\n' | sort -rn | head -20

# Find which package owns a command
rpm -qf $(which curl)

# Show install date of a package
rpm -qi httpd | grep "Install Date"

# List all repos and their URLs
dnf repolist -v

# Compare installed packages between two systems
rpm -qa --queryformat '%{NAME}\n' | sort > /tmp/packages.txt

# Dry-run an update (see what would change)
dnf update --assumeno

# Reinstall all packages from a specific repo
dnf reinstall $(dnf list installed | grep @epel | awk '{print $1}')

# Downgrade a package
dnf downgrade httpd

# Show package dependencies as a tree
dnf repoquery --tree --whatrequires httpd
```

### Downgrade a Package

List available versions first, then downgrade:

```bash
dnf list firefox --showduplicates
dnf downgrade firefox
dnf downgrade firefox-102.12.0-1.el8_8
```

Note: downgrade does not resolve dependencies automatically. Related packages may be removed unless downgraded together:

```bash
# Downgrade related packages together to preserve dependencies
dnf downgrade httpd-2.4.37-43.el8 httpd-tools-2.4.37-43.el8 mod_ssl-2.4.37-43.el8
```

> **Warning:** Downgrading the following packages is unsupported — they assume an update-only process:
> - `dbus`
> - `glibc` (and its dependencies such as gcc)
> - `selinux-policy*`
>
> Downgrading an entire system to a previous minor release (e.g., RHEL 8.5 → 8.4) is not recommended. Use `yum history undo` for small rollbacks instead.

---

## yum vs dnf Differences

| Feature | yum (RHEL 7) | dnf (RHEL 8+) |
|---------|-------------|---------------|
| Performance | Slower | Faster (C library for dep resolution) |
| API | Python-based | libdnf (C/C++) |
| Module streams | Not available | Supported |
| Automatic conflict resolution | Limited | Improved |
| Config file | `/etc/yum.conf` | `/etc/dnf/dnf.conf` |
| Parallel downloads | No | `max_parallel_downloads` |
| Weak dependencies | No | Recommends/Supplements |
| Rich dependencies | No | Boolean expressions |
| `download` subcommand | `yumdownloader` | `dnf download` |
| `versionlock` | Plugin | Plugin |
| `needs-restarting` | `needs-restarting` | `dnf needs-restarting` |
| Command compatibility | — | `yum` symlinks to `dnf` |

---

## Plugins

### yum-utils (RHEL 7) / dnf-utils (RHEL 8+)

```bash
# RHEL 7
yum install yum-utils

# RHEL 8+
dnf install dnf-utils
```

#### package-cleanup

```bash
# Remove old kernels — keep only the last 2 versions
package-cleanup --oldkernels --count=2

# Scan for duplicate packages in local RPM database
package-cleanup --dupes

# Clean out older duplicate versions
package-cleanup --cleandupes

# List dependency problems in local RPM database
package-cleanup --problems

# List installed packages not available from configured repos (orphans)
package-cleanup --orphans

# Find packages not required by any other package (leaves)
package-cleanup --leaves
```

#### repoquery

```bash
# Query all packages (like rpm -ql for uninstalled packages)
repoquery -ql httpd

# Show duplicates
repoquery php-gd --show-duplicates

# Display the entire dependency tree
repoquery --tree-requires bash

# Query all packages providing a capability
repoquery -a --whatprovides 'perl(Net::HTTP)'

# List files in a package
repoquery --list mlocate
```

#### Other yum-utils Tools

```bash
# Find incomplete/aborted yum transactions and complete them
yum-complete-transaction

# Find which repo a package was installed from
find-repos-of-install wget

# Output a full package dependency graph in dot format
repo-graph --repoid=epel

# Display unresolved dependencies for a repository
repoclosure

# Synchronize a remote repo to a local directory
reposync --repoid=epel --download_path=/tmp

# List all yum-utils tools
man yum-utils
```

### Security Updates (yum-plugin-security)

Available by default on RHEL 7+. On older versions:

```bash
# RHEL 6
yum install yum-plugin-security

# RHEL 5
yum install yum-security
```

#### Commands

```bash
# List all security-relevant updates
dnf --security check-update

# Apply all security updates
dnf --security update

# List all updates with errata ID
dnf updateinfo list

# List only critical security updates
dnf updateinfo list --security --sec-severity=Critical

# Apply only critical security updates
dnf update --security --sec-severity=Critical

# Show errata summary (bugzillas, CVEs, security updates)
dnf updateinfo summary

# Show CVEs for all available updates
dnf updateinfo list cves

# Show detailed info about a specific CVE
dnf updateinfo info --cve CVE-2017-5470

# Update the package for a specific CVE
dnf update --cve CVE-2017-5470

# Update the package for a specific errata
dnf update --advisory RHSA-2012:1407
```

### Changelog Plugin

```bash
# RHEL 7
yum install yum-plugin-changelog

# View all changelogs for a package
dnf changelog kernel

# View latest changelog entry
dnf changelog --count=1 kernel

# View changelogs since a specific date
dnf changelog --since=2017-01-01 kernel

# Show changelogs for pending updates before applying
dnf update --changelog
```

---

## Important Files

| Path | Purpose |
|------|---------|
| `/etc/yum.conf` | Main YUM configuration (RHEL 7) |
| `/etc/dnf/dnf.conf` | Main DNF configuration (RHEL 8+) |
| `/etc/yum.repos.d/` | Repository configuration files |
| `/etc/yum/pluginconf.d/` | Plugin configuration files |
| `/etc/sysconfig/kernel` | Kernel configuration (default kernel, update behavior) |
| `/var/cache/yum/` | YUM cache (RHEL 7) |
| `/var/cache/dnf/` | DNF cache (RHEL 8+) |
| `/var/lib/rpm/` | RPM database |
| `/var/log/dnf.log` | DNF transaction log |
| `/var/log/yum.log` | YUM transaction log (RHEL 7) |

---

## Useful Links

- How to use yum/dnf to downgrade or rollback packages: https://access.redhat.com/solutions/29617
- Restricting a Package to a Fixed Version Number with yum: https://access.redhat.com/solutions/98873
- Listing/installing security updates only: https://access.redhat.com/solutions/10021
- Creating a local mirror without Satellite: https://access.redhat.com/solutions/55654

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| "No package available" | Package not in enabled repos | `dnf repolist` and enable correct repo |
| "Conflict" during install | Package versions conflict | `dnf install --allowerasing package` |
| Metadata download errors | Stale cache or network issues | `dnf clean all && dnf makecache` |
| "Protected packages" on remove | Package is protected (systemd, dnf) | Cannot be removed (by design) |
| GPG key error | Key not imported | `rpm --import /path/to/key` |
| Slow downloads | Single-threaded | Add `max_parallel_downloads=10` to dnf.conf |
| Broken dependencies | Partially installed packages | `dnf check && dnf distro-sync` |
| Module stream locked | Previous module enabled | `dnf module reset module_name` |
| "Transaction check error" | File conflicts with another package | `dnf install --allowerasing package` |

### Check for Problems

```bash
# Verify package integrity
rpm -Va

# Check for broken dependencies
dnf check

# Sync all packages to latest available
dnf distro-sync

# Rebuild RPM database
rpm --rebuilddb

# Check which services need restarting after update
dnf needs-restarting

# List only services that need restarting
dnf needs-restarting -s

# Check if reboot is needed
dnf needs-restarting -r

# Mark a package as user-installed (prevents autoremove)
dnf mark install package_name

# Mark a package as a dependency (eligible for autoremove)
dnf mark remove package_name
```
