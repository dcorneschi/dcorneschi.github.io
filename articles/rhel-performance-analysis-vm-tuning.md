# RHEL Performance Analysis and VM Tuning

A comprehensive guide to Linux memory management, virtual memory tuning, hugepages, I/O schedulers, and performance monitoring on Red Hat Enterprise Linux 7 through 10. Covers the kernel internals that drive performance decisions and the tunables that control them.

---

## Performance Tuning Methodology

Before tuning anything, follow a systematic approach:

1. **Document configuration** — hardware, kernel version, sysctl settings, mount options
2. **Baseline results** — measure current performance with relevant workloads
3. **Monitor and instrument** — identify the bottleneck (CPU, memory, I/O, network)
4. **Apply one change at a time** — isolate the effect of each tunable
5. **Analyze results** — compare against baseline, exit or loop
6. **Document final configuration** — record what worked and why

The two fundamental tuning goals are often in tension:
- **Throughput** — maximize total work done per unit time (batch jobs, data warehouses)
- **Latency** — minimize response time per request (OLTP, web servers, trading systems)

---

## Memory Architecture

### Physical Memory Layout

Linux organizes physical RAM into NUMA nodes and memory zones. Each zone has its own free list, active/inactive page lists, and reclaim watermarks.

**64-bit memory zones (x86_64):**

```
┌─────────────────────────────────────────────────┐
│                  Normal Zone                    │  (4GB → End of RAM)
│  Kernel static/dynamic allocations              │
│  User: anonymous, pagecache, pagetables         │
├─────────────────────────────────────────────────┤
│                  DMA32 Zone                     │  (16MB → 4GB)
│  32-bit DMA devices                             │
│  Normal zone overflow                           │
├─────────────────────────────────────────────────┤
│                  DMA Zone                       │  (0 → 16MB)
│  Legacy 24-bit DMA devices                      │
└─────────────────────────────────────────────────┘
```

On 64-bit systems, both kernel and user allocations come from the same Normal zone — no more highmem/lowmem split issues from 32-bit era.

### Per-Zone Page Lists

Each zone maintains three page lists:

| List | Contents | Behavior |
|------|----------|----------|
| **Active** | Most recently referenced pages | Anonymous (heap, stack) + pagecache (file data) |
| **Inactive** | Least recently referenced pages | Dirty → writeback → clean → ready to free |
| **Free** | Available pages | Coalesced buddy allocator for contiguous allocation |

Pages flow: **Allocation → Active → Inactive → Free** (with reactivation if accessed again while inactive).

```
                    ┌──────────────────────────────────────────────┐
                    │           User Allocations                   │
                    └──────────────┬───────────────────────────────┘
                                   │
                                   ▼
                   ┌───────────────────────────────┐
           ┌──────▶│           ACTIVE              │
           │       │  (most recently referenced)   │
           │       └──────────────┬────────────────┘
           │                      │ Page aging
           │                      ▼
           │       ┌──────────────────────────────┐
           │       │          INACTIVE            │
  Reactivate       │  (Dirty → Writeback → Clean) │──── swapout/pdflush ───┐
  (re-accessed)    └──────────────┬───────────────┘                        │
           │                      │ Reclaiming (clean pages)               │
           │                      ▼                                        ▼
           │       ┌──────────────────────────────┐                   ┌────────┐
           └───────│            FREE              │◀──────────────────│  Disk  │
                   │   (buddy allocator)          │   User deletions  └────────┘
                   └──────────────────────────────┘
```

### Per-NUMA Node Resources

Each NUMA node contains:
- Memory zones (DMA + Normal on node 0; Normal only on other nodes)
- CPUs assigned to that node
- I/O and DMA capacity (PCI devices attached to the node)
- Interrupt processing for local devices
- A dedicated page reclamation thread (`kswapd<N>`)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        4-Node NUMA System                                   │
│                                                                             │
│  ┌─────────────────────┐              ┌─────────────────────┐               │
│  │      Node 0         │              │      Node 1         │               │
│  │  C0  C1  C2  C3     │◄────QPI────► │  C4  C5  C6  C7     │               │
│  │  Memory (32GB)      │              │  Memory (32GB)      │               │
│  │  PCI: eth0, sda     │              │  PCI: eth1, sdb     │               │
│  └─────────┬───────────┘              └─────────┬───────────┘               │
│            │                                    │                           │
│            │            QPI/UPI                 │                           │
│            │                                    │                           │
│  ┌─────────┴───────────┐              ┌─────────┴───────────┐               │
│  │      Node 2         │              │      Node 3         │               │
│  │  C8  C9  C10 C11    │◄────QPI────► │  C12 C13 C14 C15    │               │
│  │  Memory (32GB)      │              │  Memory (32GB)      │               │
│  │  PCI: eth2, sdc     │              │  PCI: eth3, sdd     │               │
│  └─────────────────────┘              └─────────────────────┘               │
│                                                                             │
│  Memory Placement:                                                          │
│                                                                             │
│  Non-Interleaved (NUMA-aware):   │  Interleaved (NUMA-unaware):             │
│  Process on N1 → memory on N1    │  Process on N1 → memory scattered        │
│  ████████░░░░░░░░                │  ██░░██░░██░░██░░                        │
│  N0   N1   N2   N3               │  N0 N1 N2 N3 N0 N1                       │
│  (fast local access)             │  (equal but slower average access)       │
└─────────────────────────────────────────────────────────────────────────────┘
```

```bash
# View per-node resources
cat /sys/devices/system/node/node0/meminfo
cat /sys/devices/system/node/node*/cpulist
```

### PageCache vs Anonymous Memory

Linux memory is fundamentally split between:

**PageCache (file-backed):**
- Grows when filesystem data is read/written
- Global — shared between all processes accessing the same files
- Freed by: file deletion, filesystem unmount, kswapd reclaim, `drop_caches`

**Anonymous memory (process-private):**
- Grows on user demand (malloc + page fault, stack growth)
- Private to each process
- Freed by: process exit, munmap, kswapd swapout

The balance between pagecache and anonymous memory is dynamic and controlled by `vm.swappiness`.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Physical Memory (RAM)                        │
│                                                                 │
│  ◀── Pagecache Allocations              Page Faults ──▶         │
│                                                                 │
│  ┌────────────────────────────┬─────────────────────────────┐   │
│  │                            │                             │   │
│  │        pagecache           │        anonymous            │   │
│  │   (filesystem data)        │   (heap, stack, mmap)       │   │
│  │                            │                             │   │
│  └────────────────────────────┴─────────────────────────────┘   │
│                                                                 │
│  ◀── Reclaim: kswapd/flushd ──▶   ◀── Reclaim: kswapd/swap ──▶  │
│      page reclaim                     page reclaim (swapout)    │
│      file deletion                    process unmap/exit        │
│      unmount filesystem                                         │
│                                                                 │
│  vm.swappiness controls the balance:                            │
│    Low (10)  → prefer reclaiming pagecache, avoid swapping      │
│    High (60) → balanced reclaim of both                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Memory Reclaim Dynamics

### Watermarks and kswapd

The kernel uses three watermarks per zone to control when memory reclamation occurs:

```
┌───────────────────────────────┐
│         All of RAM            │
│                               │
│         Do nothing            │  (free > pages_high)
│                               │
├─ pages_high ──────────────────┤  kswapd sleeps above this
│                               │
│     kswapd reclaims memory    │  (pages_low < free < pages_high)
│                               │
├─ pages_low ───────────────────┤  kswapd wakes up here
│                               │
│     kswapd reclaims memory    │  (pages_min < free < pages_low)
│                               │
├─ pages_min ───────────────────┤  ALL allocators reclaim (direct reclaim)
│                               │
│  user processes + kswapd      │  (free < pages_min)
│  both reclaim memory          │
└───────────────────────────────┘
```

**Direct reclaim** is bad for performance — it means the allocating process must stall to free memory before it can proceed.

### Controlling Watermarks: min_free_kbytes

`vm.min_free_kbytes` directly sets the pages_min watermark. The kernel derives pages_low and pages_high from it:

```bash
# View current setting
cat /proc/sys/vm/min_free_kbytes

# Increase to reduce direct reclaim on large-memory systems
# Default scales with RAM but may be too low for I/O-heavy workloads
sysctl -w vm.min_free_kbytes=524288    # 512MB reserve

# Persistent
echo "vm.min_free_kbytes = 524288" >> /etc/sysctl.d/99-perf.conf
```

**When to increase:**
- Systems with high I/O rates that trigger direct reclaim
- Large-memory systems (256GB+) where the default reserve is proportionally tiny
- Database servers where allocation stalls cause latency spikes

**When NOT to increase:**
- Small memory systems — wastes RAM that applications could use
- Don't exceed 5% of total RAM

### What kswapd Reclaims

**User memory reclamation:**
- Page aging (move active → inactive)
- Pagecache shrinking (write dirty, free clean)
- Swapping (write anonymous pages to swap)

**Kernel memory reclamation:**
- Slab cache reaping (inode cache, dentry cache, buffer heads)

---

## VM Tunables

### vm.swappiness

Controls the balance between reclaiming pagecache vs swapping anonymous memory:

```bash
# View current value (default: 60 on RHEL 7, reduced to 30 on some RHEL 8+ profiles)
cat /proc/sys/vm/swappiness

# Set temporarily
sysctl -w vm.swappiness=10

# Persistent
echo "vm.swappiness = 10" >> /etc/sysctl.d/99-perf.conf
```

| Value | Behavior | Use Case |
|-------|----------|----------|
| 0 | Avoid swapping unless absolutely necessary | Database servers with large SGA/buffer pools |
| 10 | Strongly prefer reclaiming pagecache | Most server workloads (tuned default for RHEL 7+) |
| 30 | Moderately prefer pagecache reclaim | Virtual guests (RHEL 8+ virtual-guest profile) |
| 60 | Default balance | General purpose |
| 100 | Aggressively swap anonymous memory | Rarely useful |

> **RHEL 7+ note:** The `throughput-performance` tuned profile sets swappiness=10. For most server workloads, this is appropriate.

### vm.dirty_ratio and vm.dirty_background_ratio

These control when dirty pagecache (modified file data not yet written to disk) gets flushed:

```bash
# View current values
sysctl vm.dirty_ratio vm.dirty_background_ratio

# Typical server tuning
sysctl -w vm.dirty_background_ratio=5
sysctl -w vm.dirty_ratio=20
```

```
100% of pagecache RAM dirty
    ↓
    flushd + writing processes both write dirty buffers
    ↓
dirty_ratio (default 20%) — processes start synchronous writes ← STALLS HERE
    ↓
    flushd writes dirty buffers in background
    ↓
dirty_background_ratio (default 10%) — wakeup flushd
    ↓
    do_nothing
0% of pagecache RAM dirty
```

**Tuning for high pagecache pressure** (lots of writes):
- **Lower** `dirty_background_ratio` (e.g., 3-5%) → start flushing sooner, smaller bursts
- **Increase** `dirty_ratio` (e.g., 40-60%) → delay synchronous stalls longer

**Tuning for databases with Direct I/O:**
- Databases using O_DIRECT bypass pagecache entirely
- These tunables mainly affect non-database filesystem I/O (logs, temp files)

**RHEL 7+ throughput-performance profile defaults:**
- `vm.dirty_background_ratio = 10`
- `vm.dirty_ratio = 40` (RHEL 7 default in profile)

**Byte-level control (RHEL 6+):**

For very large memory systems where percentages are too coarse:

```bash
# Set absolute bytes instead of percentages
sysctl -w vm.dirty_background_bytes=104857600    # 100MB
sysctl -w vm.dirty_bytes=1073741824              # 1GB

# Note: setting _bytes overrides the corresponding _ratio
```

### vm.vfs_cache_pressure

Controls how aggressively the kernel reclaims inode/dentry cache memory:

```bash
# Default is 100 (balanced)
cat /proc/sys/vm/vfs_cache_pressure

# Lower = keep more inode/dentry cache (good for many-file workloads)
sysctl -w vm.vfs_cache_pressure=50

# Higher = reclaim inode/dentry cache more aggressively
sysctl -w vm.vfs_cache_pressure=200
```

### Dropping Caches (Manual Flush)

For benchmarking or diagnostics — not for production tuning:

```bash
# Free pagecache
echo 1 > /proc/sys/vm/drop_caches

# Free slab cache (dentries, inodes)
echo 2 > /proc/sys/vm/drop_caches

# Free both pagecache and slab cache
echo 3 > /proc/sys/vm/drop_caches
```

> **Warning:** If a database relies on the kernel pagecache (not using Direct I/O), dropping caches will cause a significant performance drop until the cache warms up again.

---

## Hugepages

### Why Hugepages Matter

The Translation Lookaside Buffer (TLB) is a small CPU cache of recent virtual-to-physical address mappings. TLB misses are expensive on modern pipelined CPUs.

| Page Size | TLB Coverage (128 entries) | Use Case |
|-----------|---------------------------|-----------|
| 4KB (standard) | 512KB | General purpose |
| 2MB (huge) | 256MB | Databases, JVMs, KVM guests |
| 1GB (gigantic) | 128GB | Very large databases, NFV |

The 512:1 ratio between 2MB and 4KB pages means hugepages dramatically reduce TLB misses for large-memory applications.

### Static Hugepages (2MB)

```bash
# Reserve 4000 hugepages (= 8GB of 2MB pages)
echo 4000 > /proc/sys/vm/nr_hugepages

# Persistent via sysctl
echo "vm.nr_hugepages = 4000" >> /etc/sysctl.d/99-hugepages.conf

# Verify
cat /proc/meminfo | grep -i huge
# HugePages_Total:     4000
# HugePages_Free:      4000
# HugePages_Rsvd:         0
# Hugepagesize:         2048 kB

# Mount hugetlbfs for applications to use
mount -t hugetlbfs hugetlbfs /dev/hugepages

# Per-NUMA node hugepage allocation
cat /sys/devices/system/node/node*/meminfo | grep Huge
```

### 1GB Hugepages

Available since RHEL 6, must be reserved at boot time:

```bash
# Kernel command line (grub)
default_hugepagesz=1G hugepagesz=1G hugepages=8

# Verify after boot
cat /proc/meminfo | grep -i huge
# HugePages_Total:        8
# HugePages_Free:         8
# Hugepagesize:     1048576 kB
```

### Transparent Hugepages (THP)

RHEL 6+ automatically promotes 4KB pages to 2MB hugepages without application changes:

```bash
# Check THP status
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never

# Disable THP (for latency-sensitive workloads or databases that prefer static hugepages)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Persistent via kernel cmdline
# transparent_hugepage=never

# Check how much THP memory is in use
cat /proc/meminfo | grep AnonHugePages
```

**When to use THP (default `always`):**
- General server workloads, JVMs, scientific computing
- Applications that allocate large contiguous memory regions

**When to disable THP:**
- Latency-sensitive workloads (THP compaction causes jitter)
- Applications already using static hugepages (Oracle SGA, SAP HANA)
- Real-time / NFV workloads (tuned `network-latency` profile disables THP)

### NUMA Hugepage Placement

Hugepages are allocated per-node. Verify distribution:

```bash
# Check per-node hugepage allocation
cat /sys/devices/system/node/node*/meminfo | grep Huge

# If imbalanced, reset and reallocate
echo 0 > /proc/sys/vm/nr_hugepages
echo 4000 > /proc/sys/vm/nr_hugepages
# Kernel distributes evenly across nodes when memory is free
```

### KVM and Hugepages

Using hugepages for KVM guest memory provides 11-17% performance improvement for database workloads:

```xml
<!-- libvirt domain XML -->
<memoryBacking>
  <hugepages/>
</memoryBacking>
```

---

## I/O Schedulers

### Buffered I/O vs Direct I/O

Understanding the I/O path is essential before choosing schedulers:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   BUFFERED I/O (default)              DIRECT I/O (O_DIRECT flag)        │
│                                                                         │
│   ┌───────────────┐                   ┌───────────────┐                 │
│   │  User Buffer  │                   │  User Buffer  │                 │
│   └───────┬───────┘                   └───────┬───────┘                 │
│           │ Read()/Write()                    │ Read()/Write()          │
│           │ memory copy                       │                         │
│           ▼                                   │                         │
│   ┌───────────────┐                           │ DMA (no copy)           │
│   │  Page Cache   │                           │                         │
│   │  (kernel)     │                           │                         │
│   └───────┬───────┘                           │                         │
│           │ DMA                               │                         │
│           ▼                                   ▼                         │
│   ┌───────────────┐                   ┌───────────────┐                 │
│   │     Disk      │                   │     Disk      │                 │
│   └───────────────┘                   └───────────────┘                 │
│                                                                         │
│   Pros:                                Pros:                            │
│   - Reads served from cache            - No double caching              │
│   - Write coalescing                   - Predictable latency            │
│   - Readahead optimization             - App controls its own cache     │
│                                                                         │
│   Cons:                                Cons:                            │
│   - Double caching with DB             - No kernel readahead            │
│   - Memory pressure from cache         - App must handle buffering      │
│   - Unpredictable writeback timing     - Requires aligned I/O           │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key insight:** Databases (Oracle, PostgreSQL, MySQL InnoDB) have their own buffer pools. Using Direct I/O avoids caching data twice — once in the database buffer pool and once in the kernel pagecache.

### Legacy Schedulers (RHEL 7 and earlier, single-queue)

| Scheduler | Description | Best For |
|-----------|-------------|----------|
| **CFQ** | Completely Fair Queuing (default) | Multiple processes/LUNs, balanced |
| **Deadline** | Low per-I/O latency, prevents starvation | Databases (OLTP/DSS), latency-sensitive |
| **NOOP** | No reordering, minimal CPU overhead | RAM disks, hardware RAID controllers, SSDs |
| **Anticipatory** | Inserts delays to aggregate I/O | Single SATA drives (removed in RHEL 7) |

### Modern Schedulers (RHEL 8+, multi-queue blk-mq)

| Scheduler | Description | Best For |
|-----------|-------------|----------|
| **mq-deadline** | Multi-queue deadline (default for non-rotational) | SSDs, NVMe, databases |
| **bfq** | Budget Fair Queueing | Desktop, interactive, mixed workloads |
| **kyber** | Token-based for fast devices | High-speed NVMe |
| **none** | No scheduling (passthrough) | NVMe with hardware queues, virtual disks |

### Configuring I/O Schedulers

```bash
# View current scheduler for a device
cat /sys/block/sda/queue/scheduler
# [mq-deadline] kyber bfq none

# Change dynamically
echo "mq-deadline" > /sys/block/sda/queue/scheduler

# Via udev rule (persistent, RHEL 7+)
# /etc/udev/rules.d/60-scheduler.rules
ACTION=="add|change", KERNEL=="sd*", ATTR{queue/scheduler}="mq-deadline"
ACTION=="add|change", KERNEL=="nvme*", ATTR{queue/scheduler}="none"

# Via tuned profile (recommended)
# tuned sets scheduler based on device type automatically
tuned-adm profile throughput-performance

# Boot-time (RHEL 7 single-queue legacy)
# Grub command line: elevator=deadline
```

### I/O Scheduler Impact

Testing with Oracle workloads showed significant differences:

**OLTP workload:** Deadline was 34% better than CFQ at high process counts. Deadline and Noop performed similarly.

**DSS (analytics) workload:** Deadline was 47-58% faster than CFQ depending on parallelism.

**Recommendation by workload:**

| Workload | RHEL 7 | RHEL 8/9/10 |
|----------|--------|-------------|
| Database (spinning disk) | deadline | mq-deadline |
| Database (SSD/NVMe) | noop or deadline | none or mq-deadline |
| General server | cfq | mq-deadline (default) |
| Virtual guest | noop | none |
| Desktop/interactive | cfq | bfq |

---

## CPU and Process Tuning

### CPU Scheduler Topology

The Linux scheduler recognizes multi-level CPU topology and uses it for optimal task placement:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CPU Scheduler                               │
│                                                                     │
│  ┌───────────────────────────────┐   ┌───────────────────────────┐  │
│  │         Socket 0              │   │        Socket 1           │  │
│  │  ┌─────────┐  ┌─────────┐     │   │  ┌─────────┐              │  │
│  │  │ Core 0  │  │ Core 1  │     │   │  │ Core 0  │    ...       │  │
│  │  │ T0 | T1 │  │ T0 | T1 │     │   │  │ T0 | T1 │              │  │
│  │  └─────────┘  └─────────┘     │   │  └─────────┘              │  │
│  └───────────────────────────────┘   └───────────────────────────┘  │
│                                                                     │
│  Scheduler Run Queues (per-level):                                  │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Socket 0 │  │ Socket 0 │  │ Socket 1 │  │ Socket 1 │             │
│  │  Core 0  │  │  Core 1  │  │  Core 0  │  │  Core 1  │             │
│  ├──────────┤  ├──────────┤  ├──────────┤  ├──────────┤             │
│  │ Process  │  │ Process  │  │ Process  │  │ Process  │             │
│  │ Process  │  │ Process  │  │ Process  │  │          │             │
│  │ Process  │  │          │  │ Process  │  │          │             │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘             │
│                                                                     │
│  Balancing priority:                                                │
│  1. Threads within same core (shared L1/L2 cache)                   │
│  2. Cores within same socket (shared L3 cache)                      │
│  3. Across sockets (cross-NUMA, most expensive)                     │
└─────────────────────────────────────────────────────────────────────┘
```

The scheduler:
- Keeps tasks on the same core for cache locality (strong CPU affinity)
- Balances load across cores within a socket before crossing sockets
- Respects NUMA boundaries — avoids migrating tasks that have memory on a different node
- Requires BIOS to report CPU topology correctly via ACPI/SRAT tables

### CPU Frequency Governors

```bash
# Check current governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Set performance governor (maximum frequency, no power saving)
echo performance > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# RHEL 7+ uses tuned to manage this
tuned-adm profile throughput-performance
# Sets: governor=performance, energy_perf_bias=performance
```

| Governor | Behavior | Use Case |
|----------|----------|----------|
| `performance` | Always maximum frequency | Servers, databases, benchmarks |
| `powersave` | Always minimum frequency | Rarely used on servers |
| `ondemand` | Scale based on CPU usage | Older default (RHEL 5/6) |
| `schedutil` | Scheduler-driven scaling | RHEL 8+ default |

> **Impact:** I/O-heavy workloads can keep CPUs stepped down with `ondemand`/`schedutil`, causing 15-30% performance loss. The `throughput-performance` profile forces `performance` governor.

### CPU Topology and Scheduler

The Linux scheduler recognizes multi-level topology:
- **Sockets** — separate physical packages
- **Cores** — physical cores within a socket
- **Threads** — hyperthreaded siblings sharing a core

```bash
# View topology
lscpu

# Key fields from /proc/cpuinfo:
# processor    — logical CPU number
# physical id  — socket number
# siblings     — logical CPUs per socket
# core id      — core number within socket
# cpu cores    — physical cores per socket
```

The scheduler:
- Keeps tasks on the same core (cache locality)
- Balances load across cores within a socket before crossing sockets
- Avoids bouncing tasks between NUMA nodes

### Process Affinity

```bash
# Pin process to specific CPUs
taskset -c 0-7 ./my_application

# Pin to NUMA node (CPU + memory)
numactl --cpunodebind=0 --membind=0 ./my_application

# View process affinity
taskset -p <pid>

# Real-time priority
chrt -f 50 ./my_rt_application
```

### cgroups Resource Control

```bash
# RHEL 7 (cgroups v1) / RHEL 8+ (cgroups v2 default)
# Limit a process to 2 CPUs and 1GB memory:

# systemd slice (recommended)
systemctl set-property myapp.service CPUQuota=200%
systemctl set-property myapp.service MemoryMax=1G

# Manual cgroups v2
echo "+cpu +memory" > /sys/fs/cgroup/cgroup.subtree_control
mkdir /sys/fs/cgroup/myapp
echo "200000 100000" > /sys/fs/cgroup/myapp/cpu.max
echo "1073741824" > /sys/fs/cgroup/myapp/memory.max
echo $PID > /sys/fs/cgroup/myapp/cgroup.procs
```

---

## Performance Monitoring Tools

### Tools by Subsystem

| CPU | Memory | Disk | Network | Process |
|-----|--------|------|---------|---------|
| top | top | iostat -x | ethtool -S | top |
| mpstat -P ALL | vmstat -s | iotop | ss -s | ps aux |
| sar -u | free -m | blktrace | sar -n DEV | strace |
| perf top | sar -r -B -W | sar -d | nstat | pidstat |
| turbostat | numastat | btt | tcpdump | perf record |

### Key Monitoring Commands

```bash
# CPU: per-CPU utilization including steal, iowait
mpstat -P ALL 1

# Memory: watch free/cache/swap/io/cpu in one view
vmstat 1

# Disk: extended I/O stats with latency
iostat -xz 1

# Network: per-interface stats
sar -n DEV 1

# Process: who is consuming what
pidstat -d -r -u 1

# All-in-one subsystem view (PCP)
pmcollectl -s cdnm
```

### /proc Key Files

```bash
# System-wide memory
cat /proc/meminfo

# Per-process memory map
cat /proc/<pid>/maps
cat /proc/<pid>/numa_maps

# VM statistics (page faults, reclaim, NUMA)
cat /proc/vmstat

# Slab allocator (kernel caches)
cat /proc/slabinfo
# or: slabtop

# Per-zone memory info
cat /proc/zoneinfo

# CPU info
cat /proc/cpuinfo
```

### SysRq Memory Diagnostics

When investigating memory issues on a live system:

```bash
# Trigger memory info dump to dmesg/console
echo m > /proc/sysrq-trigger

# Output shows per-node, per-zone:
# - Free pages, watermarks (min/low/high)
# - Active/inactive pages
# - Buddy allocator free list fragmentation
# - all_unreclaimable flag
```

### perf (Replaces OProfile since RHEL 6+)

```bash
# System-wide top functions by CPU cycles
perf top

# Record a workload
perf record -a -g sleep 30
perf report

# Count hardware events
perf stat -a sleep 10

# List available events
perf list

# Record specific events
perf record -e cache-misses,branch-misses -a sleep 10
```

### SystemTap (Deep Kernel Tracing)

```bash
# Install prerequisites
dnf install systemtap kernel-devel-$(uname -r) kernel-debuginfo-$(uname -r)

# Simple script: trace page allocations
stap -e 'probe vm.pagefault { printf("%s(%d) fault at %p\n", execname(), pid(), address) }' -T 10
```

---

## Overcommit and OOM

### Memory Overcommit Modes

```bash
cat /proc/sys/vm/overcommit_memory
```

| Value | Behavior | Use Case |
|-------|----------|----------|
| 0 | Heuristic overcommit (default) | General purpose |
| 1 | Always allow overcommit | Sparse memory applications, scientific computing |
| 2 | Strict — never overcommit beyond swap + ratio*RAM | Databases that must never be OOM-killed |

```bash
# Strict no-overcommit (for databases)
sysctl -w vm.overcommit_memory=2
sysctl -w vm.overcommit_ratio=80    # Allow commit up to swap + 80% RAM
```

### OOM Killer Control

```bash
# View OOM score for a process (higher = more likely to be killed)
cat /proc/<pid>/oom_score

# Adjust OOM priority (-1000 to +1000, RHEL 7+)
echo -1000 > /proc/<pid>/oom_score_adj    # Never OOM kill this process
echo 1000 > /proc/<pid>/oom_score_adj     # Kill this first

# Via systemd
systemctl edit mydb.service
# [Service]
# OOMScoreAdjust=-1000
```

### Diagnosing OOM Kills

The OOM killer can trigger even when the system appears to have free memory — the key is per-zone exhaustion:

```
┌─────────────────────────────────────────────────────────────────────┐
│           OOM Kill Scenario: Zone Exhaustion                        │
│                                                                     │
│  The system has 9GB free overall, but Normal zone is exhausted:     │
│                                                                     │
│  ┌─────────────┐  ┌─────────────────┐  ┌───────────────────────┐    │
│  │  DMA Zone   │  │  Normal Zone    │  │   (DMA32/HighMem)     │    │
│  │  16MB total │  │  901MB total    │  │   ~8GB+ total         │    │
│  │             │  │                 │  │                       │    │
│  │  free: 12MB │  │  free: 656KB ◀──│──│── EXHAUSTED!          │    │
│  │             │  │  min:  928KB    │  │   free: 8.9GB         │    │
│  │             │  │  all_unreclaim: │  │   (not usable for     │    │
│  │             │  │    YES          │  │    kernel allocs)     │    │
│  └─────────────┘  └─────────────────┘  └───────────────────────┘    │
│                                                                     │
│  Result: Kernel allocation from Normal zone fails → OOM kill        │
│                                                                     │
│  Common causes:                                                     │
│  - slab cache growth (inode, dentry, buffer_head)                   │
│  - Kernel driver allocations (GFP_KERNEL)                           │
│  - I/O subsystem stalling dirty writeback (all pages under I/O)     │
│                                                                     │
│  Fix: Move to 64-bit (no highmem), or reduce kernel memory pressure │
└─────────────────────────────────────────────────────────────────────┘
```

> **Note:** On 64-bit RHEL 7+, the highmem issue is gone — all memory is in the Normal zone. OOM kills on 64-bit are almost always genuine out-of-memory conditions or cgroup memory limit hits.

```bash
# Check dmesg for OOM messages
dmesg | grep -i "out of memory\|oom"
journalctl -k | grep -i oom

# The OOM dump shows:
# - Per-zone free memory, watermarks, and all_unreclaimable status
# - Buddy allocator fragmentation
# - Which process was killed and why
```

---

## Database and JVM Tuning

### Synchronous vs Asynchronous I/O

The I/O model significantly impacts database throughput:

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   SYNCHRONOUS I/O                      ASYNCHRONOUS I/O             │
│                                                                     │
│   ┌───────────┐                        ┌───────────┐                │
│   │App I/O    │                        │App I/O    │                │
│   │Request    │                        │Request    │                │
│   └─────┬─────┘                        └─────┬─────┘                │
│         │                                    │                      │
│         ▼                                    ▼                      │
│   ┌───────────┐                        ┌───────────┐                │
│   │  Device   │                        │  Device   │                │
│   │  Driver   │                        │  Driver   │                │
│   │           │                        │           │                │
│   │ I/O Issue │                        │ I/O Issue │                │ 
│   └─────┬─────┘                        └─────┬─────┘                │
│         │                                    │                      │
│    ▓▓▓▓▓▓▓▓▓▓▓  ◀── Stall!                   │                      │
│    ▓ Waiting ▓  (app blocked)                ▼                      │
│    ▓▓▓▓▓▓▓▓▓▓▓                         ┌───────────┐                │
│         │                              │Application│                │
│         ▼                              │ continues │  ◀── No stall! │
│   ┌───────────┐                        │ working   │                │
│   │Completion │                        └─────┬─────┘                │
│   └─────┬─────┘                              │                      │
│         │                                    ▼                      │
│         ▼                              ┌───────────┐                │
│   ┌───────────┐                        │Completion │                │
│   │Application│                        │ callback  │                │
│   │ resumes   │                        └───────────┘                │
│   └───────────┘                                                     │
│                                                                     │
│   Impact: 100% CPU stall per I/O       Impact: overlap compute+I/O  │
└─────────────────────────────────────────────────────────────────────┘
```

Oracle with AIO+DIO achieves 2.5x throughput vs synchronous buffered I/O.

### Oracle Database

```bash
# Key settings for Oracle on RHEL 7+:
# 1. Hugepages for SGA
vm.nr_hugepages = <SGA_in_MB / 2 + 10% buffer>

# 2. Eliminate swapping
vm.swappiness = 10

# 3. Use Direct I/O (set in Oracle: filesystemio_options=setall)
# Bypasses pagecache, avoids double-caching

# 4. Use Async I/O (set in Oracle: disk_asynch_io=true)
# Allows parallel I/O without process stalls

# 5. I/O scheduler
echo "mq-deadline" > /sys/block/sda/queue/scheduler

# 6. SHM limits
kernel.shmmax = <at_least_SGA_size_in_bytes>
kernel.shmall = <shmmax / page_size>
```

**AIO + DIO performance impact:** Testing showed Oracle with AIO+DIO achieved 100% throughput, DIO only achieved 68%, AIO only 40%, and no AIO/DIO only 50%.

### JVM Applications (JBoss, Tomcat, Kafka)

```bash
# 1. Eliminate swapping
vm.swappiness = 10

# 2. Promote pagecache reclaiming
vm.dirty_background_ratio = 5
vm.dirty_ratio = 20

# 3. Lower vfs_cache_pressure for many-file workloads
vm.vfs_cache_pressure = 50

# 4. Use Hugepages for JVM heap
# JVM flag: -XX:+UseLargePages
# Requires hugepages pre-allocated and user in hugetlb group

# 5. NUMA-aware JVM startup
numactl --interleave=all java -jar myapp.jar
# or for single-node:
numactl --cpunodebind=0 --membind=0 java -jar myapp.jar
```

### Sybase / SAP ASE

```bash
# Similar to Oracle:
# 1. Hugepages for shared memory
# 2. Low swappiness
# 3. NUMA pinning per instance
# 4. Deadline/mq-deadline I/O scheduler

# Testing showed:
# - RHEL 6 with hugepages: 176K trans/min (vs 159K without) = 11% gain
# - KVM guest with hugepages: 135K (vs 114K RHEL 5.5) = 18% improvement
```

---

## Capacity Tuning Checklist

Tunables to increase when hitting resource limits:

```bash
# Memory
vm.overcommit_memory          # Overcommit mode
vm.overcommit_ratio           # Commit limit ratio
vm.max_map_count = 262144     # Max memory mappings per process (JVMs need this)
vm.nr_hugepages               # Hugepage reservation

# Kernel IPC (databases)
kernel.shmmax                 # Max shared memory segment size
kernel.shmall                 # Total shared memory pages
kernel.shmmni                 # Max shared memory segments
kernel.msgmax                 # Max message size
kernel.msgmnb                 # Max message queue size
kernel.msgmni                 # Max message queues
kernel.sem                    # Semaphore limits

# Filesystems
fs.file-max = 6553600        # System-wide file descriptor limit
fs.aio-max-nr = 1048576      # Max async I/O requests

# Processes
kernel.threads-max            # Max threads system-wide
kernel.pid_max = 131072       # Max PID value
```

---

## Tuned Profiles Quick Reference

RHEL 7+ uses `tuned` to apply performance profiles that bundle multiple sysctl, CPU governor, I/O scheduler, and other settings:

```bash
# List profiles
tuned-adm list

# Recommended profile for your system
tuned-adm recommend

# Apply
tuned-adm profile throughput-performance

# Check active
tuned-adm active
```

| Profile | Key Settings | Use Case |
|---------|-------------|----------|
| `throughput-performance` | governor=performance, swappiness=10, readahead=4096 | Server default (RHEL 7+) |
| `latency-performance` | force_latency=1, governor=performance | Low-latency (no power saving) |
| `network-latency` | THP=never, busy_poll=50, numa_balancing=0 | Financial, telecom |
| `virtual-host` | THP=always, KSM settings | KVM hypervisors |
| `virtual-guest` | dirty_ratio=30, swappiness=30 | VMs |

---

## Quick Reference

```bash
# === Memory Tunables ===
sysctl vm.swappiness=10
sysctl vm.dirty_background_ratio=5
sysctl vm.dirty_ratio=20
sysctl vm.min_free_kbytes=524288
sysctl vm.vfs_cache_pressure=50

# === Hugepages ===
echo 4000 > /proc/sys/vm/nr_hugepages
cat /proc/meminfo | grep Huge
cat /sys/kernel/mm/transparent_hugepage/enabled

# === I/O Scheduler ===
cat /sys/block/sda/queue/scheduler
echo "mq-deadline" > /sys/block/sda/queue/scheduler

# === CPU Governor ===
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
tuned-adm profile throughput-performance

# === Drop Caches (benchmarking only) ===
echo 1 > /proc/sys/vm/drop_caches    # pagecache
echo 2 > /proc/sys/vm/drop_caches    # slab
echo 3 > /proc/sys/vm/drop_caches    # both

# === OOM Protection ===
echo -1000 > /proc/<pid>/oom_score_adj

# === Monitoring ===
vmstat 1
mpstat -P ALL 1
iostat -xz 1
perf top
numastat
cat /proc/meminfo
cat /proc/vmstat
```
