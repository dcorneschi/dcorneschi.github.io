# Disk Partitioning Cheatsheet

Tools for viewing, creating, modifying, and managing disk partitions on Linux: `fdisk`, `gdisk`, `parted`, `lsblk`, `blkid`, and `mkfs`.

---

## Partition Tables: MBR vs GPT

| Feature | MBR (msdos) | GPT |
|---------|-------------|-----|
| Max disk size | 2 TB | 9.4 ZB (effectively unlimited) |
| Max partitions | 4 primary (or 3 primary + 1 extended with logical) | 128 (default) |
| Boot mode | BIOS / Legacy | UEFI (and BIOS with protective MBR) |
| Redundancy | Single copy of partition table | Primary + backup copy at end of disk |
| Tools | `fdisk`, `parted` | `gdisk`, `parted`, `fdisk` (2.26+) |

> **Rule of thumb:** Use GPT for disks > 2 TB or UEFI systems. Use MBR only for legacy BIOS compatibility.

---

## Viewing Disk Information

### lsblk

```bash
# Show all block devices in tree format
lsblk

# Show with filesystem type, label, UUID, and mount point
lsblk -f

# Show sizes in bytes
lsblk -b

# Show specific columns
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,UUID

# Show only disks (no partitions)
lsblk -d

# Show all devices including empty ones
lsblk -a

# Show in list format (no tree)
lsblk -l

# Show SCSI devices
lsblk -S
```

### blkid

```bash
# Show all block device attributes
blkid

# Show info for a specific device
blkid /dev/sda1

# Show only UUID
blkid -s UUID -o value /dev/sda1

# Show only filesystem type
blkid -s TYPE -o value /dev/sda1

# Show only label
blkid -s LABEL -o value /dev/sda1

# Probe a device (force re-read)
blkid -p /dev/sda1

# Output in key=value format for scripting
blkid -o export /dev/sda1
```

### fdisk -l

```bash
# List all disks and partitions
fdisk -l

# List a specific disk
fdisk -l /dev/sda

# Show sizes in sectors
fdisk -l -u /dev/sda

# List disks in short format (just disk names and sizes)
fdisk -l | grep "Disk /"

# Show sector size information
fdisk -l /dev/sda | grep -E "(Disk|Sector)"

# Show in JSON format (fdisk 2.27+, RHEL 8+, Ubuntu 18.04+)
fdisk -J -l
```

### parted print

```bash
# Print partition table
parted /dev/sda print

# Print in machine-readable format
parted -m /dev/sda print

# Print free space
parted /dev/sda print free

# Print all disks
parted -l
```

---

## fdisk (MBR and GPT)

Interactive partition editor. Supports MBR (traditional) and GPT (fdisk 2.26+, available on RHEL 7+ and Ubuntu 16.04+).

### Starting fdisk

```bash
# Open a disk for editing
fdisk /dev/sda

# Open in sector display mode (default on RHEL 7+; needed on RHEL 6)
fdisk -u /dev/sda

# RHEL 6: use -cu for sector mode with compatibility
fdisk -cu /dev/sda

# Non-interactive: list partitions only
fdisk -l /dev/sda
```

### Interactive Commands

| Key | Action |
|-----|--------|
| `m` | Help menu |
| `p` | Print partition table |
| `n` | Create new partition |
| `d` | Delete a partition |
| `t` | Change partition type |
| `l` | List known partition types |
| `a` | Toggle bootable flag (MBR) |
| `w` | Write changes and exit |
| `q` | Quit without saving |
| `g` | Create new GPT partition table |
| `o` | Create new MBR (DOS) partition table |
| `F` | List free unpartitioned space |
| `i` | Print information about a partition |
| `x` | Extra functionality (expert mode) |

### Create a New Partition (Interactive)

```bash
fdisk /dev/sdb
# Command: n        (new partition)
# Partition type: p  (primary) or l (logical)
# Partition number: 1
# First sector: (press Enter for default)
# Last sector: +10G  (or +500M, or press Enter for remaining space)
# Command: w        (write and exit)
```

### Delete a Partition

```bash
fdisk /dev/sdb
# Command: d
# Partition number: 2
# Command: w
```

### Create Extended Partition with Logical Drives (MBR Only)

MBR allows max 4 primary partitions. To have more, create an extended partition containing logical drives:

```bash
fdisk /dev/sdb
# Command: n        (new partition)
# Type: e           (extended)
# Partition number: 3
# First sector: (Enter for default)
# Last sector: (Enter to use remaining space)

# Now create logical partitions inside the extended:
# Command: n        (new partition — automatically logical)
# First sector: (Enter)
# Last sector: +5G

# Command: n        (another logical)
# First sector: (Enter)
# Last sector: +5G

# Command: w        (write)
```

### Change Partition Type

```bash
fdisk /dev/sdb
# Command: t
# Partition number: 1
# Hex code: 8e      (Linux LVM)
# Command: w
```

Common partition type codes (MBR):

| Code | Type |
|------|------|
| `83` | Linux |
| `82` | Linux swap |
| `8e` | Linux LVM |
| `fd` | Linux RAID autodetect |
| `07` | NTFS / HPFS |
| `0c` | FAT32 (LBA) |
| `ef` | EFI System Partition |

### Scripting fdisk (Non-Interactive)

```bash
# Create a single partition using the full disk
echo -e "n\np\n1\n\n\nw" | fdisk /dev/sdb

# Create partition with specific size
echo -e "n\np\n1\n\n+10G\nw" | fdisk /dev/sdb

# Using printf
printf "o\nn\np\n1\n\n\nw\n" | fdisk /dev/sdb

# Using heredoc
fdisk /dev/sdb <<EOF
n
p
1


+10G
w
EOF

# RHEL 6 style (with -cu flag for cylinder/unit mode)
cat << EOF | fdisk -cu /dev/sdb
n
p
1


t
8e
w
EOF

# Using sfdisk (better for scripting)
echo ",,L" | sfdisk /dev/sdb
```

### Create LVM Partition Non-Interactively

```bash
# Create a single LVM partition (type 8e) using entire disk
echo -e "n\np\n1\n\n\nt\n8e\nw" | fdisk /dev/sdb

# Then set up LVM
pvcreate /dev/sdb1
vgcreate vg_data /dev/sdb1
lvcreate -l 100%FREE -n lv_data vg_data
mkfs.ext4 /dev/vg_data/lv_data
```

### Informing the Kernel After Partition Changes (RHEL 6–10)

| Version | Recommended Method | Notes |
|---------|-------------------|-------|
| RHEL 6 | `partx -v -a /dev/sdb` | `partprobe` only works if **no** partitions on the disk are mounted. Use `partx` for adding new partitions without reboot. |
| RHEL 7 | `partprobe /dev/sdb` | Works reliably when existing partitions are not modified. |
| RHEL 8 | `partprobe /dev/sdb` | Same as RHEL 7. |
| RHEL 9 | `partprobe /dev/sdb` | Same as RHEL 7. |
| RHEL 10 | `partprobe /dev/sdb` | Same as RHEL 7. |

**RHEL 6 specifics:**

`partprobe` in RHEL 6 will only trigger the OS to update partitions on a disk if **none** of its partitions are in use (mounted). If any partition on the disk is mounted, `partprobe` will not update the system partition table.

Options when a partition is mounted on RHEL 6:

1. Unmount all partitions on the disk, then run `partprobe`
2. If a new partition was added (and existing partitions were not modified), use `partx`:

```bash
# List partitions as seen on disk
partx -l /dev/sdb

# Add new partitions to the kernel (verbose)
partx -v -a /dev/sdb

# Verify
ls /dev/sdb*
```

3. If existing partitions were modified, reboot the system

> **Warning:** `partx` does minimal validation between the new and existing partition table. It assumes the user knows what they are doing. Use at your own risk — it can corrupt data if existing partitions were modified or the partition table is incorrect.

---

## gdisk (GPT Only)

GPT-specific partition editor. Same interactive style as fdisk but for GPT disks.

### Starting gdisk

```bash
# Open a disk for editing
gdisk /dev/sda

# Non-interactive: list partitions
gdisk -l /dev/sda
```

### Interactive Commands

| Key | Action |
|-----|--------|
| `?` | Help menu |
| `p` | Print partition table |
| `n` | Create new partition |
| `d` | Delete a partition |
| `t` | Change partition type (GUID) |
| `l` | List known partition types |
| `i` | Show detailed partition info |
| `o` | Create new empty GPT |
| `w` | Write changes and exit |
| `q` | Quit without saving |
| `v` | Verify disk |
| `x` | Extra functionality (expert mode) |
| `r` | Recovery and transformation options |

### Common GPT Type GUIDs

| Code | Type |
|------|------|
| `8300` | Linux filesystem |
| `8200` | Linux swap |
| `8e00` | Linux LVM |
| `fd00` | Linux RAID |
| `ef00` | EFI System Partition |
| `ef02` | BIOS boot partition |
| `0700` | Microsoft basic data |

### Create a GPT Partition

```bash
gdisk /dev/sdb
# Command: o        (create new GPT — destroys existing data)
# Command: n        (new partition)
# Partition number: 1
# First sector: (Enter for default)
# Last sector: +10G
# Type code: 8300   (Linux filesystem)
# Command: w        (write)
```

### Convert MBR to GPT

```bash
gdisk /dev/sdb
# Command: w        (gdisk auto-detects MBR and offers conversion)
# Confirm: Y

# Or non-destructively with sgdisk
sgdisk -g /dev/sdb
```

---

## sgdisk (GPT Scripting Tool)

Non-interactive command-line tool for GPT operations. Ideal for automation.

```bash
# Print partition table
sgdisk -p /dev/sdb

# Create a new GPT partition table (wipes existing)
sgdisk -Z /dev/sdb

# Create partition 1: starts at first available, size 10G, type Linux filesystem
sgdisk -n 1:0:+10G -t 1:8300 /dev/sdb

# Create partition 2: remaining space, type Linux LVM
sgdisk -n 2:0:0 -t 2:8e00 /dev/sdb

# Create EFI system partition
sgdisk -n 1:0:+512M -t 1:ef00 -c 1:"EFI System" /dev/sdb

# Delete partition 2
sgdisk -d 2 /dev/sdb

# Change partition name (label)
sgdisk -c 1:"Boot" /dev/sdb

# Change partition type
sgdisk -t 1:8300 /dev/sdb

# Backup partition table
sgdisk -b /tmp/sdb-gpt-backup.bin /dev/sdb

# Restore partition table
sgdisk -l /tmp/sdb-gpt-backup.bin /dev/sdb

# Clone partition table from one disk to another
sgdisk -R /dev/sdc /dev/sdb

# Randomize GUIDs after cloning (required for unique identity)
sgdisk -G /dev/sdc

# Verify partition table
sgdisk -v /dev/sdb

# Convert MBR to GPT
sgdisk -g /dev/sdb

# Sort partitions by start sector
sgdisk -s /dev/sdb
```

---

## parted (MBR and GPT)

Supports both MBR and GPT. Can resize partitions (unlike fdisk/gdisk). Changes take effect immediately — no "write" step.

> **Warning:** parted applies changes immediately. There is no "quit without saving" for destructive operations.

### Interactive Mode

```bash
parted /dev/sdb

# Inside parted:
# print                       — show partition table
# print free                  — show free space
# mklabel gpt                 — create GPT table (destroys all data)
# mklabel msdos               — create MBR table (destroys all data)
# mkpart primary ext4 1MiB 10GiB  — create partition
# mkpart primary linux-swap 10GiB 12GiB
# rm 2                        — remove partition 2
# resizepart 1 20GiB          — resize partition 1 to 20GiB
# name 1 "boot"               — set partition name (GPT only)
# set 1 boot on               — set boot flag
# set 1 lvm on                — set LVM flag
# align-check optimal 1       — verify alignment
# quit
```

### Non-Interactive (Scripting)

```bash
# Create GPT partition table
parted -s /dev/sdb mklabel gpt

# Create a partition (start at 1MiB for alignment, end at 10GiB)
parted -s /dev/sdb mkpart primary ext4 1MiB 10GiB

# Create a second partition using remaining space
parted -s /dev/sdb mkpart primary ext4 10GiB 100%

# Create swap partition
parted -s /dev/sdb mkpart primary linux-swap 10GiB 12GiB

# Set boot flag
parted -s /dev/sdb set 1 boot on

# Set LVM flag
parted -s /dev/sdb set 1 lvm on

# Remove partition 2
parted -s /dev/sdb rm 2

# Resize partition (only shrinks/expands the partition entry, not the filesystem)
parted -s /dev/sdb resizepart 1 20GiB

# Print partition table
parted -s /dev/sdb print

# Print in machine-readable format
parted -ms /dev/sdb print

# Check alignment
parted /dev/sdb align-check optimal 1
```

---

## sfdisk (MBR/GPT Scripting)

Non-interactive partition tool, best for cloning and scripting.

```bash
# Dump partition table (backup)
sfdisk -d /dev/sdb > sdb-partitions.dump

# Restore partition table
sfdisk /dev/sdb < sdb-partitions.dump

# Clone partition layout to another disk
sfdisk -d /dev/sdb | sfdisk /dev/sdc

# Create a single Linux partition using entire disk
echo ",,L" | sfdisk /dev/sdb

# Create specific layout
sfdisk /dev/sdb <<EOF
,10G,L
,2G,S
,,L
EOF
# L = Linux, S = swap

# List partition table
sfdisk -l /dev/sdb

# Verify partition table
sfdisk --verify /dev/sdb

# Delete all partitions
sfdisk --delete /dev/sdb

# Delete specific partition
sfdisk --delete /dev/sdb 2

# Reorder partitions (util-linux 2.34+, RHEL 9+, Ubuntu 20.04+)
sfdisk --reorder /dev/sdb
```

---

## Creating Filesystems (mkfs)

After creating partitions, format them:

```bash
# ext4 (most common Linux filesystem)
mkfs.ext4 /dev/sdb1

# ext4 with label
mkfs.ext4 -L "data" /dev/sdb1

# ext4 with specific block size
mkfs.ext4 -b 4096 /dev/sdb1

# ext4 with reserved space set to 1% (default 5%)
mkfs.ext4 -m 1 /dev/sdb1

# XFS
mkfs.xfs /dev/sdb1

# XFS with label
mkfs.xfs -L "data" /dev/sdb1

# XFS force (overwrite existing filesystem)
mkfs.xfs -f /dev/sdb1

# Btrfs
mkfs.btrfs /dev/sdb1

# Btrfs with label
mkfs.btrfs -L "data" /dev/sdb1

# FAT32 (for EFI partitions or USB drives)
mkfs.vfat -F 32 /dev/sdb1

# FAT32 with label
mkfs.vfat -F 32 -n "EFI" /dev/sdb1

# Swap
mkswap /dev/sdb2
mkswap -L "swap" /dev/sdb2

# Enable swap
swapon /dev/sdb2
```

---

## Partition Alignment

Modern disks (SSDs, AF HDDs) require 1 MiB alignment for optimal performance:

```bash
# Check if partition is aligned
parted /dev/sdb align-check optimal 1

# Verify alignment manually (start sector should be divisible by 2048)
fdisk -l /dev/sdb | grep "^/dev"

# parted automatically aligns with MiB units
parted -s /dev/sdb mkpart primary ext4 1MiB 100%

# fdisk aligns by default on modern versions (2048-sector boundary)
```

---

## Inform the Kernel of Changes

After modifying partitions on a disk that's in use:

```bash
# Re-read partition table
# On RHEL 7+: works for new partitions even if others are mounted
# Fails if you modified/deleted a partition that is currently mounted
partprobe /dev/sdb

# Force kernel to re-read
blockdev --rereadpt /dev/sdb

# Add a specific partition to the kernel
partx -a /dev/sdb

# Update kernel's view of a specific partition
partx -u /dev/sdb

# Remove a partition from kernel (without touching disk)
partx -d --nr 2 /dev/sdb

# Verify kernel sees the partitions
cat /proc/partitions | grep sdb
ls /dev/sdb*
```

---

## Wiping and Zeroing

```bash
# Wipe filesystem signatures (removes FS magic bytes)
wipefs -a /dev/sdb1

# Show what would be wiped
wipefs /dev/sdb1

# Zero the first 1MB (wipes partition table and boot sector)
dd if=/dev/zero of=/dev/sdb bs=1M count=1

# Zero entire disk (slow — writes zeros to every sector)
dd if=/dev/zero of=/dev/sdb bs=1M status=progress

# Secure erase with random data
dd if=/dev/urandom of=/dev/sdb bs=1M status=progress

# Wipe GPT and MBR structures with sgdisk
sgdisk -Z /dev/sdb

# Discard/TRIM entire SSD (fast secure erase)
blkdiscard /dev/sdb
```

> **Warning:** These commands destroy data permanently. Double-check the device name before running.

---

## Common Workflows

### New Disk: GPT + Single ext4 Partition

```bash
# Create GPT table
parted -s /dev/sdb mklabel gpt

# Create partition using full disk
parted -s /dev/sdb mkpart primary ext4 1MiB 100%

# Format
mkfs.ext4 -L "data" /dev/sdb1

# Mount
mkdir -p /mnt/data
mount /dev/sdb1 /mnt/data

# Add to fstab (by UUID)
UUID=$(blkid -s UUID -o value /dev/sdb1)
echo "UUID=$UUID /mnt/data ext4 defaults 0 2" >> /etc/fstab
```

### New Disk: GPT + EFI + Root + Swap

```bash
parted -s /dev/sda mklabel gpt
parted -s /dev/sda mkpart primary fat32 1MiB 512MiB
parted -s /dev/sda set 1 esp on
parted -s /dev/sda mkpart primary linux-swap 512MiB 4GiB
parted -s /dev/sda mkpart primary ext4 4GiB 100%

mkfs.vfat -F 32 /dev/sda1
mkswap /dev/sda2
mkfs.ext4 /dev/sda3
```

### New Disk: LVM Setup

```bash
# Create partition with LVM type
parted -s /dev/sdb mklabel gpt
parted -s /dev/sdb mkpart primary 1MiB 100%
parted -s /dev/sdb set 1 lvm on

# Create physical volume
pvcreate /dev/sdb1

# Create volume group
vgcreate vg_data /dev/sdb1

# Create logical volumes
lvcreate -L 50G -n lv_app vg_data
lvcreate -l 100%FREE -n lv_logs vg_data

# Format
mkfs.ext4 /dev/vg_data/lv_app
mkfs.xfs /dev/vg_data/lv_logs
```

### Extend a Partition (Non-LVM)

```bash
# 1. Unmount
umount /dev/sdb1

# 2. Delete and recreate partition with larger size (fdisk keeps data if same start sector)
fdisk /dev/sdb
# d → delete partition 1
# n → new partition 1, same start sector, larger end
# w → write

# 3. Inform kernel
partprobe /dev/sdb

# 4. Resize filesystem
resize2fs /dev/sdb1       # ext4
xfs_growfs /mountpoint    # XFS (must be mounted)
```

---

## Troubleshooting

### "Device or resource busy"

```bash
# Check what's using the disk
fuser -mv /dev/sdb
lsof /dev/sdb

# Check for mounted partitions
mount | grep sdb

# Check for active swap
swapon --show | grep sdb

# Check for LVM
pvs | grep sdb
dmsetup ls
```

### Partition table not updating in kernel

```bash
partprobe /dev/sdb
# If that fails:
blockdev --rereadpt /dev/sdb
# Alternative using kpartx (for device-mapper devices)
kpartx -a /dev/sdb
# If still failing (partition mounted):
# Reboot, or unmount first
```

### "WARNING: Re-reading the partition table failed with error 16: Device or resource busy"

The disk has mounted partitions. Unmount all partitions on that disk first, then re-run `partprobe`.

### Disk shows wrong size

```bash
# Rescan SCSI bus (for hot-added or expanded virtual disks)
echo 1 > /sys/class/block/sdb/device/rescan

# Or rescan all SCSI hosts
for host in /sys/class/scsi_host/host*; do echo "- - -" > $host/scan; done

# Verify new size
lsblk /dev/sdb
fdisk -l /dev/sdb
```

---

## Quick Reference

| Command | Description |
|---------|-------------|
| `lsblk -f` | Show devices with filesystem info |
| `blkid` | Show UUIDs and filesystem types |
| `fdisk -l` | List all partitions |
| `fdisk /dev/sdb` | Edit partitions (interactive) |
| `gdisk /dev/sdb` | Edit GPT partitions (interactive) |
| `sgdisk -p /dev/sdb` | Print GPT partition table |
| `sgdisk -Z /dev/sdb` | Zap (wipe) partition table |
| `sgdisk -n 1:0:+10G /dev/sdb` | Create 10G partition |
| `parted -s /dev/sdb mklabel gpt` | Create GPT table |
| `parted -s /dev/sdb mkpart primary ext4 1MiB 100%` | Create partition |
| `parted -s /dev/sdb print` | Print partition layout |
| `sfdisk -d /dev/sdb > backup` | Backup partition table |
| `sfdisk /dev/sdb < backup` | Restore partition table |
| `mkfs.ext4 /dev/sdb1` | Format as ext4 |
| `mkfs.xfs /dev/sdb1` | Format as XFS |
| `mkfs.vfat -F 32 /dev/sdb1` | Format as FAT32 |
| `mkswap /dev/sdb2` | Create swap |
| `wipefs -a /dev/sdb1` | Wipe filesystem signatures |
| `partprobe /dev/sdb` | Reload partition table in kernel |
| `resize2fs /dev/sdb1` | Resize ext4 filesystem |
| `xfs_growfs /mnt` | Grow XFS filesystem |
| `df -h` | Show mounted filesystem usage |
| `mount \| column -t` | Show mounted devices formatted |
| `testdisk /dev/sdb` | Recover lost partitions |
| `cfdisk /dev/sdb` | Curses-based partition editor |

---

## MBR Backup and Restore

```bash
# Backup MBR (first 512 bytes = boot code + partition table)
dd if=/dev/sdb of=/tmp/sdb_mbr.backup bs=512 count=1

# Backup only partition table (bytes 446-511, without boot code)
dd if=/dev/sdb of=/tmp/sdb_pt.backup bs=1 skip=446 count=66

# Restore full MBR from backup
dd if=/tmp/sdb_mbr.backup of=/dev/sdb bs=512 count=1

# Restore only partition table (preserve boot code)
dd if=/tmp/sdb_pt.backup of=/dev/sdb bs=1 seek=446 count=66
```

---

## Recovery with testdisk

`testdisk` recovers lost partitions and repairs boot sectors:

```bash
# Install
sudo apt install testdisk       # Debian/Ubuntu
sudo yum install testdisk       # RHEL/CentOS (EPEL)

# Launch (interactive)
testdisk /dev/sdb

# Steps:
# 1. Select disk
# 2. Select partition table type (Intel/GPT)
# 3. Analyse → Quick Search → Deeper Search
# 4. Select recovered partitions
# 5. Write partition table
```

---

## sfdisk Layout File Format

For repeatable deployments, use sfdisk with an explicit layout file:

```bash
cat > /tmp/partition_layout << 'EOF'
label: dos
device: /dev/sdb
unit: sectors

/dev/sdb1 : start=2048, size=20971520, type=83
/dev/sdb2 : start=20973568, size=8388608, type=82
/dev/sdb3 : start=29362176, type=83
EOF

# Apply layout
sfdisk /dev/sdb < /tmp/partition_layout
```

GPT layout:

```bash
cat > /tmp/gpt_layout << 'EOF'
label: gpt
device: /dev/sdb

/dev/sdb1 : start=2048, size=1048576, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B, name="EFI"
/dev/sdb2 : start=1050624, size=8388608, type=0657FD6D-A4AB-43C4-84E5-0933C84B4F4F, name="swap"
/dev/sdb3 : start=9439232, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, name="root"
EOF

sfdisk /dev/sdb < /tmp/gpt_layout
```

---

## cfdisk (Curses-Based Interface)

`cfdisk` provides a visual menu-driven partition editor:

```bash
cfdisk /dev/sdb
```

Navigation:
- Arrow keys to select partitions
- Enter to choose actions (New, Delete, Type, Write, Quit)
- Tab to switch between action buttons

Useful when you need a visual partition editor without a GUI.

---

## Automation Script: Complete Disk Setup

```bash
#!/bin/bash
set -e

DISK="${1:-/dev/sdb}"
MOUNT_POINT="${2:-/mnt/data}"

if [[ -z "$1" ]]; then
    echo "Usage: $0 /dev/sdX [/mount/point]"
    exit 1
fi

echo "Setting up disk: $DISK"
echo "Mount point: $MOUNT_POINT"

# Safety check
if mount | grep -q "$DISK"; then
    echo "ERROR: $DISK has mounted partitions. Unmount first."
    exit 1
fi

# Backup existing MBR
dd if="$DISK" of="/tmp/$(basename $DISK)_mbr.backup" bs=512 count=1 2>/dev/null

# Create single partition using full disk
fdisk "$DISK" <<EOF
n
p
1


w
EOF

# Wait for kernel to pick up partition
partprobe "$DISK"
sleep 1

# Format with ext4
mkfs.ext4 "${DISK}1"

# Create mount point and mount
mkdir -p "$MOUNT_POINT"
mount "${DISK}1" "$MOUNT_POINT"

# Add to fstab
UUID=$(blkid -s UUID -o value "${DISK}1")
echo "UUID=$UUID $MOUNT_POINT ext4 defaults 0 2" >> /etc/fstab

echo "Done!"
echo "  Partition: ${DISK}1"
echo "  UUID: $UUID"
echo "  Mount: $MOUNT_POINT"
echo "  Filesystem: ext4"
```

---

## Safety Warnings

1. **Always verify the correct device** — `lsblk` and `fdisk -l` before any operation
2. **Double-check device names** — `/dev/sda` vs `/dev/sdb` (a mistake here destroys data)
3. **Backup partition tables** before modifying: `sfdisk -d /dev/sdb > backup`
4. **Never interrupt fdisk while writing** — `w` commits immediately to disk
5. **Use `q` to quit** without saving if you make a mistake
6. **Test on non-production systems first** — partition operations are often irreversible
7. **parted writes immediately** — unlike fdisk, there is no "save" step; changes are instant

---

## fdisk vs Other Tools

| Tool | Partition Table | Interface | Best for |
|------|----------------|-----------|----------|
| `fdisk` | MBR + GPT (2.26+) | Interactive | Quick manual partitioning |
| `gdisk` | GPT only | Interactive | GPT-specific features |
| `sgdisk` | GPT only | Command-line | Scripting GPT operations |
| `parted` | MBR + GPT | Both | Resizing, scripting |
| `sfdisk` | MBR + GPT | Command-line | Backup/restore, cloning |
| `cfdisk` | MBR + GPT | Curses TUI | Visual editor without GUI |
| `gparted` | MBR + GPT | GUI | Desktop users |
