# Disk Health & Maintenance

Commands for monitoring disk health, checking hardware, performance tuning, secure wiping, cloning, and SSD optimization on RHEL and Ubuntu.

## Hardware Discovery

```bash
# List all block devices (tree format)
lsblk
lsblk -f                           # Include filesystem info
lsblk -o NAME,MODEL,SERIAL,SIZE    # Show model and serial numbers
lsblk -o NAME,TRAN,SIZE,ROTA       # Show transport and rotation (SSD vs HDD)

# Detailed hardware information
sudo lshw -class disk
sudo lshw -class disk -short

# List SCSI devices
lsscsi
lsscsi -s                           # Include device size

# Show disk hardware details
sudo hdparm -i /dev/sda             # Identify info (model, serial, firmware)
sudo hdparm -I /dev/sda             # Detailed identification (capabilities)

# Show disk geometry
fdisk -l /dev/sda | head -5

# Physical disk info from kernel
cat /proc/partitions
cat /proc/scsi/scsi

# Check if disk is SSD or HDD
cat /sys/block/sda/queue/rotational
# 0 = SSD, 1 = HDD

# Show disk scheduler
cat /sys/block/sda/queue/scheduler
```

## SMART Monitoring

S.M.A.R.T. (Self-Monitoring, Analysis and Reporting Technology) detects failing disks before they die.

### Install

```bash
# RHEL / CentOS / Rocky
sudo dnf install smartmontools

# Ubuntu / Debian
sudo apt install smartmontools
```

### Check Disk Health

```bash
# Quick health status (PASS/FAIL)
sudo smartctl -H /dev/sda

# Full SMART info (attributes, errors, capabilities)
sudo smartctl -a /dev/sda

# Show only attributes
sudo smartctl -A /dev/sda

# Show error log
sudo smartctl -l error /dev/sda

# Show self-test log
sudo smartctl -l selftest /dev/sda

# Check if SMART is supported and enabled
sudo smartctl -i /dev/sda

# Enable SMART on a disk
sudo smartctl -s on /dev/sda
```

### Run SMART Tests

```bash
# Short test (~2 minutes)
sudo smartctl -t short /dev/sda

# Long test (hours — depends on disk size)
sudo smartctl -t long /dev/sda

# Conveyance test (after shipping/transport)
sudo smartctl -t conveyance /dev/sda

# Check test progress
sudo smartctl -l selftest /dev/sda

# Cancel a running test
sudo smartctl -X /dev/sda
```

### Key SMART Attributes

| Attribute | ID | Warning Sign |
|-----------|----|----|
| `Reallocated_Sector_Ct` | 5 | > 0 means bad sectors remapped |
| `Current_Pending_Sector` | 197 | Sectors waiting to be remapped |
| `Offline_Uncorrectable` | 198 | Sectors that can't be read |
| `UDMA_CRC_Error_Count` | 199 | Cable/connection problems |
| `Wear_Leveling_Count` | 177 | SSD wear (lower = more worn) |
| `Reallocated_Event_Count` | 196 | Number of remap operations |
| `Temperature_Celsius` | 194 | High temp = reduced lifespan |

```bash
# Check for concerning attributes
sudo smartctl -A /dev/sda | grep -E "Reallocated|Pending|Offline_Uncorrectable"
```

### Automatic SMART Monitoring (smartd)

```bash
# Enable smartd daemon
sudo systemctl enable --now smartd

# Configuration
sudo vi /etc/smartd.conf
# Monitor all disks, run short test daily, long test weekly:
# DEVICESCAN -a -o on -S on -s (S/../.././02|L/../../6/03) -m admin@example.com

# Restart after config change
sudo systemctl restart smartd
```

## Disk Temperature

```bash
# Using smartctl
sudo smartctl -A /dev/sda | grep Temperature

# Using hddtemp (deprecated but still available)
sudo hddtemp /dev/sda

# Using nvme-cli for NVMe drives
sudo nvme smart-log /dev/nvme0n1 | grep temperature

# All disks temperature
for disk in /dev/sd?; do
    echo -n "$disk: "
    sudo smartctl -A "$disk" 2>/dev/null | grep Temperature | awk '{print $10 "°C"}'
done
```

## Bad Sector Detection

```bash
# Read-only check (safe on mounted filesystems)
sudo badblocks -v /dev/sda

# Read-only with progress
sudo badblocks -sv /dev/sda

# Non-destructive read-write test (UNMOUNT FIRST — safe but slow)
sudo badblocks -nsv /dev/sda

# Destructive write test (DESTROYS ALL DATA — fastest, most thorough)
sudo badblocks -wsv /dev/sda

# Output bad blocks to file (for use with fsck)
sudo badblocks -o /tmp/bad-blocks.txt /dev/sda

# Add bad blocks to ext4 filesystem
sudo e2fsck -l /tmp/bad-blocks.txt /dev/sda1
```

> **Warning:** `badblocks -w` destroys all data. Only use on empty disks or when you've already backed up.

## Disk Performance Testing

```bash
# Buffered read speed (tests cache + disk)
sudo hdparm -t /dev/sda

# Cache read speed only
sudo hdparm -T /dev/sda

# Both together
sudo hdparm -tT /dev/sda

# Raw read speed with dd
sudo dd if=/dev/sda of=/dev/null bs=1M count=1024 status=progress

# Write speed test (to a file, not raw device)
dd if=/dev/zero of=/tmp/testfile bs=1M count=1024 oflag=direct status=progress
rm /tmp/testfile

# Random read IOPS with fio
sudo fio --name=randread --ioengine=libaio --iodepth=32 --rw=randread \
  --bs=4k --direct=1 --size=1G --numjobs=4 --filename=/dev/sdb --group_reporting

# Sequential write with fio
sudo fio --name=seqwrite --ioengine=libaio --iodepth=16 --rw=write \
  --bs=1M --direct=1 --size=4G --numjobs=1 --filename=/tmp/fio-test
```

## Disk Cloning

```bash
# Clone entire disk (same size or larger target)
sudo dd if=/dev/sda of=/dev/sdb bs=64K status=progress

# Clone with progress using pv
sudo dd if=/dev/sda | pv -s $(sudo blockdev --getsize64 /dev/sda) | sudo dd of=/dev/sdb bs=64K

# Clone single partition
sudo dd if=/dev/sda1 of=/dev/sdb1 bs=64K status=progress

# Create compressed disk image
sudo dd if=/dev/sda bs=64K status=progress | gzip > /backup/sda.img.gz

# Restore from compressed image
gunzip -c /backup/sda.img.gz | sudo dd of=/dev/sda bs=64K status=progress

# Clone with rescue (skip bad sectors)
sudo ddrescue /dev/sda /dev/sdb /tmp/rescue.log

# Sync and verify after clone
sync
sudo cmp /dev/sda /dev/sdb    # Byte-by-byte comparison (slow)
```

## Secure Disk Wiping

```bash
# Zero out (fast, single pass — sufficient for most cases)
sudo dd if=/dev/zero of=/dev/sdb bs=1M status=progress

# Random data (stronger — single pass)
sudo dd if=/dev/urandom of=/dev/sdb bs=1M status=progress

# NIST 800-88 compliant wipe (3 passes: random, random, zeros)
sudo shred -vfz -n 3 /dev/sdb
# -v = verbose
# -f = force (change permissions if needed)
# -z = final pass with zeros (hide the shredding)
# -n 3 = number of random passes

# Quick wipe (remove filesystem signatures only — data still recoverable)
sudo wipefs -a /dev/sdb

# ATA Secure Erase (fastest for SSDs — hardware-level)
sudo hdparm --user-master u --security-set-pass password /dev/sdb
sudo hdparm --user-master u --security-erase password /dev/sdb
# Takes seconds on SSD, minutes on HDD

# NVMe Secure Erase
sudo nvme format /dev/nvme0n1 --ses=1    # User data erase
sudo nvme format /dev/nvme0n1 --ses=2    # Cryptographic erase
```

### Wiping Comparison

| Method | Speed | Security | Use Case |
|--------|-------|----------|----------|
| `wipefs -a` | Instant | Low (data recoverable) | Quick reformat |
| `dd if=/dev/zero` | Moderate | Medium (1 pass) | Reuse within org |
| `shred -n 1` | Moderate | Good (1 random pass) | General disposal |
| `shred -n 3 -z` | Slow | High (3+1 passes) | Sensitive data |
| ATA Secure Erase | Fast | High (hardware) | SSD disposal |
| NVMe format --ses=2 | Instant | Very High (crypto) | NVMe SSD disposal |

## SSD Optimization

### Set I/O Scheduler

```bash
# Check current scheduler
cat /sys/block/sda/queue/scheduler

# Set scheduler (runtime — not persistent)
echo none > /sys/block/sda/queue/scheduler     # Best for NVMe
echo mq-deadline > /sys/block/sda/queue/scheduler  # Good for SATA SSD

# Make persistent via udev rule
cat <<'EOF' | sudo tee /etc/udev/rules.d/60-scheduler.rules
# Set scheduler for NVMe
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/scheduler}="none"
# Set scheduler for SATA SSD
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"
# Set scheduler for HDD
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="bfq"
EOF
sudo udevadm control --reload-rules
```

### TRIM / Discard

```bash
# Manual TRIM (run periodically or via timer)
sudo fstrim -v /
sudo fstrim -av    # All mounted filesystems

# Enable periodic TRIM via systemd timer
sudo systemctl enable --now fstrim.timer
sudo systemctl status fstrim.timer

# Enable continuous TRIM (in fstab — less recommended, slight performance impact)
# /dev/sda1  /  ext4  defaults,discard  0  1

# Enable discard on ext4 (without fstab)
sudo tune2fs -o discard /dev/sda1

# Check if TRIM is supported
sudo hdparm -I /dev/sda | grep TRIM
lsblk -D    # Shows DISC-GRAN and DISC-MAX (non-zero = TRIM supported)
```

### Read-Ahead Tuning

```bash
# Show current read-ahead (in 512-byte sectors)
sudo blockdev --getra /dev/sda

# Set read-ahead (higher for sequential workloads)
sudo blockdev --setra 4096 /dev/sda    # 2MB read-ahead
sudo blockdev --setra 256 /dev/sda     # 128K (better for random I/O)

# Make persistent via udev
echo 'ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/read_ahead_kb}="128"' | \
  sudo tee /etc/udev/rules.d/61-readahead.rules
```

### SSD Mount Options

```bash
# Recommended fstab options for SSD
# noatime — don't update access time (reduces writes)
# discard — enable continuous TRIM (or use fstrim.timer instead)
/dev/sda1  /  ext4  defaults,noatime  0  1

# XFS on SSD
/dev/sda1  /  xfs  defaults,noatime,inode64  0  0
```

## Force Filesystem Check on Next Boot

```bash
# ext4 — create marker file
sudo touch /forcefsck

# Or set the mount count to trigger fsck
sudo tune2fs -C 100 /dev/sda1    # Set mount count high
sudo tune2fs -c 1 /dev/sda1      # Set max mount count to 1 (forces check)

# Verify settings
sudo tune2fs -l /dev/sda1 | grep -E "Mount count|Maximum mount count"

# XFS — cannot be forced the same way (use xfs_repair after unmounting)
# Boot to rescue mode and run:
sudo xfs_repair /dev/sda1

# systemd: force fsck on specific unit
sudo systemctl edit systemd-fsck-root.service
```

## Emergency Commands

```bash
# Force read-only remount (prevent further damage)
sudo mount -o remount,ro /

# Emergency sync (flush all buffers to disk)
sync

# Show processes using a device/mount
sudo lsof +D /mnt/data
sudo fuser -vm /mnt/data

# Kill processes using a device
sudo fuser -km /mnt/data

# Force unmount
sudo umount -f /mnt/data
sudo umount -l /mnt/data    # Lazy unmount (detaches immediately)
```

## Monitoring Script

```bash
#!/bin/bash
# disk_health_check.sh — quick health overview

echo "=== Disk Health Report — $(date) ==="
echo ""

# List disks
echo "--- Disks ---"
lsblk -o NAME,SIZE,ROTA,TRAN,MODEL | grep -v "^├\|^└\|loop"
echo ""

# Check SMART for all disks
echo "--- SMART Status ---"
for disk in /dev/sd? /dev/nvme?n1; do
    [ -b "$disk" ] || continue
    status=$(sudo smartctl -H "$disk" 2>/dev/null | grep "result" | awk -F: '{print $2}')
    temp=$(sudo smartctl -A "$disk" 2>/dev/null | grep Temperature | head -1 | awk '{print $10}')
    echo "$disk: Health=${status:-N/A}  Temp=${temp:-N/A}°C"
done
echo ""

# Check disk space
echo "--- Space Usage ---"
df -h | grep -vE "tmpfs|loop|udev"
echo ""

# Check for concerning SMART attributes
echo "--- Warnings ---"
for disk in /dev/sd?; do
    [ -b "$disk" ] || continue
    issues=$(sudo smartctl -A "$disk" 2>/dev/null | grep -E "Reallocated|Pending|Offline_Uncorrect" | awk '$10 > 0 {print $2 "=" $10}')
    [ -n "$issues" ] && echo "WARNING $disk: $issues"
done
```

## Quick Reference

```bash
# Health
smartctl -H /dev/sda            # Quick health check
smartctl -a /dev/sda            # Full SMART report
smartctl -t short /dev/sda      # Run short test

# Hardware info
lsblk -o NAME,MODEL,SERIAL,SIZE
lshw -class disk -short
hdparm -I /dev/sda
lsscsi

# Performance
hdparm -tT /dev/sda             # Throughput test
fstrim -av                      # TRIM all SSDs
blockdev --setra 4096 /dev/sda  # Set read-ahead

# Cloning
dd if=/dev/sda of=/dev/sdb bs=64K status=progress
ddrescue /dev/sda /dev/sdb /tmp/log

# Wiping
wipefs -a /dev/sdb              # Remove signatures only
shred -vfz -n 3 /dev/sdb       # Secure wipe
hdparm --security-erase ...     # ATA secure erase

# Repair
touch /forcefsck                # Force fsck on next boot
badblocks -sv /dev/sda          # Scan for bad sectors
mount -o remount,ro /           # Emergency read-only
```
