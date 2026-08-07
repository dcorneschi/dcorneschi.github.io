# XFS Internals: Superblock and Addressing

XFS is a high-performance journaled filesystem originally developed by Silicon Graphics for IRIX, now the default filesystem on RHEL 7+. This article covers its internal structure — the superblock, allocation groups, and the block/inode addressing scheme.

## Overview of XFS

Key characteristics:

- **Journaled** — metadata changes are logged for crash recovery
- **Extent-based allocation** — files are stored in contiguous ranges of blocks, not individual block pointers
- **B+Tree directories** — scales efficiently from small to very large directories
- **Dynamic inode allocation** — inodes are created on demand, not pre-allocated at format time
- **Big-endian metadata** — all on-disk structures are stored in network byte order, regardless of CPU architecture
- **Allocation Groups** — the filesystem is split into independent sections (4 by default), allowing parallel I/O from multiple threads
- **Extended attributes** — arbitrary key-value metadata on files
- **CRC32 checksums** (v5) — metadata integrity verification
- **Default block size:** 4096 bytes
- **Default inode size:** 512 bytes

### XFS Versions

| Version | Introduced | Key features |
|---------|-----------|--------------|
| v4 | Original Linux port | Basic XFS structures |
| v5 | RHEL 7 / kernel 3.15+ | CRC32 checksums, metadata UUIDs, creation timestamps (btime), nanosecond resolution, self-describing metadata |

### Timestamps

XFS uses 32-bit signed Unix epoch timestamps (same as traditional Unix):
- `mtime` — last data modification
- `atime` — last access
- `ctime` — last metadata change
- `btime` — creation time (v5 only)

Each timestamp includes an additional 32-bit nanosecond field for sub-second precision.

> **Note:** XFS v4/v5 has the **Year 2038** problem due to signed 32-bit epoch timestamps. XFS bigtime (kernel 5.10+) extends timestamps to 2486.

---

## Allocation Groups

A single XFS filesystem is divided into multiple **Allocation Groups (AGs)** — 4 by default on RHEL. Each AG functions as a semi-independent filesystem with its own:

- Free block tracking (B+Tree)
- Free inode tracking (B+Tree)
- Superblock copy (for redundancy)

This design enables parallel allocation — multiple threads can write to different AGs simultaneously without contention, making XFS high-performing on multi-core systems.

```bash
# Check number of AGs and their size
xfs_info /mountpoint | grep -E "agcount|agsize"
# agcount=4, agsize=2427136 blks

# Or via xfs_db
xfs_db -r /dev/sda1
xfs_db> sb 0
xfs_db> print agcount
xfs_db> print agsize
```

---

## The Superblock

The superblock occupies the first 512 bytes of each AG. The primary superblock is in AG 0 (the very beginning of the filesystem). Superblocks in other AGs serve as backups.

Only the first 272 bytes are currently used (v5).

### Key Superblock Fields

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0–3 | 4 | Magic number | Always `0x58465342` ("XFSB") |
| 4–7 | 4 | Block size | In bytes (default: 4096) |
| 8–15 | 8 | Total blocks | Total blocks in the filesystem |
| 32–47 | 16 | UUID | Unique filesystem identifier |
| 48–55 | 8 | Journal start | First block of the journal (absolute address) |
| 56–63 | 8 | Root inode | Inode number of the root directory (usually 64) |
| 80–83 | 4 | RT extent size | Real-time extent size in blocks |
| 84–87 | 4 | AG size | Blocks per allocation group |
| 88–91 | 4 | AG count | Number of allocation groups |
| 96–99 | 4 | Journal blocks | Length of journal in blocks |
| 100–101 | 2 | Version/flags | Low nibble = version number |
| 102–103 | 2 | Sector size | Physical sector size (usually 512) |
| 104–105 | 2 | Inode size | Bytes per inode (default: 512) |
| 106–107 | 2 | Inodes/block | Number of inodes per block |
| 120 | 1 | log2(block size) | 12 = 4096 bytes |
| 121 | 1 | log2(sector size) | 9 = 512 bytes |
| 122 | 1 | log2(inode size) | 9 = 512 bytes |
| 123 | 1 | log2(inodes/block) | 3 = 8 inodes per block |
| 124 | 1 | log2(AG size) | Rounded up; used for address decoding |
| 127 | 1 | Max inode % | Maximum percentage of filesystem for inodes (default: 25%) |
| 128–135 | 8 | Allocated inodes | Total inodes allocated |
| 136–143 | 8 | Free inodes | Number of free inodes |
| 144–151 | 8 | Free blocks | Number of free data blocks |
| 180–183 | 4 | Inode alignment | In blocks |
| 184–187 | 4 | RAID unit | In blocks (0 if no RAID) |
| 188–191 | 4 | RAID stripe | In blocks (0 if no RAID) |
| 224–227 | 4 | CRC32 | Superblock checksum (v5 only) |

### Reading the Superblock

```bash
# Using xfs_db (read-only on mounted filesystem)
xfs_db -r /dev/sda1
xfs_db> sb 0
xfs_db> print

# Key fields to look at:
# blocksize, dblocks, agcount, agsize, uuid, rootino, logstart

# Using xfs_info (simpler output)
xfs_info /mountpoint
```

### Example xfs_db Output

```
xfs_db> sb 0
xfs_db> print
magicnum = 0x58465342
blocksize = 4096
dblocks = 9708544
uuid = e56c3b41-ca03-4b41-b15c-dd609cb7da71
logstart = 8388612
rootino = 64
agcount = 4
agsize = 2427136
```

This tells us:
- Block size: 4096 bytes
- Total filesystem blocks: 9,708,544 (≈ 37 GB)
- 4 allocation groups, each 2,427,136 blocks (≈ 9.3 GB)
- Root directory is inode 64
- Journal starts at absolute block 8,388,612

---

## Block and Inode Addressing

XFS uses a compound addressing scheme that packs the AG number and a relative offset into a single value. Understanding this is essential for locating data on disk.

### Absolute Block Addresses

A 64-bit absolute block address contains:

```
[  AG number  |  relative block offset within AG  ]
  upper bits        lower bits (length = log2(AG size))
```

The number of bits for the relative portion is the `log2(AG size)` value from superblock byte 124.

#### Example: Decoding a Block Address

Given: journal starts at block `0x800004`, `log2(AG size) = 22`

```
0x800004 in binary:  10 0000 0000 0000 0000 0100
                     ^^                          
                     AG=2   relative block = 4
```

- AG number: upper bits = 2
- Relative block: lower 22 bits = 4

**Physical block offset on disk:**

```
physical_block = (AG number × blocks_per_AG) + relative_block
               = (2 × 2,427,136) + 4
               = 4,854,276
```

Extract with dd:

```bash
dd if=/dev/mapper/centos-root bs=4096 skip=$((2*2427136 + 4)) count=1 | xxd | head
```

### Absolute Inode Addresses

Inode addresses are similar but use more bits for the relative portion because there are multiple inodes per block:

```
relative_inode_bits = log2(inodes/block) + log2(AG size)
                    = 3 + 22 = 25
```

```
[  AG number  |  relative inode number within AG  ]
  upper bits          lower 25 bits
```

#### Example: Locating an Inode on Disk

Given: inode 67,761,631 (`0x409f5df`), 8 inodes/block, AG size = 2,427,136 blocks

```
0x409f5df in binary:  010 0000 0 1001 1111 0101 1101 1111
                      ^^^
                      AG=2    relative inode = 0x9f5df (652,767)
```

**Calculate physical location:**

```
block_within_AG = relative_inode ÷ inodes_per_block
                = 652,767 ÷ 8 = 81,595 (integer division)

offset_within_block = relative_inode % inodes_per_block
                    = 652,767 % 8 = 7

physical_block = (AG × blocks_per_AG) + block_within_AG
               = (2 × 2,427,136) + 81,595 = 4,935,867
```

Extract the inode:

```bash
# Get the block, then extract the correct inode (each inode is 512 bytes)
dd if=/dev/mapper/centos-root bs=4096 skip=$((2*2427136 + 81595)) count=1 | \
    dd bs=512 skip=7 count=1 | xxd | head
```

### Using xfs_db for Address Conversion

`xfs_db` can convert addresses without manual calculation:

```bash
xfs_db -r /dev/mapper/centos-root

# Convert absolute block to AG number and AG-relative block
xfs_db> convert fsblock 0x800004 agno
0x2 (2)
xfs_db> convert fsblock 0x800004 agblock
0x4 (4)

# Convert inode to AG number, AG-relative inode, block, and offset
xfs_db> convert inode 67761631 agno
0x2 (2)
xfs_db> convert inode 67761631 agino
0x9f5df (652767)
xfs_db> convert inode 67761631 agblock
0x13ebb (81595)
xfs_db> convert inode 67761631 offset
0x7 (7)
```

---

## File Deletion Behaviour

XFS preserves significant metadata after file deletion:

- **Directory entries** — marked as unused but not zeroed
- **Inode extent data** — block ranges (extents) remain visible in the inode after deletion
- **Inode metadata** — timestamps, size, and permissions persist

This makes file recovery and forensic analysis more feasible compared to filesystems that aggressively zero metadata on deletion.

---

## XFS v5 Self-Describing Metadata

Every metadata structure in XFS v5 includes:

| Field | Purpose |
|-------|---------|
| Magic number | Identifies the structure type |
| UUID | Matches the filesystem UUID — detects data from other filesystems |
| Owner inode | Which inode this metadata belongs to |
| CRC32 checksum | Detects corruption |
| Log sequence number | When this structure was last modified |

This makes it possible to:
- Validate metadata integrity without context
- Carve filesystem structures from raw disk images
- Identify which filesystem a block belongs to

---

## Practical Commands

```bash
# Show filesystem info
xfs_info /mountpoint

# Read-only access to mounted filesystem structures
xfs_db -r /dev/sda1

# Print superblock
xfs_db -r /dev/sda1 -c "sb 0" -c "print"

# Show AG free space summary
xfs_db -r /dev/sda1 -c "freesp -s"

# Show inode details
xfs_db -r /dev/sda1 -c "inode 64" -c "print"

# Convert addresses
xfs_db -r /dev/sda1 -c "convert fsblock 0x800004 agno"

# Check filesystem consistency (unmounted)
xfs_repair -n /dev/sda1

# Dump superblock via raw hex
dd if=/dev/sda1 bs=512 count=1 | xxd | head -20

# Backup superblock locations (each AG starts with one)
# AG 0: block 0, AG 1: block agsize, AG 2: block 2*agsize, etc.
xfs_db -r /dev/sda1 -c "sb 0" -c "print agsize"
# Superblock 1 is at block offset = agsize
# Superblock 2 is at block offset = 2 × agsize
```

---

## Quick Reference

| Item | How to find |
|------|-------------|
| Block size | `xfs_info /mnt \| grep bsize` |
| Inode size | `xfs_info /mnt \| grep isize` |
| AG count/size | `xfs_info /mnt \| grep agcount` |
| UUID | `xfs_db -r /dev/sda1 -c "sb 0" -c "print uuid"` |
| Root inode | `xfs_db -r /dev/sda1 -c "sb 0" -c "print rootino"` |
| Journal start | `xfs_db -r /dev/sda1 -c "sb 0" -c "print logstart"` |
| Free blocks | `xfs_db -r /dev/sda1 -c "sb 0" -c "print fdblocks"` |
| Free inodes | `xfs_db -r /dev/sda1 -c "sb 0" -c "print ifree"` |
| Superblock backup | AG N superblock at block offset `N × agsize` |
| Decode block address | `xfs_db -r /dev -c "convert fsblock ADDR agno"` |
| Decode inode address | `xfs_db -r /dev -c "convert inode NUM agblock"` |
