# Resize a Partition on Proxmox

This guide covers resizing a VM disk in Proxmox VE and expanding the partition and filesystem inside the guest — for example, growing a 20 GB disk to 30 GB.

## Overview

Resizing a disk on Proxmox involves two steps:

1. **Host side (Proxmox)** — increase the virtual disk size
2. **Guest side (inside the VM)** — grow the partition and filesystem to use the new space

## Step 1: Resize the Disk in Proxmox

### Via the Web UI

1. Select the VM → **Hardware**
2. Select the disk (e.g., `scsi0`)
3. Click **Resize disk**
4. Enter the amount to add (e.g., `10` for 10 GB)
5. Click **Resize**

### Via the CLI

```bash
# Add 10 GB to the disk (20 GB → 30 GB)
qm resize <vmid> scsi0 +10G

# Or set the total size directly
qm resize <vmid> scsi0 30G
```

> **Note:** `qm resize` can be done while the VM is running (online resize). No shutdown needed.

### Verify from the Host

```bash
# Check the disk size
qm config <vmid> | grep scsi0
```

## Step 2: Grow the Partition Inside the Guest

After Proxmox resizes the virtual disk, the guest OS still sees the old partition table. You need to expand the partition and filesystem inside the VM.

### Check Current State

```bash
# See the disk size (should show new size)
lsblk

# Example output:
# NAME                  MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
# sda                     8:0    0   30G  0 disk
# ├─sda1                  8:1    0    1G  0 part /boot
# └─sda2                  8:2    0   19G  0 part
#   ├─rootvg-root       253:0    0   17G  0 lvm  /
#   └─rootvg-swap       253:1    0    2G  0 lvm  [SWAP]

# Note: sda shows 30G but sda2 is still 19G — needs expanding
```

### Scenario A: Simple Partition (No LVM)

If the disk uses a simple partition layout (e.g., cloud images with `/dev/sda1` as root):

```bash
# Grow the partition to fill available space
growpart /dev/sda 2    # grows partition 2

# Resize the filesystem
# For ext4:
resize2fs /dev/sda2

# For XFS:
xfs_growfs /
```

### Scenario B: LVM (Common on RHEL/CentOS)

If the disk uses LVM (typical layout: `/dev/sda1` = `/boot`, `/dev/sda2` = LVM PV):

#### 1. Grow the Partition

```bash
# Install growpart if not present
dnf install -y cloud-utils-growpart    # RHEL 8+
apt install -y cloud-guest-utils       # Ubuntu

# Grow partition 2 to fill the disk
growpart /dev/sda 2
```

#### 2. Resize the Physical Volume

```bash
# Tell LVM the PV is larger
pvresize /dev/sda2

# Verify
pvs
# PV         VG     Fmt  Attr PSize   PFree
# /dev/sda2  rootvg lvm2 a--  <29.00g  10.00g   ← 10G free now
```

#### 3. Extend the Logical Volume

```bash
# Extend the root LV to use all free space
lvextend -l +100%FREE /dev/rootvg/root

# Or extend by a specific amount
lvextend -L +10G /dev/rootvg/root

# Verify
lvs
```

#### 4. Resize the Filesystem

```bash
# For ext4:
resize2fs /dev/rootvg/root

# For XFS:
xfs_growfs /

# Verify
df -h /
```

### Scenario C: GPT + UEFI (Partition at End of Disk)

If there's a GPT protective partition or the partition table doesn't extend to the end:

```bash
# Fix GPT to use full disk
parted /dev/sda print
# If it asks to fix, answer "Fix"

# Or use sgdisk
sgdisk -e /dev/sda

# Then grow the partition
growpart /dev/sda 2
```

### Scenario D: Ubuntu Cloud Image (Single Partition)

Ubuntu cloud images typically have a simple layout:

```bash
# Disk layout: sda1=/boot/efi, sda2=root (ext4)
growpart /dev/sda 2
resize2fs /dev/sda2

# Verify
df -h /
```

## Complete Example: RHEL 9 VM (20 GB → 30 GB)

### From Proxmox Host

```bash
# Resize the disk (VM can be running)
qm resize 201 scsi0 +10G
```

### Inside the Guest

```bash
# Check current state
lsblk
df -h

# Grow the partition
growpart /dev/sda 2

# Resize PV
pvresize /dev/sda2

# Extend LV
lvextend -l +100%FREE /dev/mapper/rootvg-root

# Resize filesystem (XFS on RHEL 9)
xfs_growfs /

# Verify
df -h /
lsblk
```

## Complete Example: Ubuntu 22.04 Cloud Image (20 GB → 30 GB)

### From Proxmox Host

```bash
qm resize 202 scsi0 +10G
```

### Inside the Guest

```bash
# Check
lsblk
df -h

# Grow partition and filesystem
growpart /dev/sda 2
resize2fs /dev/sda2

# Verify
df -h /
```

## Online vs Offline Resize

| Method | Supported | Notes |
|--------|-----------|-------|
| Online (VM running) | Yes | `qm resize` + growpart + resize2fs/xfs_growfs all work online |
| Offline (VM stopped) | Yes | Safer for complex layouts, required for shrinking |

> **Online resize works for extending.** Shrinking a disk is not supported by Proxmox and requires complex offline procedures.

## Resize with virtio-scsi (Automatic in Some Images)

Some cloud images with `cloud-init` and `growpart` configured will automatically expand on reboot:

```bash
# If growpart is in cloud-init config, just reboot after qm resize
qm resize <vmid> scsi0 +10G
qm reboot <vmid>

# Check after reboot
ssh admin@vm "df -h /"
```

Cloud images with this in `/etc/cloud/cloud.cfg`:

```yaml
growpart:
  mode: auto
  devices: ['/']
```

## Shrinking a Disk (Not Recommended)

Proxmox does not support shrinking disks via `qm resize`. If you need to shrink:

1. Create a new smaller disk
2. Copy data from the old disk to the new one
3. Detach old disk, attach new disk

```bash
# This is complex and risky — backup first!
# Better approach: create a new VM with the desired size
```

## VirtIO Disk Names

Proxmox VMs using VirtIO SCSI or VirtIO block show disks as `/dev/vda` (not `/dev/sda`):

| Bus Type | Device Name |
|----------|-------------|
| VirtIO SCSI (`virtio-scsi-single`) | `/dev/sda` |
| VirtIO Block (`virtio`) | `/dev/vda` |
| IDE | `/dev/sda` |

Adjust commands accordingly:

```bash
# VirtIO block disks
growpart /dev/vda 1
resize2fs /dev/vda1

# VirtIO SCSI disks
growpart /dev/sda 2
resize2fs /dev/sda2
```

## Alternative: Manual Resize with parted

If `growpart` doesn't work or isn't available:

```bash
parted /dev/vda
```

Inside parted:

```
(parted) resizepart 1 100%
(parted) quit
```

Then resize the filesystem:

```bash
# ext4
resize2fs /dev/vda1

# XFS
xfs_growfs /
```

## Notes

- **Snapshot before resizing** — always take a snapshot or backup before resizing in production
- **Partitions "not in disk order"** — some cloud images have `vda14`/`vda15` (BIOS/EFI boot partitions) starting before `vda1`. This is fine — the free space sits after `vda1`'s end sector, so nothing blocks the expansion
- **No reboot needed** — all resize operations (growpart, pvresize, lvextend, resize2fs, xfs_growfs) work online

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| `lsblk` shows old size | Kernel hasn't re-read partition table | Run `partprobe` or reboot |
| `growpart` says "NOCHANGE" | Partition already fills the disk | Check `lsblk` — maybe already grown |
| `pvresize` says "no space" | Partition not grown yet | Run `growpart` first |
| `resize2fs: bad magic number` | Not ext4 (probably XFS) | Use `xfs_growfs /` instead |
| `xfs_growfs: not a mounted XFS` | Wrong device or not mounted | Use the mount point: `xfs_growfs /` |
| GPT backup header at wrong location | GPT not fixed after resize | Run `parted /dev/sda print` and fix |

## Quick Reference

```bash
# Host: resize the disk
qm resize <vmid> scsi0 +10G

# Guest: simple partition (no LVM)
growpart /dev/sda 2
resize2fs /dev/sda2          # ext4
xfs_growfs /                 # XFS

# Guest: LVM
growpart /dev/sda 2
pvresize /dev/sda2
lvextend -l +100%FREE /dev/mapper/rootvg-root
resize2fs /dev/mapper/rootvg-root    # ext4
xfs_growfs /                         # XFS

# Verify
df -h /
lsblk
```
