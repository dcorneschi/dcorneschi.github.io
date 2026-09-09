# Evicting Pods from Nodes: A Practical Guide

Hands-on methods for removing pods from Kubernetes nodes — deleting individual pods, draining nodes, scaling workloads down, and cordoning. Useful when freeing resources for a stuck DaemonSet, doing node maintenance, or relieving resource pressure. For the underlying mechanics of *how* each eviction path behaves (PDB handling, grace periods, which controller acts), see the [Kubernetes Pod Evictions Cheatsheet](articles/kubernetes-evictions-cheatsheet.md).

> Deleting a pod that belongs to a controller (Deployment, ReplicaSet, StatefulSet, DaemonSet) does **not** remove the workload — the controller immediately recreates the pod, possibly on the same node. To actually reduce load, scale the workload down or cordon/drain the node so the replacement can't land back where you started.

## Method 1: Delete a Specific Pod

The simplest eviction. A controller-managed pod is recreated; a bare pod is gone for good.

```sh
# Delete a pod (recreated by its controller)
kubectl delete pod <POD_NAME> -n <NAMESPACE>

# With an explicit grace period (default is 30s)
kubectl delete pod <POD_NAME> -n <NAMESPACE> --grace-period=30

# Delete several at once
kubectl delete pods pod1 pod2 pod3 -n <NAMESPACE>
```

> Direct pod deletion bypasses PodDisruptionBudgets. For availability-sensitive workloads, prefer the Eviction API (Method 2) or `kubectl drain`, which honor PDBs.

## Method 2: Eviction API (Respects PDBs)

The Eviction API is the only path that consults PodDisruptionBudgets. It's what `kubectl drain` uses under the hood, but you can call it directly.

```sh
kubectl create -f - <<EOF
apiVersion: policy/v1
kind: Eviction
metadata:
  name: <POD_NAME>
  namespace: <NAMESPACE>
EOF
```

If the pod's PDB has no disruptions available, the request is rejected with `429 Too Many Requests` — that's the budget doing its job. Retry once other replicas are healthy.

## Method 3: Drain the Node (Evicts All Pods)

`kubectl drain` cordons the node, then evicts its pods via the Eviction API. This is the standard for node maintenance.

```sh
# Standard drain — keep DaemonSet pods, clear emptyDir data
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data

# Also evict pods not managed by a controller (bare pods)
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data --force

# Bound how long drain waits
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data --timeout=300s
```

- `--ignore-daemonsets` is almost always required — DaemonSet pods are managed per-node and drain refuses to proceed otherwise.
- `--delete-emptydir-data` is needed when any pod uses an `emptyDir`, since that data is lost on eviction.
- `--force` evicts standalone pods that no controller will recreate. Use knowingly — those pods won't come back.

## Method 4: Scale the Workload Down

The right move when the goal is to actually reduce load rather than shuffle pods around.

```sh
kubectl scale deployment  <NAME> --replicas=0 -n <NAMESPACE>
kubectl scale replicaset  <NAME> --replicas=0 -n <NAMESPACE>
kubectl scale statefulset <NAME> --replicas=0 -n <NAMESPACE>

# Restore later
kubectl scale deployment <NAME> --replicas=3 -n <NAMESPACE>
```

## Method 5: Cordon (Stop New Scheduling)

Cordon marks a node unschedulable without touching running pods — useful before maintenance or to keep new work off a troubled node.

```sh
kubectl cordon <NODE_NAME>     # no new pods scheduled here
kubectl uncordon <NODE_NAME>   # allow scheduling again
kubectl get nodes              # SchedulingDisabled shows in STATUS
```

Cordon then drain is the safe maintenance sequence: stop new pods landing, then evict the existing ones.

## Recommended Workflow: Unblock a Stuck DaemonSet

When a DaemonSet pod is Pending because a node is full:

```sh
# 1. Find the pending DaemonSet pod
kubectl get pods -A --field-selector=status.phase=Pending

# 2. Read why it can't schedule (look at Events)
kubectl describe pod <PENDING_POD> -n <NAMESPACE>

# 3. See what's occupying the target node
kubectl get pods -A -o wide --field-selector spec.nodeName=<NODE_NAME>
kubectl top pods -A --sort-by=cpu

# 4. Free room — scale down a non-critical workload (preferred over delete)
kubectl scale deployment <NON_CRITICAL> --replicas=0 -n <NAMESPACE>

# 5. Confirm the DaemonSet pod schedules
kubectl get pods -A | grep <DAEMONSET_NAME>
kubectl get daemonsets -A
```

Scaling down is preferable to deleting here: a deleted Deployment pod just reschedules and may reclaim the same space, leaving the DaemonSet stuck again.

## Common Scenarios

### Node Maintenance

```sh
kubectl cordon <NODE_NAME>
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data --grace-period=60
# ... perform maintenance ...
kubectl uncordon <NODE_NAME>
```

### Resource Pressure

```sh
kubectl top nodes
kubectl top pods -A --sort-by=cpu
kubectl scale deployment <APP_NAME> --replicas=1 -n <NAMESPACE>   # trim, don't kill
kubectl top nodes                                                # confirm relief
```

## Finding Resource-Heavy Pods

`kubectl top` requires the metrics-server to be installed.

```sh
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory
kubectl get pods -A -o wide --field-selector spec.nodeName=<NODE_NAME>
```

## Emergency: Force Deletion (Use With Caution)

Force deletion removes the pod's API object immediately with no grace period and no `preStop` hooks. If the kubelet is unreachable, the container may keep running on the node (a "ghost" pod) while a replacement starts elsewhere — risky for StatefulSets and anything with singleton semantics.

```sh
# Force-remove a stuck pod
kubectl delete pod <POD_NAME> -n <NAMESPACE> --force --grace-period=0

# Force-drain an unresponsive node
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data --force --grace-period=0
```

> `kubectl delete pods --all -n <NAMESPACE> --force --grace-period=0` wipes every pod in a namespace with no grace period. This is destructive and rarely the right tool — scale controllers down or drain nodes instead.

## Troubleshooting

### Pod Won't Terminate

Usually a lingering finalizer or an unreachable kubelet.

```sh
kubectl describe pod <POD_NAME> -n <NAMESPACE>          # check Events and finalizers
kubectl delete pod <POD_NAME> -n <NAMESPACE> --force --grace-period=0

# Last resort — clear finalizers if the object is wedged
kubectl patch pod <POD_NAME> -n <NAMESPACE> -p '{"metadata":{"finalizers":null}}'
```

### Node Won't Drain

Drain blocks on pods it can't safely evict — bare pods, PDBs at their limit, or emptyDir data.

```sh
# Preview what would happen and why it's blocked
kubectl drain <NODE_NAME> --dry-run=client --ignore-daemonsets

# Address the specific blocker it reports (bare pods → --force, emptyDir → --delete-emptydir-data)
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data --force
```

If a PDB is blocking, that's intentional — verify other replicas are healthy before overriding.

### DaemonSet Still Won't Schedule

Freeing CPU/memory isn't the only possible cause; check taints and tolerations too.

```sh
kubectl describe node <NODE_NAME> | grep -A5 Taints
kubectl describe daemonset <DAEMONSET_NAME> -n <NAMESPACE>   # tolerations, nodeSelector
kubectl describe pod <PENDING_POD> -n <NAMESPACE>            # Events explain the block
```

## Best Practices

**Before evicting**
- Check PodDisruptionBudgets so you don't break availability.
- Know the pod's controller (Deployment vs. StatefulSet vs. bare pod) so you can predict what happens next.
- Avoid evicting system-critical pods (`kube-system`).

**During eviction**
- Prefer graceful termination — let the default grace period run.
- Evict progressively (one pod at a time) for critical services.
- Watch that replacements come back healthy.

**After eviction**
- Confirm the DaemonSet or target workload actually scheduled.
- Scale temporarily-reduced workloads back up.
- Uncordon any nodes you cordoned.
- Watch node resource usage to keep the problem from recurring.

## Quick Reference

```sh
# Delete
kubectl delete pod <pod> -n <ns>
kubectl delete pod <pod> -n <ns> --force --grace-period=0   # emergency

# Drain / cordon
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>

# Scale
kubectl scale deployment <name> --replicas=0 -n <ns>
kubectl scale deployment <name> --replicas=3 -n <ns>

# Inspect
kubectl get pods -A --field-selector=status.phase=Pending
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>
kubectl top pods -A --sort-by=cpu
kubectl top nodes
```
