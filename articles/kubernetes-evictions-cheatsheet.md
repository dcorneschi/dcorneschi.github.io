# Kubernetes Pod Evictions — Summary & Cheatsheet

## Quick Reference Table

| Method | Eviction Actor | PDB Respected? | Graceful Termination? |
|--------|---------------|----------------|----------------------|
| Eviction API | Node drainers, `kubectl drain` | Yes | Yes |
| Direct Pod deletion | Workload controllers (ReplicaSet, StatefulSet), `kubectl delete` | No | Yes |
| Node pressure (soft) | kubelet | No | Yes (capped grace period) |
| Node pressure (hard) | kubelet | No | No |
| NoExecute taint | kube-controller-manager (taint manager) | No | Yes |
| Kubelet admission | kubelet | No | No |
| Priority preemption | kube-scheduler | Best effort | Yes |
| Node deletion | kube-controller-manager (podgc controller) | No | No |
| Local storage eviction | kubelet | No | Yes |

## Eviction Paths

**Overview — all eviction paths at a glance:**

```
                         ┌─────────────────────────────┐
                         │   Pod Running on Node       │
                         └──────────────┬──────────────┘
                                        │
            ┌───────────────────────────┼───────────────────────────────┐
            │                           │                               │
            ▼                           ▼                               ▼
   ┌─────────────────┐      ┌────────────────────┐         ┌─────────────────────┐
   │  API-Initiated  │      │  Kubelet-Initiated │         │ Controller-Initiated│
   └────────┬────────┘      └─────────┬──────────┘         └──────────┬──────────┘
            │                         │                               │
            ▼                         ▼                               ▼
   ┌─────────────────┐      ┌────────────────────┐         ┌─────────────────────┐
   │ Eviction API    │      │ Node Pressure      │         │ Taint Eviction      │
   │ (kubectl drain) │      │ Kubelet Admission  │         │ Pod Preemption      │
   │                 │      │ Local Storage      │         │ Node Deletion(podgc)│
   └────────┬────────┘      └─────────┬──────────┘         └──────────┬──────────┘
            │                         │                               │
            ▼                         ▼                               ▼
   ┌─────────────────┐      ┌────────────────────┐         ┌─────────────────────┐
   │ Checks PDB      │      │ NO PDB check       │         │ NO PDB check        │
   │ Atomic decrement│      │ Direct termination │         │ Direct Pod DELETE   │
   └────────┬────────┘      └─────────┬──────────┘         └──────────┬──────────┘
            │                         │                               │
            ▼                         ▼                               ▼
   ┌─────────────────┐      ┌────────────────────┐         ┌─────────────────────┐
   │ Graceful        │      │ Depends on path:   │         │ Graceful (taint/    │
   │ termination     │      │ • Hard: SIGKILL    │         │  preemption)        │
   │ (SIGTERM + wait)│      │ • Soft: capped     │         │ Forcible (node del) │
   └─────────────────┘      │ • Admission: none  │         └─────────────────────┘
                            │ • Storage: full    │
                            └────────────────────┘
```

### 1. Eviction API

**What it is:**
The Eviction API is a sub-resource on the Pod object (`POST /api/v1/namespaces/{ns}/pods/{name}/eviction`). It is the only eviction mechanism in Kubernetes that honors PodDisruptionBudgets (PDBs).

**How it works internally:**

1. A client (e.g., `kubectl drain`, cluster-autoscaler, Karpenter) sends an Eviction request to the API server.
2. The API server checks the PDB associated with the target Pod.
3. It performs an **atomic decrement** on the PDB's `status.disruptionsAllowed` counter. This ensures that concurrent eviction requests from multiple actors cannot bring availability below the configured minimum.
4. If `disruptionsAllowed > 0`, the eviction proceeds (the Pod is deleted). If not, the request is rejected with a `429 Too Many Requests` or `500` error.
5. The Pod is then deleted with its configured `terminationGracePeriodSeconds` respected.

**Critical nuance — almost nothing in Kubernetes core uses it:**

- Workload controllers like ReplicaSet, Deployment, and StatefulSet **directly delete Pods** during rollouts. They completely bypass PDBs.
- This means during a rolling update, your PDB is invisible to the controller managing the rollout.
- Deployments and StatefulSets have their own `maxUnavailable` field that is independent from PDB's `maxUnavailable`. As the application owner, you must ensure the Deployment's `maxUnavailable` does not conflict with (i.e., exceed) your PDB's allowed disruptions — otherwise you'll dip below your availability target during rollouts.

**Who actually uses the Eviction API:**

- `kubectl drain`
- cluster-autoscaler
- Karpenter
- Cluster API (CAPI)
- Custom node draining tools

**Admission webhook complexity:**

If you want tight control over Pod lifecycle via webhooks, you must handle two distinct paths with two separate webhook configurations:
- `DELETE` on Pod resources (for direct deletions)
- `CREATE` on Eviction sub-resources (for eviction API calls)

Intercepting eviction requests selectively is particularly difficult because the Eviction request body only contains the Pod name and namespace — not labels. You cannot filter by labels, meaning your webhook may need to intercept all evictions cluster-wide, creating a single point of failure.

### 2. Kubelet Node-Pressure Eviction

**What it is:**
Kubelet monitors node resource usage (memory, disk, inodes) and proactively evicts Pods when thresholds are breached, preventing total resource exhaustion that would impact all workloads.

**How it works internally:**

1. Kubelet continuously monitors node resources via cAdvisor and the node's `/proc` filesystem.
2. It compares current usage against configured eviction thresholds.
3. When a threshold is breached, kubelet ranks pods by their QoS class and resource usage to determine eviction order:
   - **BestEffort** pods (no requests/limits) are evicted first.
   - **Burstable** pods exceeding their requests are next.
   - **Guaranteed** pods (requests == limits) are evicted last.
   - Within each QoS class, pods consuming the most of the starved resource relative to their request are evicted first.

**Hard vs. Soft thresholds:**

| Aspect | Hard Threshold | Soft Threshold |
|--------|---------------|----------------|
| Grace period | None — immediate kill | Configurable (`eviction-soft-grace-period`) |
| Pod termination | `SIGKILL` immediately | `SIGTERM` sent, but grace period is **capped** at `eviction-max-pod-grace-period` (not the Pod's own `terminationGracePeriodSeconds`) |
| Default values | `memory.available < 100Mi`, `nodefs.available < 10%`, `imagefs.available < 15%` | Not set by default |

**Default kubelet eviction thresholds:**

```yaml
# KubeletConfiguration
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"
  nodefs.inodesFree: "5%"
evictionSoft:
  memory.available: "200Mi"   # example
evictionSoftGracePeriod:
  memory.available: "1m30s"   # example
evictionMaxPodGracePeriod: 30  # seconds
```

**Signals monitored:**

| Signal | Description | Eviction Threshold Trigger |
|--------|-------------|---------------------------|
| `memory.available` | Available RAM on the node | Falls below threshold |
| `nodefs.available` | Available space on the root filesystem | Falls below threshold |
| `nodefs.inodesFree` | Available inodes on root filesystem | Falls below threshold |
| `imagefs.available` | Available space on the image filesystem | Falls below threshold |
| `pid.available` | Available PIDs on the node | Falls below threshold |

**Key behavior:**

- PDBs are **never** consulted.
- Once eviction is triggered, kubelet sets the corresponding node condition (e.g., `MemoryPressure=True`, `DiskPressure=True`).
- The node is then tainted with `node.kubernetes.io/memory-pressure:NoSchedule` (or disk-pressure equivalent), preventing new pods from being scheduled.
- Eviction is local — no API-level eviction request is made, kubelet directly terminates the container runtime processes.

**Pod selection order diagram:**

```
  Node under memory pressure
  ═══════════════════════════

  Eviction Order (first to last):

  ┌─────────────────────────────────────────────────────────┐
  │  1. BestEffort Pods (no requests/limits set)            │
  │     └─ Sorted by: memory usage (highest first)          │
  ├─────────────────────────────────────────────────────────┤
  │  2. Burstable Pods exceeding their requests             │
  │     └─ Sorted by: usage relative to request             │
  │        (highest over-request ratio first)               │
  ├─────────────────────────────────────────────────────────┤
  │  3. Burstable Pods within their requests                │
  │     └─ Sorted by: priority (lowest first)               │
  ├─────────────────────────────────────────────────────────┤
  │  4. Guaranteed Pods (requests == limits)                │
  └─────────────────────────────────────────────────────────┘

  Note: Within same QoS class, lower Pod priority is evicted first.
```

**How to disable:**

```yaml
# KubeletConfiguration — effectively disable by setting impossible thresholds
evictionHard:
  memory.available: "0Mi"
  nodefs.available: "0%"
  imagefs.available: "0%"
  nodefs.inodesFree: "0%"
```

### 3. Taint-Based Eviction (NoExecute)

**What it is:**
When a node becomes unreachable or has a `NoExecute` taint applied, pods that don't tolerate the taint are evicted after a configurable delay.

**How it works internally — the full chain:**

1. **Kubelet heartbeat**: Every 10 seconds, kubelet updates a `Lease` object in the `kube-node-lease` namespace, serving as a lightweight heartbeat.

2. **Node-lifecycle controller** (in `kube-controller-manager`): Monitors node Leases. If no heartbeat is received within `node-monitor-grace-period` (default: **40 seconds**), it sets the Node's `Ready` condition to `Unknown`.

3. **Taint application**: After the condition is `Unknown`, the node-lifecycle controller adds taints:
   - `node.kubernetes.io/unreachable:NoExecute` — if the node is unreachable
   - `node.kubernetes.io/not-ready:NoExecute` — if the node is not ready

4. **Default toleration**: Kubernetes' `DefaultTolerationSeconds` admission controller automatically adds the following tolerations to every Pod (unless overridden):
   ```yaml
   tolerations:
   - key: "node.kubernetes.io/not-ready"
     operator: "Exists"
     effect: "NoExecute"
     tolerationSeconds: 300    # 5 minutes
   - key: "node.kubernetes.io/unreachable"
     operator: "Exists"
     effect: "NoExecute"
     tolerationSeconds: 300    # 5 minutes
   ```

5. **Taint eviction controller**: Once the toleration timer expires (default 5 minutes), this controller deletes the Pod from the API. It does NOT use the Eviction API — it issues a direct `DELETE`.

**Timeline of a node failure:**

```
T+0s     — kubelet stops heartbeating
T+40s    — node-lifecycle controller sets Ready=Unknown
T+40s    — NoExecute taint is applied to the node
T+340s   — (5 min toleration expires) taint eviction controller deletes pods
```

**What happens to the actual containers:**

- If kubelet is unreachable (network partition, node crash), the pod's containers may still be running on the node.
- The API entry moves to `Terminating` but stays there indefinitely because kubelet can't confirm termination.
- The Pod will only be force-deleted if someone (or a controller) explicitly issues a force delete (`gracePeriodSeconds=0`).
- This creates a "split-brain" risk: the Pod may still be running on the partitioned node while a replacement is scheduled elsewhere.

**Kubernetes 1.29+ improvement:**

The taint eviction controller was separated from the node-lifecycle controller and can now be independently disabled via feature gate. This is useful if you want to implement custom logic for handling unreachable nodes (e.g., waiting longer for stateful workloads, or verifying the node is truly gone before evicting).

**Events to watch for:**
- `TaintManagerEviction` event on the Pod indicates this eviction path was triggered.

**Timeline diagram:**

```
Time ─────────────────────────────────────────────────────────────────────────────►

│ Kubelet stops │  Node-lifecycle   │         Toleration         │ Taint eviction │
│  heartbeating │  controller acts  │          timer             │ controller     │
│               │                   │         (5 min)            │ deletes Pod    │
│               │                   │                            │                │
├───── 0s ─────►├──── +40s ────────►├────────── +300s ──────────►├── Pod gone ───►│
│               │                   │                            │                │
│  Lease not    │  Ready=Unknown    │  Pod still running,        │  Pod is        │
│  updated      │  NoExecute taint  │  toleration counting down  │  deleted       │
│               │  applied          │                            │                │
│               │                   │  ◄─── Window to recover ──►│                │
│               │                   │   (fix kubelet, restore    │                │
│               │                   │    network, etc.)          │                │
```

### 4. Kubelet Admission (The Sneaky One)

**What it is:**
Kubelet re-runs scheduling predicates (admission checks) on pods after every restart. Pods that previously passed admission can fail after a restart if node state has changed, causing immediate termination without grace period.

**How it works internally:**

1. When kube-scheduler assigns a Pod to a node, it evaluates scheduling **filters** (predicates): resource availability, node selectors, affinity rules, taints, port conflicts, etc.

2. Kubelet also runs a subset of these same filters as **admission checks** when accepting a Pod. This is a safety net — for example, you can bypass the scheduler by directly setting `spec.nodeName`, and kubelet needs to validate this.

3. **The critical problem**: Kubelet runs these admission checks not just on first admission, but **on every kubelet restart**. After a restart, kubelet starts with an empty in-memory cache — it treats every existing Pod as if it were newly arriving.

4. If a Pod fails re-admission (e.g., a node label was removed that the Pod's `nodeSelector` requires), kubelet immediately terminates the Pod.

**What kubelet admission checks include:**

| Check | What it validates |
|-------|-------------------|
| NodeSelector | Pod's `spec.nodeSelector` matches current node labels |
| NodeAffinity | Pod's `spec.affinity.nodeAffinity` rules match current node |
| Resource fit | Node has enough allocatable resources |
| Host ports | Requested host ports are not in conflict |
| Node OS | Pod's OS requirement matches the node |

**What kubelet admission does NOT check (inconsistencies):**

- `NoSchedule` taints are NOT re-evaluated. Cordoning a node (which applies a `NoSchedule` taint) and restarting kubelet will NOT evict existing pods.
- This inconsistency between what the scheduler checks and what kubelet re-checks creates confusing behavior.

**The resulting Pod state:**

```yaml
status:
  phase: Failed
  message: "Pod was rejected: Predicate NodeAffinity failed: node(s) didn't match Pod's node affinity/selector"
  reason: "NodeAffinity"
```

- This is a **terminal** and **permanent** state. Even if the node labels are restored, this specific Pod will never run again.
- The Pod entry remains in the API indefinitely until the `podgc` controller cleans it up (threshold: 12,500 failed pods by default, configurable via `--terminated-pod-gc-threshold`).

**Reproduction steps:**

```bash
# 1. Label a node
kubectl label node worker-1 role=worker

# 2. Create a pod with nodeSelector
kubectl run test --image=busybox --overrides='{"spec":{"nodeSelector":{"role":"worker"},"containers":[{"name":"test","image":"busybox","command":["sleep","3600"]}]}}'

# 3. Verify it's running
kubectl get pod test   # STATUS: Running

# 4. Remove the label
kubectl label node worker-1 role-

# 5. Pod is still running (kubelet doesn't watch labels in real-time)
kubectl get pod test   # STATUS: Running

# 6. Restart kubelet on worker-1
systemctl restart kubelet

# 7. Pod is now failed
kubectl get pod test   # STATUS: Error
kubectl describe pod test | grep Message
# Message: Pod was rejected: Predicate NodeAffinity failed
```

**Why this matters:**

- Kubelet restarts happen routinely: certificate rotation, version upgrades, crash loops, OS patches.
- Combined with any node label mutation (which might happen via automation, cluster-autoscaler annotations, etc.), this creates an unpredictable eviction path.
- Workload controllers (ReplicaSet, StatefulSet) will create replacement pods, but custom controllers must explicitly handle `phase=Failed`.

**The root cause:**

Kubelet doesn't persist the state of previously admitted pods to disk. After restart, it has no memory of what was already running and validated — so it re-validates everything from scratch against the current node state.

**Diagram:**

```
  BEFORE RESTART                          AFTER RESTART
  ══════════════                          ═════════════

  ┌──────────┐                            ┌──────────┐
  │  Node    │                            │  Node    │
  │          │                            │          │
  │ labels:  │  ─── label removed ───►    │ labels:  │
  │  role=   │      (by automation,       │  (empty) │
  │  worker  │       human, etc.)         │          │
  └──────────┘                            └──────────┘
       │                                       │
       │                                       │
  ┌──────────┐                            ┌──────────┐
  │  Pod     │                            │  Pod     │
  │          │                            │          │
  │ selector:│                            │ selector:│
  │  role=   │   ── kubelet restart ──►   │  role=   │
  │  worker  │                            │  worker  │
  │          │                            │          │
  │ STATUS:  │                            │ STATUS:  │
  │ Running ✓│                            │ Failed ✗ │
  └──────────┘                            └──────────┘
                                               │
                                               ▼
                                    "Predicate NodeAffinity
                                     failed: node(s) didn't
                                     match Pod's node
                                     affinity/selector"

                                    ► PERMANENT FAILURE
                                    ► Pod never recovers
                                    ► Cleaned at 12,500 threshold
```

### 5. Pod Preemption

**What it is:**
When kube-scheduler cannot find a suitable node for a pending Pod, it evaluates whether evicting lower-priority Pods from a node would create enough room. This is the scheduler-driven preemption mechanism.

**How it works internally:**

1. **Scheduling failure**: A Pod enters the scheduling queue but no node passes all filter plugins (not enough resources, affinity mismatch, etc.).

2. **Preemption evaluation**: The scheduler enters the preemption phase and evaluates all nodes:
   - For each node, it identifies which lower-priority Pods could be evicted to free enough resources.
   - It simulates removing those Pods and re-runs the filter plugins to confirm the pending Pod would fit.

3. **Node ranking for preemption**: Nodes are scored based on:
   - **Minimum PDB violations** — nodes where preemption causes fewer PDB violations are preferred.
   - **Fewest preemptions** — fewer evictions is better.
   - **Highest priority victims** — prefer evicting the lowest-priority pods.
   - **Shortest runtime** — among equal-priority victims, newer pods are preferred.

4. **Victim selection on the chosen node**:
   - Pods not protected by a PDB are evicted first.
   - If that's not enough, PDB-protected pods are evicted (PDB is violated).
   - Among candidates, pods are sorted by priority (lowest first), then by runtime (newest first).

5. **Eviction execution**:
   - The scheduler sets `status.nominatedNodeName` on the preempting Pod (indicating where it intends to schedule).
   - Victim Pods are deleted directly (not via Eviction API) with their `terminationGracePeriodSeconds` respected.
   - The preempting Pod waits in the queue until victims terminate and resources are freed.

**Priority classes and how they interact:**

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority  # or "Never" to disable preemption for this class
description: "High priority workloads"
```

- Higher `value` = higher priority.
- System priority classes (value >= 1 billion) are reserved for system components.
- `preemptionPolicy: Never` means this Pod has high priority for scheduling order but will NOT trigger preemption of other Pods.

**PDB interaction — "best effort":**

- The scheduler first tries to find nodes where preemption doesn't violate any PDB.
- If no such node exists, it selects the node with the fewest PDB violations.
- PDB violations are a tiebreaker, not a hard constraint — a high-priority Pod will eventually get scheduled even if it means breaking PDBs.

**Important edge case — starvation:**

- If victim Pods have long `terminationGracePeriodSeconds`, the preempting Pod waits in the queue.
- During this wait, another higher-priority Pod could "steal" the nomination by preempting additional Pods on the same node.
- The original preempting Pod may need to re-evaluate and find a new node.

**Decision flow diagram:**

```
  ┌────────────────────────────┐
  │  High-priority Pod pending │
  │  (no node available)       │
  └──────────────┬─────────────┘
                 │
                 ▼
  ┌────────────────────────────┐
  │  Evaluate ALL nodes:       │
  │  "Which lower-priority     │
  │   pods could I evict to    │
  │   make room?"              │
  └──────────────┬─────────────┘
                 │
                 ▼
  ┌────────────────────────────┐     ┌──────────────────────┐
  │  Any nodes where eviction  │─YES─►  Pick node with      │
  │  causes ZERO PDB           │     │  fewest evictions    │
  │  violations?               │     └──────────┬───────────┘
  └──────────────┬─────────────┘                │
                 │ NO                           │
                 ▼                              │
  ┌────────────────────────────┐                │
  │  Pick node with fewest     │                │
  │  PDB violations            │                │
  └──────────────┬─────────────┘                │
                 │                              │
                 ◄──────────────────────────────┘
                 │
                 ▼
  ┌────────────────────────────┐
  │  On selected node:         │
  │  1. Evict non-PDB pods     │
  │     (lowest priority first)│
  │  2. If needed, evict PDB   │
  │     pods (lowest priority) │
  └──────────────┬─────────────┘
                 │
                 ▼
  ┌────────────────────────────┐
  │  Set nominatedNodeName     │
  │  Wait for victims to       │
  │  terminate                 │
  │  Schedule preempting Pod   │
  └────────────────────────────┘
```

### 6. Node Deletion

**What it is:**
When a Node resource is deleted from the Kubernetes API, all Pods assigned to that node become orphans and are forcibly cleaned up by the pod-garbage-collector (podgc) controller.

**How it works internally:**

1. A Node resource is deleted from the API (by cloud-controller-manager, Cluster API, or manual action).
2. The **pod-garbage-collector controller** (running in kube-controller-manager) detects that Pods exist with `spec.nodeName` pointing to a non-existent Node.
3. After approximately **60 seconds**, it issues a force-delete on these orphaned Pods (`gracePeriodSeconds=0`).
4. This is equivalent to `kubectl delete pod <name> --force --grace-period=0`.

**Implications:**

- No `SIGTERM` is sent to the container (assuming the node is already gone).
- No `preStop` hooks are executed.
- PDBs are completely ignored.
- If the node is actually still running (e.g., the Node resource was accidentally deleted), containers continue running with no API representation — a ghost state.

**Who deletes nodes:**

| Actor | When |
|-------|------|
| cloud-controller-manager | Cloud VM is terminated/not found |
| Cluster API (CAPI) | Machine resource is deleted |
| cluster-autoscaler | Scale-down (after draining via Eviction API) |
| Karpenter | Node consolidation/expiry (after draining) |
| Custom tooling | Manual or automated decommissioning |

**The safe pattern:**

Well-behaved node lifecycle managers (cluster-autoscaler, Karpenter, CAPI) will:
1. Cordon the node (prevent new scheduling)
2. Drain the node using the Eviction API (respecting PDBs)
3. Wait for all Pods to terminate
4. Only then delete the Node resource

The podgc force-delete is a safety net for cases where this process wasn't followed or the node disappeared unexpectedly.

### 7. Local Storage Eviction

**What it is:**
Kubelet monitors ephemeral storage usage per Pod (container filesystem writes, logs, and emptyDir volumes) and evicts Pods that exceed their configured limits.

**How it works internally:**

1. Kubelet periodically scans (every 10 seconds by default) each Pod's ephemeral storage consumption:
   - Container writable layer usage (files written inside the container)
   - Log file sizes (`/var/log/pods/...`)
   - `emptyDir` volume usage (if `sizeLimit` is configured)

2. If a Pod's total ephemeral storage exceeds its `resources.limits.ephemeral-storage`, kubelet evicts it.

3. For `emptyDir` volumes with a `sizeLimit`, kubelet evicts the Pod if the volume exceeds the limit.

4. Eviction is graceful — kubelet sends `SIGTERM` and respects `terminationGracePeriodSeconds`.

5. The Pod moves to a terminal phase (`Failed` or `Succeeded` depending on the container exit code).

**Example manifest triggering this eviction:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: disk-filler
spec:
  restartPolicy: Always
  terminationGracePeriodSeconds: 5
  containers:
    - name: main
      image: busybox
      command:
        - sh
        - -c
        - |
          trap 'echo "TERMINATING..." >&2; exit 1' TERM
          while true; do
            dd if=/dev/zero of=/tmp/file$RANDOM bs=1M count=50 conv=fsync 2>/dev/null || true
            sleep 1
          done
      resources:
        requests:
          ephemeral-storage: "10Mi"
        limits:
          ephemeral-storage: "20Mi"
```

**Key configuration:**

```yaml
# Pod spec
resources:
  requests:
    ephemeral-storage: "1Gi"   # Used for scheduling decisions
  limits:
    ephemeral-storage: "2Gi"   # Exceeding this triggers eviction

# emptyDir with sizeLimit
volumes:
- name: scratch
  emptyDir:
    sizeLimit: "500Mi"    # Exceeding this triggers eviction
```

**Difference from node-pressure disk eviction:**

| Aspect | Local storage eviction | Node-pressure disk eviction |
|--------|----------------------|---------------------------|
| Trigger | Individual Pod exceeds its limit | Overall node disk usage exceeds threshold |
| Scope | Targets the specific offending Pod | Targets pods based on QoS ranking |
| Configuration | Per-pod `resources.limits.ephemeral-storage` | Kubelet config `evictionHard.nodefs.available` |
| Grace period | Full `terminationGracePeriodSeconds` | Hard: none, Soft: capped |

## Key Takeaways

1. **Only the Eviction API respects PDBs** — everything else deletes pods directly.
2. **Kubelet restarts can kill running pods** if node state has changed (labels, taints).
3. **Hard eviction thresholds give zero grace period** — pods are killed immediately.
4. **Deployments ignore PDBs during rollouts** — use `maxUnavailable` carefully.
5. **Node deletion is forcible** — no grace period, no PDB.
6. **Monitor failed pods** — kubelet admission failures create permanently failed pods that accumulate.
7. **Taint-based eviction has a ~5.5 minute delay** — 40s detection + 300s toleration.
8. **Preemption is best-effort on PDBs** — high-priority pods will eventually break PDBs if needed.
9. **Local storage eviction is per-pod** — unlike node-pressure which is a global threshold.

## Defensive Measures

- Set appropriate Pod Priority Classes to control preemption order.
- Configure resource requests/limits (including ephemeral-storage) to avoid node pressure and local storage evictions.
- Use PDBs, but understand their limitations (only the Eviction API honors them).
- Align Deployment `maxUnavailable` with PDB `maxUnavailable`.
- Monitor for pods in `Failed` phase due to kubelet admission.
- Consider disabling hard/soft eviction thresholds if you run custom node remediation.
- Since K8s 1.29, consider disabling the taint eviction controller for finer control.
- Avoid mutating node labels on nodes with running workloads that depend on those labels.
- Set `preemptionPolicy: Never` on pods that should never trigger preemption of others.
- Implement proper `preStop` hooks and handle `SIGTERM` gracefully in your applications.
- Monitor `TaintManagerEviction` events for visibility into taint-based evictions.
- Set the `--terminated-pod-gc-threshold` flag if 12,500 failed pods is too high for your cluster.

## Troubleshooting Guide

### Identifying the Eviction Type

When a pod disappears unexpectedly, use this decision tree:

```
Pod disappeared — what happened?
│
├── kubectl describe pod <name> → still exists?
│   │
│   ├── YES, status.phase = Failed
│   │   └─► Check status.reason:
│   │       • "Evicted" → Node pressure eviction (check status.message for resource)
│   │       • "NodeAffinity" → Kubelet admission failure
│   │       • "Preempting" → Scheduler preemption
│   │
│   ├── YES, status.phase = Terminating
│   │   └─► Check events:
│   │       • "TaintManagerEviction" → Taint-based eviction
│   │       • "Killing" with "ephemeral local storage" → Local storage eviction
│   │
│   └── NO (pod is gone from API)
│       └─► Check events on namespace:
│           • Eviction event → Eviction API (drain)
│           • No event, node also gone → Node deletion
│           • No event, node exists → Direct deletion (rollout or manual)
```

### Commands for Diagnosis

**1. Check why a pod was evicted:**

```bash
# Get pod status and reason
kubectl get pod <pod-name> -o jsonpath='{.status.phase} {.status.reason} {.status.message}'

# Get detailed eviction info
kubectl describe pod <pod-name> | grep -A5 "Status:\|Reason:\|Message:\|Events:"

# List all evicted/failed pods
kubectl get pods --all-namespaces --field-selector=status.phase=Failed

# Count failed pods per reason
kubectl get pods --all-namespaces --field-selector=status.phase=Failed \
  -o jsonpath='{range .items[*]}{.status.reason}{"\n"}{end}' | sort | uniq -c | sort -rn
```

**2. Check node conditions and pressure:**

```bash
# Check node conditions
kubectl describe node <node-name> | grep -A10 "Conditions:"

# Check for memory/disk pressure
kubectl get nodes -o custom-columns=NAME:.metadata.name,MEMORY_PRESSURE:.status.conditions[?(@.type==\"MemoryPressure\")].status,DISK_PRESSURE:.status.conditions[?(@.type==\"DiskPressure\")].status,PID_PRESSURE:.status.conditions[?(@.type==\"PIDPressure\")].status

# Check node resource usage
kubectl top nodes

# Check kubelet eviction thresholds (requires node access)
kubectl get --raw "/api/v1/nodes/<node-name>/proxy/configz" | jq '.kubeletconfig.evictionHard'
```

**3. Check taints and tolerations:**

```bash
# List all taints on a node
kubectl describe node <node-name> | grep -A5 "Taints:"

# Find nodes with NoExecute taints
kubectl get nodes -o json | jq '.items[] | select(.spec.taints[]?.effect=="NoExecute") | .metadata.name'

# Check pod tolerations
kubectl get pod <pod-name> -o jsonpath='{.spec.tolerations}' | jq .

# Check if node heartbeat is current
kubectl get lease -n kube-node-lease <node-name> -o jsonpath='{.spec.renewTime}'
```

**4. Check PDB status:**

```bash
# List all PDBs with their disruption allowance
kubectl get pdb --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,MIN_AVAILABLE:.spec.minAvailable,MAX_UNAVAILABLE:.spec.maxUnavailable,ALLOWED_DISRUPTIONS:.status.disruptionsAllowed,CURRENT_HEALTHY:.status.currentHealthy,DESIRED_HEALTHY:.status.desiredHealthy

# Check if a specific PDB is blocking evictions
kubectl describe pdb <pdb-name> -n <namespace>
```

**5. Check preemption events:**

```bash
# Find preempted pods
kubectl get events --all-namespaces --field-selector reason=Preempted

# Find pods that triggered preemption
kubectl get events --all-namespaces --field-selector reason=Preempting

# Check nominated node for pending pods
kubectl get pods --all-namespaces --field-selector=status.phase=Pending -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,PRIORITY:.spec.priority,NOMINATED_NODE:.status.nominatedNodeName
```

**6. Check kubelet admission failures:**

```bash
# Find pods failed due to admission
kubectl get pods --all-namespaces --field-selector=status.phase=Failed -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,REASON:.status.reason,MESSAGE:.status.message | grep -i "nodeaffinity\|predicate\|admission"

# Check kubelet logs for admission rejections (on the node)
journalctl -u kubelet | grep -i "predicate\|admit\|reject"
```

**7. Check local storage usage:**

```bash
# Check ephemeral storage usage per pod (requires metrics-server)
kubectl get pods -o json | jq '.items[] | {
  name: .metadata.name,
  ephemeral_limit: .spec.containers[].resources.limits["ephemeral-storage"],
  node: .spec.nodeName
}'

# Check emptyDir usage (requires exec into node or node-shell)
# On the node:
du -sh /var/lib/kubelet/pods/*/volumes/kubernetes.io~empty-dir/*

# Check for pods evicted due to storage
kubectl get events --field-selector reason=Evicted \
  | grep -i "ephemeral\|storage\|disk"
```

**8. Check node deletion events:**

```bash
# Watch for node deletions
kubectl get events --field-selector involvedObject.kind=Node,reason=DeletingNode

# Check if nodes disappeared recently
kubectl get events --all-namespaces | grep -i "node.*delet\|RemovingNode"

# Check cloud-controller-manager logs
kubectl logs -n kube-system -l component=cloud-controller-manager | grep -i "node.*delete\|remove"
```

### Common Scenarios and Fixes

#### Scenario 1: Pods keep getting evicted for memory pressure

```bash
# Diagnose
kubectl describe node <node> | grep -A5 "Allocated resources"
kubectl top pods -n <namespace> --sort-by=memory

# Fix: Set proper memory requests/limits
# Ensure sum of requests < node allocatable
# Consider using VPA (Vertical Pod Autoscaler) for right-sizing
```

**Root cause checklist:**
- Pod memory requests too low → OOM or eviction under pressure
- No memory limits set → pods classified as BestEffort, evicted first
- Memory leak in application → gradual increase until node pressure
- Too many pods scheduled on node → overcommitment

#### Scenario 2: Pods fail after kubelet restart

```bash
# Diagnose
kubectl get pods --field-selector=status.phase=Failed -o wide
kubectl describe pod <failed-pod> | grep "Message:"

# Check what changed on the node
kubectl get node <node> --show-labels
kubectl get events --field-selector involvedObject.kind=Node

# Fix: Ensure node labels are stable, or remove nodeSelector/affinity
# from pods that don't strictly need them
```

**Prevention:**
- Use `requiredDuringSchedulingIgnoredDuringExecution` affinity with caution — it's what kubelet re-checks
- Prefer `preferredDuringSchedulingIgnoredDuringExecution` which is NOT re-checked by kubelet
- Lock down node label mutation with RBAC or admission webhooks

#### Scenario 3: Taint eviction happening too fast

```bash
# Check current toleration seconds on pods
kubectl get pod <pod> -o jsonpath='{.spec.tolerations}' | jq .

# Increase toleration time for critical workloads
```

```yaml
# Add to PodSpec to tolerate unreachable for 10 minutes instead of 5
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 600
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 600
```

#### Scenario 4: Preemption evicting important workloads

```bash
# Check priority classes in use
kubectl get priorityclasses

# Check which pods have which priority
kubectl get pods --all-namespaces -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,PRIORITY:.spec.priority,CLASS:.spec.priorityClassName | sort -k3 -n
```

**Fix:**
- Assign appropriate PriorityClasses to all workloads
- Critical services should have higher priority than batch jobs
- Use `preemptionPolicy: Never` for pods that should be high-priority but shouldn't evict others

#### Scenario 5: Node deletion causing ungraceful pod termination

```bash
# Check if nodes are being drained before deletion
kubectl get events | grep -i "drain\|cordon\|evict"

# Verify your node lifecycle tooling uses proper drain
```

**Fix:**
- Ensure your node lifecycle controller drains nodes before deleting them
- Implement a pre-delete hook or finalizer on Node resources
- Use PDBs so drain operations respect availability

#### Scenario 6: Local storage evictions

```bash
# Find pods evicted for ephemeral storage
kubectl get pods --field-selector=status.phase=Failed \
  -o jsonpath='{range .items[*]}{.metadata.name} {.status.message}{"\n"}{end}' \
  | grep -i storage

# Check current disk usage on the node
kubectl exec -it <debug-pod> -- df -h
kubectl exec -it <debug-pod> -- du -sh /var/log/pods/
```

**Fix:**
- Set realistic `ephemeral-storage` limits based on observed usage
- Implement log rotation in your application
- Use persistent volumes instead of emptyDir for large scratch data
- Monitor ephemeral storage usage before it hits limits
