# Cluster Autoscaler Loop: DaemonSets Preempting Overprovisioning Pods

## The Problem

A DaemonSet with higher priority can preempt an overprovisioning pod
(priority `-1`), causing the Cluster Autoscaler (CA) to enter an infinite
scale-up loop.

## The Basic Cycle

1. A new node scales up (triggered by the overprovisioning pod needing to be scheduled).
2. The DaemonSet pod lands on the node — it has higher priority, so it preempts the overprovisioning pod (priority `-1`).
3. The overprovisioning pod becomes `Pending` again.
4. CA sees a pending pod and decides to scale up a new node.
5. New node comes up → DaemonSet lands → preempts the overprovisioning pod again → repeat.

## Why It Happens

- CA treats any `Pending` pod as a signal to potentially add capacity.
- The overprovisioning pod with a `PriorityClass` of `-1` is **designed** to be evicted — that's its job.
- But CA doesn't distinguish between "intentionally sacrificial" pods and real workloads that need scheduling.
- DaemonSets are guaranteed to run on every node, so they always claim their resources first.

## How CA Calculates Scale-Up

CA doesn't care about priority values when deciding *whether* to scale up. It
only asks:

1. Is there a `Pending` pod that can't be scheduled?
2. Would adding a node from one of the configured node groups make it schedulable?

### The simulation

```text
For each node group (instance type):
  1. Take the instance type's allocatable resources
     e.g. t3.large -> ~1.9 vCPU (after kubelet/system reserved)

  2. Subtract DaemonSet pods that will land on the node
     e.g. kube-proxy 100m + CNI 250m + custom DS 500m = 850m
     -> remaining: ~1.05 vCPU

  3. Can the pending pod fit in the remaining space?
     Overprovisioning pod requests 1 CPU
     1.0 <= 1.05 -> YES -> this node group is a candidate

  4. Pick the best candidate (expander: random, least-waste, most-pods, priority)
```

CA checks whether the pod fits on a **new** node, not on existing ones. If a
fresh node would have room after DaemonSet overhead, CA scales up.

### Two outcomes

| Scenario | CA behavior |
|---|---|
| Pod fits on a new node (after DS overhead) | CA scales up |
| Pod doesn't fit even on a fresh node | CA does nothing, pod stays Pending forever |

## The DaemonSet Timing Race Condition

This is the nastiest variant. DaemonSets don't all schedule at the same instant
— there's a timing gap.

```text
Step 1: New node comes up. Allocatable: 1.9 vCPU

Step 2: DaemonSets start scheduling (not all at once)
  DS-A (priority 100, 300m) -> scheduled
  DS-B (priority 50,  400m) -> scheduled
  DS-C (priority 200, 500m) -> not yet running, still rolling out
  Used: 700m, Remaining: 1.2 vCPU

Step 3: Overprov pod (priority -1, requests 1 CPU)
  1.0 <= 1.2 -> fits -> scheduled

Step 4: DS-C (priority 200) finally arrives, needs 500m, only 200m free
  Scheduler looks for preemption victims
  Overprov pod has priority -1 -> lowest on the node -> evicted
  DS-C schedules. Remaining: 1.9 - 0.3 - 0.4 - 0.5 = 0.7 vCPU

Step 5: Overprov pod (1 CPU) is Pending again
  Can't fit here (0.7 free) or on other nodes
  CA simulates a new node -> fits after DS-A + DS-B (before DS-C arrives)
  -> scale up another node

Step 6: New node -> same thing happens -> infinite loop
```

The scheduler doesn't wait for all DaemonSets to land before scheduling other
pods. The overprovisioning pod sneaks in during the window, then gets kicked out
by a late-arriving higher-priority DaemonSet. CA's simulation doesn't model this
race — it calculates all DaemonSet overhead upfront, but reality doesn't work
that cleanly.

## When DaemonSets Use More Than the Buffer

If DaemonSets collectively consume most of the node's capacity, the
overprovisioning pod may never fit.

```text
Instance type: t3.large, allocatable ~1.9 vCPU

DaemonSets (all nodes):
  kube-proxy       100m
  CNI (Calico)     250m
  Monitoring agent 200m
  Custom DS        500m
  Security agent   300m
  ----------------------
  Total DS overhead 1.35 vCPU

Remaining for workloads: 1.9 - 1.35 = 0.55 vCPU
Overprovisioning pod requests 1 CPU
  1.0 > 0.55 -> NEVER fits on any node
```

### The sneaky case: DaemonSets without resource requests

If DaemonSets don't declare CPU requests, CA sees them as **0 CPU** overhead:

```text
CA simulation:
  DS overhead (only those with requests): 0.35 vCPU
  Overprov pod: 1 CPU
  1.0 <= 1.55 -> fits -> scale up!

Reality:
  DS actually uses 1.3 vCPU (burstable, no requests declared)
  Actual remaining: 0.25 vCPU
  Overprov pod can't schedule -> Pending -> loop
```

This is the worst variant — CA keeps scaling because its simulation is wrong.

## Sizing Formula

```text
overprov_cpu_request = node_allocatable - sum(ALL_daemonset_cpu_requests) - safety_margin

# Example:
# t3.large allocatable: 1.9 vCPU
# All DaemonSets total: 1.2 vCPU
# Safety margin:        0.1 vCPU
# Max overprov request: 0.6 vCPU
#
# If the result is <= 0, you need a bigger instance type.
```

### Alternative: multiple smaller overprovisioning pods

Instead of one large pod, use a Deployment with smaller replicas so each fits in
the leftover space after DaemonSets:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overprovisioning
spec:
  replicas: 3
  template:
    spec:
      priorityClassName: overprovisioning
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.9
          resources:
            requests:
              cpu: "200m"      # 3 x 200m = 600m total buffer
              memory: "256Mi"
```

## How to Fix It

### 1. Use `--expendable-pods-priority-cutoff`

The CA flag `--expendable-pods-priority-cutoff` (default `-10`) controls which
pods are ignored for scale-up decisions. Pods with priority **below** this
threshold are not considered.

If your overprovisioning pod has priority `-1`, CA still considers it (since
`-1 > -10`). Set the pod priority below the cutoff:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: overprovisioning
value: -11   # below the expendable-pods-priority-cutoff
globalDefault: false
description: "Overprovisioning pods — ignored by Cluster Autoscaler"
```

### 2. Annotate the overprovisioning pod

```yaml
metadata:
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
```

This marks the pod as expendable, but alone it won't fully prevent the loop.

### 3. Size the overprovisioning pod correctly

Account for ALL DaemonSet resource usage. The request must be less than
`allocatable - total DS requests`.

### 4. Always set resource requests on DaemonSets

Without requests, CA's simulation is blind and miscalculates:

```yaml
resources:
  requests:
    cpu: "300m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### 5. Use a bigger instance type if needed

```text
t3.xlarge allocatable: ~3.9 vCPU
DS overhead:            1.35 vCPU
Remaining:             2.55 vCPU  <- room for overprov + real workloads
```

## Quick Reference

| Scenario | CA behavior | Result |
|---|---|---|
| Overprov fits on new node, DS overhead accounted for | Scale up, pod schedules, buffer works | Normal |
| Overprov fits in simulation but DS arrives late and evicts it | Scale up, evict, Pending, scale up again | Loop |
| DS has no resource requests, actual usage exceeds simulation | Scale up, can't schedule, Pending, scale up again | Loop |
| Overprov doesn't fit even on a fresh node | CA does nothing | Pod stuck Pending |
| Overprov priority below expendable cutoff | CA ignores it | No scale-up for overprov |

| Setting | Default | Recommendation |
|---|---|---|
| `--expendable-pods-priority-cutoff` | `-10` | Ensure overprov pod priority is **below** this |
| Overprovisioning pod priority | `-1` (common) | Use `-11` or lower |
| `safe-to-evict` annotation | not set | Set to `"true"` on overprov pods |
| DaemonSet resource requests | varies | **Always set them** |
| Overprov pod CPU request | varies | `allocatable - all DS requests - margin` |

## Related

- [Cluster Autoscaler Scale-Up Troubleshooting](articles/kubernetes-cluster-autoscaler-scale-up-troubleshooting.md)
- [Kubernetes Cluster Autoscaler Tuning](articles/kubernetes-cluster-autoscaler-tuning.md)
- [Cluster Autoscaler on EKS](articles/eks-cluster-autoscaler-setup.md)
- [Kubernetes Resource Scheduling & Node Capacity](articles/kubernetes-resource-scheduling-node-capacity.md)

## Skills Practiced

- Recognizing a CA scale-up loop caused by DaemonSet preemption of overprovisioning pods
- Understanding how CA simulates new-node fit and why it ignores the DaemonSet scheduling race
- Sizing overprovisioning buffers against total DaemonSet requests
- Fixing loops with `--expendable-pods-priority-cutoff`, priority tuning, and DS resource requests
