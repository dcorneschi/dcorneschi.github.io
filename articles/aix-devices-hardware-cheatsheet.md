# AIX Devices and Hardware Cheatsheet

Command reference for inspecting and managing hardware on IBM AIX — listing devices and their parent/child relationships (`lsdev`/`lsparent`), reading configuration and Vital Product Data (`lscfg`), querying system parameters (`getconf`/`prtconf`/`bootinfo`), locating adapters in slots (`lsslot`), and common device tasks like assigning PVIDs and reading HBA WWNs.

> Device data comes from the [ODM](articles/aix-odm-cheatsheet.md); `lsdev`/`lsattr` read it and `chdev`/`rmdev` write it. Most `ls*` commands are read-only and safe; `rmdev`/`chdev` change the system.

## Listing Devices (lsdev)

The `-C` flag reads the **Customized** database (devices actually on this system); `-c` filters by class, `-p` by parent, `-l` by name.

```sh
lsdev                          # all devices
lsdev -C -r class              # all customized device classes
lsdev -Cc processor            # physical/virtual processors
lsdev -Cc disk                 # all disks
lsdev -Cc adapter              # all adapters
lsdev -Cl hdisk3               # a specific device from the customized DB
lsdev -Cc disk -p scsi0        # disks attached to scsi0
lsdev -Cp scsi0                # all child devices of scsi0
lsdev -Cp scsi0 -l hdisk0      # test if hdisk0 is a child of scsi0
```

| Flag | Meaning |
|------|---------|
| `-C` | Read the Customized (configured) database |
| `-c <class>` | Filter by device class (disk, adapter, processor, ...) |
| `-p <parent>` | Filter by parent device |
| `-l <name>` | A specific device |
| `-r` | Report a column of values (e.g. `-r class`) |
| `-F <field>` | Output only the named attribute field(s) |

## Parent / Child Relationships

```sh
lsdev -l hdisk0 -F parent      # the parent of a device
lsparent -Cl hdisk0            # possible parent devices for hdisk0
lsdev -Cp scsi0                # children of scsi0
```

## Configuration and VPD (lscfg)

```sh
lscfg                          # location codes of all devices
lscfg -l ent0                  # location of one device
lscfg -vl ent0                 # Vital Product Data (VPD) for a device
lscfg -vpl fcs0                # VPD incl. platform-specific details

# Read an HBA's World Wide Name (WWPN)
lscfg -vl fcs0 | grep 'Network Address'
```

> From the SMS menu you can also read WWPNs without booting the OS: **SMS → 8 (Open Firmware Prompt) → `ioinfo` → 6 → 1**.

## System Parameters (getconf / prtconf / bootinfo)

### getconf

```sh
getconf -a                     # all system configuration variables
getconf REAL_MEMORY            # physical memory (bytes)
getconf DISK_SIZE /dev/hdisk0  # disk size (MB)
getconf DISK_DEVNAME hdisk1    # device address of a disk
getconf KERNEL_BITMODE         # 32/64-bit kernel
```

### prtconf

```sh
prtconf                        # full system summary (model, memory, CPU, firmware, adapters)
prtconf -L                     # LPAR ID
prtconf -m                     # memory size
prtconf -s                     # CPU speed
prtconf -c                     # CPU type
```

### bootinfo

```sh
bootinfo -p                    # hardware platform (e.g. chrp)
bootinfo -K                    # kernel bit mode (32/64)
bootinfo -s hdisk1             # size of a disk (MB)
bootinfo -b                    # last device the system booted from
bootinfo -r                    # memory (KB, i.e. bytes / 1024)
bootinfo -B hdisk0             # 1 if the disk is bootable, else 0
```

## Adapter Slots (lsslot)

```sh
lsslot -c pci                  # all PCI slots
lsslot -c pci -a               # available (empty) PCI slots
lsslot -c pci -l ent0          # the slot holding a specific adapter
lsslot -c slot                 # all slots (physical + virtual)
```

## Common Device Tasks

### PVID assignment

```sh
chdev -l hdisk1 -a pv=yes      # write a PVID to a disk
chdev -l hdisk1 -a pv=clear    # clear the PVID
```

### Attributes and configuration

```sh
lsattr -El hdisk0              # current attribute values of a device
lsattr -Rl hdisk0 -a queue_depth   # allowed range for one attribute
chdev -l hdisk0 -a queue_depth=32  # change an attribute (device must be available or use -P)
chdev -l hdisk0 -a queue_depth=32 -P   # defer until reboot
cfgmgr                         # detect and configure newly attached devices
rmdev -dl hdisk3               # remove (delete) a device definition
rmdev -l hdisk3 -R             # recursively unconfigure a device and its children
```

### Fibre Channel

```sh
fcstat fcs0                    # FC adapter statistics and extended info
lsdev -Cc adapter | grep fcs   # list FC adapters
lscfg -vl fcs0 | grep 'Network Address'   # WWPN
```

## Quick Reference

| Task | Command |
|------|---------|
| All devices | `lsdev` |
| Disks / adapters | `lsdev -Cc disk` / `lsdev -Cc adapter` |
| A device's parent | `lsdev -l hdisk0 -F parent` |
| Children of an adapter | `lsdev -Cp scsi0` |
| Device VPD | `lscfg -vl <dev>` |
| HBA WWPN | `lscfg -vl fcs0 \| grep 'Network Address'` |
| Memory size | `getconf REAL_MEMORY` / `prtconf -m` |
| Disk size | `getconf DISK_SIZE /dev/hdisk0` / `bootinfo -s <disk>` |
| LPAR ID | `prtconf -L` |
| Is a disk bootable? | `bootinfo -B <disk>` |
| PCI slots | `lsslot -c pci` |
| Assign / clear PVID | `chdev -l <disk> -a pv=yes\|clear` |
| Detect new devices | `cfgmgr` |
| Remove a device | `rmdev -dl <dev>` |
| FC adapter stats | `fcstat fcs0` |

## Related

- [AIX ODM Cheatsheet](articles/aix-odm-cheatsheet.md) — the database behind lsdev/lsattr/chdev
- [AIX LVM Cheatsheet](articles/aix-lvm-cheatsheet.md) — disks, PVIDs, and volume groups
- [AIX VIOS Cheatsheet](articles/aix-vios-cheatsheet.md) — virtual adapters, NPIV, and FC tuning
- [AIX Boot and Init Cheatsheet](articles/aix-boot-init-cheatsheet.md) — bootlist and boot device checks
