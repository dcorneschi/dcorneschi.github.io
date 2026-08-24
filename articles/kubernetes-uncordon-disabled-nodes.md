# Uncordon Disabled Nodes in Kubernetes

How to find and re-enable cordoned (scheduling-disabled) nodes in EKS or any Kubernetes cluster.

## What Cordoned Means

A cordoned node has `spec.unschedulable: true` set. Existing pods continue running, but no new pods will be scheduled on it. The node shows `SchedulingDisabled` in its status.

```
NAME                          STATUS                     ROLES    AGE
ip-10-0-1-50.ec2.internal    Ready,SchedulingDisabled   <none>   5d
ip-10-0-2-30.ec2.internal    Ready                      <none>   5d
```

## Find All Cordoned Nodes

```bash
# Nodes with SchedulingDisabled status
kubectl get nodes | grep SchedulingDisabled

# Using field selector
kubectl get nodes -o json | jq -r '.items[] | select(.spec.unschedulable==true) | .metadata.name'

# With more context (age, instance type, AZ)
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.conditions[-1].type,\
SCHEDULABLE:.spec.unschedulable,\
AGE:.metadata.creationTimestamp | grep -v "<none>"
```

## Uncordon a Single Node

```bash
kubectl uncordon <node-name>
```

Verify:

```bash
kubectl get node <node-name>
# STATUS should show "Ready" without "SchedulingDisabled"
```

## Uncordon All Disabled Nodes

```bash
# One-liner: uncordon all cordoned nodes
kubectl get nodes -o json | jq -r '.items[] | select(.spec.unschedulable==true) | .metadata.name' | xargs -I {} kubectl uncordon {}
```

Or with a loop for visibility:

```bash
for node in $(kubectl get nodes -o json | jq -r '.items[] | select(.spec.unschedulable==true) | .metadata.name'); do
  echo "Uncordoning $node..."
  kubectl uncordon "$node"
done
```

## Why Nodes Get Cordoned

| Cause | How It Happens | Auto-Uncordons? |
|-------|---------------|-----------------|
| Manual cordon | `kubectl cordon <node>` | No — must manually uncordon |
| Node drain | `kubectl drain <node>` (cordons before draining) | No — must manually uncordon |
| Cluster Autoscaler scale-down | CA cordons before terminating | Node is terminated (no uncordon needed) |
| EKS managed node group update | Rolling update cordons old nodes | Old node terminated, new node joins Ready |
| Karpenter disruption | Cordons before consolidation/expiry | Node is terminated |
| Spot interruption handler | Cordons on 2-min warning | Node is terminated |
| Failed drain (stuck) | Drain started but didn't complete | No — node stays cordoned |
| Maintenance scripts | Automated tooling cordons for patching | Depends on the tool |

## Common Scenario: Stuck After Failed Drain

A drain that didn't complete (PDB blocked eviction, pod stuck terminating) leaves the node cordoned:

```bash
# Check if the drain is still in progress (pods still terminating)
kubectl get pods --field-selector spec.nodeName=<node-name> -A

# If no pods are stuck, just uncordon
kubectl uncordon <node-name>

# If pods are stuck terminating, force delete them first
kubectl delete pod <stuck-pod> -n <namespace> --grace-period=0 --force
kubectl uncordon <node-name>
```

## EKS-Specific: Nodes Cordoned After Update

During a managed node group rolling update, if the update fails midway, some nodes may remain cordoned:

```bash
# Find cordoned nodes in the node group
kubectl get nodes -l eks.amazonaws.com/nodegroup=<nodegroup-name> | grep SchedulingDisabled

# Uncordon them
kubectl get nodes -l eks.amazonaws.com/nodegroup=<nodegroup-name> -o json | \
  jq -r '.items[] | select(.spec.unschedulable==true) | .metadata.name' | \
  xargs -I {} kubectl uncordon {}
```

Check node group update status:

```bash
aws eks describe-update --name <cluster-name> --update-id <update-id> --nodegroup-name <nodegroup-name>
```

## Verify Pods Are Scheduling Again

After uncordoning, verify the node is accepting new pods:

```bash
# Check node conditions
kubectl describe node <node-name> | grep -A5 "Conditions"

# Check if new pods land on the node
kubectl get pods -A --field-selector spec.nodeName=<node-name> --sort-by=.metadata.creationTimestamp | tail -5
```

## Prevent Accidental Cordoning

### Annotate Critical Nodes

The Cluster Autoscaler respects this annotation:

```bash
kubectl annotate node <node-name> cluster-autoscaler.kubernetes.io/scale-down-disabled=true
```

### Monitor for Cordoned Nodes

```bash
# Quick check — should return nothing in normal operation
kubectl get nodes | grep SchedulingDisabled

# Prometheus alert (if using kube-state-metrics)
# kube_node_spec_unschedulable == 1
```

### Datadog Monitor

```
avg(last_5m):sum:kubernetes.nodes.by_condition{condition:schedulable,status:false,kube_cluster_name:my-cluster} > 0
```

## Cordon vs Drain vs Delete

| Action | What It Does | Pods Affected |
|--------|-------------|---------------|
| `kubectl cordon` | Marks node unschedulable | None — existing pods keep running |
| `kubectl drain` | Cordons + evicts all pods (respects PDBs) | All pods moved off the node |
| `kubectl delete node` | Removes node from cluster | Pods become orphaned (rescheduled by controllers) |

```bash
# Cordon only (safe — no pod disruption)
kubectl cordon <node-name>

# Drain (evicts pods gracefully)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Drain with timeout (don't wait forever for stuck pods)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --timeout=120s

# Uncordon (re-enable scheduling)
kubectl uncordon <node-name>
```
