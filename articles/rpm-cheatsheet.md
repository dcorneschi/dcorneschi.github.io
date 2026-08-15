# RPM Cheatsheet

RPM (Red Hat Package Manager) is the low-level package management tool on RHEL, CentOS, Fedora, and SUSE. Unlike `dnf`/`yum`, RPM does not resolve dependencies automatically — it operates directly on `.rpm` files and the local RPM database.

## Quick Reference

| Action | Command |
|--------|---------|
| Install a package | `rpm -ivh package.rpm` |
| Upgrade a package | `rpm -Uvh package.rpm` |
| Remove a package | `rpm -e package_name` |
| Query if installed | `rpm -q package_name` |
| List all installed | `rpm -qa` |
| List files in package | `rpm -ql package_name` |
| Which package owns a file | `rpm -qf /path/to/file` |
| Show package info | `rpm -qi package_name` |
| Verify a package | `rpm -V package_name` |
| Verify all packages | `rpm -Va` |

---

## Install, Upgrade, and Remove

### Install

```bash
# Install a package (verbose, with progress bar)
rpm -ivh package.rpm

# Install multiple packages
rpm -ivh package1.rpm package2.rpm

# Install even if already installed (replace existing)
rpm -ivh --replacepkgs package.rpm

# Install even if files conflict with other installed packages
rpm -ivh --replacefiles package.rpm

# Install without dependency check (use with caution)
rpm -ivh --nodeps package.rpm

# Install ignoring conflicts
rpm -ivh --force package.rpm

# Install from URL
rpm -ivh https://example.com/package.rpm

# Test install (dry run — don't actually install)
rpm -ivh --test package.rpm
```

### Upgrade and Freshen

```bash
# Upgrade (install if not present, upgrade if older version exists)
rpm -Uvh package.rpm

# Freshen (upgrade only if an older version is already installed)
rpm -Fvh package.rpm

# Upgrade without dependency check
rpm -Uvh --nodeps package.rpm

# Downgrade a package
rpm -Uvh --oldpackage package-older-version.rpm
```

| Flag | Behavior |
|------|----------|
| `-U` (upgrade) | Installs if new, upgrades if older version exists |
| `-F` (freshen) | Only upgrades if package is already installed |
| `-i` (install) | Installs; fails if package already installed |

### Remove

```bash
# Remove a package
rpm -e package_name

# Remove without dependency check
rpm -e --nodeps package_name

# Test removal (dry run)
rpm -e --test package_name

# Remove multiple packages
rpm -e package1 package2

# Remove with verbose output
rpm -evh package_name
```

---

## Common Flags

| Flag | Description |
|------|-------------|
| `-v` | Verbose output |
| `-h` | Print hash marks for progress |
| `--nodeps` | Skip dependency checks |
| `--force` | Force install (overwrite files, ignore conflicts) |
| `--test` | Dry run — don't actually modify the system |
| `--oldpackage` | Allow downgrade to an older version |
| `--noscripts` | Don't execute pre/post install scripts |
| `--nosignature` | Don't verify package signatures |
| `--nodigest` | Don't verify package digests |
| `--replacepkgs` | Reinstall even if already installed |
| `--replacefiles` | Install even if files conflict with other packages |
| `--prefix` | Relocate to a different path (if package supports it) |
| `--excludedocs` | Don't install documentation files |

---

## Query Packages

### Query Installed Packages

```bash
# Check if a package is installed
rpm -q httpd

# List all installed packages
rpm -qa

# List all installed packages sorted by name
rpm -qa | sort

# List all installed packages sorted by install date
rpm -qa --last

# Search installed packages by pattern
rpm -qa | grep php
rpm -qa "php*"

# List packages in a specific group
rpm -qg "Applications/System"

# Count installed packages
rpm -qa | wc -l
```

### Package Information

```bash
# Show package description and details
rpm -qi httpd

# Show package description from an RPM file (not installed)
rpm -qip package.rpm

# Show only the package version
rpm -q --queryformat '%{VERSION}-%{RELEASE}\n' httpd

# Show install date
rpm -qi httpd | grep "Install Date"

# Show package architecture
rpm -q --queryformat '%{ARCH}\n' httpd
```

### List Files

```bash
# List all files in an installed package
rpm -ql httpd

# List files from an RPM file (not installed)
rpm -qlp package.rpm

# List only configuration files
rpm -qc httpd

# List only documentation files
rpm -qd httpd

# List files with attributes (permissions, owner, size)
rpm -qlv httpd

# Show the state of each file
rpm -qs httpd
```

### Find Which Package Owns a File

```bash
# Find which package provides a file
rpm -qf /etc/httpd/conf/httpd.conf

# Find which package provides a command
rpm -qf $(which curl)

# Show package info for the owner of a file
rpm -qif /usr/bin/vim
```

### Dependencies

```bash
# List what a package requires
rpm -qR httpd
rpm -q --requires httpd

# List what a package provides
rpm -q --provides httpd

# List requirements from an RPM file
rpm -qRp package.rpm

# List what requires a specific package (reverse dependency)
rpm -q --whatrequires libcurl

# List what provides a specific capability
rpm -q --whatprovides "libc.so.6()(64bit)"
```

### Scripts

```bash
# Show pre/post install/uninstall scripts
rpm -q --scripts httpd

# Show scripts from an RPM file
rpm -qp --scripts package.rpm

# List scripts for all installed packages
rpm -qa --queryformat "\n\nPACKAGE: %{name}\n" --scripts

# Show triggers
rpm -q --triggers httpd
```

### Changelog

```bash
# Show full changelog
rpm -q --changelog httpd

# Show last 10 changelog entries
rpm -q --changelog httpd | head -40
```

---

## Custom Query Formats

The `--queryformat` flag allows custom output using package tags:

```bash
# Package name and version
rpm -qa --queryformat '%{NAME}-%{VERSION}-%{RELEASE}.%{ARCH}\n'

# Show install date with package name
rpm -qa --queryformat '%{INSTALLTIME:date} %{NAME}-%{VERSION}\n' | sort

# Show package size (bytes)
rpm -qa --queryformat '%{SIZE}\t%{NAME}\n' | sort -rn | head -20

# Show vendor
rpm -qa --queryformat '%{VENDOR}\t%{NAME}\n' | sort | head

# Show source RPM
rpm -q --queryformat '%{SOURCERPM}\n' httpd

# Show group
rpm -q --queryformat '%{GROUP}\n' httpd

# Show packager
rpm -q --queryformat '%{PACKAGER}\n' httpd

# Show build date
rpm -q --queryformat '%{BUILDTIME:date}\n' httpd

# Show license
rpm -q --queryformat '%{LICENSE}\n' httpd

# List all available tags
rpm --querytags
```

### Commonly Used Tags

| Tag | Description |
|-----|-------------|
| `%{NAME}` | Package name |
| `%{VERSION}` | Version |
| `%{RELEASE}` | Release |
| `%{ARCH}` | Architecture |
| `%{SIZE}` | Installed size (bytes) |
| `%{INSTALLTIME}` | Install timestamp |
| `%{BUILDTIME}` | Build timestamp |
| `%{VENDOR}` | Vendor |
| `%{LICENSE}` | License |
| `%{SOURCERPM}` | Source RPM name |
| `%{SUMMARY}` | One-line description |
| `%{DESCRIPTION}` | Full description |
| `%{URL}` | Project URL |
| `%{GROUP}` | Package group |
| `%{PACKAGER}` | Packager name |
| `%{SIGPGP}` | PGP signature |

---

## Verify Packages

`rpm -V` compares installed files against the RPM database. It reports differences in size, mode, digest, owner, group, modification time, and capabilities.

```bash
# Verify a single package
rpm -V httpd

# Verify all installed packages
rpm -Va

# Verify with verbose output
rpm -Vv httpd

# Verify a package from an RPM file
rpm -Vp package.rpm

# Verify only file attributes (skip digest)
rpm -V --nodigest httpd
```

### Verification Output Codes

Each character position represents a specific check:

```
S.5....T.  c /etc/httpd/conf/httpd.conf
```

| Position | Code | Meaning |
|----------|------|---------|
| 1 | `S` | File Size differs |
| 2 | `M` | Mode (permissions) differs |
| 3 | `5` | MD5/SHA digest differs |
| 4 | `D` | Device major/minor number mismatch |
| 5 | `L` | readLink path mismatch (symlink) |
| 6 | `U` | User ownership differs |
| 7 | `G` | Group ownership differs |
| 8 | `T` | Modification Time differs |
| 9 | `P` | caP abilities differ |

A `.` means the check passed. A `?` means the test couldn't be performed.

### File Types in Verify Output

| Code | Type |
|------|------|
| `c` | Configuration file |
| `d` | Documentation file |
| `g` | Ghost file (not in package payload) |
| `l` | License file |
| `r` | Readme file |

### Practical Examples

```bash
# Find all packages with modified config files
rpm -Va | grep "^..5.*c "

# Find all packages with changed permissions
rpm -Va | grep "^.M"

# Check if a specific binary was tampered with
rpm -Vf /usr/bin/ssh
```

---

### Restore Original Configuration Files

If a config file has been modified or corrupted, restore it by reinstalling the owning package:

```bash
# 1. Find which package owns the file
rpm -qf /etc/sysctl.conf

# 2. Verify the file is modified
rpm -V initscripts

# 3. Back up and reinstall
mv /etc/sysctl.conf /etc/sysctl.conf.bak
dnf reinstall initscripts
```

---

## GPG Signatures

```bash
# Check the signature of an RPM file
rpm --checksig package.rpm
rpm -K package.rpm

# Import a GPG key
rpm --import https://example.com/RPM-GPG-KEY

# Import Red Hat GPG key
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

# List all imported GPG keys
rpm -qa gpg-pubkey*

# Show details of an imported key
rpm -qi gpg-pubkey-fd431d51-4ae0493b

# Remove an imported key
rpm -e gpg-pubkey-fd431d51-4ae0493b

# Install without GPG check
rpm -ivh --nosignature package.rpm
```

---

## RPM Database

The RPM database is stored in `/var/lib/rpm/`.

```bash
# Rebuild the RPM database
rpm --rebuilddb

# Initialize a new database (rarely needed)
rpm --initdb

# Show database statistics
rpm --dbstat

# Back up the RPM database
tar czf /root/rpm-db-backup.tar.gz /var/lib/rpm/
```

### When to Rebuild

Rebuild when you see errors like:
- `rpmdb: Thread/process failed: Thread died in Berkeley DB library`
- `error: db5 error(11) from dbenv->open: Resource temporarily unavailable`
- `error: cannot open Packages index using db5`

```bash
# Safe rebuild procedure
mkdir /root/rpm-backup
cp -a /var/lib/rpm/ /root/rpm-backup/
rpm --rebuilddb
```

---

## Extract Files from an RPM (Without Installing)

```bash
# Extract all files to current directory
rpm2cpio package.rpm | cpio -idmv

# Extract a specific file
rpm2cpio package.rpm | cpio -idmv ./etc/httpd/conf/httpd.conf

# List contents without extracting
rpm2cpio package.rpm | cpio -t

# Extract to a specific directory
mkdir /tmp/extracted
cd /tmp/extracted && rpm2cpio /path/to/package.rpm | cpio -idmv

# Convert RPM to tar.gz (newer rpm versions)
rpm2archive package.rpm
```

---

## Building RPMs

```bash
# Install build dependencies for a source RPM
dnf builddep package.src.rpm

# Install a source RPM (extracts to ~/rpmbuild/)
rpm -ivh package.src.rpm

# Build a binary RPM from spec file
rpmbuild -bb ~/rpmbuild/SPECS/package.spec

# Build source + binary RPM
rpmbuild -ba ~/rpmbuild/SPECS/package.spec

# Build from source RPM
rpmbuild --rebuild package.src.rpm

# Show the rpmbuild directory structure
rpmbuild --showrc | grep _topdir
```

### rpmbuild Directory Structure

| Directory | Purpose |
|-----------|---------|
| `~/rpmbuild/SOURCES/` | Source tarballs and patches |
| `~/rpmbuild/SPECS/` | Spec files |
| `~/rpmbuild/BUILD/` | Build working directory |
| `~/rpmbuild/RPMS/` | Built binary RPMs |
| `~/rpmbuild/SRPMS/` | Built source RPMs |

---

## Useful One-Liners

```bash
# 10 largest installed packages
rpm -qa --queryformat '%{SIZE}\t%{NAME}-%{VERSION}\n' | sort -rn | head -10

# List packages installed in the last 7 days
rpm -qa --last | head -20

# Find all packages from a specific vendor
rpm -qa --queryformat '%{VENDOR}\t%{NAME}\n' | grep "Red Hat"

# List packages that have no dependencies
rpm -qa --queryformat '%{NAME}: [%{REQUIRES}\n]\n' | grep "^.*: \[\]"

# Find packages with modified files
rpm -Va 2>/dev/null | awk '{print $NF}' | xargs rpm -qf | sort -u

# Compare packages between two systems
rpm -qa --queryformat '%{NAME}\n' | sort > /tmp/system-packages.txt
# Then diff with the other system's list

# Show total disk usage of all RPMs
rpm -qa --queryformat '%{SIZE}\n' | awk '{s+=$1} END {printf "%.2f GB\n", s/1024/1024/1024}'

# List all packages without documentation installed
rpm -qa --queryformat '%{NAME}: %{INSTALLFLAGS}\n' | grep excludedocs

# Find packages installed outside of yum/dnf (manual rpm -i)
rpm -qa --queryformat '%{NAME}-%{VERSION}-%{RELEASE} %{INSTALLTIME:date}\n' | sort -k2

# Export list of all installed packages (for migration)
rpm -qa --queryformat '%{NAME}\n' | sort > installed-packages.txt

# Full NVRA listing (useful for comparing systems)
rpm -qa --queryformat '%{NAME}-%{VERSION}-%{RELEASE}.%{ARCH}\n' | sort > packages-nvra.txt

# Check if a specific package is GPG signed
rpm -qi httpd | grep Signature

# Find all setuid/setgid files from RPMs
rpm -Va | grep '^......[SG]'
```

---

## rpm vs dnf/yum

| Feature | rpm | dnf/yum |
|---------|-----|---------|
| Dependency resolution | No | Yes |
| Repository access | No (local files only) | Yes |
| Transaction history | No | Yes |
| Rollback | No | Yes |
| Install from repo | No | Yes |
| Query installed packages | Yes | Yes |
| Verify package integrity | Yes | Limited |
| Signature verification | Yes | Yes |
| Database management | Yes | No |
| Extract without install | Yes (`rpm2cpio`) | No |
| Custom query formats | Yes (`--queryformat`) | Limited |

**Use `rpm` when:**
- Querying package details, files, or dependencies
- Verifying package integrity
- Installing a local `.rpm` without repo access
- Extracting files without installing
- Rebuilding the package database
- Checking GPG signatures

**Use `dnf`/`yum` when:**
- Installing packages from repositories
- Resolving dependencies automatically
- Performing upgrades and updates
- Rolling back transactions
