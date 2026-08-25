# NUMA Performance Tuning on RHEL 7–10

Non-Uniform Memory Access (NUMA) is a memory architecture used in multi-socket servers where memory access time depends on which CPU socket requests it. Local memory is fast; remote memory (attached to another socket) is significantly slower. On multi-socket systems, NUMA awareness determines whether you get 100% or 70% of your hardware's potential.

This guide covers NUMA tuning across Red Hat Enterprise Linux 7 through 10, including the evolution of NUMA support, diagnostic tools, tuning strategies, and workload-specific guidance for bare metal, KVM, containers, and NFV.

---

## NUMA Architecture Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│                      4-Socket Server Example                        │
│                                                                     │
│  ┌────────────────┐  ┌────────────────┐                             │
│  │   Socket 0     │  │   Socket 1     │                             │
│  │  c1 c2 c3 c4   │  │  c1 c2 c3 c4   │                             │
│  │ Local Mem (M1) │──│ Local Mem (M2) │                             │
│  └────────────────┘  └────────────────┘                             │
│         │ QPI/UPI            │ QPI/UPI                              │
│  ┌────────────────┐  ┌────────────────┐                             │
│  │   Socket 2     │  │   Socket 3     │                             │
│  │  c1 c2 c3 c4   │  │  c1 c2 c3 c4   │                             │
│  │ Local Mem (M3) │──│ Local Mem (M4) │                             │
│  └────────────────┘  └────────────────┘                             │
│                                                                     │
│  Access latency (relative):                                         │
│    Local (same node):   ~10                                         │
│    Remote (1 hop):      ~21                                         │
│    Remote (2 hops):     ~31 (some topologies)                       │
└─────────────────────────────────────────────────────────────────────┘
```

- **Local access** — CPU accessing memory on its own node: fast (~10 relative)
- **Remote access** — CPU accessing memory on another node: 1.5–2x slower (~21 relative)
- **Without NUMA optimization** — Processes scatter memory across all nodes randomly
- **With NUMA optimization** — Each process keeps its memory on the local node

The performance penalty of remote access comes from crossing the QPI (Intel) or Infinity Fabric (AMD) interconnect between sockets. The kernel's NUMA subsystem tries to keep processes and their memory co-located on the same node.

---

## NUMA Evolution Across RHEL Versions

| Feature | RHEL 7 | RHEL 8 | RHEL 9 | RHEL 10 |
|---------|--------|--------|--------|---------|
| Automatic NUMA Balancing | Default ON | Default ON | Default ON | Default ON |
| numad daemon | Available | Available | Available | Available |
| Tuned NUMA-aware profiles | Yes | Yes | Yes | Yes |
| irqbalance NUMA-enhanced | Yes | Yes | Yes | Yes |
| cgroups NUMA support | v1 | v1 + v2 | v2 (default) | v2 |
| Transparent Hugepages | Default ON | Default ON | Default ON | Default ON |
| NUMA-aware memory tiering | No | No | Kernel support | Enhanced |
| CXL memory NUMA nodes | No | No | Basic | Full support |

### RHEL 7 NUMA Milestones

- Automatic NUMA balancing introduced as default (kernel.numa_balancing=1)
- `throughput-performance` tuned profile as server default (keeps NUMA balancing on)
- `network-latency` profile disables NUMA balancing for determinism
- Containers (Docker) inherit host NUMA topology
- NUMA-enhanced irqbalance

### RHEL 8 NUMA Changes

- cgroups v2 support with NUMA memory placement
- Improved NUMA balancing heuristics in kernel 4.18+
- Memory tiering foundations
- Better huge page NUMA placement
- `numactl` and `numad` remain primary tools

### RHEL 9 NUMA Changes

- Kernel 5.14+ with improved NUMA balancing for tiered memory
- CXL (Compute Express Link) memory exposed as NUMA nodes
- cgroups v2 as default with `memory.numa_stat`
- Improved transparent hugepage NUMA placement
- Better multi-level NUMA distance handling

### RHEL 10 NUMA Changes

- Kernel 6.x with memory tiering and demotion policies
- Full CXL memory support as additional NUMA tiers
- Enhanced automatic NUMA balancing with tiered awareness
- Improved cgroups v2 NUMA memory accounting
- AMD EPYC topology (CCX/CCD as NUMA domains) optimizations

---

## Check NUMA Topology

### numactl --hardware

```bash
numactl --hardware
```

Example output (4-socket server):

```
available: 4 nodes (0-3)
node 0 cpus: 0 4 8 12 16 20 24 28 32 36
node 0 size: 65415 MB
node 0 free: 43971 MB
node 1 cpus: 2 6 10 14 18 22 26 30 34 38
node 1 size: 65536 MB
node 1 free: 44321 MB
node 2 cpus: 1 5 9 13 17 21 25 29 33 37
node 2 size: 65536 MB
node 2 free: 44304 MB
node 3 cpus: 3 7 11 15 19 23 27 31 35 39
node 3 size: 65536 MB
node 3 free: 44329 MB
node distances:
node   0   1   2   3
  0:  10  21  21  21
  1:  21  10  21  21
  2:  21  21  10  21
  3:  21  21  21  10
```

**Reading the distance matrix:** distance 10 = local (fastest), 21 = one hop. Higher numbers mean more QPI/UPI hops and slower access. Some systems have asymmetric distances (e.g., 10, 16, 21) depending on the interconnect topology.

### lscpu

```bash
lscpu | grep -i numa
```

```
NUMA node(s):          4
NUMA node0 CPU(s):     0,4,8,12,16,20,24,28,32,36
NUMA node1 CPU(s):     2,6,10,14,18,22,26,30,34,38
NUMA node2 CPU(s):     1,5,9,13,17,21,25,29,33,37
NUMA node3 CPU(s):     3,7,11,15,19,23,27,31,35,39
```

### lstopo — Visualize NUMA Topology

`lstopo` from the `hwloc` package provides a graphical or text view of the full system topology including NUMA nodes, caches, cores, and PCI devices:

```bash
# Install
yum install hwloc    # RHEL 7
dnf install hwloc    # RHEL 8/9/10

# Text output
lstopo-no-graphics

# Graphical output (requires X11)
lstopo

# Export to image
lstopo topology.png
lstopo topology.pdf
```

`lstopo` shows which PCI devices (NICs, storage controllers) are attached to which NUMA node — critical for network and storage performance tuning. Each NIC will appear under its NUMA node, and routing traffic through CPUs on the same node as the NIC avoids cross-socket overhead.

### /sys/devices/system/node/

```bash
# List NUMA nodes
ls /sys/devices/system/node/

# Memory per node
cat /sys/devices/system/node/node0/meminfo

# CPUs in each node
cat /sys/devices/system/node/node0/cpulist

# Distance matrix
cat /sys/devices/system/node/node0/distance
```

### Full Hardware Discovery Commands

```bash
# CPU architecture
lscpu

# System topology (visual)
lstopo-no-graphics --whole-system

# NUMA node inventory
numactl --hardware

# DMI/BIOS info
dmidecode -t memory | grep -A5 "Memory Device"

# PCI device tree (shows NUMA affinity)
lspci -t -vv

# Block devices
lsblk

# Kernel boot flags
cat /proc/cmdline

# NIC NUMA node
cat /sys/class/net/eth0/device/numa_node
```

---

## NUMA Statistics and Diagnostics

### numastat — Basic View

```bash
numastat
```

```
                           node0           node1
numa_hit                 1234567          1234890
numa_miss                      7               12
numa_foreign                  12                7
interleave_hit              1234             1230
local_node               1234560          1234878
other_node                     7               12
```

### Key Metrics

| Metric | Meaning | Concern |
|--------|---------|---------|
| `numa_hit` | Allocation succeeded on the intended node | Good — higher is better |
| `numa_miss` | Allocation fell back to another node | Bad — remote memory access |
| `numa_foreign` | Another node intended to allocate here but couldn't | Bad — memory pressure |
| `local_node` | Allocation came from the local node | Good — should match numa_hit |
| `other_node` | Allocation came from a remote node | Bad — cross-node access |
| `interleave_hit` | Interleave policy allocation hit this node | Neutral — expected for interleaved allocs |

> **Red flag:** High `numa_miss` / `other_node` values mean processes are frequently accessing remote memory, adding latency to every memory operation.

### numastat -c (Compact Per-Process View)

Shows per-node memory allocation for specific processes. This is the best way to see if a process is NUMA-aligned:

```bash
numastat -c qemu
```

**Unaligned example** (memory scattered across all nodes):

```
Per-node process memory usage (in MBs)
PID              Node 0  Node 1  Node 2  Node 3  Total
---------------  ------  ------  ------  ------  -----
10587 (qemu-kvm)   1216    4022    4028    1456  10722
10629 (qemu-kvm)   2108      56     473    8077  10714
10671 (qemu-kvm)   4096    3470    3036     110  10712
10713 (qemu-kvm)   4043    3498    2135    1055  10730
---------------  ------  ------  ------  ------  -----
Total             11462   11045    9672   10698  42877
```

**Aligned example** (each VM's memory on a single node):

```
Per-node process memory usage (in MBs)
PID              Node 0  Node 1  Node 2  Node 3  Total
---------------  ------  ------  ------  ------  -----
10587 (qemu-kvm)      0   10723       5       0  10728
10629 (qemu-kvm)      0       0       5   10717  10722
10671 (qemu-kvm)      0       0   10726       0  10726
10713 (qemu-kvm)  10733       0       5       0  10738
---------------  ------  ------  ------  ------  -----
Total             10733   10723   10740   10717  42913
```

The aligned case gives significantly better performance because each VM accesses only local memory.

### numastat -mczs (System-Wide Per-Node Memory)

```bash
numastat -mczs
```

```
Per-node system memory usage (in MBs):
             Node 0  Node 1  Node 2  Node 3  Total
             ------  ------  ------  ------  ------
MemTotal      32766   32768   32768   32768  131070
MemFree       31863   31965   32120   32086  127934
MemUsed         903     803     648     682    3036
FilePages        11      26       8      37      82
Slab             25      16       7      10      58
Active            5      13       4      25      47
KernelStack       9       0       0       0       9
AnonPages         2       1       1       2       6
```

This view helps identify memory imbalance across nodes.

### numastat — Per-PID for Java/Database Workloads

```bash
# Default scheduler (non-optimal) — memory scattered
numastat -c java

PID           Node 0  Node 1  Node 2  Node 3  Total
-----------   ------  ------  ------  ------  -----
57501 (java)     755    1121     480     698   3054
57502 (java)    1068     702     573     723   3067
57503 (java)     649    1129     687     606   3071
57504 (java)    1202     678    1043     150   3073
-----------   ------  ------  ------  ------  -----
Total           3674    3630    2783    2177  12265

# After NUMA balancing (close to optimal) — each JVM on one node
numastat -c java

PID           Node 0  Node 1  Node 2  Node 3  Total
-----------   ------  ------  ------  ------  -----
56918 (java)      49    2791      56      37   2933
56919 (java)    2769      76      55      32   2932
56920 (java)      19      55      77    2780   2932
56921 (java)      97      65    2727      47   2936
-----------   ------  ------  ------  ------  -----
Total           2935    2987    2916    2896  11734
```

### Per-Process NUMA Maps

```bash
cat /proc/<pid>/numa_maps | head -20
```

Look for lines like `N0=XXX N1=YYY` — if memory is spread across nodes, the process is not optimally placed.

### Kernel NUMA Balancing Stats

```bash
grep numa /proc/vmstat
```

Key counters:
- `numa_pte_updates` — pages scanned for migration
- `numa_pages_migrated` — pages actually moved
- `numa_hint_faults` — access detected on wrong node
- `numa_hint_faults_local` — faults that were already local

### perf NUMA Tracepoints

Use `perf` to trace NUMA scheduler events:

```bash
# List NUMA-related tracepoints
perf list | grep sched: | grep numa

# Available tracepoints:
#   sched:sched_move_numa    — task moved to a different node
#   sched:sched_stick_numa   — task stayed on current node
#   sched:sched_swap_numa    — two tasks swapped between nodes

# Record NUMA scheduling events
perf record -e sched:sched_move_numa,sched:sched_stick_numa,sched:sched_swap_numa -a sleep 60
perf report
```

---

## Automatic NUMA Balancing

The kernel's automatic NUMA balancing periodically scans process memory mappings, detects cross-node access patterns, and migrates pages (and sometimes tasks) to reduce remote access.

```bash
# Check status (1 = enabled)
cat /proc/sys/kernel/numa_balancing

# Disable
echo 0 > /proc/sys/kernel/numa_balancing
# or
sysctl -w kernel.numa_balancing=0

# Enable
echo 1 > /proc/sys/kernel/numa_balancing
# or
sysctl -w kernel.numa_balancing=1

# Persistent
echo "kernel.numa_balancing = 1" >> /etc/sysctl.d/99-numa.conf
sysctl -p /etc/sysctl.d/99-numa.conf
```

### How It Works

1. Kernel periodically unmaps pages from a process's page tables (causes NUMA hint faults)
2. When the process accesses those pages, a minor page fault records which node the access came from
3. If the page is on the wrong node, the kernel migrates it to the local node
4. If many pages are on a remote node, the kernel may migrate the task to that node instead

### When to Disable NUMA Balancing

- **Manual pinning** — If using `numactl --membind` or `numad`, disable to avoid conflicts
- **Latency-sensitive workloads** — The page scanning and migration cause jitter
- **Real-time / NFV** — The tuned `network-latency` and `realtime` profiles disable it
- **Large hugepage allocations** — THP and static hugepages are not migrated by NUMA balancing

### SAP HANA Performance Example

Testing RHEL 6.6 vs RHEL 7.1 with SAP HANA showed significant improvement from auto NUMA balancing alone — up to 25-30% better query runtime at high user counts (100-150 concurrent users). The only difference was `kernel.numa_balancing = 1` in RHEL 7.1.

---

## numad — Automatic NUMA Affinity Daemon

`numad` is a userspace daemon that monitors NUMA topology and process resource consumption, then automatically pins significant processes to optimal NUMA nodes.

```bash
# Install
yum install numad    # RHEL 7
dnf install numad    # RHEL 8/9/10

# Enable and start
systemctl enable --now numad

# Check status
systemctl status numad

# View numad decisions
journalctl -u numad
```

### numad vs Kernel NUMA Balancing vs Manual Pinning

| Method | Scope | Granularity | Best For |
|--------|-------|-------------|----------|
| Kernel NUMA Balancing | Page-level migration | Individual pages | General mixed workloads |
| numad | Process-level pinning | Entire process + memory | Large long-running processes (VMs, DBs) |
| Manual (numactl) | Explicit placement | Full control | Deterministic latency requirements |

### Performance Comparison (4 KVM Guests, OLTP Workload)

Testing with 4 KVM guests running an OLTP workload showed:

- **No NUMA pinning**: Baseline
- **numad**: ~7% improvement over no-pin
- **Manual NUMA pinning**: ~7.5% improvement over no-pin

NUMAD performance is effectively as good as manual NUMA pinning, with the benefit of being dynamic and automatic.

### When to Use numad

- KVM/QEMU hypervisors with multiple VMs
- Multi-instance database servers (Oracle RAC, multiple PostgreSQL instances)
- HPC applications with large memory footprints
- Long-running applications consuming significant CPU and memory

### When NOT to Use numad

- Short-lived processes (only run a few minutes)
- Already using manual pinning with `numactl`
- Single-node systems
- Latency-sensitive workloads where any migration is unacceptable

---

## Manual NUMA Pinning (numactl)

For workloads that need deterministic placement:

```bash
# Run on node 0 (CPUs and memory)
numactl --cpunodebind=0 --membind=0 ./my_application

# Run on nodes 0 and 1
numactl --cpunodebind=0,1 --membind=0,1 ./my_application

# Interleave memory across all nodes (good for shared data)
numactl --interleave=all ./my_application

# Preferred node (use node 0, fall back to others if full)
numactl --preferred=0 ./my_application

# Show current NUMA policy
numactl --show
```

### Memory Policies

| Policy | Flag | Behavior | Use Case |
|--------|------|----------|----------|
| Bind | `--membind=N` | Only allocate from specified nodes (OOM if full) | Dedicated workloads |
| Preferred | `--preferred=N` | Try specified node first, fall back to others | Most pinning scenarios |
| Interleave | `--interleave=N` | Round-robin across specified nodes | memcached, Redis, shared caches |
| Local | (default) | Allocate from the node where CPU is running | Default kernel behavior |

> **Warning:** `--membind` will cause OOM kills if the bound node runs out of memory, even if other nodes have free memory. Use `--preferred` unless you need strict isolation.

### Examples

```bash
# Pin MySQL to node 0
numactl --cpunodebind=0 --membind=0 mysqld_safe &

# Pin a JVM to nodes 0-1
numactl --cpunodebind=0,1 --membind=0,1 java -jar app.jar

# Redis with interleaved memory (shared data)
numactl --interleave=all redis-server /etc/redis.conf

# PostgreSQL pinned to node 1
numactl --preferred=1 pg_ctl start -D /var/lib/pgsql/data
```

### NUMA-Aware systemd Services

**RHEL 7 (numactl wrapper):**

```ini
[Service]
ExecStart=/usr/bin/numactl --cpunodebind=0 --membind=0 /usr/bin/myapp
```

**RHEL 8+ (native NUMAPolicy):**

```ini
[Service]
NUMAPolicy=bind
NUMAMask=0
CPUAffinity=numa
```

Available policies: `default`, `preferred`, `bind`, `interleave`, `local`.

---

## Tuned Profiles and NUMA

The `tuned` daemon manages system-wide performance profiles that include NUMA-related settings.

### Profile Hierarchy

```
Parents:
  throughput-performance    balanced    latency-performance

Children:
  network-throughput        desktop     network-latency

Grandchildren:
  virtual-host                          realtime
  virtual-guest                         realtime-virtual-host
                                        realtime-virtual-guest
```

### Profile NUMA Settings

| Profile | NUMA Balancing | Use Case |
|---------|---------------|----------|
| `throughput-performance` | Enabled | Server default (RHEL 7+) |
| `latency-performance` | Enabled | General low-latency |
| `network-latency` | **Disabled** | Financial trading, telecom |
| `network-throughput` | Enabled | High-bandwidth networking |
| `virtual-host` | Enabled | KVM hypervisors |
| `virtual-guest` | Enabled | VMs |
| `realtime` | **Disabled** | RT workloads |
| `realtime-virtual-host` | **Disabled** | NFV host |
| `realtime-virtual-guest` | **Disabled** | NFV guest |

### network-latency Profile Details

```
include=latency-performance
transparent_hugepages=never
net.core.busy_read=50
net.core.busy_poll=50
net.ipv4.tcp_fastopen=3
kernel.numa_balancing=0       ← NUMA balancing disabled
```

This profile disables NUMA balancing because for latency-sensitive workloads, the overhead of page scanning and migration causes unpredictable jitter.

### throughput-performance Profile Details

```
governor=performance
energy_perf_bias=performance
min_perf_pct=100
readahead=4096
kernel.sched_min_granularity_ns=10000000
kernel.sched_wakeup_granularity_ns=15000000
vm.dirty_background_ratio=10
vm.swappiness=10
```

NUMA balancing remains enabled — appropriate for throughput workloads where occasional page migration overhead is acceptable.

### Apply a Profile

```bash
# List available profiles
tuned-adm list

# Apply
tuned-adm profile throughput-performance

# Check active profile
tuned-adm active

# Recommend profile based on system type
tuned-adm recommend
```

---

## KVM/QEMU NUMA Tuning

In KVM, guests run as processes in userspace on the host. A virtual CPU is implemented using a Linux thread, scheduled by the host's Linux scheduler. Guests inherit NUMA, huge pages, and hardware support from the host kernel.

### Check VM NUMA Alignment

```bash
# Check vCPU placement
virsh vcpuinfo <vm_name>

# Check NUMA memory allocation for QEMU processes
numastat -c qemu

# Detailed per-PID
numastat -p $(pgrep -f "qemu.*<vm_name>")
```

### Pin vCPUs to NUMA Nodes

```bash
# Pin vCPU 0 to physical CPUs on node 0
virsh vcpupin <vm_name> 0 0-9

# Pin vCPU 1 to physical CPUs on node 0
virsh vcpupin <vm_name> 1 0-9

# Set NUMA memory policy
virsh numatune <vm_name> --mode strict --nodeset 0
```

### libvirt XML NUMA Configuration

```xml
<domain>
  <numatune>
    <memory mode="strict" nodeset="0"/>
  </numatune>
  <cputune>
    <vcpupin vcpu="0" cpuset="0"/>
    <vcpupin vcpu="1" cpuset="4"/>
    <vcpupin vcpu="2" cpuset="8"/>
    <vcpupin vcpu="3" cpuset="12"/>
  </cputune>
</domain>
```

### VM Sizing Rule

**Size VMs to fit within a single NUMA node.** A VM with 8 vCPUs and 32GB RAM should be placed on a node that has at least 8 physical cores and 32GB RAM. A VM split across nodes performs significantly worse.

### I/O Cache Settings Impact on NUMA

| Cache Mode | Behavior | NUMA Impact |
|------------|----------|-------------|
| `cache=none` | Guest I/O not cached on host | Less host memory pressure per node |
| `cache=writethrough` | I/O cached and written through on host | Higher host memory use, potential cross-node cache |

`cache=none` is generally recommended for multi-VM environments to avoid host memory pressure that leads to cross-node allocations.

### Transparent Huge Pages and KVM

Testing with 4 VMs running OLTP workloads showed THP with a scan interval of 100ms performs comparably to static hugepages. However, KSM (Kernel Samepage Merging) breaks down THP pages, losing the THP performance advantage. If KSM is enabled, lower the THP scan interval to recover some performance.

Recommendation:
- **Density-optimized**: Enable KSM, accept THP fragmentation
- **Performance-optimized**: Disable KSM, keep THP or use static hugepages

### Hyperthreading + NUMA

Testing with multi-instance databases aligned to NUMA nodes showed approximately 35% improvement with hyperthreading enabled. When each instance is aligned to its own NUMA node, hyperthreading provides additional throughput without cross-node memory access.

---

## Container NUMA Tuning

### RHEL 7 Container Architecture

Containers share the host kernel and inherit its NUMA topology. The isolation stack consists of:
- **cgroups** — Resource limits (CPU, memory)
- **Namespaces** — Process/network isolation
- **SELinux** — Security

### NUMA-Aware Container Placement

```bash
# Run container pinned to NUMA node 0
docker run --cpuset-cpus="0,4,8,12" --cpuset-mems="0" myimage

# Podman (RHEL 8+) with NUMA pinning
podman run --cpuset-cpus="0,4,8,12" --cpuset-mems="0" myimage

# Kubernetes (via CPU Manager and Topology Manager)
# Set kubelet policies:
#   --cpu-manager-policy=static
#   --topology-manager-policy=single-numa-node
```

### Container Performance vs Bare Metal

Testing across workloads showed containers have minimal overhead compared to bare metal:
- **CPU-bound (calculate primes)**: 100% of bare metal
- **OLTP workload**: 97% of bare metal
- **Analytics application**: 94% of bare metal

The slight overhead comes from network namespacing (bridge) and storage layers, not from NUMA-related issues.

### Tuned Atomic/Container Profiles

The `atomic-host` and `atomic-guest` profiles inherit from `throughput-performance` and add container-specific tuning:

```
# atomic-host / atomic-guest additions:
avc_cache_threshold=65536
nf_conntrack_hashsize=131072
kernel.pid_max=131072
net.netfilter.nf_conntrack_max=1048576
```

---

## Network NUMA Alignment

### NIC-to-Node Mapping

Every network interface is physically attached to a specific NUMA node via PCI. Running network-intensive applications on the same node as their NIC avoids cross-socket overhead:

```bash
# Find which NUMA node a NIC is on
cat /sys/class/net/eth0/device/numa_node
cat /sys/class/net/ens1f0/device/numa_node

# Find IRQs for a NIC
grep eth0 /proc/interrupts
cat /proc/irq/<irq_number>/smp_affinity_list
```

### Pin Application to NIC's NUMA Node

```bash
# If eth0 is on node 1:
numactl --cpunodebind=1 --membind=1 ./network_app

# Pin IRQs to same node
# (irqbalance usually handles this, but for manual control:)
echo <node1_cpumask> > /proc/irq/<irq>/smp_affinity
```

### irqbalance NUMA Awareness

`irqbalance` in RHEL 7+ is NUMA-enhanced — it distributes IRQs across CPUs while respecting NUMA locality (keeping NIC IRQs on the same node as the NIC):

```bash
systemctl status irqbalance

# For latency-sensitive NICs, consider pinning IRQs manually
# and stopping irqbalance for those specific IRQs
```

### DPDK and NFV NUMA Requirements

For DPDK (Data Plane Development Kit) and NFV workloads:

```bash
# Boot options for NFV
# CPU cstate=1, reserve 1GB hugepages, isolate CPUs
GRUB_CMDLINE_LINUX="... intel_idle.max_cstate=1 hugepagesz=1G hugepages=16 isolcpus=2-15"

# DPDK applications must use CPUs and hugepage memory from the same NUMA node
# as the NIC's PCI function
```

---

## Low-Latency and Real-Time NUMA Considerations

### Why Disable NUMA Balancing for Low Latency

The NUMA balancer's page scanning and migration introduces:
- TLB shootdowns (interrupt all CPUs using the page)
- Page migration latency (copy data between nodes)
- Unpredictable jitter during the migration

For latency-sensitive workloads, this jitter is unacceptable. Instead, pin everything manually.

### Low-Latency NUMA Tuning Checklist

```bash
# 1. Disable NUMA balancing
sysctl -w kernel.numa_balancing=0

# 2. Use tuned latency profile
tuned-adm profile network-latency
# or for real-time:
tuned-adm profile realtime

# 3. Pin application to specific node
numactl --cpunodebind=0 --membind=0 ./trading_app

# 4. Isolate CPUs from scheduler
# (via grub: isolcpus=2-7 or via cgroups)

# 5. Use nohz_full for tickless operation on latency-sensitive cores
# (via grub: nohz_full=2-7)

# 6. Pin IRQs away from application cores
# (use tuna or manual /proc/irq/*/smp_affinity)

# 7. Disable THP (causes compaction stalls)
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# 8. Lock C-state to C0/C1 (avoid wakeup latency)
# (tuned latency-performance does this via force_latency=1)
```

### nohz_full — Tickless Active Cores

Available since RHEL 7, `nohz_full` stops the kernel timer tick on specified CPUs when only one task is running. This eliminates periodic interruptions to latency-sensitive tasks:

```bash
# Kernel command line
nohz_full=2-15

# Moves timekeeping to non-latency-sensitive cores (core 0)
# If nr_running=1 on a nohz_full core, scheduler tick is suppressed
```

---

## Diagnosing NUMA Problems

### Symptoms of Poor NUMA Alignment

- Higher than expected memory latency
- CPU utilization looks low but application is slow
- `numastat` shows high `numa_miss` / `other_node`
- Performance varies between runs (different CPU placement)
- `perf stat` shows high LLC cache misses

### Diagnostic Workflow

```bash
# 1. Understand topology
numactl --hardware
lscpu | grep NUMA
lstopo-no-graphics

# 2. Check current allocations
numastat -v
numastat -c <process_name>

# 3. Check per-process NUMA maps
cat /proc/<pid>/numa_maps | head -20

# 4. Check kernel NUMA balancing activity
grep numa /proc/vmstat

# 5. Monitor in real time
watch -n 2 numastat

# 6. Benchmark local vs remote access
numactl --cpunodebind=0 --membind=0 sysbench memory run   # local
numactl --cpunodebind=0 --membind=1 sysbench memory run   # remote
# Remote is typically 30-50% slower
```

### perf c2c — Cacheline Contention

Available since RHEL 7.3, `perf c2c` detects cacheline false sharing across NUMA nodes. When CPUs on different sockets modify data in the same 64-byte cacheline, the cache coherency protocol causes significant performance degradation:

```bash
# Record cacheline contention data
perf c2c record -a sleep 10

# Report
perf c2c report
```

The report shows:
- Hottest contended cachelines
- Process names, data addresses, instruction pointers, PIDs, TIDs
- Which NUMA nodes and CPUs are accessing each cacheline
- Whether access is read or write

This helps identify false sharing in multi-threaded applications — a common cause of poor scaling on NUMA systems.

---

## Performance Co-Pilot (PCP) NUMA Monitoring

PCP provides distributed, historical performance monitoring including NUMA metrics:

```bash
# Install
yum install pcp pcp-system-tools    # RHEL 7
dnf install pcp pcp-system-tools    # RHEL 8/9/10

# Enable
systemctl enable --now pmcd pmlogger

# All-in-one subsystem view (CPU, disk, network, memory)
pmcollectl -s cdnm

# NUMA-specific metrics
pminfo -f mem.numa
pminfo -f kernel.all.numa

# Record for historical analysis
pmlogger -c /etc/pcp/pmlogger/config.default /var/log/pcp/
```

---

## Best Practices Summary

1. **Always check topology** — Run `numactl --hardware` and `lstopo` on all multi-socket servers
2. **Monitor `numastat` regularly** — High `numa_miss` values indicate suboptimal placement
3. **Let the kernel handle it first** — Automatic NUMA balancing works for most general workloads
4. **Use `numad` for VM hosts** — Especially on KVM hypervisors with multiple VMs
5. **Pin large databases** — Oracle, PostgreSQL, MySQL, SAP HANA benefit from explicit NUMA binding
6. **Size VMs within a single node** — Never create a VM larger than one node's CPU/memory
7. **Interleave for shared caches** — Use `numactl --interleave=all` for memcached, Redis
8. **Match applications to their NICs** — Check `cat /sys/class/net/<iface>/device/numa_node`
9. **Disable NUMA balancing for latency workloads** — Use `network-latency` or `realtime` profiles
10. **Don't over-pin** — Binding to one node wastes the other nodes if workload could use them
11. **Monitor after changes** — Always verify with `numastat` before and after tuning
12. **Consider hyperthreading** — HT with NUMA alignment gives ~35% improvement for multi-instance databases
13. **Align IRQs to application nodes** — irqbalance does this automatically, verify with `/proc/interrupts`
14. **Use `perf c2c` for scaling issues** — Cacheline contention across nodes can silently kill performance

---

## Quick Reference

```bash
# === Topology ===
numactl --hardware
lscpu | grep NUMA
lstopo-no-graphics

# === Statistics ===
numastat
numastat -c <process>
numastat -mczs
cat /proc/<pid>/numa_maps
grep numa /proc/vmstat

# === Run on specific node ===
numactl --cpunodebind=0 --membind=0 ./app
numactl --preferred=0 ./app
numactl --interleave=all ./app

# === NUMA Balancing ===
cat /proc/sys/kernel/numa_balancing
sysctl -w kernel.numa_balancing=0    # disable
sysctl -w kernel.numa_balancing=1    # enable

# === numad daemon ===
systemctl enable --now numad

# === Tuned profiles ===
tuned-adm profile throughput-performance    # NUMA balancing ON
tuned-adm profile network-latency           # NUMA balancing OFF
tuned-adm profile realtime                  # NUMA balancing OFF

# === NIC NUMA node ===
cat /sys/class/net/<iface>/device/numa_node

# === KVM NUMA pinning ===
virsh vcpupin <vm> 0 0-9
virsh numatune <vm> --mode strict --nodeset 0
numastat -c qemu

# === Container NUMA ===
podman run --cpuset-cpus="0,4,8,12" --cpuset-mems="0" myimage

# === Cacheline contention ===
perf c2c record -a sleep 10
perf c2c report

# === NUMA scheduler tracepoints ===
perf record -e sched:sched_move_numa -a sleep 60
```
