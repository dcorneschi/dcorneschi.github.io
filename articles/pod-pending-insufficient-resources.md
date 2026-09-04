# Troubleshooting a Pending Pod: "Insufficient" Resources and Where the Reason Hides

A pod stuck in `Pending` almost always means the **scheduler could not find a node to place
it on**. The most common cause is that no node has enough allocatable resources (CPU,
memory, ephemeral storage, GPUs, or free pod slots) to satisfy the pod's requests.

The frustrating part: `kubectl describe pod` sometimes shows the scheduling failure clearly
in the Events, but sometimes **it does not** — the events have aged out, the scheduler
message never landed as an event, or `describe` simply shows no relevant reason. In those
cases the answer is in the pod's own status, which you can read with
`kubectl get pod -o yaml`.

---

## Step 1: Confirm it is a scheduling problem

```bash
kubectl get pod <pod> -o wide
```

If the pod is `Pending` **and the `NODE` column is empty**, it has not been scheduled —
this is a scheduler/placement problem, not a container-startup problem. (If a node *is*
assigned but the pod is still Pending, the issue is on the node — image pulls, volume
mounts, etc. — not scheduling.)

---

## Special case: describe shows a [NodeAffinity] / NodeAffinity failed error

Sometimes `kubectl describe pod` is *not* empty — it shows something like:

```
Status:   Failed
Reason:   NodeAffinity
Message:  Pod was rejected: Predicate NodeAffinity failed
```

or events referencing `NodeAffinity` / `MatchNodeSelector`. This is a **different failure
mode** from the "scheduler can't place it" case above, and it is easy to misread. There are
two situations that produce a `NodeAffinity` message:

### A. The scheduler never placed it — affinity/selector doesn't match any node

If the pod is still `Pending` with an **empty NODE column** and the message says the pod
"didn't match node affinity/selector," then no node's labels satisfy the pod's
`nodeAffinity` / `nodeSelector`. The rules are too strict or point at labels no node has.

- Check the pod's rules:

  ```bash
  kubectl get pod <pod> -o jsonpath='{.spec.nodeSelector}{"\n"}{.spec.affinity.nodeAffinity}{"\n"}'
  ```

- Check which labels nodes actually have:

  ```bash
  kubectl get nodes --show-labels
  ```

- **Fix:** correct the `nodeSelector`/`nodeAffinity` to match real node labels, or label the
  intended nodes (`kubectl label node <node> <key>=<value>`).

### B. The pod was already scheduled, then the kubelet rejected it (Status: Failed)

This is the sneaky one, and it lines up with **node reboots / recycling for patching**. Here
the pod has a **node assigned**, but its phase is `Failed` with `Reason: NodeAffinity` and a
message like `Predicate NodeAffinity failed`. What happened:

- The pod uses `requiredDuringSchedulingIgnoredDuringExecution` node affinity.
- The node was rebooted or the kubelet restarted (exactly what happens during patching).
- On restart, the kubelet re-runs its **admission** checks for pods bound to it. Due to a
  known race, the kubelet can evaluate node affinity **before** it has loaded the node
  object and its labels, decide the node "doesn't match," and reject the pod — marking it
  `Failed` / `NodeAffinity` even though nothing actually changed. See
  [kubernetes/kubernetes#100467](https://github.com/kubernetes/kubernetes/issues/100467) and
  the related `MatchNodeSelector` reports
  ([#99708](https://github.com/kubernetes/kubernetes/issues/99708),
  [#72502](https://github.com/kubernetes/kubernetes/issues/72502)).

Tell-tale signs:

- Phase is `Failed` (not `Pending`), and the pod has a `nodeName` set.
- It appeared right after a node reboot / patch cycle, often on many pods at once.
- The affinity rule is objectively still satisfiable — the node still has the labels.

**Fix / mitigation:**

- These `Failed` pods are dead and will not restart themselves. **Delete them** so their
  controller reschedules a fresh pod (which will admit cleanly):

  ```bash
  kubectl delete pod <pod>
  # or clean up all NodeAffinity-failed pods in a namespace:
  kubectl get pods --field-selector status.phase=Failed -o name \
    | xargs -r kubectl delete
  ```

  A `Deployment`/`ReplicaSet`/`StatefulSet` will recreate them; **bare pods will not** and
  must be re-applied.
- To reduce recurrence during patch rolls, drain nodes properly (cordon + `kubectl drain`)
  before rebooting so pods are evicted and rescheduled cleanly instead of being re-admitted
  after a cold kubelet restart. Keeping the kubelet/Kubernetes version current also helps,
  as the admission race has been addressed over time.

If your `NodeAffinity` error is case B, skip the resource-capacity steps below — the node
had room, the kubelet just wrongly rejected an already-placed pod.

---

## Root cause: PriorityClasses and preemption

Very often the `NodeAffinity` / `Failed` / `Pending` symptom is **downstream of pod priority
and preemption**, not a capacity or label problem in isolation. This is the case you are
most likely hitting when PriorityClasses are in play.

### How preemption produces these symptoms

When a **higher-priority** pod cannot be scheduled because the cluster is full, the scheduler
does not just wait — it looks for **lower-priority pods to preempt (evict)** on a node so the
higher-priority pod can fit. The mechanism (from the Kubernetes
[Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
docs) is:

1. A pod's priority comes from its `priorityClassName` → a `PriorityClass` `value`.
2. A pending high-priority pod triggers the scheduler to select **victim** pods with lower
   priority on a candidate node.
3. The victims are **gracefully terminated** to free their reserved requests.
4. The high-priority pod schedules onto the freed space.

The **victim** pods are the ones that show up broken:

- A preempted pod's status carries a message like `Preempted in order to admit ...` /
  `Preempted by a pod on node ...`.
- If the victim is managed by a controller, it reschedules — but if the cluster is still
  tight (e.g. during a patch roll), it lands back in `Pending` with `Insufficient ...`, or
  gets preempted again, or hits the `NodeAffinity` re-admission race described above.
- If the victim is a **bare pod** (no controller), preemption **deletes it for good**.

### Why patch rolls make this acute

During node recycling for patching, capacity temporarily shrinks (nodes are drained one or a
few at a time). Higher-priority system/critical pods keep their spots by preempting your
lower-priority workloads, which then churn through `Pending`/`Failed` states until capacity
returns. So the same patch window that produces client-side throttling and `pull QPS
exceeded` also concentrates preemption.

### Confirm priority/preemption is the cause

- **Read the priorities in play:**

  ```bash
  # This pod's priority
  kubectl get pod <pod> -o jsonpath='{.spec.priorityClassName}{" -> "}{.spec.priority}{"\n"}'

  # All priority classes and their values (higher value = higher priority)
  kubectl get priorityclass -o custom-columns=NAME:.metadata.name,VALUE:.value,DEFAULT:.globalDefault
  ```

- **Look for preemption events / messages** on the victim and on the node:

  ```bash
  kubectl get events -A --field-selector reason=Preempted -o wide
  kubectl get pod <pod> -o jsonpath='{.status.message}{"\n"}'
  ```

- **Check the preemptor.** The scheduler's `FailedScheduling` / preemption line names the
  incoming high-priority pod. A high-value PriorityClass that is broadly applied (or set as
  `globalDefault: true`) is a common culprit — it silently outranks everything without one.

### Fixes

- **Give critical-but-not-supreme workloads a sane priority.** Don't leave important pods at
  priority `0` while a high PriorityClass is `globalDefault`. Assign an appropriate
  `priorityClassName` so they are not first in line to be preempted.
- **Right-size the PriorityClass values.** A few well-separated tiers (e.g. system-critical,
  platform, standard, best-effort) are easier to reason about than many overlapping ones.
- **Use `preemptionPolicy: Never`** on a PriorityClass when you want a pod to be scheduled
  ahead of lower-priority pods in the queue **without evicting** running ones. It waits for
  room instead of preempting.
- **Add capacity / headroom.** Preemption is fundamentally a symptom of not enough room for
  everything at the requested priority. Scale the node group, or reserve headroom with
  low-priority **"balloon"/placeholder** pods (a negative-value PriorityClass) that get
  preempted first, absorbing the churn instead of your real workloads.
- **During patch rolls, slow the roll** (lower surge / max-unavailable) so capacity does not
  dip far enough to trigger preemption cascades.
- **Recreate dead bare pods.** Preemption permanently deletes pods with no controller —
  re-apply them (and consider wrapping them in a Deployment so they self-heal).

---

## Step 2: Try kubectl describe first

```bash
kubectl describe pod <pod>
```

Look at the **Events** section at the bottom. When it works, you will see the scheduler's
`FailedScheduling` message spelling out exactly why each node was rejected:

```
Warning  FailedScheduling  default-scheduler  0/6 nodes are available:
  3 Insufficient cpu, 2 Insufficient memory, 1 node(s) had untolerated taint {...}.
  preemption: 0/6 nodes are available: ...
```

That line is the gold standard — it tells you the count of nodes rejected for each reason.

### Why describe sometimes shows nothing useful

`describe` reads **Events**, and Events are not part of the pod object — they are separate,
short-lived objects with a **TTL (typically ~1 hour)**. So the reason can be missing when:

- The pod has been Pending **longer than the event TTL**, so the `FailedScheduling` events
  were garbage-collected.
- The cluster is busy and events were dropped or throttled.
- The events were emitted against a different object or lost during an apiserver restart.

When that happens, `describe` shows an empty or unhelpful Events section even though the pod
is very much unschedulable.

---

## Step 3: Read the reason from the pod status (-o yaml)

The scheduler also records the latest scheduling verdict **on the pod object itself**, in
`status.conditions`. Unlike Events, this lives on the pod and does not age out, so it is the
reliable place to look:

```bash
kubectl get pod <pod> -o yaml
```

Look at `status.conditions` for the `PodScheduled` condition:

```yaml
status:
  conditions:
    - type: PodScheduled
      status: "False"
      reason: Unschedulable
      message: '0/6 nodes are available: 3 Insufficient cpu, 2 Insufficient memory,
        1 node(s) had untolerated taint {dedicated: gpu}. preemption: 0/6 nodes are
        available: 3 No preemption victims found for incoming pod, 3 Preemption is
        not helpful for scheduling.'
  phase: Pending
```

The `message` field carries the same detail the scheduler would have put in the event —
the per-reason node counts. This is the key trick: **when `describe` is silent, the reason
is in `status.conditions[].message` of `kubectl get pod -o yaml`.**

### Pull just the message with jsonpath or jq

No need to eyeball the whole YAML:

```bash
# jsonpath
kubectl get pod <pod> -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].message}{"\n"}'
```

```bash
# jq
kubectl get pod <pod> -o json \
  | jq -r '.status.conditions[] | select(.type=="PodScheduled") | .reason + ": " + .message'
```

---

## Step 4: Decode the "Insufficient" message

The scheduler message is a summary across all nodes. Typical fragments:

- **`Insufficient cpu` / `Insufficient memory`** — the sum of pod `requests` (not limits, and
  not actual usage) does not fit in any node's remaining **allocatable** capacity.
- **`Insufficient ephemeral-storage`** — the pod requests more scratch/root disk than a node
  can offer.
- **`Insufficient <extended-resource>`** (e.g. `nvidia.com/gpu`) — no node has the requested
  device count free.
- **`Too many pods`** — nodes are at their `maxPods` limit even if CPU/memory is free.
- Mixed with non-resource reasons like **untolerated taints**, **node affinity/selector
  mismatch**, **volume/topology (zone) constraints**, or **pod (anti-)affinity** — a node can
  be rejected for any of these, so the count per reason matters.

> "Insufficient" is about **requests vs. allocatable**, not live utilization. A node showing
> 20% CPU usage can still reject a pod if existing pods have *reserved* (requested) most of
> the CPU. The scheduler packs by requests, not by what is actually being used.

---

## Step 5: Verify against node capacity

Confirm the math the scheduler did.

**See what is allocatable and what is already requested per node:**

```bash
kubectl describe node <node> | sed -n '/Allocatable:/,/Allocated resources:/p'
```

The **Allocated resources** table shows CPU/memory **Requests** and the % of allocatable
already committed. If every node's request total is near 100%, that is your `Insufficient`.

**Quick fleet-wide view of allocatable:**

```bash
kubectl get nodes -o custom-columns=\
'NODE:.metadata.name,CPU_ALLOC:.status.allocatable.cpu,MEM_ALLOC:.status.allocatable.memory,PODS:.status.allocatable.pods'
```

**What the pod is asking for:**

```bash
kubectl get pod <pod> -o jsonpath=\
'{range .spec.containers[*]}{.name}{" req="}{.resources.requests}{"\n"}{end}'
```

Compare the pod's total requests to the largest single node's free allocatable. If the pod's
request exceeds any node's *total* allocatable, it will never schedule as written — no amount
of waiting helps.

---

## Fixes

Pick based on what Step 4/5 revealed:

- **Pod requests too high for any node:** lower the pod's CPU/memory/ephemeral-storage
  `requests` to fit, or move it to a node type large enough to hold it.
- **Cluster is genuinely full:** add nodes (scale the node group), or let the **Cluster
  Autoscaler / Karpenter** provision capacity. An unschedulable pod is exactly the signal
  those tools use to scale up — if they are not reacting, check their logs.
- **`Too many pods`:** raise `maxPods` on the node config (bounded by the CNI/IP limits on
  some platforms) or use larger nodes.
- **Fragmentation (space exists but not on one node):** the total free capacity is enough but
  no single node fits the pod. Bin-pack better, use fewer/larger nodes, or reduce the request.
- **Non-resource reasons in the message:** fix the specific cause — add a matching
  `toleration` for a taint, correct `nodeSelector`/affinity, or resolve zone/topology
  constraints on the pod's volumes.
- **GPU / extended resource:** ensure the device plugin is installed and advertising capacity
  on at least one node, and that the pod requests the exact resource name.

After a change, watch it clear:

```bash
kubectl get pod <pod> -w
```

---

## TL;DR

- `Pending` with an **empty NODE column** = the scheduler could not place the pod.
- `kubectl describe pod` shows the reason **only while the `FailedScheduling` events still
  exist** (events have a ~1h TTL), so it can come up empty.
- The durable reason lives on the pod: **`kubectl get pod -o yaml` →
  `status.conditions[].message`** for the `PodScheduled: False / Unschedulable` condition.
- `Insufficient <resource>` compares pod **requests** to node **allocatable**, not live
  usage. Verify with `kubectl describe node` (Allocated resources) and fix by lowering
  requests, adding/larger nodes, or resolving the non-resource constraint in the message.
