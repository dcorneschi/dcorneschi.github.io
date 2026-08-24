# Cluster Autoscaler Scale-Up Troubleshooting

Why the Cluster Autoscaler might not scale up despite pending pods — how it estimates capacity, evaluates node groups, and how to debug overprovisioning failures.

## How Cluster Autoscaler Estimates Available Memory

When the autoscaler simulates adding a new node, it:

1. Takes the node's **allocatable memory** (total capacity minus kubelet reserved, system reserved, and eviction thresholds)
2. Subtracts the **resource requests** of all DaemonSet pods that would run on that node
3. Checks if the remaining capacity can fit the pending pod(s)

### Allocatable vs Total Capacity

- **Total capacity**: raw instance memory (e.g., 16 GB)
- **Allocatable**: what Kubernetes makes available for pods = total - kubelet reserved - system reserved - eviction thresholds
- The allocatable value does **not** include DaemonSet overhead — that's subtracted separately during scheduling simulation

### DaemonSet Awareness

The autoscaler queries all DaemonSets in the cluster and simulates which ones would land on the new node (based on nodeSelectors, tolerations, affinities). It then subtracts their **requests** from allocatable to determine what's left for workload pods.

> **Important**: If DaemonSet pods have no resource requests or understate them, the autoscaler's simulation will overestimate available space.

## How the Autoscaler Decides to Scale Up

The autoscaler only acts on pods that meet **all** of these conditions:

- Pod is in `Pending` state
- Pod has condition: `PodScheduled = False` with reason `Unschedulable`
- Pod priority is **above** the `--expendable-pods-priority-cutoff` (default: `-10`)

If any of these aren't met, the autoscaler ignores the pod.

### Why "Unschedulable" Matters

| Reason | Autoscaler Action |
|---|---|
| `Unschedulable` (insufficient resources, no matching nodes) | **Triggers scale-up evaluation** |
| Volume not found / PVC issue | Ignored |
| Image pull error | Ignored |
| SchedulerError | Ignored |

## Expander Strategy: `most-pods`

The autoscaler uses an **expander** to choose which node group to scale when multiple groups are eligible:

| Expander | Behavior |
|---|---|
| `random` | Picks any eligible node group randomly |
| `least-waste` | Picks the group where the pod wastes the least resources |
| `most-pods` | Picks the group that can schedule the most pending pods |
| `priority` | Uses a priority configmap to rank node groups |

With `most-pods`, the autoscaler simulates adding one node from each eligible group and picks the one that fits the **most pending pods**. This favors larger instances or groups with less DaemonSet overhead.

> Even if instance types are identical, differences in DaemonSets targeted to specific node groups (via nodeSelectors) can shift the calculation.

## Overprovisioning and PriorityClass

Overprovisioning works by running low-priority "placeholder" pods that get preempted when real workloads need space.

### How It Works

1. Placeholder pods consume space on nodes
2. When a real workload pod arrives and there's no room, the scheduler preempts the low-priority placeholder
3. Real pod takes the space immediately (no waiting for scale-up)
4. The evicted placeholder goes `Pending` → triggers the autoscaler to add a new node
5. New node comes up → placeholder schedules there → buffer is restored

### Expendable Pods Priority Cutoff

The flag `--expendable-pods-priority-cutoff` (default: `-10`) determines which pods are "expendable":

- Pods with priority **< cutoff** → expendable, won't trigger scale-up
- Pods with priority **≥ cutoff** → will trigger scale-up

With cutoff at `-10` and overprovisioning pods at priority `-1`:
- `-1` > `-10` → pods **will** trigger scale-up ✓
- Real pods (priority ≥ 0) can preempt them ✓

> **Warning**: Setting overprovisioning priority to `0` means regular pods (also at `0`) can no longer preempt them, defeating the purpose.

## Root Cause Example: Why Scale-Up Is Not Happening

### Autoscaler Event Message

```
pod didn't trigger scale-up: 3 Insufficient memory, 3 node(s) didn't match Pod's node affinity/selector
```

### Scheduler Event Message

```
0/8 nodes are available: 3 node(s) didn't match Pod's node affinity/selector, 4 Insufficient cpu, 5 Insufficient memory.
preemption: 0/8 nodes are available: 3 Preemption is not helpful for scheduling, 5 No preemption victims found for incoming pod.
```

### The Math (16 GB Instance Type)

```
~16 GB total instance memory
~14.5 GB allocatable (after kernel, kubelet overhead)
- ~1.0 GB kube-reserved
- ~2.0 GB DaemonSet requests
= ~11.5-12.5 GB available for workload pods
```

The overprovisioning pod requests **13 GB** → does not fit in ~12.5 GB available.

**Result:** No eligible node group can fit this pod. The autoscaler correctly refuses to scale up.

## Solutions

### Option 1: Reduce Overprovisioning Pod Resource Requests

Lower the memory request to what actually fits on a node after DaemonSet overhead:

```yaml
resources:
  requests:
    cpu: 6
    memory: 12Gi  # Adjusted to fit after DaemonSet overhead
```

### Option 2: Use a Larger Instance Type

Scale the node group to use larger instances (e.g., 32 GB) so that after DaemonSet overhead, 13 GB fits comfortably.

### Option 3: Switch the Expander Strategy

Use the `priority` expander to prefer a specific node group:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-priority-expander
  namespace: kube-system
data:
  priorities: |-
    10:
      - .*old-node-group.*
    50:
      - .*preferred-new-node-group.*
```

## Calculating Actual Available Memory Per Node

### Step 1: Get Node Allocatable

```bash
kubectl get nodes -l <worker-label> -o json | \
  jq '.items[0] | {name: .metadata.name, allocatable_memory: .status.allocatable.memory, allocatable_cpu: .status.allocatable.cpu}'
```

### Step 2: Get DaemonSet Requests on That Node

```bash
NODE="<node-name>"
kubectl get pods --all-namespaces --field-selector spec.nodeName=$NODE -o json | \
  jq '[.items[] | select(.metadata.ownerReferences[]?.kind == "DaemonSet") | .spec.containers[].resources.requests] | {total_memory_Mi: [.[].memory // "0Mi" | gsub("Mi";"") | gsub("Gi";"000") | tonumber] | add, total_cpu_m: [.[].cpu // "0m" | gsub("m";"") | tonumber] | add}'
```

### Step 3: Combined Command

```bash
NODE=$(kubectl get nodes -l <worker-label> -o jsonpath='{.items[0].metadata.name}')

echo "=== Node: $NODE ==="
echo "--- Allocatable ---"
kubectl get node $NODE -o json | jq '{memory: .status.allocatable.memory, cpu: .status.allocatable.cpu}'

echo "--- DaemonSet Requests ---"
kubectl get pods --all-namespaces --field-selector spec.nodeName=$NODE -o json | \
  jq '{memory_Mi: [.items[] | select(.metadata.ownerReferences[]?.kind == "DaemonSet") | .spec.containers[].resources.requests.memory // "0Mi" | gsub("Mi";"") | gsub("Gi";"1000") | tonumber] | add, cpu_m: [.items[] | select(.metadata.ownerReferences[]?.kind == "DaemonSet") | .spec.containers[].resources.requests.cpu // "0m" | gsub("m";"") | tonumber] | add}'

echo "--- Available = Allocatable - DaemonSets ---"
echo "(Subtract manually or adjust overprovisioning request to fit)"
```

### The Formula

```
Available for workload pods = Node Allocatable - Sum(DaemonSet requests)
```

Set your overprovisioning pod request to be **less than** this value.

## Other Common Reasons Scale-Up Can Fail

| Reason | Description |
|---|---|
| Node group at max size | ASG/managed node group already at maximum capacity |
| Pod too large for any node | Pod requests more resources than any node group can provide after overhead |
| nodeSelector/affinity mismatch | Pod requires labels that no scalable node group provides |
| Taints without tolerations | Node group has taints the pod doesn't tolerate |
| PVC zone lock | Volume is bound to one AZ, but only another AZ's node group can scale |
| Scale-up backoff | Autoscaler backing off after a recent failed attempt (default 3 min) |
| Pod anti-affinity | Anti-affinity rules prevent scheduling on any available group |

## Debugging Commands

```bash
# Check autoscaler status
kubectl -n kube-system get configmap cluster-autoscaler-status -o yaml

# Check autoscaler logs for scale-up decisions
kubectl -n kube-system logs -l app=cluster-autoscaler --tail=500 | grep -iE "scale.up|unschedulable|rejected|predicate|insufficient|max"

# Check pending pods
kubectl get pods --all-namespaces --field-selector status.phase=Pending

# Check pod scheduling condition and reason
kubectl get pod <pod-name> -n <namespace> -o json | jq '.status.conditions[] | select(.type=="PodScheduled") | {status: .status, reason: .reason, message: .message}'

# Check node allocatable resources
kubectl get nodes -l <worker-label> -o json | \
  jq '.items[] | {name: .metadata.name, allocatable_memory: .status.allocatable.memory, allocatable_cpu: .status.allocatable.cpu}'

# Sum all DaemonSet memory requests
kubectl get ds --all-namespaces -o json | \
  jq '[.items[].spec.template.spec.containers[].resources.requests.memory // "0Mi" | gsub("Mi";"") | gsub("Gi";"000") | tonumber] | add'

# Check DaemonSet resource requests per DaemonSet
kubectl get ds --all-namespaces -o json | \
  jq '.items[] | {name: .metadata.name, namespace: .metadata.namespace, memory_request: ([.spec.template.spec.containers[].resources.requests.memory // "0Mi"] | join(", ")), cpu_request: ([.spec.template.spec.containers[].resources.requests.cpu // "0m"] | join(", "))}'

# Check expander setting
kubectl -n kube-system get deployment cluster-autoscaler -o yaml | grep -A2 "expander"

# Check expendable pods priority cutoff
kubectl -n kube-system get deployment cluster-autoscaler -o yaml | grep "expendable"

# Check what node groups the autoscaler manages
kubectl -n kube-system get deployment cluster-autoscaler -o yaml | grep -A5 "node-group-auto-discovery"

# Check for scale-up backoff
kubectl -n kube-system logs -l app=cluster-autoscaler | grep "backoff"

# Check pending pod constraints (nodeSelector, affinity, tolerations)
kubectl get pods --all-namespaces --field-selector status.phase=Pending -o json | \
  jq '.items[] | {name: .metadata.name, nodeSelector: .spec.nodeSelector, affinity: .spec.affinity, tolerations: .spec.tolerations}'

# Compare DaemonSet requests vs actual usage on a node
NODE="<node-name>"
kubectl get pods --all-namespaces --field-selector spec.nodeName=$NODE -o json | \
  jq -r '.items[] | select(.metadata.ownerReferences[]?.kind == "DaemonSet") | "\(.metadata.namespace) \(.metadata.name)"' | \
  while read ns name; do
    actual=$(kubectl top pod $name -n $ns --no-headers 2>/dev/null | awk '{print $3}')
    echo "$ns/$name actual=${actual:-unknown}"
  done
```

## Key Takeaways

1. The autoscaler **knows about DaemonSets** but uses their resource **requests**, not actual usage
2. Node `allocatable` does **not** include DaemonSet overhead — you must subtract it yourself to understand true available capacity
3. The `most-pods` expander picks the node group that fits the most pods per new node
4. Overprovisioning pods at priority `-1` are above the default cutoff (`-10`) and will trigger scale-up
5. **Common blocker**: overprovisioning pod requests more memory than is available on a new node after DaemonSet overhead
6. Fix by reducing the pod's memory request or using larger instance types
7. Use the combined commands to calculate exact available capacity before setting overprovisioning requests
