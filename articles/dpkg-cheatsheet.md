# dpkg Cheatsheet

`dpkg` is the low-level package manager for Debian-based systems (Ubuntu, Debian, Linux Mint, Pop!_OS). It handles `.deb` packages directly — installing, removing, querying, and verifying them. Higher-level tools like `apt` and `apt-get` use `dpkg` under the hood but add dependency resolution on top.

## Install Packages

```bash
# Install a .deb file
sudo dpkg -i package.deb

# Install multiple .deb files
sudo dpkg -i pkg1.deb pkg2.deb pkg3.deb

# Install all .deb files in a directory
sudo dpkg -i /path/to/debs/*.deb

# Install and fix broken dependencies after
sudo dpkg -i package.deb
sudo apt-get install -f    # Resolve missing dependencies

# Force install (skip dependency checks — use with caution)
sudo dpkg -i --force-depends package.deb

# Force overwrite files from another package
sudo dpkg -i --force-overwrite package.deb

# Force install even if architecture doesn't match
sudo dpkg -i --force-architecture package.deb

# Unpack without configuring (two-step install)
sudo dpkg --unpack package.deb
sudo dpkg --configure package-name
```

## Remove Packages

```bash
# Remove a package (keep config files)
sudo dpkg -r package-name
sudo dpkg --remove package-name

# Purge a package (remove everything including config files)
sudo dpkg -P package-name
sudo dpkg --purge package-name

# Remove multiple packages
sudo dpkg -r pkg1 pkg2 pkg3

# Purge all removed packages (configs still on disk)
dpkg -l | grep '^rc' | awk '{print $2}' | sudo xargs dpkg --purge

# Force remove even if other packages depend on it
sudo dpkg -r --force-depends package-name
```

## Query Installed Packages

```bash
# List all installed packages
dpkg -l
dpkg --list

# List with pattern matching (wildcards)
dpkg -l 'nginx*'
dpkg -l 'lib*dev'

# Check if a specific package is installed
dpkg -l package-name
dpkg -s package-name

# Show package status and details
dpkg -s package-name
dpkg --status package-name

# Show available package info (from dpkg available database)
dpkg -p package-name

# List files installed by a package
dpkg -L package-name
dpkg --listfiles package-name

# Find which package owns a file
dpkg -S /path/to/file
dpkg --search /path/to/file

# Find package for a command
dpkg -S $(which nginx)

# Search for file pattern across all packages
dpkg -S '*nginx.conf*'
dpkg -S '*/bin/python3'
```

## Query .deb Files (Not Installed)

```bash
# Show package info from a .deb file
dpkg -I package.deb
dpkg --info package.deb

# List files inside a .deb file (without installing)
dpkg -c package.deb
dpkg --contents package.deb

# Extract a .deb file without installing
dpkg -x package.deb /tmp/extracted/

# Extract control information from a .deb
dpkg -e package.deb /tmp/control/

# Extract everything (data + control)
dpkg-deb --raw-extract package.deb /tmp/full-extract/
```

## Configure Packages

```bash
# Configure all unpacked but unconfigured packages
sudo dpkg --configure -a

# Configure a specific package
sudo dpkg --configure package-name

# Reconfigure an installed package (re-run its config scripts)
sudo dpkg-reconfigure package-name

# Reconfigure with a specific frontend (noninteractive for scripts)
sudo DEBIAN_FRONTEND=noninteractive dpkg-reconfigure package-name

# Reconfigure with dialog/readline
sudo dpkg-reconfigure --frontend=dialog package-name

# Reconfigure all packages (useful after locale/timezone changes)
sudo dpkg-reconfigure -a

# Set default priority for reconfigure prompts
sudo dpkg-reconfigure --priority=low package-name
```

## Package States

The `dpkg -l` output has a three-character status field:

```
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name           Version      Architecture Description
+++-==============-============-============-=====================================
ii  nginx          1.18.0-6     amd64        high performance web server
rc  apache2        2.4.52-1     amd64        Apache HTTP Server (config files remain)
```

| Code | Meaning |
|------|---------|
| `ii` | Installed, configured correctly |
| `rc` | Removed, config files still present |
| `un` | Unknown, not installed |
| `hi` | Hold, installed |
| `iU` | Installed, unpacked but not configured |
| `iF` | Installed, half-configured (config failed) |
| `iH` | Installed, half-installed (install failed midway) |
| `pn` | Purged, not installed |
| `iW` | Installed, trigger awaited |
| `iT` | Installed, trigger pending |

## Hold / Unhold Packages

Prevent a package from being upgraded:

```bash
# Hold a package (prevent upgrades)
sudo dpkg --set-selections <<< "package-name hold"
echo "package-name hold" | sudo dpkg --set-selections

# Unhold (allow upgrades again)
sudo dpkg --set-selections <<< "package-name install"
echo "package-name install" | sudo dpkg --set-selections

# Check hold status
dpkg --get-selections | grep hold

# Check status of a specific package
dpkg --get-selections package-name

# Alternative: use apt-mark (easier syntax)
sudo apt-mark hold package-name
sudo apt-mark unhold package-name
apt-mark showhold
```

## dpkg-deb (Working with .deb Files)

```bash
# Build a .deb from a directory structure
dpkg-deb --build /path/to/package-dir
dpkg-deb -b /path/to/package-dir output.deb

# Show package info
dpkg-deb --info package.deb

# Show specific field from control
dpkg-deb --field package.deb Version
dpkg-deb --field package.deb Depends
dpkg-deb --field package.deb Architecture

# List contents
dpkg-deb --contents package.deb

# Extract data files
dpkg-deb --extract package.deb /tmp/data/

# Extract control info (postinst, prerm, control, etc.)
dpkg-deb --control package.deb /tmp/ctrl/

# Verify .deb integrity
dpkg-deb --info package.deb > /dev/null && echo "OK" || echo "CORRUPT"
```

## dpkg-query (Advanced Queries)

```bash
# Show installed packages in custom format
dpkg-query -W -f '${Package} ${Version}\n'

# Show package with specific fields
dpkg-query -W -f '${Package}\t${Version}\t${Installed-Size}\n' nginx

# List all packages and their sizes (sorted)
dpkg-query -W -f '${Installed-Size}\t${Package}\n' | sort -n

# Show all packages with architecture
dpkg-query -W -f '${Package}:${Architecture} ${Version}\n'

# List only package names
dpkg-query -W -f '${Package}\n'

# Show description
dpkg-query -W -f '${Package}: ${Description}\n' nginx

# Show status of a package
dpkg-query -s package-name

# List files of a package
dpkg-query -L package-name

# Find package owning a file
dpkg-query -S /usr/bin/curl

# Search by command name (without full path)
dpkg-query -S umount

# Count installed packages
dpkg-query -W | wc -l

# Show packages matching a pattern
dpkg-query -W 'python3*'

# Show packages in a specific state
dpkg-query -W -f '${Package} ${db:Status-Abbrev}\n' | grep '^.i'
```

### Useful Format Strings

| Field | Description |
|-------|-------------|
| `${Package}` | Package name |
| `${Version}` | Package version |
| `${Architecture}` | Package architecture |
| `${Installed-Size}` | Installed size in KB |
| `${Description}` | Short description |
| `${Status}` | Full status string |
| `${db:Status-Abbrev}` | Short status (ii, rc, etc.) |
| `${Maintainer}` | Package maintainer |
| `${Section}` | Package section/category |
| `${Priority}` | Package priority |
| `${Depends}` | Dependencies |
| `${Pre-Depends}` | Pre-dependencies |
| `${Provides}` | Virtual packages provided |
| `${Conffiles}` | Configuration files |
| `${Homepage}` | Upstream homepage URL |
| `${Source}` | Source package name |

## Verify Package Integrity

```bash
# Verify all installed packages (check for modified files)
sudo dpkg --verify
sudo dpkg -V

# Verify a specific package
sudo dpkg --verify package-name
sudo dpkg -V package-name

# Verify output format:
# ??5?????? c /etc/nginx/nginx.conf
# Characters: file-type, mode, owner, group, size, checksum, ...
# 'c' = config file (expected to be modified)
```

### Verify Output Columns

| Position | Meaning |
|----------|---------|
| 1 | File type changed |
| 2 | Permissions changed |
| 3 | Owner changed |
| 4 | Group changed |
| 5 | Size changed |
| 6 | (reserved) |
| 7 | (reserved) |
| 8 | (reserved) |
| 9 | Checksum (MD5) changed |

A `?` means the check was not performed. A `.` means no change detected.

## dpkg Triggers

```bash
# Process all pending triggers
sudo dpkg --configure --pending
sudo dpkg --triggers-only -a

# Process triggers for a specific package
sudo dpkg --triggers-only package-name
```

## Diversions

Diversions tell dpkg to install a file to a different location than the package specifies — useful when you need to replace a package-provided file with your own:

```bash
# Create a diversion (redirect a file)
sudo dpkg-divert --add --rename --divert /usr/bin/original.distrib /usr/bin/original

# List all diversions
dpkg-divert --list

# List diversions for a specific file
dpkg-divert --list /usr/bin/original

# Remove a diversion
sudo dpkg-divert --remove --rename /usr/bin/original

# Divert with a local package (won't be overwritten by any package)
sudo dpkg-divert --local --add --rename /etc/default/grub
```

## Alternatives System

Manage multiple versions of the same command (e.g., `python`, `java`, `editor`):

```bash
# List all alternatives for a group
update-alternatives --list editor

# Display current alternative and all options
update-alternatives --display editor

# Set alternative interactively
sudo update-alternatives --config editor
sudo update-alternatives --config java

# Set alternative non-interactively (auto mode)
sudo update-alternatives --auto editor

# Install a new alternative
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.11 1
sudo update-alternatives --install /usr/bin/editor editor /usr/bin/vim 100

# Set manually (override auto)
sudo update-alternatives --set editor /usr/bin/vim

# Remove an alternative
sudo update-alternatives --remove editor /usr/bin/nano

# Remove all alternatives for a group
sudo update-alternatives --remove-all editor

# Show all registered alternative groups
update-alternatives --get-selections
```

## Architecture Management

```bash
# List configured architectures
dpkg --print-architecture          # Primary architecture
dpkg --print-foreign-architectures # Additional architectures

# Add a foreign architecture (e.g., i386 on amd64)
sudo dpkg --add-architecture i386
sudo apt update

# Remove a foreign architecture
sudo dpkg --remove-architecture i386

# Install a package for a specific architecture
sudo apt install package-name:i386
```

## Repair and Recovery

```bash
# Fix interrupted installs
sudo dpkg --configure -a

# Force reconfiguration of all packages
sudo dpkg --configure -a --force-confold

# Clear dpkg lock (if another process crashed)
sudo rm /var/lib/dpkg/lock-frontend
sudo rm /var/lib/dpkg/lock
sudo dpkg --configure -a

# Reinstall a package (refresh all files)
sudo apt install --reinstall package-name

# Fix broken dpkg database
sudo cp /var/lib/dpkg/status /var/lib/dpkg/status.bak
sudo dpkg --configure -a

# Rebuild dpkg available file
sudo dpkg --clear-avail
sudo apt-cache dumpavail | sudo dpkg --update-avail /dev/stdin

# Force remove a stuck package
sudo dpkg --remove --force-remove-reinstreq package-name

# Audit packages (find broken ones)
sudo dpkg --audit
```

## One-Liners

```bash
# List all installed packages (names only)
dpkg --get-selections | grep -v deinstall | awk '{print $1}'

# List all installed packages (alternative — grep for "install" state)
dpkg --get-selections | grep -w "install" | cut -f1 > packages_list.txt

# List packages sorted by installed size (largest first)
dpkg-query -W -f '${Installed-Size}\t${Package}\n' | sort -rn | head -20

# Find all config files for a package
dpkg-query -W -f '${Conffiles}\n' nginx | tr ',' '\n'

# Find orphaned config files (packages removed, configs remain)
dpkg -l | grep '^rc' | awk '{print $2}'

# Count total installed packages
dpkg -l | grep '^ii' | wc -l

# Export list of installed packages (for cloning to another machine)
dpkg --get-selections > packages.txt

# Import and install packages on another machine
sudo dpkg --set-selections < packages.txt
sudo apt-get dselect-upgrade

# Find which package provides a missing library
dpkg -S libssl.so.1.1

# List all packages from a specific repository/source
apt list --installed 2>/dev/null | grep "docker"

# Show package changelog
apt changelog package-name

# Find manually installed packages (not auto-installed as deps)
apt-mark showmanual

# List auto-installed packages
apt-mark showauto

# Show reverse dependencies (what depends on this package)
apt-cache rdepends package-name

# Compare installed version with available
apt list --upgradeable 2>/dev/null

# Extract a single file from a .deb without installing
dpkg-deb --fsys-tarfile package.deb | tar -xf - ./path/to/file

# List all files installed by all packages (full system manifest)
dpkg -l | grep '^ii' | awk '{print $2}' | xargs dpkg -L 2>/dev/null

# Find packages installed from a PPA
apt list --installed 2>/dev/null | grep ppa

# Show when packages were last installed/upgraded
grep " install\| upgrade" /var/log/dpkg.log | tail -20

# Show full install history
zcat /var/log/dpkg.log.*.gz | grep " install " | sort
```

## dpkg vs apt Commands

| Task | dpkg | apt / apt-get |
|------|------|---------------|
| Install .deb file | `dpkg -i pkg.deb` | `apt install ./pkg.deb` |
| Remove package | `dpkg -r pkg` | `apt remove pkg` |
| Purge package | `dpkg -P pkg` | `apt purge pkg` |
| List installed | `dpkg -l` | `apt list --installed` |
| Package info | `dpkg -s pkg` | `apt show pkg` |
| List files | `dpkg -L pkg` | — (use dpkg) |
| Find owner | `dpkg -S /path` | — (use dpkg) |
| Fix broken | `dpkg --configure -a` | `apt --fix-broken install` |
| Hold package | `dpkg --set-selections` | `apt-mark hold pkg` |
| Reconfigure | `dpkg-reconfigure pkg` | — (use dpkg-reconfigure) |

## Important Files

| File | Purpose |
|------|---------|
| `/var/lib/dpkg/status` | Database of installed packages and their state |
| `/var/lib/dpkg/available` | Database of available packages |
| `/var/lib/dpkg/info/` | Package control files (postinst, prerm, conffiles, md5sums) |
| `/var/lib/dpkg/info/*.list` | List of files belonging to each package |
| `/var/lib/dpkg/info/*.conffiles` | Config files tracked by dpkg |
| `/var/lib/dpkg/info/*.md5sums` | MD5 checksums of installed files |
| `/var/lib/dpkg/info/*.postinst` | Post-install script |
| `/var/lib/dpkg/info/*.prerm` | Pre-remove script |
| `/var/lib/dpkg/info/*.postrm` | Post-remove script |
| `/var/lib/dpkg/info/*.preinst` | Pre-install script |
| `/var/lib/dpkg/lock` | Lock file (prevents concurrent dpkg operations) |
| `/var/lib/dpkg/lock-frontend` | Frontend lock file |
| `/var/log/dpkg.log` | dpkg operation log |
| `/etc/dpkg/dpkg.cfg` | Global dpkg configuration |
| `/etc/dpkg/dpkg.cfg.d/` | Drop-in config directory |

## Force Options

Use `--force-<thing>` to override safety checks. These are dangerous — use only when you know what you're doing:

| Option | Description |
|--------|-------------|
| `--force-depends` | Install even if dependencies are broken |
| `--force-remove-reinstreq` | Remove a package that requires reinstallation |
| `--force-overwrite` | Overwrite files from another package |
| `--force-confold` | Keep old config files without prompting |
| `--force-confnew` | Always install new config files without prompting |
| `--force-confdef` | Use default action for config files |
| `--force-confmiss` | Always install missing config files |
| `--force-architecture` | Install package for wrong architecture |
| `--force-breaks` | Install even if it breaks another package |
| `--force-conflicts` | Install even if it conflicts with another package |
| `--force-bad-path` | Allow install even if PATH has problems |
| `--force-not-root` | Try to (un)install even without root |
| `--force-all` | Apply all force options (extremely dangerous) |

```bash
# Common force patterns
sudo dpkg -i --force-overwrite package.deb    # File conflicts
sudo dpkg -i --force-depends package.deb      # Missing deps (fix after with apt -f install)
sudo dpkg -r --force-remove-reinstreq pkg     # Stuck package
sudo dpkg --configure -a --force-confold      # Keep all old configs during recovery
```

## Maintainer Scripts

Each package can have these scripts that run during install/remove:

| Script | When It Runs |
|--------|--------------|
| `preinst` | Before files are unpacked |
| `postinst` | After files are unpacked and configured |
| `prerm` | Before package is removed |
| `postrm` | After package is removed |

```bash
# View a package's maintainer scripts
cat /var/lib/dpkg/info/package-name.postinst
cat /var/lib/dpkg/info/package-name.prerm

# List all scripts for a package
ls /var/lib/dpkg/info/package-name.*

# Run a postinst manually (rare, for recovery)
sudo /var/lib/dpkg/info/package-name.postinst configure
```

## Creating .deb Packages (Minimal)

```bash
# Directory structure for a minimal .deb
mkdir -p mypackage/DEBIAN
mkdir -p mypackage/usr/local/bin

# Create control file (required)
cat > mypackage/DEBIAN/control << 'EOF'
Package: mypackage
Version: 1.0.0
Section: utils
Priority: optional
Architecture: amd64
Maintainer: Your Name <you@example.com>
Description: Short description of mypackage
 Longer description that can span
 multiple lines with a leading space.
Depends: bash (>= 4.0), curl
EOF

# Add your files
cp myscript.sh mypackage/usr/local/bin/myscript
chmod 755 mypackage/usr/local/bin/myscript

# Optional: add postinst script
cat > mypackage/DEBIAN/postinst << 'EOF'
#!/bin/bash
echo "mypackage installed successfully"
EOF
chmod 755 mypackage/DEBIAN/postinst

# Build the .deb
dpkg-deb --build mypackage
# Creates mypackage.deb

# Or specify output name
dpkg-deb -b mypackage mypackage_1.0.0_amd64.deb

# Verify the built package
dpkg-deb --info mypackage_1.0.0_amd64.deb
dpkg-deb --contents mypackage_1.0.0_amd64.deb
```

## Package Dependencies & Analysis

```bash
# Show package dependencies from status
dpkg -s package-name | grep -E '^(Depends|Pre-Depends)'

# Show Provides/Conflicts/Replaces
dpkg -s package-name | grep -E '^(Provides|Conflicts|Replaces)'

# Find what depends on a package (reverse dependencies)
apt-cache rdepends package-name

# Show full dependency tree (recursive)
apt-cache depends --recurse --no-recommends package-name

# Show dependency chain drill-down
apt-cache depends package-name | grep "Depends:" | awk '{print $2}' | \
  xargs -I {} apt-cache depends {}

# Find orphaned packages (no longer needed by anything)
deborphan

# Purge all orphaned libraries in one shot
sudo dpkg --purge $(deborphan)

# Find config files for a package
dpkg -L package-name | grep /etc

# Show package changelog from disk
zcat /usr/share/doc/package-name/changelog.Debian.gz | head -20
```

## Advanced .deb Extraction

```bash
# Extract .deb with verbose listing (-X is -x with verbose output)
dpkg -X package.deb /target/directory

# Extract with control metadata into DEBIAN subdirectory (--raw-extract)
dpkg-deb -R package.deb /target/directory

# Manual .deb extraction using ar (low-level)
ar x package.deb
# Produces: debian-binary, control.tar.xz (or .gz), data.tar.xz (or .gz)

# Inspect data archive contents
tar -tf data.tar.xz | head -20

# Inspect control archive
tar -tf control.tar.xz

# Download a package without installing
apt-get download package-name
```

## Backup & Restore Package State

```bash
# Backup package selection (names + state)
dpkg --get-selections > package-list.txt

# Backup with exact versions
dpkg-query -f '${Package}=${Version}\n' -W > packages-with-versions.txt

# Restore package selection
sudo dpkg --set-selections < package-list.txt
sudo apt-get dselect-upgrade

# Clone packages to a remote machine via SSH
dpkg --get-selections | ssh user@remote 'sudo dpkg --set-selections && sudo apt-get dselect-upgrade -y'

# Mass install from a plain list
cat package-list.txt | xargs sudo apt-get install -y
```

## APT Version Pinning

Control which versions of packages can be installed:

```bash
# Pin a package to a specific version
cat > /etc/apt/preferences.d/package-name << 'EOF'
Package: package-name
Pin: version 1.2.3*
Pin-Priority: 1001
EOF

# Pin all packages from a specific repository
cat > /etc/apt/preferences.d/my-repo << 'EOF'
Package: *
Pin: origin "repo.example.com"
Pin-Priority: 100
EOF

# Show effective pin for a package
apt-cache policy package-name
```

| Priority | Effect |
|----------|--------|
| `1001+` | Install even if it's a downgrade |
| `990` | Install if not yet installed or if newer |
| `500` | Default (normal install) |
| `100` | Install only if no other version available |
| `-1` | Never install |

## Package Search (apt-file)

Search for files across all packages (installed and not installed):

```bash
# Install apt-file
sudo apt install apt-file
sudo apt-file update

# Find which package provides a file
apt-file search /usr/bin/command
apt-file search filename

# Search by exact path
apt-file find /usr/bin/htop

# Search package contents by pattern
apt-file search --regexp '.*nginx.*\.conf$'

# Search by description (apt-cache)
apt-cache search --names-only "web server"
apt-cache search "image manipulation"
```

## Integration with systemd

```bash
# Find systemd services installed by a package
dpkg -L package-name | grep "\.service$"

# Find which package installed a service file
dpkg -S /lib/systemd/system/nginx.service
dpkg -S /etc/systemd/system/service-name.service

# List all packages that ship systemd units
dpkg -l | awk '/^ii/{print $2}' | xargs -I {} sh -c \
  'dpkg -L {} 2>/dev/null | grep -q "\.service$" && echo {}'
```

## Integration with find

```bash
# Find all config files on system and their owning packages
find /etc -type f -exec dpkg -S {} \; 2>/dev/null | cut -d: -f1 | sort | uniq

# Find packages that installed files in a specific path
find /usr/bin -type f -exec dpkg -S {} \; 2>/dev/null | grep package-name

# Locate config/cfg files from a package
dpkg -L package-name | grep -E '\.(conf|cfg|ini|yaml|yml)$'
```

## Security & Audit

```bash
# Find SUID/SGID files installed by a package
dpkg -L package-name | xargs ls -la 2>/dev/null | grep "^...s\|^......s"

# Check for packages with security updates
apt list --upgradable 2>/dev/null | grep -i security

# Find packages with known vulnerabilities (requires debsecan)
sudo apt install debsecan
debsecan --suite $(lsb_release -cs) --format packages | head -10

# Verify package checksums with debsums
sudo apt install debsums
debsums -c package-name              # Check a specific package
debsums -c 2>/dev/null | head -20   # Check all packages (show only failures)

# Verify all installed packages and report failures
for pkg in $(dpkg -l | awk '/^ii/{print $2}'); do
  debsums -c "$pkg" 2>/dev/null || echo "Checksum failed: $pkg"
done

# Verify package signatures (requires debsig-verify)
debsig-verify package.deb

# Check only for modified non-config files (5 = checksum changed, no 'c' flag)
dpkg -V | grep "^..5" | grep -v " c "
```

## System Analysis & Reporting

```bash
# Generate full package report (name, version, arch, size)
dpkg-query -f '${Package}\t${Version}\t${Architecture}\t${Installed-Size}\n' -W | \
  sort -k4 -n | \
  awk 'BEGIN{print "Package\tVersion\tArch\tSize(KB)"}{print}' > package-report.txt

# Package count by maintainer
dpkg-query -f '${Maintainer}\n' -W | sort | uniq -c | sort -nr | head -10

# Find packages with the most files
dpkg -l | awk '/^ii/{print $2}' | \
  xargs -I {} sh -c 'echo -n "{}: "; dpkg -L {} 2>/dev/null | wc -l' | \
  sort -t: -k2 -n | tail -10

# Show disk space per package (human-readable)
dpkg-query -Wf '${Installed-Size}\t${Package}\n' | sort -n | tail -20 | \
  while read size pkg; do
    echo "$(echo "$size" | awk '{printf "%.1fMB", $1/1024}') $pkg"
  done

# Find biggest space wasters (>10MB)
dpkg-query -Wf '${Installed-Size}\t${Package}\n' | \
  awk '$1 > 10000 {printf "%s (%.0f MB)\n", $2, $1/1024}' | sort -k2 -n

# Find recently installed packages (today)
grep "$(date '+%Y-%m-%d')" /var/log/dpkg.log | grep " install "

# Package installation timeline
awk '/install/{print $1" "$2" "$4}' /var/log/dpkg.log | tail -20

# Show package installation history with sizes
grep "status installed" /var/log/dpkg.log | \
  awk '{print $1" "$2" "$4}' | \
  while read date time package; do
    size=$(dpkg-query -f '${Installed-Size}' -W "$package" 2>/dev/null || echo "0")
    echo "$date $time $package ${size}KB"
  done | tail -10

# Show packages installed on a specific date
ls -la /var/lib/dpkg/info/*.list | grep "2024-01-15"

# Find packages modified in last 24 hours
find /var/lib/dpkg/info -name "*.list" -mtime -1 -exec basename {} .list \;

# Show package installation order
grep "status installed" /var/log/dpkg.log | sort | awk '{print $4}' | nl

# Find and remove old kernel packages (keep current)
dpkg -l | grep linux-image | grep -v "$(uname -r)" | awk '{print $2}' | \
  xargs sudo apt-get remove -y

# Remove all packages with a specific prefix
dpkg -l | grep "^ii.*old-prefix" | awk '{print $2}' | xargs sudo apt-get remove -y

# Generate minimal system package list (only deps, not explicitly installed)
comm -23 <(dpkg-query -f '${Package}\n' -W | sort) <(apt-mark showmanual | sort)

# Find packages without man pages
dpkg -l | grep "^ii" | awk '{print $2}' | \
  xargs -I {} sh -c 'ls /usr/share/man/man*/{}.* 2>/dev/null || echo "No man page: {}"'
```

## Database Operations

```bash
# Backup entire dpkg database
sudo tar -czf dpkg-backup.tar.gz /var/lib/dpkg/

# Force database unlock (use only if dpkg crashed)
sudo rm -f /var/lib/dpkg/lock*
sudo rm -f /var/cache/apt/archives/lock
sudo dpkg --configure -a

# Rebuild available packages database
sudo dpkg --clear-avail
sudo dpkg --update-avail /var/lib/apt/lists/*Packages

# Check database consistency
sudo dpkg --audit

# Find packages with missing files
dpkg -l | awk '/^ii/{print $2}' | \
  xargs -I {} sh -c 'dpkg -L {} 2>/dev/null | xargs ls -l >/dev/null 2>&1 || echo "Missing files: {}"'
```

## Process & Library Correlation

```bash
# Cross-reference running processes with packages
ps aux | awk '{print $11}' | grep "^/" | sort | uniq | xargs dpkg -S 2>/dev/null

# Find libraries used by running processes and their packages
lsof | grep "\.so" | awk '{print $9}' | sort | uniq | xargs dpkg -S 2>/dev/null

# Find which package provides a specific library
dpkg -S libssl.so
ldconfig -p | grep libssl
```

## Related Tools

| Tool | Package | Purpose |
|------|---------|---------|
| `apt` / `apt-get` | apt | High-level package management with dependency resolution |
| `aptitude` | aptitude | Alternative TUI package manager |
| `deborphan` | deborphan | Find orphaned packages (installed as deps, no longer needed) |
| `apt-file` | apt-file | Search file contents across all packages (installed or not) |
| `debsums` | debsums | Verify installed file checksums against package md5sums |
| `debsecan` | debsecan | Find packages with known security vulnerabilities |
| `equivs` | equivs | Create empty/dummy packages to satisfy dependencies |
| `alien` | alien | Convert between .deb, .rpm, .tgz, and .slp formats |
| `gdebi` | gdebi-core | Install .deb files with automatic dependency resolution |
| `dpkg-repack` | dpkg-repack | Recreate .deb from an installed package |
| `apt-rdepends` | apt-rdepends | Show recursive dependency listings |

## Quick Reference

```bash
dpkg -i pkg.deb            # Install
dpkg -r pkg                # Remove (keep configs)
dpkg -P pkg                # Purge (remove everything)
dpkg -l                    # List installed
dpkg -l 'pattern*'         # List matching pattern
dpkg -s pkg                # Show status/info
dpkg -L pkg                # List files in package
dpkg -S /path/to/file      # Find owning package
dpkg -I pkg.deb            # Info from .deb file
dpkg -c pkg.deb            # Contents of .deb file
dpkg -x pkg.deb dir/       # Extract .deb to dir
dpkg --configure -a        # Fix broken installs
dpkg --verify pkg          # Verify file integrity
dpkg --audit               # Find broken packages
dpkg --get-selections      # Export package selections
dpkg --set-selections      # Import package selections
dpkg-reconfigure pkg       # Reconfigure package
dpkg-deb -b dir/ out.deb   # Build .deb from directory
update-alternatives --config name  # Choose between alternatives
```
