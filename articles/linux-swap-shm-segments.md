# Linux Swap Usage: When Processes Aren't the Culprit

You check `free -m` and see swap is being used. You run a per-process swap check and the numbers don't add up. The total swap consumed by processes is a fraction of what the system reports. Where is the rest?

The answer is often **System V shared memory (SHM) segments** — memory that doesn't belong to any single process and can be swapped independently.

## Checking Per-Process Swap Usage

The standard approach to find swap consumers is iterating over `/proc/*/status`:

```bash
for file in /proc/*/status ; do
    awk '/VmSwap|Name/{printf $2 " " $3}END{ print ""}' $file
done | sort -k 2 -n -r | grep -v "^$" | awk 'NF==3' | head -20
```

The `awk 'NF==3'` filter removes kernel threads that don't have a `VmSwap` entry (they only print the name without a value).

To get the total swap used by all processes:

```bash
for file in /proc/*/status ; do
    awk '/VmSwap|Name/{printf $2 " " $3}END{ print ""}' $file
done | sort -k 2 -n -r | awk '{ SUM += $2 } END { print SUM/1024/1024 " GB"}'
```

If the result is significantly lower than what `free` or `swapon -s` reports, the remaining swap is consumed by something outside the per-process accounting — typically SHM segments.

## System V Shared Memory and Swap

System V shared memory (`shmget`/`shmat`) creates memory segments that exist independently of any single process. Multiple processes can attach to the same segment, and the kernel can swap these segments to disk just like any other page.

The key difference: **VmSwap in `/proc/<pid>/status` only accounts for the process's own private pages that are swapped out.** Shared memory segments that have been swapped are not attributed to any process in that counter.

## Inspecting SHM Segments

The kernel exposes all SHM segment details in `/proc/sysvipc/shm`:

```bash
cat /proc/sysvipc/shm
```

Example output:

```
       key      shmid perms                  size  cpid  lpid nattch   uid   gid  cuid  cgid      atime      dtime      ctime                   rss                  swap
-100203331          0   666              59828264  2353  2353      1     0     0     0     0 1696890521          0 1696890521                     0              59830272
-100203327          1   666              59828264  2353  2353      1     0     0     0     0 1696890521          0 1696890521                     0              59830272
         0      65538   600            8573157376 23253 19028     83  1101  1021  1101  1021 1698760860 1698760860 1698294790            3056758784            4030705664
 631185196          3   600                 73728 32237 18380     26  1101  1023  1101  1023 1698760801 1698760801 1697081967                  4096                     0
         0      65540   600               7827456 23253 19028     83  1101  1021  1101  1021 1698760860 1698760860 1698294790               7712768                114688
         0      32812   600            8573157376 32092 18929     80  1101  1021  1101  1021 1698760839 1698760869 1698204659            2413105152            4673310720
```

The important columns:

| Column | Meaning |
|--------|---------|
| `key` | Identifier used to create/access the segment |
| `shmid` | Kernel-assigned segment ID |
| `size` | Allocated size in bytes |
| `nattch` | Number of processes currently attached |
| `uid` | Owner's user ID |
| `rss` | Portion of the segment currently in physical RAM (bytes) |
| `swap` | Portion of the segment currently in swap (bytes) |

## Reading the Numbers

In the example above, two segments stand out:

| shmid | size | rss | swap | uid |
|-------|------|-----|------|-----|
| 65538 | ~8 GB | ~3.0 GB | ~4.0 GB | 1101 |
| 32812 | ~8 GB | ~2.4 GB | ~4.7 GB | 1101 |

Each segment is approximately 8 GB in size. The kernel keeps part of each in RAM and swaps the rest. Combined, these two segments alone account for nearly **8.7 GB of swap** — far more than the ~2.1 GB attributed to processes via `VmSwap`.

The UID 1101 identifies the owner (in this case an Oracle database user). Oracle's SGA (System Global Area) is typically backed by large SHM segments, and the kernel will swap portions of it when memory pressure exists.

## Calculating Total SHM Swap Usage

To sum up all swap used by SHM segments:

```bash
awk 'NR>1 { sum += $NF } END { printf "%.2f GB\n", sum/1024/1024/1024 }' /proc/sysvipc/shm
```

Or for a more detailed breakdown per segment:

```bash
awk 'NR>1 && $NF > 0 { printf "shmid=%s uid=%s size=%.2fGB rss=%.2fGB swap=%.2fGB\n", $2, $9, $4/1024/1024/1024, $(NF-1)/1024/1024/1024, $NF/1024/1024/1024 }' /proc/sysvipc/shm
```

## Why This Happens

The kernel treats SHM segment pages like any other swappable page. When there's memory pressure:

1. The kernel identifies pages that haven't been accessed recently
2. SHM pages that are not actively being read/written are candidates for swap-out
3. The segment continues to exist — processes can still attach and access it — but accessing a swapped page triggers a page fault and swap-in

For Oracle databases, the SGA can be tens of gigabytes. If the system doesn't have enough RAM to keep it all resident, portions will be swapped. This can cause significant performance degradation because database operations that hit swapped SGA pages will stall on disk I/O.

## What Can Be Done

### 1. Lock SHM in RAM (if enough memory exists)

Oracle can be configured to use huge pages, which are **not swappable**. This is the most common solution:

```bash
# Check if huge pages are configured
grep -i huge /proc/meminfo

# HugePages_Total:     0      ← not configured
# HugePages_Free:      0
# Hugepagesize:     2048 kB
```

Configuring huge pages for Oracle's SGA prevents it from being swapped entirely.

### 2. Increase Physical RAM

If the workload legitimately needs the memory, adding RAM eliminates the pressure that causes swapping.

### 3. Adjust vm.swappiness

Lowering `swappiness` makes the kernel less aggressive about swapping:

```bash
# Check current value
cat /proc/sys/vm/swappiness

# Reduce swap tendency (0-100, default is usually 30 or 60)
sysctl vm.swappiness=10
```

This doesn't prevent swapping entirely but shifts the kernel's preference toward reclaiming file-backed pages first.

### 4. Consult Oracle Support

For Oracle-specific SHM tuning, the DBA team or Oracle Support can advise on:

- Proper SGA sizing relative to available RAM
- Configuring `USE_LARGE_PAGES` in the Oracle parameter file
- AMM (Automatic Memory Management) vs ASMM (Automatic Shared Memory Management) settings
- Locking SGA in memory via the `LOCK_SGA` parameter

## Summary

| Question | Where to look |
|----------|---------------|
| How much swap are processes using? | `/proc/*/status` (VmSwap) |
| How much swap are SHM segments using? | `/proc/sysvipc/shm` (swap column) |
| Who owns the SHM segments? | `uid` column in `/proc/sysvipc/shm` |
| Total system swap usage? | `free -m` or `swapon -s` |

**When per-process swap doesn't add up to total swap, check `/proc/sysvipc/shm`.** The gap is almost always SHM segments — and on database servers, it's usually the Oracle SGA or similar large shared memory allocations that the kernel has partially swapped out.
