# Taint vs Cordon vs Drain: Keeping Pods Off a Node

Kubernetes gives you three distinct tools for controlling whether pods run on a node, and they're easy to confuse. **Taint** repels pods selectively (only those without a matching toleration). **Cordon** blocks all new scheduling unconditionally. **Drain** cordons and then evicts what's already running. Picking the right one depends on whether you need selectivity, whether existing pods should move, and whether the node is going down.

For the full mechanics behind each, see [Kubernetes Taints and Tolerations](articles/kubernetes-taints-tolerations.md) and the [Evicting Pods from Nodes guide](articles/kubectl-evict-pods-guide.md).

## Taint a Node

A taint adds a scheduling condition to the node. Only pods with a matching **toleration** can land there — everything else is repelled. This is the selective option.

```sh
kubectl taint nodes <node-name> key=value:NoSchedule
```

### Taint Effects

| Effect | New pods without toleration | Existing pods without toleration |
|--------|-----------------------------|----------------------------------|
| `NoSchedule` | Not scheduled | Left running |
| `PreferNoSchedule` | Avoided if possible (soft) | Left running |
| `NoExecute` | Not scheduled | **Evicted** |

```sh
# Remove a taint (note the trailing hyphen)
kubectl taint nodes <node-name> key=value:NoSchedule-

# Inspect taints on a node
kubectl describe node <node-name> | grep -i taint
kubectl get node <node-name> -o jsonpath='{.spec.taints}'
```

> A tainted node stays in `Ready` state — it does **not** show `SchedulingDisabled`. Reserved-capacity nodes (GPU, licensed software, dedicated tenants) are the classic use case: taint them so only the workloads that tolerate the taint schedule there.

## Cordon a Node

Cordon is an unconditional block on all new scheduling. It sets `.spec.unschedulable = true` and ignores tolerations entirely — nothing new schedules, period. Running pods are untouched.

```sh
kubectl cordon <node-name>
kubectl uncordon <node-name>   # reverse it
```

> A cordoned node shows `Ready,SchedulingDisabled`. Cordon is the "freeze this node" button — commonly used to quarantine a flaky node while you investigate, without disrupting what's already on it.

Under the hood, cordon applies a `node.kubernetes.io/unschedulable:NoSchedule` taint too, but the `.spec.unschedulable` flag is the authoritative signal and what `kubectl` reports.

## Drain a Node

Drain is cordon plus eviction. It marks the node unschedulable, then evicts the running pods (via the Eviction API, so PodDisruptionBudgets are respected). This is what you run before taking a node down.

```sh
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

- `--ignore-daemonsets` — almost always required; DaemonSet pods are per-node and drain won't proceed otherwise.
- `--delete-emptydir-data` — required if any pod uses an `emptyDir` (that data is lost).

To bring the node back into service afterward, `kubectl uncordon <node-name>` — drain does not uncordon for you.

## Choosing Between Them

| Action | `SchedulingDisabled`? | Existing pods affected? | Selective (tolerations)? | Typical use |
|--------|-----------------------|-------------------------|--------------------------|-------------|
| **Taint** | No | Only with `NoExecute` | Yes | Reserve a node for specific workloads |
| **Cordon** | Yes | No | No | Freeze a node without disrupting it |
| **Drain** | Yes | Yes (evicted) | No | Empty a node for maintenance/removal |

A quick way to decide:

- Want only *certain* pods to stay off, based on their tolerations? → **Taint**
- Want to stop *all* new pods but leave current ones running? → **Cordon**
- Want the node *empty* so you can reboot, upgrade, or delete it? → **Drain**

## Typical Maintenance Sequence

```sh
kubectl cordon <node-name>                                          # stop new pods landing
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data  # evict the rest
# ... reboot / patch / replace the node ...
kubectl uncordon <node-name>                                        # return to service
```

Cordoning first (before drain) is optional since drain cordons anyway, but doing it explicitly makes intent clear and stops new pods immediately while you prepare the drain.

## Quick Reference

```sh
# Taint
kubectl taint nodes <node> key=value:NoSchedule
kubectl taint nodes <node> key=value:NoExecute      # also evicts non-tolerating pods
kubectl taint nodes <node> key=value:NoSchedule-    # remove

# Cordon
kubectl cordon <node>
kubectl uncordon <node>

# Drain
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# Inspect
kubectl describe node <node> | grep -i taint
kubectl get nodes                                   # STATUS shows SchedulingDisabled
```
