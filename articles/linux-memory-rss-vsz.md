# Linux Memory: RSS, VSZ, and Why RSS Alone Is Misleading

## Virtual Memory Basics

Every Linux process operates within its own virtual address space. The kernel, via the MMU (Memory Management Unit), translates virtual addresses to physical page frames. This abstraction allows:

- Processes to have isolated address spaces
- Memory to be allocated without being physically backed immediately (lazy allocation)
- Pages to be shared across processes (shared libraries, copy-on-write after fork)
- Unused pages to be swapped to disk

The key takeaway: **what a process thinks it has (virtual) and what it actually uses (physical) are two very different things**.

---

## VSZ — Virtual Memory Size

VSZ (also shown as VIRT in `top`) represents the total virtual address space of a process. It includes:

- The process binary (text segment)
- All linked shared libraries (entire library, not just what's loaded)
- Stack and heap allocations (including memory allocated but never touched)
- Memory-mapped files
- Memory that has been swapped out

```bash
# View VSZ for all processes
ps -eo pid,vsz,cmd --sort=-vsz | head
```

**VSZ is not a measure of physical memory usage.** A process can have a VSZ of several gigabytes while using only a few megabytes of actual RAM. This happens because:

1. `malloc()` can reserve address space without the kernel backing it with physical pages until the memory is actually written to (demand paging)
2. Shared libraries are mapped entirely into the virtual space but only the used pages are loaded
3. Memory-mapped files (e.g. via `mmap`) add to VSZ even if only a few pages are accessed

---

## RSS — Resident Set Size

RSS represents the amount of physical memory (RAM) currently held by a process. It includes:

- Pages from the process's own code and data that are in RAM
- Pages from shared libraries that are in RAM
- Stack and heap pages that are in RAM

It does **not** include:

- Swapped out pages
- Pages that are allocated but never accessed (not yet faulted in)

```bash
# View RSS for all processes
ps -eo pid,rss,cmd --sort=-rss | head

# RSS from /proc
cat /proc/<pid>/status | grep VmRSS
```

---

## Why RSS Is Misleading

RSS is better than VSZ for understanding physical memory usage, but it has a fundamental flaw: **it double-counts shared memory**.

### Problem 1: Shared Libraries

When `libc.so` is loaded into memory, every process that links to it shares the same physical pages. But RSS counts those pages fully for **each** process.

Example: 100 processes using a 2 MB shared library:
- Sum of RSS includes 100 × 2 MB = 200 MB for that library
- Actual physical memory used by the library: 2 MB

### Problem 2: Copy-on-Write After fork()

When a process calls `fork()`, the child initially shares all physical pages with the parent via copy-on-write (CoW). The pages are only duplicated when one of the processes writes to them.

- Right after fork: both parent and child report roughly the same RSS
- Actual additional physical memory used: near zero
- RSS of parent + child ≈ 2× the actual usage

This is why Redis with `BGSAVE` (which forks) can appear to use twice its memory in `ps` output even though CoW means physical usage barely increases — until pages start being modified.

### Problem 3: Memory-Mapped Files

A process that `mmap()`s a large file will have those pages counted in RSS once they're accessed, but those pages may be shared with other processes reading the same file or reclaimable by the kernel at any time (file-backed pages).

### The Sum Problem

If you sum RSS across all processes:

```bash
ps -eo rss | awk '{sum+=$1} END {print sum/1024 " MB"}'
```

The result will **exceed** your total physical RAM. This doesn't mean you're out of memory — it means shared pages are being counted multiple times.

---

## Better Metrics: PSS and USS

### PSS — Proportional Set Size

PSS divides shared pages proportionally among the processes sharing them.

If 3 processes share a library that occupies 30 pages in RAM, each process's PSS includes only 10 pages from that library.

**Summing PSS across all processes gives a realistic approximation of total physical memory usage.**

```bash
# View PSS for a process
cat /proc/<pid>/smaps_rollup | grep Pss
```

### USS — Unique Set Size

USS counts only the pages that are private to a process — not shared with anyone else. This tells you how much memory would be freed if you killed just that process.

```bash
# Private memory (USS approximation)
grep Private /proc/<pid>/smaps | awk '{sum+=$2} END {print sum " kB"}'
```

### Comparison

| Metric | Includes shared libs | Includes private memory | Summable across processes | Available in |
|--------|---------------------|------------------------|--------------------------|--------------|
| **VSZ** | Yes (full) | Yes (allocated) | No (meaningless) | `ps`, `top` |
| **RSS** | Yes (full) | Yes (resident) | No (overcounts) | `ps`, `top`, `/proc/pid/status` |
| **PSS** | Proportionally | Yes (resident) | Yes (accurate) | `/proc/pid/smaps`, `/proc/pid/smaps_rollup` |
| **USS** | No | Yes (resident) | No (undercounts) | `/proc/pid/smaps` |

---

## Reading /proc/\<pid\>/smaps

The `smaps` file provides per-mapping memory detail:

```
00400000-0048a000 r-xp 00000000 fd:03 960637       /bin/bash
Size:                552 kB      ← virtual size of this mapping
Rss:                 460 kB      ← physical pages resident
Pss:                 100 kB      ← proportional share
Shared_Clean:        452 kB      ← shared pages, unmodified
Shared_Dirty:          0 kB      ← shared pages, modified
Private_Clean:         8 kB      ← private pages, unmodified
Private_Dirty:         0 kB      ← private pages, modified
Referenced:          460 kB      ← pages accessed recently
Anonymous:             0 kB      ← not backed by a file
Swap:                  0 kB      ← pages swapped out
```

For a quick summary without parsing the full file:

```bash
# Aggregated view (Linux 4.14+)
cat /proc/<pid>/smaps_rollup
```

---

## Practical Commands

### Compare RSS vs PSS vs USS for a process

```bash
# ps doesn't support PSS/USS columns — they only exist in /proc/<pid>/smaps.
# This combines ps output with smaps data:
sudo bash -c '
  pid=<pid>
  ps -fp $pid
  echo "---"
  echo "RSS: $(grep VmRSS /proc/$pid/status | awk "{print \$2, \$3}")"
  echo "PSS: $(grep "^Pss:" /proc/$pid/smaps_rollup | awk "{print \$2, \$3}")"
  echo "USS: $(grep Private /proc/$pid/smaps_rollup | awk "{sum+=\$2} END {print sum, \"kB\"}")"
'

# On older kernels without smaps_rollup, use smaps:
sudo bash -c '
  pid=<pid>
  ps -fp $pid
  echo "---"
  echo "RSS: $(grep VmRSS /proc/$pid/status | awk "{print \$2, \$3}")"
  echo "PSS: $(grep "^Pss:" /proc/$pid/smaps | awk "{sum+=\$2} END {print sum, \"kB\"}")"
  echo "USS: $(grep Private /proc/$pid/smaps | awk "{sum+=\$2} END {print sum, \"kB\"}")"
'
```

### Find the real memory hogs (sorted by PSS)

```bash
# Requires root for reading other processes' smaps
for pid in $(ps -eo pid --no-headers); do
    pss=$(grep Pss /proc/$pid/smaps_rollup 2>/dev/null | awk '{sum+=$2} END {print sum}')
    [ -n "$pss" ] && echo "$pss $pid $(cat /proc/$pid/cmdline 2>/dev/null | tr '\0' ' ')"
done | sort -rn | head -20
```

### Use smem for a cleaner view

`smem` is a tool that reports PSS, USS, and RSS in a friendly format:

```bash
# Install
apt install smem    # Debian/Ubuntu
yum install smem    # RHEL/CentOS

# Usage
smem -r -k         # sorted by PSS, human-readable
smem -u            # per-user summary
smem -m            # per-mapping summary
```

### Use pmap for per-mapping detail

```bash
pmap -x <pid>      # extended format with RSS per mapping
pmap -X <pid>      # includes PSS (Linux 4.5+)
```

---

## Memory Overcommit

Linux allows processes to allocate more virtual memory than physically available (overcommit). This is controlled by:

```bash
# Check current policy
cat /proc/sys/vm/overcommit_memory
# 0 = heuristic (default) — kernel guesses if allocation is reasonable
# 1 = always overcommit — never refuse malloc (used by Redis, etc.)
# 2 = never overcommit — limit to swap + (ratio × physical RAM)

# Check the ratio (used when overcommit_memory=2)
cat /proc/sys/vm/overcommit_ratio   # default: 50 (percent)

# See committed vs limit
grep -i commit /proc/meminfo
# CommitLimit:    total virtual memory the kernel will allow
# Committed_AS:  total memory currently committed to processes
```

When overcommit is enabled and memory runs out, the OOM killer steps in and terminates processes. This is why VSZ alone tells you nothing about actual resource pressure — a process can `malloc` 100 GB, never touch it, and the system is fine.

---

## Summary

| Question | Use |
|----------|-----|
| How much address space does the process have? | VSZ |
| How much RAM is the process touching right now? | RSS |
| How much RAM is this process uniquely responsible for? | PSS |
| How much RAM would be freed if I kill only this process? | USS |
| Is my system actually running out of memory? | `free -m`, `/proc/meminfo`, `vmstat` |

**Rule of thumb:** Don't sum RSS to estimate system memory usage. Use PSS if you need per-process accounting that adds up to reality. Use USS to estimate the cost of killing a single process.
