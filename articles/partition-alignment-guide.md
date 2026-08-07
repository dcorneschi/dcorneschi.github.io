# Partition Alignment Guide

Partition alignment ensures that filesystem blocks line up with the underlying physical storage boundaries. Misalignment causes degraded performance on SSDs, 4K-sector (Advanced Format) HDDs, and RAID volumes — writes that should hit a single physical sector or page end up spanning two, triggering read-modify-write penalties.

## Why Alignment Matters

### The Problem

Traditional partitioning (pre-2010) started the first partition at LBA sector 63 (the 64th sector). With 512-byte logical sectors, that places the partition start at byte offset 32,256 — which is **not** aligned to any 4096-byte boundary.

When a 4K filesystem block is written at a misaligned offset, the underlying device must:

1. Read two physical sectors/pages (the one that contains the start and the one that contains the end)
2. Modify the relevant portions
3. Write both physical sectors/pages back

This **read-modify-write** penalty can reduce write performance by up to 25x for small I/O operations.

### Who Is Affected

| Device type | Internal unit | Aligned to |
|-------------|--------------|------------|
| Traditional HDD (512-byte sectors) | 512 bytes | Any offset (no penalty) |
| Advanced Format HDD (4Kn / 512e) | 4096 bytes physical, 512 bytes logical | 4096-byte boundary |
| SSD | 4096–8192 byte pages, 512 KB–4 MB erase blocks | Page boundary (minimum), erase block (optimal) |
| NVMe | 4096-byte pages | Page boundary |
| RAID volume | Stripe size (64K, 128K, 256K, etc.) | Stripe boundary |

### The Solution

Align all partitions to **1 MiB (1,048,576 bytes = 2048 sectors of 512 bytes)**. This is a conservative approach that satisfies:

- 4096-byte sectors (1 MiB / 4096 = 256 — evenly divisible)
- 8192-byte SSD pages (1 MiB / 8192 = 128 — evenly divisible)
- Common RAID stripe sizes (64K, 128K, 256K, 512K — all divide into 1 MiB)

All modern partitioning tools default to 1 MiB alignment since approximately 2010.

---

## Checking Alignment

### Verify Current Partitions

```bash
# Method 1: fdisk — check start sector
fdisk -l -u /dev/sda
# Start sector divisible by 2048 = aligned to 1 MiB

# Method 2: sfdisk — dump partition table
sfdisk -d /dev/sda
# Check start= values; divisible by 2048 = aligned

# Method 3: parted — built-in alignment check
parted /dev/sda align-check optimal 1

# Method 4: calculate manually
# Start sector × 512 bytes = byte offset
# If byte offset % 4096 == 0 → aligned to 4K
# If byte offset % 1048576 == 0 → aligned to 1 MiB
```

### Example: Checking Alignment

```bash
$ fdisk -l -u /dev/sda

Device     Boot   Start        End    Sectors   Size Id Type
/dev/sda1  *       2048   39063551   39061504  18.6G 83 Linux
/dev/sda2      39065600   43063295    3997696   1.9G 82 Linux swap
/dev/sda3      43065344 1800876031 1757810688 838.4G 83 Linux
```

Check: `2048 % 2048 = 0` ✅, `39065600 % 2048 = 0` ✅, `43065344 % 2048 = 0` ✅

All partitions are aligned to 1 MiB.

### Example: Misaligned Partition (Legacy)

```bash
Device     Boot Start       End    Blocks  Id System
/dev/sdb1          63  20980889  10490413  83 Linux
```

`63 % 2048 = 63` ❌ — starts at byte offset 32,256, not aligned to 4K or 1 MiB.

### Quick Alignment Check Script

```bash
# Check all partitions on all disks
for part in /sys/class/block/sd*/; do
    START=$(cat ${part}start 2>/dev/null)
    if [ -n "$START" ] && [ "$START" -gt 0 ]; then
        ALIGNED=$((START % 2048))
        NAME=$(basename $part)
        echo "$NAME: start=$START, aligned=$([[ $ALIGNED -eq 0 ]] && echo 'yes' || echo 'NO')"
    fi
done

# Or check a specific disk
for part in /dev/sda*; do
    START=$(cat /sys/class/block/$(basename $part)/start 2>/dev/null)
    if [ -n "$START" ]; then
        ALIGNED=$((START % 2048))
        echo "$part: start=$START, aligned=$([[ $ALIGNED -eq 0 ]] && echo 'yes' || echo 'NO')"
    fi
done
```

### Symptoms of Misalignment

- Write throughput significantly below device capability
- High I/O wait (`%iowait` in `top` / `iostat`)
- RAID rebuild times much longer than expected
- `await` values in `iostat -x` consistently elevated under write load
- Write amplification on SSDs (check via `smartctl -A` → `Total_LBAs_Written`)

### Performance Impact of Misaligned Partitions

| Device Type | Operation | Penalty |
|-------------|-----------|---------|
| 4Kn / 512e HDD | Every sub-4K or misaligned write | Read-modify-write: 1 write becomes 1 read + 1 write (2–3× slower) |
| SSD | Every misaligned write | Crosses page boundary: 2 pages written instead of 1. Doubles write amplification, accelerates NAND wear. |
| RAID 5/6 | Every misaligned write crossing stripe boundary | Full-stripe read + parity recalculation + write. Can be up to 25× slower for small random writes. |
| NVMe | Minimal for aligned I/O; misaligned sub-4K writes | Kernel must perform software read-modify-write in page cache, adding CPU overhead. |

**Concrete numbers:**
- IBM benchmarks showed up to **25× write performance degradation** for small misaligned writes on 4K-sector drives
- Database workloads (small random writes) are most affected — sequential large I/O is less impacted because it naturally spans full blocks
- SSD lifespan is reduced because write amplification increases internal erase/write cycles beyond what the workload would normally generate
- On RAID arrays, a single 4K write crossing a stripe boundary can trigger I/O on every disk in the array instead of just one

**What does NOT change:**
- Read performance is usually unaffected on HDDs (the controller just reads an extra sector and discards the excess)
- On SSDs, misaligned reads still hit page cache and are served from RAM in most cases
- Large sequential writes (>= stripe size) are naturally aligned regardless of partition offset
- If your workload is read-heavy with large sequential I/O, misalignment may have negligible impact

---

## How Modern Tools Handle Alignment

### fdisk (util-linux 2.17.2+, RHEL 7+, Ubuntu 12.04+)

Modern fdisk aligns to 1 MiB by default when DOS compatibility mode is disabled (which is the default on all modern versions):

```bash
fdisk /dev/sdb
# First sector defaults to 2048 (= 1 MiB offset)
# Partition automatically aligned
```

On older fdisk versions that still default to DOS compatibility mode:

```bash
# Disable DOS compatibility and use sector units
fdisk -c -u /dev/sdb
```

### fdisk on Very Old Systems (RHEL 5/6, pre-2.17)

Force 1 MiB alignment with explicit geometry:

```bash
# -S 32 -H 64: 32 sectors/track × 64 heads × 512 bytes = 1 MiB per cylinder
fdisk -S 32 -H 64 /dev/sdb

# Start the first partition at cylinder 2 (byte offset 1 MiB)
# Cylinder 1 = first MiB (contains MBR + partition table)
# Cylinder 2 = second MiB = start of first partition
```

### parted (All Versions)

parted aligns automatically when you specify sizes in MiB/GiB:

```bash
# This is aligned (1MiB start)
parted -s /dev/sdb mklabel gpt
parted -s /dev/sdb mkpart primary ext4 1MiB 100%

# Verify
parted /dev/sdb align-check optimal 1
# 1 aligned
```

### sgdisk (GPT)

sgdisk uses sector 2048 as default start (1 MiB alignment):

```bash
# Aligned by default (start=0 means "first available aligned sector")
sgdisk -n 1:0:+10G /dev/sdb
```

---

## LVM Alignment

LVM's Physical Extent (PE) start position determines alignment for logical volumes.

| LVM Version | Default PE start alignment | Notes |
|-------------|---------------------------|-------|
| < 2.02.73 (Aug 2010) | 64 KiB | Aligns to 4K pages but not to large RAID stripes |
| ≥ 2.02.73 | 1 MiB | Aligned for all modern devices |

```bash
# Check PE start alignment
pvs -o+pe_start /dev/sdb1

# Check PE size
vgdisplay vg_data | grep "PE Size"

# Force 1 MiB alignment explicitly (usually not needed on modern LVM)
pvcreate --dataalignment 1m /dev/sdb1
```

As long as the underlying partition is aligned (starts at sector 2048) and LVM is version 2.02.73+, logical volumes inherit proper alignment automatically.

---

## Software RAID (mdadm) Alignment

| Metadata Version | Data start | Aligned? |
|-----------------|-----------|----------|
| 0.90 | Byte 0 of device (superblock at end) | Depends on partition alignment |
| 1.0 | Byte 0 of device (superblock at end) | Depends on partition alignment |
| 1.1 | After superblock (≈ 1 KiB from start) | Usually not 1 MiB aligned |
| 1.2 (default) | 1 MiB offset (superblock at start) | ✅ Always 1 MiB aligned |

```bash
# Check data offset
mdadm --examine /dev/sdb1 | grep "Data Offset"

# Force data offset to 2048 sectors (1 MiB)
mdadm --create /dev/md0 --level=1 --raid-devices=2 \
    --data-offset=2048s /dev/sdb1 /dev/sdc1
```

For RAID arrays, the partition start **plus** the RAID data offset should be aligned. With metadata 1.2, both default to 1 MiB, so the total offset is 2 MiB — still aligned to any 4K boundary.

---

## RAID Stripe Alignment

For RAID 5/6/10, optimal I/O should also align to the stripe size (chunk size × number of data disks):

```bash
# Check chunk size
mdadm --detail /dev/md0 | grep "Chunk Size"

# Calculate stripe size
# RAID 5 with 4 disks, 64K chunk: stripe = 64K × 3 data disks = 192K
# RAID 6 with 6 disks, 128K chunk: stripe = 128K × 4 data disks = 512K

# Check optimal I/O size (kernel calculates this)
cat /sys/block/md0/queue/optimal_io_size

# Filesystem should use stride and stripe-width for ext4
mkfs.ext4 -E stride=16,stripe-width=48 /dev/md0
# stride = chunk_size / fs_block_size = 64K / 4K = 16
# stripe-width = stride × data_disks = 16 × 3 = 48
```

---

## Virtual Machines

In virtual environments, alignment matters at multiple levels:

```
Guest filesystem block
    ↓ (aligned to?)
Guest partition start
    ↓ (aligned to?)
Virtual disk file on host filesystem
    ↓ (aligned to?)
Host partition start
    ↓ (aligned to?)
Physical device sector
```

Misalignment at **any** layer cascades down, causing read-modify-write penalties on the physical device.

### Recommendations

- Host: ensure host partitions are aligned (1 MiB)
- Virtual disk: use thin-provisioned or raw disk formats that don't add headers that break alignment
- Guest: ensure guest partitions start at 1 MiB (modern OS installers do this by default)

```bash
# Check alignment from within the guest
fdisk -l -u /dev/sda
# Verify start sector is 2048 or a multiple of 2048
```

---

## Fixing Misaligned Partitions

Fixing alignment requires moving the partition start, which means **data must be moved or recreated**.

### Option 1 — Backup, Repartition, Restore

```bash
# 1. Backup the data
dd if=/dev/sdb1 of=/backup/sdb1.img bs=64K

# 2. Delete and recreate partition with aligned start
fdisk /dev/sdb
# d → delete partition 1
# n → new partition 1, start at 2048 (default on modern fdisk)
# w → write

# 3. Restore data
dd if=/backup/sdb1.img of=/dev/sdb1 bs=64K

# 4. Resize filesystem to match new partition size
resize2fs /dev/sdb1     # ext4
xfs_growfs /mountpoint  # XFS
```

### Option 2 — GParted (Live CD)

GParted can move partitions to aligned positions while preserving data.

### Option 3 — Leave It (If Performance Is Acceptable)

On traditional 512-byte-sector HDDs, misalignment has no performance impact. If the workload is acceptable, the effort to fix may not be worth it.

---

## Quick Reference

```bash
# Check if aligned (start sector divisible by 2048 = 1 MiB aligned)
fdisk -l -u /dev/sda
sfdisk -d /dev/sda
parted /dev/sda align-check optimal 1

# Create aligned partition (modern fdisk — default)
fdisk /dev/sdb
# Default start sector is 2048

# Create aligned partition (parted)
parted -s /dev/sdb mkpart primary ext4 1MiB 100%

# Create aligned partition (old fdisk)
fdisk -S 32 -H 64 /dev/sdb
# Start at cylinder 2

# Check LVM PE alignment
pvs -o+pe_start

# Check RAID data offset
mdadm --examine /dev/sdb1 | grep "Data Offset"

# Check device sector sizes
cat /sys/block/sda/queue/physical_block_size
cat /sys/block/sda/queue/logical_block_size
cat /sys/block/sda/queue/optimal_io_size

# Check filesystem stride/stripe for ext4
tune2fs -l /dev/md0p1 | grep -i stride
dumpe2fs /dev/md0p1 | grep -i stride
```

---

## Sector Formats: 512n, 512e, and 4Kn

Modern drives come in three sector format variants:

| Format | Logical Sector | Physical Sector | Description |
|--------|---------------|-----------------|-------------|
| **512n** (native) | 512 bytes | 512 bytes | Legacy. Rare in modern drives. |
| **512e** (emulation) | 512 bytes | 4096 bytes | Most common today. Firmware emulates 512B for compatibility, but writes internally in 4K. Misalignment triggers RMW. |
| **4Kn** (native) | 4096 bytes | 4096 bytes | No emulation layer. OS writes directly to 4K pages. Best performance, but requires UEFI boot and 4K-aware tools. |

### Detecting Sector Format

```bash
# Method 1: lsblk -t (shows physical and logical sector sizes)
lsblk -t /dev/sda
# PHY-SEC=4096, LOG-SEC=512 → 512e
# PHY-SEC=4096, LOG-SEC=4096 → 4Kn
# PHY-SEC=512,  LOG-SEC=512  → 512n

# Method 2: sysfs
cat /sys/block/sda/queue/physical_block_size   # physical sector
cat /sys/block/sda/queue/logical_block_size    # logical sector

# Method 3: smartctl (SATA/SAS)
smartctl -i /dev/sda | grep "Sector Size"
# "512 bytes logical, 4096 bytes physical" → 512e

# Method 4: NVMe — check supported LBA formats
nvme id-ns /dev/nvme0n1 -H | grep "LBA Format"

# Method 5: Check discard/TRIM support
lsblk -D /dev/sda
# DISC-GRAN and DISC-MAX > 0 means TRIM is supported
```

### Converting NVMe to 4Kn

Enterprise NVMe SSDs often support multiple LBA formats. Converting to 4Kn eliminates the 512e emulation overhead:

```bash
# List available formats
nvme id-ns /dev/nvme0n1 -H | grep "LBA Format"
# LBA Format 0: Data Size: 512 bytes - Relative Performance: 0x2 (Good)
# LBA Format 1: Data Size: 4096 bytes - Relative Performance: 0x1 (Better)

# WARNING: Destroys ALL data on the namespace
umount /dev/nvme0n1*
nvme format /dev/nvme0n1 --lbaf=1 --force

# Force kernel to re-read geometry
echo 1 > /sys/block/nvme0n1/device/rescan
partprobe /dev/nvme0n1

# Verify
lsblk -t /dev/nvme0n1
```

### Converting SATA/SAS to 4Kn (sg_format)

```bash
# Install sg3_utils
apt install sg3-utils       # Debian/Ubuntu
yum install sg3_utils       # RHEL/CentOS

# Check current sector size
sg_readcap -l /dev/sg1

# Reformat to 4Kn (DESTROYS ALL DATA, can take hours on HDDs)
sg_format --format --size=4096 /dev/sg1
```

> **Note:** 4Kn drives cannot boot with legacy BIOS/MBR. UEFI is required. Modern GRUB2 and systemd-boot handle 4Kn natively.

---

## Avoiding Partitions Entirely

For dedicated data disks (not boot), you can skip partitioning and create the filesystem directly on the block device:

```bash
mkfs.xfs /dev/sdb
mkfs.ext4 /dev/sdb
```

This avoids any possible partition alignment issues. The filesystem starts at byte 0 of the device, which is inherently aligned. This approach is common in high-performance storage setups (BeeGFS, Ceph OSDs).

> **Downside:** No partition table means `lsblk -f` won't show partition entries, and some tools may not auto-detect the filesystem. It also prevents multiple filesystems on a single device.

---

## XFS RAID-Optimized Creation

XFS supports RAID parameters natively via `-d su=<stripe unit>,sw=<stripe width>`:

```bash
# RAID-6 with 11 disks (9 data disks), 64K chunk size
mkfs.xfs -d su=64k,sw=9 /dev/md0

# Same with journal optimized for RAID
mkfs.xfs -d su=64k,sw=9 -l version=2,su=64k /dev/md0
```

Where:
- `su` = stripe unit = RAID chunk size
- `sw` = stripe width = number of **data** disks (total disks minus parity disks)

For ext4, the equivalent uses stride and stripe-width:

```bash
# stride = chunk_size / fs_block_size = 64K / 4K = 16
# stripe-width = stride × data_disks = 16 × 9 = 144
mkfs.ext4 -E stride=16,stripe-width=144 /dev/md0
```

### Verifying Filesystem RAID Geometry

```bash
# XFS: check current sunit and swidth (in 512-byte sectors)
xfs_info /mountpoint | grep -E "sunit|swidth"
# sunit=128 means 128 × 512 = 64K stripe unit
# swidth=1152 means 1152 × 512 = 576K (9 data disks × 64K)
# sunit=0, swidth=0 means NO RAID geometry recorded — recreate fs to fix

# ext4: check stride and stripe-width
tune2fs -l /dev/md0 | grep -Ei "stride|stripe"
dumpe2fs -h /dev/md0 | grep -Ei "stride|stripe"
```

### LVM Striped Logical Volumes

When using LVM on RAID or multiple disks, create striped LVs for aligned I/O:

```bash
# Create a striped logical volume (4 stripes, 64K each)
lvcreate -i 4 -I 64K -L 100G -n lv_data vg_data

# Then format with matching geometry
mkfs.xfs -d su=64k,sw=4 /dev/vg_data/lv_data
mkfs.ext4 -E stride=16,stripe-width=64 /dev/vg_data/lv_data
```

---

## LUKS Encryption on 4Kn Devices

When using LUKS on 4K-native devices, specify the sector size to maintain alignment:

```bash
# Create LUKS container with 4096-byte sectors
cryptsetup luksFormat --sector-size 4096 /dev/nvme0n1p1

# Open
cryptsetup luksOpen /dev/nvme0n1p1 crypt_data

# Create filesystem on the mapper device
mkfs.ext4 /dev/mapper/crypt_data
```

Without `--sector-size 4096`, LUKS defaults to 512-byte sectors, reintroducing the 512e-style RMW penalty on 4Kn hardware.

---

## parted Alignment Flags

parted supports explicit alignment modes:

```bash
# Optimal alignment (default — 1 MiB)
parted --align=opt /dev/sdb mkpart primary ext4 0% 100%

# Minimal alignment (physical sector boundary only)
parted --align=min /dev/sdb mkpart primary ext4 0% 100%

# No alignment (dangerous — allows misaligned partitions)
parted --align=none /dev/sdb mkpart primary ext4 63s 100%

# Check alignment after creation
parted /dev/sdb align-check optimal 1
```

---

## Summary Table

| Scenario | Aligned at | How to verify |
|----------|-----------|---------------|
| Partition on SSD/4Kn HDD | 4096 bytes (minimum), 1 MiB (recommended) | Start sector % 2048 == 0 |
| Partition on RAID | 1 MiB + stripe-aligned filesystem | `parted align-check`, `optimal_io_size` |
| LVM PE start | 1 MiB (LVM 2.02.73+) | `pvs -o+pe_start` |
| RAID data offset | 1 MiB (metadata 1.2) | `mdadm --examine` |
| Virtual machine guest | 1 MiB (guest partition start) | `fdisk -l -u` inside guest |
