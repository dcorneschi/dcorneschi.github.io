# Understanding **vmstat** Output

## Overview

`vmstat` (virtual memory statistics) is one of the most essential tools for quickly assessing system health on Linux and other Unix-like operating systems. It provides a single-line summary of CPU, memory, I/O, and process scheduling activity — making it an ideal first-response diagnostic tool when a system feels slow or unresponsive.

This article explains how to use `vmstat`, what each field means, how the tool works internally, and how to identify common performance problems.

## What Does vmstat Do?

`vmstat` reports information about processes, memory, paging, block I/O, traps, disks, and CPU activity. It reads from `/proc/stat`, `/proc/meminfo`, `/proc/vmstat`, and other procfs files to compute per-interval deltas.

Unlike `top` or `htop`, `vmstat` is non-interactive and produces a compact, scriptable output — ideal for quick checks and for piping into monitoring systems.

### How vmstat Works Internally

`vmstat` reads kernel counters from `/proc` and computes the difference between two consecutive samples taken at the specified interval. Fields expressed as rates (e.g., per second) are calculated by dividing the delta by the sample interval.

### Important: The First Sample

The **first line** of `vmstat` output represents averages since the last reboot, not the current interval. This is because there is no previous sample to compare against.

Always use at minimum:

```bash
vmstat 1 5
```

This outputs 5 samples at 1-second intervals. Ignore the first line and focus on lines 2-5 for current activity.

To skip the first line entirely in scripts:

```bash
vmstat 1 | tail -n +3
```

## Basic Usage

```bash
# Default output (single snapshot since boot)
vmstat

# 1-second intervals, continuous
vmstat 1

# 2-second intervals, 10 samples
vmstat 2 10

# Display timestamps with each line
vmstat -t 1

# Display in megabytes instead of kilobytes
vmstat -S M 1

# Display active/inactive memory breakdown
vmstat -a 1

# Display disk statistics (all disks)
vmstat -d

# Display disk statistics for a specific disk (with header)
vmstat -d | head -2; vmstat -d | grep sda

# Monitor a specific disk every 1 second for 60 seconds
# Note: counters are cumulative — identical lines mean no new I/O during that interval.
# To see per-second deltas, use iostat -x sda 1 60 instead.
vmstat -d | head -2; vmstat -d 1 60 | grep sda

# Display partition statistics
vmstat -p sda1

# Display event counters and memory stats (summary mode)
vmstat -s

# Display total forks since boot (process creation rate)
vmstat -f

# Display slab/cache info (kernel memory allocator)
vmstat -m

# Wide output format (prevents column truncation on high-memory systems)
vmstat -w 1
```

## Output Format

### Standard Output (Linux)

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 245612  89712 1204536    0    0     5    12  152  312  3  1 95  1  0
```

### Wide Output (vmstat -w)

```
procs -----------------------memory---------------------- ---swap-- -----io---- -system-- --------cpu--------
 r  b         swpd         free         buff        cache   si   so    bi    bo   in   cs  us  sy  id  wa  st
 1  0            0       245612        89712      1204536    0    0     5    12  152  312   3   1  95   1   0
```

Use `-w` on systems with large amounts of RAM where the default column widths cause numbers to run together.

### Disk Output (vmstat -d)

```
disk- ------------reads------------ ------------writes----------- -----IO------
       total   merged  sectors      ms   total   merged  sectors      ms  cur  sec
sda    82431     1245  4512896  123456   45123     8921  3601840   98712    0   89
sdb     1204       32    96320    2840     512       64    40960    1520    0    2
```

| Column | Meaning |
|--------|---------|
| `total` (reads) | Total completed read requests |
| `merged` (reads) | Total read requests merged by the I/O scheduler |
| `sectors` (reads) | Total sectors read |
| `ms` (reads) | Total milliseconds spent reading |
| `total` (writes) | Total completed write requests |
| `merged` (writes) | Total write requests merged by the I/O scheduler |
| `sectors` (writes) | Total sectors written |
| `ms` (writes) | Total milliseconds spent writing |
| `cur` | I/O requests currently in progress |
| `sec` | Total seconds spent doing I/O |

**Notes:**
- All values are cumulative counters since boot (not per-second rates)
- Use `vmstat -d | head -2; vmstat -d | grep sda` to filter output to a specific disk while keeping the header
- `cur` is the only non-cumulative field — it reflects the instantaneous in-flight I/O count
- Data is sourced from `/proc/diskstats`

### Partition Output (vmstat -p)

```
sda1  reads  read sectors  writes  requested writes
       82431    4512896      45123       3601840
```

| Column | Meaning |
|--------|---------|
| `reads` | Total number of reads issued to this partition |
| `read sectors` | Total sectors read from this partition |
| `writes` | Total number of writes issued to this partition |
| `requested writes` | Total sectors written to this partition |

**Notes:**
- Partition stats do not include merge or timing information — use `vmstat -d` on the parent disk or `iostat -x` for that level of detail
- Useful for identifying which partition is generating the most I/O on a multi-partition disk

## Field Definitions

### Procs (Process) Section

#### `r` — Run Queue Length

The number of runnable processes (running + waiting for CPU time).

- Includes processes currently executing on a CPU **and** those ready to run but waiting for a CPU
- **Rule of thumb:** if `r` consistently exceeds the number of logical CPUs, the system is CPU-saturated
- A sustained `r` value of 2x the CPU count or higher indicates severe CPU contention
- `r` should always be higher than `b` — if `b` > `r`, it usually means you have an I/O bottleneck (more processes are stuck waiting on I/O than actively running on CPU)
- High `r` combined with high `cs` suggests possible lock contention

**Example:** On a 4-core system, `r = 8` means 4 processes are running and 4 are waiting — CPU is a bottleneck. An `r` of 13 is acceptable on a 16-CPU server, while 16 would be a serious problem on a 12-CPU server.

**Checking CPU count for comparison:**

```bash
cat /proc/cpuinfo | grep processor | wc -l
```

**Remediation for sustained CPU overload:**
1. Add more processors (CPUs) to the server
2. Load balance by rescheduling large batch tasks to off-peak hours

#### `b` — Blocked Processes

The number of processes in uninterruptible sleep (D state) — waiting for a resource (e.g., filesystem I/O, inode lock).

- Typically processes waiting for I/O to complete (disk, network, NFS)
- A non-zero `b` column usually indicates I/O bottlenecks
- On Linux, `r + b` roughly corresponds to the instantaneous system load average
- Sustained high `b` values warrant investigation with `iostat -x` or `iotop`
- High `b` values indicate slow disks or I/O conditions associated with paging due to lack of memory

**Finding blocked processes:**

```bash
ps -eLo state,pid,cmd | grep ^D
```

**Common causes of high `b`:**
- Slow or stalled storage (SAN, NFS, local disk issues)
- Heavy paging/swapping activity
- Processes waiting on NFS mounts that are unresponsive
- Kernel bugs causing processes to get stuck in D state

### Memory Section

All values are in kilobytes by default. Use `-S M` for megabytes, `-S m` for mebibytes.

#### `swpd` — Swap Used

Amount of swap space currently in use (KiB).

- Non-zero `swpd` is not necessarily a problem — pages can sit in swap untouched for long periods
- **What matters is activity:** watch `si` and `so` columns for active swapping
- A system with `swpd > 0` but `si = 0` and `so = 0` has swapped out cold pages but is not actively under memory pressure

**Checking configured swap:**

```bash
cat /proc/swaps
```

#### `free` — Free Memory

Amount of idle (completely unused) memory (KiB).

- On modern Linux, free memory is often intentionally low because the kernel uses available memory for page cache (`buff`/`cache`)
- **Low `free` alone does not mean the system is out of memory**
- To understand actual memory availability: `free + buff + cache` (approximately), or better yet use `free -h` and look at the "available" column

#### `buff` — Buffer Memory

Memory used for kernel buffer cache (KiB).

- Buffer cache holds raw block device data (metadata, directory entries, raw device I/O)
- Typically small compared to page cache
- Used by the block layer to cache disk metadata

#### `cache` — Page Cache

Memory used for page cache (KiB).

- Page cache holds file content read from or written to disk
- This is the kernel's primary mechanism for speeding up file I/O
- Cache memory is reclaimable — the kernel will shrink it when applications need more memory
- High cache usage is **healthy** — it means the kernel is effectively using idle memory to speed up I/O

#### `inact` — Inactive Memory (with `-a` flag)

Memory on the inactive list (KiB). Pages that have not been recently accessed and are candidates for reclaim.

#### `active` — Active Memory (with `-a` flag)

Memory on the active list (KiB). Pages that have been recently accessed and are less likely to be reclaimed.

### Swap Section

#### `si` — Swap In (Pages Read from Swap)

Amount of memory swapped in from disk per second (KiB/s).

- Non-zero `si` means the system is reading pages back from swap into RAM
- This happens when a process accesses a page that was previously swapped out
- Sustained non-zero `si` indicates the system doesn't have enough RAM for the working set

#### `so` — Swap Out (Pages Written to Swap)

Amount of memory swapped out to disk per second (KiB/s).

- Non-zero `so` means the system is actively moving pages from RAM to swap
- This is the more critical indicator of memory pressure
- Sustained non-zero `so` degrades performance significantly, especially for latency-sensitive workloads

**Interpreting swap activity:**

| `si` | `so` | Interpretation |
|------|------|----------------|
| 0 | 0 | No memory pressure (even if `swpd > 0`) |
| > 0 | 0 | Process accessing previously swapped pages (minor issue) |
| 0 | > 0 | System actively pushing pages to swap (memory pressure building) |
| > 0 | > 0 | Active thrashing — pages being swapped in and out continuously (critical) |

In ideal conditions, `si` and `so` should be at 0 most of the time. More than 10 blocks per second sustained is a concern.

### I/O Section

**Note:** The memory, swap, and I/O statistics are in blocks, not in bytes. In Linux, blocks are usually 1,024 bytes (1 KiB).

#### `bi` — Blocks In

Blocks received from a block device per second (blocks/s).

- 1 block = 1 KiB on most modern Linux systems (historically 512 bytes on older kernels)
- Represents reads from disk (or reads from swap, which are also counted here)
- High `bi` with high `b` (blocked processes) suggests I/O-bound reads

#### `bo` — Blocks Out

Blocks sent to a block device per second (blocks/s).

- Represents writes to disk (or writes to swap)
- Periodic spikes in `bo` are normal — the kernel's pdflush/writeback threads flush dirty pages in batches
- Sustained high `bo` with high `wa` (I/O wait) suggests the system is write-bound

**Note:** `bi` and `bo` include swap I/O. If you see high `bi`/`bo` along with non-zero `si`/`so`, the I/O is at least partially due to swapping.

### System Section

#### `in` — Interrupts Per Second

Total number of interrupts per second, including the clock interrupt.

- Hardware interrupts from devices (network cards, disk controllers, timers)
- High interrupt rates can indicate heavy network traffic, disk I/O, or a misbehaving device
- Compare against baseline — what's "high" depends entirely on the workload
- On modern multi-queue NIC systems, interrupt rates can be very high during heavy network load without indicating a problem

#### `cs` — Context Switches Per Second

Total number of context switches per second.

- A context switch occurs when the currently running thread is different from the previously running thread, so it is taken off the CPU
- Voluntary context switches: a process yields the CPU (e.g., waiting for I/O)
- Involuntary context switches: the scheduler preempts a process (time slice expired, higher priority process ready)
- It is not uncommon for `cs` to be approximately the same as the device interrupt rate (`in` column)
- If `cs` is higher than `sy`, the system is doing more context switching than actual work
- High `r` with high `cs` suggests possible **lock contention**
- High context switch rates can indicate:
  - Too many threads competing for too few CPUs
  - Excessive locking/contention causing threads to sleep and wake frequently
  - Small, frequent I/O operations causing rapid voluntary yields
- **Rule of thumb:** context switches above 10,000-50,000/s per CPU may warrant investigation, but this is highly workload-dependent

**Lock contention** occurs when one process or thread attempts to acquire a lock held by another process or thread. The more granular the available locks, the less likely one process/thread will request a lock held by another (e.g., locking a row rather than the entire table).

### CPU Section

All CPU values are percentages of total CPU time across all CPUs.

#### `us` — User Time

Time spent running user-space (non-kernel) code.

- Application code, libraries, user-space processes
- High `us` indicates CPU-intensive application workloads
- This is generally "productive" CPU usage

#### `sy` — System Time

Time spent running kernel code.

- System calls, kernel threads, interrupt handling, scheduling
- High `sy` can indicate:
  - Heavy system call activity (lots of small I/O operations, frequent process creation)
  - Network-intensive workloads (packet processing happens in kernel)
  - Lock contention in kernel (spinlocks, mutexes)
- **Rule of thumb:** `sy` consistently above 20-30% may indicate an inefficient I/O pattern or kernel-level bottleneck
- If `sy` is higher than `us`, the system is spending less time on real work than on kernel overhead — not good

**Measuring CPU utilization (`us` + `sy` together):**

- If `us` + `sy` is consistently greater than 80%, the CPU is approaching its limits
- If `us` + `sy` = 100%, possible CPU bottleneck
- High `sy` means the application is issuing many system calls to the kernel — it measures how heavily the application is using kernel services

#### `id` — Idle Time

Time the CPU is idle with no outstanding work.

- `id` close to 0 means the CPU is fully utilized (check `r` column to see if there's CPU contention)
- High `id` with high `wa` means CPUs are idle because they're waiting on I/O, not because there's no work

#### `wa` — I/O Wait Time

Time the CPU is idle while the system has outstanding I/O requests.

- CPU had nothing to run, AND there was pending I/O
- High `wa` indicates the system is I/O bound — CPUs are waiting for disks
- **Important nuance:** `wa` is a subset of idle time. If there were other runnable processes, the CPU would run them instead of reporting `wa`. A system with high `wa` but also high `r` would show lower `wa` because CPUs stay busy running other processes while I/O completes.
- On single-CPU systems, `wa` is more meaningful. On multi-CPU systems, I/O wait may be "hidden" by other CPU activity.

**Understanding `wa` and `id` together (measuring true idle = `id` + `wa`):**

1. `id = 0%` does not mean all CPU is consumed — `wa` can be 100% and the CPU is just waiting for I/O to complete
2. `wa = 0%` does not mean there are no I/O issues — as long as there are threads keeping the CPU busy, additional threads waiting for I/O will be masked by the running threads
3. If process A is running and process B is waiting on I/O, `wa` will still show 0 — a zero value doesn't mean I/O is not occurring, it means the system is not *idle* waiting on I/O
4. If process A and process B are both waiting on I/O, and nothing else can use the CPU, then `wa` increases
5. High `wa` does not necessarily mean an I/O performance problem — it can indicate that I/O is occurring but the CPU is simply not kept busy at all
6. High `id` likely means there is no CPU or I/O problem

**Common causes of high `wa`:**
- Slow storage (mechanical disks, degraded RAID, SAN issues)
- Excessive swapping (`si`/`so` non-zero)
- Application doing synchronous I/O on slow devices
- NFS mounts with latency issues

#### `st` — Steal Time

Time the virtual CPU waited while the hypervisor was servicing another virtual processor.

- Only relevant for virtual machines (KVM, Xen, VMware, cloud instances)
- Indicates the host is overcommitted — your VM's CPU time is being "stolen" by other VMs
- Non-zero `st` means your VM is not getting all the CPU time it requests
- Sustained `st > 5-10%` typically indicates noisy neighbors or an undersized host

## Common Diagnostic Patterns

### CPU Saturation

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
12  0      0 245612  89712 1204536    0    0     0     0  892 1523 92  6  2  0  0
14  0      0 245580  89712 1204536    0    0     0     4  901 1498 94  5  1  0  0
```

**Indicators:**
- `r` significantly exceeds CPU count (e.g., 12-14 on a 4-core system)
- `us` + `sy` near 100%
- `id` near 0%
- `wa` = 0 (not I/O bound)
- `b` = 0 (no processes stuck on I/O)

**Action:** Identify CPU-hungry processes with `top` or `pidstat`. Consider scaling horizontally or optimizing application code.

### I/O Bottleneck

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  8      0 245612  89712 1204536    0    0  8520     0  452  312  5  3 12 80  0
 0 10      0 245580  89712 1204536    0    0 12040     0  489  298  3  2  8 87  0
```

**Indicators:**
- `b` is elevated (processes blocked on I/O)
- `wa` is high (CPUs idle waiting for I/O)
- `bi` and/or `bo` are high
- `r` is low (CPU is not the bottleneck)
- `id` is low but due to `wa`, not `us`/`sy`

**Action:** Investigate with `iostat -x` to identify which device is slow. Check for disk stalls, SAN issues, or NFS problems.

### Memory Pressure / Swap Thrashing

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 3  4 1048576   8420   1204   12536 4096 8192  4200  8300  652  812 15 12 23 50  0
 2  6 1056768   4120    980    8240 2048 12288 2100 12400  701  798 10 14 18 58  0
```

**Indicators:**
- `swpd` is large and growing
- `si` and `so` are both non-zero (active thrashing)
- `free` is very low
- `buff` and `cache` are shrinking (kernel is reclaiming page cache to feed applications)
- `b` is elevated (processes waiting for pages to be swapped in)
- `wa` is high (swap I/O contributing to I/O wait)

**Action:** Identify the memory consumer with `ps aux --sort=-%mem` or `smem`. Add RAM, reduce workload, or tune `vm.swappiness`. Check for memory leaks.

**OOM killer escalation:** When `free` is near zero, `buff`/`cache` are fully reclaimed, and `so` is maxed out with nowhere left to swap, the kernel's OOM killer activates next — it will forcibly terminate processes to free memory. Check `dmesg` or `/var/log/messages` for "Out of memory: Kill process" entries.

### Healthy Idle System

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 2456120  89712 4204536    0    0     0     4   52  112  0  0 100  0  0
 0  0      0 2456088  89712 4204536    0    0     0     0   48  108  0  0 100  0  0
```

**Indicators:**
- `r` = 0 or 1 (minimal CPU demand)
- `b` = 0 (no I/O blocking)
- `si`/`so` = 0 (no swap activity)
- `id` near 100%
- Large `cache` (kernel is caching filesystem data in idle memory — this is healthy)

### VM Steal Time Issue

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 4  0      0 512000  45000  800000    0    0     0    12  312  456 35 10  5  0 50
 3  0      0 511800  45000  800000    0    0     0     0  298  412 30  8  7  0 55
```

**Indicators:**
- `st` is high (50-55%)
- `r` > CPU count despite apparent CPU availability
- `us` + `sy` don't account for all non-idle time
- Processes feel slow despite "available" CPU

**Action:** Contact cloud provider or VM administrator. Resize to a larger instance, move to a dedicated host, or schedule workload during off-peak hours.

## vmstat vs. Other Tools

| Tool | Best For |
|------|----------|
| `vmstat` | Quick overall system health check — CPU, memory, swap, I/O at a glance |
| `iostat -x` | Detailed per-device I/O analysis (latency, throughput, queue depth) |
| `mpstat -P ALL` | Per-CPU breakdown (detecting uneven CPU load) |
| `free -h` | Current memory usage summary with available memory |
| `sar -q` | Historical run queue and load average data |
| `top` / `htop` | Per-process CPU and memory usage |
| `pidstat` | Per-process CPU, memory, and I/O stats |

## Useful One-Liners

```bash
# Quick system health check (5 samples, 1-second interval)
vmstat 1 5

# Monitor memory pressure (watch si/so columns)
vmstat -w 1 | awk 'NR<=2 || $7>0 || $8>0'

# Log vmstat output with timestamps to a file
vmstat -t 5 >> /var/log/vmstat.log &

# Check if the system is I/O bound (wa > 20%)
vmstat -w 1 5 | awk 'NR>2 && $16>20 {print "I/O WAIT HIGH: "$16"%"}'

# Quick check for swap activity
vmstat 1 5 | awk 'NR>2 {if ($7>0 || $8>0) print "SWAP ACTIVE: si="$7" so="$8; else print "No swap activity"}'

# Display vmstat in MB for easier reading on large-memory systems
vmstat -S M -w 1

# Capture vmstat output for one minute with hostname and timestamp in filename
vmstat 1 60 > vmstat_$(hostname)_$(date +%F_%H-%M-%S)

# Capture vmstat output with tee (view and save simultaneously)
vmstat 1 60 | tee /tmp/vmstat.out

# Continuous monitoring with only relevant fields (procs + cpu)
vmstat 5 | awk '{print $1, $2, $13, $14, $15, $16, $17}'
```

## Best Practices

1. **Always ignore the first line** — it represents averages since boot, not current state.
2. **Use intervals of 1-5 seconds** for troubleshooting, longer intervals (30-60s) for monitoring.
3. **Correlate `r` with CPU count** — `r` exceeding logical CPUs indicates CPU saturation.
4. **Watch `si`/`so`, not just `swpd`** — swap usage alone is not a problem; active swap I/O is.
5. **Interpret `wa` carefully on multi-CPU systems** — I/O wait can be masked by other CPU activity.
6. **Use `-w` (wide output)** on systems with large memory to prevent column truncation.
7. **Combine with specialized tools** — use `vmstat` for the big picture, then drill down with `iostat`, `mpstat`, `pidstat`, or `sar` for specifics.
8. **Baseline your system** — know what normal looks like so you can recognize abnormal.
9. **Don't panic over high `cache`** — large page cache is healthy and will be reclaimed when needed.
10. **Check `st` on cloud/virtual instances** — steal time reveals host-level resource contention invisible to in-guest tools.

## Quick Reference

| What You Want to Know | Where to Look |
|---|---|
| Is the system CPU-bound? | `r` > CPU count, `us` + `sy` near 100%, `id` near 0 |
| Is the system I/O-bound? | `b` > 0, `wa` > 20%, `bi`/`bo` elevated |
| Is the system swapping? | `si` > 0 or `so` > 0 |
| Is the VM being starved? | `st` > 5-10% |
| Is there enough memory? | `free` + `buff` + `cache` is adequate; `si`/`so` = 0 |
| Are processes stuck? | `b` sustained > 0, cross-reference with `ps` for D-state processes |
| Context switch storms? | `cs` very high relative to baseline; correlate with lock contention |
| Instantaneous load estimate | `r` + `b` (roughly equals current load) |

## Quick Notes

1. Thrashing → `si` ≈ `so` (pages constantly swapped in and out)
2. For running processes (`r`) → compare with logical CPUs; `r` > `b` expected, if `b` > `r` then I/O bottleneck
3. For blocked processes (`b`) → investigate with `ps -eLo state,pid,cmd | grep ^D`
4. High `r` + high `cs` → possible lock contention
5. `cs` > `sy` → more context switching than actual work
6. `us` + `sy` > 80% sustained → CPU approaching limits

## Data Source: /proc/vmstat

The `vmstat` command summarizes data from several `/proc` files, but the raw `/proc/vmstat` file provides much more granular kernel virtual memory counters. Key fields for diagnostics:

```bash
# View all counters
cat /proc/vmstat

# Watch specific paging counters
grep -E "pgpgin|pgpgout|pswpin|pswpout" /proc/vmstat
```

| Counter | Meaning |
|---------|---------|
| `pgpgin` | Total pages paged in from disk (cumulative) |
| `pgpgout` | Total pages paged out to disk (cumulative) |
| `pswpin` | Total pages swapped in from swap space (cumulative) |
| `pswpout` | Total pages swapped out to swap space (cumulative) |
| `pgfault` | Total page faults (minor + major) |
| `pgmajfault` | Total major page faults (required disk I/O) |
| `oom_kill` | Total OOM killer invocations (Linux 4.13+) |
| `nr_dirty` | Current number of dirty pages waiting for writeback |
| `nr_writeback` | Current number of pages actively being written back |

**Note:** `pswpin`/`pswpout` count only swap-space paging. `pgpgin`/`pgpgout` include all block device paging (file-backed + swap). The difference helps distinguish file I/O from memory pressure.
