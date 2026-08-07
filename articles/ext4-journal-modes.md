# ext4 Journal Modes

The ext3/ext4 journal (also called the **JBD2** — Journaling Block Device 2) protects filesystem metadata integrity during crashes. The `data=` mount option controls how **file data** (not just metadata) interacts with the journal, offering trade-offs between safety, performance, and consistency.

## The Three Modes

| Mode | Data handling | Metadata handling | Default |
|------|--------------|-------------------|---------|
| `ordered` | Flushed to disk **before** metadata is committed to journal | Journaled | ✅ Yes |
| `writeback` | No ordering — may be written before or after metadata journal commit | Journaled | No |
| `journal` | Written to journal **first**, then to final location on disk | Journaled | No |

All three modes journal metadata. The difference is what happens to **file data**.

---

## ordered (Default)

```
data=ordered
```

All file data is forced directly to the main filesystem **before** its metadata is committed to the journal. This guarantees that after a crash and journal replay, files either contain the old data or the new data — never garbage or stale blocks from deleted files.

**Behaviour:**
1. Application writes data blocks
2. Data blocks are flushed to their final on-disk location
3. Only then is the metadata (inode, directory entry, block pointers) committed to the journal
4. On crash recovery, metadata is replayed from the journal; data is already in place

**Guarantees:**
- After crash recovery, files never contain stale/unrelated data from previously deleted files
- Metadata is always consistent
- Data that was `fsync()`'d is durable

**Does NOT guarantee:**
- Data that was `write()`'d but not `fsync()`'d may be lost (the write may not have been flushed yet)

**Trade-offs:**
- Slight write latency compared to `writeback` (must flush data before committing metadata)
- Good balance of safety and performance for most workloads

---

## writeback

```
data=writeback
```

Data ordering is **not** preserved. File data may be written to the main filesystem before or after its metadata has been committed to the journal. The journal only records metadata changes.

**Behaviour:**
1. Application writes data blocks
2. Metadata is committed to the journal whenever the journal transaction commits
3. Data blocks may or may not have reached disk yet — no ordering enforced
4. On crash recovery, metadata is replayed; data may be outdated

**Guarantees:**
- Internal filesystem structure is always consistent (metadata integrity)
- No filesystem corruption after crash

**Does NOT guarantee:**
- File contents after crash may contain **stale data** (old blocks from previously deleted files can appear in newly allocated files)
- After crash, a file may have its new size (from metadata) but old/garbage content

**Trade-offs:**
- Highest throughput — no ordering constraint means the kernel can reorder writes freely
- Useful when data integrity is handled at the application level (databases with their own WAL)
- Risk of data exposure (old file contents visible in new files after crash)

> **Security note:** In `writeback` mode, after a crash, newly created files may expose data from previously deleted files. This can be a security concern in multi-tenant environments.

---

## journal (Full Data Journaling)

```
data=journal
```

All data is committed to the journal **before** being written to the main filesystem. Effectively double-writes all data.

**Behaviour:**
1. Application writes data blocks
2. Data blocks are written to the journal first
3. Journal transaction commits (data + metadata are now safely in the journal)
4. Data blocks are then written to their final location in the filesystem
5. On crash recovery, both data and metadata are replayed from journal

**Guarantees:**
- Strongest consistency — after crash, files always contain committed data
- No stale data exposure
- Data that reached the journal is recoverable even if the final write didn't complete

**Does NOT guarantee:**
- Writes that hadn't been committed to the journal yet are still lost

**Trade-offs:**
- **Slowest for writes** — all data is written twice (journal + final location)
- Uses more journal space (journal must be large enough to hold data)
- Can actually **improve read performance** in some workloads because writes are sequential to the journal, freeing the disk head for reads
- Higher journal disk wear

---

## Comparison

| Aspect | ordered | writeback | journal |
|--------|---------|-----------|---------|
| Write performance | Good | Best | Worst (2× writes) |
| Read performance | Good | Good | Can be better (sequential journal writes free disk for reads) |
| Crash safety (metadata) | ✅ Always consistent | ✅ Always consistent | ✅ Always consistent |
| Crash safety (data) | No stale data in files | Stale data possible | No stale data; committed data recoverable |
| Stale data exposure | No | Yes (security risk) | No |
| fsync() durability | Data + metadata durable | Metadata durable, data may not be | Data + metadata durable |
| Best for | General purpose, default | Throughput-critical, app-level journaling | Maximum data safety, small writes |

---

## Configuring the Journal Mode

### At Mount Time

```bash
# Mount with ordered (default — usually no need to specify)
mount -o data=ordered /dev/sda1 /mnt

# Mount with writeback
mount -o data=writeback /dev/sda1 /mnt

# Mount with full data journaling
mount -o data=journal /dev/sda1 /mnt
```

### In /etc/fstab

```
/dev/sda1  /data  ext4  defaults,data=writeback  0  2
/dev/sda2  /logs  ext4  defaults,data=journal    0  2
```

### Set as Default in the Filesystem Superblock

```bash
# Set default mount option (applied even without -o at mount time)
tune2fs -o journal_data /dev/sda1          # data=journal
tune2fs -o journal_data_ordered /dev/sda1  # data=ordered
tune2fs -o journal_data_writeback /dev/sda1  # data=writeback

# Remove default mount option
tune2fs -O ^journal_data /dev/sda1

# Check current default
tune2fs -l /dev/sda1 | grep "Default mount options"
```

### Verify Current Mode

```bash
# Check how a filesystem is mounted
mount | grep /dev/sda1
# Or
cat /proc/mounts | grep /dev/sda1

# Check via dmesg (shows mode at mount time)
dmesg | grep -i "ext4.*mounted"
# EXT4-fs (sda1): mounted filesystem with ordered data mode
```

---

## Journal Size

The journal size affects performance and recovery time:

```bash
# Check journal size
tune2fs -l /dev/sda1 | grep "Journal size"
dumpe2fs /dev/sda1 | grep "Journal size"

# Create filesystem with specific journal size (at mkfs time)
mkfs.ext4 -J size=256 /dev/sda1    # 256 MB journal

# Resize journal on existing filesystem (must be unmounted)
tune2fs -J size=256 /dev/sda1
```

Guidelines:
- `data=journal` needs a larger journal (holds data + metadata)
- Default journal size is typically 128 MB on large filesystems
- Larger journal = fewer forced commits = better throughput, but longer replay on crash
- Smaller journal = more frequent commits = lower throughput, but faster recovery

---

## Journal Location

The journal can be internal (default) or on a separate device for performance:

```bash
# Create filesystem with external journal on a fast device
mke2fs -O journal_dev /dev/nvme0n1p1          # create journal device
mkfs.ext4 -J device=/dev/nvme0n1p1 /dev/sda1  # create fs using external journal

# Move journal to external device (existing filesystem)
tune2fs -O ^has_journal /dev/sda1              # remove internal journal
e2fsck -f /dev/sda1
tune2fs -j -J device=/dev/nvme0n1p1 /dev/sda1 # add external journal
```

External journals on fast devices (NVMe) can significantly improve write performance, especially with `data=journal` mode.

---

## Journal Commit Interval

Controls how frequently the journal flushes transactions to disk:

```bash
# Default is 5 seconds
mount -o data=ordered,commit=5 /dev/sda1 /mnt

# More frequent commits (lower data loss window, higher IOPS)
mount -o data=ordered,commit=1 /dev/sda1 /mnt

# Less frequent commits (better throughput, larger loss window)
mount -o data=ordered,commit=30 /dev/sda1 /mnt
```

In `/etc/fstab`:

```
/dev/sda1  /data  ext4  defaults,data=ordered,commit=5  0  2
```

---

## Recommendations by Workload

| Workload | Recommended Mode | Reasoning |
|----------|-----------------|-----------|
| General server (default) | `ordered` | Safe default, good performance |
| Database (PostgreSQL, MySQL) | `writeback` | Database has its own WAL; no need for double-ordering. Best throughput. |
| Mail server (Postfix, Dovecot) | `ordered` or `journal` | Mail must not expose other users' data after crash |
| Web server (static files) | `ordered` | Standard safety, good read performance |
| Log aggregation (high-write) | `writeback` | Throughput matters more than log data safety |
| Financial / compliance | `journal` | Maximum data protection, audit trail integrity |
| NFS export | `ordered` | Clients expect data consistency after server crash |
| Temporary / scratch space | `writeback` | Data is disposable; maximise throughput |
| Virtual machine images | `writeback` | Guest OS handles its own journaling inside the image |

---

## barrier / nobarrier

Write barriers ensure that journal commits actually hit persistent storage before dependent data is written. Related to journal mode:

```bash
# Barriers enabled (default — safe)
mount -o barrier=1 /dev/sda1 /mnt

# Barriers disabled (dangerous without battery-backed write cache)
mount -o nobarrier /dev/sda1 /mnt
```

> **Warning:** Disabling barriers improves performance but risks data loss on power failure unless the storage has a battery-backed write cache (BBU/BBWC). Never disable barriers on consumer SSDs or drives without BBU.

---

## Troubleshooting

### Filesystem mounted as read-only after crash

The journal replay failed or detected errors:

```bash
# Check for errors
dmesg | grep -i "ext4.*error\|journal"

# Force fsck
umount /dev/sda1
e2fsck -f /dev/sda1

# If journal is corrupt
tune2fs -O ^has_journal /dev/sda1
e2fsck -f /dev/sda1
tune2fs -j /dev/sda1
```

### Cannot change data= mode on remount

The `data=` mode can only be set at initial mount, not changed via `remount`:

```bash
# This does NOT work:
mount -o remount,data=writeback /mnt

# Must unmount and remount:
umount /mnt
mount -o data=writeback /dev/sda1 /mnt
```

### Journal too small for data=journal mode

If the journal fills up frequently with `data=journal`, increase its size:

```bash
umount /dev/sda1
tune2fs -J size=512 /dev/sda1
mount /dev/sda1 /mnt
```

---

## Inodes

On ext2/3/4, the number of inodes is fixed at filesystem creation time — it cannot be increased on an existing volume. Every inode consumes 256 bytes (configurable as 128 on older systems).

Default: one inode per 16,384 bytes (16 KB) of disk space.

```bash
# Check inode count
tune2fs -l /dev/sda1 | grep "Inode count"
dumpe2fs /dev/sda1 | grep "Inode count"

# Check inode usage
df -i /dev/sda1

# Check inode size
tune2fs -l /dev/sda1 | grep "Inode size"

# Create filesystem with custom inode ratio (more inodes for small files)
mkfs.ext4 -i 8192 /dev/sda1    # one inode per 8KB

# Create filesystem with specific inode size
mkfs.ext4 -I 256 /dev/sda1
```

> **Note:** Modern filesystems like Btrfs and XFS use dynamic inode allocation — they don't have a fixed inode limit. ZFS does not use inodes at all.

---

## Mount Options

### defaults

`defaults` is shorthand for: `rw, suid, dev, exec, auto, nouser, async, relatime`

### atime vs relatime

With the `relatime` mount option (default since kernel 2.6.30), the access time (atime) is only updated when:

- The modified time (mtime) or change time (ctime) is newer than the current atime
- The atime is older than a defined interval (1 day by default on RHEL)

This drastically reduces write I/O compared to the old `atime` behaviour (which updated on every single read).

```bash
# Mount with no atime updates at all (best performance)
mount -o noatime /dev/sda1 /mnt

# Mount with relatime (default)
mount -o relatime /dev/sda1 /mnt

# Mount with strict atime (every read updates atime — avoid this)
mount -o strictatime /dev/sda1 /mnt
```

### Checking Mount Options

```bash
# Show current mount options for all filesystems
cat /proc/mounts

# Show mount options for a specific device
mount | grep /dev/sda1

# Find read-only mounted filesystems
grep "[[:space:]]ro[[:space:],]" /proc/mounts

# Show fstab entries with their resolved options
findmnt --fstab

# Check filesystem state (clean/not clean)
tune2fs -l /dev/sda1 | grep state
```

---

## mkfs — Creating ext4 Filesystems

```bash
# Standard ext4 filesystem
mkfs.ext4 /dev/sda1

# With label
mkfs.ext4 -L "data" /dev/sda1

# With custom inode ratio (more inodes)
mkfs.ext4 -i 8192 /dev/sda1

# With reduced reserved space (default 5%)
mkfs.ext4 -m 1 /dev/sda1

# Using a usage-type profile from /etc/mke2fs.conf
mkfs.ext4 -T largefile /dev/sda1      # fewer inodes, for large files
mkfs.ext4 -T largefile4 /dev/sda1     # even fewer inodes
mkfs.ext4 -T small /dev/sda1          # many inodes, for tiny files
mkfs.ext4 -T news /dev/sda1           # many inodes

# With specific journal size
mkfs.ext4 -J size=256 /dev/sda1

# With specific block size
mkfs.ext4 -b 4096 /dev/sda1
```

Usage types are defined in `/etc/mke2fs.conf`:

```bash
cat /etc/mke2fs.conf
```

---

## Converting ext3 to ext4

```bash
umount /dev/sda1
tune2fs -O extents,uninit_bg,dir_index /dev/sda1
e2fsck -pf /dev/sda1
mount /dev/sda1 /home
```

This enables ext4 features on an existing ext3 filesystem. Existing files retain their block maps; only new files use extents.

---

## Useful ext4 Commands

```bash
# Check filesystem state
tune2fs -l /dev/sda1 | grep state

# Inode count
tune2fs -l /dev/sda1 | grep "Inode count"
dumpe2fs /dev/sda1 | grep "Inode count"

# Force filesystem check every 30 mounts
tune2fs -c 30 /dev/sda1

# Force filesystem check every 3 months
tune2fs -i 3m /dev/sda1

# Disable automatic fsck
tune2fs -c 0 /dev/sda1
# or
tune2fs -c -1 /dev/sda1

# Check default mount options stored in superblock
tune2fs -l /dev/sda1 | grep "Default mount options"
```

---

## lsblk and findmnt

```bash
# Show UUIDs
lsblk -o +UUID

# Show serial numbers
lsblk -o +SERIAL

# Show filesystem info
lsblk -f

# Custom columns
lsblk -o name,size,mountpoint,label

# Show fstab entries
findmnt --fstab

# Show all mounted filesystems in tree format
findmnt

# Find where a device is mounted
findmnt /dev/sda1
```

---

## Forensic Tools

For low-level filesystem analysis and recovery:

| Tool | Purpose |
|------|---------|
| `mmls` | Display partition layout (Sleuth Kit) |
| `fsstat` | Display filesystem details and statistics (Sleuth Kit) |
| `fls` | List files and directories, including deleted (Sleuth Kit) |
| `debugfs` | Interactive ext2/3/4 filesystem debugger |
| `xfs_db` | XFS low-level metadata viewer |

---

## Quick Reference

```bash
# Check current journal mode
mount | grep /dev/sda1
dmesg | grep "ext4.*mounted"

# Mount with specific mode
mount -o data=ordered /dev/sda1 /mnt
mount -o data=writeback /dev/sda1 /mnt
mount -o data=journal /dev/sda1 /mnt

# Set default mode in superblock
tune2fs -o journal_data_ordered /dev/sda1
tune2fs -o journal_data_writeback /dev/sda1
tune2fs -o journal_data /dev/sda1

# Check journal size
tune2fs -l /dev/sda1 | grep "Journal size"

# Set journal size
tune2fs -J size=256 /dev/sda1

# Set commit interval
mount -o commit=5 /dev/sda1 /mnt

# Check default mount options
tune2fs -l /dev/sda1 | grep "Default mount options"
```
