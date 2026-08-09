# Extending EBS Volumes and Filesystems on EC2

Resizing an EBS volume on AWS is a live, online operation — no downtime required. The process has three steps: resize the volume in AWS, extend the partition (if one exists), then grow the filesystem.

## Overview

```
1. AWS Console/CLI: Modify EBS volume size
2. Inside the instance: Extend the partition (growpart)
3. Inside the instance: Grow the filesystem (xfs_growfs / resize2fs)
```

## Step 1: Resize the EBS Volume (AWS Side)

### AWS CLI

```bash
# Get volume ID for an instance
aws ec2 describe-instances \
    --instance-ids i-0123456789abcdef0 \
    --query 'Reservations[0].Instances[0].BlockDeviceMappings[*].[DeviceName,Ebs.VolumeId]' \
    --output table

# Modify volume size (e.g., from 20GB to 50GB)
aws ec2 modify-volume \
    --volume-id vol-0123456789abcdef0 \
    --size 50

# Check modification progress
aws ec2 describe-volumes-modifications \
    --volume-ids vol-0123456789abcdef0 \
    --query 'VolumesModifications[*].{ID:VolumeId,State:ModificationState,Progress:Progress,OrigSize:OriginalSize,NewSize:TargetSize}' \
    --output table

# Wait until state is "completed" or "optimizing"
# "optimizing" means you can already use the new space
```

### AWS Console

1. EC2 → Volumes → select the volume
2. Actions → Modify Volume
3. Enter new size → Modify
4. Wait for state to change from "modifying" to "in-use - optimizing" or "in-use"

> **Note:** You can only increase volume size, not decrease. After modification, you must wait 6 hours before modifying the same volume again.

## Step 2: Verify the New Size (Inside the Instance)

SSH into the instance:

```bash
# Check current block device sizes
lsblk

# Example output (before extending partition):
# NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
# nvme0n1       259:0    0   50G  0 disk            <-- disk sees 50G
# ├─nvme0n1p1   259:1    0   20G  0 part /          <-- partition still 20G
# └─nvme0n1p128 259:2    0    1M  0 part

# Or for older instance types:
# xvda          202:0    0   50G  0 disk
# └─xvda1       202:1    0   20G  0 part /
```

The disk shows the new size but the partition and filesystem haven't been extended yet.

## Step 3: Extend the Partition

### Identify Device Names

| Instance Type | Root Device | Partition |
|---------------|-------------|-----------|
| Nitro (t3, m5, c5, etc.) | `/dev/nvme0n1` | `/dev/nvme0n1p1` |
| Older (t2, m4, c4, etc.) | `/dev/xvda` | `/dev/xvda1` |

### Using growpart

```bash
# Install growpart if not present
sudo yum install cloud-utils-growpart    # Amazon Linux / RHEL
sudo apt install cloud-guest-utils       # Ubuntu / Debian

# Extend partition 1 on the disk
sudo growpart /dev/nvme0n1 1

# Or for older instances
sudo growpart /dev/xvda 1

# Verify partition was extended
lsblk
# nvme0n1p1 should now show 50G (or new size minus small overhead)
```

### Without Partitions (Raw Disk)

If the filesystem is directly on the disk (no partition table), skip this step — go directly to Step 4.

```bash
# Check: if lsblk shows no partitions under the disk, it's raw
lsblk
# nvme1n1   259:3    0   50G  0 disk /data   <-- no partitions, raw fs
```

### LVM (Logical Volume Manager)

If using LVM:

```bash
# Extend the partition first
sudo growpart /dev/nvme0n1 2    # assuming LVM is on partition 2

# Extend the physical volume
sudo pvresize /dev/nvme0n1p2

# Extend the logical volume
sudo lvextend -l +100%FREE /dev/mapper/vg-lv_root

# Then grow the filesystem (see Step 4)
```

## Step 4: Grow the Filesystem

### XFS

```bash
# XFS grows using the mount point (not device)
sudo xfs_growfs /

# Or for a data volume
sudo xfs_growfs /data

# Verify
df -h /
```

> **Note:** XFS can only be grown, never shrunk. The filesystem must be mounted to grow it.

### ext4

```bash
# ext4 grows using the device (can be done online)
sudo resize2fs /dev/nvme0n1p1

# Or for a data volume
sudo resize2fs /dev/nvme1n1

# Or for LVM
sudo resize2fs /dev/mapper/vg-lv_root

# Verify
df -h /
```

> **Note:** `resize2fs` can be run on a mounted filesystem (online resize). For ext2/ext3, the same command works.

### Determine Filesystem Type

```bash
# Method 1: df -T
df -Th /
# Filesystem     Type  Size  Used Avail Use% Mounted on
# /dev/nvme0n1p1 xfs    50G   8G   42G  16% /

# Method 2: blkid
sudo blkid /dev/nvme0n1p1
# /dev/nvme0n1p1: UUID="..." TYPE="xfs" ...

# Method 3: lsblk -f
lsblk -f
```

## Complete Examples

### Example 1: Amazon Linux 2023 / RHEL (XFS, Nitro Instance)

```bash
# After modifying volume in AWS:
lsblk
sudo growpart /dev/nvme0n1 1
sudo xfs_growfs /
df -h /
```

### Example 2: Ubuntu (ext4, Nitro Instance)

```bash
# After modifying volume in AWS:
lsblk
sudo growpart /dev/nvme0n1 1
sudo resize2fs /dev/nvme0n1p1
df -h /
```

### Example 3: Older Instance Type (t2, xvda)

```bash
# After modifying volume in AWS:
lsblk
sudo growpart /dev/xvda 1
sudo xfs_growfs /         # if XFS
# or
sudo resize2fs /dev/xvda1 # if ext4
df -h /
```

### Example 4: Additional Data Volume (No Partition)

```bash
# After modifying volume in AWS:
lsblk
# nvme1n1 shows new size, mounted at /data, no partitions

# For XFS:
sudo xfs_growfs /data

# For ext4:
sudo resize2fs /dev/nvme1n1

df -h /data
```

### Example 5: LVM Root Volume

```bash
# After modifying volume in AWS:
lsblk
sudo growpart /dev/nvme0n1 2
sudo pvresize /dev/nvme0n1p2
sudo lvextend -l +100%FREE /dev/mapper/rootvg-rootlv
sudo xfs_growfs /                    # XFS
# or
sudo resize2fs /dev/mapper/rootvg-rootlv   # ext4
df -h /
```

### Example 6: NVMe Additional Volume (Data Partition)

```bash
# After modifying volume in AWS (e.g., /dev/nvme1n1 mounted at /data with partition):
lsblk
sudo growpart /dev/nvme1n1 1
sudo xfs_growfs /data        # XFS
# or
sudo resize2fs /dev/nvme1n1p1   # ext4
df -h /data
```

## NVMe Device Mapping

On Nitro instances, EBS devices show as `/dev/nvme*`. Map them to their EBS volume IDs:

```bash
# Install NVMe CLI
sudo yum install -y nvme-cli    # Amazon Linux / RHEL
sudo apt install -y nvme-cli    # Ubuntu

# List NVMe devices with volume IDs
sudo nvme list

# Get volume ID for a specific device
sudo nvme id-ctrl -v /dev/nvme1n1 | grep sn
# The serial number (sn) is the volume ID without "vol-" prefix and dashes

# Alternative: check EBS device name mapping
cat /dev/disk/by-id/ | ls -la
ls -la /dev/disk/by-id/ | grep nvme
```

## Scripted Resize (All-in-One)

```bash
#!/bin/bash
# resize-root.sh — Extend root volume filesystem after AWS resize
# Run as root

set -e

# Detect root device and filesystem
ROOT_DEVICE=$(findmnt -no SOURCE /)
ROOT_DISK=$(lsblk -ndo PKNAME $ROOT_DEVICE)
ROOT_PART=$(echo $ROOT_DEVICE | grep -o '[0-9]*$')
FS_TYPE=$(findmnt -no FSTYPE /)

echo "Root device: $ROOT_DEVICE"
echo "Root disk: /dev/$ROOT_DISK"
echo "Partition: $ROOT_PART"
echo "Filesystem: $FS_TYPE"

# Extend partition
if [ -n "$ROOT_PART" ]; then
    echo "Extending partition $ROOT_PART on /dev/$ROOT_DISK..."
    growpart /dev/$ROOT_DISK $ROOT_PART
fi

# Grow filesystem
case $FS_TYPE in
    xfs)
        echo "Growing XFS filesystem..."
        xfs_growfs /
        ;;
    ext4|ext3|ext2)
        echo "Growing ext filesystem..."
        resize2fs $ROOT_DEVICE
        ;;
    *)
        echo "Unknown filesystem type: $FS_TYPE"
        exit 1
        ;;
esac

echo "Done. New size:"
df -h /
```

## Troubleshooting

### growpart: "NOCHANGE: partition 1 is size X"

The partition already uses all available space — the volume modification may not have completed yet:

```bash
# Check volume modification state from inside
lsblk
# If disk size hasn't changed, wait and check again
# Or from AWS CLI:
aws ec2 describe-volumes-modifications --volume-ids vol-xxx
```

### resize2fs: "bad magic number in super-block"

You're running resize2fs on an XFS filesystem (or the wrong device):

```bash
# Check filesystem type
blkid /dev/nvme0n1p1
# Use xfs_growfs for XFS, resize2fs for ext4
```

### xfs_growfs: "is not a mounted XFS filesystem"

The target must be a mount point, not a device:

```bash
# Wrong
sudo xfs_growfs /dev/nvme0n1p1

# Right
sudo xfs_growfs /
```

### "No space left on device" Despite New Size

The filesystem hasn't been grown yet:

```bash
# Check: lsblk shows large disk/partition, df shows small filesystem
lsblk    # shows 50G partition
df -h /  # shows 20G filesystem

# Solution: grow the filesystem
sudo xfs_growfs /
# or
sudo resize2fs /dev/nvme0n1p1
```

### Volume Stuck in "modifying" State

Wait. Large volumes can take several hours to optimize. The space is usable once state is "optimizing" (not just "completed").

### Cannot Modify Volume ("already being modified")

You must wait 6 hours after the last modification before modifying again. Check:

```bash
aws ec2 describe-volumes-modifications --volume-ids vol-xxx \
    --query 'VolumesModifications[0].{State:ModificationState,Start:StartTime}'
```

## Quick Reference

| Filesystem | Grow Command | Notes |
|-----------|-------------|-------|
| XFS | `xfs_growfs /mountpoint` | Must be mounted, uses mount point |
| ext4 | `resize2fs /dev/device` | Can be mounted (online), uses device |
| ext3 | `resize2fs /dev/device` | Same as ext4 |
| ext2 | `resize2fs /dev/device` | Same as ext4 |

| Step | Command | When |
|------|---------|------|
| Resize volume | `aws ec2 modify-volume --size N` | AWS side |
| Extend partition | `growpart /dev/disk N` | If partitioned |
| Extend PV (LVM) | `pvresize /dev/partition` | If using LVM |
| Extend LV (LVM) | `lvextend -l +100%FREE /dev/vg/lv` | If using LVM |
| Grow XFS | `xfs_growfs /mountpoint` | XFS filesystem |
| Grow ext4 | `resize2fs /dev/device` | ext4 filesystem |
