# Node Affinity in Kubernetes

Node affinity is a set of scheduling rules that constrain which nodes a pod can be placed on, based on node labels. It's the more powerful successor to `nodeSelector` — supporting required and preferred rules, multiple operators, and weighted preferences.

## nodeSelector vs Node Affinity

| Feature | nodeSelector | Node Affinity |
|---------|:------------:|:-------------:|
| Logic | AND only (all must match) | AND, OR, In, NotIn, Exists, Gt, Lt |
| Hard constraint | Yes (only) | Yes (`required`) |
| Soft constraint | No | Yes (`preferred` with weights) |
| Exclude nodes | No | Yes (`NotIn`, `DoesNotExist`) |
| Multiple expressions | No (single key=value pairs) | Yes (complex match rules) |

Use `nodeSelector` for simple cases. Use node affinity when you need OR logic, exclusion, or soft preferences.

## Types of Node Affinity

### requiredDuringSchedulingIgnoredDuringExecution

A hard constraint — the pod WILL NOT schedule unless a node matches:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
                  - us-east-1b
```

If no node matches, the pod stays `Pending`.

### preferredDuringSchedulingIgnoredDuringExecution

A soft constraint — the scheduler TRIES to place the pod on matching nodes but will use others if needed:

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values:
                  - m5.xlarge
                  - m5.2xlarge
        - weight: 20
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
```

Weights range from 1 to 100. Higher weight = stronger preference. The scheduler adds up scores from all matching preferences per node.

### What "IgnoredDuringExecution" Means

Once a pod is running on a node, it stays there even if the node's labels change and no longer match the affinity rules. The rules are only evaluated at scheduling time. Kubernetes does not currently support `RequiredDuringExecution` (evict running pods if labels change), though it's a planned future feature.

## Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `In` | Label value is one of the listed values | `zone In [us-east-1a, us-east-1b]` |
| `NotIn` | Label value is NOT one of the listed values | `instance-type NotIn [t3.micro, t3.small]` |
| `Exists` | Label key exists (any value) | `gpu Exists` |
| `DoesNotExist` | Label key does NOT exist | `spot DoesNotExist` |
| `Gt` | Label value is greater than (numeric strings) | `node-memory Gt 16` |
| `Lt` | Label value is less than (numeric strings) | `node-cpu Lt 8` |

### Operator Examples

```yaml
# In: node must be in one of these zones
- key: topology.kubernetes.io/zone
  operator: In
  values: ["us-east-1a", "us-east-1b", "us-east-1c"]

# NotIn: avoid specific instance types
- key: node.kubernetes.io/instance-type
  operator: NotIn
  values: ["t3.micro", "t3.small", "t3.nano"]

# Exists: node must have a GPU label (regardless of value)
- key: nvidia.com/gpu.present
  operator: Exists

# DoesNotExist: avoid nodes with a "decommissioning" label
- key: node-lifecycle/decommissioning
  operator: DoesNotExist

# Gt: node must have more than 32GB memory (custom label)
- key: node-memory-gb
  operator: Gt
  values: ["32"]

# Lt: prefer smaller nodes (cost optimization)
- key: node-cpu-cores
  operator: Lt
  values: ["16"]
```

## How nodeSelectorTerms and matchExpressions Work Together

### Multiple nodeSelectorTerms = OR

```yaml
nodeSelectorTerms:
  - matchExpressions:    # Term 1
      - key: zone
        operator: In
        values: ["us-east-1a"]
  - matchExpressions:    # Term 2
      - key: zone
        operator: In
        values: ["eu-west-1a"]
```

The pod can schedule on a node matching Term 1 **OR** Term 2.

### Multiple matchExpressions within one term = AND

```yaml
nodeSelectorTerms:
  - matchExpressions:
      - key: zone                          # AND
        operator: In
        values: ["us-east-1a"]
      - key: node.kubernetes.io/instance-type  # AND
        operator: In
        values: ["m5.xlarge"]
```

The node must match ALL expressions within the same term.

### Combined: OR of ANDs

```yaml
nodeSelectorTerms:
  # Term 1: (zone=us-east-1a AND type=m5.xlarge)
  - matchExpressions:
      - key: topology.kubernetes.io/zone
        operator: In
        values: ["us-east-1a"]
      - key: node.kubernetes.io/instance-type
        operator: In
        values: ["m5.xlarge"]
  # OR
  # Term 2: (zone=us-east-1b AND type=m5.2xlarge)
  - matchExpressions:
      - key: topology.kubernetes.io/zone
        operator: In
        values: ["us-east-1b"]
      - key: node.kubernetes.io/instance-type
        operator: In
        values: ["m5.2xlarge"]
```

## Practical Examples

### Schedule in Specific AZs Only (Required)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values:
                      - us-east-1a
                      - us-east-1b
      containers:
        - name: web
          image: nginx:latest
```

### Prefer Large Instances, Avoid Spot (Weighted Preferences)

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 70
          preference:
            matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values:
                  - m5.2xlarge
                  - m5.4xlarge
                  - r5.2xlarge
        - weight: 30
          preference:
            matchExpressions:
              - key: karpenter.sh/capacity-type
                operator: NotIn
                values:
                  - spot
```

### Avoid Specific Nodes (Anti-Affinity Pattern)

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              # Must NOT be on decommissioning nodes
              - key: lifecycle
                operator: NotIn
                values:
                  - decommissioning
              # Must NOT be on spot instances
              - key: eks.amazonaws.com/capacityType
                operator: NotIn
                values:
                  - SPOT
```

### Require GPU Nodes (Exists Operator)

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: nvidia.com/gpu.present
                operator: Exists
  containers:
    - name: training
      image: ml-training:latest
      resources:
        limits:
          nvidia.com/gpu: 1
```

### Combine Required + Preferred

```yaml
spec:
  affinity:
    nodeAffinity:
      # MUST be in these zones
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
                  - us-east-1b
                  - us-east-1c
      # PREFER large instances
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values:
                  - m5.2xlarge
                  - m5.4xlarge
```

This says: "Only schedule in us-east-1a/b/c (hard), but prefer larger instances within those zones (soft)."

### EKS: Schedule on Specific Node Group

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: eks.amazonaws.com/nodegroup
                operator: In
                values:
                  - compute-optimized
                  - memory-optimized
```

### EKS: Prefer On-Demand Over Spot

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: eks.amazonaws.com/capacityType
                operator: In
                values:
                  - ON_DEMAND
```

Spot is still allowed, but on-demand is preferred.

## Combining Node Affinity with Other Scheduling Features

### With nodeSelector

Both must be satisfied:

```yaml
spec:
  nodeSelector:
    disktype: ssd              # Hard: must have SSD
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 50
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["us-east-1a"]   # Soft: prefer zone-a
```

### With Taints and Tolerations

Node affinity directs the pod TO a node. Taints keep OTHER pods AWAY:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: gpu
                operator: In
                values: ["true"]
  tolerations:
    - key: "gpu-dedicated"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
```

Without the toleration, the pod can't schedule on the tainted GPU node — even if affinity matches.

### With Topology Spread Constraints

Spread pods across zones while restricting to certain node types:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values: ["m5.xlarge", "m5.2xlarge"]
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: web
```

## matchFields (Node Fields)

Besides labels, you can match on node spec fields:

```yaml
nodeSelectorTerms:
  - matchFields:
      - key: metadata.name
        operator: In
        values:
          - worker-1
          - worker-2
```

This is similar to using `kubernetes.io/hostname` label but matches the actual node name field.

## How Weight Scoring Works

For preferred affinity, the scheduler calculates a score per node:

```
Node score += weight (if the preference matches)
```

Example with two preferences:

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 80    # SSD nodes get +80
    preference:
      matchExpressions:
        - key: disktype
          operator: In
          values: ["ssd"]
  - weight: 20    # zone-a nodes get +20
    preference:
      matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["us-east-1a"]
```

| Node | Has SSD? | In zone-a? | Score |
|------|:--------:|:----------:|:-----:|
| node-1 | Yes | Yes | 80 + 20 = 100 |
| node-2 | Yes | No | 80 |
| node-3 | No | Yes | 20 |
| node-4 | No | No | 0 |

The scheduler picks node-1 (highest score). If node-1 is full, it falls back to node-2, then node-3.

## Troubleshooting

### Pod Stuck in Pending

```sh
# Check scheduling events
kubectl describe pod <pod-name> | grep -A 10 "Events"

# Common messages:
# "0/5 nodes are available: 5 node(s) didn't match Pod's node affinity/selector"
# "0/3 nodes are available: 2 node(s) didn't match Pod's node affinity/selector, 1 Insufficient cpu"
```

### Verify Node Labels

```sh
# Check what labels a node has
kubectl get nodes --show-labels

# Check specific labels across all nodes
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
ZONE:.metadata.labels.topology\\.kubernetes\\.io/zone,\
TYPE:.metadata.labels.node\\.kubernetes\\.io/instance-type,\
CAPACITY:.metadata.labels.eks\\.amazonaws\\.com/capacityType

# Find nodes matching specific expressions
kubectl get nodes -l "topology.kubernetes.io/zone in (us-east-1a,us-east-1b)"
kubectl get nodes -l "node.kubernetes.io/instance-type in (m5.xlarge,m5.2xlarge)"
kubectl get nodes -l '!lifecycle/decommissioning'   # DoesNotExist equivalent
```

### Debug Scoring (Why Did It Pick THAT Node?)

```sh
# Check where a pod landed
kubectl get pod <pod-name> -o wide

# Enable verbose scheduler logging (self-managed clusters)
# Add --v=10 to kube-scheduler flags

# EKS: enable scheduler logs
aws eks update-cluster-config --name <cluster> \
  --logging '{"clusterLogging":[{"types":["scheduler"],"enabled":true}]}'
```

## One-Liners

```sh
# Find all deployments using node affinity
kubectl get deploy -A -o json | jq '.items[] | select(.spec.template.spec.affinity.nodeAffinity != null) | {name: .metadata.name, ns: .metadata.namespace}'

# Find pods with required node affinity
kubectl get pods -A -o json | jq '.items[] | select(.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution != null) | {name: .metadata.name, ns: .metadata.namespace, node: .spec.nodeName}'

# Check which nodes would match a specific expression
kubectl get nodes -l "topology.kubernetes.io/zone in (us-east-1a,us-east-1b)" --no-headers | wc -l

# List unique values for a label across all nodes
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.labels.node\.kubernetes\.io/instance-type}{"\n"}{end}' | sort -u
```

## Gotchas

- **`IgnoredDuringExecution` means no eviction**: If you remove a label from a node, running pods with affinity to that label stay put. Only new scheduling decisions use the updated labels.
- **Empty `nodeSelectorTerms` matches everything**: An empty list means no constraint — every node is valid.
- **`Gt` and `Lt` compare strings as integers**: The label value must be a parseable integer. `"16"` works, `"16Gi"` does not.
- **Required affinity is per-scheduling-attempt**: If a node goes down and the pod is rescheduled, the affinity rules are re-evaluated against current cluster state.
- **Too many preferred rules slow scheduling**: Each preference is evaluated against every feasible node. Keep preferences focused.
- **`NotIn` with empty values = `DoesNotExist`**: `operator: NotIn, values: []` is equivalent to `DoesNotExist`.
- **Affinity doesn't reserve nodes**: Node affinity says "put me here" but doesn't prevent other pods from landing on the same node. Use taints for exclusivity.
- **nodeSelector + nodeAffinity = both must match**: They're ANDed together. A pod with both that points to different nodes will be Pending forever.
- **Labels are case-sensitive**: `Zone=us-east-1a` is different from `zone=us-east-1a`.


## Node Anti-Affinity

Node Anti-Affinity isn't a separate API — it's node affinity using `NotIn` or `DoesNotExist` operators to repel pods away from certain nodes. The intent is the opposite: instead of attracting pods to nodes, you're excluding nodes from consideration.

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: NotIn
                values:
                  - hdd
              - key: node-lifecycle
                operator: NotIn
                values:
                  - spot
                  - preemptible
```

This pod will never schedule on HDD nodes or spot/preemptible nodes. Combine positive and negative affinity in the same spec to both attract and repel.

## Scheduling vs Execution Lifecycle

| Phase | Behavior |
|-------|----------|
| **During Scheduling** | The scheduler evaluates affinity rules when the pod is created. If `required` and no matching node exists, the pod stays Pending. |
| **During Execution** | Current types ignore label changes after the pod is running. The pod keeps running even if an admin removes the matching label. |

### Planned Future Type

A third type — `requiredDuringSchedulingRequiredDuringExecution` — would evict running pods if the node no longer satisfies the affinity rules. This is not yet available but is planned for a future Kubernetes release.

**Scenario:**
1. Pod is scheduled on a node with label `size=Large`
2. Admin removes the `size` label from that node
3. With current types → Pod continues running (ignored during execution)
4. With future `RequiredDuringExecution` → Pod would be evicted

## Combining Node Affinity + Pod Affinity

Layer both in a single spec for multi-dimensional placement:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - cache
          topologyKey: kubernetes.io/hostname
```

This pod must be in zone `us-east-1a` (node affinity) AND on the same node as a `cache` pod (pod affinity). Both conditions must be satisfied.

## Affinity and Autoscaling Interaction

Strict affinity rules can trigger unexpected autoscaling. Example — a Deployment with pod anti-affinity requiring each replica on a different node:

```yaml
spec:
  replicas: 10
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - web-app
              topologyKey: kubernetes.io/hostname
```

Even if total cluster CPU/memory is sufficient, the cluster autoscaler may add nodes because the anti-affinity rule demands 10 distinct nodes. Keep this in mind when designing HA spreads — prefer `preferred` over `required` anti-affinity when possible to avoid forced scale-out.

## Production Tips

- Use `required` rules only for hard constraints (compliance, hardware). Prefer `preferred` for everything else to avoid unschedulable pods.
- Monitor pending pods caused by affinity — if pods stay Pending, consider relaxing rules or expanding matching labels.
- Watch scheduling latency — complex affinity rules across large clusters slow down the scheduler.
- Track pod distribution across topology domains to verify your spread rules are working as intended.
- Strict affinity can cause resource fragmentation — nodes may be underutilized because pods can only land on specific ones.
- Affinity rules don't prevent other pods from using the same nodes — combine with taints for true isolation.
