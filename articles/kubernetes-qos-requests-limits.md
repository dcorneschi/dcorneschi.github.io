# Kubernetes QoS Classes — When Requests and Limits Matter

Pod Quality of Service (QoS) classes determine how Kubernetes treats your workloads under resource pressure. They control eviction order, OOM kill priority, and scheduling behavior. Understanding them is essential for running stable clusters.

## The Three QoS Classes

| QoS Class | Condition | Eviction Priority | OOM Kill Priority |
|-----------|-----------|-------------------|-------------------|
| **BestEffort** | No requests or limits set on any container | First to be evicted | First to be OOM-killed |
| **Burstable** | At least one container has a request or limit, but requests ≠ limits | Second | Second |
| **Guaranteed** | Every container has CPU and memory requests = limits | Last | Last |

Kubernetes assigns the QoS class automatically based on your resource spec. You cannot set it directly.

## How QoS Class Is Determined

```
For EVERY container in the Pod:
│
├── Has NO requests AND NO limits?
│   └─► ALL containers like this → QoS = BestEffort
│
├── Has requests = limits (for both CPU and memory)?
│   └─► ALL containers like this → QoS = Guaranteed
│
└── Anything else (partial requests, requests ≠ limits, only some containers have them)
    └─► QoS = Burstable
```

### Examples

**Guaranteed:**

```yaml
containers:
- name: app
  resources:
    requests:
      cpu: "500m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "256Mi"
```

All containers must have both CPU and memory requests equal to their limits. If you set only limits, Kubernetes auto-fills requests to match — so this also works:

```yaml
containers:
- name: app
  resources:
    limits:
      cpu: "500m"
      memory: "256Mi"
# requests are auto-set to match limits → Guaranteed
```

**Burstable:**

```yaml
containers:
- name: app
  resources:
    requests:
      cpu: "250m"
      memory: "128Mi"
    limits:
      cpu: "1000m"
      memory: "512Mi"
```

Requests ≠ limits → Burstable. Also Burstable if only one container in a multi-container pod has resources set.

**BestEffort:**

```yaml
containers:
- name: app
  image: nginx
  # no resources block at all
```

No requests, no limits, on any container → BestEffort.

## When Requests Matter

### Scheduling

Requests are the **only** thing the scheduler looks at when placing pods on nodes.

```
Node allocatable: 4 CPU, 8Gi memory

Pod A requests: 2 CPU, 4Gi     → Scheduled ✓ (2 CPU, 4Gi remaining)
Pod B requests: 1 CPU, 2Gi     → Scheduled ✓ (1 CPU, 2Gi remaining)
Pod C requests: 2 CPU, 3Gi     → NOT scheduled ✗ (only 1 CPU left)
```

- Limits are irrelevant for scheduling decisions
- A pod with no requests (BestEffort) can be scheduled anywhere that has any capacity at all
- Overcommit happens when the sum of limits > node capacity, which is fine as long as actual usage stays within bounds

### Node Pressure Eviction Order

When a node runs out of memory, kubelet evicts pods in this order:

1. **BestEffort** — no requests means nothing to compare against, evicted first
2. **Burstable exceeding requests** — sorted by how much they exceed their request (highest ratio first)
3. **Burstable within requests** — sorted by priority
4. **Guaranteed** — only evicted as a last resort

The request is the **safety line**. If your actual usage stays at or below your request, you survive longer under pressure.

### OOM Kill Order

When the kernel's OOM killer fires, it uses `oom_score_adj` values set by kubelet:

| QoS Class | oom_score_adj | Meaning |
|-----------|---------------|---------|
| BestEffort | 1000 | Always killed first |
| Burstable | 2–999 (scaled by request-to-limit ratio) | Middle ground |
| Guaranteed | -997 | Almost never killed by OOM |

For Burstable pods, the `oom_score_adj` is calculated based on the memory request relative to the node's memory. Lower requests (relative to the node) = higher score = killed sooner.

**Checking oom_score_adj on a node:**

```bash
# From inside the node (SSH or debug pod):

# Check oom_score_adj for a specific container's PID
crictl ps | grep <pod-name>
crictl inspect <container-id> | grep pid
cat /proc/<pid>/oom_score_adj

# Check the actual OOM score the kernel uses to decide who to kill (0-1000, higher = killed first)
cat /proc/<pid>/oom_score

# List all processes sorted by OOM score (highest risk first)
printf "%-6s %-6s %s\n" "PID" "SCORE" "COMMAND"; \
  for pid in /proc/[0-9]*; do \
    p=${pid##*/}; \
    score=$(cat $pid/oom_score 2>/dev/null) && \
    cmd=$(cat $pid/comm 2>/dev/null) && \
    printf "%-6s %-6s %s\n" "$p" "$score" "$cmd"; \
  done | sort -k2 -n -r | head -20
```

If you don't have direct node access:

```bash
# Launch a debug pod on a specific node
kubectl debug node/<node-name> -it --image=busybox

# Inside the debug pod (host filesystem is at /host):
chroot /host
crictl inspect <container-id> | grep pid
cat /proc/<pid>/oom_score_adj
```

The two files:
- `/proc/<pid>/oom_score_adj` — the adjustment value set by kubelet (-997 to 1000)
- `/proc/<pid>/oom_score` — the final computed score the kernel actually uses (0–1000, higher = killed first)

**How they relate:**

| File | Who sets it | What it means | Range |
|------|-------------|---------------|-------|
| `oom_score_adj` | kubelet (based on QoS class) | Static bias / priority hint | -1000 to 1000 |
| `oom_score` | kernel (computed dynamically) | Actual kill priority right now | 0 to 1000 |

The kernel calculates `oom_score` roughly as:

```
oom_score ≈ (process_memory_usage / total_memory) × 1000 + oom_score_adj
```

It combines **actual memory consumption** with the **static bias**. A Guaranteed pod (adj = -997) would need to consume nearly all node memory before its score gets high enough to be targeted. A BestEffort pod (adj = 1000) is essentially always at max score regardless of actual usage.

In practice: kubelet sets `oom_score_adj` once when the container starts, the kernel recalculates `oom_score` dynamically. When the OOM killer fires, the process with the highest `oom_score` at that moment is killed.

### CPU Throttling vs. Memory OOM

Resources behave very differently:

| Resource | Compressible? | What happens when exceeded |
|----------|---------------|---------------------------|
| **CPU** | Yes | Pod is **throttled** (slowed down), never killed |
| **Memory** | No | Pod is **OOM-killed** (terminated immediately) |

This is why memory requests/limits are critical for stability, while CPU limits are often debated.

## When Limits Matter

### Memory Limits — Hard Kill Boundary

If a container tries to allocate memory beyond its limit, the kernel OOM-kills it immediately:

```
Container memory limit: 512Mi
Container tries to use: 600Mi
Result: SIGKILL → container restart (if restartPolicy allows)
```

The pod status shows `OOMKilled` as the reason.

**Best practice for memory:** Always set memory requests equal to memory limits. If your limit is higher than your request, the pod can be OOM-killed even while below its limit — because the node ran out of memory and the kernel kills pods based on how much they exceed their *request*, not their limit.

```yaml
resources:
  requests:
    memory: "512Mi"   # set equal to limit
  limits:
    memory: "512Mi"   # the OOM kill boundary
```

### CPU Limits — Throttling

CPU limits use CFS (Completely Fair Scheduler) bandwidth control:

```
Container CPU limit: 1000m (1 core)
Container tries to use: 2000m (2 cores)
Result: Throttled to 1000m — runs slower but is NOT killed
```

CPU throttling is implemented as:
- CFS period: 100ms (default)
- CFS quota: limit × period (e.g., 1000m → 100ms quota per 100ms period)
- If the container exhausts its quota within a period, it's paused until the next period

### The CPU Limits Debate

Many teams choose to **not set CPU limits** while keeping CPU requests. This is the [recommended approach by Robusta](https://home.robusta.dev/blog/stop-using-cpu-limits) and endorsed by Tim Hockin, one of the original Kubernetes maintainers at Google.

The core argument: CPU is a **compressible, renewable** resource. If a pod doesn't use its allocated CPU, another pod can use it — but only if limits aren't preventing it. Setting CPU limits means leaving CPU cycles unused on the table even when no one else needs them.

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    # cpu: intentionally omitted — let pods burst when spare capacity is available
    memory: "256Mi"
```

**Why no CPU limits works:**
- CPU requests already guarantee a minimum allocation via CFS weights
- If Pod A requests 500m, it will always get at least 500m regardless of what other pods are doing
- Without limits, pods can burst above their request when spare CPU is available
- No one gets starved (requests protect you), and no CPU goes wasted

**The water analogy:** Two explorers with 2 liters of water total. With limits, each gets exactly 1 liter — if one only needs 0.5L, the other still can't drink the spare 0.5L and dies of thirst. With requests (no limits), one explorer has a guaranteed 1L reservation, and the other can drink whatever is left over. Everyone survives.

**When you might still want CPU limits:**
- Noisy neighbor isolation in multi-tenant clusters where fairness matters more than efficiency
- Regulated environments requiring strict resource accounting
- When you need Guaranteed QoS class (requires requests = limits)
- Latency-sensitive workloads where **consistent** performance matters more than **peak** performance

### The Counter-Argument: Why You Might Keep CPU Limits

[Daniel Nastacio argues](https://dnastacio.medium.com/why-you-should-keep-using-cpu-limits-on-kubernetes-60c4e50dfc61) that CPU limits have real benefits that the "stop using limits" advice overlooks:

**1. Resilience under throttling — probes are vulnerable**

When a container bursts above its request, all threads get throttled — including the ones handling liveness and readiness probes. A runaway thread consuming a disproportionate CPU share can starve the probe handler, causing:
- Readiness probe failures → traffic diverted away
- Liveness probe failures → container restarted

Developing with CPU limits from the start forces you to build containers that behave well under constrained CPU, rather than discovering the problem in production.

**2. CPU and memory usage are correlated**

CPU and memory are interlinked. A container that bursts to 10x its CPU request likely needs proportionally more memory too. But memory limits don't stretch the same way — so you get unbalanced resource usage that can lead to OOM kills.

Removing CPU limits opens the door for CPU to run unchecked while memory stays capped, which doesn't match how most applications actually work.

**3. Pod autoscalers do it better**

Instead of relying on CPU bursting (uncontrolled, unbalanced), use HPA/VPA to scale pods in a balanced way — adding both CPU and memory proportionally:
- HPA adds replicas → more aggregate CPU and memory
- VPA increases individual pod requests and limits → balanced growth

Autoscalers give you the extra capacity without the side effects of uncontrolled bursting.

**4. Guaranteed QoS requires limits**

Dropping CPU limits downgrades your pod from Guaranteed to Burstable QoS, making it eligible for eviction sooner (based on memory). It also makes the pod ineligible for benefits like the CPU Manager's Static policy (pinning CPUs to containers for low-latency workloads).

**5. Managed services require limits**

If you ever move workloads to managed container services (AWS ECS, IBM Code Engine, etc.), they require CPU limits for resource allocation and billing. Building without limits creates a hidden dependency on spare node CPU that breaks in different environments.

### The Balanced View

| Approach | Best For |
|----------|----------|
| No CPU limits (requests only) | Most stateless services, microservices with good autoscaling, environments where utilization efficiency matters most |
| CPU limits = requests (Guaranteed) | Latency-critical services, services with strict SLAs, workloads needing CPU Manager static policy |
| CPU limits > requests (Burstable) | Workloads with known burst patterns, when you want some bursting but with a ceiling |

### The Runtime Thread Pool Problem

[Hans Elias B. Josephsen points out](https://hansihe.com/blog/kubernetes-cpu-limits/) a subtle but impactful issue: many language runtimes use CPU limits (via cgroups) to determine how many worker threads to spawn.

**Runtimes that read cgroup CPU limits:**
- **Go** — uses CPU limit to derive default `GOMAXPROCS`
- **Erlang/Elixir (BEAM)** — uses CPU limit for default online scheduler count
- **JVM** — uses CPU limit for default thread pool sizes and GC threads (since Java 10+)

**The problem without limits:** Consider a container with 1 CPU request on a 48-core node. Without a CPU limit, these runtimes see 48 cores and spawn 48 worker threads. But the container only has 1 CPU worth of time. Result: 48 threads fighting for 1 core, causing excessive context switching — a silent "constant background drag" on performance that doesn't show up clearly in CPU graphs or profiles.

This gets worse when multiple containers on the same node all do this. You end up with hundreds of threads fighting for a few cores.

**Solutions:**

```yaml
# Option 1: Set a CPU limit (simplest)
resources:
  requests:
    cpu: "1000m"
  limits:
    cpu: "2000m"    # runtime sees 2 cores, spawns 2 threads

# Option 2: Override runtime defaults explicitly (no limit needed)
env:
- name: GOMAXPROCS
  value: "2"        # Go: set directly
- name: JAVA_OPTS
  value: "-XX:ActiveProcessorCount=2"  # JVM: override detected CPUs
```

For Go specifically, the [automaxprocs](https://github.com/uber-go/automaxprocs) library reads the cgroup CPU quota and sets `GOMAXPROCS` accordingly — useful even without explicit limits.

**The sweet spot:** Set worker thread count at or above CPU requests, but well below total node CPU count. If you drop CPU limits, you must manually configure runtime thread pools to avoid this trap.

The key insight from both sides: **accurate CPU requests are non-negotiable**. The debate is only about what happens above the request. Use tools like [KRR](https://github.com/robusta-dev/krr) or VPA recommendations to right-size your requests based on actual usage data.

## Practical Patterns

### Pattern 1: Critical Production Services → Guaranteed

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payment-service
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "2000m"
        memory: "4Gi"
      limits:
        cpu: "2000m"
        memory: "4Gi"
```

When to use:
- Latency-sensitive services (payment, auth)
- Services that must never be evicted under pressure
- When predictable performance matters more than efficiency

Trade-off: You're reserving resources even when idle. No bursting above the limit.

### Pattern 2: Standard Workloads → Burstable (with memory limit)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-server
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "250m"
        memory: "512Mi"
      limits:
        memory: "1Gi"
        # no CPU limit — allow bursting
```

When to use:
- Most web services and APIs
- Services with variable load patterns
- When you want scheduling guarantees but don't need eviction immunity

Trade-off: Can be throttled/evicted under pressure, but uses resources more efficiently.

### Pattern 3: Batch Jobs → Burstable (low requests, high limits)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-pipeline
spec:
  containers:
  - name: worker
    resources:
      requests:
        cpu: "100m"
        memory: "256Mi"
      limits:
        cpu: "4000m"
        memory: "8Gi"
```

When to use:
- Batch processing, CI/CD jobs
- Workloads that are OK being evicted and retried
- Short-lived burst workloads

Trade-off: Easy to schedule (low requests), can burst when available, but first in line for eviction.

### Pattern 4: Best Effort → Development/Testing Only

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
spec:
  containers:
  - name: debug
    image: busybox
    # no resources at all
```

When to use:
- Dev/debug pods
- One-off troubleshooting containers
- Never in production

Trade-off: First to die under any pressure. No scheduling guarantees.

## How It All Connects — Node Under Pressure

```
Node: 8Gi memory, currently at 7.9Gi usage
Kubelet eviction threshold: memory.available < 100Mi

EVICTION ORDER:
═══════════════

Step 1: Kill BestEffort pods
┌─────────────────────────────────────────────┐
│  debug-pod (BestEffort, using 50Mi)         │  ← killed first
│  test-runner (BestEffort, using 200Mi)      │  ← killed second
└─────────────────────────────────────────────┘
         │ Still under pressure?
         ▼
Step 2: Kill Burstable pods exceeding requests
┌─────────────────────────────────────────────┐
│  api-server (request: 512Mi, using: 900Mi)  │  ← over by 76%
│  worker (request: 256Mi, using: 400Mi)      │  ← over by 56%
└─────────────────────────────────────────────┘
         │ Still under pressure?
         ▼
Step 3: Kill Burstable pods within requests
┌─────────────────────────────────────────────┐
│  cache (request: 1Gi, using: 800Mi)         │  ← within request
└─────────────────────────────────────────────┘
         │ Still under pressure?
         ▼
Step 4: Kill Guaranteed pods (last resort)
┌─────────────────────────────────────────────┐
│  payment-service (Guaranteed, 4Gi)          │  ← only if desperate
└─────────────────────────────────────────────┘
```

## Common Mistakes

### 1. Setting requests too low to "fit more pods"

```yaml
# Dangerous: actual usage is ~500Mi but request is 64Mi
resources:
  requests:
    memory: "64Mi"
  limits:
    memory: "1Gi"
```

Problem: Scheduler packs too many pods on a node. Under real load, node runs out of memory and evicts pods aggressively. The pod is classified as Burstable and exceeding its request by a massive ratio — it's near the top of the eviction list.

### 2. No memory limit on a leaky application

```yaml
# Memory leak will consume the entire node
resources:
  requests:
    memory: "256Mi"
  # no limit → can grow indefinitely
```

Problem: A memory leak slowly consumes the node until kubelet starts evicting everything. Always set memory limits.

### 3. CPU limits causing latency spikes

```yaml
# Over-constrained CPU
resources:
  requests:
    cpu: "100m"
  limits:
    cpu: "100m"
```

Problem: Even brief CPU spikes (GC pauses, request bursts) hit the CFS throttle, causing tail latency spikes. For latency-sensitive services, either increase the limit or remove it entirely.

### 4. Mixing Guaranteed and BestEffort on the same node

Problem: BestEffort pods can spike memory usage, triggering eviction of pods you didn't expect. Use node affinity or taints to separate critical (Guaranteed) workloads from opportunistic (BestEffort) ones.

### 5. Setting only CPU requests without memory requests

```yaml
resources:
  requests:
    cpu: "500m"
  # no memory request → effectively BestEffort for memory
```

Problem: Pod is Burstable but has no memory request "safety line." Under memory pressure, it's treated almost like BestEffort for memory eviction purposes.

## Checking QoS Class

```bash
# Check a pod's QoS class
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'

# List all pods with their QoS class
kubectl get pods -o custom-columns=\
NAME:.metadata.name,\
QOS:.status.qosClass,\
CPU_REQ:.spec.containers[0].resources.requests.cpu,\
CPU_LIM:.spec.containers[0].resources.limits.cpu,\
MEM_REQ:.spec.containers[0].resources.requests.memory,\
MEM_LIM:.spec.containers[0].resources.limits.memory

# Find BestEffort pods (candidates for resource requests)
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.status.qosClass=="BestEffort") | "\(.metadata.namespace)/\(.metadata.name)"'

# Check actual resource usage vs requests
kubectl top pods --sort-by=memory
```

## LimitRange and ResourceQuota — Cluster-Level Defaults

If you want to prevent BestEffort pods from being created, use a `LimitRange`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-resources
  namespace: production
spec:
  limits:
  - type: Container
    default:          # default limits (applied if not specified)
      cpu: "1000m"
      memory: "512Mi"
    defaultRequest:   # default requests (applied if not specified)
      cpu: "100m"
      memory: "128Mi"
    min:              # minimum allowed
      cpu: "50m"
      memory: "64Mi"
    max:              # maximum allowed
      cpu: "4000m"
      memory: "8Gi"
```

With this in place, any pod created without resource specs gets the defaults applied — preventing accidental BestEffort pods.

## Quick Decision Guide

```
What QoS class should my workload use?
│
├── Is it a critical production service (payment, auth, database)?
│   └─► Guaranteed (requests = limits)
│
├── Is it a standard service with variable load?
│   └─► Burstable (set requests based on p50 usage, memory limit at p99)
│
├── Is it a batch job / CI pipeline?
│   └─► Burstable (low requests, high limits, use PriorityClass to control eviction)
│
└── Is it a dev/debug/throwaway pod?
    └─► BestEffort is fine (but never in production namespaces)
```

## Key Takeaways

1. **Always set memory requests and limits** — memory is incompressible, exceeding it means OOM kill
2. **Set CPU requests, but consider omitting CPU limits** — throttling often hurts more than it helps
3. **Requests are your scheduling guarantee and eviction protection** — set them close to actual usage
4. **Guaranteed QoS buys eviction immunity** — use it for workloads that must never be disrupted
5. **BestEffort has no guarantees whatsoever** — first to be evicted, first to be OOM-killed
6. **Use LimitRange to enforce defaults** — prevent accidental BestEffort pods in production namespaces
7. **Monitor actual usage vs. requests** — right-size with VPA or manual observation before setting values
