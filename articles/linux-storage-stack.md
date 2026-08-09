# Linux Storage Stack

The Linux storage stack is a layered architecture that handles I/O from user applications down to physical storage hardware. Understanding these layers is essential for performance tuning, troubleshooting, and storage design.

## Stack Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    USER SPACE                               │
│  Applications (read/write/fsync)                            │
│  Libraries (libaio, io_uring, POSIX AIO)                    │
└──────────────────────────┬──────────────────────────────────┘
                           │ System calls (read, write, pread, io_uring_enter)
┌──────────────────────────▼──────────────────────────────────┐
│                    VFS (Virtual File System)                │
│  Unified interface: open, read, write, close, stat          │
│  Dentry cache, inode cache, page cache                      │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    FILESYSTEMS                              │
│  ext4, XFS, Btrfs, bcachefs, EROFS, tmpfs, NFS, CIFS        │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    PAGE CACHE                               │
│  Buffered I/O (read-ahead, write-back, dirty pages          │
│  Direct I/O bypasses this layer                             │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    BLOCK LAYER                              │
│  Bio submission, I/O schedulers (mq-deadline, bfq, none)    │
│  Request merging, plugging/unplugging                       │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    DEVICE MAPPER / MD                       │
│  dm-linear, dm-crypt, dm-thin, dm-multipath, dm-vdo         │
│  MD (software RAID): raid0, raid1, raid5, raid6, raid10     │
│  LVM: PV → VG → LV                                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    SCSI / NVMe / VIRTIO LAYER               │
│  SCSI midlayer, NVMe driver, virtio-blk, virtio-scsi        │
│  HBA drivers: lpfc (Emulex), qla2xxx (QLogic), megaraid     │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    HARDWARE                                 │
│  SSD, HDD, NVMe, SAN (FC/iSCSI), Hardware RAID              │
└─────────────────────────────────────────────────────────────┘
```

## Layer Details

### VFS (Virtual File System)

The VFS provides a unified interface for all filesystems. Applications interact with VFS through system calls — they never touch the filesystem or block layer directly.

```bash
# View VFS caches
cat /proc/slabinfo | grep -E "dentry|inode"

# Drop caches (testing only)
echo 3 > /proc/sys/vm/drop_caches
# 1 = page cache, 2 = dentries/inodes, 3 = both

# View open files per process
ls /proc/<pid>/fd | wc -l
cat /proc/sys/fs/file-nr    # allocated, unused, max
```

### Filesystems

| Filesystem | Type | Key Feature |
|-----------|------|-------------|
| ext4 | Local | Default on many distros, journaling, stable |
| XFS | Local | High performance, large files, default on RHEL |
| Btrfs | Local | Copy-on-write, snapshots, checksums |
| bcachefs | Local | COW, compression, encryption (kernel 6.7+) |
| EROFS | Local | Read-only, compressed, for containers/images |
| tmpfs | Memory | RAM-backed, volatile |
| NFS | Network | Standard network filesystem |
| CIFS/SMB | Network | Windows/Samba shares |
| GlusterFS | Distributed | Scale-out network storage |
| CephFS | Distributed | RADOS-backed distributed FS |

```bash
# View mounted filesystems with types
df -Th

# View filesystem features
tune2fs -l /dev/sda1 | grep features    # ext4
xfs_info /dev/sda1                       # XFS

# View filesystem type
lsblk -f
blkid
```

### Page Cache

The page cache stores recently accessed file data in RAM. Buffered I/O goes through the cache; Direct I/O (`O_DIRECT`) bypasses it.

```bash
# View cache usage
free -h
# "buff/cache" column shows combined buffer + page cache

# Detailed cache breakdown
cat /proc/meminfo | grep -E "Cached|Buffers|Dirty|Writeback"

# View dirty page settings
sysctl vm.dirty_ratio
sysctl vm.dirty_background_ratio
sysctl vm.dirty_expire_centisecs
sysctl vm.dirty_writeback_centisecs

# Tune dirty page flushing (e.g., for write-heavy workloads)
sysctl -w vm.dirty_ratio=20
sysctl -w vm.dirty_background_ratio=5
```

### Block Layer

The block layer manages I/O requests: merging, scheduling, and submitting to drivers.

```bash
# View I/O scheduler for a device
cat /sys/block/sda/queue/scheduler

# Change scheduler
echo mq-deadline > /sys/block/sda/queue/scheduler

# View queue parameters
cat /sys/block/sda/queue/nr_requests
cat /sys/block/sda/queue/read_ahead_kb
cat /sys/block/sda/queue/max_sectors_kb
cat /sys/block/sda/queue/rotational    # 0=SSD, 1=HDD

# I/O stats
cat /proc/diskstats
iostat -x 1
```

#### I/O Schedulers

| Scheduler | Type | Best For |
|-----------|------|----------|
| `none` | No scheduling | NVMe, SSDs, virtualized storage |
| `mq-deadline` | Deadline-based | General purpose, databases, mixed workloads |
| `bfq` | Budget Fair Queueing | Interactive/desktop, latency-sensitive |
| `kyber` | Token-based | Fast devices with low latency |

Legacy (single-queue, pre-5.0 kernels):
| Scheduler | Best For |
|-----------|----------|
| `noop` | SSDs, virtualized |
| `deadline` | Databases, mixed |
| `cfq` | Desktop, interactive |

### Device Mapper (DM)

Device Mapper creates virtual block devices from physical ones. It's the foundation for LVM, encryption, multipath, and more.

| Target | Purpose | Created By |
|--------|---------|------------|
| `dm-linear` | Concatenate devices | LVM (basic LV) |
| `dm-striped` | Stripe across devices | LVM (striped LV) |
| `dm-mirror` | Mirror devices | LVM (mirror LV) |
| `dm-snapshot` | Copy-on-write snapshots | LVM snapshots |
| `dm-thin` | Thin provisioning | LVM thin pools |
| `dm-crypt` | Encryption (LUKS) | cryptsetup |
| `dm-multipath` | Multipathing (SAN) | multipathd |
| `dm-cache` | SSD caching for HDD | lvmcache |
| `dm-vdo` | Deduplication + compression | VDO (kernel 6.9+) |
| `dm-raid` | Software RAID via DM | mdadm alternative |
| `dm-integrity` | Data integrity | dm-integrity |

```bash
# View all device-mapper devices
dmsetup ls
dmsetup table
dmsetup info

# View DM device dependencies
dmsetup deps /dev/dm-0

# LVM stack
pvs     # Physical volumes
vgs     # Volume groups
lvs     # Logical volumes
```

### MD (Multiple Devices — Software RAID)

```bash
# View RAID arrays
cat /proc/mdstat

# RAID detail
mdadm --detail /dev/md0

# RAID levels
# md/raid0   — striping (performance, no redundancy)
# md/raid1   — mirroring (redundancy)
# md/raid5   — striping + parity (1 disk fault tolerance)
# md/raid6   — striping + double parity (2 disk fault tolerance)
# md/raid10  — mirror + stripe (performance + redundancy)
```

### SCSI Subsystem

The SCSI layer manages communication between the kernel and SCSI/FC/SAS devices.

```bash
# View SCSI devices
lsscsi
cat /proc/scsi/scsi

# SCSI host info
ls /sys/class/scsi_host/

# Rescan SCSI bus
echo "- - -" > /sys/class/scsi_host/host0/scan

# SCSI timeout
cat /sys/block/sda/device/timeout

# Change timeout (seconds)
echo 60 > /sys/block/sda/device/timeout
```

### NVMe

NVMe bypasses the SCSI layer entirely, connecting directly to the block layer via the `nvme` driver.

```bash
# List NVMe devices
nvme list

# NVMe device info
nvme id-ctrl /dev/nvme0

# NVMe namespaces
nvme list-ns /dev/nvme0

# NVMe SMART health
nvme smart-log /dev/nvme0

# View NVMe queue count
cat /sys/block/nvme0n1/device/queue_count
```

## I/O Path: Buffered vs Direct

### Buffered I/O (Default)

```
Application → VFS → Filesystem → Page Cache → Block Layer → Driver → Hardware
                                      ↑
                          (read-ahead, write-back)
```

- Reads are served from cache if available
- Writes go to page cache, flushed later by `pdflush`/`flush` threads
- `fsync()` forces write to stable storage

### Direct I/O (O_DIRECT)

```
Application → VFS → Filesystem → Block Layer → Driver → Hardware
                                (bypasses page cache)
```

- Used by databases (Oracle, PostgreSQL with `wal_sync_method`)
- Lower latency for large sequential I/O
- Application manages its own caching

```bash
# Check if a process uses direct I/O
cat /proc/<pid>/fdinfo/<fd> | grep flags
# O_DIRECT = 040000 in flags
```

## io_uring (Modern Async I/O)

The newest I/O submission interface (kernel 5.1+) — replaces `libaio`:

```bash
# Check if io_uring is available
grep io_uring /proc/kallsyms | head -1

# View io_uring usage per process
cat /proc/<pid>/fdinfo/<fd> | grep -i uring
```

## Tracing and Debugging the Stack

```bash
# Trace block I/O events
blktrace -d /dev/sda -o trace
blkparse -i trace

# One-liner: trace for 10 seconds
blktrace -d /dev/sda -w 10 -o - | blkparse -i -

# BCC/BPF tools (more modern)
biolatency       # I/O latency histogram
biosnoop         # Trace every I/O
biotop           # Top-like for block I/O
ext4slower       # Slow ext4 operations
xfsslower        # Slow XFS operations

# ftrace
echo 1 > /sys/kernel/debug/tracing/events/block/enable
cat /sys/kernel/debug/tracing/trace_pipe

# View I/O stack for a device
cat /sys/block/sda/stat
cat /proc/diskstats | grep sda
```

## Performance Tools by Layer

| Layer | Tool | What It Shows |
|-------|------|---------------|
| Application | `strace -e trace=read,write` | System calls |
| VFS | `/proc/sys/fs/file-nr` | Open file descriptors |
| Page Cache | `vmstat 1`, `free -h` | Cache hit/miss, dirty pages |
| Filesystem | `xfsslower`, `ext4slower` | Slow FS operations |
| Block | `iostat -x 1`, `blktrace` | IOPS, latency, queue depth |
| Device Mapper | `dmsetup status` | DM device stats |
| SCSI | `sg_inq`, `/proc/scsi/scsi` | SCSI device info |
| NVMe | `nvme smart-log` | Health, wear, errors |
| Hardware | `smartctl -a /dev/sda` | SMART health, temperature |

## Key sysfs Paths

| Path | Purpose |
|------|---------|
| `/sys/block/<dev>/queue/scheduler` | I/O scheduler |
| `/sys/block/<dev>/queue/nr_requests` | Queue depth |
| `/sys/block/<dev>/queue/read_ahead_kb` | Read-ahead size |
| `/sys/block/<dev>/queue/rotational` | 0=SSD, 1=HDD |
| `/sys/block/<dev>/queue/max_sectors_kb` | Max request size |
| `/sys/block/<dev>/stat` | I/O statistics |
| `/sys/block/<dev>/device/timeout` | SCSI command timeout |
| `/sys/block/<dev>/device/queue_depth` | Device queue depth |
| `/sys/class/scsi_host/host*/scan` | Trigger SCSI rescan |
| `/sys/class/fc_host/host*/port_state` | FC port status |
| `/sys/class/fc_remote_ports/rport-*/dev_loss_tmo` | FC path timeout |

## Key /proc Paths

| Path | Purpose |
|------|---------|
| `/proc/diskstats` | Per-device I/O statistics |
| `/proc/mdstat` | Software RAID status |
| `/proc/scsi/scsi` | SCSI device list |
| `/proc/meminfo` | Memory/cache stats (Cached, Dirty, Writeback) |
| `/proc/sys/vm/dirty_ratio` | Max dirty pages before blocking writes |
| `/proc/sys/vm/dirty_background_ratio` | Dirty pages before background flush starts |
| `/proc/sys/fs/file-nr` | Open/unused/max file descriptors |
