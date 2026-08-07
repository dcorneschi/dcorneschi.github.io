# Extending Partitions with growpart

`growpart` extends a partition to fill available free space on a disk — without unmounting, without data loss, and without rebooting. It's the standard tool for resizing partitions on cloud instances (AWS, Azure, GCP) and virtual machines after expanding the underlying disk.

### Works on All Disk Types

- **Cloud VMs** — AWS, Azure, GCP, Oracle Cloud, etc.
- **Virtual disks** — KVM, VirtualBox, VMware, Hyper-V
- **Physical disks** — internal SATA/NVMe, external USB, SSDs, HDDs
- **Storage arrays** and LVM logical volumes

The only requirement is **unallocated space** after the partition you want to expand.

## What growpart Does

`growpart` only resizes the **partition table entry**. It does not resize the filesystem. After running growpart, you must also grow the filesystem with a separate command (`resize2fs`, `xfs_growfs`, etc.).

```
Disk expanded (e.g., 20GB → 50GB)
         ↓
growpart /dev/sda 1          ← extends partition 1 to use new space
         ↓
resize2fs /dev/sda1          ← extends the filesystem to fill the partition
```

---

## Installation

```bash
# RHEL / CentOS / Rocky / Amazon Linux
yum install cloud-utils-growpart      # RHEL 7
dnf install cloud-utils-growpart      # RHEL 8+

# Ubuntu / Debian
apt install cloud-guest-utils

# Alpine
apk add cloud-utils

# Verify
growpart --help
```

> **Note:** On Amazon Linux and most cloud images, `growpart` is pre-installed.

---

## growpart vs fdisk

| Tool | Best for | Difficulty |
|------|----------|------------|
| `growpart` | Expanding a partition to fill free space | Easy — one command |
| `fdisk` | Creating/deleting partitions, changing types | Manual — multi-step, error-prone for resizing |

Use `growpart` for resizing — it's safe, single-command, and works online. Avoid using `fdisk` to resize (it requires deleting and recreating the partition with manual calculations).

### Side-by-Side: Extending Partition 1 to Fill Disk

| Step | growpart | fdisk |
|------|----------|-------|
| 1 | `growpart /dev/sda 1` | `fdisk /dev/sda` |
| 2 | Done. | `p` — note start sector of partition 1 |
| 3 | | `d` — delete partition 1 |
| 4 | | `n` — new partition, same number (1) |
| 5 | | Enter **same start sector** as before |
| 6 | | Enter for default end (uses all space) |
| 7 | | `N` — do NOT remove ext4 signature if prompted |
| 8 | | `w` — write and exit |
| 9 | `resize2fs /dev/sda1` | `partprobe /dev/sda` then `resize2fs /dev/sda1` |

### fdisk Method (When growpart Is Not Available)

> **Always create a snapshot/backup before resizing with fdisk.** Unlike growpart, the fdisk method deletes and recreates the partition — a wrong start sector means total data loss.

```bash
# 1. Note the start sector of the partition
fdisk -l /dev/sda | grep /dev/sda1
# /dev/sda1  *  2048  41943039  41940992  20G  83  Linux
#                ^^^^
#                start sector = 2048 (REMEMBER THIS)

# 2. Delete and recreate with same start, larger end
fdisk /dev/sda
# Command: d
# Partition number: 1
#
# Command: n
# Partition type: p (primary)
# Partition number: 1
# First sector: 2048          ← MUST match the old start sector
# Last sector: (Enter)        ← uses all remaining space
#
# "Do you want to remove the signature?" → N (No!)
#
# Command: w

# 3. Re-read partition table
partprobe /dev/sda

# 4. Resize filesystem
resize2fs /dev/sda1       # ext4
xfs_growfs /              # XFS
```

> **Warning:** The fdisk method is risky. If you enter the wrong start sector, you will lose all data. The partition type and boot flag may also need to be re-set. `growpart` handles all of this automatically and safely.

---

## Syntax

```bash
growpart [OPTIONS] DISK PARTITION_NUMBER
```

| Option | Description |
|--------|-------------|
| `-n` / `--dry-run` | Show what would be done without making changes |
| `-N` | Same as `--dry-run` |
| `-v` / `--verbose` | Verbose output |
| `-u` / `--update` | Update kernel partition table after growing (`auto`, `force`, `off`, `on`) |
| `--free` | Show free space after the partition (without modifying) |
| `--fudge` | Amount of free space (in sectors) to leave at end |

---

## Basic Usage

### Extend a Partition

```bash
# Extend partition 1 on /dev/sda to fill all available space
growpart /dev/sda 1

# Extend partition 2 on /dev/xvda
growpart /dev/xvda 2

# Extend partition 1 on NVMe device (AWS Nitro instances)
growpart /dev/nvme0n1 1
```

> **Important:** Note the space between the disk and partition number. It's `growpart /dev/sda 1`, not `growpart /dev/sda1`.

### Dry Run (Preview Changes)

```bash
# See what would happen without changing anything
growpart -N /dev/sda 1
growpart --dry-run /dev/sda 1
```

### Check Free Space

```bash
# Show how much space is available to grow into
growpart --free /dev/sda 1
```

---

## Complete Workflow: Extend Disk on Cloud/VM

### Step 1 — Expand the Disk

This step depends on your platform:

| Platform | How to expand |
|----------|--------------|
| AWS EC2 | Console → EBS → Modify Volume → increase size |
| Azure | Portal → Disk → Configuration → increase size |
| GCP | Console → Disks → Edit → increase size |
| VMware | vSphere → VM → Edit Settings → increase disk |
| KVM/libvirt | `virsh blockresize` or `qemu-img resize` |
| VirtualBox | `VBoxManage modifyhd --resize` |

### Step 2 — Rescan the Disk (If Needed)

The guest OS may not see the new size immediately:

```bash
# For SCSI/SATA disks (VMware, Azure, etc.)
echo 1 > /sys/class/block/sda/device/rescan

# For all SCSI hosts
for host in /sys/class/scsi_host/host*; do echo "- - -" > $host/scan; done

# For NVMe (AWS Nitro) — usually automatic, but if not:
echo 1 > /sys/block/nvme0n1/device/rescan

# Verify new disk size
lsblk
fdisk -l /dev/sda

# Get exact size in bytes
blockdev --getsize64 /dev/nvme0n1

# Check partition size via sysfs
cat /sys/block/nvme0n1/nvme0n1p1/size
```

### Step 3 — Extend the Partition

```bash
growpart /dev/sda 1
# Expected output:
# CHANGED: partition=/dev/sda1 start=2048 old: size=41940992 end=41943040 new: size=104855519 end=104857567
```

If growpart reports `NOCHANGE`, the partition already uses the full disk — skip to Step 4 (filesystem resize).

### Step 4 — Resize the Filesystem

```bash
# Check filesystem type first
df -hT /

# ext4
resize2fs /dev/sda1

# XFS (must be mounted — use mount point, not device)
xfs_growfs /
xfs_growfs -d /      # -d flag explicitly grows data section

# XFS on device without partitions
xfs_growfs /dev/xvdb

# Btrfs (must be mounted)
btrfs filesystem resize max /
```

### Step 5 — Verify

```bash
df -h /
lsblk
```

---

## Common Scenarios

### AWS EC2 — Extend Root Volume

```bash
# After expanding EBS volume in AWS console:

# 1. Check current state
lsblk
df -h /

# 2. Grow partition (NVMe instance)
growpart /dev/nvme0n1 1

# 3. Grow filesystem
# For ext4:
resize2fs /dev/nvme0n1p1
# For XFS (Amazon Linux 2):
xfs_growfs /

# 4. Verify
df -h /
```

### AWS EC2 — Non-NVMe (Older Instance Types)

```bash
growpart /dev/xvda 1
resize2fs /dev/xvda1
# or
xfs_growfs /
```

### Azure VM

```bash
# Rescan disk after expanding in portal
echo 1 > /sys/class/block/sda/device/rescan

# Grow partition
growpart /dev/sda 2    # Root is often partition 2 on Azure

# Grow filesystem
resize2fs /dev/sda2
```

### Google Cloud (GCP)

```bash
# 1. Take a snapshot first (from your workstation or Cloud Shell)
gcloud compute disks snapshot DISK_NAME --zone=ZONE --snapshot-names=pre-resize-snap

# 2. Resize the disk at the GCP layer
gcloud compute disks resize DISK_NAME --size=200 --zone=ZONE
# --size is in GiB

# 3. SSH into the VM — kernel usually sees the new size immediately
lsblk

# 4. If disk still shows old size (rare), force rescan
echo 1 > /sys/class/block/sda/device/rescan

# 5. Grow partition
growpart /dev/sda 1

# 6. Grow filesystem
resize2fs /dev/sda1       # ext4
xfs_growfs /              # XFS

# 7. Verify
df -h /
```

> **GCP notes:**
> - Disks **cannot be shrunk** — only grown. The only path to smaller is: create new disk, copy data, swap.
> - **Extreme PD** disks can only be resized once per 6 hours.
> - Maximum disk size: 64 TiB (Balanced, Performance, Standard, Extreme).
> - GCP public images auto-grow the boot partition on first boot after resize. Custom/imported images require manual growpart.
> - NVMe-backed instances (C3, N4, 3rd-gen+) use `/dev/nvme0n1` instead of `/dev/sda`.

### VMware / KVM Guest

```bash
# Rescan disk
echo 1 > /sys/class/block/sda/device/rescan

# Verify new size is visible
fdisk -l /dev/sda

# Grow partition
growpart /dev/sda 1

# Grow filesystem
resize2fs /dev/sda1
```

### LVM — Extend Physical Volume After growpart

If the partition holds an LVM physical volume:

```bash
# 1. Grow the partition
growpart /dev/sda 2

# 2. Resize the PV to use new partition size
pvresize /dev/sda2

# 3. Extend the logical volume (use all free space)
lvextend -l +100%FREE /dev/vg_root/lv_root

# Or extend to a specific size
lvextend -L 50G /dev/vg_root/lv_root

# 4. Resize the filesystem
resize2fs /dev/vg_root/lv_root     # ext4
xfs_growfs /                        # XFS
```

### Swap Partition After Root

If swap is between root and the free space, you cannot simply grow root. Options:

1. Delete swap, grow root, recreate swap at end
2. Use a swap file instead of a swap partition
3. Add a new partition for root data

---

## growpart with cloud-init

On most cloud images, `growpart` runs automatically at first boot via cloud-init's `growpart` module. It reads configuration from:

```yaml
# /etc/cloud/cloud.cfg or /etc/cloud/cloud.cfg.d/

growpart:
  mode: auto          # auto, growpart, gpart, off
  devices:
    - /               # grow the partition containing /
  ignore_growroot_disabled: false
```

To disable automatic grow:

```bash
# Create a file to disable growpart on boot
touch /etc/growroot-disabled

# Or in cloud-init config:
growpart:
  mode: off
```

---

## Troubleshooting

### "NOCHANGE: partition 1 is size X. it cannot be grown"

The partition already uses all available space. Confirm the disk was actually expanded:

```bash
fdisk -l /dev/sda
# Compare "Disk /dev/sda" size with the partition end sector
```

If the disk shows the old size, rescan:

```bash
echo 1 > /sys/class/block/sda/device/rescan
```

### "FAILED: failed to get start and end for /dev/sda1"

Possible causes:
- Wrong syntax (remember: `growpart /dev/sda 1` not `/dev/sda1`)
- Disk has no partition table (filesystem directly on device)
- GPT backup header at end of disk is stale

For GPT stale header:

```bash
# Fix GPT backup header after disk expansion
sgdisk -e /dev/sda
# Then retry
growpart /dev/sda 1
```

### "FAILED: sfdisk failed"

On older systems, growpart uses `sfdisk` internally. This can fail if:
- The partition table is corrupt
- The disk uses a non-standard layout

Try with verbose:

```bash
growpart -v /dev/sda 1
```

### GPT: "Not enough space at end of disk"

GPT stores a backup partition table at the end of the disk. After expanding the disk, the backup is in the wrong position:

```bash
# Move GPT backup header to new end of disk
sgdisk -e /dev/sda

# Or with parted
parted /dev/sda print
# parted will offer to fix the backup GPT — answer "Fix"

# Then grow
growpart /dev/sda 1
```

### resize2fs says "Nothing to do"

The filesystem already fills the partition. Check that growpart actually expanded the partition:

```bash
# Compare partition size vs filesystem size
lsblk /dev/sda1         # partition size
df -h /                  # filesystem size
tune2fs -l /dev/sda1 | grep "Block count"  # filesystem blocks
```

### Permission denied

`growpart` requires root privileges:

```bash
sudo growpart /dev/sda 1
```

If running in a container or restricted environment, ensure the process has `CAP_SYS_ADMIN` capability.

### xfs_growfs shows "input/output error"

`xfs_growfs` requires the **mount point**, not the device path:

```bash
# Wrong — will error
xfs_growfs /dev/nvme0n1p1

# Correct — use mount point
xfs_growfs /
xfs_growfs /data
```

### resize2fs was interrupted

If `resize2fs` was interrupted during operation, simply run it again — it will resume and complete:

```bash
resize2fs /dev/sda1
```

---

## growpart vs Other Tools

| Tool | What it does | Online? | MBR/GPT |
|------|-------------|---------|---------|
| `growpart` | Extends partition to fill free space | ✅ Yes (online) | Both |
| `fdisk` | Delete + recreate partition (same start sector) | ❌ Risky online | MBR + GPT |
| `parted resizepart` | Resize partition entry | ✅ Yes | Both |
| `sgdisk -e` | Move GPT backup header | ✅ Yes | GPT only |
| `pvresize` | Extend LVM physical volume | ✅ Yes | N/A (LVM layer) |
| `resize2fs` | Extend ext4 filesystem | ✅ Yes (online) | N/A (filesystem) |
| `xfs_growfs` | Extend XFS filesystem | ✅ Yes (must be mounted) | N/A (filesystem) |
| `btrfs filesystem resize` | Extend Btrfs filesystem | ✅ Yes (mounted) | N/A (filesystem) |
| `zpool online -e` | Extend ZFS pool device | ✅ Yes (online) | N/A (ZFS) |

---

## Tips

- **Always take a snapshot/backup** before resizing — in case something goes wrong
- **Check filesystem type** before resizing: `df -hT` or `lsblk -f`
- **`lsblk -f`** is the single best command to see device tree, filesystem type, label, UUID, and mountpoint at once
- **AWS device naming** differs between Xen and Nitro instances:
  - Xen (older): `/dev/xvda`, `/dev/xvda1`
  - Nitro (newer): `/dev/nvme0n1`, `/dev/nvme0n1p1`
- Most modern filesystems (ext4, XFS) support **online resizing** without unmounting
- `growpart` only expands the partition table — always resize the filesystem after
- Cloud-init often runs growpart automatically on first boot
- **Disks cannot be shrunk** on any cloud provider (AWS, GCP, Azure) — only grown
- **MBR partition tables cap at 2 TB** — check with:
  ```bash
  parted /dev/sda print | grep "Partition Table"
  # "msdos" = MBR (2 TB limit), "gpt" = GPT (no practical limit)
  ```
- **Don't use fdisk** for resizing unless growpart is unavailable — growpart handles the partition table update safely on its own

---

## Related Commands

```bash
# Check filesystem type
df -hT

# Check filesystem type for a specific mount point
findmnt -T / -o SOURCE,TARGET,FSTYPE

# Check partition table
parted -l

# Show partition information
fdisk -l

# Check current disk usage
lsblk -f
```

---

## Quick Reference

```bash
# Extend partition 1 to fill disk
growpart /dev/sda 1

# NVMe device (AWS)
growpart /dev/nvme0n1 1

# Dry run
growpart -N /dev/sda 1

# After growpart — resize filesystem
resize2fs /dev/sda1          # ext4
xfs_growfs /                  # XFS (mounted)
btrfs filesystem resize max / # Btrfs (mounted)

# LVM after growpart
pvresize /dev/sda2
lvextend -l +100%FREE /dev/vg/lv
resize2fs /dev/vg/lv

# Fix GPT backup header
sgdisk -e /dev/sda

# Rescan disk size in guest
echo 1 > /sys/class/block/sda/device/rescan

# Check available free space
growpart --free /dev/sda 1

# Cloud-init: disable auto-grow
touch /etc/growroot-disabled
```
