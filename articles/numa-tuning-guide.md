# NUMA Tuning Guide

Non-Uniform Memory Access (NUMA) is a memory architecture where memory access time depends on which CPU socket requests it. Local memory is fast; remote memory (attached to another socket) is slower. On multi-socket systems, NUMA awareness determines whether you get 100% or 70% of your hardware's potential.

## NUMA Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Multi-Socket Server                      │
│                                                             │
│  ┌──────────────────┐         ┌──────────────────┐          │
│  │   Socket 0       │         │   Socket 1       │          │
│  │                  │         │                  │          │
│  │  CPU 0,2,4,6,8   │◄──QPI──►│  CPU 1,3,5,7,9   │          │
│  │                  │         │                  │          │
│  │  Local Memory    │         │  Local Memory    │          │
│  │  (Node 0: 64GB)  │         │  (Node 1: 64GB)  │          │
│  └──────────────────┘         └──────────────────┘          │
│                                                             │
│  Access latency:                                            │
│    Local (same node):  ~10 (relative)                       │
│    Remote (cross-node): ~21 (relative)                      │
└─────────────────────────────────────────────────────────────┘
```

- **Local access** (CPU accessing memory on its own node): fast
- **Remote access** (CPU accessing memory on another node): 1.5–2x slower
- The kernel's NUMA balancing tries to keep processes and their memory together

## Check NUMA Topology

### numactl --hardware

```bash
# Show NUMA nodes, CPUs, memory, and distances
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

### lscpu (NUMA Summary)

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

## NUMA Statistics (numastat)

`numastat` shows per-node memory allocation statistics — the key diagnostic tool for NUMA problems.

```bash
# Basic NUMA hit/miss stats
numastat

# Verbose per-node stats (in MB)
numastat -v

# Per-process NUMA allocation
numastat -p <pid>
numastat -p qemu-kvm

# Per-node memory info (similar to /proc/meminfo but per node)
numastat -m
```

### Key Metrics to Watch

| Metric | Meaning | Concern |
|--------|---------|---------|
| `numa_hit` | Allocation succeeded on the intended node | Good — higher is better |
| `numa_miss` | Allocation fell back to another node | Bad — process accessing remote memory |
| `numa_foreign` | Another node intended to allocate here but couldn't | Bad — memory pressure on this node |
| `local_node` | Allocation came from the local node | Good — should match numa_hit |
| `other_node` | Allocation came from a remote node | Bad — cross-node access |
| `interleave_hit` | Interleave policy allocation hit this node | Neutral — expected for interleaved allocs |

```bash
# Example output
numastat
                           node0           node1
numa_hit                 1234567          1234890
numa_miss                      7               12
numa_foreign                  12                7
interleave_hit              1234             1230
local_node               1234560          1234878
other_node                     7               12
```

> **Red flag:** High `numa_miss` / `other_node` values mean processes are frequently accessing remote memory. This adds latency to every memory operation.

## Automatic NUMA Balancing (Kernel)

RHEL 7+ kernels include automatic NUMA balancing — the kernel periodically scans process memory mappings, detects cross-node access, and migrates pages to the local node.

```bash
# Check if NUMA balancing is enabled (1 = enabled)
cat /proc/sys/kernel/numa_balancing

# Disable NUMA balancing
echo 0 > /proc/sys/kernel/numa_balancing

# Enable NUMA balancing
echo 1 > /proc/sys/kernel/numa_balancing

# Persistent (via sysctl)
echo "kernel.numa_balancing = 1" >> /etc/sysctl.d/99-numa.conf
sysctl -p /etc/sysctl.d/99-numa.conf
```

> For most workloads, automatic NUMA balancing works well out of the box. Disable it only if you're doing manual pinning with `numactl` or `numad`, as they can conflict.

## numad — Automatic NUMA Affinity Daemon

`numad` is a userspace daemon that monitors NUMA topology and process resource consumption, then automatically pins significant processes to optimal NUMA nodes.

```bash
# Install
yum install numad    # RHEL 7
dnf install numad    # RHEL 8/9

# Enable and start
systemctl enable --now numad

# Check status
systemctl status numad

# View numad decisions
cat /var/log/numad.log
```

### When numad Helps

- KVM/QEMU virtual machines (pins vCPUs and memory to local nodes)
- HPC applications with large memory footprints
- Long-running processes consuming significant resources
- Multi-instance database servers

### When numad Doesn't Help

- Short-lived processes (run only a few minutes)
- Processes using very little memory or CPU
- Single-node systems (nothing to optimize)

### numad vs Kernel NUMA Balancing

| | Kernel NUMA Balancing | numad |
|---|---|---|
| **Scope** | Page-level migration | Process-level pinning |
| **Overhead** | Minimal (kernel TLB scanning) | Low (periodic userspace checks) |
| **Granularity** | Individual pages | Entire process + its memory |
| **Best for** | General mixed workloads | Large, long-running processes (VMs, DBs) |
| **Conflict** | Can conflict with manual pinning | Can conflict with kernel balancing |

> On RHEL 7+, kernel NUMA balancing is usually sufficient. Use `numad` for KVM hypervisors or when running multiple large applications that should be isolated to separate sockets.

## Manual NUMA Pinning (numactl)

For workloads that need deterministic placement, use `numactl` to bind processes to specific nodes:

```bash
# Run process on node 0 CPUs and memory
numactl --cpunodebind=0 --membind=0 ./my_application

# Run on nodes 0 and 1
numactl --cpunodebind=0,1 --membind=0,1 ./my_application

# Interleave memory across all nodes (good for shared data structures)
numactl --interleave=all ./my_application

# Preferred node (use node 0, fall back to others if full)
numactl --preferred=0 ./my_application

# Show current NUMA policy for a running process
numactl --show
cat /proc/<pid>/numa_maps

# Pin MySQL to node 0
numactl --cpunodebind=0 --membind=0 mysqld_safe &

# Pin a JVM to nodes 0-1
numactl --cpunodebind=0,1 --membind=0,1 java -jar app.jar
```

### numactl Policies

| Policy | Flag | Behavior |
|--------|------|----------|
| Bind | `--membind=N` | Allocate memory only from specified nodes (OOM if full) |
| Preferred | `--preferred=N` | Try specified node first, fall back to others |
| Interleave | `--interleave=N` | Round-robin across specified nodes |
| Local | (default) | Allocate from the node where the CPU is running |

> **Warning:** `--membind` will cause OOM kills if the bound node runs out of memory, even if other nodes have free memory. Use `--preferred` for most cases.

## NUMA-Aware systemd Services

Pin services to specific NUMA nodes via systemd unit files:

```bash
# Create a drop-in for a service
systemctl edit myapp.service
```

```ini
[Service]
# Pin to NUMA node 0
ExecStart=
ExecStart=/usr/bin/numactl --cpunodebind=0 --membind=0 /usr/bin/myapp
```

Or use `CPUAffinity` and `NUMAPolicy` (systemd 243+, RHEL 8.2+):

```ini
[Service]
NUMAPolicy=bind
NUMAMask=0
CPUAffinity=numa
```

## KVM/QEMU NUMA Tuning

For virtual machines, NUMA alignment is critical. A VM split across nodes performs significantly worse:

```bash
# Check if a VM's vCPUs are NUMA-aligned
virsh vcpuinfo <vm_name>

# Pin vCPUs to specific physical CPUs
virsh vcpupin <vm_name> 0 0-9    # vCPU 0 → physical CPUs 0-9 (node 0)
virsh vcpupin <vm_name> 1 0-9    # vCPU 1 → physical CPUs 0-9 (node 0)

# Set NUMA memory policy for a VM
virsh numatune <vm_name> --mode strict --nodeset 0

# Check NUMA memory allocation for QEMU processes
numastat -p $(pgrep qemu)
```

> `numad` handles this automatically for libvirt VMs when enabled. It's the recommended approach unless you need specific pinning.

## Diagnosing NUMA Problems

### Symptoms of Poor NUMA Alignment

- Higher than expected memory latency
- CPU utilization looks low but application is slow
- `numastat` shows high `numa_miss` / `other_node`
- Performance varies depending on which CPU a process lands on

### Diagnostic Workflow

```bash
# 1. Check topology
numactl --hardware
lscpu | grep NUMA

# 2. Check current allocations
numastat -v
numastat -p <pid>    # For specific process

# 3. Check per-process NUMA maps
cat /proc/<pid>/numa_maps | head -20
# Look for: N0=XXX N1=YYY — if spread across nodes, not optimal

# 4. Check if NUMA balancing is doing its job
grep -r "" /proc/vmstat | grep numa
# numa_pte_updates — pages scanned for migration
# numa_pages_migrated — pages actually moved
# numa_hint_faults — access detected on wrong node

# 5. Monitor in real time
watch -n 2 numastat
```

### Performance Impact Example

```bash
# Benchmark: local vs remote memory access
# Local node access (fast)
numactl --cpunodebind=0 --membind=0 sysbench memory run

# Remote node access (slow — forces cross-node)
numactl --cpunodebind=0 --membind=1 sysbench memory run

# Compare throughput — remote access is typically 30-50% slower
```

## Best Practices

1. **Check topology first** — Run `numactl --hardware` and `lscpu` on all multi-socket servers to understand the layout
2. **Monitor `numastat`** — High `numa_miss` values indicate suboptimal placement
3. **Let the kernel handle it** — Automatic NUMA balancing works for most workloads without intervention
4. **Use `numad` for VMs** — Especially on KVM hypervisors with multiple VMs
5. **Pin large databases** — Oracle, PostgreSQL, MySQL benefit from explicit NUMA binding
6. **Size VMs within a single node** — Don't create a VM larger than one node's CPU/memory
7. **Interleave for shared caches** — Use `numactl --interleave=all` for memcached, Redis, or other shared-memory workloads
8. **Don't over-pin** — Binding to one node wastes the other node's resources if not needed
9. **Match NUMA to NICs** — Network-intensive apps should run on the same node as their NIC (`cat /sys/class/net/eth0/device/numa_node`)
10. **Monitor after changes** — Use `numastat` before and after pinning to verify improvement

## Quick Reference

```bash
# Topology
numactl --hardware
lscpu | grep NUMA

# Statistics
numastat
numastat -p <pid>
numastat -m

# Run on specific node
numactl --cpunodebind=0 --membind=0 ./app
numactl --preferred=0 ./app
numactl --interleave=all ./app

# NUMA balancing
cat /proc/sys/kernel/numa_balancing
echo 0 > /proc/sys/kernel/numa_balancing    # disable
echo 1 > /proc/sys/kernel/numa_balancing    # enable

# numad daemon
systemctl enable --now numad

# NIC NUMA node
cat /sys/class/net/<iface>/device/numa_node

# Process NUMA mapping
cat /proc/<pid>/numa_maps
```
