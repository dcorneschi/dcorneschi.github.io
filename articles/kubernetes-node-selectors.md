# Node Selectors in Kubernetes

Node selectors are the simplest way to constrain pods to run on specific nodes. They match against node labels — if all labels match, the pod can be scheduled there.

## How nodeSelector Works

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  nodeSelector:
    disktype: ssd
    environment: production
  containers:
    - name: app
      image: my-app:latest
```

The scheduler only considers nodes that have **all** the specified labels. If no node matches, the pod stays in `Pending`.

## Built-in Node Labels

Every node automatically gets labels from the kubelet and cloud provider:

| Label | Example Value | Source |
|-------|---------------|--------|
| `kubernetes.io/hostname` | `ip-10-0-1-42` | kubelet |
| `kubernetes.io/os` | `linux` | kubelet |
| `kubernetes.io/arch` | `amd64` | kubelet |
| `node.kubernetes.io/instance-type` | `m5.xlarge` | cloud provider |
| `topology.kubernetes.io/zone` | `us-east-1a` | cloud provider |
| `topology.kubernetes.io/region` | `us-east-1` | cloud provider |
| `node.kubernetes.io/instance-type` | `Standard_D2s_v3` | cloud provider (AKS) |
| `karpenter.sh/capacity-type` | `spot` or `on-demand` | Karpenter |
| `eks.amazonaws.com/capacityType` | `ON_DEMAND` or `SPOT` | EKS managed nodes |

```sh
# View all labels on a node
kubectl get node <node-name> --show-labels

# View labels in a readable format
kubectl describe node <node-name> | grep -A 20 "Labels:"

# List nodes with a specific label
kubectl get nodes -l disktype=ssd
kubectl get nodes -l topology.kubernetes.io/zone=us-east-1a
```

## Adding Custom Labels to Nodes

### Label a Node Manually

```sh
# Add a label
kubectl label node <node-name> disktype=ssd

# Add multiple labels
kubectl label node <node-name> team=platform environment=production tier=frontend

# Overwrite an existing label
kubectl label node <node-name> environment=staging --overwrite

# Remove a label
kubectl label node <node-name> disktype-
```

### Label Nodes at Boot (EKS)

Set labels via kubelet arguments in the bootstrap script (userdata):

```sh
/etc/eks/bootstrap.sh my-cluster \
  --kubelet-extra-args '--node-labels=team=platform,environment=prod,workload=compute'
```

### Label Nodes via Node Group / Node Pool

```sh
# EKS managed node group
aws eks create-nodegroup --cluster-name <cluster> \
  --nodegroup-name gpu-pool \
  --labels gpu=true,team=ml \
  ...

# AKS node pool
az aks nodepool add --cluster-name <cluster> -g <rg> \
  --name gpupool \
  --labels gpu=true team=ml

# GKE node pool
gcloud container node-pools create gpu-pool \
  --cluster <cluster> \
  --node-labels=gpu=true,team=ml
```

### Label Nodes with Karpenter

Labels defined in the NodePool are applied to all nodes Karpenter creates:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu
spec:
  template:
    metadata:
      labels:
        team: ml
        gpu: "true"
        workload: training
```

## nodeSelector Examples

### Schedule on a Specific Instance Type

```yaml
spec:
  nodeSelector:
    node.kubernetes.io/instance-type: m5.2xlarge
```

### Schedule in a Specific AZ

```yaml
spec:
  nodeSelector:
    topology.kubernetes.io/zone: us-east-1a
```

### Schedule on Spot Instances (EKS)

```yaml
# EKS managed node groups
spec:
  nodeSelector:
    eks.amazonaws.com/capacityType: SPOT

# Karpenter
spec:
  nodeSelector:
    karpenter.sh/capacity-type: spot
```

### Schedule on a Specific OS

```yaml
spec:
  nodeSelector:
    kubernetes.io/os: linux
```

### Schedule on GPU Nodes

```yaml
spec:
  nodeSelector:
    gpu: "true"
  containers:
    - name: training
      image: ml-training:latest
      resources:
        limits:
          nvidia.com/gpu: 1
```

### Schedule by Team

```yaml
spec:
  nodeSelector:
    team: data-engineering
```

## nodeSelector in Deployments

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
      nodeSelector:
        environment: production
        disktype: ssd
      containers:
        - name: web
          image: nginx:latest
```

## nodeSelector vs Node Affinity vs Taints

| Feature | nodeSelector | Node Affinity | Taints + Tolerations |
|---------|:------------:|:-------------:|:--------------------:|
| Complexity | Simple | Flexible | Flexible |
| Logic | AND only (all must match) | AND, OR, In, NotIn, Exists | Repel pods unless tolerated |
| Soft preference | No (hard constraint only) | Yes (`preferredDuringScheduling`) | Yes (`PreferNoSchedule`) |
| Anti-affinity | No | Yes (avoid nodes with label) | No (use pod anti-affinity) |
| Use case | "Run here" | "Prefer here" or "Not there" | "Keep everything else out" |

### When to Use Each

- **nodeSelector**: Simple cases — "this workload must run on SSD nodes" or "only on GPU nodes"
- **Node Affinity**: Complex cases — "prefer zone-a but acceptable in zone-b" or "not on spot instances"
- **Taints + Tolerations**: Reservation — "only ML workloads on GPU nodes, nothing else"

## Node Affinity (The Advanced Alternative)

When nodeSelector isn't flexible enough, use node affinity:

### Required (Hard Constraint)

Equivalent to nodeSelector but with richer operators:

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

### Preferred (Soft Constraint)

Try to schedule here, but it's not mandatory:

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
        - weight: 20
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
```

### Node Affinity Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `In` | Value is one of the listed values | `zone In [us-east-1a, us-east-1b]` |
| `NotIn` | Value is NOT one of the listed values | `instance-type NotIn [t3.micro]` |
| `Exists` | Label key exists (any value) | `gpu Exists` |
| `DoesNotExist` | Label key does not exist | `spot DoesNotExist` |
| `Gt` | Value is greater than (numeric) | `memory Gt 16` |
| `Lt` | Value is less than (numeric) | `cpu Lt 8` |

## Combining nodeSelector with Taints

A common pattern is to use **both** for dedicated nodes:

```sh
# 1. Label the node (for nodeSelector targeting)
kubectl label node gpu-node-1 gpu=true

# 2. Taint the node (to repel non-GPU workloads)
kubectl taint node gpu-node-1 gpu=true:NoSchedule
```

```yaml
# Pod that runs on GPU nodes (has both nodeSelector and toleration)
spec:
  nodeSelector:
    gpu: "true"
  tolerations:
    - key: gpu
      value: "true"
      effect: NoSchedule
  containers:
    - name: training
      image: ml-training:latest
      resources:
        limits:
          nvidia.com/gpu: 1
```

Without the toleration, the pod can't schedule on the tainted node — even with a matching nodeSelector.

## Troubleshooting

### Pod Stuck in Pending

```sh
# Check why the pod isn't scheduling
kubectl describe pod <pod-name> | grep -A 5 "Events"

# Common messages:
# "0/5 nodes are available: 5 node(s) didn't match Pod's node affinity/selector"
# "0/5 nodes are available: 2 node(s) had untolerated taint"
```

### Verify Node Labels Match

```sh
# Check what labels a node has
kubectl get nodes --show-labels | grep <expected-label>

# Find nodes matching specific selectors
kubectl get nodes -l "disktype=ssd,environment=production"

# Count matching nodes
kubectl get nodes -l disktype=ssd --no-headers | wc -l
```

### Check All Nodes and Their Labels

```sh
# Formatted label view
kubectl get nodes -o custom-columns=NAME:.metadata.name,LABELS:.metadata.labels

# Specific labels across all nodes
kubectl get nodes -o custom-columns=NAME:.metadata.name,TYPE:.metadata.labels.node\\.kubernetes\\.io/instance-type,ZONE:.metadata.labels.topology\\.kubernetes\\.io/zone,TEAM:.metadata.labels.team
```

## One-Liners

```sh
# Label all nodes in a node group
kubectl label nodes -l eks.amazonaws.com/nodegroup=workers environment=production

# Find pods with nodeSelector set
kubectl get pods -A -o json | jq '.items[] | select(.spec.nodeSelector != null) | {name: .metadata.name, ns: .metadata.namespace, nodeSelector: .spec.nodeSelector}'

# Find which nodes a deployment can schedule on
LABELS=$(kubectl get deploy <name> -o jsonpath='{.spec.template.spec.nodeSelector}')
echo "nodeSelector: $LABELS"
kubectl get nodes -l $(echo $LABELS | jq -r 'to_entries | map("\(.key)=\(.value)") | join(",")')

# Remove a label from all nodes
kubectl label nodes --all disktype-

# Copy labels from one node to another
kubectl get node node-1 -o jsonpath='{.metadata.labels}' | jq -r 'to_entries[] | "\(.key)=\(.value)"' | xargs -I {} kubectl label node node-2 {}
```

## Gotchas

- **nodeSelector is AND logic only**: All labels must match. If you need OR logic (zone-a OR zone-b), use node affinity with the `In` operator.
- **Labels are strings**: `gpu: true` and `gpu: "true"` are the same in YAML, but `gpu: yes` is different from `gpu: "true"`. Be consistent.
- **nodeSelector doesn't prevent other pods**: It only controls where YOUR pod goes. Other pods without a nodeSelector can still land on your labeled nodes. Use taints if you need exclusivity.
- **Label changes don't evict running pods**: If you remove a label from a node, pods already running there stay. Only new scheduling decisions use the updated labels.
- **Case sensitive**: `Disktype=SSD` is different from `disktype=ssd`.
- **EKS managed node group labels are immutable per node group**: You can change them via `update-nodegroup-config`, but it doesn't relabel existing nodes — only new nodes get the updated labels.
- **Don't use `kubernetes.io/` prefix for custom labels**: That prefix is reserved for Kubernetes system labels.


## nodeName vs nodeSelector

`nodeName` and `nodeSelector` are fundamentally different scheduling mechanisms:

| Aspect | `nodeName` | `nodeSelector` |
|--------|------------|----------------|
| Scheduler involvement | Bypassed | Uses scheduler |
| Respects taints | No (ignores) | Yes |
| Respects resource limits | No (ignores) | Yes |
| Failure behavior | Fails on node | Stays Pending |
| Rescheduling | No | Yes |
| Production use | Avoid | Recommended |

### nodeName (Direct Assignment)

```yaml
spec:
  nodeName: worker-3  # FORCES placement, bypasses scheduler
  containers:
    - name: nginx
      image: nginx:alpine
```

- Bypasses the scheduler completely
- Ignores ALL constraints (taints, resources, affinity)
- Runs immediately on the specified node
- If the node can't run it, the pod fails (not rescheduled)

### nodeSelector with hostname (Constraint-Based)

```yaml
spec:
  nodeSelector:
    kubernetes.io/hostname: worker-3  # CONSTRAINS scheduler choice
  containers:
    - name: nginx
      image: nginx:alpine
```

- Uses the scheduler normally
- Respects ALL constraints (taints, resources)
- Stays Pending if constraints can't be met
- Proper scheduling evaluation

### Scheduling Process Comparison

**nodeSelector process:**
```
1. Pod created with nodeSelector
2. Scheduler evaluates ALL nodes
3. Filters nodes matching nodeSelector labels
4. Applies additional constraints (taints, resources, affinity)
5. Scores remaining candidates
6. No suitable node → pod stays Pending
7. Suitable node found → assigns pod
```

**nodeName process:**
```
1. Pod created with nodeName
2. Scheduler is bypassed entirely
3. Pod directly assigned to specified node
4. Kubelet attempts to start pod immediately
5. Node exists → pod runs (ignoring all constraints)
6. Node doesn't exist → pod stays Pending forever
```

### Constraint Handling Differences

```sh
# Taint a node
kubectl taint nodes worker-3 maintenance=true:NoSchedule

# nodeSelector: Respects taint, pod stays Pending (unless tolerated)
# nodeName: Ignores taint, pod runs anyway
```

```yaml
# Node has 8Gi memory, pod requests 16Gi:
# nodeSelector → scheduler checks resources → pod stays Pending
# nodeName → ignores resources → kubelet tries to start (may OOM)
```

### Migration: From nodeName to nodeSelector

```yaml
# Before (problematic)
spec:
  nodeName: worker-3

# After (proper scheduling)
spec:
  nodeSelector:
    kubernetes.io/hostname: worker-3
  tolerations:
    - key: "maintenance"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
```

### When to Use Each

**Use nodeSelector when:**
- Deploying regular workloads
- You want proper scheduling behavior
- You need to respect taints and resources

**Avoid nodeName unless:**
- Creating static pods (kubelet-managed)
- Debugging specific node issues
- Testing node-specific behavior temporarily

## Managing Labels with eksctl

### Create a Node Group with Labels

```yaml
# eksctl ClusterConfig
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: managed-cluster
  region: us-west-2
managedNodeGroups:
  - name: managed-ng-1
    minSize: 2
    maxSize: 4
    desiredCapacity: 3
    labels:
      role: worker
      environment: production
```

### Update Labels on an Existing Node Group

```sh
# Set labels
eksctl set labels --cluster managed-cluster --nodegroup managed-ng-1 \
  --labels kubernetes.io/managed-by=eks,environment=prod

# Remove labels
eksctl unset labels --cluster managed-cluster --nodegroup managed-ng-1 \
  --labels kubernetes.io/managed-by
```

## Listing Node Group Names

```sh
# Get node group label for all nodes
kubectl get nodes -o custom-columns=NAME:.metadata.name,NODEGROUP:.metadata.labels.eks\.amazonaws\.com/nodegroup

# Using jq for cleaner output
kubectl get nodes -o json | jq -r '.items[] | "\(.metadata.name): \(.metadata.labels["eks.amazonaws.com/nodegroup"])"'

# List all unique node groups in the cluster
kubectl get nodes -o json | jq -r '.items[].metadata.labels["eks.amazonaws.com/nodegroup"]' | sort -u
```

## Saving Node Information to Files

```sh
# Save node group labels to a text file
kubectl get nodes -o custom-columns=NAME:.metadata.name,NODEGROUP:.metadata.labels.eks\.amazonaws\.com/nodegroup > node_groups.txt

# Save all node labels to JSON
kubectl get nodes -o json > nodes_all_labels.json

# Save comprehensive node information
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
NODEGROUP:.metadata.labels.eks\.amazonaws\.com/nodegroup,\
INSTANCE-TYPE:.metadata.labels.node\.kubernetes\.io/instance-type,\
ZONE:.metadata.labels.topology\.kubernetes\.io/zone > node_details.txt
```

## Combining nodeSelector with Node Affinity

Use nodeSelector for hard requirements and node affinity for soft preferences together:

```yaml
spec:
  nodeSelector:
    disktype: ssd  # Hard requirement: MUST have SSD
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["us-west-1a", "us-west-1b"]  # Soft preference: prefer these zones
```

If you specify both `nodeSelector` and `nodeAffinity`, **both** must be satisfied for the pod to be scheduled.

## Where nodeSelector Fits in the Scheduling Pipeline

When a pod is created, the kube-scheduler runs it through a multi-stage pipeline:

### 1. Filtering (Predicates)

The scheduler eliminates nodes that don't meet hard constraints. `nodeSelector` acts here — any node missing the required labels is filtered out. Default predicates include:

- `PodFitsResources` — node has enough CPU, memory
- `PodToleratesNodeTaints` — pod tolerates taints on the node
- `NodeUnschedulable` — excludes cordoned nodes
- `VolumeBinding` — node can attach required storage volumes
- `MatchNodeSelector` — node labels match nodeSelector

### 2. Scoring (Priorities)

Remaining candidates are scored to find the best fit. Default scoring functions include:

- `ImageLocality` — prefers nodes with the container image cached
- `NodeResourcesBalancedAllocation` — favors balanced CPU/memory utilization
- `InterPodAffinity` — scores based on pod affinity/anti-affinity rules
- `NodePreferAvoidPods` — avoids nodes annotated to discourage certain pods

`nodeSelector` does NOT influence scoring — it's purely a filter.

### 3. Selection and Binding

The node with the highest combined score wins. The scheduler binds the pod to the chosen node, and the kubelet starts the containers.

```
Pod Created
    │
    ▼
┌─────────────────────────┐
│  Filtering (Predicates)  │ ← nodeSelector acts HERE
│  - MatchNodeSelector     │
│  - PodFitsResources      │
│  - PodToleratesNodeTaints│
│  - VolumeBinding         │
└───────────┬─────────────┘
            │ (candidate nodes)
            ▼
┌─────────────────────────┐
│  Scoring (Priorities)    │ ← nodeAffinity preferred acts HERE
│  - ImageLocality         │
│  - BalancedAllocation    │
│  - InterPodAffinity      │
└───────────┬─────────────┘
            │ (highest score)
            ▼
┌─────────────────────────┐
│  Binding                 │
│  Pod → Node              │
└─────────────────────────┘
```

Because `nodeSelector` operates in the filtering phase, it's a hard constraint — if no node has the matching labels, the pod stays `Pending` indefinitely. For soft preferences, use `preferredDuringSchedulingIgnoredDuringExecution` in node affinity.


## Related Scheduling Features

### Topology Spread Constraints

Distribute pods evenly across zones or nodes to avoid single-point failures:

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: "topology.kubernetes.io/zone"
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: web
```

This ensures the difference in pod count between any two zones is at most 1.

### Pod Affinity and Anti-Affinity

**Pod affinity** — schedule pods together (e.g., low-latency microservices):

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: cache
          topologyKey: "kubernetes.io/hostname"
```

**Pod anti-affinity** — spread pods apart (e.g., replicas on different nodes):

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: web
          topologyKey: "kubernetes.io/hostname"
```

### Custom Schedulers

For advanced use cases, you can bypass the default scheduler entirely:

```yaml
spec:
  schedulerName: my-custom-scheduler
```

This tells Kubernetes to use a custom scheduler instead of the default `kube-scheduler`. Useful for AI/ML workloads or business-rule-based placement.

## Best Practices Summary

| Practice | Why |
|----------|-----|
| Use nodeSelector for hardware-specific workloads | Ensure GPU/SSD workloads run on appropriate nodes |
| Separate workloads by environment | Prevent prod/dev resource contention with `env=prod` labels |
| Combine with node affinity for flexibility | Use `preferred` rules when exact matching is too rigid |
| Implement standardized labeling conventions | Avoid `env=prod` on some nodes and `environment=production` on others |
| Apply taints alongside nodeSelector | Prevent non-critical pods from consuming specialized node resources |
| Always set resource requests | Even with nodeSelector, the scheduler needs resource info to avoid overloading nodes |
| Avoid over-labeling | Too many labels on a node makes scheduling overly restrictive |
| Keep labels consistent across the cluster | Inconsistent labels lead to hard-to-debug scheduling failures |
