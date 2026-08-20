# Linux Swap Management

Managing swap space on Linux — creating swap files and partitions, tuning swappiness, monitoring, resizing, and troubleshooting.

## Recommended Swap Space

| RAM | Recommended Swap | With Hibernation |
|-----|:----------------:|:----------------:|
| ≤ 2 GB | 2× RAM | 3× RAM |
| > 2 GB – 8 GB | Equal to RAM | 2× RAM |
| > 8 GB – 64 GB | At least 4 GB | 1.5× RAM |
| > 64 GB | At least 4 GB | Hibernation not recommended |

Guidelines:

- Servers: 4–8 GB is usually sufficient regardless of RAM
- Desktops with hibernation: need at least as much swap as RAM
- Kubernetes nodes: swap typically disabled
- Database servers: minimal swap (safety net only)

## Check Current Swap

```bash
# Summary
free -h

# Detailed swap info
swapon --show
swapon -s

# Swap usage in /proc
cat /proc/swaps

# Memory info including swap
cat /proc/meminfo | grep -i swap

# Swap usage per process (top consumers)
for file in /proc/*/status; do
    awk '/VmSwap|Name/{printf $2 " " $3}END{ print ""}' "$file"
done | sort -k 2 -n -r | head -20
```

## Create a Swap File

### Modern Method (fallocate)

```bash
# Create a 4 GB swap file
sudo fallocate -l 4G /swapfile

# Set correct permissions (must be 600)
sudo chmod 600 /swapfile

# Format as swap
sudo mkswap /swapfile

# Enable swap
sudo swapon /swapfile

# Verify
swapon --show
free -h
```

### Alternative Method (dd)

Use `dd` if the filesystem doesn't support `fallocate` (e.g., XFS with certain mount options, or older ext3):

```bash
# Create a 4 GB swap file
sudo dd if=/dev/zero of=/swapfile bs=1M count=4096 status=progress

# Set permissions
sudo chmod 600 /swapfile

# Format and enable
sudo mkswap /swapfile
sudo swapon /swapfile
```

### Make Permanent (fstab)

```bash
# Add to /etc/fstab
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Verify fstab is correct (don't skip this)
sudo findmnt --verify
```

## Create a Swap Partition

```bash
# Create partition (using fdisk or gdisk)
sudo fdisk /dev/sdb
# n → new partition
# t → change type to 82 (Linux swap)
# w → write

# Or with parted
sudo parted /dev/sdb mkpart primary linux-swap 0% 100%

# Format as swap
sudo mkswap /dev/sdb1

# Enable
sudo swapon /dev/sdb1

# Make permanent
echo '/dev/sdb1 none swap sw 0 0' | sudo tee -a /etc/fstab

# Or use UUID (preferred)
UUID=$(sudo blkid -s UUID -o value /dev/sdb1)
echo "UUID=${UUID} none swap sw 0 0" | sudo tee -a /etc/fstab
```

## Remove Swap

### Remove Swap File

```bash
# Disable swap
sudo swapoff /swapfile

# Remove from fstab
sudo sed -i '/swapfile/d' /etc/fstab

# Delete the file
sudo rm /swapfile
```

### Remove Swap Partition

```bash
# Disable
sudo swapoff /dev/sdb1

# Remove from fstab
sudo sed -i '/sdb1.*swap/d' /etc/fstab

# Delete partition with fdisk/parted
```

## Resize Swap

### Resize Swap File

```bash
# Disable current swap
sudo swapoff /swapfile

# Resize (e.g., to 8 GB)
sudo fallocate -l 8G /swapfile
# Or: sudo dd if=/dev/zero of=/swapfile bs=1M count=8192 status=progress

# Re-format and enable
sudo mkswap /swapfile
sudo swapon /swapfile

# Verify
free -h
```

## Swappiness

Swappiness controls how aggressively the kernel moves pages from RAM to swap. Range: 0–200 (0–100 on older kernels).

| Value | Behavior |
|:-----:|----------|
| 0 | Avoid swapping as much as possible (only swap under extreme memory pressure) |
| 10 | Minimal swapping (common for desktops and databases) |
| 60 | Default on most distributions |
| 100 | Kernel equally considers swap and filesystem cache eviction |
| 200 | Aggressively use swap (cgroups v2 only) |

### Check Current Value

```bash
cat /proc/sys/vm/swappiness
# or
sysctl vm.swappiness
```

### Change Temporarily

```bash
sudo sysctl vm.swappiness=10
```

### Change Permanently

```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system
```

### Recommended Values

| Use Case | Swappiness |
|----------|:----------:|
| Database servers (MySQL, PostgreSQL) | 1–10 |
| Desktop/workstation | 10–30 |
| General server | 30–60 |
| Default | 60 |
| Memory-constrained systems | 80–100 |

## VFS Cache Pressure

Controls how aggressively the kernel reclaims memory for directory and inode caches. Works alongside swappiness.

```bash
# Check current value (default: 100)
cat /proc/sys/vm/vfs_cache_pressure

# Lower = keep caches longer (good for file servers)
sudo sysctl vm.vfs_cache_pressure=50

# Make permanent
echo 'vm.vfs_cache_pressure=50' | sudo tee -a /etc/sysctl.d/99-swappiness.conf
```

## Swap Priority

When multiple swap spaces exist, priority determines which is used first (higher = used first).

```bash
# Set priority when enabling
sudo swapon -p 10 /swapfile
sudo swapon -p 5 /dev/sdb1

# In fstab (pri= option)
/swapfile none swap sw,pri=10 0 0
/dev/sdb1 none swap sw,pri=5  0 0

# Check priorities
swapon --show
```

Equal priority distributes across swap spaces in a round-robin fashion (like RAID 0 for swap).

## Encrypted Swap

### With dm-crypt (Random Key Each Boot)

```bash
# In /etc/crypttab
# swap    /dev/sdb1    /dev/urandom    swap,cipher=aes-xts-plain64,size=256

# In /etc/fstab
# /dev/mapper/swap    none    swap    sw    0    0
```

### With LUKS (Persistent Key, Supports Hibernate)

```bash
# Format with LUKS
sudo cryptsetup luksFormat /dev/sdb1
sudo cryptsetup open /dev/sdb1 cryptswap

# Create swap on encrypted device
sudo mkswap /dev/mapper/cryptswap
sudo swapon /dev/mapper/cryptswap
```

## Monitoring Swap

### Real-Time Monitoring

```bash
# Watch swap usage
watch -n 1 'free -h; echo; swapon --show'

# vmstat shows swap in/out (si/so columns)
vmstat 1

# sar swap statistics
sar -W 1 5

# Top swap consumers
top -o %MEM
# Press 'f' → add SWAP column → sort by it
```

### Check Swap I/O (Activity)

```bash
# si = swap in (pages read from swap)
# so = swap out (pages written to swap)
vmstat 1 5

# If si/so are consistently high, you may need more RAM
# Occasional swap is fine — constant swapping indicates memory pressure

# sar: swap utilization over time
sar -S
# kbswpfree  kbswpused  %swpused  kbswpcad  %swpcad

# sar: swap in/out rate (pswpin/s and pswpout/s)
sar -W
# High pswpin/s = frequently reading pages back from swap (thrashing)
# High pswpout/s = frequently writing pages to swap (memory pressure)
# Both high = active thrashing — system needs more RAM
```

> **Key insight:** High swap usage alone is not a problem. Swap being "full" just means idle pages were moved to disk. The real issue is constant swap I/O (`si`/`so` in vmstat, `pswpin`/`pswpout` in sar). If swap is used but I/O is zero, the system is fine — those pages are simply not being accessed.

### Per-Process Swap Usage

```bash
# Using smem (if installed)
smem -s swap -r

# From /proc (quick one-liner)
for pid in /proc/[0-9]*; do
    name=$(cat "$pid/comm" 2>/dev/null)
    swap=$(grep VmSwap "$pid/status" 2>/dev/null | awk '{print $2}')
    [ -n "$swap" ] && [ "$swap" -gt 0 ] && echo "$swap kB $name"
done | sort -n -r | head -20

# Using awk one-liner
awk '/VmSwap/{swap=$2} /Name/{name=$2} END{if(swap>0) print swap, name}' /proc/*/status 2>/dev/null | sort -rn | head -20
```

### Full Script: Top Swap Consumers (VmSwap)

```bash
#!/bin/bash
# sum_vmswap.bash — reports top processes by swapped-out pages

ps ax -o pid,user,args | grep -v '^  PID' | sed -e 's,^ *,,' > /tmp/ps_ax.output
echo -n > /tmp/results_vmswap

for swappid in $(grep -l VmSwap /proc/[1-9]*/status); do
    swapusage=0
    for x in $(grep VmSwap "$swappid" 2>/dev/null | grep -v '\W0 kB' | awk '{print $2}'); do
        let swapusage+=$x
    done
    pid=$(echo "$swappid" | cut -d'/' -f3)
    if [ $swapusage -ne 0 ]; then
        echo -ne "${swapusage} kb\t\t" >> /tmp/results_vmswap
        egrep "^$pid " /tmp/ps_ax.output | sed -e 's,^[0-9]* ,,' >> /tmp/results_vmswap
    fi
done

echo "Top processes with swapped-out pages:"
sort -nr /tmp/results_vmswap | head -n 10
```

### Full Script: SwapPss (More Accurate for Shared Pages)

On RHEL 8+, `SwapPss` in `/proc/PID/smaps` provides proportional swap accounting for shared memory — more accurate than `VmSwap` when shared pages are swapped:

```bash
#!/bin/bash
# sum_swap_or_swappss.bash — uses SwapPss (RHEL 8+) or Swap (RHEL 7)

ps ax -o pid,user,args | grep -v '^  PID' | sed -e 's,^ *,,' > /tmp/ps_ax.output
echo -n > /tmp/results_smaps

# SwapPss is available in RHEL 8+
if grep -q SwapPss /proc/self/smaps 2>/dev/null; then
    SWAP_KEYWORD="SwapPss"
else
    SWAP_KEYWORD="Swap"
fi

for swappid in $(grep -l "${SWAP_KEYWORD}" /proc/[1-9]*/smaps 2>/dev/null); do
    swapusage=0
    for x in $(grep "${SWAP_KEYWORD}" "$swappid" 2>/dev/null | grep -v '\W0 kB' | awk '{print $2}'); do
        let swapusage+=$x
    done
    pid=$(echo "$swappid" | cut -d'/' -f3)
    if [ $swapusage -ne 0 ]; then
        echo -ne "${swapusage} kb\t\t" >> /tmp/results_smaps
        egrep "^$pid " /tmp/ps_ax.output | sed -e 's,^[0-9]* ,,' >> /tmp/results_smaps
    fi
done

echo "Top processes with swapped-out proportional shared memory:"
sort -nr /tmp/results_smaps | head -n 10
```

> **Note:** `VmSwap` does not accurately account for shared memory. If you have processes sharing large memory regions (databases, KVM), use the `SwapPss`-based script for more accurate results.

## Disable Swap Entirely

Common for Kubernetes nodes (kubelet requires swap off by default):

```bash
# Disable all swap immediately
sudo swapoff -a

# Remove swap entries from fstab
sudo sed -i '/swap/d' /etc/fstab

# Or comment them out
sudo sed -i '/swap/ s/^/#/' /etc/fstab

# Verify
free -h
swapon --show
```

### Kubernetes Note

```bash
# Kubernetes 1.22+ can use swap (alpha feature)
# Default behavior: kubelet refuses to start with swap on
# To allow swap (K8s 1.28+ beta):
# Add to kubelet config:
#   failSwapOn: false
#   memorySwap:
#     swapBehavior: LimitedSwap
```

## How Much Swap Do You Need?

| RAM | Recommended Swap | With Hibernation |
|-----|:----------------:|:----------------:|
| ≤ 2 GB | 2× RAM | 3× RAM |
| 2–8 GB | Equal to RAM | 2× RAM |
| 8–64 GB | 4–8 GB (minimum) | Equal to RAM |
| > 64 GB | 4 GB (minimum) | Not practical |

Guidelines:
- Servers: 4–8 GB is usually sufficient regardless of RAM
- Desktops with hibernation: need at least as much swap as RAM
- Kubernetes nodes: swap typically disabled
- Database servers: minimal swap (safety net only)

## Swap File vs Swap Partition

| Feature | Swap File | Swap Partition |
|---------|-----------|---------------|
| Ease of resizing | Easy (recreate file) | Requires partition tools |
| Setup complexity | Simple | Medium |
| Performance | Slightly slower (filesystem overhead) | Marginally faster |
| Flexibility | Can be on any filesystem | Dedicated device |
| Hibernation | Supported (with some setup) | Supported |
| Cloud/VM | Preferred (no repartitioning) | Less common |
| SSD wear | Same | Same |

For most modern use cases, swap files are preferred due to flexibility.

## Testing Swap (Force Swap Usage)

### Using stress-ng (Recommended)

```bash
# Install
sudo apt install stress-ng        # Ubuntu/Debian
sudo dnf install stress-ng        # RHEL/Fedora

# Consume 90% of RAM forcing swap usage, auto-stops after 60 seconds
stress-ng --vm 1 --vm-bytes 90% --vm-method all --timeout 60s

# More aggressive — consume specific amount
stress-ng --vm 2 --vm-bytes 2G --timeout 120s
```

### Pure Bash (Manual Stop)

```bash
#!/bin/bash
# Allocate memory in a loop until swap is used
# Press Ctrl+C to stop
echo "Allocating memory to force swap..."
MEMBLOCKS=()
while true; do
    MEMBLOCKS+=("$(head -c 10M /dev/zero | cat)")
    sleep 0.1
done
```

### Monitor While Testing

```bash
# Watch swap usage in real-time
watch -n 1 'free -h; echo; swapon --show'

# Or with per-process swap
watch -n 1 'free -h && echo && for f in /proc/*/status; do awk "/VmSwap|Name/{printf \$2 \" \" \$3}END{print \"\"}" "$f"; done | sort -k2 -nr | head -10'
```

> **Note:** `stress-ng` is preferred because it has a built-in timeout and won't accidentally crash the system. The bash method requires manual Ctrl+C and can trigger OOM if left running.

## Known Issues

- **cgroupsv1 on RHEL 8/9**: More aggressive swapping behavior when `vm.swappiness` is set with cgroupsv1. cgroups v2 handles swappiness differently (range 0–200).
- **systemd-journal swap consumption**: `systemd-journald` stores logs in `/run/log/journal` (tmpfs), which can be reclaimed by swapping. Mitigation: limit journal size or configure persistent storage (`Storage=persistent` in `/etc/systemd/journald.conf`).
- **RHEL 6 BZ#949166**: Increased swap usage due to kernel bug, resolved in `kernel-2.6.32-504.el6` (RHSA-2014-1392).

## Troubleshooting

### Swap Full but Low Memory Usage

```bash
# Check for shared memory segments consuming swap
ipcs -m

# See per-segment details
ipcs -m -l

# Total swap used by SHM (not visible in per-process accounting)
# See: linux-swap-shm-segments article for details
```

### Cannot Create Swap File on Btrfs

```bash
# Btrfs requires special handling (no copy-on-write for swap)
sudo btrfs filesystem mkswapfile --size 4g /swapfile
sudo swapon /swapfile

# Or manually disable CoW
sudo truncate -s 0 /swapfile
sudo chattr +C /swapfile
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### "swapon: /swapfile: swapon failed: Invalid argument"

```bash
# Usually means the file has holes (fallocate on some filesystems)
# Solution: use dd instead
sudo swapoff /swapfile 2>/dev/null
sudo rm /swapfile
sudo dd if=/dev/zero of=/swapfile bs=1M count=4096 status=progress
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### High Swap Usage with Plenty of Free RAM

```bash
# This is normal — Linux proactively swaps idle pages
# Check if swap I/O is actually happening
vmstat 1 5
# If si/so are 0, old pages are in swap but not actively being used

# To force pages back into RAM (temporarily)
sudo swapoff -a && sudo swapon -a
# Warning: this can cause OOM if there's not enough RAM
```

## Per-Process Swap Detection by RHEL Version

### RHEL 5 (Kernel 2.6.18-99.el5+)

On RHEL 5.3+, per-process swap is available in `/proc/PID/smaps` (the "Swap:" row per mapping). You must sum all entries:

```bash
# Sum swap for a single PID
sudo cat /proc/<PID>/smaps | grep Swap | awk '{ SUM += $2 } END { print SUM " kB" }'
```

Example `/proc/PID/smaps` entry:

```
bf93b000-bf950000 rw-p 00000000 00:00 0          [stack]
Size:                 88 kB
Rss:                  12 kB
Pss:                  12 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:        12 kB
Referenced:            4 kB
Anonymous:            12 kB
AnonHugePages:         0 kB
Swap:                  8 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
```

> **Note:** The `top` formula `SWAP = VIRT - RES` is incorrect. Always use `/proc/PID/smaps` or `/proc/PID/status` for accurate swap data.

### RHEL 6+ (Kernel 2.6.32+)

RHEL 6 introduced the `VmSwap` field in `/proc/<pid>/status`:

```bash
# Direct per-process swap
grep VmSwap /proc/<PID>/status
# VmSwap:      140 kB
```

### All Versions — Top 10 Swap Consumers

```bash
for file in /proc/*/status; do
    awk '/VmSwap|Name/{printf $2 " " $3}END{ print ""}' "$file"
done | sort -k 2 -n -r | head -10
```

## Finding Swap Consumers with top

| RHEL Version | Method |
|-------------|--------|
| RHEL 6 | SWAP field: `Shift+o`, then `p`, then `Enter` |
| RHEL 7+ | Press `f`, navigate to SWAP, select with `d` or `Space`, type `s` to sort by SWAP |

## System V Shared Memory (SHM) and Swap

Shared memory segments are swappable and not accounted in per-process VmSwap:

```bash
# Quick check — are any SHM pages swapped?
ipcs -um
# ------ Shared Memory Status --------
# segments allocated 1
# pages allocated 1
# pages resident  1
# pages swapped   0
# Swap performance: 0 attempts     0 successes

# Detailed per-segment view (last column = swap in bytes)
cat /proc/sysvipc/shm
# key  shmid  perms  size  cpid  lpid  nattch  uid  gid  cuid  cgid  atime  dtime  ctime  rss  swap
#   0  144048128  0  2097152  2259  2259  1  0  0  0  0  ...  0  0

# Check which process uses a shared memory segment
lsof | grep "shmid"

# Get details about a specific segment
ipcs -m -i <shmid>
```

**Notes:**
- `/proc/sysvipc/shm` is present since kernel 2.6.18-293.el5 (RHEL 5.8) and kernel 2.6.32-85.el6 (RHEL 6.1)
- It exports RSS and swap usage of each SHM segment in bytes
- HugePage-backed SHM segments are pinned in memory and don't swap
- The RSS column represents actual pages making up the mapping
- If `ipcs -um` shows "pages swapped > 0" but no process shows VmSwap, SHM is your culprit

## Create Swap on LVM

```bash
# Create a 2 GB logical volume for swap
lvcreate VolGroup00 -n LogVol02 -L 2G

# Format as swap
mkswap /dev/VolGroup00/LogVol02

# Add to /etc/fstab
echo '/dev/VolGroup00/LogVol02 none swap defaults 0 0' | sudo tee -a /etc/fstab

# Reload systemd mount units
sudo systemctl daemon-reload

# Activate
sudo swapon -v /dev/VolGroup00/LogVol02
```

## Flushing Swap

Forces all swap contents back into RAM. Use with caution — can trigger OOM if RAM is low:

```bash
# Flush and re-enable
sudo swapoff -a
sudo swapon -a
```

> **Warning:** If the system is low on memory, flushing swap forces everything back into RAM at once. This may cause OOM kills or degraded performance while pages are reclaimed.

## Quick Reference

| Action | Command |
|--------|---------|
| Check swap | `free -h` / `swapon --show` |
| Create swap file | `sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile` |
| Enable permanently | `echo '/swapfile none swap sw 0 0' >> /etc/fstab` |
| Disable swap | `sudo swapoff -a` |
| Check swappiness | `cat /proc/sys/vm/swappiness` |
| Set swappiness | `sudo sysctl vm.swappiness=10` |
| Swap priority | `sudo swapon -p 10 /swapfile` |
| Per-process swap | `grep VmSwap /proc/*/status \| sort -k2 -n -r` |
| Swap I/O | `vmstat 1` (si/so columns) |
