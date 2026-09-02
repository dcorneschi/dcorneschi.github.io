# Solaris SVR4 Package Management (pkgadd / pkgrm / pkginfo / pkgchk)

The classic System V Release 4 (SVR4) packaging tools on Oracle Solaris 10 and earlier — installing, querying, verifying, and removing software packages with `pkgadd`, `pkginfo`, `pkgchk`, and `pkgrm`. This covers the two package formats, the installed-software databases, and installing from CD/DVD, spool, data stream, and HTTP.

> Solaris 11 introduced the **Image Packaging System (IPS / `pkg`)** as the primary package manager. The SVR4 `pkg*` commands below still exist on Solaris 11 for legacy packages, but IPS is preferred there. This article focuses on the SVR4 tooling used through Solaris 10.

## Package Formats

An SVR4 package comes in one of two formats:

| Format | Structure | Typical use |
|--------|-----------|-------------|
| **File system (directory) format** | Multiple files and directories | Packages on install media (CD/DVD `Product` dir) |
| **Data stream format** | A single file | Distributable `.pkg` file (download, spool, HTTP) |

You can convert between them — e.g. spooling a directory-format package into a stream, or adding a stream to a spool directory.

## Where Solaris Records Installed Software

```bash
# Directory holding a record of every installed package
ls -l /var/sadm/pkg

# Master database: complete record of all installed files and their attributes
cat /var/sadm/install/contents
```

- `/var/sadm/pkg/` — one subdirectory per installed package (its metadata and install scripts).
- `/var/sadm/install/contents` — the system-wide file inventory used by `pkgchk` to verify integrity.

## Querying Packages (`pkginfo`)

```bash
# List installed packages, filter for one
pkginfo -l | grep wget

# Detailed info about an installed package
pkginfo -l SUNWwgetr

# Inspect a package before installing (from a spool directory)
pkginfo -d /var/spool/pkg -l SUNWwgetr

# View packages available on install media
pkginfo -d /cdrom/cdrom0/s0/Solaris_10/Product | more

# Limit output to a category (e.g. application, system)
pkginfo -c application
```

- `-l` — long/detailed listing.
- `-d <device|dir|file>` — look at packages at a source (spool dir, media, stream file) rather than the installed system.
- `-c <category>` — filter by package category.

Sample `pkginfo` output (default short form is `category  PKG  name`):

```
system      SUNWcsr    Core Solaris, (Root)
application SUNWwgetr  wget - GNU file retrieval utility (root)
system      SUNWman    On-Line Manual Pages
```

Sample `pkginfo -l SUNWwgetr` (long form):

```
   PKGINST:  SUNWwgetr
      NAME:  wget - GNU file retrieval utility (root)
  CATEGORY:  application
      ARCH:  i386
   VERSION:  11.11.0,REV=2010.01.13
    STATUS:  completely installed
```

**Naming convention:** the historical `SUNW*` prefix identifies Sun/Oracle packages (e.g. `SUNWwgetr`, `SUNWman`). A trailing letter often marks the delivery target — `r` = root filesystem components, `u` = `/usr` components. Third-party packages use their own prefixes (e.g. `SMCwget` from sunfreeware).

## Installing Packages (`pkgadd`)

```bash
# Install from install media (directory format)
pkgadd -d /cdrom/cdrom0/Solaris_10/Product SUNWman

# Copy a package into the spool directory (/var/spool/pkg) without installing
pkgadd -d /cdrom/cdrom0/Solaris_10/Product/ -s spool SUNWman

# Install every package in a data stream file
pkgadd -d /var/tmp/SUNWwgetr.pkg all

# Install a data stream package directly from a web server
pkgadd -d http://instructor/packages/SUNWrsc.pkg all
```

- `-d <source>` — the source: a directory, a device, a `.pkg` stream file, or an HTTP URL.
- `-s spool` — spool (stage) the package into `/var/spool/pkg` instead of installing it.
- `all` — install all packages contained in the source (useful for multi-package stream files).

### Inspecting a Data Stream File

```bash
# A data stream package is a single file; peek at its header
head -5 /var/tmp/stream.pkg
```

### Converting Between Formats (`pkgtrans`)

`pkgtrans` converts a directory-format package to a data-stream file and vice-versa:

```bash
# Directory format -> single data-stream file
pkgtrans /var/spool/pkg /tmp/SUNWwget.pkg SUNWwgetr

# Data-stream file -> directory format (into /var/spool/pkg)
pkgtrans /tmp/SUNWwget.pkg /var/spool/pkg SUNWwgetr
```

A data-stream file is convenient to copy or serve over HTTP; directory format is what lives in a spool directory.

### Non-Interactive Installs (admin file)

Some packages prompt during install (to run scripts as root, overwrite files, etc.). For automation, pass an **admin file** that answers those prompts:

```bash
# Use the built-in non-interactive admin profile
pkgadd -a /var/sadm/install/admin/default -d . SUNWxxx

# Or a custom admin file that auto-accepts
cat > /tmp/noask <<'EOF'
mail=
instance=overwrite
partial=nocheck
runlevel=nocheck
idepend=nocheck
rdepend=nocheck
space=nocheck
setuid=nocheck
conflict=nocheck
action=nocheck
basedir=default
EOF
pkgadd -n -a /tmp/noask -d . SUNWxxx
```

`-n` runs in non-interactive mode; the admin file's `nocheck`/`quit` keywords control how prompts are handled.

## Verifying Packages (`pkgchk`)

`pkgchk` checks installed files against the `/var/sadm/install/contents` database — useful for detecting tampering or corruption.

```bash
# Check contents and attributes of an installed package
pkgchk SUNWladm

# List the files that make up a package
pkgchk -v SUNWladm

# Check whether a specific file's contents/attributes have changed
pkgchk -p /etc/shadow

# Show package info for selected files
pkgchk -l -p /usr/bin/showrev
```

- `-v` — verbose; lists each file in the package.
- `-p <path>` — check a specific file (or files) rather than a whole package.
- `-l` — list information about the files instead of just reporting problems.

## Removing Packages (`pkgrm`)

```bash
# Remove an installed package
pkgrm SUNWfirefoxl10n-pt-BR

# Remove a package from the spool directory (not the installed system)
pkgrm -s spool SUNWman
```

`-s spool` targets the spool area (`/var/spool/pkg`) so you can clean up staged packages.

## Installing Software from CD/DVD

If the removable-media volume manager isn't presenting the disc, restart it or mount manually.

### 1. Restart the volume manager

```bash
# Solaris 9 style (init scripts)
/etc/init.d/volmgt stop
/etc/init.d/volmgt start

# Solaris 10 style (SMF)
svcadm disable volfs
svcadm enable volfs
```

### 2. Or mount the CD/DVD manually

```bash
# Identify the disc device
iostat -En

# Mount the ISO9660 (hsfs) filesystem read-only
mount -F hsfs -o ro /dev/dsk/c0t1d0p0 /cd

# Install from the media's Product directory
cd /cdrom/cdrom0/Solaris_10/Product
pkgadd -d . SUNWfirefoxl10n-pt-BR
```

- `hsfs` is the High Sierra / ISO9660 filesystem type used by CDs/DVDs.
- `pkgadd -d .` installs from the current directory (directory-format packages).

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `pkgadd` prompts block automation | Interactive scripts/conflicts | Use `-n -a <admin-file>` (see above) |
| "package already installed" | Existing instance | `instance=overwrite` in admin file, or `pkgrm` first |
| `pkgchk` reports file errors | Modified/corrupt files | Reinstall the package; investigate tampering |
| CD not visible at `/cdrom` | volfs not running | `svcadm enable volfs` or mount `hsfs` manually |
| Removed package left files | Custom postinstall data | Check the package's `/var/sadm/pkg/<PKG>` scripts |
| Dependency errors on install | Missing prerequisite package | Install the required `SUNW*` package first |

```bash
# Verify every installed package against the contents DB (integrity sweep)
pkgchk -n
```

## Command Reference

| Task | Command |
|------|---------|
| List installed packages | `pkginfo` / `pkginfo -l` |
| Package details | `pkginfo -l SUNWxxx` |
| Inspect before install | `pkginfo -d <source> -l SUNWxxx` |
| Install (directory) | `pkgadd -d /path/Product SUNWxxx` |
| Install (data stream) | `pkgadd -d file.pkg all` |
| Install (HTTP) | `pkgadd -d http://host/file.pkg all` |
| Spool a package | `pkgadd -d <source> -s spool SUNWxxx` |
| Verify a package | `pkgchk SUNWxxx` / `pkgchk -v SUNWxxx` |
| Verify a file | `pkgchk -p /path/to/file` |
| Remove a package | `pkgrm SUNWxxx` |
| Installed record | `/var/sadm/pkg`, `/var/sadm/install/contents` |

## References

- [Managing Software with Oracle Solaris (SVR4 packages)](https://docs.oracle.com/cd/E19253-01/817-1985/index.html) — official Oracle docs
- [pkgadd(1M) man page](https://docs.oracle.com/cd/E23824_01/html/821-1462/pkgadd-1m.html) — official Oracle docs
