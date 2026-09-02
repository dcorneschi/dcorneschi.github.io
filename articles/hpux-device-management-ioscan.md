# HP-UX Device Management (ioscan, scsimgr, DSFs)

Inspecting and managing hardware and storage on HP-UX — the hardware-addressing schemes, the **agile view** (persistent) vs **legacy** device model, `ioscan` for discovery and health, `scsimgr` for LUN/lunpath management, and the device special file (DSF) tools `insf`/`mksf`/`mknod`/`rmsf`/`lssf`. Covers HP-UX 11i v1–v3.

## Hardware Addressing

### Parallel (legacy) HBA address

```
1/0/0/2/0
│ │ │ │ └─ device/function
│ │ │ └─── LBA (Local Bus Adapter)
│ │ └───── SBA (System Bus Adapter)
│ └─────── (bus)
└───────── Cell
```

### Agile-view FC lunpath address

```
1/0/2/1/0 . 0x<64-bit> . 0x<64-bit>
│           │            └─ LUN address
│           └────────────── WW Port Name (target)
└────────────────────────── HBA hardware address
```

### Agile-view FC LUN hardware path

```
64000 / 0xfa00 / 0x4
│       │        └─ virtual LUN ID
│       └────────── virtual bus
└────────────────── virtual root node
```

The **agile view** (11i v3) gives each LUN a single persistent "LUN hardware path" (`64000/...`) regardless of how many physical paths (lunpaths) reach it — so a device file no longer changes when cabling or paths change. The **legacy view** uses the old `cXtXdX` per-path addressing.

### Why the Agile View Exists

Under the legacy model, a disk's name encoded the exact physical route to it: `c5t0d1` meant "controller instance 5, SCSI target 0, LUN 1." That worked for direct-attached SCSI but broke down with Fibre Channel SAN storage, where the *same* LUN is deliberately reached through several HBAs and fabric paths for redundancy. Legacy naming gave you a *different* `cXtXdX` name for every path to the same disk, and those names could change if you recabled, replaced an HBA, or the SAN re-presented the LUN — a fragile situation for `/etc/fstab`, LVM, and scripts.

The agile view fixes this by separating three distinct concepts:

- **LUN hardware path** (`64000/0xfa00/0x4`) — a *virtual*, persistent identity for the storage volume itself. The `64000` virtual root and `0xfa00` virtual bus are not real hardware; they are stable handles that never change for the life of the LUN.
- **Lunpath** (`1/0/2/1/0.0x<wwpn>.0x<lun>`) — one physical route to that LUN, expressed as HBA hardware path + target World-Wide Port Name + LUN address. A LUN typically has several lunpaths.
- **Legacy path** (`cXtXdX`) — retained for compatibility so old tools and configs keep working.

Native multipathing binds all the lunpaths of a LUN under its single agile DSF and handles failover/load-balancing transparently. This is why the agile DSFs live in `/dev/disk/diskN` and `/dev/rdisk/diskN` (one name per LUN) rather than `/dev/dsk/cXtXdX` (one name per path). See [HP-UX Fibre Channel and SAN](articles/hpux-fibre-channel-san.md) for how LUNs are presented, and [HP-UX LVM](articles/hpux-lvm.md) for building volume groups on agile DSFs.

## ioscan — Discovery

`ioscan` has two fundamentally different modes, and choosing the right one matters on a production system:

- **Probing mode** (no `-k`) physically walks the I/O buses and interrogates each device. It discovers newly attached hardware but briefly quiesces the bus and can disturb sensitive devices, so avoid it casually on a busy SAN-attached host.
- **Kernel-cache mode** (`-k`) reads the I/O tree the kernel already built, returning instantly and touching no hardware. Use `-k` for routine inventory; reserve a bare probe for when you have genuinely just changed hardware.

The common flags compose: `-f` gives a full multi-column listing, `-n` adds the device special file names, `-N` selects the agile (new) view, `-C class` filters by device class, and `-H path` restricts to one hardware path.

```bash
ioscan                      # all components
ioscan -f                   # full listing
ioscan -kf                  # full listing from kernel cache (fast, no probe)
ioscan -fn                  # full listing including device file names
ioscan -kfH 0/0/0/3/0       # full listing at a specific hardware path

# Agile view
ioscan -kfN -C disk         # LUN hardware paths (agile), disk class
ioscan -kfNH 64000/0xfa00/0x4   # cached listing of one agile device
ioscan -kfNH 1/0/2/1/0      # lunpaths serviced by that HBA
```

Filter by device class with `-C`:

```bash
ioscan -C cell              # cell boards
ioscan -C lan               # LAN interfaces
ioscan -C disk              # disks
ioscan -C fc                # fibre channel interfaces
ioscan -C ext_bus           # SCSI buses
ioscan -C processor         # processors
ioscan -C tty               # serial/teletype
```

### LUN and Path Mapping

```bash
ioscan -m lun                          # LUNs and their lunpaths
ioscan -m lun -H 64000/0xfa00/0x10     # one LUN by hardware path
ioscan -m lun -D /dev/disk/disk30      # one LUN by device file

# Map persistent (agile) DSFs to legacy DSFs and back
ioscan -m dsf                          # all persistent ↔ legacy mappings
ioscan -m dsf /dev/dsk/c5t0d1          # a legacy DSF → its persistent DSF
ioscan -m dsf /dev/disk/disk30         # a persistent DSF → its legacy DSFs

# Map LUN hwpath ↔ lunpath hwpath ↔ legacy hwpath
ioscan -m hwpath
```

### Health Status

```bash
ioscan -P health -C disk                       # health of all disks/LUNs
ioscan -P health -C lunpath                    # health of each path
ioscan -P health -C fc                         # all FC adapters
ioscan -P health -H 64000/0xfa00/0x4           # a specific LUN/adapter
ioscan -P health -H 1/0/2/1/0                  # an FC adapter and its lunpaths
```

**LUN health states:**

| State | Meaning |
|-------|---------|
| `online` | All paths usable (not necessarily all active — active/passive has one active, others standby) |
| `offline` | The LUN isn't accessible on any path |
| `limited` | Not all paths are accessible |
| `disabled` | The LUN was suspended due to an error |

**Lunpath health states:**

| State | Meaning |
|-------|---------|
| `online` / `offline` | Path usable / not usable |
| `unusable` | Lunpath authentication failed |
| `disabled` | Path suspended because of an error |
| `standby` | Path for LUNs reached via an active/passive policy |

## scsimgr — LUN and Lunpath Management

```bash
# WWIDs and LUN IDs
scsimgr get_attr -a wwid all_lun                    # all LUN WWIDs
scsimgr get_attr -a wwid -H 64000/0xfa00/0x4        # one LUN's WWID by hwpath
scsimgr get_attr -a wwid -D /dev/rdisk/disk30       # by device file
scsimgr get_attr -a lunid -H 1/0/2/1/0.0x...        # LUN ID on a lunpath

# Attributes (multiple -a)
scsimgr get_attr -D /dev/rdisk/disk22 -a wwid -a serial_number

# Info and statistics
scsimgr get_info -H 0/0/10/0/0.0x5006016030201d6b.0x4000000000000000   # lunpath info
scsimgr get_info -H 64000/0xfa00/0x9                # LUN info
scsimgr get_stat -H 64000/0xfa00/0x9                # LUN statistics
scsimgr get_stat -D /dev/rdisk/disk22               # stats by device file

# List all lunpaths of a LUN
scsimgr lun_map -D /dev/rdisk/disk22
```

### Disabling / Re-enabling a Path

```bash
# Disable a lunpath (force with -f)
scsimgr -f disable -H 1/0/2/1/0.0x50001fe15003112c.0x4001000000000000

# Confirm the path state
ioscan -P health -H 1/0/2/1/0.0x50001fe15003112c.0x4001000000000000

# Re-enable it
scsimgr enable -H 1/0/2/1/0.0x50001fe15003112c.0x4001000000000000
```

Disabling a lunpath is useful before storage maintenance (e.g. taking one fabric down) so I/O fails over to the remaining paths cleanly. Manually disabling the path you are about to disturb makes the failover deterministic and keeps error-log noise down, rather than letting the stack discover the loss through timeouts.

> **Gotcha:** Never disable the *last* usable path to a LUN that is carrying I/O — that turns the volume offline and can hang or fault applications and LVM. Confirm at least one other path is `online` (via `ioscan -P health`) before disabling one. Disabling a lunpath is not persistent across reboot; the kernel re-enables paths on the next boot/probe unless the underlying hardware is genuinely gone.

### Reading the ioscan S/W State

In a full `ioscan -fn` listing, each device shows a software state that tells you whether the driver successfully bound to it:

| S/W state | Meaning |
|-----------|---------|
| `CLAIMED` | A driver is bound and the device is usable |
| `UNCLAIMED` | Hardware was found but no driver claimed it (missing driver or firmware issue) |
| `NO_HW` | The device once existed but no hardware currently responds at that path (stale) |
| `DISABLED` | The device/interface was administratively disabled |
| `DIFF_HW` | Different hardware is now present at a path that previously held something else |

`NO_HW` on a disk after you removed a LUN is exactly the stale entry the cleanup workflow below removes.

## Device Special Files (DSFs)

A device special file is the `/dev` entry through which programs reach a device driver; it is not the device itself but a named handle carrying a **major number** (which driver) and **minor number** (which instance/options). HP-UX provides each device in two flavors:

- **Block DSFs** (`/dev/dsk/...`, `/dev/disk/...`) buffer I/O and are used for mounted filesystems and LVM.
- **Character (raw) DSFs** (`/dev/rdsk/...`, `/dev/rdisk/...`) do unbuffered I/O and are used by tools like `dd`, `fsck`, `pvcreate`, and backup utilities.

Under the agile view, disks appear as `/dev/disk/diskN` (block) and `/dev/rdisk/diskN` (raw); the legacy equivalents are `/dev/dsk/cXtXdX` and `/dev/rdsk/cXtXdX`. Most DSFs are created automatically by `insf` when hardware is discovered, so you rarely call `mksf`/`mknod` by hand.

```bash
# Drivers configured in the kernel
lsdev

# Slot addresses/properties (11i v1)
rad -q

# Create DSFs
insf -v                        # for newly added devices
insf -v -e                     # new devices + recreate missing DSFs for existing ones
insf -v -e -H 64000/0xfa00/0x0 # for a specific hardware path
insf -v -e -C estape           # both legacy and persistent DSFs for a class
mksf                           # non-default DSFs for auto-configurable devices
mknod                          # custom DSFs for non-auto-configurable devices

# Inspect DSFs
lssf /dev/dsk/c2t2d0           # a device file's characteristics
lssf -s                        # list DSFs for stale (non-existent) devices (11i v3)

# Remove DSFs
rmsf -v /dev/disk/disk1                 # a specific DSF
rmsf -v -a /dev/disk/disk1              # all DSFs for the device + its definition
rmsf -v -H 64000/0xfa00/0x1             # by hardware path
rmsf -v -x                              # remove DSFs for stale devices (11i v3)
```

### Legacy Mode (agile vs legacy DSFs)

11i v3 uses the agile (persistent) naming by default but can also maintain legacy `cXtXdX` DSFs:

```bash
insf -v -L                     # is legacy mode currently enabled?
rmsf -v -L                     # disable legacy mode and remove legacy DSFs
insf -L                        # re-enable legacy mode and recreate legacy DSFs
```

## Typical Workflows

Bring a newly presented LUN online:

```bash
ioscan -kfN -C disk            # rescan / confirm the LUN appears
insf -v -e                     # create its DSFs
ioscan -m lun                  # verify LUN and its lunpaths
ioscan -P health -C disk       # confirm health
scsimgr get_attr -a wwid all_lun   # correlate to the array's WWID
```

Clean up after removing a LUN (11i v3):

```bash
lssf -s                        # find stale DSFs
rmsf -v -x                     # remove them
```

## Command Reference

| Task | Command |
|------|---------|
| Full device list (cached) | `ioscan -kf` |
| Agile LUN paths | `ioscan -kfN -C disk` |
| LUNs + lunpaths | `ioscan -m lun` |
| Map persistent ↔ legacy DSF | `ioscan -m dsf [dsf]` |
| Path/LUN health | `ioscan -P health -C disk\|lunpath\|fc` |
| LUN WWID | `scsimgr get_attr -a wwid -D <dsf>` |
| Lunpath info/stats | `scsimgr get_info` / `get_stat -H <path>` |
| List a LUN's paths | `scsimgr lun_map -D <dsf>` |
| Disable / enable path | `scsimgr -f disable` / `scsimgr enable -H <path>` |
| Create DSFs | `insf -v -e` (`-H`/`-C` to scope) |
| Inspect a DSF | `lssf <dsf>` |
| Remove a DSF | `rmsf -v [-a] <dsf>` / `-H <path>` |
| Stale DSFs (v3) | `lssf -s` / `rmsf -v -x` |
| Legacy mode | `insf -v -L` / `rmsf -v -L` / `insf -L` |
| Kernel drivers | `lsdev` |
| Full listing with DSF names | `ioscan -fn` |
| LUN ↔ lunpath ↔ legacy hwpath map | `ioscan -m hwpath` |
| Disable / enable a lunpath | `scsimgr -f disable` / `scsimgr enable -H <path>` |

## Troubleshooting

**A newly presented LUN doesn't appear.** Run `ioscan -kfN -C disk` first (cached); if it's still missing, run a probing `ioscan -fnC disk` to force discovery, then `insf -v -e` to create DSFs. If it appears `UNCLAIMED`, a driver or firmware issue is likely; if it never appears, check the fabric/zoning and array LUN masking (see [HP-UX Fibre Channel and SAN](articles/hpux-fibre-channel-san.md)).

**A device shows as `NO_HW` after removing storage.** That is a stale entry for hardware that no longer responds. On 11i v3, find stale DSFs with `lssf -s` and remove them with `rmsf -v -x`. Do this only after the LUN is truly gone and removed from any volume group.

**`ioscan` output disagrees with reality.** If you ran with `-k`, you saw the cached tree. Re-run without `-k` to force a fresh probe. Conversely, if a bare `ioscan` is slow or disruptive on a large SAN host, use `-k` for routine checks.

**A path shows `standby` and I expected `online`.** The array is active/passive and that path leads to the passive controller — it only carries I/O after failover. This is normal, not an error. `unusable` (authentication failure) is the state that warrants investigation.

**`insf` didn't create a legacy `cXtXdX` name.** 11i v3 defaults to agile naming. Confirm legacy mode with `insf -v -L`; enable it with `insf -L` if a legacy application still needs `cXtXdX` DSFs.

## Related Articles

- [HP-UX Fibre Channel and SAN](articles/hpux-fibre-channel-san.md)
- [HP-UX LVM](articles/hpux-lvm.md)
- [HP-UX System Information and Initial Configuration](articles/hpux-system-information.md)
- [HP-UX Startup, Run Levels, and Network Services](articles/hpux-startup-and-services.md)
- [HP-UX Kernel Configuration](articles/hpux-kernel-configuration.md)
