# AIX Boot and Init Cheatsheet

Command reference for managing the IBM AIX boot process and system initialization — inspecting and changing the boot device list (`bootlist`), rebuilding the boot image (`bosboot`), checking bootability (`ipl_varyon`), editing `/etc/inittab` (`lsitab`/`mkitab`/`chitab`/`rmitab`), controlling run levels with `init`/`telinit`, and saving the ODM to the boot logical volume (`savebase`/`restbase`).

> These commands require `root`. Boot changes (`bootlist`, `bosboot`, `savebase`) affect whether the system can start — take a [mksysb backup](articles/aix-backup-recovery-cheatsheet.md) first and double-check device names with `lsdev -Cc disk` / `lsdev -Cc tape`.

## bootlist — Boot Device Order

`bootlist` displays and sets the ordered list of devices the system tries at boot, per mode (`normal`, `service`, `prevboot`, `both`).

```sh
# List the current normal-mode bootlist
bootlist -m normal -o

# List the current service-mode bootlist
bootlist -m service -o

# Set cd0 then hdisk0 as the first and second normal boot devices
bootlist -m normal cd0 hdisk0

# Set the service-mode bootlist (e.g. CD then tape)
bootlist -m service cd0 rmt0

# Invalidate the device the system last booted from
bootlist -m prevboot

# Show boot devices with their location codes (-v = verbose)
bootlist -m normal -ov
```

| Mode (`-m`) | Purpose |
|-------------|---------|
| `normal` | Devices used during a normal boot |
| `service` | Devices used when booting into service/maintenance mode |
| `prevboot` | The device booted from last time |
| `both` | Set normal and service lists together |

## bosboot — Rebuild the Boot Image

`bosboot` creates the boot image on the boot logical volume. Run it after certain system changes (kernel, boot LV, or ODM changes) so the machine stays bootable.

```sh
# Recreate the boot image on a device (-a = create image, -d = target device)
bosboot -ad device

# Common form targeting the current IPL device
bosboot -ad /dev/ipldevice

# Create a boot image with the KDB kernel debugger enabled (-D)
bosboot -ad /dev/ipldevice -D
```

> After a successful `bosboot`, verify/adjust the bootlist with `bootlist -m normal -o`. Never interrupt `bosboot` — a partial boot image can leave the system unbootable.

## ipl_varyon — Check Disk Bootability

```sh
# Report whether disks are valid boot (IPL) devices
ipl_varyon -i
```

## /etc/inittab — Init Records

AIX manages init entries through ODM-aware commands rather than editing `/etc/inittab` by hand. Each record has an identifier, run level, action, and command.

```sh
# List records in /etc/inittab
lsitab                 # all records
lsitab -a              # (alternate listing)

# Add a record
mkitab "myid:2:once:/usr/local/bin/startup.sh"

# Change an existing record
chitab "myid:2:respawn:/usr/local/bin/startup.sh"

# Remove a record by identifier
rmitab myid
```

## init / telinit — Run Levels and Init Control

`init` (and its alias `telinit`) controls the system run level and tells init to re-read its configuration.

```sh
# Re-examine /etc/inittab (reload without changing run level)
telinit -q            # or:  telinit -Q  /  init -q

# Disable respawning — stop processes from being respawned
telinit -N

# Enter maintenance (single-user) mode
telinit -s            # equivalently: -S, -m, or -M

# Change to a numbered run level (e.g. multi-user)
telinit 2
```

| Argument | Effect |
|----------|--------|
| `-q` / `-Q` | Re-read `/etc/inittab` without changing run level |
| `-N` | Stop respawning processes |
| `-s`/`-S`/`-m`/`-M` | Enter maintenance / single-user mode |
| `0`–`9` | Switch to that run level |

## ODM Boot Image (savebase / restbase)

The Object Data Manager (ODM) device configuration is stored in the boot logical volume so devices are configured correctly early in boot.

```sh
# Save the current ODM (device config) to the boot logical volume
savebase -d /dev/hdisk0

# Restore customized device info from the boot image (boot phase 1)
restbase
```

`savebase` writes the customized ODM database to the boot LV (run it after device configuration changes so they survive a reboot). `restbase` is invoked during early boot to restore that information — you rarely run it manually.

## Quick Reference

| Task | Command |
|------|---------|
| Show normal bootlist | `bootlist -m normal -o` |
| Set normal bootlist | `bootlist -m normal cd0 hdisk0` |
| Show location codes | `bootlist -m normal -ov` |
| Rebuild boot image | `bosboot -ad /dev/ipldevice` |
| Check disk bootability | `ipl_varyon -i` |
| List inittab records | `lsitab` |
| Add / change / remove inittab record | `mkitab` / `chitab` / `rmitab` |
| Reload inittab | `telinit -q` |
| Maintenance mode | `telinit -s` |
| Save ODM to boot LV | `savebase -d /dev/hdisk0` |

For backing up and restoring the system itself, see the [AIX Backup and Recovery Cheatsheet](articles/aix-backup-recovery-cheatsheet.md).
