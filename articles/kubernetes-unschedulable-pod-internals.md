# What Happens When a Pod is Unschedulable

The kube-scheduler's internal pipeline — filtering, scoring, preemption decisions, and what happens when no node can run your pod.

## High-Level Flow

```
Pod created (spec.nodeName empty)
        │
        ▼
┌───────────────────┐     ┌──────────────┐     ┌──────────────────┐
│  Scheduling Queue │────▶│   Filtering  │────▶│    Scoring       │
│  (priority-sorted)│     │ (eliminate   │     │  (rank remaining │
│                   │     │  unfit nodes)│     │   nodes)         │
└───────────────────┘     └──────────────┘     └──────────────────┘
                                │                       │
                                ▼                       ▼
                          No nodes pass?          Best node found
                                │                       │
                                ▼                       ▼
                    ┌──────────────────┐        ┌──────────────┐
                    │   Preemption     │        │   Binding    │
                    │ (evict lower-    │        │ (assign pod  │
                    │  priority pods?) │        │  to node)    │
                    └──────────────────┘        └──────────────┘
                          │
                    No victims found?
                          │
                          ▼
                    ┌──────────────────┐
                    │  Unschedulable   │
                    │  (pod stays in   │
                    │   Pending state) │
                    └──────────────────┘
```

## The Scheduling Queue

The scheduler maintains a priority queue of unscheduled pods:

```
┌────────────────────────────────────────────────────────┐
│  Scheduling Queue                                      │
│                                                        │
│  ActiveQ (ready to schedule, sorted by priority):      │
│    1. [Priority 1000000] system-critical pod           │
│    2. [Priority 100]     production app pod            │
│    3. [Priority 0]       default pod                   │
│                                                        │
│  BackoffQ (recently failed, waiting for retry):        │
│    - pod-x (retry in 1s)                               │
│    - pod-y (retry in 2s)                               │
│                                                        │
│  UnschedulableQ (no viable node, waiting for change):  │
│    - pod-z (waiting for node conditions to change)     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

When a pod fails scheduling:
1. First failure → moved to **BackoffQ** (exponential backoff: 1s, 2s, 4s, ... up to 10s)
2. After backoff → retried from ActiveQ
3. If still unschedulable and no cluster changes → moved to **UnschedulableQ**
4. When cluster state changes (new node, pod deleted, etc.) → moved back to ActiveQ

## Phase 1: Filtering (Predicates)

Filtering eliminates nodes that cannot run the pod. Each filter plugin returns "fits" or "doesn't fit":

```
All Nodes: [node-1, node-2, node-3, node-4, node-5]
    │
    ▼ NodeResourcesFit
Remaining: [node-1, node-2, node-3, node-5]  (node-4 has insufficient CPU)
    │
    ▼ NodeAffinity
Remaining: [node-1, node-2, node-5]  (node-3 doesn't match affinity)
    │
    ▼ TaintToleration
Remaining: [node-1, node-5]  (node-2 tainted, pod doesn't tolerate)
    │
    ▼ PodTopologySpread
Remaining: [node-5]  (node-1 violates topology spread)
    │
    ▼
Feasible nodes: [node-5]
```

### Filter Plugins (Order of Evaluation)

| Plugin | What It Checks |
|--------|---------------|
| `NodeUnschedulable` | Node is NOT cordoned (`spec.unschedulable: false`) |
| `NodeName` | Pod's `spec.nodeName` matches (if set) |
| `TaintToleration` | Pod tolerates all node taints |
| `NodeAffinity` | Pod's `nodeAffinity` rules match node labels |
| `NodeResourcesFit` | Node has enough allocatable CPU, memory, ephemeral storage |
| `NodePorts` | Requested `hostPort` is available on the node |
| `PodTopologySpread` | Placing pod doesn't violate `topologySpreadConstraints` |
| `InterPodAffinity` | Pod affinity/anti-affinity rules satisfied |
| `VolumeBinding` | Required PVs/PVCs can be satisfied on this node (zone, node affinity) |
| `VolumeRestrictions` | No volume conflicts (e.g., ReadWriteOnce already mounted) |
| `NodeVolumeLimits` | Node hasn't hit max volume attachments (e.g., 25 EBS volumes) |

If ALL nodes are filtered out → the pod is **unschedulable**.

## Phase 2: Scoring (Priorities)

If multiple nodes pass filtering, scoring ranks them (0-100 per plugin, weighted):

```
Feasible nodes: [node-1, node-3, node-5]

Scoring:
  NodeResourcesBalancedAllocation:
    node-1: 70  node-3: 85  node-5: 60
  InterPodAffinity:
    node-1: 50  node-3: 80  node-5: 40
  ImageLocality:
    node-1: 30  node-3: 10  node-5: 90  (image already cached)
  NodeAffinity (preferred):
    node-1: 0   node-3: 100 node-5: 0

Weighted total (example weights):
    node-1: 150  node-3: 275  node-5: 190

Winner: node-3 (highest score)
```

### Scoring Plugins

| Plugin | What It Favors |
|--------|---------------|
| `NodeResourcesBalancedAllocation` | Nodes with balanced CPU/memory usage |
| `NodeResourcesFit` (LeastAllocated) | Nodes with most free resources |
| `NodeResourcesFit` (MostAllocated) | Nodes with least free resources (bin-packing) |
| `InterPodAffinity` | Nodes satisfying preferred pod affinity |
| `NodeAffinity` | Nodes matching `preferredDuringScheduling` rules |
| `TaintToleration` | Nodes with fewer taints (less restrictive) |
| `ImageLocality` | Nodes that already have the container image cached |
| `PodTopologySpread` | Nodes that best satisfy topology spread preferences |

## Phase 3: Preemption

If no node passes filtering, the scheduler considers preemption — evicting lower-priority pods to make room:

```
┌────────────────────────────────────────────────────────────────┐
│  Preemption Decision                                           │
│                                                                │
│  1. Pending pod has PriorityClass with priority > 0            │
│  2. For each node, find pods with LOWER priority that          │
│     could be evicted to make room                              │
│  3. Check if evicting those pods would satisfy the pending     │
│     pod's resource requirements                                │
│  4. Check PodDisruptionBudgets — can we evict without          │
│     violating PDBs?                                            │
│  5. Choose the node with the fewest/lowest-priority victims    │
│  6. Evict victim pods (graceful termination)                   │
│  7. Set nominatedNodeName on pending pod                       │
│  8. Retry scheduling (pod may still need to wait for victims   │
│     to fully terminate)                                        │
└────────────────────────────────────────────────────────────────┘
```

### Preemption Requirements

- The pending pod must have **higher priority** than potential victims
- PodDisruptionBudgets are respected (victims with PDB protection may not be evicted)
- Pods with `preemptionPolicy: Never` cannot preempt (even with high priority)
- Pods being preempted get graceful termination (not instant kill)

### nominatedNodeName

When preemption is decided, the scheduler sets `status.nominatedNodeName` on the pending pod:

```yaml
status:
  nominatedNodeName: node-3   # "I plan to schedule here after victims are evicted"
  conditions:
  - type: PodScheduled
    status: "False"
    reason: Unschedulable
```

This is a **hint**, not a guarantee. Other higher-priority pods could take the spot.

## The Unschedulable State

When a pod cannot be scheduled (no nodes pass filtering, no preemption possible):

```yaml
status:
  phase: Pending
  conditions:
  - type: PodScheduled
    status: "False"
    reason: Unschedulable
    message: "0/5 nodes are available: 2 Insufficient cpu,
              1 node(s) had taint {dedicated=gpu:NoSchedule},
              2 node(s) didn't match Pod's node affinity/selector."
```

The message tells you exactly which filter plugins rejected which nodes.

## Common Unschedulable Reasons

| Message Pattern | Meaning | Fix |
|-----------------|---------|-----|
| `Insufficient cpu` | Requested CPU > node's allocatable - already allocated | Scale nodes, reduce requests, or evict other pods |
| `Insufficient memory` | Same for memory | Same |
| `node(s) had taint ... NoSchedule` | Node is tainted, pod lacks toleration | Add toleration or remove taint |
| `node(s) didn't match Pod's node affinity/selector` | nodeAffinity or nodeSelector doesn't match any node | Fix labels or relax affinity rules |
| `node(s) didn't satisfy existing pod anti-affinity` | Anti-affinity blocks co-location | Add more nodes or relax anti-affinity |
| `pod topology spread constraints not satisfiable` | Can't place without violating `whenUnsatisfiable: DoNotSchedule` | Use `ScheduleAnyway` or add nodes in needed zones |
| `persistentvolumeclaim not found` | PVC doesn't exist or not bound | Create PVC or check StorageClass |
| `volume node affinity conflict` | PV is in a different zone than available nodes | Use `WaitForFirstConsumer` binding mode |
| `exceeded max volume count` | Node hit EBS/disk attachment limit | Schedule on different node or clean up unused PVCs |

## Scheduler Retries and Backoff

```
┌────────────────────────────────────────────────────────────────┐
│  Retry Behavior                                                │
│                                                                │
│  Attempt 1: filtering fails → pod goes to BackoffQ (1s)        │
│  Attempt 2: filtering fails → BackoffQ (2s)                    │
│  Attempt 3: filtering fails → BackoffQ (4s)                    │
│  Attempt 4: filtering fails → BackoffQ (8s)                    │
│  Attempt 5: filtering fails → BackoffQ (10s, max)              │
│  ...                                                           │
│  No cluster changes → UnschedulableQ                           │
│                                                                │
│  Cluster state changes (node added, pod deleted, etc.):        │
│    → Pod moved back to ActiveQ for immediate retry             │
│                                                                │
│  Events that trigger retry:                                    │
│    - New node joins cluster                                    │
│    - Existing node becomes Ready                               │
│    - Pod deleted (frees resources)                             │
│    - Node labels/taints change                                 │
│    - PV becomes available                                      │
│    - ResourceQuota changes                                     │
└────────────────────────────────────────────────────────────────┘
```

## Scheduler Events

The scheduler emits events that explain its decisions:

```bash
# See scheduling events:
kubectl describe pod <pending-pod> | grep -A10 Events

# Events:
#   Type     Reason             Age   From               Message
#   ----     ------             ----  ----               -------
#   Warning  FailedScheduling   10s   default-scheduler  0/3 nodes are available:
#                                                        1 Insufficient cpu,
#                                                        2 node(s) had taint
#                                                        {dedicated=gpu:NoSchedule}
#   Normal   Preempted          5s    default-scheduler  Preempting pod
#                                                        default/low-pri-pod on node-2
#   Normal   Scheduled          1s    default-scheduler  Successfully assigned
#                                                        default/my-pod to node-2
```

## Scheduler Profiles and Extension Points

The scheduler is a plugin-based framework. Each phase is an extension point:

```
┌──────────────────────────────────────────────────────────────┐
│  Scheduling Framework Extension Points                       │
│                                                              │
│  QueueSort       → Order pods in the scheduling queue        │
│  PreFilter       → Pre-processing before filtering           │
│  Filter          → Eliminate unfit nodes                     │
│  PostFilter      → Called when no node passes filter         │
│                    (preemption happens here)                 │
│  PreScore        → Pre-processing before scoring             │
│  Score           → Rank feasible nodes                       │
│  NormalizeScore  → Normalize scores to 0-100                 │
│  Reserve         → Reserve resources (optimistic)            │
│  Permit          → Approve/deny/wait (gang scheduling)       │
│  PreBind         → Pre-binding actions (volume attachment)   │
│  Bind            → Assign pod to node (set spec.nodeName)    │
│  PostBind        → Cleanup after successful binding          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Resource Calculation

The scheduler compares pod requests against node allocatable resources:

```
Node allocatable:     4 CPU,    16Gi memory
Already allocated:    2.5 CPU,  10Gi memory
Available:            1.5 CPU,   6Gi memory

Pod requests:         2 CPU,     4Gi memory

Result: FAILS (2 CPU requested > 1.5 CPU available)
```

**Key point**: The scheduler uses **requests**, not limits. Limits are enforced at runtime by the kubelet/cgroup, but don't affect scheduling decisions.

```bash
# See node allocatable vs allocated:
kubectl describe node <name> | grep -A 10 "Allocated resources"

# Example output:
# Allocated resources:
#   Resource           Requests     Limits
#   --------           --------     ------
#   cpu                2500m (62%)  4000m (100%)
#   memory             10Gi (65%)   16Gi (100%)
#   ephemeral-storage  0 (0%)       0 (0%)
```

## Debugging Unschedulable Pods

```bash
# Check why pod is Pending:
kubectl describe pod <name> | grep -A5 "Events"
kubectl get pod <name> -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")]}'

# See what resources are available on each node:
kubectl describe nodes | grep -A10 "Allocated resources"

# Check node taints:
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints[*].key

# Check node labels (for affinity matching):
kubectl get nodes --show-labels

# Check if scheduler is running:
kubectl get pods -n kube-system -l component=kube-scheduler

# Scheduler logs (verbose):
kubectl logs -n kube-system -l component=kube-scheduler --tail=50

# Simulate scheduling (dry-run):
kubectl run test --image=nginx --dry-run=server -o yaml \
  --overrides='{"spec":{"containers":[{"name":"test","resources":{"requests":{"cpu":"4"}}}]}}'

# Find which pods are using the most resources on a node:
kubectl get pods --field-selector spec.nodeName=<node> -o json | \
  jq -r '.items[] | "\(.metadata.name)\t\(.spec.containers[].resources.requests.cpu // "0")\t\(.spec.containers[].resources.requests.memory // "0")"'
```

## Unblocking Unschedulable Pods

| Situation | Solution |
|-----------|----------|
| Insufficient resources | Add nodes (Cluster Autoscaler), reduce requests, or delete idle pods |
| Taint mismatch | Add toleration to pod, or remove taint from node |
| Node affinity mismatch | Fix labels on nodes, or relax affinity rules |
| Topology spread unsatisfiable | Add nodes in required zones, or use `whenUnsatisfiable: ScheduleAnyway` |
| Volume zone conflict | Use `WaitForFirstConsumer` StorageClass binding mode |
| Max volume limit | Spread PVCs across nodes, or use a different storage driver |
| PVC not bound | Create missing PV or fix StorageClass provisioner |

## Quick Reference

```bash
# Scheduling pipeline:
# Queue (priority-sorted) → Filter → Score → Bind
# If Filter fails all nodes → Preemption → or Unschedulable

# Check pod scheduling status:
kubectl get pod <name> -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")]}'

# See scheduling failure reason:
kubectl describe pod <name> | grep -A5 "Events"

# See node resources available:
kubectl describe node <name> | grep -A10 "Allocated resources"
kubectl top nodes

# Check scheduler health:
kubectl get pods -n kube-system -l component=kube-scheduler

# Key facts:
# - Scheduler uses REQUESTS (not limits) for placement decisions
# - Pods retry with exponential backoff (1s → 10s max)
# - Cluster state changes trigger immediate retry
# - Preemption respects PDBs and priority classes
# - nominatedNodeName is a hint, not a guarantee
```
