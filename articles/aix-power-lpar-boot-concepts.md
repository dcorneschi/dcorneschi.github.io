# AIX Power Systems, LPAR, and Boot Concepts

Reference notes for IBM Power Systems running AIX — firmware/edition naming, logical partition (LPAR) sizing rules for CPU and memory, the AIX boot process internals (BLV, `bosboot`, `rc.boot` phases), device states, physical location codes, single-user and maintenance mode recovery, and firmware update commands.

> This is a concepts-plus-commands reference drawn from IBM Power5/6/7-era documentation. Ratios, partition maximums, and minimums are model- and firmware-dependent — always confirm against the specific system's documentation before sizing partitions or updating firmware.

## Firmware and Model Naming

### POWER5

- **SF** stands for "Squadron Firmware".

### POWER6

| Code | Segment |
|------|---------|
| `EH` | Enterprise High-End |
| `EM` | Enterprise Mid-Range (formerly Intermediate-High) |
| `EL` | Enterprise Low-End |

### POWER7

| Code | Models |
|------|--------|
| `AL` | 750, 755 |
| `AM` | 770, 780 |
| `AH` | 775, 790, 795 (ultra high-end) |

## AIX Editions

AIX ships in three editions: **Express**, **Standard**, and **Enterprise**.

```sh
# Show / manage the installed edition
/usr/sbin/chedition -l
```

## Base OS Memory Requirements

| AIX Version | Minimum RAM to install BOS |
|-------------|----------------------------|
| AIX 5L V5.2 / V5.3 | 128 MB |
| AIX 6.1 | 256 MB |

As of AIX 6.1 the 32-bit kernel is deprecated, so 64-bit hardware is required to run AIX 6.1 (POWER4, POWER5, or POWER6 systems only).

## LPAR Sizing

### Minimum partition configuration

- 0.1 processing units if shared; 1 processor if dedicated
- 128 MB memory
- Access to necessary I/O devices — adapter for the boot disk and network adapters

### Smallest granularity for adding resources

- 0.01 processing units if shared; 1 processor if dedicated
- 1 LMB (16–256 MB) of memory
- One I/O slot

### Partition maximums

The maximum number of partitions depends on the system model and available resources. The software maximum is **254 partitions**.

| System | Max partitions | Processors |
|--------|----------------|-----------|
| IBM Power 550 | 40 | 4 |
| IBM Power 570 | 160 | 16 |
| IBM Power 595 | 254 | 64 |

## Memory Allocation

Memory is allocated in **Logical Memory Blocks (LMB)** sized 16 MB to 256 MB. Each partition profile configures three values:

| Setting | Meaning |
|---------|---------|
| **Minimum** | Partition will not start if this amount is unavailable; can be decreased to this via dynamic LPAR |
| **Desired** | Amount used at activation if available |
| **Maximum** | Ceiling the partition can be grown to via dynamic LPAR; used for sizing the partition's page table |

Platform minimums:

- **POWER4** — smallest partition memory is 256 MB (256 MB LMB); pre-October 2002 firmware requires a 1 GB minimum.
- **POWER5 / POWER6** — support a 128 MB minimum partition.

### Min/Max memory ratios

Valid ranges are constrained by a ratio between the minimum and maximum settings:

- Partitions with **< 256 MB**: maximum cannot exceed **16×** the actual memory size.
- Partitions with **>= 256 MB**: maximum cannot exceed **64×** the actual memory size.

| Minimum | Maximum | Ratio |
|---------|---------|-------|
| 128 MB | 2 GB | 1:16 |
| 256 MB | 16 GB | 1:64 |
| 512 MB | 32 GB | 1:64 |
| 768 MB | 48 GB | 1:64 |
| 1 GB | 64 GB | 1:64 |

### Processor sharing check boxes

Partition profile properties include **Allow when partition is inactive** and **Allow when partition is active**:

- First box selected: when the partition is shut down, its processors return to the shared pool.
- Second box selected: the active partition can cede idle CPU cycles to the Shared Processor Pool (POWER6-based systems only).

```sh
# View OS, partition ID, and partition name
uname -Ls
lparstat -i
```

## Virtual I/O Adapters

Set the **Maximum virtual adapters** field higher than the number you expect to configure — even if none are configured now — because this value **cannot be changed dynamically**.

## CPU

### Shared vs dedicated processors

Dedicated processors are physical processors allocated to and reserved for a single partition. On POWER5, other partitions never use time slices on them while that partition is active. On POWER6, a dedicated-processor partition can donate unused capacity to a shared pool.

Shared processors are drawn from a shared pool. POWER5 has a single shared pool; POWER6 supports multiple shared pools. Shared-pool partitions (SPLPAR) consume processing units from the pool as needed within configuration limits.

POWER4 servers may have deconfigured and Capacity on Demand (CoD) processors; processors are dedicated when allocated. Shared and virtual processors were introduced with POWER5 and enhanced on POWER6.

### Virtual processors

- By default, one virtual processor is allocated per 1.00 processing unit (or part thereof). Example: 3.6 processing units → 4 virtual processors.
- Up to 10 virtual processors can be assigned per processing unit. Example: 3.6 processing units → up to 36 virtual processors.
- Both entitled capacity and the number of virtual processors can be changed dynamically for tuning.
- Maximum virtual processors per partition is **64**.

### Simultaneous multi-threading (SMT)

Enabling SMT makes the OS create two logical processors for each virtual or physical processor. SMT is enabled by default and supported on AIX V5.3, AIX V6.1, and Linux.

```sh
# Query / change SMT dynamically
smtctl
```

You can also change SMT through the SMIT menus.

### Shared Dedicated Capacity (POWER6)

Dedicated processors can be shared only while idle:

- The processors stay committed to the dedicated partition, which can reclaim them at any time.
- Only **uncapped** partitions can use the donated idle cycles.
- Sharing happens only when the dedicated partition has spare cycles **and** uncapped partitions have consumed all of their entitled capacity — the dedicated partition keeps absolute priority.

### Processor utilization threshold

The `schedo` parameter `ded_cpu_donate_thresh` controls donation. If a dedicated processor's utilization is **below** the threshold, its idle cycles are donated; if **equal to or above** it, they are not. By default the threshold is 80%, so the remaining 20% of idle cycles are not ceded to the Shared Processor Pool.

## AIX Boot Process

System firmware (also called Read Only Storage, **ROS**) initializes the hardware, reads the boot list to locate the boot image, and loads it into memory. During a normal boot the image is usually on a hard disk; it can also be on tape or CD-ROM (maintenance/service mode) or loaded over the network via NIM. Press the appropriate function keys during boot to select an alternate boot list.

### Boot Logical Volume (BLV)

The **BLV** is a logical volume on the boot disk that contains the boot image. It is part of `rootvg` with a logical-volume type of `boot`. On RSPC and CHRP platforms an additional **SOFTROS** program (resident in the BLV) performs initialization not provided by the hardware ROS.

The boot record is a 512-byte block holding the size and location of the boot image; ROS reads it to locate the BLV. ROS (RS6K) or SOFTROS (RSPC/CHRP) loads `bootexpand`, the compressed kernel, the compressed RAM file system, and the reduced ODM into memory. `bootexpand` uncompresses the kernel and RAM file system, then passes control to the kernel. Compression roughly halves the boot image size, saving space and load time; an uncompressed BLV (without `bootexpand`) can also be created.

The kernel loaded from the BLV is never replaced during boot and is the same kernel used in multiuser mode. A copy also exists on disk as `/unix`, but that copy is only referenced by `ps` — to use a new kernel you must re-create the BLV.

The BLV also contains a **reduced copy of the ODM**, because many devices are configured before `hd4` (rootvg) is available.

### bosboot — create/copy the boot image

`bosboot` creates the boot image from disk files, copies it to the BLV, and updates the boot record. The BLV is first created during installation and may be re-created during a BOS upgrade. Re-create it manually when:

- The BLV is corrupted and the system won't boot.
- A new kernel is needed (the kernel loads from the BLV).
- You want to enable kernel debug features or change performance-related variables.

`bosboot` performs these functions:

- Creates the base ODM from the full ODM in the root file system.
- Creates the RAM file system from root file system files.
- Compresses the kernel and RAM file system.
- Copies the base ODM, RAM file system, kernel, `bootexpand`, and SOFTROS (CHRP/RSPC) to the boot device.
- Updates the boot record if the boot device is a disk (calls `mkboot`).

For disk images the image goes into the BLV on the specified boot disk; for CD/tape it goes at the start of the media. `bosboot` selects files via prototype (`.proto`) files named for the architecture and boot-image type:

| Prototype file | Purpose |
|----------------|---------|
| `/usr/lib/boot/chrp.disk.proto` | Disk boot image, CHRP |
| `/usr/lib/boot/chrp.cd.proto` | CD boot image, CHRP |
| `/usr/lib/boot/rspc.tape.proto` | Tape boot image, RSPC |

By default `bosboot` uses `/unix` for the kernel. On CHRP/RSPC it copies the matching SOFTROS:

- `/usr/lib/boot/aixmon.chrp` — SOFTROS for CHRP
- `/usr/lib/boot/aixmon_rspc` — SOFTROS for RSPC

### Kernel initialization and rc.boot phases

After the boot loader loads it into memory, the kernel initializes, mounts the RAM file system, and starts `/etc/init`. The RAM file system version of `init` (`/usr/lib/boot/ssh` on the boot disk root file system) drives the early phases. `rc.boot` is the script that controls AIX initialization:

| Phase | Action |
|-------|--------|
| **rc.boot 1** | RAM `init` configures base devices needed to access the rootvg file systems |
| **rc.boot 2** | RAM `init` activates rootvg and mounts `/`, `/usr`, `/var` |
| *(switch)* | Kernel restarts `init` from the disk-based root file system; `init` reads `/etc/inittab` |
| **rc.boot 3** | `init` runs the `sysinit` action `brc::sysinit:/sbin/rc.boot 3`; `rc.boot 3` configures the remaining devices |

`init` then continues through `/etc/inittab` to complete the transition to multiuser mode. AIX can boot from hard disk, CD-ROM, tape, or a network server.

## Device States

| State | Meaning |
|-------|---------|
| **Undefined** | Supported but not configured; not in the customized database |
| **Defined** | In the customized database with a logical name, location code, and attributes — but not yet usable |
| **Available** | In the customized database, fully configured, ready for use |

A newly identified device is configured to **Available**. If a previously configured device is powered off before a reboot, it appears as **Defined** — the system expects it but can't use it. A device may also stay Defined if it was physically removed without running `rmdev -dl <name>` to clear it from the ODM. All devices are self-configuring except parallel and serial devices (printers, ASCII terminals), because `cfgmgr` runs during boot.

## Physical Location Codes

Assigned by system firmware to uniquely identify hardware for assigning adapters to LPARs and identifying Field Replaceable Units (FRU).

```
<enclosure>-<planar>-<slot>-<port>-<logical location>
```

The enclosure is usually `<machine type>.<model>.<serial#>`. Example: `U787A.001.DNZ0713-P1-C3`. Location codes are displayed by default with `lscfg`.

## Booting AIX in Single-User Mode

Single-user mode is less common on AIX because many repairs need `rootvg` unmounted. But when a filesystem or VG problem exists **and** the system still boots from rootvg, single-user mode has advantages: it boots faster than maintenance mode, needs no install media or NIM SPOT, and gives access to all the commands you'd normally have in multiuser.

### Standalone system (no HMC)

1. Boot with no media in the CD/DVD drive.
2. Wait until the "choose another boot list" prompt appears and you hear beeps on the console.
3. Press **6** to start diagnostics.

### System using an HMC

1. Select the LPAR in the HMC GUI.
2. Choose **Operations → Activate**.
3. In the Activate window click **Advanced**.
4. Change **Boot mode** to **Diagnostic with stored boot list**.
5. Click **OK** to save, then **OK** again to activate.

```sh
# Switch to run level 2 (multiuser) once repairs are done
telinit 2
```

See IBM's guide: [Booting AIX in Single-User Mode](https://www.ibm.com/support/pages/booting-aix-single-user-mode).

## Maintenance Mode

When accessing rootvg in maintenance mode you select the volume group that is rootvg. The menu shows only volume group **IDs**, not names, and may list multiple VGs — identify the correct one by **PVID, VGID, or SCSI ID** rather than the physical volume name. After selecting a VG, the list of its logical volumes is shown so you can confirm it is rootvg. Two options are offered:

- **Access this Volume Group and start a shell**
- **Access this Volume Group and start a shell before mounting file systems**

### Access before mounting file systems

This activates rootvg but does **not** mount its file systems — the typical choice for repairing a corrupted file system with `fsck`, which requires the file system be unmounted.

A corrupted `hd8` transaction log is another case. Changes to the superblock or i-nodes are recorded in the log logical volume; a corrupted log must be reinitialized with `logform` (only possible when no file system is mounted), followed by an `fsck` of every file system that uses that log. From AIX 5L V5.1 you must specify the file system type (`jfs` or `jfs2`):

```sh
logform -V jfs /dev/hd8
fsck -y -V jfs /dev/hd1
fsck -y -V jfs /dev/hd2
fsck -y -V jfs /dev/hd3
fsck -y -V jfs /dev/hd4
fsck -y -V jfs /dev/hd9var
fsck -y -V jfs /dev/hd10opt
```

## Firmware / Microcode Commands

```sh
# Display microcode for all supported devices
lsmcode -A

# Show firmware level for the system / processor
lsmcode -c

# Show which firmware (microcode) should be updated
invscout

# Check the platform firmware level
lscfg -vp | grep Platform

# Copy new firmware from CD-ROM into a working directory
cp /mnt/cdrom/microcode/... /tmp/fwupdate

# Verify the firmware image with a checksum
sum vvYYMMDD.img

# Perform the update
cd /usr/lpp/diagnostics/bin && ./update_flash -f /tmp/fwupdate/3R041029.img
```

## Quick Reference

| Task | Command |
|------|---------|
| Show/manage AIX edition | `/usr/sbin/chedition -l` |
| OS / partition ID / name | `uname -Ls` |
| Partition resource info | `lparstat -i` |
| Query/change SMT | `smtctl` |
| Re-create the boot image | `bosboot -ad /dev/ipldevice` |
| Remove a device from the ODM | `rmdev -dl <name>` |
| Show location codes | `lscfg -vp` |
| Enter run level 2 | `telinit 2` |
| Microcode for all devices | `lsmcode -A` |
| System/processor firmware | `lsmcode -c` |
| Firmware update advisor | `invscout` |
| Flash firmware | `./update_flash -f <image>` |

## Related

- [AIX / Power Service Processor and ASMI](articles/aix-service-processor-asmi.md) — the service processor that initializes hardware before the boot process begins.
- [AIX System Dump and Core File Cheatsheet](articles/aix-system-dump-core-cheatsheet.md) — kernel dumps, KDB, and the boot debug options mentioned here.
- [AIX MPIO and Fibre Channel Cheatsheet](articles/aix-mpio-fibre-channel-cheatsheet.md) — SAN boot disks and the FC/MPIO paths behind the BLV.
