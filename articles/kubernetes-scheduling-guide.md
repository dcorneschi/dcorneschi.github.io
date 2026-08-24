# Kubernetes Scheduling

Scheduling in Kubernetes is the process of assigning pods to the most appropriate worker node in a cluster. The `kube-scheduler` is the control plane component responsible for this — it watches for newly created pods that have no node assigned and selects a node for each one.

## What the Scheduler Does (and Doesn't Do)

A common misconception is that the scheduler creates pods. It doesn't. When you create a Deployment, the flow is:

1. The API server receives the request and stores it in etcd.
2. The controller manager creates the ReplicaSet, which in turn creates the pod objects in a `Pending` state.
3. The kube-scheduler watches for `Pending` pods, evaluates available nodes, and assigns each pod to a node.
4. The kubelet on the assigned node pulls images, starts containers, and manages the pod lifecycle.

The scheduler's only job is step 3 — deciding *where* a pod runs, not *whether* it exists.

## The Scheduling Pipeline

Each pod goes through four stages:

```
Scheduling Queue → Filtering → Scoring → Binding
```

## kube-scheduler Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        kube-scheduler                              │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Scheduling Queue                          │  │
│  │  ┌──────────────┐  ┌───────────────┐  ┌───────────────────┐  │  │
│  │  │ Active Queue │  │ Backoff Queue │  │Unschedulable Queue│  │  │
│  │  └──────┬───────┘  └───────────────┘  └───────────────────┘  │  │
│  └─────────┼────────────────────────────────────────────────────┘  │
│            │                                                       │
│            ▼                                                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                 Scheduling Framework                         │  │
│  │                                                              │  │
│  │  PreFilter → Filter → PreScore → Score → Reserve → Permit    │  │
│  │                                      │                       │  │
│  │                              PreBind → Bind → PostBind       │  │
│  └──────────────────────────────────────────────────────────────┘  │
│            │                                                       │
│            ▼                                                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Scheduler Cache                           │  │
│  │  Node resources, labels, taints, pod info, PVs, StorageClass │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
         ▲                                        │
         │ watches for unscheduled pods           │ binds pod to node
         ▼                                        ▼
┌────────────────────┐                   ┌──────────────────┐
│    API Server      │                   │  kubelet (node)  │
│  (etcd backing)    │                   │  starts the pod  │
└────────────────────┘                   └──────────────────┘
```

## Scheduling Queue Internals

The queue has three sub-queues that manage pod lifecycle before scheduling:

| Sub-Queue | Purpose |
|-----------|---------|
| **Active Queue** | Pods ready for scheduling, ordered by PriorityClass |
| **Backoff Queue** | Pods that failed scheduling, waiting with exponential backoff before retry |
| **Unschedulable Queue** | Pods that can't be scheduled due to current cluster state (re-evaluated when state changes) |

Flow between queues:

```
New pod → Active Queue → scheduling attempt → success → Bind
                                            → failure → Backoff Queue → (timer) → Active Queue
                                            → unschedulable → Unschedulable Queue → (cluster change) → Active Queue
```

Pods move from Unschedulable back to Active when relevant cluster state changes (new node added, pod deleted, labels changed). This prevents wasting cycles retrying pods that can't possibly schedule.

### Scheduling Queue

Pending pods enter a priority queue. Pods with higher `PriorityClass` values are dequeued first. Without a PriorityClass, pods default to priority `0`.

### Filtering (Predicates)

The scheduler eliminates nodes that can't run the pod. Only nodes that pass all filters — called "feasible nodes" — move to the next stage. Kubernetes runs 13 default predicates, including:

| Predicate | What It Checks |
|-----------|---------------|
| `PodFitsResources` | Node has enough available CPU, memory, and other requested resources |
| `PodToleratesNodeTaints` | Pod tolerates any taints applied to the node |
| `NodeUnschedulable` | Node is not cordoned (`kubectl cordon`) |
| `VolumeBinding` | Node can attach the required persistent volumes |
| `MatchNodeSelector` | Node labels satisfy the pod's `nodeSelector` |
| `PodFitsHostPorts` | Requested host ports are not already in use on the node |

If no nodes pass filtering, the pod stays `Pending`.

### Scoring (Priorities)

The scheduler ranks feasible nodes on a 0–100 scale using scoring functions. Kubernetes uses 13 default scoring functions, including:

| Scoring Function | What It Favors |
|-----------------|---------------|
| `NodeResourcesBalancedAllocation` | Nodes with balanced CPU and memory utilization |
| `ImageLocality` | Nodes that already have the pod's container image cached |
| `InterPodAffinity` | Nodes that satisfy pod affinity/anti-affinity preferences |
| `NodePreferAvoidPods` | Nodes not annotated to discourage certain pods |
| `TaintToleration` | Nodes with fewer untolerated taints |

If multiple nodes tie with the highest score, the scheduler picks one at random.

### Binding

The scheduler tells the API server to bind the pod to the winning node. The API server updates the pod object in etcd, and the kubelet on that node takes over.

The entire filtering → scoring → binding cycle repeats for each pod in the queue.

## Configuring the Scheduler

There are two ways to customize how the kube-scheduler filters and scores nodes:

- **Scheduling Policies** (legacy) — Define rules called Predicates for filtering and Priorities for scoring. Being phased out in favor of profiles.
- **Scheduling Profiles** — Use the plugin-based Scheduling Framework to customize individual stages: `QueueSort`, `PreFilter`, `Filter`, `PostFilter`, `PreScore`, `Score`, `Reserve`, `Permit`, `PreBind`, `Bind`, `PostBind`. Multiple profiles can run in a single scheduler instance, each with different plugin configurations.

### Scheduler Configuration Example

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  plugins:
    filter:
      enabled:
      - name: NodeResourcesFit
      - name: NodeAffinity
      - name: TaintToleration
      - name: NodeUnschedulable
    score:
      enabled:
      - name: NodeResourcesFit
        weight: 1
      - name: NodeAffinity
        weight: 2
      - name: InterPodAffinity
        weight: 1
      - name: ImageLocality
        weight: 1
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: LeastAllocated       # Prefer nodes with more free resources
```

You can also run multiple profiles for different workload types — pods select their profile via `spec.schedulerName`.

### Default Plugins by Phase

| Phase | Default Plugins | Purpose |
|-------|----------------|---------|
| **PreFilter** | NodeResourcesFit, NodePorts, VolumeBinding | Early feasibility checks |
| **Filter** | NodeUnschedulable, NodeName, TaintToleration, NodeAffinity, NodePorts, NodeResourcesFit, VolumeBinding | Remove infeasible nodes |
| **PostFilter** | DefaultPreemption | Preempt lower-priority pods if no nodes are feasible |
| **PreScore** | InterPodAffinity, TaintToleration | Prepare data for scoring |
| **Score** | NodeResourcesFit, NodeAffinity, InterPodAffinity, ImageLocality, TaintToleration | Rank feasible nodes 0–100 |
| **Reserve** | VolumeBinding | Reserve resources before binding |
| **Bind** | DefaultBinder | Bind pod to node via API server |

### Scheduler Cache

The scheduler maintains a local cache of cluster state to avoid hammering the API server on every scheduling cycle:

- **Node info** — allocatable resources, labels, taints, conditions
- **Pod info** — running pods per node, resource requests
- **PV/PVC info** — volume availability and binding state
- **StorageClass info** — provisioner details

The cache is updated via informers (watch events from the API server). This means the scheduler works with a slightly stale view of the cluster — in rare cases, a pod may be bound to a node that became infeasible between the scheduling decision and the bind.

### Performance Tuning

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `percentageOfNodesToScore` | 50% (scales with cluster size) | Limits how many feasible nodes are scored — reduces latency in large clusters |
| `parallelism` | 16 | Number of pods scheduled in parallel |

For clusters with 5000+ nodes, reducing `percentageOfNodesToScore` to 10–20% significantly improves scheduling throughput with minimal impact on placement quality.

### Monitoring the Scheduler

Key Prometheus metrics:

```bash
# Scheduling latency (time to schedule a pod)
scheduler_scheduling_algorithm_duration_seconds

# Queue depth (pods waiting to be scheduled)
scheduler_pending_pods{queue="active"}
scheduler_pending_pods{queue="backoff"}
scheduler_pending_pods{queue="unschedulable"}

# Scheduling attempts (success/failure/error)
scheduler_schedule_attempts_total

# Preemption attempts
scheduler_preemption_attempts_total

# Plugin execution duration
scheduler_plugin_execution_duration_seconds
```

## Scheduling Techniques at a Glance

Kubernetes offers several mechanisms to influence pod placement. Each targets a different aspect of the scheduling decision:

| Technique | What It Does |
|-----------|-------------|
| `nodeSelector` | Hard constraint — pod only runs on nodes with matching labels |
| Node Affinity / Anti-Affinity | Expressive hard and soft rules for node selection based on labels |
| Pod Affinity / Anti-Affinity | Co-locate or spread pods relative to other pods |
| Taints and Tolerations | Repel pods from nodes unless they explicitly tolerate the taint |
| Topology Spread Constraints | Distribute pods evenly across zones, nodes, or other topology domains |
| Priority and Preemption | Queue ordering and eviction of lower-priority pods under resource pressure |
| `nodeName` | Bypass the scheduler entirely — force a pod onto a specific node |
| Custom Schedulers | Run alternative schedulers alongside kube-scheduler via `schedulerName` |
| Topology Management Policies | Kubelet-level NUMA-aware resource alignment for latency-sensitive workloads |
| QoS Classes | Affects eviction order during preemption — Guaranteed pods are evicted last |
| Pod Disruption Budgets | Limits how many pods can be unavailable during voluntary disruptions |

## Resource Management and Scheduling

The scheduler's filtering phase depends heavily on resource declarations. Without them, the scheduler can't make informed placement decisions, and pods may land on nodes that can't actually support them.

### Resource Requests and Limits

- **Requests** define the minimum CPU and memory a pod needs. The scheduler uses requests (not limits) during filtering — a node must have enough allocatable resources to satisfy the request.
- **Limits** define the maximum a pod can consume. The kubelet enforces limits at runtime, but the scheduler doesn't consider them during placement.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "64Mi"
  limits:
    cpu: "500m"
    memory: "128Mi"
```

If you don't set requests, the scheduler treats the pod as needing zero resources, which means it can land anywhere — even on a node that's already overcommitted.

### Resource Quotas

`ResourceQuota` objects cap the total resource consumption within a namespace. They don't directly affect scheduling decisions, but they prevent pods from being created if the namespace has exceeded its quota — meaning those pods never reach the scheduling queue.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: demo-ns
spec:
  hard:
    cpu: "20"
    memory: "100Gi"
    pods: "50"
```

### LimitRange

A `LimitRange` enforces per-pod or per-container resource boundaries within a namespace. It sets default requests/limits for containers that don't specify them, and rejects pods that exceed the defined min/max range.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: demo-ns
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "250m"
      memory: "128Mi"
    max:
      cpu: "1"
      memory: "512Mi"
    min:
      cpu: "100m"
      memory: "64Mi"
    type: Container
```

This is particularly useful as a safety net — it ensures every container in the namespace has resource requests, which in turn gives the scheduler the data it needs to make good placement decisions.

## Horizontal Scaling and Scheduling

When Horizontal Pod Autoscaler (HPA) scales a Deployment up, the new pod replicas enter the scheduling queue like any other pod. The scheduler then places them across the cluster based on the same filtering and scoring logic.

This interaction matters because:

- **Affinity rules affect scale-out** — If your pods have strict anti-affinity (one replica per node), HPA can only scale up to the number of available nodes. Beyond that, new replicas stay `Pending` and may trigger the Cluster Autoscaler to add nodes.
- **Resource requests gate placement** — HPA creates pods with the same resource requests as the template. If the cluster is tight on resources, new replicas may not schedule even though HPA decided they're needed.
- **Topology spread constraints distribute replicas** — If configured, the scheduler will spread HPA-created replicas across zones or nodes, improving fault tolerance automatically.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

HPA adjusts replica count based on observed metrics. The scheduler handles where those replicas land. They're complementary — HPA decides *how many*, the scheduler decides *where*.

## Pod Disruption Budgets

A `PodDisruptionBudget` (PDB) limits how many pods from a set can be unavailable at the same time during voluntary disruptions — node drains, cluster upgrades, or autoscaler scale-downs. PDBs don't affect scheduling directly, but they constrain what the scheduler and other controllers can evict.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

You can specify either `minAvailable` (minimum pods that must stay running) or `maxUnavailable` (maximum pods that can be down at once) — not both.

A common mistake is making PDBs too restrictive. If `minAvailable` equals your replica count, no pod can ever be voluntarily evicted, which blocks node drains and cluster upgrades entirely. A good rule of thumb: allow roughly 25% of replicas to be unavailable for most services.

PDBs pair well with Priority and Preemption — the scheduler tries to honor PDBs when selecting preemption victims, though it will violate them if that's the only way to schedule a critical pod.

## Real-World Scheduling Patterns

### Multi-Tenant Workload Isolation

In shared clusters, use pod anti-affinity to distribute a single tenant's pods across nodes for fault tolerance, and node affinity with dedicated node pools for strict tenant separation:

```yaml
# Distribute tenant-a pods across nodes
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: tenant
          operator: In
          values:
          - tenant-a
      topologyKey: kubernetes.io/hostname
```

For hard isolation where tenants never share nodes, combine taints on dedicated node pools with node affinity:

```yaml
# Strict: tenant-a pods only on tenant-a nodes
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: dedicated
          operator: In
          values:
          - tenant-a
```

### Data Locality

Co-locate processing pods with their data stores to reduce network latency. Use soft pod affinity so placement degrades gracefully if the ideal node is full:

```yaml
affinity:
  podAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - data-store
        topologyKey: kubernetes.io/hostname
```

### High Availability Across Failure Domains

For critical services, combine hard anti-affinity at the node level with soft anti-affinity at the zone level:

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - critical-service
      topologyKey: kubernetes.io/hostname
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - critical-service
        topologyKey: topology.kubernetes.io/zone
```

This guarantees replicas are on different nodes and tries to spread them across zones. Typical pattern for database clusters and API gateways.

## Cloud Provider Notes for Custom Schedulers

Custom scheduler support varies by managed Kubernetes offering:

| Provider | Support | Caveats |
|----------|---------|---------|
| **Amazon EKS** | Scheduler framework extensions and standalone custom schedulers as pods | EKS Fargate does not support custom schedulers at all |
| **Azure AKS** | Scheduler extensions and custom schedulers as workloads | AKS updates may override default scheduler modifications unless you use specific container hooks |
| **Google GKE** | Full support in GKE Standard for both approaches | GKE Autopilot does not support custom schedulers |

If you're on a managed platform with serverless/autopilot modes, verify custom scheduler support before investing in one.

## Best Practices

- **Set resource requests on everything.** The scheduler can't make good decisions without them. For stable workloads, set requests to average usage plus a small buffer. For variable workloads, add a larger buffer. Avoid arbitrary round numbers like "1 CPU for every service."
- **Start with soft constraints.** Use `preferred` affinity rules first and observe behavior before switching to `required`. Hard constraints that are too strict leave pods `Pending` when infrastructure changes.
- **Combine priority classes with PDBs.** Priority ensures critical pods get scheduled; PDBs ensure they aren't all disrupted at once during maintenance. Together they cover both resource contention and voluntary disruptions.
- **Use dedicated node pools for specialized hardware.** Taint GPU/high-memory nodes and require matching tolerations + node affinity. This prevents general workloads from wasting expensive resources.
- **Monitor pending pods.** Pods stuck in `Pending` are the first sign of scheduling problems — overly restrictive constraints, missing labels, or insufficient resources. Set up alerts for pods pending longer than a few minutes.
- **Audit scheduling policies regularly.** As clusters evolve, old affinity rules and taints can cause resource fragmentation — nodes sit underutilized because only specific pods can land on them. Review and simplify periodically.
- **Watch scheduler latency in large clusters.** Complex inter-pod affinity rules are expensive to evaluate. If scheduling latency increases, consider simplifying rules or tuning `percentageOfNodesToScore`.

## Debugging Scheduling Issues

### Why is my pod Pending?

```bash
# Check the pod's events for FailedScheduling reason
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 Events

# Common messages and what they mean:
# "Insufficient cpu"              → No node has enough free CPU (requests)
# "Insufficient memory"           → No node has enough free memory (requests)
# "didn't match Pod's node affinity/selector" → No node has the required labels
# "node(s) had taint ... that the pod didn't tolerate" → Missing toleration
# "didn't match topology spread constraints" → Can't spread evenly
# "persistentvolumeclaim not found" → PVC doesn't exist or isn't bound
```

### Check node capacity and allocations

```bash
# Overview of all nodes — what's available vs used
kubectl top nodes

# Detailed allocations on a specific node
kubectl describe node <node-name> | grep -A 6 "Allocated resources"

# Allocatable resources per node
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
CPU:.status.allocatable.cpu,\
MEM:.status.allocatable.memory

# Which pods are on a node and how much they request
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node-name> \
  -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,CPU-REQ:.spec.containers[*].resources.requests.cpu,MEM-REQ:.spec.containers[*].resources.requests.memory
```

### Check taints and tolerations

```bash
# List all node taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Check if a pod tolerates a specific taint
kubectl get pod <pod-name> -o jsonpath='{.spec.tolerations}' | jq
```

### Check affinity and selectors

```bash
# View pod's scheduling constraints
kubectl get pod <pod-name> -o jsonpath='{.spec.nodeSelector}'
kubectl get pod <pod-name> -o jsonpath='{.spec.affinity}' | jq

# Find nodes matching a label selector
kubectl get nodes -l <key>=<value>
```

### Check scheduler events cluster-wide

```bash
# All scheduling events (recent)
kubectl get events -A --field-selector reason=FailedScheduling --sort-by='.lastTimestamp'

# Scheduler logs (if accessible)
kubectl logs -n kube-system -l component=kube-scheduler --tail=50
```

### Common fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Insufficient cpu/memory` | Nodes full (by requests) | Reduce requests, add nodes, or evict lower-priority pods |
| `didn't match node affinity` | No node has required labels | Add labels to nodes or relax affinity rules |
| `had taint ... didn't tolerate` | Pod missing toleration | Add toleration or remove taint |
| `topology spread constraints` | Can't balance across domains | Add nodes in the lacking domain or relax `maxSkew` |
| `persistentvolumeclaim not found` | PVC missing or in wrong namespace | Create the PVC or fix the name |
| `0/N nodes available` + no specific reason | Check for cordoned nodes | `kubectl uncordon <node>` |

## Scheduler Limitations and Challenges

The default scheduler handles most workloads well, but has known limitations:

- **Misconfigured resource requests** — The scheduler relies entirely on declared requests and limits. If these are missing or wrong, pods end up on unsuitable nodes or cause resource contention the scheduler can't predict.
- **Configuration complexity** — Affinity rules, taints, tolerations, topology constraints, and priority classes are powerful but easy to misconfigure. A single wrong operator or missing toleration can leave pods permanently `Pending`.
- **Scalability at large scale** — In clusters with thousands of nodes and pods, the default scheduler can become a bottleneck. Filtering and scoring every node for every pod adds latency. The `percentageOfNodesToScore` parameter can help by limiting how many nodes the scheduler evaluates.
- **Single-scheduler bottleneck** — Clusters typically run one scheduler instance. For specialized workloads, you can run multiple schedulers or use Scheduling Profiles to define different behaviors within a single scheduler.
- **Preemption side effects** — Evicting lower-priority pods to make room for higher-priority ones can disrupt running workloads. Preemption is not always deterministic in which pods get evicted, which can cause instability if not planned carefully.
- **No distinction between unready and unschedulable** — The scheduler doesn't differentiate between pods that are waiting for external dependencies (e.g., a volume that hasn't been provisioned yet) and pods that genuinely can't be placed due to missing requirements. Both sit in the queue and get retried, which can waste scheduling cycles.
- **No rescheduling** — The scheduler makes a one-time placement decision. If conditions change after binding (node becomes overloaded, labels change), the scheduler doesn't move the pod. Tools like [descheduler](https://github.com/kubernetes-sigs/descheduler) can help rebalance.
