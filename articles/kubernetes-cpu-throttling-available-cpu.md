# Why Pods Get Throttled Even When Node Has Available CPU

CPU throttling happens at the container cgroup level, not at the node level. Pods hit their individual CPU limit and get throttled regardless of how much spare CPU the node has.

## The Confusing Reality

```
Node CPU Usage: 40% (60% available!)
Pod CPU Limit: 500m
Pod CPU Usage: 500m
Result: Pod is THROTTLED

Why? The node has plenty of CPU left!
```

## The Answer: CPU Limits Are Per-Container, Not Per-Node

CPU throttling happens at the **container cgroup level**, NOT at the node level.

### How It Actually Works

```
┌─────────────────────────────────────────┐
│ Node: 4 CPUs (4000m)                    │
│ Current Usage: 1600m (40%)              │
│ Available: 2400m (60%)                  │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │ Pod A (limit: 500m)              │   │
│  │ Trying to use: 600m              │   │
│  │ Kernel says: NO! Limit is 500m   │   │
│  │ Result: THROTTLED                │   │
│  └──────────────────────────────────┘   │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │ Pod B (no limit)                 │   │
│  │ Trying to use: 800m              │   │
│  │ Kernel says: OK! No limit set    │   │
│  │ Result: NOT THROTTLED            │   │
│  └──────────────────────────────────┘   │
│                                         │
│  Available CPU: 2400m (unused!)         │
└─────────────────────────────────────────┘
```

## The Throttling Mechanism

### Step-by-Step: What Happens

1. **Pod tries to use CPU**
   ```
   Pod wants to execute code → needs CPU time
   ```

2. **Kernel checks the container's cgroup**
   ```bash
   # Kernel looks at: /sys/fs/cgroup/kubepods.slice/.../cpu.max
   # Example: "50000 100000" means 50ms per 100ms = 500m (0.5 CPU)
   ```

3. **Kernel enforces the limit**
   ```
   IF (container_cpu_usage >= container_cpu_limit):
       THROTTLE (pause the container)
       DO NOT give it more CPU time
   ELSE:
       ALLOW (give it CPU time)
   ```

4. **Node CPU availability is IGNORED**
   ```
   The kernel doesn't care if the node has spare CPU!
   It only cares about the container's individual limit.
   ```

## Real Example

### Scenario: Node with 4 CPUs

```bash
# Node capacity
Total CPUs: 4000m

# Pods running:
- Defender: 900m (no limit)    → Running fine
- App-1: 500m (limit: 500m)   → THROTTLED
- App-2: 480m (limit: 500m)   → THROTTLED
- App-3: 490m (limit: 500m)   → THROTTLED

# Total usage: 2370m out of 4000m (59%)
# Available: 1630m (41%) - WASTED!
```

Each app is hitting its **individual 500m limit**, even though the node has 1630m available.

## The cgroup Configuration

### What Kubernetes Sets Up

When you define:

```yaml
resources:
  limits:
    cpu: 500m
```

Kubernetes writes to the container's cgroup:

**cgroup v2:**

```bash
# /sys/fs/cgroup/kubepods.slice/.../cpu.max
50000 100000
# Meaning: 50ms of CPU time per 100ms period = 500m
```

**cgroup v1:**

```bash
# /sys/fs/cgroup/cpu/kubepods/.../cpu.cfs_quota_us
50000
# /sys/fs/cgroup/cpu/kubepods/.../cpu.cfs_period_us
100000
```

### How the Kernel Enforces It

```c
// Simplified kernel logic (CFS scheduler)

every 100ms period:
    if (container_cpu_used >= 50ms):
        throttle_container()
        // Container is paused, cannot run
        // Even if node has spare CPU!
    else:
        allow_container_to_run()
```

## Visualizing the Problem

### Timeline of Throttling

```
Time: 0ms
├─ Container starts executing
├─ Uses 10ms of CPU ✓
├─ Uses 20ms of CPU ✓
├─ Uses 30ms of CPU ✓
├─ Uses 40ms of CPU ✓
├─ Uses 50ms of CPU ✓ (hit limit!)
├─ Tries to use 51ms ❌ THROTTLED
├─ Kernel: "You've used your 50ms quota"
├─ Container is PAUSED
├─ Liveness probe arrives ❌ TIMEOUT
└─ Pod is killed and restarted

Time: 100ms (new period)
├─ Quota resets to 0ms
├─ Container can run again
└─ Cycle repeats...
```

### What the Container Sees

```bash
# Inside the throttled container
$ time curl http://localhost:8080/health
# Normally: 10ms
# When throttled: 500ms-1000ms (timeout!)

# Why? The process is paused by the kernel
# It's not slow, it's literally frozen
```

## Why This Design?

### The Intent (Good)

CPU limits were designed to:
1. Prevent runaway processes from consuming all node CPU
2. Guarantee fair sharing among containers
3. Protect other workloads from noisy neighbors

### The Reality (Bad)

In practice:
1. Limits are too restrictive for bursty workloads
2. Throttling happens during normal operation
3. Available CPU goes unused
4. Performance degrades unnecessarily

## CPU Requests vs Limits

### CPU Requests (Use These)

```yaml
resources:
  requests:
    cpu: 500m
```

What it does:
- Kubernetes scheduler guarantees this much CPU
- Node must have 500m available to schedule pod
- Pod can use MORE than 500m if available
- No throttling based on request

How it works:
- Uses CPU shares (cgroup `cpu.weight`)
- Proportional sharing when node is busy
- No hard limit enforced

### CPU Limits (Avoid These)

```yaml
resources:
  limits:
    cpu: 500m
```

What it does:
- Hard cap at 500m
- Kernel throttles if exceeded
- Happens even with spare node CPU
- Causes probe timeouts and restarts

How it works:
- Uses CPU quota (cgroup `cpu.max`)
- Hard enforcement by kernel
- Container is paused when limit hit

## The Kubernetes Community Consensus

### Official Recommendation: Don't Use CPU Limits

From Kubernetes maintainers and industry experts:

- "For the love of God, stop using CPU limits on Kubernetes" — Robusta.dev
- "CPU limits are harmful and should be avoided" — Multiple Kubernetes SIG members

### Why?

1. **Throttling with spare capacity** — wastes resources
2. **Unpredictable performance** — same workload, different latency
3. **Probe failures** — causes unnecessary restarts
4. **No real benefit** — requests provide sufficient protection

## Comparison: With and Without Limits

### With CPU Limits (Problem)

```yaml
resources:
  requests:
    cpu: 500m
  limits:
    cpu: 500m
```

Result:
- Pod scheduled on node with 500m available ✓
- Pod tries to use 600m during traffic spike
- Kernel throttles at 500m ✗
- Liveness probe times out ✗
- Pod restarts ✗
- Node has 2000m available (wasted) ✗

### Without CPU Limits (Recommended)

```yaml
resources:
  requests:
    cpu: 500m
  # No CPU limit!
```

Result:
- Pod scheduled on node with 500m available ✓
- Pod tries to use 600m during traffic spike
- Kernel allows it (no limit) ✓
- Liveness probe succeeds ✓
- Pod stays running ✓
- Node CPU used efficiently ✓

## But What About Protection?

### Question: Won't pods consume all CPU without limits?

**Answer: No, because of CPU requests and the CFS scheduler.**

### How CPU Sharing Works Without Limits

```
Node: 4 CPUs (4000m)

Pods:
- Pod A: request 1000m, no limit
- Pod B: request 1000m, no limit
- Pod C: request 1000m, no limit
- Pod D: request 1000m, no limit

Total requests: 4000m (100% of node)
```

**When all pods want CPU:**
1. CFS scheduler uses CPU shares (weights)
2. Each pod gets proportional share based on request
3. Pod A gets ~25%, Pod B gets ~25%, etc.
4. Fair sharing without throttling

**When some pods are idle:**
1. Active pods can use spare CPU
2. No artificial limits
3. Better performance
4. No wasted resources

### The Math

```
CPU shares = cpu_request * 1024 / 1000

Pod A request: 1000m → 1024 shares
Pod B request: 500m  → 512 shares
Pod C request: 500m  → 512 shares

Total shares: 2048

When node is 100% busy:
- Pod A gets: 1024/2048 = 50% of CPU
- Pod B gets: 512/2048  = 25% of CPU
- Pod C gets: 512/2048  = 25% of CPU

When Pod A is idle:
- Pod B and C can use its share
- No throttling, just fair sharing
```

## Practical Example

### Current State (With Limits)

```
Node: 4 CPUs (4000m)
Defender: 900m (no limit)    - running fine
App-1: 500m (limit 500m)    - THROTTLED
App-2: 500m (limit 500m)    - THROTTLED
App-3: 500m (limit 500m)    - THROTTLED

Total: 2400m / 4000m (60% node usage)
Available: 1600m (WASTED)

Apps are throttled despite 40% node CPU available!
```

### Recommended State (Without Limits)

```
Node: 4 CPUs (4000m)
Defender: 900m (no limit)    - running fine
App-1: 700m (no limit)      - running fine
App-2: 650m (no limit)      - running fine
App-3: 680m (no limit)      - running fine

Total: 2930m / 4000m (73% node usage)
Available: 1070m

Apps use available CPU, no throttling, better performance!
```

## How to Fix It

### Step 1: Remove CPU Limits

```bash
kubectl edit deployment <app-name> -n <namespace>
```

Change from:

```yaml
resources:
  requests:
    cpu: 500m
  limits:
    cpu: 500m
    memory: 1Gi
```

To:

```yaml
resources:
  requests:
    cpu: 500m
  limits:
    memory: 1Gi    # Keep memory limit, remove CPU limit
```

### Step 2: Verify No Throttling

```bash
# On the node, check throttling after change
grep -r "nr_throttled" /sys/fs/cgroup/kubepods.slice/*/cpu.stat 2>/dev/null | grep -v " 0$"

# Should see much less or no throttling
```

### Step 3: Monitor Performance

```bash
# Watch CPU usage
kubectl top pods -n <namespace>

# Pods should now use more CPU when needed
# No more artificial 500m cap
```

## Detecting Throttling

### From Inside the Pod

```bash
# cgroup v2 (newer kernels, EKS AL2023)
kubectl exec <pod-name> -- cat /sys/fs/cgroup/cpu.stat

# cgroup v1 (older kernels, EKS AL2)
kubectl exec <pod-name> -- cat /sys/fs/cgroup/cpu/cpu.stat
```

Look for:
- `nr_throttled` — number of times the process was throttled
- `throttled_time` — total time (in nanoseconds) spent throttled

If `nr_throttled` is high or increasing, the container is being CPU-throttled.

### From the Node

```bash
# Find throttled containers (cgroup v2)
grep -r "nr_throttled" /sys/fs/cgroup/kubepods.slice/*/cpu.stat 2>/dev/null | grep -v " 0$"

# cgroup v1
find /sys/fs/cgroup/cpu/kubepods -name "cpu.stat" -exec grep -l "nr_throttled [1-9]" {} \;
```

### Per-Pod Check via Container ID

```bash
# Get the container ID
CONTAINER_ID=$(kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].containerID}' | cut -d'/' -f3)

# Check throttle stats on the node
cat /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/cri-containerd-${CONTAINER_ID}.scope/cpu.stat
```

### Using Prometheus Queries

If you have Prometheus with cAdvisor metrics:

```promql
# Throttling ratio per container (fraction of periods that were throttled)
rate(container_cpu_cfs_throttled_periods_total{namespace="your-ns", pod="your-pod"}[5m])
/
rate(container_cpu_cfs_periods_total{namespace="your-ns", pod="your-pod"}[5m])
```

A ratio above ~0.1 (10%) is worth investigating.

Other useful metrics:

```promql
# Total throttled seconds per second
rate(container_cpu_cfs_throttled_seconds_total[5m])

# CPU usage vs limit
rate(container_cpu_usage_seconds_total[5m])
```

### Quick Sanity Check with kubectl top

```bash
kubectl top pod <pod-name> -n <namespace>
```

Compare the CPU usage against the pod's `resources.limits.cpu`. If usage is consistently near the limit, throttling is likely occurring.

```bash
# See the configured limits
kubectl describe pod <pod-name> -n <namespace> | grep -A5 "Limits"
```

### Bursty Workloads

If you see high throttling but low **average** CPU usage, it means the workload is bursty — it needs short bursts above the limit but averages below it. This is the most common pattern where CPU limits cause harm without providing benefit.

## When to Keep CPU Limits

CPU limits are still valid in some cases:

| Scenario | Keep Limits? | Reason |
|----------|-------------|--------|
| Multi-tenant clusters | Yes | Hard isolation between tenants |
| Batch/HPC workloads that must not starve others | Yes | Prevent single job from dominating |
| Compliance requirements | Yes | Some standards require hard limits |
| Cost allocation per team | Yes | Accurate chargeback |
| Single-tenant, trusted workloads | No | Let workloads use available CPU |
| Microservices with bursty traffic | No | Bursts cause throttling |
| Latency-sensitive applications | No | Throttling adds unpredictable latency |

## Why Memory Limits Are Different

CPU is compressible (can be throttled). Memory is non-compressible (can only be killed). This is why the recommendation is: **remove CPU limits, keep memory limits**.

### Memory vs CPU Behavior

| Aspect | CPU | Memory |
|--------|-----|--------|
| Type | Compressible (time-sliced) | Non-compressible (allocated) |
| Sharing | Can be shared among processes | Cannot be shared or borrowed |
| Limit exceeded | Throttled (paused) | OOMKilled (terminated) |
| Impact of no limit | Pods compete fairly via requests | One pod can exhaust entire node |
| Recommendation | No limit | Always set limit |

### What Happens Without Memory Limits

```
Node: 16GB total memory
Pod A: No memory limit → keeps allocating...
10GB... 12GB... 14GB... 15GB...
↓
Node runs out of memory
↓
Kernel OOMKiller starts killing random pods
↓
Cluster instability — multiple pods affected
```

### What Happens With Memory Limits

```
Node: 16GB total memory
Pod A: 4GB memory limit → tries to allocate 4.1GB
↓
Kernel: "Pod A exceeded its limit"
↓
OOMKiller kills ONLY Pod A
↓
Other pods unaffected
↓
Pod A restarts with fresh memory
```

### Recommended Configuration

```yaml
resources:
  requests:
    cpu: 500m        # Scheduler guarantee — used for bin-packing
    memory: 2Gi      # Scheduler guarantee
  limits:
    # NO CPU LIMIT   # Allow bursting to available capacity
    memory: 4Gi      # Prevent memory exhaustion — always set this
```

This gives you Burstable QoS:
- Guaranteed minimum resources (requests)
- Can burst CPU when node has spare capacity
- Protected from memory exhaustion
- Medium eviction priority (better than BestEffort)

### Memory Limit Best Practices

```yaml
# Set memory limit higher than request (room for spikes)
resources:
  requests:
    memory: 2Gi      # Normal usage
  limits:
    memory: 4Gi      # Peak usage + buffer

# For DaemonSets (predictable usage)
resources:
  requests:
    memory: 512Mi
  limits:
    memory: 512Mi    # Or slightly higher
```

### Monitoring OOMKills

```bash
# Find OOMKilled pods
kubectl get pods -A -o json | jq -r '.items[] | select(.status.containerStatuses[]?.lastState.terminated.reason == "OOMKilled") | "\(.metadata.namespace)/\(.metadata.name)"'

# If you see OOMKills, increase memory limits — don't remove them
```

## Summary

| Question | Answer |
|----------|--------|
| Why throttled with available CPU? | CPU limits are enforced per-container by the kernel, not per-node |
| Does the kernel check node capacity? | No — it only checks the container's cgroup quota |
| What's the fix? | Remove CPU limits, keep CPU requests |
| Is it safe? | Yes — CFS scheduler handles fair sharing via requests |
| When to keep limits? | Multi-tenant clusters, compliance, cost allocation |

## References

- [Linux CFS Bandwidth Control](https://www.kernel.org/doc/Documentation/scheduler/sched-bwc.txt)
- [cgroup v2 CPU Controller](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#cpu)
- [Stop Using CPU Limits — Robusta.dev](https://home.robusta.dev/blog/stop-using-cpu-limits)
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
