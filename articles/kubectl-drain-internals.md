# How kubectl drain Works Internally

The step-by-step mechanics of `kubectl drain` — how it cordons the node, uses the Eviction API, respects PodDisruptionBudgets, handles DaemonSets, and what happens when pods refuse to leave.

Note: For the race condition between pod termination and traffic routing (preStop hooks, HAProxy 5xx), see the HAProxy drain article. This article focuses on the drain command's internal logic.

## High-Level Flow

```
kubectl drain node-1
        │
        ├── Step 1: Cordon the node (mark unschedulable)
        │
        ├── Step 2: List all pods on the node
        │
        ├── Step 3: Filter pods (skip DaemonSets, mirror pods)
        │
        ├── Step 4: Evict pods via Eviction API (respects PDBs)
        │
        ├── Step 5: Wait for all pods to terminate
        │
        └── Step 6: Node is drained (no workload pods remain)
```

## Step 1: Cordon

Drain starts by cordoning the node — preventing new pods from being scheduled:

```bash
# Drain always cordons first (equivalent to):
kubectl cordon node-1
```

What cordon does:
1. Sets `spec.unschedulable: true` on the node object
2. Adds taint `node.kubernetes.io/unschedulable:NoSchedule`
3. Node shows `SchedulingDisabled` in `kubectl get nodes`

```bash
# After cordon:
kubectl get nodes
# NAME     STATUS                     ROLES    AGE   VERSION
# node-1   Ready,SchedulingDisabled   <none>   10d   v1.30.0
```

New pods won't land here, but existing pods are still running.

## Step 2: List Pods

kubectl lists all pods on the node:

```bash
# Equivalent query:
kubectl get pods --all-namespaces --field-selector spec.nodeName=node-1
```

## Step 3: Filter Pods

kubectl categorizes pods into groups:

```
┌────────────────────────────────────────────────────────────────┐
│  Pod Filtering                                                 │
│                                                                │
│  Will be evicted:                                              │
│    - Regular pods (Deployments, StatefulSets, Jobs, bare pods) │
│                                                                │
│  Skipped (not evicted):                                        │
│    - DaemonSet pods (--ignore-daemonsets required)             │
│    - Mirror pods (static pods managed by kubelet)              │
│    - Pods with local storage (--delete-emptydir-data required) │
│                                                                │
│  Drain fails if it finds:                                      │
│    - DaemonSet pods (without --ignore-daemonsets)              │
│    - Pods with local storage (without --delete-emptydir-data)  │
│    - Bare pods with no controller (without --force)            │
└────────────────────────────────────────────────────────────────┘
```

### Required Flags for Common Scenarios

| Flag | When Required | What It Skips/Allows |
|------|--------------|---------------------|
| `--ignore-daemonsets` | Always (DaemonSets exist on almost every node) | Skips DaemonSet-owned pods |
| `--delete-emptydir-data` | If any pod has emptyDir volumes | Allows eviction (data in emptyDir is lost) |
| `--force` | If bare pods exist (no controller) | Deletes bare pods (they won't be recreated) |
| `--disable-eviction` | If you want DELETE instead of Eviction API | Bypasses PDB checks (dangerous) |

```bash
# Typical production drain:
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# Force drain (bare pods, don't care about PDBs):
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data --force --disable-eviction
```

## Step 4: Eviction API

kubectl does NOT directly delete pods. It uses the **Eviction API**, which respects PodDisruptionBudgets:

```
┌──────────┐                    ┌───────────────┐
│  kubectl │── POST eviction ──▶│   API Server  │
│  drain   │                    │               │
│          │                    │  Checks:      │
│          │                    │ 1. PDB allows?│
│          │                    │ 2. Pod exists?│
│          │◀── 200 OK ──────── │ 3. Evict pod  │
│          │  or 429 (retry)    │               │
│          │  or 500 (blocked)  │               │
└──────────┘                    └───────────────┘
```

### Eviction Request

```
POST /api/v1/namespaces/{ns}/pods/{pod}/eviction

{
  "apiVersion": "policy/v1",
  "kind": "Eviction",
  "metadata": {
    "name": "my-pod",
    "namespace": "default"
  },
  "deleteOptions": {
    "gracePeriodSeconds": 30
  }
}
```

### Eviction vs Delete

| Method | PDB Respected | Grace Period | Use Case |
|--------|:------------:|:------------:|----------|
| Eviction API | Yes | Yes | Normal drain (default) |
| DELETE pod | No | Yes | `--disable-eviction` flag |
| DELETE --force --grace-period=0 | No | No | Emergency (orphaned pods) |

## Step 5: PDB Enforcement

The Eviction API checks PodDisruptionBudgets before allowing eviction:

```
┌────────────────────────────────────────────────────────────────┐
│  PDB Check During Drain                                        │
│                                                                │
│  PDB: minAvailable=2, selector: app=web, current healthy=3     │
│                                                                │
│  Eviction request for pod web-abc:                             │
│    healthy after eviction = 3 - 1 = 2                          │
│    minAvailable = 2                                            │
│    2 >= 2 → ALLOWED                                            │
│                                                                │
│  Eviction request for pod web-def (now healthy=2):             │
│    healthy after eviction = 2 - 1 = 1                          │
│    minAvailable = 2                                            │
│    1 < 2 → BLOCKED (429 Too Many Requests)                     │
│                                                                │
│  kubectl retries every few seconds until:                      │
│    - Another replica becomes Ready (on a different node)       │
│    - Or timeout expires                                        │
└────────────────────────────────────────────────────────────────┘
```

```bash
# Check PDBs:
kubectl get pdb -A
kubectl describe pdb my-pdb

# PDB status shows allowed disruptions:
# Allowed disruptions: 1  ← how many pods can be evicted right now
# Current healthy: 3
# Min available: 2
```

### When PDB Blocks Drain

```
kubectl drain node-1 ...
# evicting pod default/web-def
# error when evicting pods/"web-def" -n "default" (will retry after 5s):
# Cannot evict pod as it would violate the pod's disruption budget.
# evicting pod default/web-def
# (retries every 5s...)
```

Drain will retry indefinitely unless `--timeout` is set:

```bash
# Set a timeout (give up after 5 minutes):
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data --timeout=300s
```

## Step 6: Wait for Termination

After the eviction is accepted, kubectl waits for the pod to actually terminate:

```
Eviction accepted (200 OK)
    │
    ▼
Pod enters Terminating state
    │
    ├── preStop hook runs
    ├── SIGTERM sent to containers
    ├── Grace period countdown
    ├── SIGKILL (if still alive)
    │
    ▼
Pod deleted from API
    │
    ▼
kubectl confirms pod is gone, moves to next pod
```

## Complete Timeline

```
Time ──────────────────────────────────────────────────────────────────▶

kubectl drain    API Server       Node/Kubelet      Controllers
    │               │                │                │
    │ cordon ──────▶│                │                │
    │               │ unschedulable  │                │
    │               │ + taint added  │                │
    │               │                │                │
    │ list pods ───▶│                │                │
    │◀── pod list ──│                │                │
    │               │                │                │
    │ POST eviction▶│                │                │
    │ (pod-a)       │ check PDB      │                │
    │               │ PDB allows     │                │
    │◀── 200 OK ─── │                │                │
    │               │ delete pod-a   │                │
    │               │───────────────▶│                │
    │               │                │ SIGTERM        │
    │               │                │ terminates     │
    │               │                │                │
    │               │                │                │── ReplicaSet creates
    │               │                │                │   replacement on
    │               │                │                │   another node
    │               │                │                │
    │ POST eviction▶│                │                │
    │ (pod-b)       │ check PDB      │                │
    │               │ PDB blocks!    │                │
    │◀── 429 ───────│                │                │
    │               │                │                │
    │ (wait 5s)     │                │                │
    │               │                │                │── new pod Ready
    │               │                │                │   on other node
    │               │                │                │   PDB now allows
    │               │                │                │
    │ POST eviction▶│                │                │
    │ (pod-b retry) │ check PDB      │                │
    │               │ PDB allows now │                │
    │◀── 200 OK ────│                │                │
    │               │                │ terminates     │
    │               │                │                │
    │ (all pods gone)                │                │
    │ "node/node-1 drained"          │                │
```

## Drain Order

kubectl evicts pods in a specific order:

```
1. Pods not managed by a ReplicaSet/Deployment/StatefulSet (unreplicated)
2. Pods from Deployments/ReplicaSets (stateless)
3. Pods from StatefulSets (stateful — evicted last)
```

Within each group, pods are evicted concurrently (all evictions sent, then wait for all).

## DaemonSet Pods — Why They're Skipped

DaemonSet pods can't be drained because:
- They're designed to run on every (or selected) node
- Evicting them means they'd be immediately rescheduled back (unless node is cordoned)
- They typically run system-level services (monitoring, logging, CNI)

With `--ignore-daemonsets`, drain skips them. They continue running on the node even after drain completes.

```bash
# After drain with --ignore-daemonsets:
kubectl get pods --field-selector spec.nodeName=node-1
# Only DaemonSet pods remain (and mirror/static pods)
```

## Mirror Pods (Static Pods)

Mirror pods are reflections of static pods managed directly by the kubelet (not through the API server). They cannot be evicted through the API:

- kube-proxy, kube-apiserver, etcd (on control plane nodes)
- Any pod defined in `/etc/kubernetes/manifests/`

Drain always ignores mirror pods.

## What Happens to Evicted Pods

| Controller | After Eviction |
|-----------|---------------|
| Deployment/ReplicaSet | New pod created on a different node immediately |
| StatefulSet | New pod created on a different node (ordered, one at a time) |
| Job | Pod recreated (if not completed and backoffLimit not exceeded) |
| DaemonSet | Pod would be recreated on same node (that's why we skip them) |
| Bare pod (no controller) | Gone forever (not recreated) — data lost |

## Common Drain Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Stuck on PDB | PDB doesn't allow disruption (no healthy replicas elsewhere) | Scale up replicas first, or increase PDB budget |
| Timeout | Pod stuck Terminating (slow preStop, stuck finalizer) | Increase timeout, or force-delete stuck pods |
| "cannot delete Pods with local storage" | emptyDir volumes present | Add `--delete-emptydir-data` |
| "cannot delete DaemonSet-managed Pods" | DaemonSet pods on node | Add `--ignore-daemonsets` |
| "cannot delete Pods not managed by RC/RS/Job/SS" | Bare pods without controller | Add `--force` (pods won't be recreated!) |
| Drain hangs forever | PDB never satisfied, no timeout set | Set `--timeout=300s` |

## Drain in Automation (Node Group Updates)

EKS/GKE/AKS node group updates use the same drain logic:

```
1. New node launched with updated AMI/config
2. Old node cordoned
3. Pods evicted (respecting PDBs)
4. Old node terminated

If drain takes too long:
  - EKS: respects PDBs for up to 15 min, then force-drains
  - Cluster Autoscaler: gives up after 10 min, moves to next node
  - Karpenter: configurable disruption budget + grace period
```

## Graceful Drain Script

```bash
#!/bin/bash
NODE=$1
TIMEOUT=${2:-600}

echo "Draining $NODE with ${TIMEOUT}s timeout..."

# Cordon first (immediate)
kubectl cordon "$NODE"

# Drain with safety flags
if kubectl drain "$NODE" \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout="${TIMEOUT}s" \
  --pod-selector='app!=critical-singleton'; then
  echo "✓ $NODE drained successfully"
else
  echo "✗ Drain failed or timed out"
  # Check what's still there:
  kubectl get pods --field-selector spec.nodeName="$NODE" --all-namespaces
  exit 1
fi
```

## Debugging Stuck Drains

```bash
# See what pods are still on the node:
kubectl get pods -A --field-selector spec.nodeName=node-1

# Check PDB status (allowed disruptions = 0 means blocked):
kubectl get pdb -A -o custom-columns=\
  NAME:.metadata.name,\
  ALLOWED:.status.disruptionsAllowed,\
  CURRENT:.status.currentHealthy,\
  MIN:.spec.minAvailable

# Check if pod is stuck Terminating:
kubectl get pods -A --field-selector spec.nodeName=node-1 -o custom-columns=\
  NAME:.metadata.name,\
  STATUS:.status.phase,\
  DELETION:.metadata.deletionTimestamp

# Force delete a stuck pod (if node is being decommissioned):
kubectl delete pod <name> -n <ns> --grace-period=0 --force

# Check drain events:
kubectl get events --field-selector involvedObject.name=node-1
```

## Quick Reference

```bash
# Standard drain:
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# With timeout:
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data --timeout=300s

# Force (bare pods, skip PDB):
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data --force --disable-eviction

# Skip specific pods:
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data \
  --pod-selector='app!=critical-singleton'

# After maintenance, allow scheduling again:
kubectl uncordon node-1

# Drain internals:
# 1. Cordon (spec.unschedulable=true + taint)
# 2. List pods on node
# 3. Filter (skip DaemonSets, mirror pods)
# 4. Evict via Eviction API (POST /pods/{name}/eviction)
# 5. PDB checked per eviction (429 = blocked, retry)
# 6. Wait for pods to terminate
# 7. "node drained" when all evictable pods are gone

# Key: drain uses Eviction API (respects PDBs)
# --disable-eviction uses DELETE (bypasses PDBs, dangerous)
```
