# Kubernetes Scheduling Deep Dive

How the kube-scheduler assigns pods to nodes — the full pipeline from Pending to Running, all scheduling methods, and tuning strategies.

## How Scheduling Works: The Full Pipeline

When a pod is created, it enters a `Pending` state and goes through a multi-stage pipeline before landing on a node:

```
Pod Created (Pending)
    │
    ▼
┌─────────────────────────┐
│  1. Scheduling Queue    │  Pods sorted by priority
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  2. Pre-Filter          │  Calculate resource needs, affinity terms, topology
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  3. Filter (Predicates  │  Remove nodes that CAN'T run the pod
└───────────┬─────────────┘
            │ (feasible nodes)
            ▼
┌─────────────────────────┐
│  4. Post-Filter         │  If no node passes → attempt preemption
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  5. Pre-Score           │  Prepare data for scoring
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  6. Score (Priorities)  │  Rank feasible nodes (best fit)
└───────────┬─────────────┘
            │ (highest score)
            ▼
┌─────────────────────────┐
│  7. Reserve             │  Temporarily claim resources
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  8. Permit              │  Approve, reject, or delay
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  9. Pre-Bind            │  Final prep (volume binding)
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  10. Bind               │  Write nodeName to pod spec in etcd
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  11. Post-Bind          │  Cleanup, logging, monitoring
└─────────────────────────┘
            │
            ▼
    Pod Running on Node
```

### Key Points

- The scheduler processes **one pod at a time** from the active queue
- Scheduling decisions are based on **resource requests**, not actual utilization
- If no node passes filtering, the pod stays `Pending` (or preemption is attempted)
- The entire process is driven by **scheduler plugins** at each extension point

## The Scheduling Queue

Before evaluation, pods enter an internal queue with three levels:

| Queue | Contains | When Retried |
|-------|----------|--------------|
| Active | Pods ready to be scheduled | Immediately (next cycle) |
| Backoff | Pods that recently failed scheduling | After exponential backoff |
| Unschedulable | Pods that can't be scheduled given current state | When cluster conditions change (new node, pod deleted, label updated) |

Higher-priority pods are evaluated first within the active queue.

## Filtering: Which Nodes CAN Run the Pod

The filter phase removes nodes that fail mandatory requirements. Common filter plugins:

| Plugin | What It Checks |
|--------|---------------|
| `NodeResourcesFit` | Node has enough requested CPU, memory, and extended resources |
| `NodeAffinity` | Node labels match required node affinity/nodeSelector |
| `TaintToleration` | Pod tolerates node taints |
| `VolumeBinding` | Required storage volumes can be attached |
| `InterPodAffinity` | Pod affinity/anti-affinity requirements are met |
| `PodTopologySpread` | Topology spread constraints are satisfiable |
| `NodePorts` | Requested host ports are available |
| `NodeUnschedulable` | Node is not cordoned |

> Scheduling uses **resource requests** — not current utilization. A node with 90% actual CPU usage but available requested capacity can still receive new pods. A node with 10% actual usage but fully committed requests cannot.

## Scoring: Which Node Is BEST

Once feasible nodes are identified, scoring ranks them. Common scoring plugins:

| Plugin | What It Favors |
|--------|---------------|
| `ImageLocality` | Nodes that already have the container image cached |
| `NodeResourcesFit` | Nodes with the most (or most balanced) available resources |
| `NodeResourcesBalancedAllocation` | Balanced CPU/memory utilization |
| `NodeAffinity` | Nodes matching preferred affinity rules |
| `InterPodAffinity` | Nodes satisfying preferred pod affinity |
| `PodTopologySpread` | Even distribution across topology domains |
| `TaintToleration` | Fewer taints = higher score |

Scores are normalized (0-100), multiplied by plugin weights, and summed. The node with the highest total wins.

### How Scoring Works: The Math

```
Final Score = ∑ (Plugin Score × Plugin Weight)
```

**NodeResourcesFit:**
```
Score = (Available CPU + Available Memory) / (Total CPU + Total Memory) × 100
```

**ImageLocality:**
```
Score = (Image size already present / Total image size) × 100
```

**BalancedResourceAllocation:**
```
Score = 100 - |CPU Usage% - Memory Usage%| × 10
```

Example:
- Node A: 60% CPU, 65% Memory → Score: 95 (well balanced)
- Node B: 80% CPU, 40% Memory → Score: 60 (imbalanced)
- Node A wins on this plugin

If multiple nodes have the same final score, the scheduler picks one randomly.

### Performance Optimization: percentageOfNodesToScore

For large clusters, the scheduler doesn't evaluate every node:

| Cluster Size | Default % Evaluated |
|:------------:|:-------------------:|
| < 100 nodes | 50% |
| 100-5000 nodes | Scales linearly from 50% to 10% |
| > 5000 nodes | 5% |

The scheduler iterates nodes in round-robin across zones and stops once it finds enough feasible nodes. You can override this in the scheduler config:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
percentageOfNodesToScore: 30
```

### Custom Scoring Configuration

Customize which plugins are active and their weights:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: custom-scheduler
    pluginConfig:
      - name: NodeResourcesFit
        args:
          scoringStrategy:
            type: LeastAllocated
            resources:
              - name: cpu
                weight: 3    # CPU 3x more important than memory
              - name: memory
                weight: 1
```

Scoring strategies:
- **LeastAllocated** — prefer nodes with most free resources (spread)
- **MostAllocated** — prefer nodes with least free resources (bin-pack)
- **RequestedToCapacityRatio** — custom curve

### The Binding Cycle

After scoring selects a node:

1. **Reserve** — temporarily claim resources on the selected node
2. **Permit** — approve or delay (for gang scheduling, etc.)
3. **Pre-Bind** — volume binding, storage prep
4. **Bind** — write `spec.nodeName` to the pod object in etcd
5. **Post-Bind** — cleanup, logging

If binding fails (e.g., another pod claimed the resources), the scheduler falls back to the next highest-scoring node.

### Topology Keys Reference

| Topology Key | Scope | Use Case |
|--------------|-------|----------|
| `kubernetes.io/hostname` | Individual nodes | Spread across different nodes |
| `topology.kubernetes.io/zone` | Availability zones | Distribute across AZs |
| `topology.kubernetes.io/region` | Geographic regions | Multi-region deployments |
| `node.kubernetes.io/instance-type` | Instance types | Hardware-specific placement |

## Preemption: When No Node Is Available

If filtering produces zero feasible nodes and the pending pod has a PriorityClass, the scheduler can **preempt** lower-priority pods:

1. Scheduler identifies nodes where evicting low-priority pods would make room
2. Lower-priority pods are marked for termination
3. After their graceful shutdown, the high-priority pod is scheduled

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-workload
value: 100000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "For critical production services"
---
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
spec:
  priorityClassName: critical-workload
  containers:
    - name: app
      image: critical-app:latest
```

### Priority Value Guidelines

| Range | Use Case |
|-------|----------|
| 1,000,000+ | System-critical (kube-system pods) |
| 100,000 - 999,999 | Production-critical |
| 10,000 - 99,999 | Production standard |
| 1,000 - 9,999 | Staging / non-critical |
| 0 - 999 | Development / batch |

> Preemption does not guarantee immediate scheduling — evicted pods need time to terminate, and conditions may change.

## All Scheduling Methods

### 1. nodeSelector

Simplest constraint — pods run only on nodes with matching labels:

```yaml
spec:
  nodeSelector:
    disktype: ssd
    gpu: "true"
```

### 2. Node Affinity

Flexible label matching with required and preferred rules:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["us-east-1a", "us-east-1b"]
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values: ["m5.xlarge", "m5.2xlarge"]
```

### 3. Pod Affinity and Anti-Affinity

Place pods near or away from other pods:

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: cache
          topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: web
            topologyKey: kubernetes.io/hostname
```

### 4. Taints and Tolerations

Taints repel pods; tolerations allow specific pods through:

```sh
# Taint a node
kubectl taint nodes worker-1 gpu-dedicated=true:NoSchedule
```

```yaml
spec:
  tolerations:
    - key: "gpu-dedicated"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  nodeSelector:
    gpu: "true"
```

| Effect | Behavior |
|--------|----------|
| `NoSchedule` | New pods without toleration cannot be scheduled |
| `PreferNoSchedule` | Scheduler avoids the node but may use it if necessary |
| `NoExecute` | Existing non-tolerating pods are evicted |

### 5. Topology Spread Constraints

Distribute pods evenly across failure domains:

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: web
```

| Field | Meaning |
|-------|---------|
| `maxSkew` | Maximum allowed difference in pod count between domains |
| `topologyKey` | The node label defining the domain (zone, node, region) |
| `whenUnsatisfiable` | `DoNotSchedule` (hard) or `ScheduleAnyway` (soft) |
| `labelSelector` | Which pods count toward the skew calculation |

### 6. Pod Priority and PriorityClasses

Higher-priority pods are scheduled first and can preempt lower-priority pods:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 50000
globalDefault: false
---
spec:
  priorityClassName: high-priority
```

### 7. nodeName (Direct Assignment)

Bypasses the scheduler entirely — use only for debugging:

```yaml
spec:
  nodeName: worker-3
```

### 8. Custom Schedulers

Run your own scheduler alongside `kube-scheduler`:

```yaml
spec:
  schedulerName: my-custom-scheduler
```

## Bin Packing vs Spreading

Two opposing placement strategies:

| Strategy | Behavior | Best For |
|----------|----------|----------|
| **Bin Packing** | Pack pods onto as few nodes as possible | Cost savings, dev/staging, batch jobs |
| **Spreading** | Distribute pods across nodes/zones | High availability, production |

Kubernetes defaults lean toward spreading (balanced allocation scoring), but you can influence this:

- **Bin packing**: Use `MostAllocated` scoring strategy in scheduler config, or let Karpenter consolidate
- **Spreading**: Use topology spread constraints and pod anti-affinity

In practice, most production clusters use **both**: spread critical workloads, pack non-critical ones.

## QoS Classes and Eviction Priority

Resource requests and limits determine a pod's QoS class, which affects eviction order under memory pressure:

| QoS Class | Definition | Eviction Priority |
|-----------|-----------|:-----------------:|
| **Guaranteed** | requests = limits for all containers | Last (lowest) |
| **Burstable** | requests < limits (or partially set) | Middle |
| **BestEffort** | No requests or limits set | First (highest) |

```yaml
# Guaranteed (requests == limits)
resources:
  requests:
    memory: "512Mi"
    cpu: "1"
  limits:
    memory: "512Mi"
    cpu: "1"

# Burstable (requests < limits)
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "1"

# BestEffort (nothing set)
resources: {}
```

> Under memory pressure, the kubelet evicts BestEffort pods first, then Burstable, then Guaranteed. Set at least requests to avoid being first in line.

## Topology Manager Policies (NUMA Awareness)

For latency-sensitive workloads, the kubelet's Topology Manager ensures resources come from the same NUMA node:

| Policy | Behavior |
|--------|----------|
| `none` | No topology alignment (default) |
| `best-effort` | Attempts alignment, doesn't guarantee |
| `restricted` | Rejects pod if alignment isn't possible |
| `single-numa-node` | All resources from a single NUMA node |

Configure in kubelet:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
topologyManagerPolicy: "single-numa-node"
```

Useful for high-performance computing, databases, and network-intensive workloads where cross-NUMA latency matters.

## Pod Disruption Budgets (PDBs)

PDBs protect against too many pods being disrupted simultaneously during voluntary operations (node drain, upgrades, autoscaler scale-down):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2          # At least 2 pods must always be running
  # OR: maxUnavailable: 1  # At most 1 pod can be down at a time
  selector:
    matchLabels:
      app: web
```

PDBs are respected by:
- `kubectl drain`
- Cluster Autoscaler / Karpenter (node scale-down)
- EKS managed node group rolling updates
- Preemption (scheduler won't preempt if it violates a PDB)

PDBs are NOT respected by:
- `kubectl delete pod` (immediate deletion)
- Node crashes (involuntary disruption)
- `--force` flag on EKS node group upgrades

## Debugging Scheduling Failures

### Why Is My Pod Pending?

```sh
# First check: describe the pod
kubectl describe pod <pod-name> | grep -A 10 "Events"

# Common messages:
# "0/5 nodes are available: 5 node(s) didn't match Pod's node affinity/selector"
# "0/5 nodes are available: 2 node(s) had untolerated taint"
# "0/5 nodes are available: 3 Insufficient cpu, 2 Insufficient memory"
# "0/5 nodes are available: 5 node(s) didn't satisfy pod topology spread constraints"
```

### Check Available Capacity

```sh
# Node allocatable vs requested
kubectl describe nodes | grep -A 5 "Allocated resources"

# Top nodes (actual usage)
kubectl top nodes

# Find nodes with available capacity
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
CPU_ALLOC:.status.allocatable.cpu,\
MEM_ALLOC:.status.allocatable.memory
```

### Check Scheduler Logs

```sh
# kube-scheduler logs (self-managed)
kubectl logs -n kube-system -l component=kube-scheduler

# EKS: enable scheduler logs in CloudWatch
aws eks update-cluster-config --name <cluster> \
  --logging '{"clusterLogging":[{"types":["scheduler"],"enabled":true}]}'
```

### Common Causes of Scheduling Failures

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `Insufficient cpu` | No node has enough unallocated CPU requests | Scale up, add nodes, or reduce requests |
| `didn't match node affinity/selector` | No node has the required labels | Label nodes or relax the selector |
| `had untolerated taint` | Node is tainted, pod doesn't tolerate | Add toleration or use a different node |
| `didn't satisfy topology spread` | Can't maintain even distribution | Add nodes in underserved topology domains |
| `persistentvolumeclaim not bound` | PVC can't find a matching PV | Check StorageClass, zone affinity |
| `preemption` events | High-priority pod evicting yours | Increase your pod's priority or add capacity |

## Scheduler Limitations

- **Single-threaded per pod**: The scheduler processes one pod at a time. At thousands of pending pods, this can create latency.
- **Request-based, not utilization-based**: Decisions use declared requests, not actual CPU/memory usage. Over-requested pods waste capacity.
- **No automatic rescheduling**: Once a pod is bound to a node, it stays there even if a better node becomes available later. Only eviction/preemption moves it.
- **Scalability**: Default scheduler can handle ~5,000 nodes well. Beyond that, tune `percentageOfNodesToScore` or use scheduling profiles.
- **No cross-cluster awareness**: The scheduler only sees nodes in its own cluster.

## Best Practices

| Practice | Why |
|----------|-----|
| Set accurate resource requests | Scheduler relies on requests for placement. Over-requesting wastes capacity, under-requesting causes contention. |
| Use topology spread for production | Prevents all replicas from landing in one AZ/node |
| Combine nodeSelector with taints | nodeSelector says "put me here", taints say "keep others out" |
| Set PDBs on all production workloads | Prevents disruption cascades during upgrades/scaling |
| Use PriorityClasses | Ensures critical services survive resource pressure |
| Prefer soft rules over hard rules | `preferred` affinity gives the scheduler flexibility to avoid Pending states |
| Monitor pending pods | A growing pending queue means you need more capacity or relaxed constraints |
| Right-size requests regularly | Review VPA recommendations to avoid over/under-provisioning |
