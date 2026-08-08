# fsck Cheatsheet

`fsck` (file system check) verifies and repairs Linux filesystems. It is a front-end that calls filesystem-specific tools (`e2fsck` for ext2/3/4, `xfs_repair` for XFS, etc.).

---

## Important Rules

1. **Never run fsck on a mounted filesystem** — it can cause data corruption.
2. Unmount the filesystem first, or run from a live CD / rescue mode.
3. The root filesystem can only be checked in single-user mode or from recovery.
4. Always have backups before running repairs.

---

## Basic Usage

```bash
# Check a filesystem (interactive — prompts for each fix)
fsck /dev/sda1

# Check by filesystem label
fsck LABEL=mylabel

# Check by UUID
fsck UUID=3a4f5c6d-1234-5678-abcd-ef0123456789

# Check all filesystems in /etc/fstab (respecting pass number)
fsck -A

# Check all filesystems except root
fsck -AR

# Check only a specific filesystem type
fsck -t ext4 /dev/sda1
```

---

## Common Flags

| Flag | Description | Risk |
|------|-------------|------|
| `-n` | Answer "no" to all — read-only check, no changes | 🟢 Safe |
| `-f` | Force check even if filesystem appears clean | 🟢 Safe |
| `-v` | Verbose — passed through to filesystem-specific checker | 🟢 Safe |
| `-p` | Automatic repair (safe fixes only, no prompts) | 🟡 Moderate |
| `-a` | Automatically repair (no prompts) — deprecated, use `-p` | 🟡 Moderate |
| `-y` | Answer "yes" to all prompts (aggressive repair) | 🔴 Risky |
| `-C` | Display progress bar (ext2/3/4 only) | 🟢 Safe |
| `-V` | Produce verbose output (shows which fsck binary is called) | 🟢 Safe |
| `-t type` | Specify filesystem type | 🟢 Safe |
| `-A` | Check all filesystems in `/etc/fstab` | — |
| `-R` | Skip root filesystem (used with `-A`) | — |
| `-N` | Dry run — show what would be done without doing it | 🟢 Safe |
| `-T` | Don't show title on startup | 🟢 Safe |

> **Note:** Flags like `-c` (bad blocks) and `-r` (interactive) are not part of the `fsck` front-end itself — they belong to `e2fsck`. The `fsck` wrapper passes unrecognized flags through to the underlying checker.

---

## Automatic Repair

```bash
# Safe automatic repair (fix non-destructive issues)
fsck -p /dev/sda1

# Answer yes to everything (more aggressive)
fsck -y /dev/sda1

# Force check + automatic repair
fsck -fp /dev/sda1

# Force check + yes to all
fsck -fy /dev/sda1
```

> **Warning:** `-y` can make destructive decisions (deleting corrupt inodes, truncating files). Use `-p` for safer unattended repairs. Only use `-y` when data loss is acceptable or you have backups.

---

## Read-Only Check (No Changes)

```bash
# Check only, don't modify anything
fsck -n /dev/sda1

# Same with verbose output
fsck -nV /dev/sda1
```

---

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | No errors |
| 1 | Filesystem errors corrected |
| 2 | System should be rebooted |
| 4 | Filesystem errors left uncorrected |
| 8 | Operational error |
| 16 | Usage or syntax error |
| 32 | Checking cancelled by user |
| 128 | Shared library error |

Codes are additive (e.g., exit code 3 = errors corrected + reboot needed).

---

## e2fsck (ext2/3/4 Specific)

`fsck.ext4`, `fsck.ext3`, and `fsck.ext2` are all symlinks to `e2fsck`.

```bash
# Standard check
e2fsck /dev/sda1

# Force check (ignore clean flag)
e2fsck -f /dev/sda1

# Auto-repair (safe)
e2fsck -p /dev/sda1

# Auto-repair (aggressive)
e2fsck -y /dev/sda1

# Read-only check
e2fsck -n /dev/sda1

# Show progress
e2fsck -C 0 /dev/sda1

# Force check with verbose output
e2fsck -f -v /dev/sda1

# Check and repair with bad block scan
e2fsck -c -f /dev/sda1

# Show only summary
e2fsck -n -f /dev/sda1 | grep -E "(errors|corrected|warnings)"

# Check and show block bitmap differences
e2fsck -f -b 32768 /dev/sda1
```

### e2fsck Phases

e2fsck runs through 5 passes:

| Pass | What it checks |
|------|----------------|
| 1 | Inodes, blocks, and sizes |
| 2 | Directory structure |
| 3 | Directory connectivity |
| 4 | Reference counts |
| 5 | Group summary information |

### Superblock Recovery

If the primary superblock is corrupt, use a backup superblock:

```bash
# List backup superblock locations
dumpe2fs /dev/sda1 | grep -i superblock

# Typical backup locations: 32768, 98304, 163840, ...

# Use backup superblock for repair
e2fsck -b 32768 /dev/sda1

# Force + backup superblock + auto-repair
e2fsck -f -b 32768 -y /dev/sda1
```

### Recovering Lost+Found

Files that fsck recovers but can't place back in the directory tree end up in `/lost+found`:

```bash
# Check what was recovered
ls -la /mountpoint/lost+found/

# Files are named by inode number (e.g., #12345)
# Use file command to identify type
file /mountpoint/lost+found/#12345
```

---

## xfs_repair (XFS)

XFS does not use `fsck` at boot. Use `xfs_repair` instead.

```bash
# Check and repair
xfs_repair /dev/sda1

# Read-only check (no modifications)
xfs_repair -n /dev/sda1

# Verbose output
xfs_repair -v /dev/sda1

# Read-only + verbose
xfs_repair -n -v /dev/sda1

# Force log zeroing (if log is corrupt — last resort)
xfs_repair -L /dev/sda1

# Use alternate AG size (for very large filesystems)
xfs_repair -o ag_stride=32 /dev/sda1

# Use more memory for large filesystems (value in megabytes)
xfs_repair -m 2048 /dev/sda1
```

> **Warning:** `xfs_repair -L` zeroes the journal log. This may lose recent data that was in-flight during a crash. Only use it when normal repair fails.

### XFS Pre-Check

```bash
# Unmount first
umount /dev/sda1

# If mount is busy, check what's using it
fuser -mv /mountpoint
lsof +f -- /mountpoint
```

---

## Btrfs

```bash
# Read-only check
btrfs check --readonly /dev/sda1

# Check with progress
btrfs check --progress /dev/sda1

# Force check even if mounted (read-only)
btrfs check --force /dev/sda1

# Check and repair (dangerous — use as last resort)
btrfs check --repair /dev/sda1

# Online scrub (can run on mounted filesystem)
btrfs scrub start /mountpoint
btrfs scrub status /mountpoint

# Recovery tools
btrfs rescue super-recover /dev/sda1
btrfs rescue zero-log /dev/sda1

# Restore data from damaged btrfs
btrfs restore /dev/sda1 /recovery/path
```

> **Note:** `btrfs check --repair` is considered experimental. Always use `--readonly` first and have backups.

---

## FAT32/VFAT (dosfsck)

```bash
# Check FAT filesystem (also available as fsck.vfat or fsck.fat)
dosfsck -v /dev/sda1

# Auto-repair
dosfsck -a /dev/sda1

# Interactive repair
dosfsck -r /dev/sda1

# Check and mark bad clusters
dosfsck -t /dev/sda1
```

---

## NTFS (ntfsfix)

```bash
# Basic NTFS check and repair
ntfsfix /dev/sda1

# Check only (no repair)
ntfsfix -n /dev/sda1

# Clear dirty flag
ntfsfix -d /dev/sda1

# Clear bad sector list
ntfsfix -b /dev/sda1
```

> **Note:** For full NTFS repair, boot into Windows and run `chkdsk /f`. `ntfsfix` only performs basic fixes.

---

## Checking the Root Filesystem

The root filesystem cannot be unmounted while running. Options:

### Method 1 — Force fsck on Next Boot

```bash
# Create a file that triggers fsck at boot
sudo touch /forcefsck

# Or use tune2fs (ext filesystems)
sudo tune2fs -C 100 /dev/sda1   # set mount count high to trigger check

# Reboot
sudo reboot
```

### Method 2 — Remount Read-Only

```bash
# Remount root as read-only
sudo mount -o remount,ro /

# Run fsck
sudo fsck -f /

# Remount read-write
sudo mount -o remount,rw /
```

### Method 3 — Recovery / Single-User Mode

```bash
# From GRUB, append to kernel line:
# systemd.unit=rescue.target
# or: single / init=/bin/bash

# Once in single-user mode:
fsck -f /dev/sda1
```

### Method 4 — Live USB / Rescue CD

Boot from external media and run fsck on the unmounted partition.

---

## Scheduled Filesystem Checks

ext filesystems can be configured to auto-check after N mounts or N days:

```bash
# View current settings
tune2fs -l /dev/sda1 | grep -i "mount count\|check interval\|last checked"

# Set check after every 30 mounts
tune2fs -c 30 /dev/sda1

# Set check every 180 days
tune2fs -i 180d /dev/sda1

# Disable automatic checks (not recommended)
tune2fs -c 0 -i 0 /dev/sda1

# Reset mount count to 0
tune2fs -C 0 /dev/sda1

# Set next check time
tune2fs -T now /dev/sda1
```

---

## /etc/fstab — Pass Field

Column 6 (pass) in `/etc/fstab` is used by fsck to determine whether the filesystem should be checked before it is mounted:

| Value | Behaviour |
|-------|-----------|
| `0` | Disabled — do not check filesystem |
| `1` | Highest priority, checked first. Usually set to root `/` partition |
| `2` | Checked after root (can run in parallel with other `2`s) |

```
/dev/VolGroup/LogVol     /mountpoint    ext4    defaults    1 1
                                                               ^ fsck enabled

/dev/VolGroup/LogVol     /mountpoint    ext4    defaults    1 0
                                                               ^ fsck disabled
```

More examples:

```
/dev/sda1  /      ext4  defaults  0  1
/dev/sda2  /home  ext4  defaults  0  2
/dev/sdb1  /data  xfs   defaults  0  0
```

> **Note:** fsck is performed if corruption is detected inside the filesystem during system boot. XFS, btrfs, and other modern filesystems typically use `0` here because they handle consistency via journaling or copy-on-write.

These systemd-fsck services are started at boot if passno in `/etc/fstab` for the filesystem is set to a value greater than zero. If a filesystem check fails for a service without `nofail`, emergency mode is activated by isolating to `emergency.target`.

---

## Force fsck on Next Reboot

### RHEL 6

`shutdown -F -r now` does not work anymore. Use one of:

```bash
# Method 1: Create /forcefsck file
touch /forcefsck

# Method 2: Pass forcefsck as kernel command line parameter at boot
# (edit GRUB and append: forcefsck)
```

- The `/fsckoptions` file can be used to pass additional fsck options
- The last field (pass) in `/etc/fstab` must not be set to `0`

### RHEL 7 / 8 / 9 / 10

Rescue mode mounts filesystems in rw mode, so force the fsck at GRUB instead:

```bash
# At boot time, add to the kernel command line (does not work for XFS):
fsck.mode=force fsck.repair=yes
```

See [Kernel Boot Parameters](#kernel-boot-parameters) above for the full `fsck.mode` and `fsck.repair` options table.

---

## Run fsck in Rescue Mode

### RHEL 6

#### Method 1 — Boot from Installation CD

Boot with installation CD in rescue mode: `linux rescue nomount`

```bash
vgchange -ay VolGroup
fsck.ext4 /dev/VolGroup/lv_root
# or
fsck.ext4 /dev/sda2
```

#### Method 2 — Append init=/bin/bash

Append `init=/bin/bash` to kernel command line at boot:

```bash
fsck.ext4 /dev/VolGroup/lv_root
# or
fsck.ext4 /dev/sda2
```

### RHEL 7 / 8 / 9

#### Method 1 — Boot from Installation CD

Boot with installation media in rescue mode: `linux rescue`

```bash
vgchange -ay rhel_<hostname>
xfs_repair /dev/rhel_<hostname>/root
```

#### Method 2 — Append rd.break

Append `rd.break` or `rw init=/sysroot/bin/sh` to kernel command line at boot:

```bash
umount /sysroot
xfs_repair /dev/rhel_<hostname>/root
reboot -f
```

### Ubuntu / Debian

#### Method 1 — Recovery Mode from GRUB

Hold Shift (BIOS) or press Esc (UEFI) during boot to show the GRUB menu:

1. Select **Advanced options for Ubuntu**
2. Select the **recovery mode** kernel entry
3. Choose **fsck — Check all file systems** from the recovery menu

Or drop to a root shell from the recovery menu:

```bash
# Root filesystem is mounted read-only in recovery mode
fsck -f /dev/sda1

# If using LVM
vgchange -ay ubuntu-vg
fsck -f /dev/ubuntu-vg/ubuntu-lv

# Reboot when done
reboot
```

#### Method 2 — Append kernel parameters

Edit the GRUB boot entry (press `e`) and append to the `linux` line:

```
init=/bin/bash
```

Then:

```bash
# Root is mounted read-only by default
fsck -f /dev/sda1

# Or for LVM
vgchange -ay ubuntu-vg
fsck -f /dev/ubuntu-vg/ubuntu-lv

# Remount and reboot
mount -o remount,rw /
reboot -f
```

#### Method 3 — Boot from Live USB

Boot from Ubuntu/Debian live USB or installation media, select "Try Ubuntu":

```bash
# Identify the partition
lsblk -f

# If using LVM, activate volume groups
sudo vgchange -ay

# Run fsck on the unmounted root partition
sudo fsck -f /dev/sda1
# or
sudo fsck -f /dev/ubuntu-vg/ubuntu-lv

# For encrypted LUKS partitions
sudo cryptsetup luksOpen /dev/sda3 crypt_root
sudo vgchange -ay
sudo fsck -f /dev/ubuntu-vg/ubuntu-lv
```

#### Method 4 — Force fsck on next boot

```bash
sudo touch /forcefsck
sudo reboot

# Or use systemd kernel parameters (18.04+)
# Edit GRUB and append: fsck.mode=force fsck.repair=yes
```

---

## badblocks

Check for physical bad sectors on a disk:

```bash
# Read-only scan (safe)
badblocks -sv /dev/sda1

# Read-write scan (DESTRUCTIVE — erases data)
badblocks -wsv /dev/sda1

# Non-destructive read-write scan (slow but preserves data)
badblocks -nsv /dev/sda1

# Use with e2fsck — mark bad blocks so they're avoided
e2fsck -c /dev/sda1          # read-only test
e2fsck -cc /dev/sda1         # non-destructive read-write test
```

> **Note:** If `badblocks` finds bad sectors on a modern drive, the drive is likely failing. Back up immediately and replace it. Modern drives handle bad sector remapping internally via SMART.

---

## SMART Checks (Complementary)

```bash
# Check drive health
smartctl -H /dev/sda

# Full SMART info
smartctl -a /dev/sda

# Run short self-test
smartctl -t short /dev/sda

# Run long self-test
smartctl -t long /dev/sda

# View test results
smartctl -l selftest /dev/sda
```

---

## Pre-Check Information Gathering

Before running fsck, identify the filesystem and verify it's not mounted:

```bash
# Check filesystem type
lsblk -f /dev/sda1
blkid /dev/sda1
file -s /dev/sda1

# Check if mounted
mount | grep /dev/sda1

# Get filesystem info
tune2fs -l /dev/sda1         # ext2/3/4
xfs_info /dev/sda1           # XFS (must be mounted)
btrfs filesystem show /dev/sda1  # btrfs

# Check disk health first
smartctl -a /dev/sda

# Check fragmentation (ext4)
e4defrag -c /mountpoint

# Check inode usage
df -i /dev/sda1
```

---

## Data Recovery Workflow

When a filesystem is severely damaged, work on a copy:

```bash
# 1. Create image of damaged filesystem
dd if=/dev/sda1 of=/backup/filesystem.img bs=64K conv=noerror,sync

# For damaged disks with read errors, use ddrescue
ddrescue -f -n /dev/sda1 /backup/rescued.img /tmp/rescue.log

# 2. Attach image as loop device
losetup /dev/loop0 /backup/filesystem.img

# 3. Check the copy first (read-only)
fsck -n /dev/loop0

# 4. Attempt repair on the copy
fsck -y /dev/loop0

# 5. Mount and verify recovered data
mount /dev/loop0 /mnt/recovery
ls -la /mnt/recovery

# 6. Clean up
umount /mnt/recovery
losetup -d /dev/loop0
```

### Emergency Data Extraction with debugfs

When mount fails entirely, use `debugfs` to pull files by inode:

```bash
debugfs /dev/sda1

# Inside debugfs:
# ls                           # list root directory
# cd /path/to/dir              # navigate
# dump <inode> /rescue/file    # extract a file
# rdump /dir /rescue/          # recursive dump
# quit
```

### Alternative Mount Options for Damaged Filesystems

```bash
# ext4 — skip journal recovery
mount -o ro,norecovery /dev/sda1 /mnt/emergency

# XFS — skip log replay
mount -o ro,noload /dev/sda1 /mnt/emergency
```

---

## Multi-Pass Repair Strategy

For systematic recovery, escalate through these passes:

```bash
# Pass 1: Safe automatic repair
e2fsck -p /dev/sda1

# Pass 2: Force full check after auto-repair
e2fsck -f /dev/sda1

# Pass 3: If issues remain, run with -y (accepts all fixes)
e2fsck -y /dev/sda1

# Pass 4: Bad block scan if hardware suspected
e2fsck -c /dev/sda1

# Pass 5: Final read-only verification
e2fsck -n /dev/sda1
```

---

## Advanced Operations

### Inode Problems

```bash
# Check inode usage
df -i /dev/sda1

# Find file by inode number
find /mountpoint -inum 12345

# Optimize directory indexing (rebuilds htree)
e2fsck -D -f /dev/sda1

# Check for disconnected inodes
e2fsck -n /dev/sda1 | grep "disconnected inode"
```

### Filesystem Feature Management

```bash
# Add journal (convert ext2 → ext3)
tune2fs -j /dev/sda1

# Convert ext3 → ext4
tune2fs -O extents,uninit_bg,dir_index /dev/sda1
e2fsck -f /dev/sda1

# Remove journal (ext3 → ext2)
tune2fs -O ^has_journal /dev/sda1
e2fsck -f /dev/sda1

# Enable directory indexing
tune2fs -O dir_index /dev/sda1

# Change reserved space percentage (default 5%)
tune2fs -m 1 /dev/sda1    # reserve 1% for root
```

### SSD-Specific

```bash
# Enable discard during fsck (SSD TRIM)
e2fsck -E discard /dev/sda1

# Check TRIM support
fstrim -v /mountpoint

# Verify SSD health before fsck
smartctl -A /dev/sda | grep -E "(Wear|Write|Erase)"
```

### XFS Advanced

```bash
# Examine the log
xfs_logprint /dev/sda1

# XFS database tool (read-only)
xfs_db -r /dev/sda1
# freesp -s      # check free space
# sb 0 ; print   # print superblock
```

### Kernel Boot Parameters

Force fsck at boot via GRUB kernel command line:

```
fsck.mode=force fsck.repair=yes
```

#### RHEL / CentOS Version Differences

| Version | Init System | Boot-time fsck Parameter | Notes |
|---------|-------------|--------------------------|-------|
| RHEL 6 | Upstart/SysV | `fsck.mode=force` not supported; use `touch /forcefsck` | `/forcefsck` checked by rc.sysinit |
| RHEL 7 | systemd | `fsck.mode=force fsck.repair=yes` | systemd-fsck reads kernel cmdline |
| RHEL 8 | systemd | `fsck.mode=force fsck.repair=yes` | Same as RHEL 7 |
| RHEL 9 | systemd | `fsck.mode=force fsck.repair=yes` | Same as RHEL 7 |
| RHEL 10 | systemd | `fsck.mode=force fsck.repair=yes` | Same as RHEL 7 |

`fsck.mode` options:

| Value | Behaviour |
|-------|-----------|
| `auto` | Default — check based on mount count / time interval |
| `force` | Force check regardless of clean flag |
| `skip` | Skip all filesystem checks |

`fsck.repair` options:

| Value | Behaviour |
|-------|-----------|
| `preen` | Default — safe automatic repair (`-p`) |
| `yes` | Answer yes to all questions (`-y`) |
| `no` | Read-only check, no repairs (`-n`) |

> **RHEL 6 alternative:** Since `fsck.mode` is a systemd feature, on RHEL 6 use `touch /forcefsck` or pass `fastboot` (to skip) / `forcefsck` (to force) as kernel parameters.

---

## 🔍 Error Detection and Diagnostics

```bash
# Common fsck error patterns
fsck -n /dev/sda1 2>&1 | grep -E "(error|corrupt|invalid|bad)"

# Count different error types
fsck -n /dev/sda1 2>&1 | grep -E -o "(superblock|inode|block|directory)" | sort | uniq -c

# Find severe errors
fsck -n /dev/sda1 2>&1 | grep -E "(multiply claimed|bad magic|corrupted)"

# Check exit codes
fsck -n /dev/sda1; echo "Exit code: $?"

# Check disk errors in kernel log
dmesg | grep -E "(ata|scsi|error)" | grep sd
grep -E "I/O error" /var/log/syslog       # Debian/Ubuntu
grep -E "I/O error" /var/log/messages     # RHEL/CentOS

# Check for reallocated sectors (SMART)
smartctl -A /dev/sda | grep -E "(Reallocated|Current_Pending|Uncorrectable)"
```

---

## Bad Block Detection and Handling

```bash
# Check for bad blocks (read-only scan, reports only — does not update bad block list)
e2fsck -c -n /dev/sda1

# Check for bad blocks and update bad block list
e2fsck -c /dev/sda1

# Non-destructive read-write test (same as e2fsck -cc)
e2fsck -c -c /dev/sda1

# Use external bad block scan and feed results to e2fsck
badblocks -v /dev/sda1 > badblocks.list
e2fsck -l badblocks.list /dev/sda1

# Force bad block check even if clean
e2fsck -c -f /dev/sda1
```

---

## Block Group Issues

```bash
# Check block group info
dumpe2fs /dev/sda1 | grep -A 5 "Group 0"

# Check filesystem stats
debugfs -R "stats" /dev/sda1

# Test specific blocks
e2fsck -c -v /dev/sda1 | grep "Testing with pattern"

# Check filesystem consistency by pass
e2fsck -n -v /dev/sda1 | grep -E "(Pass [1-5]|errors)"
```

---

## 🛡️ Corruption Recovery Strategies

### Mild Corruption

```bash
# Step 1: Read-only check first
fsck -n /dev/sda1

# Step 2: Gentle auto-repair
fsck -p /dev/sda1

# Step 3: Verify repair
fsck -n /dev/sda1

# Step 4: Test mount
mount -o ro /dev/sda1 /mnt/test
ls -la /mnt/test
umount /mnt/test
```

### Moderate Corruption

```bash
# Step 1: Create filesystem backup
dd if=/dev/sda1 of=/backup/fs-backup.img bs=64K conv=noerror,sync

# Step 2: Force full check
fsck -f -v /dev/sda1

# Step 3: Answer yes to remaining issues (if you have backups)
fsck -y /dev/sda1

# Step 4: If e2fsck, try alternative superblock
e2fsck -b 32768 /dev/sda1
```

### Severe Corruption

```bash
# Option 1: Try different backup superblocks
mke2fs -n /dev/sda1  # Find backup locations
e2fsck -b 32768 /dev/sda1
e2fsck -b 98304 /dev/sda1
e2fsck -b 163840 /dev/sda1

# Option 2: Force repair ignoring certain errors
e2fsck -y -f /dev/sda1

# Option 3: Salvage data with debugfs
debugfs /dev/sda1
# ls
# dump <inode> /recovery/filename
# quit

# Option 4: Partial recovery mount
mount -o ro,norecovery /dev/sda1 /mnt/recovery  # ext4
# Copy what you can, then:
umount /mnt/recovery
mke2fs /dev/sda1  # Reformat (data loss!)
```

---

## 🚨 Rescue Mode Operations

```bash
# Boot from rescue media, then:

# Mount damaged filesystem read-only first
mount -o ro /dev/sda1 /mnt/rescue

# Copy critical data
cp -a /mnt/rescue/important/* /backup/location/

# Unmount and repair
umount /mnt/rescue
fsck -f -y /dev/sda1

# Test mount after repair
mount /dev/sda1 /mnt/test
ls -la /mnt/test
umount /mnt/test
```

---

## Filesystem Cloning for Safety

```bash
# Create exact copy before repair
dd if=/dev/sda1 of=/backup/filesystem-backup.img bs=64K conv=noerror,sync

# Use ddrescue for damaged disks (handles read errors gracefully)
ddrescue -f -n /dev/sda1 /backup/rescued.img /backup/rescue.log

# Work on loop device
losetup /dev/loop0 /backup/filesystem-backup.img
fsck /dev/loop0
```

---

## 📈 Performance and Monitoring

### Speed Optimization

```bash
# Fast check (skip some tests)
e2fsck -p /dev/sda1

# Parallel checking of multiple filesystems
fsck -A -M -y  # -M: don't check if already mounted

# Check with minimal output
e2fsck -p -T /dev/sda1
```

### Progress Monitoring

```bash
# Show progress during long operations
e2fsck -C 0 /dev/sda1

# Monitor in another terminal
watch -n 1 'ps aux | grep fsck'

# Check progress in syslog
tail -f /var/log/syslog | grep fsck        # Debian/Ubuntu
tail -f /var/log/messages | grep fsck      # RHEL/CentOS
journalctl -f | grep fsck                  # systemd (all distros)
```

### Network Filesystem Checks

```bash
# NFS filesystem check (client side)
mount | grep nfs
df -h | grep nfs

# Check NFS server accessibility
showmount -e nfs-server-ip
rpcinfo -p nfs-server-ip

# CIFS/SMB share check
mount | grep cifs
smbclient -L //server-ip
```

---

## 📋 Maintenance Best Practices

### Preventive Measures

```bash
# Schedule automatic checks
tune2fs -c 50 /dev/sda1    # Check every 50 mounts
tune2fs -i 30d /dev/sda1   # Check every 30 days
tune2fs -C 49 /dev/sda1    # Set mount count to 49 (check on next mount)

# Enable filesystem features for better recovery
tune2fs -O has_journal /dev/sda1      # Add journal
tune2fs -O dir_index /dev/sda1        # Enable directory indexing
tune2fs -O extents /dev/sda1          # Enable extents (ext4)

# Enable SSD discard
tune2fs -o discard /dev/sda1
```

### Monitoring Cron Jobs

```bash
# Weekly filesystem check (all unmounted)
0 2 * * 0 root fsck -A -R -n >> /var/log/weekly-fsck.log 2>&1

# Monitor filesystem health every 6 hours
0 */6 * * * root smartctl -H /dev/sda && tune2fs -l /dev/sda1 | grep -E '(errors|Mount count)' >> /var/log/fs-health.log

# Alert on filesystem errors
*/15 * * * * root dmesg | grep -E 'ext4.*error' && echo 'Filesystem error detected' | mail root
```

---

## 🏆 Pro Tips

### Before Running fsck
1. Check SMART data: `smartctl -a /dev/sdX`
2. Verify unmounted: `mount | grep /dev/sdX1`
3. Check processes using it: `lsof | grep /dev/sdX1`
4. Make a backup: `dd if=/dev/sdX1 of=backup.img`
5. Note filesystem type: `blkid /dev/sdX1`

### During fsck
1. Monitor progress: Use `-C 0` for progress bar
2. Save output: `fsck -v /dev/sdX1 2>&1 | tee fsck.log`
3. Don't interrupt: Let it complete naturally
4. Watch system load: `top` in another terminal

### After fsck
1. Verify exit code: Check `$?` for success/failure
2. Test mount: Always test mount before putting back in service
3. Check data integrity: Verify critical files exist
4. Update maintenance logs: Document what was fixed
5. Schedule follow-up: Plan next check if issues were found

---

## 🚁 Emergency One-Liners

```bash
# Quick health check
lsblk -f && df -h && mount | grep -E "(ext|xfs|btrfs)"

# Check all unmounted filesystems
for dev in $(lsblk -rpo NAME,TYPE,MOUNTPOINT | awk '$2=="part" && $3=="" {print $1}'); do echo "=== $dev ==="; fsck -n "$dev" 2>&1 | head -10; done

# Find filesystems needing check
tune2fs -l /dev/sd* 2>/dev/null | grep -B5 -A5 "needs checking"

# Emergency read-only mount test
for dev in /dev/sd*1; do echo "Testing $dev:"; mount -o ro "$dev" /mnt 2>/dev/null && echo "OK" || echo "Failed"; umount /mnt 2>/dev/null; done

# Partial recovery with dd
dd if=/dev/sda1 of=/rescue/partial.img bs=64K count=1000 skip=0
```

---

## Automation Scripts

### Safe Check Script

```bash
#!/bin/bash
# safe-fsck.sh — Check a filesystem safely

DEVICE="$1"
if [[ -z "$DEVICE" ]]; then
    echo "Usage: $0 /dev/sdX1"
    exit 1
fi

# Check if mounted
if mount | grep -q "$DEVICE"; then
    echo "ERROR: $DEVICE is mounted! Unmount first."
    exit 1
fi

# Detect filesystem type
FSTYPE=$(blkid -o value -s TYPE "$DEVICE")
echo "Filesystem type: $FSTYPE"
echo "Running read-only check..."

case "$FSTYPE" in
    ext2|ext3|ext4) e2fsck -n -v "$DEVICE" ;;
    xfs)            xfs_repair -n -v "$DEVICE" ;;
    btrfs)          btrfs check --readonly "$DEVICE" ;;
    vfat)           dosfsck -v "$DEVICE" ;;
    *)              echo "Unsupported: $FSTYPE"; exit 1 ;;
esac

echo "Exit code: $?"
```

### Batch Check Script

```bash
#!/bin/bash
# batch-fsck.sh — Check all unmounted partitions

LOG="/var/log/fsck-batch-$(date +%Y%m%d-%H%M%S).log"

echo "Batch fsck started at $(date)" | tee "$LOG"

lsblk -rpo NAME,TYPE,FSTYPE,MOUNTPOINT | awk '$2=="part" && $3~/^ext/ && $4=="" {print $1}' | while read -r fs; do
    echo "--- Checking $fs ---" | tee -a "$LOG"
    fsck -n "$fs" 2>&1 | tee -a "$LOG"
    echo "Exit code: $?" | tee -a "$LOG"
done

echo "Completed at $(date)" | tee "$LOG"
```

---

## Troubleshooting

### "Device is busy" / Cannot unmount

```bash
# Find what's using the filesystem
fuser -mv /mountpoint
lsof +f -- /mountpoint

# Kill processes using it
fuser -k /mountpoint

# Force unmount (dangerous)
umount -l /mountpoint     # lazy unmount
umount -f /mountpoint     # force unmount (NFS)
```

### Superblock corrupt

```bash
# Try backup superblocks
mke2fs -n /dev/sda1 | grep -i superblock   # list locations without formatting
e2fsck -b 32768 /dev/sda1
```

### fsck running automatically at every boot

```bash
# Check and reset mount count
tune2fs -l /dev/sda1 | grep -i "mount count\|maximum mount"

# Disable forced checks
tune2fs -c -1 -i 0 /dev/sda1

# Check for /forcefsck file
ls -la /forcefsck && rm /forcefsck
```

### Journal recovery fails (ext3/4)

```bash
# Remove and recreate journal
tune2fs -O ^has_journal /dev/sda1
e2fsck -f /dev/sda1
tune2fs -j /dev/sda1
```

### XFS log corrupt

```bash
# Try normal repair first
xfs_repair /dev/sda1

# If that fails, zero the log (data loss risk)
xfs_repair -L /dev/sda1
```

---

## Quick Reference

| Command | Description |
|---------|-------------|
| `fsck /dev/sda1` | Interactive check |
| `fsck -p /dev/sda1` | Auto-repair (safe) |
| `fsck -y /dev/sda1` | Auto-repair (aggressive) |
| `fsck -n /dev/sda1` | Read-only check |
| `fsck -f /dev/sda1` | Force check (even if clean) |
| `fsck -A` | Check all in fstab |
| `e2fsck -b 32768 /dev/sda1` | Use backup superblock |
| `e2fsck -D -f /dev/sda1` | Rebuild directory index |
| `e2fsck -E discard /dev/sda1` | SSD-aware check |
| `xfs_repair /dev/sda1` | Repair XFS filesystem |
| `xfs_repair -n /dev/sda1` | XFS read-only check |
| `xfs_repair -L /dev/sda1` | XFS force log zero |
| `btrfs check --readonly /dev/sda1` | Btrfs read-only check |
| `btrfs scrub start /mnt` | Online btrfs scrub |
| `dosfsck -a /dev/sda1` | Auto-repair FAT |
| `ntfsfix /dev/sda1` | Basic NTFS repair |
| `tune2fs -c 30 /dev/sda1` | Check every 30 mounts |
| `tune2fs -i 180d /dev/sda1` | Check every 180 days |
| `tune2fs -m 1 /dev/sda1` | Set reserved space to 1% |
| `badblocks -sv /dev/sda1` | Scan for bad sectors |
| `ddrescue /dev/sda1 img log` | Clone damaged disk |
| `smartctl -H /dev/sda` | SMART health check |
| `touch /forcefsck` | Force fsck on next boot |
| `e4defrag -c /mountpoint` | Check ext4 fragmentation |

---

## Quick Reference Card

| Task | Command | Risk |
|------|---------|------|
| Safe check | `fsck -n /dev/sdX1` | 🟢 Safe |
| Auto repair | `fsck -p /dev/sdX1` | 🟡 Moderate |
| Force check | `fsck -f /dev/sdX1` | 🟡 Moderate |
| Interactive repair | `e2fsck /dev/sdX1` (default without -p/-y/-n) | 🟡 Moderate |
| Auto fix everything | `fsck -y /dev/sdX1` | 🔴 Risky |
| Bad block scan | `fsck -c /dev/sdX1` | 🟡 Moderate |
| Use backup superblock | `e2fsck -b 32768 /dev/sdX1` | 🟡 Moderate |

**Remember:** Always unmount first, backup when possible, and start with read-only checks.

---

## 🚨 Emergency Cheat Commands

```bash
# EMERGENCY: Quick assessment
lsblk -f && fsck -A -n 2>&1 | grep -E "(error|corrupt|bad)"

# EMERGENCY: Safe auto-repair all unmounted filesystems
fsck -A -p -R

# EMERGENCY: Force check root filesystem on next boot
touch /forcefsck && shutdown -r now

# EMERGENCY: Boot-time filesystem repair commands
# (Add to kernel parameters at boot)
# fsck.mode=force fsck.repair=yes

# EMERGENCY: Data rescue from corrupted filesystem
mkdir /rescue && mount -o ro,noload /dev/sdX1 /rescue 2>/dev/null || debugfs -R "ls" /dev/sdX1
```
