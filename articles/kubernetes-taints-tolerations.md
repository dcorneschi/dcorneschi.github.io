# Kubernetes Taints and Tolerations

Taints repel pods from nodes. Tolerations allow pods through. Together they control which workloads run where.

## Taint Structure

```
kubectl taint nodes <node> <key>=<value>:<effect>
                           ↑     ↑       ↑
                       REQUIRED  OPTIONAL  REQUIRED
```

### All Valid Syntax Combinations

```sh
# Key + Value + Effect (full taint)
kubectl taint nodes node1 workload=database:NoSchedule

# Key + Empty Value + Effect
kubectl taint nodes node1 maintenance=:NoSchedule

# Key + Effect (no value shorthand — most common for boolean taints)
kubectl taint nodes node1 maintenance:NoSchedule

# ❌ Key only (INVALID — no effect)
kubectl taint nodes node1 maintenance
# Error: at least one taint update is required

# ❌ Effect only (INVALID — no key)
kubectl taint nodes node1 :NoSchedule
# Error: invalid taint spec
```

| Field | Required | Rules |
|-------|:--------:|-------|
| **Key** | Yes | Must start with letter/number, max 253 chars, can have DNS prefix |
| **Value** | No | Defaults to empty string if omitted, max 63 chars |
| **Effect** | Yes | Must be `NoSchedule`, `PreferNoSchedule`, or `NoExecute` |

## Taint Effects

### NoSchedule

Prevents new pods from scheduling unless they have matching tolerations.

```sh
kubectl taint nodes node1 maintenance=true:NoSchedule
```

- ✅ Existing pods continue running
- ❌ New pods without tolerations cannot schedule
- ✅ Pods with matching tolerations can schedule

**Use cases:** Maintenance windows, dedicated workload nodes, resource isolation.

### PreferNoSchedule

Soft preference — scheduler tries to avoid but will schedule if necessary.

```sh
kubectl taint nodes node1 spot-instance=true:PreferNoSchedule
```

- ✅ Existing pods continue running
- ⚠️ Scheduler tries to avoid placing new pods here
- ✅ Will schedule if no better options available

**Use cases:** Spot instances, lower-performance nodes, cost optimization.

### NoExecute

Prevents scheduling AND evicts existing pods without tolerations.

```sh
kubectl taint nodes node1 emergency=evacuation:NoExecute
```

- ❌ Existing pods without tolerations are evicted
- ❌ New pods without tolerations cannot schedule
- ✅ Pods with matching tolerations remain/can schedule

**Use cases:** Emergency node evacuation, hardware failure, immediate workload isolation.

## Taint Keys

### Simple Keys

```sh
kubectl taint nodes node1 maintenance:NoSchedule
kubectl taint nodes node1 gpu-required:NoSchedule
kubectl taint nodes node1 spot-instance:PreferNoSchedule
```

### Descriptive Keys (Key=Value)

```sh
kubectl taint nodes node1 workload=database:NoSchedule
kubectl taint nodes node1 environment=production:NoSchedule
kubectl taint nodes node1 instance-type=gpu:NoSchedule
```

### Namespaced Keys (DNS-Style)

```sh
kubectl taint nodes node1 example.com/maintenance=true:NoSchedule
kubectl taint nodes node1 mycompany.io/workload=ml:NoSchedule
```

### Built-in Kubernetes Taints

```sh
# Cordon taint (applied by kubectl cordon)
node.kubernetes.io/unschedulable:NoSchedule

# Node condition taints (applied automatically by node lifecycle controller)
node.kubernetes.io/not-ready:NoExecute
node.kubernetes.io/unreachable:NoExecute
node.kubernetes.io/disk-pressure:NoSchedule
node.kubernetes.io/memory-pressure:NoSchedule
node.kubernetes.io/pid-pressure:NoSchedule
node.kubernetes.io/network-unavailable:NoSchedule

# OS/Architecture taints
node.kubernetes.io/os=windows:NoSchedule
node.kubernetes.io/arch=arm64:NoSchedule
```

## Taint Values

Values are optional. All of these are valid:

```sh
# No value (empty)
kubectl taint nodes node1 maintenance:NoSchedule

# Boolean values
kubectl taint nodes node1 maintenance=true:NoSchedule

# Descriptive values
kubectl taint nodes node1 maintenance=scheduled:NoSchedule
kubectl taint nodes node1 workload=database:NoSchedule

# Numeric values
kubectl taint nodes node1 gpu-count=4:NoSchedule
```

## Tolerations in Pod Specs

### Exact Match (Equal Operator)

```yaml
spec:
  tolerations:
    - key: "workload"
      operator: "Equal"
      value: "database"
      effect: "NoSchedule"
```

Matches taint: `workload=database:NoSchedule`

### Key Exists (Exists Operator)

```yaml
spec:
  tolerations:
    - key: "workload"
      operator: "Exists"
      effect: "NoSchedule"
```

Matches any taint with key `workload` and effect `NoSchedule`, regardless of value.

### Matching Taints Without Values

For taints with no value (e.g., `maintenance:NoSchedule`), you can match with either:

```yaml
# Option 1: Equal with empty string
tolerations:
  - key: "maintenance"
    operator: "Equal"
    value: ""              # Empty string matches no-value taints
    effect: "NoSchedule"

# Option 2: Exists (matches any value including empty)
tolerations:
  - key: "maintenance"
    operator: "Exists"
    effect: "NoSchedule"
```

`Exists` is simpler and works whether the taint has a value or not.

### Tolerate All Effects for a Key

```yaml
spec:
  tolerations:
    - key: "workload"
      operator: "Exists"
```

Matches any taint with key `workload`, regardless of value or effect.

### Tolerate Everything (Dangerous)

```yaml
spec:
  tolerations:
    - operator: "Exists"
```

Matches ALL taints — the pod can schedule anywhere.

### NoExecute with tolerationSeconds

```yaml
spec:
  tolerations:
    - key: "node.kubernetes.io/not-ready"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 300    # Evict after 5 minutes
```

The pod tolerates the taint for 300 seconds, then gets evicted. Useful for giving pods time to gracefully shut down during node issues.

## Multiple Taints on Same Node

```sh
# Apply multiple taints
kubectl taint nodes node1 workload=database:NoSchedule
kubectl taint nodes node1 environment=production:NoSchedule
kubectl taint nodes node1 maintenance=scheduled:PreferNoSchedule

# Check result
kubectl describe node node1 | grep -A5 Taints
# Taints: workload=database:NoSchedule
#         environment=production:NoSchedule
#         maintenance=scheduled:PreferNoSchedule
```

A pod must tolerate ALL taints to be scheduled on the node. If it tolerates only some, it's rejected.

## Overwriting Taints

```sh
# Original taint
kubectl taint nodes node1 maintenance=scheduled:NoSchedule

# Overwrite with new value (same key+effect)
kubectl taint nodes node1 maintenance=emergency:NoSchedule --overwrite

# Result: maintenance=emergency:NoSchedule
```

## Removing Taints

```sh
# Remove exact taint (key=value:effect-)
kubectl taint nodes node1 maintenance=scheduled:NoSchedule-

# Remove taint by key and effect (any value)
kubectl taint nodes node1 maintenance:NoSchedule-

# Remove all taints with a key
kubectl taint nodes node1 maintenance-

# Remove ALL taints from a node (dangerous!)
kubectl patch node node1 --type='json' -p='[{"op": "remove", "path": "/spec/taints"}]'
```

## Real-World Patterns

### GPU Nodes

```sh
# Taint
kubectl taint nodes gpu-node1 nvidia.com/gpu=true:NoSchedule
```

```yaml
# Pod toleration + nodeSelector (both needed)
spec:
  nodeSelector:
    nvidia.com/gpu.present: "true"
  tolerations:
    - key: "nvidia.com/gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  containers:
    - name: training
      resources:
        limits:
          nvidia.com/gpu: 1
```

### Spot/Preemptible Instances

```sh
kubectl taint nodes spot-node1 spot-instance=true:PreferNoSchedule
```

```yaml
spec:
  tolerations:
    - key: "spot-instance"
      operator: "Exists"
      effect: "PreferNoSchedule"
```

### Environment Isolation

```sh
# Production nodes
kubectl taint nodes prod-node1 environment=production:NoSchedule

# Dev nodes
kubectl taint nodes dev-node1 environment=development:NoSchedule
```

### Maintenance Window

```sh
# Start maintenance — new pods avoid this node
kubectl taint nodes node1 maintenance=scheduled:NoSchedule

# Emergency — evict everything immediately
kubectl taint nodes node1 maintenance=emergency:NoExecute

# End maintenance — remove taint
kubectl taint nodes node1 maintenance-
```

### DaemonSets (Tolerate Everything)

DaemonSets typically need to run on all nodes including tainted ones:

```yaml
spec:
  tolerations:
    - operator: "Exists"    # Tolerate all taints
```

## Applying Taints by Label

```sh
# Taint all nodes with a specific label
kubectl taint nodes -l disktype=ssd ssd-only=true:NoSchedule

# Taint all nodes in a node group
kubectl taint nodes -l eks.amazonaws.com/nodegroup=gpu-workers gpu=true:NoSchedule
```

## Monitoring Taints

```sh
# View all node taints (custom columns)
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINT-KEYS:.spec.taints[*].key,TAINT-VALUES:.spec.taints[*].value,TAINT-EFFECTS:.spec.taints[*].effect

# JSON format
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'

# Find nodes with a specific taint
kubectl get nodes -o json | jq '.items[] | select(.spec.taints[]?.key == "maintenance") | .metadata.name'

# Find nodes with NoExecute taints
kubectl get nodes -o json | jq '.items[] | select(.spec.taints[]?.effect == "NoExecute") | .metadata.name'

# Count tainted nodes
kubectl get nodes -o json | jq '[.items[] | select(.spec.taints != null)] | length'
```

## Documentation with Annotations

```sh
# Document taint purpose
kubectl annotate node node1 taint.purpose="Dedicated for database workloads"
kubectl annotate node node1 taint.owner="database-team"
kubectl annotate node node1 taint.expiry="2024-12-31"
```

## Validation Rules

### Key Rules

- Must start with letter or number
- Can contain: letters, numbers, hyphens, dots, underscores
- Maximum 253 characters
- Can have DNS subdomain prefix with single `/`

### Value Rules

- Optional (can be empty)
- If provided, must start with letter or number
- Can contain: letters, numbers, hyphens, dots, underscores
- Maximum 63 characters

### Effect Rules

- Must be exactly one of: `NoSchedule`, `PreferNoSchedule`, `NoExecute`
- Case-sensitive
- No custom effects allowed

## Gotchas

- **Taints don't attract pods**: Taints only repel. Use nodeSelector or node affinity to direct pods TO a node, and taints to keep others OUT.
- **Pod must tolerate ALL taints**: If a node has 3 taints, the pod needs tolerations for all 3.
- **DaemonSets tolerate by default**: The DaemonSet controller automatically adds tolerations for `node.kubernetes.io/not-ready` and `unreachable` with NoExecute.
- **`kubectl cordon` adds a taint**: Cordoning applies `node.kubernetes.io/unschedulable:NoSchedule`.
- **NoExecute eviction isn't instant**: Pods get their `terminationGracePeriodSeconds` before being killed.
- **Taint key changes don't update existing tolerations**: If you change a taint's value, pods with old tolerations referencing the old value won't match.
- **Empty value vs no value**: `key=:NoSchedule` (empty value) is different from `key:NoSchedule` (no value field). Use `operator: Exists` to match both.


## NoExecute: Detailed Eviction Behavior

NoExecute is the most aggressive effect — it affects both new AND existing pods:

```sh
# Add NoExecute taint — all pods without tolerations are evicted immediately
kubectl taint nodes node1 key=value:NoExecute
```

### tolerationSeconds — Time-Limited Tolerance

```yaml
# Pod tolerates for 1 hour, then gets evicted
tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoExecute"
    tolerationSeconds: 3600
```

Without `tolerationSeconds`, the pod tolerates indefinitely (never evicted due to this taint):

```yaml
# Pod tolerates forever
tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoExecute"
    # No tolerationSeconds = indefinite
```

### Automatic NoExecute Taints

When a node becomes unreachable or not-ready, Kubernetes automatically adds NoExecute taints:

```
node.kubernetes.io/not-ready:NoExecute        — node condition Ready=False
node.kubernetes.io/unreachable:NoExecute      — node stops sending heartbeats
```

Default behavior:
- Pods without tolerations are evicted after ~5 minutes (controlled by `pod-eviction-timeout`)
- DaemonSet pods automatically tolerate these with no time limit
- You can set `tolerationSeconds` to control how long your pods wait before eviction

```yaml
# Tolerate node unreachable for 5 minutes before being evicted
tolerations:
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300
```

## What kubectl cordon Does Behind the Scenes

`kubectl cordon` marks a node as unschedulable by:

1. Setting `spec.unschedulable: true` on the node object
2. Adding the taint `node.kubernetes.io/unschedulable:NoSchedule`

```sh
# These are equivalent:
kubectl cordon node1

# Same as:
kubectl patch node node1 -p '{"spec":{"unschedulable":true}}'
```

The node shows as `SchedulingDisabled` in `kubectl get nodes`:

```sh
# Verify
kubectl get nodes
# NAME    STATUS                     ROLES    AGE   VERSION
# node1   Ready,SchedulingDisabled   <none>   10d   v1.30.0

# Check the taint
kubectl describe node node1 | grep -A 3 "Taints:"
# Taints: node.kubernetes.io/unschedulable:NoSchedule

# Check the spec
kubectl get node node1 -o jsonpath='{.spec.unschedulable}'
# true
```

### Uncordon

```sh
kubectl uncordon node1
```

Sets `spec.unschedulable` back to `false` and removes the taint.

### Cordon vs Drain

| Action | New Pods | Existing Pods | Taint Added |
|--------|:--------:|:-------------:|-------------|
| `kubectl cordon` | Blocked | Stay running | `unschedulable:NoSchedule` |
| `kubectl drain` | Blocked | Evicted (respecting PDBs) | `unschedulable:NoSchedule` |

`drain` = `cordon` + evict all pods (except DaemonSets with `--ignore-daemonsets`).

### System Pods That Tolerate the Cordon Taint

Many system components have built-in tolerations for `node.kubernetes.io/unschedulable:NoSchedule`:

- kube-proxy
- CNI pods (aws-node, calico, flannel)
- Node monitoring agents (node-exporter, datadog-agent)
- Log collectors (fluentd, fluent-bit)
- Some ingress controllers

This means `kubectl cordon` blocks workload pods but system pods can still schedule — which is usually what you want during maintenance.

### Combining Cordon and Manual Taints

You can use both together:

```sh
# Apply custom taint first (dedicate node)
kubectl taint nodes node1 workload=database:NoSchedule

# Then cordon for maintenance
kubectl cordon node1

# Node now has both taints
kubectl describe node node1 | grep -A5 Taints
# Taints: workload=database:NoSchedule
#         node.kubernetes.io/unschedulable:NoSchedule

# Uncordon removes only the cordon taint
kubectl uncordon node1
# workload=database:NoSchedule remains
```

```sh
# Cordon only (prep for maintenance, keep workloads running)
kubectl cordon node1

# Drain (evict everything before taking node offline)
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data

# After maintenance, allow scheduling again
kubectl uncordon node1
```


## Taint-Based Cordon vs Traditional Cordon

| Feature | `kubectl cordon` | Taint-based |
|---------|:----------------:|:-----------:|
| Blocks all new pods | Yes | Only pods without tolerations |
| Shows `SchedulingDisabled` | Yes | No (node still shows `Ready`) |
| System pods can schedule | No | Yes (if they have tolerations) |
| Evict existing pods | No (use drain) | Yes (with NoExecute) |
| Custom logic per workload | No | Yes (different keys/values) |

**Key limitation:** Unlike cordon, taints don't change the node STATUS column. You need custom monitoring to track tainted nodes:

```sh
# Cordon shows SchedulingDisabled
kubectl get nodes
# NAME     STATUS                     ROLES    AGE   VERSION
# node1    Ready,SchedulingDisabled   <none>   10d   v1.30.0

# Taints don't change STATUS — node still shows Ready
kubectl get nodes
# NAME     STATUS   ROLES    AGE   VERSION
# node1    Ready    <none>   10d   v1.30.0     ← tainted but not obvious!
```

Use the monitoring commands from the earlier section to track tainted nodes.

## Rolling Node Updates with Taints

```sh
#!/bin/bash
# Roll through nodes one at a time for updates
for node in $(kubectl get nodes --no-headers -o custom-columns=NAME:.metadata.name | grep -v control-plane); do
  echo "=== Updating $node ==="

  # Taint to prevent new scheduling
  kubectl taint nodes $node update=rolling:NoSchedule

  # Drain existing pods
  kubectl drain $node --ignore-daemonsets --delete-emptydir-data --timeout=120s

  # Perform update (e.g., reboot, AMI update, etc.)
  echo "Performing update on $node..."
  sleep 10  # Replace with actual update command

  # Remove taint and uncordon
  kubectl taint nodes $node update=rolling:NoSchedule-
  kubectl uncordon $node

  # Wait for node to be Ready
  kubectl wait --for=condition=Ready node/$node --timeout=300s

  echo "$node updated successfully"
done
```

## Shell Aliases for Taint Monitoring

Add to `~/.bashrc` or `~/.zshrc`:

```sh
# Quick taint overview for all nodes
alias k-node-taints='kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].type,TAINT-KEYS:.spec.taints[*].key,TAINT-EFFECTS:.spec.taints[*].effect'

# Detailed taint info for a specific node
k-taint-details() {
  if [ -z "$1" ]; then
    echo "Usage: k-taint-details <node-name>"
    kubectl get nodes --no-headers -o custom-columns=NAME:.metadata.name
    return 1
  fi
  kubectl describe node "$1" | grep -A10 "Taints:"
}

# Find nodes that are tainted (not obvious from kubectl get nodes)
alias k-tainted='kubectl get nodes -o json | jq -r ".items[] | select(.spec.taints != null) | .metadata.name + \": \" + ([.spec.taints[].key] | join(\", \"))"'
```

Example output:

```
$ k-node-taints
NAME     STATUS   TAINT-KEYS                              TAINT-EFFECTS
node1    Ready    maintenance,workload                     NoSchedule,NoSchedule
node2    Ready    <none>                                   <none>
node3    Ready    spot-instance                            PreferNoSchedule

$ k-tainted
node1: maintenance, workload
node3: spot-instance
```

## EKS-Specific Taints

| Taint | When Applied |
|-------|--------------|
| `eks.amazonaws.com/compute-type=fargate:NoSchedule` | Fargate virtual nodes |
| `karpenter.sh/disruption=disrupting:NoExecute` | Karpenter is draining the node |

## Taints + Node Affinity — Combined Pattern

Taints only repel — they don't attract pods. To dedicate nodes to specific workloads, combine taints (keep others out) with node affinity (pull target pods in):

```bash
# Taint the nodes
kubectl taint nodes -l role=ml gpu=true:NoSchedule

# Label the nodes (if not already)
kubectl label nodes -l role=ml workload=ml
```

```yaml
# Pod spec — tolerate the taint + prefer these nodes
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: workload
            operator: In
            values:
            - ml
  containers:
  - name: training
    image: ml-training:latest
```

Without node affinity, a pod with a GPU toleration could still land on a non-GPU node if one is available — the toleration just means it's *allowed* on tainted nodes, not that it's *required* to go there.

## Graceful Node Maintenance with Taints

```bash
# Step 1: Prevent new pods (soft)
kubectl taint nodes node-5 maintenance=planned:PreferNoSchedule

# Step 2: Cordon the node
kubectl cordon node-5

# Step 3: Drain existing pods
kubectl drain node-5 --ignore-daemonsets --delete-emptydir-data

# Step 4: After maintenance, reverse everything
kubectl uncordon node-5
kubectl taint nodes node-5 maintenance:PreferNoSchedule-
```

## Troubleshooting

### Pod Stuck in Pending — Taint Related

```bash
# Check events for taint-related scheduling failures
kubectl describe pod <pod-name> | grep -A 3 "Events"

# Look for: "0/3 nodes are available: 3 node(s) had taint {key=value:NoSchedule}"
```

### Verify a Pod's Tolerations

```bash
kubectl get pod <pod-name> -o jsonpath='{.spec.tolerations}' | jq .
```

### Find Pods That Would Be Evicted by a NoExecute Taint

Before applying a `NoExecute` taint, check which pods would be affected:

```bash
# List pods on the node that do NOT tolerate a specific taint
kubectl get pods --field-selector spec.nodeName=<node-name> -o json | \
  jq -r '.items[] | select(
    .spec.tolerations == null or
    (.spec.tolerations | map(select(.key == "YOUR_TAINT_KEY" and .effect == "NoExecute")) | length == 0)
  ) | .metadata.name'
```

### DaemonSet Pods Not Running on Tainted Nodes

DaemonSets need explicit tolerations for custom taints. Check what's configured:

```bash
kubectl get daemonset <name> -o jsonpath='{.spec.template.spec.tolerations}' | jq .
```

Add the required toleration to the DaemonSet spec if missing.

### Remove All Taints from a Node (Loop)

When the JSON patch approach isn't suitable:

```bash
for taint in $(kubectl get node <node-name> -o jsonpath='{.spec.taints[*].key}'); do
  kubectl taint nodes <node-name> "${taint}-"
done
```

## When to Use Taints vs Cordon

| Scenario | Use |
|----------|-----|
| Block ALL scheduling (quick and visible) | `kubectl cordon` |
| Block most pods but allow system pods through | Taint with `NoSchedule` |
| Soft preference to avoid a node | Taint with `PreferNoSchedule` |
| Evacuate all non-essential pods immediately | Taint with `NoExecute` |
| Dedicate nodes to specific workloads | Taint + toleration on target pods |
| Temporary debugging isolation | Taint (easy to add/remove) |
| Pre-maintenance drain | `kubectl drain` (cordon + evict) |
