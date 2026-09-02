# Solaris Disk and Filesystem Management

Adding, slicing, and mounting disks on Oracle Solaris — the disk device naming scheme, how slicing differs between SPARC and x86, reading the VTOC, and the end-to-end workflow to bring a new disk online with UFS. Also covers preparing a whole disk for ZFS.

## Disk Slices: SPARC vs x86

How a disk is divided depends on the platform:

- **SPARC** — the entire disk is devoted to Solaris. The disk is divided into **8 slices**, numbered **0–7**.
- **x86** — the disk is first divided into **fdisk partitions** (via the `fdisk` command). The Solaris fdisk partition is then divided into **10 slices**, numbered **0–9**.

| Platform | Partitioning | Slices |
|----------|--------------|--------|
| SPARC | Whole disk is Solaris | 8 slices (0–7) |
| x86 | fdisk partitions, one is Solaris | 10 slices (0–9) |

Slice numbers have conventional roles — for example, slice 2 (`s2`) traditionally represents the whole disk ("backup" slice), and `s0` is commonly the root/primary data slice.

Conventional slice assignments (UFS-era systems):

| Slice | Traditional use |
|-------|-----------------|
| `s0` | root (`/`) or primary data |
| `s1` | swap |
| `s2` | "backup" — the entire disk (do not create a filesystem here) |
| `s3`–`s6` | `/var`, `/usr`, `/opt`, `/export`, etc. |
| `s7` | often `/home` or spare |
| `s8`, `s9` (x86 only) | `s8` = boot slice (1 cyl), `s9` = alternate sectors |

On x86 the extra slices (`s8`/`s9`) are reserved by the fdisk/label mechanism, which is why the Solaris partition holds 10 slices rather than 8. On modern ZFS root systems, this classic per-slice layout is largely replaced by a single pool on the whole disk.

## Device Naming

Solaris disk devices follow a `cCtTdDsS` (or `pP` for fdisk partitions) convention:

```
/dev/dsk/c0t2d0s0
          │ │ │ │
          │ │ │ └─ slice number (s0–s7 SPARC, s0–s9 x86)
          │ │ └─── disk/LUN number
          │ └───── target (SCSI/SAS target)
          └─────── controller number
```

- `/dev/dsk/...` — block device
- `/dev/rdsk/...` — raw/character device (used by `newfs`, `fdisk`, `prtvtoc`)
- `...p0` — the whole fdisk partition (x86), e.g. `/dev/rdsk/c0t1d0p0`

## Inspecting Disks

```bash
# List all disks (format lists devices, then pipe a newline to exit)
echo | format

# Read a disk's VTOC (Volume Table of Contents — the slice layout)
prtvtoc /dev/dsk/c0t2d0s0

# Scan for newly attached disks and rebuild device nodes
devfsadm -Cv

# View fdisk partition table details (x86)
fdisk -W - /dev/rdsk/c0t3d0p0
```

- `devfsadm -Cv` — `-C` cleans up stale `/dev` entries, `-v` is verbose; run it after hot-adding a disk so it appears.
- `prtvtoc` shows the slice table and is also used to copy a layout between identical disks (piping to `fmthard`).

Sample `format` disk listing:

```
AVAILABLE DISK SELECTIONS:
       0. c0t0d0 <VBOX-HARDDISK-1.0 cyl 2085 alt 2 hd 255 sec 63>
          /pci@0,0/pci8086,2829@d/disk@0,0
       1. c0t2d0 <VBOX-HARDDISK-1.0 cyl 1042 alt 2 hd 128 sec 32>
          /pci@0,0/pci8086,2829@d/disk@2,0
Specify disk (enter its number): ^D
```

Sample `prtvtoc` output (the slice table):

```
* /dev/dsk/c0t2d0s0 partition map
* Dimensions: 512 bytes/sector, 4270080 sectors
* First     Sector    Last
* Partition  Tag  Flags    Sector     Count    Sector  Mount Directory
       0      2    00          0    4269000   4268999
       2      5    01          0    4270080   4270079
```

The **Tag** column names the intended use (2 = root, 3 = swap, 5 = backup/whole disk), and **Flags** `01` means unmountable (the backup slice), `00` means a normal mountable slice.

### Copying a Slice Layout Between Identical Disks

`prtvtoc` piped to `fmthard` clones a VTOC — handy when preparing a mirror or replacement of the same disk model:

```bash
prtvtoc /dev/rdsk/c0t2d0s2 | fmthard -s - /dev/rdsk/c0t3d0s2
```

## Adding a New Disk (UFS)

The full workflow to bring a new disk online with a UFS filesystem:

```bash
# 1. Detect the new disk and create device nodes
devfsadm -Cv

# 2. Confirm the OS sees it
echo | format

# 3. Partition/slice the disk interactively in format (see walkthrough below)
format

# 4. Create a UFS filesystem on the target slice (raw device)
newfs /dev/rdsk/c0t2d0s0

# 5. Create a mount point
mkdir /newfs
```

Add a persistent entry to `/etc/vfstab` so it mounts at boot:

```
# device-to-mount   device-to-fsck        mount-point  FS   fsck-pass  mount-at-boot  options
/dev/dsk/c0t2d0s0   /dev/rdsk/c0t2d0s0    /newfs       ufs  2          yes            -
```

Then mount it:

```bash
mount /newfs

# Verify
df -h /newfs
```

### /etc/vfstab Fields

| Field | Example | Meaning |
|-------|---------|---------|
| device to mount | `/dev/dsk/c0t2d0s0` | Block device |
| device to fsck | `/dev/rdsk/c0t2d0s0` | Raw device for `fsck` (`-` for none) |
| mount point | `/newfs` | Where it mounts |
| FS type | `ufs` | Filesystem type |
| fsck pass | `2` | Order for `fsck` (0/`-` to skip) |
| mount at boot | `yes` | Mount automatically at boot |
| mount options | `-` | Options (`-` = defaults) |

### Slicing Interactively in `format`

Inside `format`, the flow to lay down slices and label the disk:

```
format> partition        # enter the partition menu
partition> print         # show current slices
partition> 0             # select slice 0 to edit
Enter partition id tag[unassigned]: root
Enter partition permission flags[wm]: wm
Enter new starting cyl[0]: 0
Enter partition size[0b, 0c, 0.00mb, 0.00gb]: 2gb
partition> label         # write the new VTOC to disk
Ready to label disk, continue? y
partition> quit
format> quit
```

- `wm` = read-write, mountable; `wu` = read-write, unmountable (e.g. swap); `rm` = read-only.
- You must `label` for changes to persist — quitting without labeling discards them.
- Always leave slice 2 as the whole-disk "backup" slice.
- Use **`format -e`** to choose the label type (SMI/VTOC vs **EFI/GPT**). An EFI label is required for disks larger than ~2 TB and is applied automatically when ZFS takes a whole disk.

### Growing a UFS Filesystem

UFS can be grown online with `growfs` after the underlying slice/volume is enlarged (e.g. an SVM/expanded LUN):

```bash
growfs -M /newfs /dev/rdsk/c0t2d0s0
```

`-M` grows a mounted filesystem in place. UFS cannot be shrunk — to reduce size you recreate it.

## Preparing a Whole Disk for ZFS

To hand an entire disk to ZFS, give it an EFI/whole-disk fdisk layout first:

```bash
# Format the disk so the whole disk can be used by ZFS (x86)
fdisk -B /dev/rdsk/c0t1d0p0

# Then create a pool on the whole disk (ZFS labels it EFI automatically)
zpool create tank c0t1d0
```

Handing ZFS the whole disk (rather than a slice) lets it manage the disk write cache and is Oracle's recommended approach.

Common follow-on ZFS operations:

```bash
# Create datasets (filesystems) inside the pool
zfs create tank/data
zfs create -o mountpoint=/export/app tank/app

# Set properties (compression, quota, reservation)
zfs set compression=lz4 tank/data
zfs set quota=50g tank/data

# Mirror / add devices
zpool create tank mirror c0t1d0 c0t2d0     # mirrored pool
zpool add tank mirror c0t3d0 c0t4d0        # grow the pool

# Status and space
zpool status tank
zpool list
zfs list

# Snapshots
zfs snapshot tank/data@backup-2024-06-01
zfs rollback tank/data@backup-2024-06-01
```

ZFS filesystems mount automatically at their `mountpoint` property — no `/etc/vfstab` entry needed (that's for UFS and legacy mounts). A pool created on the whole disk survives being moved between controllers because ZFS identifies devices by their labels, not fixed `cXtXdX` paths.

## Default UFS Mount Options

When a UFS filesystem is mounted with defaults (`-` in vfstab), Solaris applies:

**read/write, setuid, intr, logging, largefiles, xattr, onerror=panic**

- **logging** — UFS journaling (default on modern Solaris), enabling fast, consistent recovery.
- **largefiles** — allows files larger than 2 GB.
- **xattr** — extended attributes support.
- **onerror=panic** — on an unrecoverable error, the system panics rather than continuing.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| New disk not listed in `format` | Device nodes not created | `devfsadm -Cv`; check cabling/SCSI target |
| `newfs` refuses / "device busy" | Slice mounted or in a pool | `umount` it; confirm it's not in a zpool |
| Mount fails at boot | Bad `/etc/vfstab` entry | Boot single-user, fix vfstab, `mount -a` to test |
| `format` shows disk but no slices | Disk unlabeled | Label it in `format` (`label` command) |
| `fsck` needed on boot | Unclean UFS unmount | Run `fsck /dev/rdsk/cXtXdXsN`; UFS logging usually avoids this |
| ZFS pool won't import after move | Devices renamed | `zpool import` (ZFS finds by label, not path) |

```bash
# A vfstab typo can block boot — always test before rebooting
mount -a          # mounts everything in vfstab; reports errors
```

## Quick Reference

| Task | Command |
|------|---------|
| List disks | `echo \| format` |
| Read VTOC | `prtvtoc /dev/dsk/cXtXdXs0` |
| Rescan for new disks | `devfsadm -Cv` |
| Create UFS filesystem | `newfs /dev/rdsk/cXtXdXs0` |
| Whole-disk fdisk for ZFS | `fdisk -B /dev/rdsk/cXtXdXp0` |
| View fdisk table (x86) | `fdisk -W - /dev/rdsk/cXtXdXp0` |
| Check filesystem | `fsck /dev/rdsk/cXtXdXs0` |
| Mount from vfstab | `mount /mountpoint` |

## References

- [Managing Disks in Oracle Solaris](https://docs.oracle.com/cd/E37838_01/html/E22525/index.html) — official Oracle docs
- [Oracle Solaris ZFS Administration Guide](https://docs.oracle.com/cd/E37838_01/html/E61017/index.html) — official Oracle docs
