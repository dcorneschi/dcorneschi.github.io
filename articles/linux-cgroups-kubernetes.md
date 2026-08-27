# Linux cgroups and Kubernetes

How Kubernetes resource requests and limits translate to Linux cgroup settings — cgroup v1 vs v2 hierarchy, CPU throttling (CFS quota), memory limits (OOMKill), QoS class mapping, and what the kubelet actually configures on the node.

## What cgroups Are

cgroups (control groups) are a Linux kernel mechanism for limiting, accounting, and isolating resource usage (CPU, memory, I/O, PIDs) of process groups.

Every container in Kubernetes runs inside a cgroup. The kubelet creates and manages the cgroup hierarchy on each node.

```
┌────────────────────────────────────────────────────────────────────┐
│  Node                                                              │
│                                                                    │
│  /sys/fs/cgroup/                                                   │
│  ├── kubepods/                   ← All pod cgroups live here       │
│  │   ├── burstable/              ← QoS: Burstable pods             │
│  │   │   ├── pod<uid>/           ← One pod                         │
│  │   │   │   ├── <container-id>  ← One container's cgroup          │
│  │   │   │   └── <container-id>                                    │
│  │   │   └── pod<uid>/                                             │
│  │   ├── besteffort/             ← QoS: BestEffort pods            │
│  │   │   └── pod<uid>/                                             │
│  │   └── pod<uid>/               ← QoS: Guaranteed (top-level)     │
│  │       └── <container-id>                                        │
│  ├── system.slice/               ← System services (kubelet, etc)  │
│  └── user.slice/                 ← User processes                  │
└────────────────────────────────────────────────────────────────────┘
```

## cgroup v1 vs v2

| Feature | cgroup v1 | cgroup v2 |
|---------|:---------:|:---------:|
| Hierarchy | Multiple hierarchies (one per controller) | Single unified hierarchy |
| Controllers | cpu, memory, blkio, pids... each separate tree | All controllers in one tree |
| Default on | AL2, Ubuntu < 22.04, RHEL < 9 | AL2023, Ubuntu 22.04+, RHEL 9+ |
| Kubernetes support | All versions | 1.25+ (stable) |
| Memory pressure detection | Limited (no PSI) | Full PSI (Pressure Stall Info) |
| OOM handling | Per-cgroup OOM killer | Improved (oom.group, memory.oom.group) |
| I/O throttling | blkio (limited) | io controller (better, includes writeback) |
| File paths | `/sys/fs/cgroup/<controller>/<path>` | `/sys/fs/cgroup/<path>` |

```bash
# Check which version your node uses:
stat -fc %T /sys/fs/cgroup/
# "cgroup2fs" = v2
# "tmpfs"     = v1

# Or:
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2 = v2
# cgroup on /sys/fs/cgroup/cpu type cgroup = v1
```

## How Kubernetes Maps to cgroups

### The Translation

```
┌───────────────────────────────────────────────────────────────────┐
│  Pod Spec                   →    cgroup Settings                  │
│                                                                   │
│  resources:                                                       │
│    requests:                                                      │
│      cpu: 500m              →    cpu.shares = 512 (v1)            │
│                                  cpu.weight = 20 (v2)             │
│      memory: 256Mi          →    (scheduling only, not enforced)  │
│                                                                   │
│    limits:                                                        │
│      cpu: 1000m             →    cpu.cfs_quota_us = 100000 (v1)   │
│                                  cpu.max = "100000 100000" (v2)   │
│      memory: 512Mi          →    memory.limit_in_bytes = 512Mi(v1)│
│                                  memory.max = 536870912 (v2)      │
└───────────────────────────────────────────────────────────────────┘
```

## CPU: Requests → cpu.shares/cpu.weight

CPU **requests** are translated to **proportional weight** — they don't enforce a hard cap, they determine priority when the node is contended:

### cgroup v1: cpu.shares

```
Formula: cpu.shares = max(2, cpu_request_millicores * 1024 / 1000)

Examples:
  requests.cpu: 100m  → cpu.shares = 102
  requests.cpu: 250m  → cpu.shares = 256
  requests.cpu: 500m  → cpu.shares = 512
  requests.cpu: 1000m → cpu.shares = 1024
  requests.cpu: 2000m → cpu.shares = 2048
```

```bash
# Check a container's cpu.shares (v1):
cat /sys/fs/cgroup/cpu/kubepods/burstable/pod<uid>/<container-id>/cpu.shares
```

**How shares work:**
- Shares are relative, not absolute
- A container with 1024 shares gets 2x the CPU of one with 512 shares (when both are competing)
- If no contention, a container can use ALL available CPU regardless of shares
- Shares only matter when the node is CPU-saturated

### cgroup v2: cpu.weight

```
Formula: cpu.weight = max(1, min(10000, (cpu_request_millicores * 100 / 1000) + 1))

Range: 1 to 10000 (default 100)

Examples:
  requests.cpu: 100m  → cpu.weight ≈ 11
  requests.cpu: 500m  → cpu.weight ≈ 51
  requests.cpu: 1000m → cpu.weight ≈ 101
  requests.cpu: 4000m → cpu.weight ≈ 401
```

```bash
# Check a container's cpu.weight (v2):
cat /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod<uid>.slice/<container-id>/cpu.weight
```

## CPU: Limits → CFS Quota (The Throttling Mechanism)

CPU **limits** are translated to a hard cap via the Completely Fair Scheduler (CFS) bandwidth control:

### cgroup v1: cpu.cfs_quota_us + cpu.cfs_period_us

```
Formula:
  cpu.cfs_period_us = 100000 (100ms, fixed by kubelet)
  cpu.cfs_quota_us = cpu_limit_millicores * 100

Examples:
  limits.cpu: 100m  → quota = 10000   (10ms out of every 100ms period)
  limits.cpu: 500m  → quota = 50000   (50ms out of every 100ms)
  limits.cpu: 1000m → quota = 100000  (100ms out of 100ms = 1 full core)
  limits.cpu: 2000m → quota = 200000  (200ms = 2 cores' worth per period)
  No limit set      → quota = -1      (unlimited)
```

```bash
# Check a container's CFS quota (v1):
cat /sys/fs/cgroup/cpu/kubepods/burstable/pod<uid>/<container-id>/cpu.cfs_quota_us
cat /sys/fs/cgroup/cpu/kubepods/burstable/pod<uid>/<container-id>/cpu.cfs_period_us

# Check throttle stats:
cat /sys/fs/cgroup/cpu/kubepods/burstable/pod<uid>/<container-id>/cpu.stat
# nr_periods: total scheduling periods
# nr_throttled: periods where the container was throttled
# throttled_time: total nanoseconds throttled
```

### cgroup v2: cpu.max

```
Format: cpu.max = "<quota> <period>"

Examples:
  limits.cpu: 500m  → cpu.max = "50000 100000"
  limits.cpu: 1000m → cpu.max = "100000 100000"
  limits.cpu: 2000m → cpu.max = "200000 100000"
  No limit          → cpu.max = "max 100000"
```

```bash
# Check (v2):
cat /sys/fs/cgroup/kubepods.slice/.../cpu.max

# Throttle stats (v2):
cat /sys/fs/cgroup/kubepods.slice/.../cpu.stat
# throttled_usec: total microseconds throttled
```

### Why CPU Throttling Happens (Even When Node Has Free CPU)

```
┌────────────────────────────────────────────────────────────────┐
│  The CFS Quota Problem                                         │
│                                                                │
│  Container limit: 1 CPU (quota = 100000us per 100ms period)    │
│  Container has 4 threads                                       │
│                                                                │
│  Thread 1: uses 30ms ─┐                                        │
│  Thread 2: uses 30ms  ├─ Total: 110ms consumed in period       │
│  Thread 3: uses 25ms  │                                        │
│  Thread 4: uses 25ms ─┘                                        │
│                                                                │
│  After 100ms of quota consumed (e.g., at 60ms wall-clock):     │
│  → ALL threads throttled for remaining 40ms of the period      │
│  → Node may have 7 idle cores, doesn't matter                  │
│  → CFS quota is absolute, not relative to available CPU        │
│                                                                │
│  This is why "CPU throttling with available CPU" happens       │
└────────────────────────────────────────────────────────────────┘
```

## Memory: Requests (Scheduling Only)

Memory **requests** don't create any cgroup setting. They're used only by the scheduler to decide which node the pod fits on:

```
requests.memory: 256Mi
→ Scheduler reserves 256Mi from node's allocatable memory
→ No cgroup enforcement — container CAN use more
→ Only affects scheduling decisions
```

## Memory: Limits → Hard Cap (OOMKill)

Memory **limits** are enforced as a hard cgroup limit. Exceeding it triggers the OOM killer:

### cgroup v1: memory.limit_in_bytes

```
Formula: memory.limit_in_bytes = memory_limit_bytes

Examples:
  limits.memory: 256Mi → memory.limit_in_bytes = 268435456
  limits.memory: 1Gi   → memory.limit_in_bytes = 1073741824
  No limit             → memory.limit_in_bytes = 9223372036854771712 (max int64)
```

```bash
# Check (v1):
cat /sys/fs/cgroup/memory/kubepods/burstable/pod<uid>/<container-id>/memory.limit_in_bytes
cat /sys/fs/cgroup/memory/kubepods/burstable/pod<uid>/<container-id>/memory.usage_in_bytes

# OOM events:
cat /sys/fs/cgroup/memory/kubepods/burstable/pod<uid>/<container-id>/memory.oom_control
# oom_kill_disable: 0 (OOM killer enabled)
# under_oom: 0 (not currently OOM)
# oom_kill: 3 (killed 3 times)
```

### cgroup v2: memory.max

```
Formula: memory.max = memory_limit_bytes

Examples:
  limits.memory: 256Mi → memory.max = 268435456
  limits.memory: 1Gi   → memory.max = 1073741824
  No limit             → memory.max = max
```

```bash
# Check (v2):
cat /sys/fs/cgroup/kubepods.slice/.../memory.max
cat /sys/fs/cgroup/kubepods.slice/.../memory.current

# OOM events (v2):
cat /sys/fs/cgroup/kubepods.slice/.../memory.events
# oom: 2 (OOM events count)
# oom_kill: 2 (OOM kills count)
```

### What Happens on OOM

```
Container allocates memory beyond memory.max
    │
    ▼
Kernel tries to reclaim (page cache, swap if any)
    │
    ▼
Still over limit → kernel OOM killer activates
    │
    ▼
Kills a process in the cgroup (usually the main process)
    │
    ▼
Container exits with code 137 (SIGKILL = 128 + 9)
    │
    ▼
Kubelet reports: reason: OOMKilled
    │
    ▼
Container restarts (per restartPolicy)
```

## QoS Classes and cgroup Structure

Kubernetes assigns QoS classes based on resource settings, which determines cgroup placement:

| QoS Class | Condition | cgroup Path | OOM Score |
|-----------|-----------|-------------|:---------:|
| Guaranteed | All containers have requests == limits (CPU + memory) | `/kubepods/pod<uid>/` | -997 (last to be killed) |
| Burstable | At least one container has requests != limits | `/kubepods/burstable/pod<uid>/` | 2-999 (scaled by usage) |
| BestEffort | No requests or limits set at all | `/kubepods/besteffort/pod<uid>/` | 1000 (first to be killed) |

```bash
# Check a pod's QoS class:
kubectl get pod <name> -o jsonpath='{.status.qosClass}'
```

### cgroup Hierarchy on the Node

```bash
# v1 hierarchy:
/sys/fs/cgroup/cpu/kubepods/
├── pod<uid-guaranteed>/           # Guaranteed pods at top level
│   └── <container-id>/
├── burstable/
│   ├── pod<uid>/                  # Burstable pods
│   │   └── <container-id>/
│   └── pod<uid>/
└── besteffort/
    └── pod<uid>/                  # BestEffort pods
        └── <container-id>/

# v2 hierarchy:
/sys/fs/cgroup/kubepods.slice/
├── kubepods-pod<uid>.slice/                    # Guaranteed
│   └── cri-containerd-<id>.scope
├── kubepods-burstable.slice/
│   └── kubepods-burstable-pod<uid>.slice/      # Burstable
│       └── cri-containerd-<id>.scope
└── kubepods-besteffort.slice/
    └── kubepods-besteffort-pod<uid>.slice/     # BestEffort
        └── cri-containerd-<id>.scope
```

## Kubelet Reserved Resources

The kubelet reserves resources for itself and the OS, keeping them out of the pod cgroup:

```
Node Total (e.g., 4 CPU, 16Gi memory)
    │
    ├── kube-reserved: resources for kubelet + container runtime
    │   (default varies, e.g., cpu=100m, memory=256Mi)
    │
    ├── system-reserved: resources for OS (sshd, journald, etc.)
    │   (e.g., cpu=100m, memory=256Mi)
    │
    ├── eviction-threshold: buffer before eviction (memory.available < 100Mi)
    │
    └── allocatable: what's left for pods
        = total - kube-reserved - system-reserved - eviction-threshold
```

```bash
# See node allocatable:
kubectl describe node <name> | grep -A 6 "Allocatable"

# Kubelet cgroup enforcement:
# --kube-reserved-cgroup=/kubelet.slice
# --system-reserved-cgroup=/system.slice
# --enforce-node-allocatable=pods,kube-reserved,system-reserved
```

## Ephemeral Storage

Ephemeral storage limits (`ephemeral-storage`) don't use cgroups. The kubelet polls disk usage and evicts pods exceeding limits:

```yaml
resources:
  limits:
    ephemeral-storage: 1Gi
```

The kubelet checks `/var/lib/kubelet/pods/<uid>/` size periodically and evicts if over limit.

## PID Limits

```yaml
# Not directly set via pod spec, but kubelet enforces:
# --pod-max-pids=100 (limits PIDs per pod cgroup)
```

```bash
# cgroup v1:
cat /sys/fs/cgroup/pids/kubepods/.../pids.max

# cgroup v2:
cat /sys/fs/cgroup/kubepods.slice/.../pids.max
```

## Inspecting cgroups on a Node

```bash
# Find a container's cgroup path:
crictl inspect <container-id> | jq '.info.runtimeSpec.linux.cgroupsPath'
# e.g., "kubepods-burstable-pod<uid>.slice:cri-containerd:<container-id>"

# v1 — check CPU settings:
CGROUP_PATH=/sys/fs/cgroup/cpu/kubepods/burstable/pod<uid>/<container-id>
cat $CGROUP_PATH/cpu.shares
cat $CGROUP_PATH/cpu.cfs_quota_us
cat $CGROUP_PATH/cpu.cfs_period_us
cat $CGROUP_PATH/cpu.stat

# v1 — check memory settings:
CGROUP_PATH=/sys/fs/cgroup/memory/kubepods/burstable/pod<uid>/<container-id>
cat $CGROUP_PATH/memory.limit_in_bytes
cat $CGROUP_PATH/memory.usage_in_bytes
cat $CGROUP_PATH/memory.stat

# v2 — all in one path:
CGROUP_PATH=/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod<uid>.slice/cri-containerd-<id>.scope
cat $CGROUP_PATH/cpu.max
cat $CGROUP_PATH/cpu.weight
cat $CGROUP_PATH/cpu.stat
cat $CGROUP_PATH/memory.max
cat $CGROUP_PATH/memory.current
cat $CGROUP_PATH/memory.events
```

## Summary Table

| K8s Setting | cgroup v1 File | cgroup v2 File | Behavior |
|-------------|---------------|---------------|----------|
| `requests.cpu` | `cpu.shares` | `cpu.weight` | Proportional weight (soft, only under contention) |
| `limits.cpu` | `cpu.cfs_quota_us` | `cpu.max` | Hard cap per period (causes throttling) |
| `requests.memory` | (none) | (none) | Scheduling only (no enforcement) |
| `limits.memory` | `memory.limit_in_bytes` | `memory.max` | Hard cap (OOMKill on exceed) |
| Pod PID limit | `pids.max` | `pids.max` | Max processes per pod |

## Quick Reference

```bash
# Check cgroup version:
stat -fc %T /sys/fs/cgroup/  # cgroup2fs = v2, tmpfs = v1

# CPU request → proportional weight (soft):
# v1: cpu.shares = millicores * 1024 / 1000
# v2: cpu.weight = millicores * 100 / 1000

# CPU limit → hard cap (throttling):
# v1: cpu.cfs_quota_us = millicores * 100 (period = 100000us)
# v2: cpu.max = "<quota> <period>"

# Memory limit → hard cap (OOMKill):
# v1: memory.limit_in_bytes = bytes
# v2: memory.max = bytes

# Memory request → scheduling only (no cgroup setting)

# Check throttling:
# v1: cat cpu.stat | grep throttled
# v2: cat cpu.stat | grep throttled_usec

# Check OOM events:
# v1: cat memory.oom_control
# v2: cat memory.events | grep oom_kill

# QoS class determines cgroup placement:
# Guaranteed: /kubepods/pod<uid>/
# Burstable:  /kubepods/burstable/pod<uid>/
# BestEffort: /kubepods/besteffort/pod<uid>/

# Key insight: CPU limits cause throttling EVEN when node has free CPU
# (CFS quota is absolute, not relative to available capacity)
```
