<img src="/articles/images/kubernetes-logo.svg" alt="Kubernetes" width="150">

# Kubernetes PodDisruptionBudgets (PDBs)

## What Is a PodDisruptionBudget?

A PodDisruptionBudget (PDB) tells Kubernetes how many Pods from a workload must remain available (or how many can be unavailable) during **voluntary disruptions**. It acts as a contract between the application owner and the cluster operator: "you can drain nodes and scale down, but never take my service below this availability threshold."

PDBs only protect against **voluntary disruptions** — actions initiated through the Eviction API:

| Voluntary Disruptions (PDB respected) | Involuntary Disruptions (PDB ignored) |
|----------------------------------------|---------------------------------------|
| `kubectl drain` | Node hardware failure |
| Cluster autoscaler scale-down | Kubelet OOM kill |
| Karpenter node consolidation | Node pressure eviction |
| Cluster API node rotation | Kernel panic / power loss |
| Manual eviction via API | Pod preemption (best-effort only) |

## Why PDBs Matter

Without a PDB, a `kubectl drain` or cluster autoscaler event can evict **all** your Pods simultaneously, causing a complete outage:

```
WITHOUT PDB                              WITH PDB (minAvailable: 2)
═══════════                              ══════════════════════════

  kubectl drain node-1                     kubectl drain node-1
       │                                        │
       ▼                                        ▼
  ┌─────────┐                             ┌─────────┐
  │ Evict   │                             │ Evict   │
  │ Pod A ✓ │ (on node-1)                 │ Pod A ✓ │ (on node-1, 2 remain)
  │ Pod B ✓ │ (on node-1)                 │ Pod B ✗ │ (blocked! only 1 would remain)
  │ Pod C ✓ │ (on node-1)                 │         │
  └─────────┘                             └─────────┘
       │                                        │
       ▼                                        ▼
  0/3 Pods running                         2/3 Pods running
  SERVICE DOWN                             Service healthy
```

## PDB Spec Fields

A PDB uses a label selector to target Pods and defines availability with one of two fields (not both):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: default
spec:
  minAvailable: 2          # OR maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

| Field | Meaning | Accepts |
|-------|---------|---------|
| `minAvailable` | Minimum number of Pods that must remain running | Integer or percentage |
| `maxUnavailable` | Maximum number of Pods that can be down | Integer or percentage |
| `selector` | Label selector matching the Pods this PDB protects | Same syntax as Deployment selectors |

### minAvailable vs maxUnavailable

| Scenario | `minAvailable` | `maxUnavailable` | Behavior |
|----------|---------------|-----------------|----------|
| 3 replicas, want at least 2 up | `minAvailable: 2` | `maxUnavailable: 1` | Same effect |
| 5 replicas, allow 40% down | `minAvailable: "60%"` | `maxUnavailable: "40%"` | Same effect |
| Scale-friendly (HPA) | Avoid absolute numbers | `maxUnavailable: 1` | Scales naturally with replica count |

**Key insight**: `maxUnavailable` is generally preferred for workloads that scale (via HPA or manual scaling), because it adapts automatically. A `minAvailable: 2` on a 3-replica Deployment allows 1 disruption, but if you scale to 10 replicas it still only allows 1 disruption — probably not what you want.

### Percentage Rounding

- `minAvailable: "50%"` on 3 Pods → rounds **up** to 2 (at least 2 must stay)
- `maxUnavailable: "50%"` on 3 Pods → rounds **down** to 1 (at most 1 can be disrupted)

Kubernetes always rounds in the direction that is **more conservative** (protects availability).

## How PDBs Work Internally

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                        API Server                               │
  │                                                                 │
  │   POST /api/v1/namespaces/{ns}/pods/{name}/eviction             │
  │                          │                                      │
  │                          ▼                                      │
  │              ┌───────────────────────┐                          │
  │              │  Find matching PDBs   │                          │
  │              │  for target Pod       │                          │
  │              └───────────┬───────────┘                          │
  │                          │                                      │
  │                          ▼                                      │
  │              ┌───────────────────────┐                          │
  │              │  Check PDB status:    │                          │
  │              │ disruptionsAllowed>0? │                          │
  │              └─────┬───────────┬─────┘                          │
  │                    │           │                                │
  │                YES ▼           ▼ NO                             │
  │         ┌──────────────┐  ┌──────────────┐                      │
  │         │ Atomic       │  │ Reject with  │                      │
  │         │ decrement    │  │ 429 / 500    │                      │
  │         │ counter,     │  │              │                      │
  │         │ delete Pod   │  │              │                      │
  │         └──────────────┘  └──────────────┘                      │
  │                                                                 │
  └─────────────────────────────────────────────────────────────────┘
```

1. A client sends an Eviction request (not a direct DELETE).
2. The API server finds all PDBs whose selector matches the target Pod.
3. It checks `status.disruptionsAllowed` — if > 0, it atomically decrements the counter and deletes the Pod. If 0, it rejects the request.
4. The disruption controller (in kube-controller-manager) continuously recalculates `disruptionsAllowed` based on the number of healthy Pods and the PDB spec.

**The atomic decrement is critical** — it prevents race conditions where two concurrent drain operations both think they can evict the last allowed Pod.

## PDB Status Explained

```bash
kubectl get pdb my-app-pdb -o wide
```

```
NAME         MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
my-app-pdb   2               N/A               1                     5d
```

The full status object:

```yaml
status:
  currentHealthy: 3       # Pods currently in Ready condition
  desiredHealthy: 2       # From minAvailable (or computed from maxUnavailable)
  disruptionsAllowed: 1   # How many more Pods can be evicted right now
  expectedPods: 3         # Total Pods matching the selector
  conditions:
  - type: DisruptionAllowed
    status: "True"
    reason: SufficientPods
```

**Unhealthy PDB** (blocking all evictions):

```yaml
status:
  currentHealthy: 2
  desiredHealthy: 2
  disruptionsAllowed: 0   # No evictions allowed — PDB is blocking
  expectedPods: 3
  conditions:
  - type: DisruptionAllowed
    status: "False"
    reason: InsufficientPods
```

## Common Patterns

### Pattern 1: Stateless Web Service (Deployment + HPA)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-api-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: web-api
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-api
  template:
    metadata:
      labels:
        app: web-api
    spec:
      containers:
      - name: api
        image: my-api:v1
```

`maxUnavailable: 1` means at any time, at most 1 Pod can be down due to voluntary disruptions. This works well with HPA — if replicas grow to 10, you still only lose 1 at a time.

### Pattern 2: StatefulSet (Database, Message Queue)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: redis-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: redis
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7
```

For stateful workloads, `maxUnavailable: 1` ensures quorum is maintained. For a 3-node Redis Sentinel or etcd cluster, losing 1 node keeps the cluster functional.

### Pattern 3: Singleton (Single-Replica Workload)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: scheduler-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: custom-scheduler
```

**Warning**: A PDB with `minAvailable: 1` on a single-replica Deployment will **block all evictions indefinitely**. The node can never be drained. This is a common misconfiguration.

**Solutions for singletons:**
- Accept brief downtime: use `maxUnavailable: 1` instead (allows eviction)
- Run 2+ replicas with leader election
- Use `minAvailable: 0` (PDB exists for documentation but doesn't block)

### Pattern 4: Percentage-Based for Large Deployments

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: workers-pdb
spec:
  maxUnavailable: "25%"
  selector:
    matchLabels:
      app: worker
```

For a pool of 20 worker Pods, this allows 5 to be down simultaneously — enabling faster node drains while maintaining 75% capacity.

## Common Pitfalls

### 1. PDB Blocking Node Drains

The most frequent operational issue. A PDB with `disruptionsAllowed: 0` will block `kubectl drain` indefinitely (or until `--timeout` is hit).

**Causes:**
- Pods are not healthy (crashing, not ready) — `currentHealthy < desiredHealthy`
- Single-replica Deployment with `minAvailable: 1`
- Too aggressive `minAvailable` relative to current healthy replicas

**Diagnosis:**

```bash
# Check which PDBs are blocking
kubectl get pdb -A -o custom-columns=\
  NS:.metadata.namespace,\
  NAME:.metadata.name,\
  MIN:.spec.minAvailable,\
  MAX:.spec.maxUnavailable,\
  ALLOWED:.status.disruptionsAllowed,\
  HEALTHY:.status.currentHealthy,\
  DESIRED:.status.desiredHealthy,\
  EXPECTED:.status.expectedPods

# Find blocking PDBs (disruptionsAllowed = 0)
kubectl get pdb -A -o json | jq -r '
  .items[] | select(.status.disruptionsAllowed == 0) |
  "\(.metadata.namespace)/\(.metadata.name) healthy=\(.status.currentHealthy) desired=\(.status.desiredHealthy)"'
```

**Resolution:**
- Fix unhealthy Pods (check CrashLoopBackOff, readiness probe failures)
- Temporarily relax the PDB (`maxUnavailable: 1`)
- Use `kubectl drain --disable-eviction=true` to bypass PDBs (dangerous, last resort)

### 2. PDBs Don't Protect During Rollouts

Deployments use direct Pod deletion during rolling updates — they bypass PDBs entirely:

```
  Deployment rollout (maxUnavailable: 1)
  ════════════════════════════════════════

  Deployment controller:
    "I'll delete old Pod directly"     ──► PDB is NOT consulted
                                            (uses DELETE, not Eviction API)

  kubectl drain (at the same time):
    "I'll evict a Pod"                 ──► PDB IS consulted
                                            (uses Eviction API)

  Combined effect: 2 Pods down simultaneously!
  PDB's maxUnavailable: 1 is violated.
```

**Fix:** Align your Deployment's `strategy.rollingUpdate.maxUnavailable` with your PDB:

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0       # Only surge, never reduce below current count
      maxSurge: 1
```

Setting `maxUnavailable: 0` with `maxSurge: 1` means the Deployment creates a new Pod before killing an old one — achieving zero-downtime rollouts that won't conflict with PDB.

### 3. Overlapping PDBs

If multiple PDBs select the same Pod, the **most restrictive** one wins. This can cause unexpected blocking:

```bash
# Find Pods matched by multiple PDBs
kubectl get pdb -A -o json | jq -r '
  .items[] | "\(.metadata.namespace)/\(.metadata.name): \(.spec.selector.matchLabels)"'
```

Avoid overlapping selectors. Use specific labels like `app: my-app, component: api` rather than broad ones.

### 4. PDBs with No Matching Pods

A PDB whose selector matches zero Pods shows `disruptionsAllowed: 0` and blocks nothing — but it might indicate a label mismatch bug:

```bash
# Verify PDB matches expected Pods
kubectl get pods -l app=my-app -n default
kubectl get pdb my-app-pdb -n default -o jsonpath='{.status.expectedPods}'
```

### 5. `minAvailable: 100%` or `maxUnavailable: 0`

Both settings mean "no Pod can ever be voluntarily disrupted." This completely blocks node drains and autoscaler scale-downs. Only use this if you have a very specific reason and understand the operational impact.

## UnhealthyPodEvictionPolicy (Kubernetes 1.27+)

By default, unhealthy Pods (not Ready) are still protected by PDBs — they count toward `disruptionsAllowed`. This creates a deadlock: if a Pod is stuck in CrashLoopBackOff on a node you want to drain, the PDB blocks the eviction because evicting it would reduce healthy count below `desiredHealthy`.

The `unhealthyPodEvictionPolicy` field (stable in Kubernetes 1.27) solves this:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  maxUnavailable: 1
  unhealthyPodEvictionPolicy: AlwaysAllow   # or "IfHealthyBudget"
  selector:
    matchLabels:
      app: my-app
```

| Policy | Behavior |
|--------|----------|
| `IfHealthyBudget` (default) | Unhealthy Pods can be evicted only if `disruptionsAllowed > 0` for healthy Pods |
| `AlwaysAllow` | Unhealthy Pods (not Ready, not Running) can always be evicted regardless of budget |

**Recommendation:** Use `AlwaysAllow` for most workloads. There's rarely a reason to protect a CrashLoopBackOff Pod from eviction.

## PDB and Cluster Autoscaler Interaction

The cluster autoscaler uses the Eviction API and respects PDBs. This has important implications:

1. **Scale-down blocking**: If the autoscaler wants to remove a node but a PDB would be violated by evicting a Pod on that node, the scale-down is blocked.

2. **Scale-down timeout**: After `--max-graceful-termination-sec` (default 600s), the autoscaler may give up on the node.

3. **Pod scheduling**: The autoscaler won't count on being able to evict PDB-protected Pods to make room — it will scale up instead.

**Best practice:** Ensure your PDB allows at least 1 disruption under normal conditions, otherwise the autoscaler can never remove nodes running your workload.

## Quick Reference

### When to use which field

| Situation | Recommendation |
|-----------|---------------|
| Stateless, scales with HPA | `maxUnavailable: 1` |
| Stateful cluster (3+ replicas) | `maxUnavailable: 1` |
| Large pool of identical workers | `maxUnavailable: "25%"` |
| Singleton with accepted downtime | `maxUnavailable: 1` |
| Singleton must never go down | Don't use PDB alone — run 2+ replicas |

### Checklist before deploying a PDB

- [ ] Verify the selector matches the intended Pods (`kubectl get pods -l <selector>`)
- [ ] Ensure replicas > `minAvailable` (otherwise PDB always blocks)
- [ ] Align Deployment `maxUnavailable` with PDB settings
- [ ] Consider `unhealthyPodEvictionPolicy: AlwaysAllow`
- [ ] Test with `kubectl drain --dry-run=client` on a node running the workload
- [ ] Confirm the PDB doesn't block cluster autoscaler under normal conditions

### Useful commands

```bash
# List all PDBs with status
kubectl get pdb -A

# Check if a specific drain would be blocked
kubectl drain <node> --dry-run=client --ignore-daemonsets --delete-emptydir-data

# Watch PDB status in real time during a drain
kubectl get pdb -w

# Manually test eviction (without drain)
kubectl create -f - <<EOF
apiVersion: policy/v1
kind: Eviction
metadata:
  name: <pod-name>
  namespace: <namespace>
EOF

# Find Pods protected by a PDB
kubectl get pods -l $(kubectl get pdb <pdb-name> -o jsonpath='{range .spec.selector.matchLabels}{@}={end}' | sed 's/=$//')
```
