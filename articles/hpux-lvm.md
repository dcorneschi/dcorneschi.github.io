# HP-UX LVM (Logical Volume Manager)

Managing storage with the Logical Volume Manager on HP-UX — the on-disk data structures, LVM versions and their limits, and the full workflow for physical volumes, volume groups, logical volumes, mirroring, configuration backup/restore, and boot-disk recovery. Covers HP-UX 11i v1–v3 (LVM 1.0 and 2.x).

## Why LVM Exists

Before LVM, a filesystem was tied to a single fixed disk partition: if you outgrew it you had to back up, repartition, and restore. LVM inserts a layer of abstraction between physical disks and the filesystems that use them. Disks become **physical volumes (PVs)**, PVs are pooled into a **volume group (VG)**, and from that pool you carve **logical volumes (LVs)** that behave like virtual disks. Because an LV is just a mapping of logical extents onto physical extents that can live on any PV in the VG, you can grow an LV, span it across several spindles, or mirror it across disks without touching the data layout the filesystem sees. This is the foundation for online storage growth, boot-disk mirroring, and clean disk replacement on HP-UX.

The core hierarchy is worth fixing in your head before the commands make sense:

- **Physical Volume (PV)** — a disk (or LUN) initialized with `pvcreate`.
- **Volume Group (VG)** — a pool of one or more PVs created with `vgcreate`.
- **Physical Extent (PE)** — the fixed-size allocation unit a VG carves each PV into.
- **Logical Extent (LE)** — a slot in a logical volume; each LE maps to one PE (or to two/three PEs when mirrored).
- **Logical Volume (LV)** — a sequence of LEs presented as a block/raw device you put a filesystem, swap, or raw database on.

## LVM On-Disk Data Structures

When a disk is initialized as a physical volume (PV) and brought into a volume group (VG), LVM writes several reserved structures at the start of the disk:

| Structure | Purpose | Created by |
|-----------|---------|-----------|
| **PVRA** (Physical Volume Reserved Area) | LVM info specific to the PV; also holds a copy of the VGRA | `pvcreate` |
| **VGRA** (Volume Group Reserved Area) | Contains VGSA + VGDA; maps logical extents (LE) to physical extents (PE); one copy per PV | `vgcreate` |
| **VGSA** (Volume Group Status Area) | Quorum information for the VG | (within VGRA) |
| **VGDA** (Volume Group Descriptor Area) | Config the device driver needs to assemble the VG | (within VGRA) |
| **BDRA** (Boot Disk Reserved Area) | Only on bootable LVM disks | `pvcreate -B` |
| **BBRA** (Bad Block Relocation Area) | Bad-block recovery info; created with the first LV. Deprecated in 11i v3 (no effect), not created on LVM 2.x | (LV creation) |

The LVM header is ~2912 KB on bootable LVM disks; on non-bootable disks it depends on the parameters given at VG creation. The remaining space is divided into **physical extents (PE)** used for filesystems, raw partitions, or swap.

Why so many reserved areas? LVM is designed so that any single surviving PV carries enough metadata to reconstruct the group. The **VGDA** is the map — it records every LV and the LE→PE assignments — and a copy lives on *each* PV, so losing a disk doesn't lose the layout knowledge. The **VGSA** tracks quorum and which mirror copies are current versus stale, which is how LVM knows, after a disk comes back, which extents to resync. The **BDRA** and boot **LIF** area exist only on bootable disks because firmware needs a fixed place to find the loader and the list of boot/root/swap/dump LVs — this is why you can't simply `pvcreate` a data disk and expect to boot from it; it needs `pvcreate -B` plus `mkboot` and `lvlnboot`. The practical upshot: **always keep a current `vgcfgbackup`**, because that saved copy of the VGDA is what `vgcfgrestore` uses to rebuild a replacement disk's metadata after a failure.

## LVM Versions and Limits

LVM has four versions: **1.0, 2.0, 2.1, 2.2**.

| Release | LVM versions available |
|---------|------------------------|
| HP-UX 11i v1 and v2 | LVM 1.0 only |
| HP-UX 11i v3 | all LVM versions (1.0 and 2.x) |

Key limits:

| Attribute | LVM 1.0 (11i v1/v2) | LVM 2.x (11i v3) |
|-----------|---------------------|-------------------|
| Extent (PE/LE) size | 1–256 MB (default 4 MB) | must be specified by the admin |
| Max PE per PV | 1016 | not fixed — admin sets max expected VG size |
| Max VGs | 10 (`maxvgs`, `0x00`–`0x09`) | 256 by default (`maxvgs` gone) |
| Mirroring | 3-way | 6-way (VxVM: up to 32) |
| Device major number | 64 | 128 |

```bash
# Determine the LVM version (if lvmadm doesn't exist, the system is LVM 1.0 only)
lvmadm -t

# Current value of the maxvgs kernel parameter (11i v1/v2)
kctune maxvgs
```

### Choosing an Extent Size

The extent size is the single most consequential decision at VG creation because on LVM 1.0 it **cannot be changed afterward** — changing it means rebuilding the VG. The trade-off is between granularity and reach. A small extent (say 4 MB) wastes little space when LVs are sized in odd amounts, but on LVM 1.0 the hard cap of **1016 PEs per PV** means a 4 MB extent limits each PV to roughly 4 GB. To bring a larger modern disk fully under LVM 1.0 you must either raise the max-PE limit with `-e` at creation (up to 65535) or use a bigger extent. A common practice on large 1.0 VGs was 8, 16, or 32 MB extents so a single big LUN fit within the PE ceiling. LVM 2.x removes the fixed 1016-PE cap and instead asks you to declare the **maximum expected VG size** up front, from which it derives the extent granularity — so on 11i v3 you plan by capacity rather than by extent count.

### Why maxvgs Matters on 11i v1/v2

Under LVM 1.0 the VG's minor number encodes the VG index, and the kernel only pre-allocates structures for `maxvgs` volume groups (default 10, `0x00`–`0x09`). If you already have ten VGs and try to create an eleventh, `vgcreate` fails until you raise `maxvgs` and rebuild the kernel. LVM 2.x drops this constraint (256 VGs by default), which is one of the practical reasons to migrate large environments to 11i v3.

## Configuration Files

```
/etc/lvmtab      # which disks belong to which VGs — LVM 1.0
/etc/lvmtab_p    # same, for LVM 2.x
/etc/lvmconf/    # per-VG configuration backups (vgcfgbackup output)
```

Find disks already in LVM VGs:

```bash
# 11i v1/v2
strings /etc/lvmtab* | grep /dev

# 11i v3
lvmadm -l
```

## Preparing Disks

```bash
# Show disks
ls -l /dev/dsk

# Determine disk size (KB); -b prints in KB directly
diskinfo /dev/rdsk/c0t2d0
diskinfo -b /dev/rdisk/disk5

# Format and verify a disk before bringing it under LVM
mediainit /dev/rdisk/disk22

# Wipe an existing PV/LVM header (destructive)
dd if=/dev/zero of=/dev/rdisk/disk1 bs=1024 count=1024
```

## Physical Volumes (pvcreate)

```bash
# Initialize a disk as a physical volume
pvcreate /dev/rdisk/disk1

# Force (disk was previously part of another VG)
pvcreate -f /dev/rdisk/disk1

# Bootable PV (adds the BDRA)
pvcreate -B /dev/rdisk/disk1
```

> **Gotcha — raw vs block device files.** `pvcreate` (and `mediainit`, `diskinfo`, `newfs`) operate on the **character/raw** device (`/dev/rdsk/...` or `/dev/rdisk/...`), while `vgcreate`/`vgextend` and mounts use the **block** device (`/dev/dsk/...` or `/dev/disk/...`). Passing the wrong one is a classic first-time error. On 11i v3, `/dev/disk`/`/dev/rdisk` are the **persistent** (agile) device special files that survive path and controller changes; `/dev/dsk`/`/dev/rdsk` are the older **legacy** DSFs tied to a specific hardware path. Prefer persistent DSFs on v3 so multipathing and hardware moves don't break your VG.

Confirm a disk really became a PV, and see which VG (if any) claims it:

```bash
pvdisplay /dev/disk/disk1            # "VG Name" is blank until vgcreate/vgextend
pvdisplay -v /dev/disk/disk1 | more  # per-extent (LE→PE) detail
```

## Volume Groups

### Create an LVM 1.0 VG (manual device-file setup)

On 11i v1/v2 you create the VG's group device file by hand; on 11i v3 (0803+) `vgcreate` can do it automatically.

```bash
mkdir /dev/vg01
mknod /dev/vg01/group c 64 0x010000     # major 64 = LVM 1.0; minor is the VG number
chown root:sys /dev/vg01/group
chmod 640 /dev/vg01/group
vgcreate vg01 /dev/disk/disk1 /dev/disk/disk2
```

`vgcreate` attributes (LVM 1.0):

| Option | Meaning | Default / Max |
|--------|---------|---------------|
| `-l` | Max logical volumes in the VG | default 255 |
| `-p` | Max physical volumes in the VG | default 16, max 255 |
| `-s` | Extent size (PE/LE), MB | default 4, max 256 |
| `-e` | Max physical extents per PV | default 1016, max 65535 |

> On 11i v1 these attributes **cannot** be changed after creation. On 11i v2/v3, all LVM 1.0 attributes except extent size can be changed later with `vgmodify`.

### Create an LVM 2.x VG

```bash
# Version 2.1, max VG size 1 TB, 4 MB extents
vgcreate -V 2.1 -S 1t -s 4 vg01 /dev/disk/disk1 /dev/disk/disk2
```

| Option | Meaning |
|--------|---------|
| `-V` | LVM version (1.0, 2.0, 2.1, 2.2) |
| `-S` | Max VG size with unit (m/g/t/p); `m` if no unit |
| `-s` | Extent size (max 256 MB) |

Plan sizing before creating:

```bash
# Possible extent sizes for a given max VG size
vgcreate -V 2.1 -E -S 1p

# Max VG size achievable with a given extent size
vgcreate -V 2.1 -E -s 1
```

### Inspect / activate / deactivate

```bash
vgdisplay -v vg01                          # full VG detail
vgdisplay -v vg01 | grep "LV Name"         # logical volumes
vgdisplay -v vg01 | grep "PV Name"         # physical volumes

vgchange -a y vg01                         # activate
vgchange -a n vg01                         # deactivate
```

## Logical Volumes

```bash
# Create an empty logical volume in a VG
lvcreate vg01
```

`lvcreate` attributes:

| Option | Meaning |
|--------|---------|
| `-L` | Size in MB (default 0) |
| `-l` | Size in logical extents |
| `-n <name>` | LV name (default `lvol1`, `lvol2`, …) |
| `-m <n>` | Number of mirror copies (1 = one extra copy) |
| `-D y` | Distributed/large-LV allocation (LVM 2.x) |
| `-C y` | Contiguous allocation (required for boot/root/swap/dump LVs) |

Each LV gives you two device files: the block device `/dev/vgNN/lvolN` (for mounting/filesystems) and the raw device `/dev/vgNN/rlvolN` (for `newfs`, `fsck`, and raw databases).

### Worked Example — Create and Grow a Filesystem

This is the everyday LVM workflow: carve an LV, put a VxFS filesystem on it, mount it, then grow it online later.

```bash
# 1. Create a 2 GB logical volume named "apps" in vg01
lvcreate -L 2048 -n apps vg01

# 2. Make a VxFS filesystem on the RAW device
newfs -F vxfs /dev/vg01/rapps

# 3. Mount it and add it to /etc/fstab so it comes back after reboot
mkdir /apps
mount -F vxfs /dev/vg01/apps /apps

# 4. Later: grow the LV by another 2 GB...
lvextend -L 4096 /dev/vg01/apps

# 5. ...then grow the filesystem to fill it (online, if OnlineJFS is licensed)
fsadm -F vxfs -b 4194304 /apps          # size in KB, or:
extendfs -F vxfs /dev/vg01/rapps        # offline grow (unmount first)
```

> **Gotcha — two-step growth.** `lvextend` only enlarges the *container*; the filesystem inside it does not grow until you run `fsadm`/`extendfs`. Online growth with `fsadm` needs the **OnlineJFS** (Advanced VxFS) license; without it you must unmount and use `extendfs`. Note also that `lvreduce`/shrinking a mounted VxFS is not supported — shrink the filesystem first, and never `lvreduce` below the filesystem size or you will truncate live data.

## Mirroring

LVM 1.0 supports 3-way mirroring, LVM 2.x supports 6-way; VxVM up to 32 mirrors per volume.

Mirroring in LVM means each logical extent maps to two (or more) physical extents on **different** PVs, so a write goes to both copies and a read can be satisfied from either. The key rule is **allocation policy**: mirror copies must land on separate physical volumes or the redundancy is pointless, which is why `lvextend -m` takes an explicit target PV. When a disk fails or is pulled, its extents become **stale**; when it returns, LVM copies the current data back from the good copy — this is the *resync*, driven by the VGSA's record of which extents are current. A "strict" allocation policy (the default) enforces the separate-PV rule; you can relax it, but doing so defeats the purpose. For the root VG specifically, mirroring must be paired with `mkboot`/`lvlnboot` so the second disk is independently bootable — a mirror that isn't a registered boot disk won't help you if the primary boot disk is the one that dies.

```bash
# Create a mirrored LV (1 mirror copy)
lvcreate -m 1 -n data vg01

# Add a mirror to an existing LV
lvextend -m 1 /dev/vg01/data /dev/disk/disk2

# Split off / merge a mirror copy
lvsplit /dev/vgweb/lvol1                   # default suffix "b" -> lvol1b
lvsplit -s mir /dev/vgweb/lvol2            # custom suffix -> lvol2mir
lvmerge /dev/vgweb/lvol1b  /dev/vgweb/lvol1
lvmerge /dev/vgweb/lvol2mir /dev/vgweb/lvol2

# Synchronize stale mirrors
lvsync /dev/vgweb/lvol1                    # one LV
vgsync vgweb                               # all LVs in a VG

# Track sync progress (count of stale extents)
lvdisplay -v $(find /dev/vg01 -type b) | grep -c stale
```

Check whether the mirroring license is installed:

```bash
# 11i v1/v2
swlist -l fileset -a state LVM.LVM-MIRROR-RUN
# 11i v3
swlist -l fileset -a state LVM-MirrorDisk.LVM-MIRROR
```

## Configuration Backup and Restore

LVM auto-saves each VG's config to `/etc/lvmconf/<vg>.conf` at creation, and `vgcfgbackup` runs automatically after structure-changing commands (`vgextend`, `vgreduce`, `lvcreate`, `lvextend`, `lvreduce`, `lvremove`, `lvsplit`, `lvmerge`, `lvchange`, `lvlnboot`, `lvrmboot`, `pvmove`, `pvchange`).

```bash
# Manual backup
vgcfgbackup vg01

# Restore config onto a (replacement) PV
vgcfgrestore -n vg01 /dev/rdisk/disk22

# List the disks recorded in a VG's backup
vgcfgrestore -l -n vg00
```

### Rebuild a Lost /etc/lvmtab

`/etc/lvmtab` records which PVs belong to which VGs. If it's lost, corrupted, or inconsistent, rescan all visible PVs and rebuild it:

```bash
vgscan
```

Without options, `vgscan` adds missing VG and PV entries (using legacy DSFs, except for VGs that were activated with persistent DSFs, which it records as persistent).

## Renaming a Volume Group

Export then re-import under the new name:

```bash
vgchange -a n vg01
vgexport -sv -m /tmp/vg01.map vg01
mkdir /dev/vg01ora
mknod /dev/vg01ora/group c 64 0x020000
vgimport -sv -m /tmp/vg01.map vg01ora
vgchange -a y vg01ora
vgcfgbackup vg01ora
```

## Modifying a VG (vgmodify)

```bash
# Change Max PV to 255
vgchange -a n vg01
vgmodify -p 255 -n vg01
vgchange -a y vg01

# Change Max PE per PV to 8128
vgchange -a n vg01
vgmodify -e 8128 -n vg01
vgchange -a y vg01
```

## Quiescing and Other VG Operations

```bash
# Hold writes for 400s (e.g. before a snapshot at the storage layer)
vgchange -Q w -t 400 vg01

# Hold both reads and writes until explicitly resumed
vgchange -Q rw vg01

# Resume a quiesced VG
vgchange -R vg01

# Sync all mirrored LVs in a VG
vgsync vg01

# Modify the VGID on a PV
vgchgid

# Migrate a VG from legacy to persistent device files
vgdsf -c vg01

# Upgrade an LVM 1.0 VG to LVM 2.x
vgversion
```

## Boot-Disk Mirroring / Recovery (vg00)

Re-mirror a replacement disk into the root VG:

```bash
pvcreate -B /dev/rdsk/c2t2d0                 # bootable PV
mkboot -l /dev/dsk/c2t2d0                    # lay down the LIF boot area
mkboot -a "hpux -lq" /dev/dsk/c2t2d0         # set the AUTO file boot string
vgcfgrestore -n vg00 /dev/rdsk/c2t2d0        # restore vg00 config onto it
vgsync vg00                                  # resync the mirror

# If needed, mark the failed original unavailable and find the mirror's minor
pvchange -a N /dev/rdsk/c0t6d0
ls -l /dev/dsk | grep 0x021000
```

Remove the boot area from a VG (make it non-bootable):

```bash
lvrmboot -r /dev/vg00
```

## Migrating Data Off a Disk (pvmove)

Before physically pulling a failing or soon-to-be-decommissioned disk, move its extents to other PVs in the same VG **online**, with no unmount:

```bash
# Move every extent off disk3 onto anywhere else in the VG
pvmove /dev/disk/disk3

# Move only a specific LV's extents, and onto a specific target PV
pvmove -n /dev/vg01/apps /dev/disk/disk3 /dev/disk/disk4

# Then remove the now-empty PV from the VG and release the disk
vgreduce vg01 /dev/disk/disk3
```

`pvmove` is transactional — if it is interrupted (crash, power loss) it can be re-run and will resume, because the intermediate state is recorded in the VGDA. Avoid running it during peak I/O, as every moved extent is a physical copy.

## Troubleshooting

| Symptom | Likely cause | Action |
|---------|--------------|--------|
| `vgcreate`/`vgextend`: "device busy" or already a PV | Disk still owned by another VG | `pvcreate -f` (only if you are sure), or `vgexport` the old VG first |
| VG won't activate: quorum not present | Too many PVs missing/offline | `vgchange -a y -q n <vg>` to override quorum, then repair |
| `stale` extents shown in `lvdisplay` | Mirror not yet resynced after a disk returned | `lvsync <lv>` or `vgsync <vg>` |
| `/etc/lvmtab` missing or inconsistent | File lost/corrupted | `vgscan` to rebuild it from the PVs |
| Replaced a failed disk, VG sees it as foreign | New disk has no VGDA | `vgcfgrestore -n <vg> <pv>` then `vgchange`/`vgsync` |
| Can't grow filesystem after `lvextend` | Filesystem not yet extended | `fsadm -F vxfs -b <kb> <mount>` (online) or `extendfs` (offline) |

> **Quorum override, carefully.** `vgchange -a y -q n` and boot-time `-lq` both bypass the majority-of-VGDA rule. They are the right tool when you have genuinely lost disks and need the survivors online, but if the "missing" copies are actually present and *newer*, overriding quorum can activate stale data. Confirm which PVs are truly gone before overriding.

## Simple Filesystem/Swap Use (non-LVM)

```bash
newfs /dev/rdisk/disk1        # create a filesystem directly on a disk
swapon /dev/disk/disk1        # use a disk as swap
```

## VxVM Note

All HP-UX Operating Environments include **Base-VxVM**, enough to configure simple VxVM volumes/disk groups and mirror the boot disk. Check for the licensed online VxVM features:

```bash
swlist B9116* T277[1-7]*
```

## Command Reference

| Task | Command |
|------|---------|
| LVM version | `lvmadm -t` |
| List PVs/VGs (v3) | `lvmadm -l` |
| Init PV | `pvcreate` (`-f` force, `-B` bootable) |
| Create VG (1.0) | `mknod .../group c 64 0xNN0000` + `vgcreate` |
| Create VG (2.x) | `vgcreate -V 2.1 -S <size> -s <extent> vg dev...` |
| Create LV | `lvcreate [-L mb|-l le][-n name] vg` |
| Display VG | `vgdisplay -v <vg>` |
| Activate/deactivate | `vgchange -a y|n <vg>` |
| Add mirror | `lvextend -m 1 <lv> <pv>` |
| Split/merge mirror | `lvsplit` / `lvmerge` |
| Sync mirrors | `lvsync <lv>` / `vgsync <vg>` |
| Config backup/restore | `vgcfgbackup` / `vgcfgrestore -n <vg> <pv>` |
| Rebuild lvmtab | `vgscan` |
| Rename VG | `vgexport` + `vgimport` |
| Modify VG limits | `vgmodify -p`/`-e` (VG deactivated) |
| Legacy → persistent DSF | `vgdsf -c <vg>` |
| Upgrade LVM version | `vgversion` |
| Boot PV / boot area | `pvcreate -B`, `mkboot`, `lvrmboot -r` |
| Migrate extents off a PV | `pvmove <pv>` then `vgreduce` |
| Grow a filesystem | `lvextend` then `fsadm`/`extendfs` |

## Related Articles

- [HP-UX Boot Process (PA-RISC and Integrity)](articles/hpux-boot-process.md) — how the boot area, quorum, and vg00 mirroring tie into booting
- [HP-UX Virtual Partitions (vPars)](articles/hpux-vpars.md) — per-vPar boot disks are LVM PVs managed with these same tools
