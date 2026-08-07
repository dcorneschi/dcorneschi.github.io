# Linux Disk I/O Internals

How data actually moves between applications and storage hardware on Linux — page cache, standard I/O, direct I/O, memory-mapped I/O, and block alignment.

## I/O Interfaces

Linux provides several interfaces for disk I/O, each with different trade-offs:

| Interface | Functions | Buffered | Use case |
|-----------|-----------|----------|----------|
| Standard I/O (stdio) | `fopen`, `fwrite`, `fread`, `fflush`, `fclose` | User-space + page cache | General application I/O |
| System calls | `open`, `write`, `read`, `fsync`, `close` | Page cache only | Low-level file operations |
| Direct I/O | `open` with `O_DIRECT`, `write`, `read` | None (bypasses page cache) | Databases with custom caching |
| Memory-mapped I/O | `open`, `mmap`, `msync`, `munmap` | Page cache (via virtual memory) | Random access patterns, shared memory |
| Vectored I/O | `writev`, `readv` | Page cache | Scatter/gather, multiple buffers in one syscall |

---

## Sectors, Blocks, and Pages

Data moves through three layers, each with its own addressing unit:

```
Application
    ↓
Virtual Memory    → Pages (typically 4096 bytes)
    ↓
Filesystem        → Blocks (512, 1024, 2048, or 4096 bytes)
    ↓
Block Device      → Sectors (typically 512 bytes)
    ↓
Hardware (HDD/SSD)
```

| Unit | Size | Set by |
|------|------|--------|
| Sector | 512 bytes (some modern drives: 4096) | Hardware — smallest unit the disk can transfer |
| Block | 512–4096 bytes | Filesystem — smallest addressable unit of the filesystem |
| Page | 4096 bytes (default on x86_64) | Kernel virtual memory — unit of caching and memory mapping |

Virtual memory pages map to filesystem blocks, which map to block device sectors.

```bash
# Check sector size
cat /sys/block/sda/queue/hw_sector_size      # hardware sector
cat /sys/block/sda/queue/physical_block_size  # physical block
cat /sys/block/sda/queue/logical_block_size   # logical block

# Check filesystem block size
tune2fs -l /dev/sda1 | grep "Block size"     # ext4
xfs_info /mountpoint | grep bsize            # XFS

# Check page size
getconf PAGESIZE
```

---

## Page Cache

The page cache is a transparent layer between applications and disk. All standard reads and writes go through it.

### How It Works

**Reads:**
1. Application calls `read()`
2. Kernel checks if the data is in the page cache
3. If **hit**: data is copied from cache to user-space buffer (no disk access)
4. If **miss** (page fault): data is loaded from disk into page cache, then copied to user-space

**Writes:**
1. Application calls `write()`
2. Data is copied from user-space buffer to the page cache
3. The page is marked **dirty**
4. Kernel later writes dirty pages to disk during **writeback** (flush)
5. Subsequent `read()` calls get the updated data from page cache, not the (now outdated) disk

### Why Page Cache Exists

Two locality principles drive its effectiveness:

- **Temporal locality** — recently accessed data is likely to be accessed again soon
- **Spatial locality** — data near recently accessed data is likely to be needed (drives **prefetch**)

Benefits:
- Serves repeated reads from memory instead of disk
- Coalesces adjacent writes before flushing
- Delays writes to batch them (reduces IOPS)
- Allows sharing cached pages across processes

### Page Cache Behaviour

```bash
# View page cache usage
free -h    # buff/cache column

# Detailed stats
cat /proc/meminfo | grep -E "Cached|Dirty|Writeback|Buffers"

# Dirty page writeback thresholds
sysctl vm.dirty_ratio              # % of RAM before writer blocks (default: 20)
sysctl vm.dirty_background_ratio   # % of RAM before background writeback starts (default: 10)
sysctl vm.dirty_expire_centisecs   # age (cs) before dirty page is eligible for writeback (default: 3000 = 30s)
sysctl vm.dirty_writeback_centisecs  # how often writeback thread wakes up (default: 500 = 5s)

# Tune: reduce dirty page thresholds for lower latency
sysctl -w vm.dirty_ratio=5
sysctl -w vm.dirty_background_ratio=2

# Drop page cache (for benchmarking only)
sync; echo 3 > /proc/sys/vm/drop_caches
```

### Cache Eviction

When the page cache is full:
- Least Recently Used (LRU) pages are evicted
- Dirty pages are written back to disk before eviction
- Clean pages are discarded immediately

### Page Cache vs Buffer Cache

These are historically distinct concepts:

- **Page cache** — uses *file-based* addressing and MMU page granularity. Caches file data.
- **Buffer cache** — uses *disk block* addressing. Caches raw block device data (filesystem metadata, non-file-backed blocks).

They were partially merged in Linux 2.4, but remain fundamentally different. The page cache is closer to applications and can cache data that has no direct disk mapping (e.g. network filesystems). In modern kernels, `free` shows them combined in the `buff/cache` column, but they serve different purposes internally.

---

## Standard I/O (Buffered I/O)

The default mode — all I/O goes through the page cache.

```c
// Simplified flow:
fd = open("/path/to/file", O_RDWR);
write(fd, buffer, size);   // → copies to page cache (dirty page)
read(fd, buffer, size);    // → served from page cache if present
fsync(fd);                 // → forces dirty pages to disk
close(fd);
```

### User-Space Buffering vs Kernel Buffering

There are two independent buffering layers:

| Layer | What | Controlled by |
|-------|------|---------------|
| User-space (stdio) | `FILE*` buffer in libc | `setvbuf()`, `fflush()` |
| Kernel (page cache) | Cached pages in RAM | `fsync()`, `fdatasync()`, `sync()` |

`fwrite()` writes to the libc buffer first. `fflush()` pushes it to the kernel page cache. `fsync()` pushes dirty pages from page cache to disk. To guarantee data is on disk, you need both `fflush()` + `fsync()` (or use raw syscalls with `fsync()`).

### Ensuring Data Reaches Disk

```bash
# fsync — flushes file data and metadata to disk
fsync(fd);

# fdatasync — flushes file data only (not metadata like timestamps)
fdatasync(fd);

# sync — flushes ALL dirty pages system-wide
sync
```

> **Important:** A successful `write()` only means data reached the page cache. Without `fsync()`, data can be lost on power failure. Errors from the actual disk write may only surface during `fsync()` or `close()`.

### Delayed Errors

Because writes go to the page cache first, disk errors (bad sectors, full disk, I/O errors) may not be reported until:
- `fsync()` is called
- `close()` is called
- The kernel performs writeback

This is why databases always `fsync()` after critical writes.

---

## Direct I/O (O_DIRECT)

Bypasses the page cache entirely. Data moves directly between user-space buffers and the block device.

```c
fd = open("/path/to/file", O_RDWR | O_DIRECT);
write(fd, aligned_buffer, size);  // → goes straight to disk
read(fd, aligned_buffer, size);   // → reads from disk, not cache
```

### When to Use Direct I/O

- **Database engines** that implement their own buffer/cache (PostgreSQL WAL, MySQL InnoDB, RocksDB)
- When you need fine-grained control over what stays in memory
- When data won't be reused (write-once logs, streaming writes)
- When page cache eviction would hurt other workloads
- **Compressed/serialized data** — the page cache stores data as it appears on disk (compressed). With O_DIRECT + userspace caching, you can cache the decompressed form and avoid repeated CPU-costly decompression on every read

### When NOT to Use Direct I/O

- General application I/O (page cache helps you)
- Small random reads (page cache prefetch helps here)
- When multiple processes access the same file (page cache shares pages)

### Trade-offs

| Aspect | Standard I/O | Direct I/O |
|--------|-------------|------------|
| Caching | Kernel page cache | Application must cache |
| Write latency | Low (writes to RAM) | Higher (writes to disk) |
| Read latency (cached) | Very low (from RAM) | Always from disk |
| CPU overhead | Higher (copy to/from page cache) | Lower (no extra copy) |
| Memory usage | Uses page cache (shared) | Only application buffers |
| Alignment | Not required | Required (sector-aligned) |

### Block Alignment Requirement

Direct I/O uses DMA (Direct Memory Access) to transfer data straight between user-space buffers and the block device. Because DMA bypasses intermediate kernel buffers, all operations must be aligned to the logical block size (usually 512 bytes):

- Starting offset must be a multiple of 512 (or the logical block size)
- Buffer size must be a multiple of 512
- Buffer memory address must be aligned (use `posix_memalign()`)

Unaligned direct I/O will fail with `EINVAL`.

```bash
# Check required alignment
cat /sys/block/sda/queue/logical_block_size   # typically 512

# Check if file is opened with O_DIRECT
cat /proc/<PID>/fdinfo/<FD> | grep flags
```

### Why Alignment Matters

```
Aligned write (good):
[  block 0  ][  block 1  ][  block 2  ]
[============write=============]
→ Writes exactly to block boundaries, one device operation

Unaligned write (bad):
[  block 0  ][  block 1  ][  block 2  ]
       [=======write========]
→ Crosses boundaries: must read-modify-write blocks 0 and 2
```

Even with standard buffered I/O, block-aligned operations perform better because they avoid partial block reads/writes at the device level.

### Real-World Usage

- **PostgreSQL** — uses `O_DIRECT` for WAL (write-ahead log) because WAL writes must be durable and won't be re-read from cache
- **MySQL InnoDB** — uses `O_DIRECT` for data and log files (`innodb_flush_method = O_DIRECT`)
- **RocksDB** — uses `O_DIRECT` for SST files, verifies block alignment upfront
- **Oracle** — uses async direct I/O for datafiles

---

## Memory-Mapped I/O (mmap)

Maps a file (or portion of it) directly into the process's virtual address space. Reads and writes become memory accesses — the kernel handles paging transparently.

```c
fd = open("/path/to/file", O_RDWR);
ptr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

// Access file as memory
ptr[0] = 'H';          // write (dirty page)
char c = ptr[100];     // read (page fault if not cached)

msync(ptr, size, MS_SYNC);  // flush changes to disk
munmap(ptr, size);
close(fd);
```

### When mmap Helps

- Random read access patterns (no syscall overhead per read)
- Large files accessed sparsely (only accessed pages are loaded)
- Shared memory between processes (`MAP_SHARED`)
- Memory-mapped databases (LMDB, SQLite in WAL mode)

### When mmap Hurts

- Sequential I/O (standard `read()` with prefetch is often faster)
- When you need control over eviction
- When page faults introduce unpredictable latency
- Very large files on 32-bit systems (address space limit)

---

## Vectored I/O (readv / writev)

Reads into or writes from multiple non-contiguous buffers in a single syscall. Avoids copying data into a single contiguous buffer.

```c
struct iovec iov[3];
iov[0].iov_base = header;  iov[0].iov_len = 64;
iov[1].iov_base = payload; iov[1].iov_len = 4096;
iov[2].iov_base = footer;  iov[2].iov_len = 32;

writev(fd, iov, 3);  // single syscall writes all three buffers
```

Benefits:
- Atomic (from the filesystem's perspective)
- Fewer syscalls than multiple `write()` calls
- No need to assemble a contiguous buffer (saves a memory copy)

---

## Page Cache Hints (fadvise / madvise)

Tell the kernel about your access pattern so it can optimize prefetch and eviction:

```bash
# posix_fadvise() hints:
POSIX_FADV_SEQUENTIAL  # will read sequentially → aggressive prefetch
POSIX_FADV_RANDOM      # will read randomly → disable prefetch
POSIX_FADV_WILLNEED    # will need this data soon → prefetch now
POSIX_FADV_DONTNEED    # done with this data → evict from cache
POSIX_FADV_NOREUSE     # data will be accessed once → don't cache long
```

Practical use:

```bash
# Evict a file from page cache (from command line)
dd if=/path/to/file of=/dev/null bs=4k iflag=nocache

# Or using vmtouch
vmtouch -e /path/to/file     # evict from cache
vmtouch -t /path/to/file     # touch (load into cache)
vmtouch /path/to/file        # show cache status
```

---

## Nonblocking Filesystem I/O (A Common Misconception)

Unlike network sockets, there is no true nonblocking I/O for regular files:

- `O_NONBLOCK` is **ignored** for regular (on-disk) files — block device operations are considered non-blocking by the kernel because there's a bounded completion time
- `select()`, `poll()`, and `epoll()` do **not** work for monitoring regular file descriptors — they always report them as "ready"
- Filesystem I/O can still block on disk latency, but the kernel doesn't provide the same non-blocking infrastructure as for sockets

For truly asynchronous file I/O, use:
- `io_uring` (Linux 5.1+) — the modern approach
- `libaio` / `io_submit()` (older AIO interface)
- Thread pools that perform blocking I/O in background threads

---

## I/O Flow Summary

```
                        Application
                            |
            ┌───────────────┼───────────────┐
            │               │               │
     Standard I/O      Direct I/O       mmap
            │               │               │
            ▼               │               ▼
      ┌──────────┐          │         ┌──────────┐
      │Page Cache│          │         │Page Cache│
      └────┬─────┘          │         └────┬─────┘
           │                │              │
           ▼                ▼              ▼
      ┌─────────────────────────────────────┐
      │         Block Layer / I/O Scheduler │
      └────────────────┬────────────────────┘
                       ▼
              ┌─────────────────┐
              │  Block Device   │
              │  (HDD/SSD/NVMe) │
              └─────────────────┘
```

---

## Practical Implications

### For Database Administrators

- Understand whether your DB uses buffered or direct I/O
- With direct I/O, the DB's own buffer pool is your cache — size it appropriately
- `fsync()` latency = actual disk latency (not hidden by page cache)
- Monitor dirty page ratios — high `vm.dirty_ratio` can cause latency spikes during writeback

### For Application Developers

- Standard buffered I/O is correct for most workloads
- Don't assume `write()` means data is on disk — use `fsync()` for durability
- Align I/O to block/page boundaries even with buffered I/O (fewer partial block operations)
- Use `fadvise(POSIX_FADV_DONTNEED)` after processing large files to free page cache

### For System Administrators

- High `Dirty` in `/proc/meminfo` = pending writes that could be lost on crash
- Tune `vm.dirty_*` parameters based on workload (latency vs throughput)
- Monitor writeback queue depth — sustained high queues indicate storage is too slow
- `O_DIRECT` workloads won't benefit from adding RAM (page cache unused)

---

## Quick Reference

```bash
# Check page cache usage
free -h
cat /proc/meminfo | grep -E "Cached|Dirty|Writeback"

# Check dirty page tunables
sysctl -a | grep vm.dirty

# Flush all dirty pages to disk
sync

# Check a file's page cache status
vmtouch /path/to/file

# Check sector/block sizes
cat /sys/block/sda/queue/logical_block_size
cat /sys/block/sda/queue/physical_block_size
tune2fs -l /dev/sda1 | grep "Block size"
getconf PAGESIZE

# Check if a process uses O_DIRECT
cat /proc/<PID>/fdinfo/* | grep flags
# O_DIRECT = 040000 in the flags field

# Monitor I/O writeback
cat /proc/meminfo | grep -E "Dirty|Writeback"
watch -n 1 'grep -E "Dirty|Writeback" /proc/meminfo'
```
