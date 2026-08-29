# AIX Package Management Cheatsheet

Command reference for managing software on IBM AIX. AIX has two parallel worlds: **native filesets** (LPP/BFF packages managed by `installp` and reported by `lslpp`), which make up the OS itself, and **RPM/open-source packages** from the AIX Toolbox for Linux Applications (managed by `rpm` and, on modern AIX, `dnf`/`yum`). This covers both, plus interim fixes (`emgr`) and maintenance levels/service packs.

> Fileset operations require `root`. Filesets can be **applied** (reversible) or **committed** (permanent). The version scheme is `V.R.M.F` (Version.Release.Modification.Fix), and a fileset name looks like `bos.net.tcp.client`.

## Filesets vs Packages

| | Native fileset (LPP) | RPM / Toolbox |
|---|----------------------|----------------|
| Format | BFF (backup file format) | RPM |
| Install | `installp` (or `smitty install`) | `rpm`, `dnf`/`yum` |
| Query | `lslpp` | `rpm -q`, `dnf list` |
| Source | AIX base media, fixes, NIM lpp_source | AIX Toolbox repository |
| Makes up | The AIX OS and IBM products | Open-source tools (bash, python, git, etc.) |

## installp — Native Filesets

### Install and update

```sh
# Install/update from the current directory (a table of contents, .toc, must exist)
installp -aXYgd . <fileset>          # apply, expand FS (-X), auto-deps (-g), accept licenses (-Y)
installp -aXYgd /dev/cd0 all         # install everything from media
installp -aXYgd /mnt/lpp all         # from a directory of images

# Update all installed filesets from a source
install_all_updates -d /mnt/fixpack

# Preview only (no changes)
installp -paXYgd . <fileset>         # -p = preview
```

Common flags:

| Flag | Meaning |
|------|---------|
| `-a` | Apply |
| `-c` | Commit |
| `-r` | Reject (back out an applied update) |
| `-u` | Deinstall (remove) |
| `-X` | Auto-expand filesystems if space is needed |
| `-Y` | Accept new license agreements |
| `-g` | Automatically install requisite (prerequisite) filesets |
| `-d` | Device/directory source |
| `-p` | Preview only |
| `-F` | Force |

More install examples:

```sh
installp -a -d . acme.custom.net.rte           # install one fileset
installp -aY -d . acme.toolkit.rte             # install one that needs license acceptance
installp -aYXg -d . acme.datacenter.nyc.rte    # apply + auto-deps + expand FS + accept licenses
installp -acpgXYd /TL02_SP01 all               # install everything from a directory (commit + preview off)
installp -aXYgd /cdrom/usr/sys/inst.images -e /tmp/install.log all   # log to a file (-e)
```

### Inspect a source or a bff file

```sh
installp -ld .                       # list filesets in ./.toc
installp -ld /usr/sys/inst.images    # list software in the default repository
installp -L -d <file.bff>            # list filesets contained in a bff file
installp -ld <file.bff>              # same, list form
installp -Xap -d U819433.bff Java5.sdk   # install a fileset from a bff file
```

### Commit, reject, remove

```sh
installp -c all                      # commit all applied updates (make permanent)
installp -cgX all                    # commit all + remove previous versions + expand FS
installp -s                          # list applied updates that can be committed/rejected
installp -r <fileset>                # reject (undo) an applied update
installp -rg                         # reject all applied-but-not-committed updates (with deps)
installp -u <fileset>                # remove a fileset entirely (committed or broken)
installp -u -g <package>             # remove a package and its dependents
installp -upg X11*                   # preview removal of all X11 software + prerequisites
```

### Regenerate the media table of contents

```sh
inutoc .                             # (re)create ./.toc so installp can read a directory
```

### geninstall — install filesets or RPMs generically

`geninstall` is the front end that can install native filesets **and** RPMs; it passes `installp` flags through with `-I`.

```sh
geninstall -I "-acgXY" -p -d . bos.rte.install       # preview installp-style install
geninstall -I "-acgXY" -p -d /TL01_SP02 all
geninstall -I "a -cgNpQqwXY -J" -Z -p -d /tmp -f File 2>&1
geninstall -d /dev/cd0 R:cdrecord R:mtools           # install RPMs (R: prefix) from media
```

## lslpp — Query Installed Filesets

```sh
lslpp -l                             # list all installed filesets and their state
lslpp -l "bos.net.*"                 # filesets matching a pattern
lslpp -L                             # like -l with different formatting
lslpp -L bos.rte                     # check whether a fileset is installed
lslpp -Lc                            # grep/awk-friendly (colon-separated) output
lslpp -Lqc | cut -d: -f2,3,8         # one line per package: name, level, description
lslpp -h bos.rte                     # installation history of a fileset
lslpp -ha                            # installation history of all filesets
lslpp -f bos.rte.filesystem          # files a fileset installed
lslpp -0r -f bos.rte.shell           # files in the root part of a fileset
lslpp -w /usr/bin/ls                 # which fileset owns a file
lslpp -p bos.rte                     # prerequisites (requisites) of a fileset
lslpp -d bos.rte                     # dependents of a fileset
lslpp -i bos.rte                     # product information
lslpp -v                             # files missing prerequisites / not completely installed
```

Fileset states in `lslpp -l`: `COMMITTED` (permanent), `APPLIED` (reversible), `BROKEN` (failed install — needs repair).

### Related lookups

```sh
which_fileset ksh                    # which known fileset provides a command/file
inulag -l                            # list all software license agreements
ls -l /usr/sys/inst.data/sys_bundles # list available install bundles
```

## Verify and Repair

```sh
lppchk -v                            # verify all filesets are complete/consistent
lppchk -c                            # checksum-verify installed files
lppchk -l                            # verify symbolic links

# Recover from a BROKEN or interrupted install
installp -C                          # cleanup after a failed install
```

## emgr — Interim Fixes (ifixes / efixes)

Interim fixes are IBM-provided emergency/temporary patches applied on top of filesets.

```sh
emgr -l                              # list installed interim fixes
emgr -e /path/to/fix.epkg.Z          # install (preview with -p)
emgr -p -e /path/to/fix.epkg.Z       # preview install
emgr -c -L <label>                   # commit an ifix (make permanent)
emgr -r -L <label>                   # remove an ifix by label
emgr -r -n <number>                  # remove by ifix number
emgr -cn <number>                    # check an ifix
epkg perf                            # create an efix (ifix) package in interactive mode
```

## RPM and the AIX Toolbox

The AIX Toolbox for Linux Applications ships open-source software as RPMs. Modern AIX (7.2 TL4+/7.3) includes **`dnf`** (with `yum` as an alias) for dependency-resolved installs from the online Toolbox repo.

### dnf / yum

```sh
dnf install git                      # install with dependency resolution
dnf remove git
dnf update                           # update all Toolbox packages
dnf search python
dnf list installed
dnf info git
dnf repolist                         # configured repositories
```

`dnf` reads its repo config from `/opt/freeware/etc/yum/yum.conf` (or `/etc/yum.conf`); the Toolbox tools live under `/opt/freeware`.

### rpm (low level)

```sh
rpm -qa                              # all installed RPMs
rpm -q git                           # is a package installed?
rpm -qi git                          # package info
rpm -ql git                          # files in a package
rpm -qf /opt/freeware/bin/git        # which package owns a file
rpm -ivh package.rpm                 # install (verbose, progress)
rpm -Uvh package.rpm                 # upgrade
rpm -e git                           # erase/remove
```

> Prefer `dnf`/`yum` over bare `rpm -ivh` for Toolbox software — `rpm` alone won't resolve dependencies, which are common in open-source packages.

## Maintenance Levels and Service Packs

AIX releases are patched in **Technology Levels (TL)** and **Service Packs (SP)**.

```sh
oslevel                              # base OS version (e.g. 7200-05)
oslevel -s                           # full TL/SP level (e.g. 7200-05-03-2148)
oslevel -g                           # filesets above the current OS level
oslevel -r                           # highest recommended maintenance level reached
oslevel -rq                          # list all known technology levels on the system
oslevel -sq                          # list known service packs on the system

# What's missing to reach a given level?
oslevel -rl 5100-04                  # fileset updates missing for that ML
oslevel -sl 5300-09-02-0849          # filesets below the given service pack
oslevel -sg 5300-09-01-0847          # filesets greater than the given service pack

# Report the current fix/ML state
instfix -i | grep ML                 # which maintenance levels are installed
instfix -i -k <APAR>                 # is a specific APAR installed?
instfix -icqk 7200-05_AIX_ML | grep ':-:'   # missing filesets for a level

# Compare installed software to IBM's latest-fix report
compare_report -s -r /tmp/LatestFixData52 -l
```

## Managing Install Media (bffcreate / lppmgr)

`bffcreate` extracts/creates installable BFF images from media; `lppmgr` prunes duplicate and superseded updates from an lpp_source or image directory.

```sh
# bffcreate — build/inspect installable images
bffcreate -ld <dir>                  # show packages in a dir (with full U#### names)
bffcreate -c -d /bb1                 # rename U#### bff to its real name (overwrites original!)
bffcreate -v -d /software -t /export/softs/tsm all   # extract packages to another location

# lppmgr — clean up an image/lpp_source directory
lppmgr -d /images -u                 # list duplicate and conflicting updates
lppmgr -d /images -u -r              # remove duplicate and conflicting updates
/usr/lib/instl/lppmgr -d ppc -u -m /software/lpp -V   # move unused files aside
```

## Extract Files from a BFF Package

A `.bff`/`.I` fileset image is a `backup`-format archive, so `restore` can list and extract its contents (paths are relative).

```sh
restore -Tqvf openssh.base.5.4.0.6100.I                       # list all files
restore -xqvf openssh.base.5.4.0.6100.I                       # extract everything
restore -xqvf openssh.base.5.4.0.6100.I ./usr/lpp/openssh.base/inst_root/liblpp.a   # one file
```

## Install / Configuration Assistants

```sh
configassist            # Configuration Assistant wizard (graphical)
install_assist          # Installation Assistant (ASCII)
```

## SMIT Fast Paths

```sh
smitty install           # software installation and maintenance menu
smitty install_latest    # install/update software (installp)
smitty update_all        # update all installed software to the latest level
smitty commit            # commit applied updates
smitty reject            # reject/remove applied filesets
smitty remove            # remove installed software
smitty software_maintain # maintenance functions (reject, commit, ...)
smitty list_installed    # list installed software
smitty install_software  # top-level software install menu
```

## Quick Reference

| Task | Command |
|------|---------|
| Install a fileset | `installp -aXYgd . <fileset>` |
| Update all filesets | `install_all_updates -d <source>` |
| Commit applied updates | `installp -c all` |
| Remove a fileset | `installp -u <fileset>` |
| List installed filesets | `lslpp -l` |
| Which fileset owns a file | `lslpp -w <file>` |
| Verify filesets | `lppchk -v` |
| List interim fixes | `emgr -l` |
| Install an ifix | `emgr -e <fix>.epkg.Z` |
| Install open-source pkg | `dnf install <pkg>` |
| Which RPM owns a file | `rpm -qf <file>` |
| Full OS level | `oslevel -s` |
| Is an APAR installed? | `instfix -i -k <APAR>` |

## Related

- [AIX NIM Cheatsheet](articles/aix-nim-cheatsheet.md) — lpp_source/SPOT resources and network-based installs/updates
- [AIX SMIT Cheatsheet](articles/aix-smit-cheatsheet.md) — the `smitty install` menus behind these commands
- [AIX Backup and Recovery Cheatsheet](articles/aix-backup-recovery-cheatsheet.md) — mksysb before major software changes
