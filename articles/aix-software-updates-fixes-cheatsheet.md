# AIX Software Updates and Fixes Cheatsheet

Command reference for patching IBM AIX — applying updates and Technology Level / Service Pack upgrades with `smitty update_all` and `install_all_updates`, managing interim fixes (ifixes) with `emgr`, and finding/installing individual APAR fixes with `instfix`. This focuses on the update *workflow*; for the underlying fileset mechanics (`installp`/`lslpp`, apply vs commit), see the [AIX Package Management Cheatsheet](articles/aix-package-management-cheatsheet.md).

> Take a `mksysb` before a TL/SP upgrade. Apply updates first (reversible), verify the system, then commit. See [AIX Backup and Recovery](articles/aix-backup-recovery-cheatsheet.md).

## Update All via SMIT

`smitty update_all` is the menu front end for updating every installed fileset to the level found in a source directory/media.

```sh
smitty update_all
```

Typical answers for a staged (reversible) update:

```
Commit software updates?  no      # apply only, so you can reject/roll back
Save replaced files?      yes     # keep the previous versions for rollback
Accept new license agreements?  yes
```

Leaving updates **applied** (not committed) lets you `smitty reject` them if something misbehaves; commit later once the system is proven healthy.

## install_all_updates

`install_all_updates` updates installp filesets (and installable RPMs) from a source. By default it **applies** (not commits), **expands** filesystems, and **installs requisites**, and it runs a recommended TL/SP verification at the end.

```sh
install_all_updates -d /cdrom -Y            # update from /dev/cd0, accept all licenses
install_all_updates -d /images -rc          # commit installp updates + install RPM updates (-r)
install_all_updates -pcYd .                 # PREVIEW a commit-style update from current dir
install_all_updates -cYd .                  # actually perform it (commit)
cat /var/adm/ras/install_all_updates.log    # log file
```

| Flag | Effect |
|------|--------|
| `-d` | Source device/directory |
| `-c` | Commit the software (default is apply only) |
| `-x` | Do **not** expand filesystems (default expands) |
| `-n` | Do **not** install requisites (default installs) |
| `-p` | Preview only |
| `-s` | Skip the recommended ML/TL verification at the end |
| `-Y` | Accept all license agreements |
| `-r` | Also install installable RPM updates |

## Technology Level / Service Pack Upgrades

For a TL upgrade, **update the installer filesets first**, then run the full update. This avoids the new packages hitting an old `bos.rte.install`.

```sh
cd /path/to/TL

# 1. Upgrade the installer (bos.rte.install) first, so it can process the new packages.
#    IBM-recommended way — install the install utilities only:
install_all_updates -i -d .
#    (equivalent manual form, APPLY only so it's reversible:)
installp -agX -d . bos.rte.install

# 2. Preview the full update
install_all_updates -pcYd .          # or: smitty update_all

# 3. Apply the full update (leave uncommitted), verify, then commit later
```

> `install_all_updates` already detects an update to `bos.rte.install`, installs it first, and re-invokes itself — the `-i` flag makes it do *only* the installer update, which is useful for getting an accurate preview.

If the installer upgrade causes trouble, roll it back:

```sh
smitty reject              # reject the applied bos.rte.install update
```

After the update completes, confirm the level:

```sh
oslevel -s                 # full TL-SP level
instfix -i | grep ML       # installed maintenance levels
```

## Interim Fixes (emgr)

Interim fixes (ifixes/efixes, `*.epkg.Z`) are IBM emergency patches applied on top of filesets.

```sh
# Install
emgr -X -e IZ12345.epkg.Z      # install a concurrent ifix, expand filesystems if needed
emgr -i IZ12345.epkg.Z         # install in memory only (not reloaded after reboot)
emgr -e IZ12345.epkg.Z         # standard install (preview with -p)

# Commit (persist)
emgr -C -L IZ12345             # commit an in-memory fix to disk
emgr -Ci IZ12345.epkg.Z        # install in memory AND commit to disk in one step

# List / check
emgr -l                        # list all installed ifixes
emgr -l -L IZ12345             # info about a specific ifix
emgr -c -L <ifix_label>        # check that an ifix is installed correctly

# Remove
emgr -r -L IZ12345             # deinstall an ifix by label
```

| Flag | Meaning |
|------|---------|
| `-e` | Install (efix package) |
| `-i` | Install in memory only |
| `-X` | Expand filesystems if needed |
| `-C` | Commit to disk |
| `-c` | Check an installed ifix |
| `-l` | List ifixes |
| `-L <label>` | Target a specific ifix |
| `-r` | Remove |

## Finding and Installing Fixes (instfix)

`instfix` locates and installs fixes by **APAR** number or **keyword** (including ML/TL keywords), and reports what's installed or missing.

```sh
instfix -i | grep ML                 # installed maintenance levels
instfix -ik IY58143                  # is APAR IY58143 installed?
instfix -k IY73748 -d /dev/cd0       # install a specific APAR from media
instfix -ik IZ82381 -v               # detail each fileset tied to a fix/keyword
instfix -T -d .                      # list all fixes present on the media/source

# Which filesets are missing for a partly installed ML/TL?
instfix -ciqk 6100-01_AIX_ML | grep ':-:'
```

| Flag | Meaning |
|------|---------|
| `-i` | Report installed fixes (query mode) |
| `-k <keyword>` | Fix keyword or APAR number |
| `-d <source>` | Device/directory to install from |
| `-c` | Colon-separated output (parseable) |
| `-q` | Quiet |
| `-v` | Verbose (per-fileset detail) |
| `-T` | List fixes available on the media |

## Quick Reference

| Task | Command |
|------|---------|
| Update all (menu) | `smitty update_all` |
| Update all from media | `install_all_updates -d /cdrom -Y` |
| Preview a commit update | `install_all_updates -pcYd .` |
| Commit update from dir | `install_all_updates -cYd .` |
| Upgrade the installer first | `installp -agX -d . bos.rte.install` |
| Roll back an applied update | `smitty reject` |
| Install an ifix (+expand) | `emgr -X -e <fix>.epkg.Z` |
| List ifixes | `emgr -l` |
| Remove an ifix | `emgr -r -L <label>` |
| Is an APAR installed? | `instfix -ik <APAR>` |
| Install an APAR from media | `instfix -k <APAR> -d /dev/cd0` |
| Missing filesets for an ML | `instfix -ciqk <ML>_AIX_ML \| grep ':-:'` |
| Confirm level | `oslevel -s` |

## Related

- [AIX Package Management Cheatsheet](articles/aix-package-management-cheatsheet.md) — installp/lslpp fileset mechanics, apply/commit/reject, lppchk
- [AIX NIM Cheatsheet](articles/aix-nim-cheatsheet.md) — network-based updates via lpp_source/SPOT and nimadm migration
- [AIX Backup and Recovery Cheatsheet](articles/aix-backup-recovery-cheatsheet.md) — mksysb before a TL/SP upgrade
