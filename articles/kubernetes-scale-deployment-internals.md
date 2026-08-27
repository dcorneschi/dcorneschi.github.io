# What Happens When You Scale a Deployment

The internal controller mechanics when you change a Deployment's replica count — how the Deployment controller updates the ReplicaSet, the ReplicaSet controller creates or deletes pods, and how the scheduler and kubelet handle the fan-out.

## High-Level Flow

```
kubectl scale deployment/app --replicas=5
        │
        ▼
┌───────────────┐     ┌────────────────────┐     ┌──────────────────┐
│   API Server  │────▶│ Deployment         │────▶│ ReplicaSet       │
│  (updates     │     │ Controller         │     │ Controller       │
│spec.replicas) │     │ (updates RS        │     │ (creates/deletes │
│               │     │  replicas)         │     │  Pods)           │
└───────────────┘     └────────────────────┘     └──────────────────┘
                                                          │
                                                          ▼
                                                 ┌──────────────────┐
                                                 │  Scheduler       │
                                                 │  (assigns nodes) │
                                                 └──────────────────┘
                                                          │
                                                          ▼
                                                 ┌──────────────────┐
                                                 │  Kubelet (per    │
                                                 │  node, starts    │
                                                 │  containers)     │
                                                 └──────────────────┘
```

## Step 1: API Server Updates the Deployment

```bash
kubectl scale deployment/app --replicas=5
# Equivalent to:
kubectl patch deployment app -p '{"spec":{"replicas":5}}'
```

The API server receives a PATCH request and updates `spec.replicas` on the Deployment object. The `metadata.generation` increments because the spec changed.

```yaml
# Before:
spec:
  replicas: 3

# After:
spec:
  replicas: 5
metadata:
  generation: 4   # Incremented
```

## Step 2: Deployment Controller Updates the ReplicaSet

The Deployment controller watches Deployments. When it detects `spec.replicas` changed (but pod template is unchanged), it:

1. Finds the **active ReplicaSet** (the one matching the current pod template hash)
2. Updates that ReplicaSet's `spec.replicas` to match the Deployment

```
┌──────────────────────────────────────────────────────────┐
│  Deployment Controller Logic (scale, not rollout)        │
│                                                          │
│  if deployment.spec.replicas != activeRS.spec.replicas:  │
│   patch activeRS.spec.replicas = deployment.spec.replicas│
│    update deployment.status                              │
│                                                          │
│  No new ReplicaSet is created (template didn't change)   │
└──────────────────────────────────────────────────────────┘
```

This is different from a rolling update — scaling only changes the replica count on the existing RS, it doesn't create a new one.

```bash
# Watch the RS replica count change:
kubectl get rs -l app=myapp -w
# NAME              DESIRED   CURRENT   READY
# myapp-6d4f7b8c9   3         3         3      ← before
# myapp-6d4f7b8c9   5         3         3      ← desired updated
# myapp-6d4f7b8c9   5         5         3      ← current updated (pods created)
# myapp-6d4f7b8c9   5         5         5      ← all ready
```

## Step 3: ReplicaSet Controller Creates/Deletes Pods

The ReplicaSet controller watches ReplicaSets. When `spec.replicas` differs from the number of owned pods:

### Scale Up (3 → 5)

```
┌────────────────────────────────────────────────────────────────┐
│  ReplicaSet Controller: Scale Up                               │
│                                                                │
│  current_pods = list pods with ownerRef = this RS              │
│  desired = rs.spec.replicas (5)                                │
│  actual = len(current_pods) (3)                                │
│  diff = desired - actual = 2                                   │
│                                                                │
│  Create 2 new pods:                                            │
│    - Use RS's pod template                                     │
│    - Set ownerReference to this RS                             │
│    - Leave spec.nodeName empty (scheduler will fill it)        │
│                                                                │
│  Pod creation is batched:                                      │
│    Batch 1: create 1 pod                                       │
│    Batch 2: create 2 pods (doubles each batch)                 │
│    Batch 3: create 4 pods                                      │
│    ...up to remaining count                                    │
│                                                                │
│  This slow-start prevents overwhelming the API server          │
│  when scaling from 0 to 1000.                                  │
└────────────────────────────────────────────────────────────────┘
```

### Scale Down (5 → 2)

```
┌────────────────────────────────────────────────────────────────┐
│  ReplicaSet Controller: Scale Down                             │
│                                                                │
│  current_pods = list pods owned by this RS (5 pods)            │
│  desired = 2                                                   │
│  excess = 5 - 2 = 3 pods to delete                             │
│                                                                │
│  Select victims (sorted by):                                   │
│    1. Pods on nodes with more pods from same RS (spread)       │
│    2. Pods that are NOT Ready (delete unhealthy first)         │
│    3. Pods with fewer restarts                                 │
│    4. Newer pods first (by creation timestamp)                 │
│                                                                │
│  Delete 3 selected pods (graceful termination applies)         │
└────────────────────────────────────────────────────────────────┘
```

### Pod Deletion Priority (Scale Down)

The ReplicaSet controller uses this priority when choosing which pods to delete:

| Priority | Delete First | Rationale |
|----------|-------------|-----------|
| 1 | Unscheduled / Pending pods | No useful work being done |
| 2 | Pods on nodes with more co-located replicas | Improve spread |
| 3 | Pods in `ContainerCreating` or not yet Ready | Less disruption |
| 4 | Pods with more restarts | Likely unhealthy |
| 5 | Newer pods (more recent creationTimestamp) | Older pods are more established |

## Step 4: Scheduler Assigns Nodes (Scale Up)

For each newly created pod (spec.nodeName is empty):

```
Scheduler picks up pod from queue
    │
    ▼
Filter: which nodes can run this pod?
    │
    ▼
Score: rank feasible nodes
    │
    ▼
Bind: set pod.spec.nodeName = chosen node
```

Multiple new pods are scheduled independently — they can land on different nodes based on resource availability and scoring.

```bash
# Watch pods getting scheduled:
kubectl get pods -l app=myapp -w
# NAME              READY   STATUS              NODE
# myapp-abc12       1/1     Running             node-1
# myapp-def34       1/1     Running             node-2
# myapp-ghi56       1/1     Running             node-1
# myapp-jkl78       0/1     ContainerCreating   node-3  ← new
# myapp-mno90       0/1     Pending             <none>  ← waiting for scheduler
```

## Step 5: Kubelet Starts Containers

On each node where new pods are assigned, the kubelet:
1. Pulls the container image (if not cached)
2. Creates the pod sandbox (CNI networking)
3. Starts containers
4. Begins readiness probes
5. Reports pod status back to API server

## Slow-Start Batch Creation

The ReplicaSet controller doesn't create all pods simultaneously. It uses **slow-start** to prevent API server overload:

```
Scale from 0 to 100:

Batch 1: create  1 pod   (total:  1)
Batch 2: create  2 pods  (total:  3)
Batch 3: create  4 pods  (total:  7)
Batch 4: create  8 pods  (total: 15)
Batch 5: create 16 pods  (total: 31)
Batch 6: create 32 pods  (total: 63)
Batch 7: create 37 pods  (total: 100) ← remaining
```

If any pod in a batch fails to create (quota exceeded, etc.), the batch size resets to 1 for the next attempt. This prevents cascading failures.

## Deployment Status During Scaling

```yaml
status:
  replicas: 5              # Total pods managed
  updatedReplicas: 5       # Pods with current template
  readyReplicas: 3         # Pods passing readiness
  availableReplicas: 3     # Pods Ready for minReadySeconds
  unavailableReplicas: 2   # Not yet available
  conditions:
  - type: Available
    status: "True"         # At least 1 pod available
  - type: Progressing
    status: "True"
    reason: ReplicaSetUpdated
```

```bash
# Watch status during scale-up:
kubectl get deployment app -w
# NAME   READY   UP-TO-DATE   AVAILABLE   AGE
# app    3/5     5            3           10m
# app    4/5     5            4           10m
# app    5/5     5            5           10m
```

## HPA-Triggered Scaling

When the Horizontal Pod Autoscaler scales a Deployment, it follows the same path:

```
HPA evaluates metrics
    │
    │ Decides: need 8 replicas (currently 3)
    ▼
HPA patches Deployment: spec.replicas = 8
    │
    ▼
Same flow as manual scaling:
  Deployment Controller → RS Controller → Scheduler → Kubelet
```

```bash
# See HPA decisions:
kubectl describe hpa <name>
# Events:
#   Normal  SuccessfulRescale  Scaled up replica count to 8
```

The HPA owns the `spec.replicas` field. Manual scaling of an HPA-managed Deployment will be overridden on the next HPA evaluation cycle (every 15s default).

## kubectl scale vs kubectl patch vs HPA

| Method | What It Does | Persists? |
|--------|-------------|-----------|
| `kubectl scale` | PATCH on `/scale` subresource | Yes |
| `kubectl patch` | PATCH on Deployment spec | Yes |
| `kubectl apply` (with different replicas) | Strategic merge patch | Yes |
| HPA | PATCH on `/scale` subresource | Yes (overrides manual) |

All methods ultimately update `spec.replicas` on the Deployment. The difference is who "owns" the field for conflict detection (managedFields).

## The /scale Subresource

`kubectl scale` uses the `/scale` subresource endpoint, not the full Deployment resource:

```
PATCH /apis/apps/v1/namespaces/default/deployments/app/scale

{
  "spec": {
    "replicas": 5
  }
}
```

This is a lightweight update — it only touches `spec.replicas`. RBAC can grant `scale` permission separately:

```yaml
rules:
- apiGroups: ["apps"]
  resources: ["deployments/scale"]
  verbs: ["get", "update", "patch"]
```

## Scaling StatefulSets

StatefulSets scale differently from Deployments:

```
Scale Up:   pods created one-at-a-time, ordered (pod-0, pod-1, pod-2...)
            Each pod must be Running+Ready before the next is created

Scale Down: pods deleted in reverse order (pod-4, pod-3, pod-2...)
            Each pod must be fully terminated before the next is deleted
```

```bash
kubectl scale statefulset/db --replicas=5
# pod-0: already running
# pod-1: already running  
# pod-2: already running
# pod-3: Creating... Running... Ready → proceed
# pod-4: Creating... Running... Ready → done
```

This guarantees ordered rollout and avoids split-brain scenarios for stateful workloads.

### Parallel StatefulSet Scaling

With `spec.podManagementPolicy: Parallel`, pods are created/deleted simultaneously (like Deployments):

```yaml
spec:
  podManagementPolicy: Parallel  # Default is "OrderedReady"
```

## Scaling with Resource Constraints

If nodes don't have enough resources for new pods:

```
Scale 3 → 10
    │
    ▼
RS Controller creates 7 pods
    │
    ▼
Scheduler:
  - 4 pods scheduled (nodes have space)
  - 3 pods stuck Pending (no node has enough CPU/memory)
    │
    ▼
Cluster Autoscaler (if enabled):
  - Detects unschedulable pods
  - Provisions new nodes
  - Pods get scheduled once nodes are Ready
```

```bash
# See which pods are Pending:
kubectl get pods -l app=myapp --field-selector status.phase=Pending

# Check why:
kubectl describe pod <pending-pod> | grep -A5 Events
```

## Complete Timeline (Scale Up 3 → 5)

```
Time ────────────────────────────────────────────────────────────────────────▶

kubectl         API Server      Deploy Ctrl    RS Controller    Scheduler   Kubelet
   │               │               │               │              │          │
   │ scale=5 ─────▶│               │               │              │          │
   │               │ patch Deploy  │               │              │          │
   │               │ replicas=5    │               │              │          │
   │ ◀── 200 OK ── │               │               │              │          │
   │               │               │               │              │          │
   │               │─ watch ──────▶│               │              │          │
   │               │               │ patch RS      │              │          │
   │               │               │ replicas=5    │              │          │
   │               │               │               │              │          │
   │               │─ watch ───────┼──────────────▶│              │          │
   │               │               │               │ create pod-4 │          │
   │               │               │               │ create pod-5 │          │
   │               │               │               │              │          │
   │               │─ watch ───────┼───────────────┼─────────────▶│          │
   │               │               │               │              │ bind     │
   │               │               │               │              │ pod-4→n1 │
   │               │               │               │              │ pod-5→n2 │
   │               │               │               │              │          │
   │               │─ watch ───────┼───────────────┼──────────────┼─────────▶│
   │               │               │               │              │          │ pull
   │               │               │               │              │          │ start
   │               │               │               │              │          │ ready
   │               │               │               │              │          │
   │               │  Deployment status: 5/5 Ready                │          │
```

## Debugging Scaling Issues

```bash
# Check current vs desired replicas:
kubectl get deployment app -o jsonpath='{.spec.replicas}' # desired
kubectl get deployment app -o jsonpath='{.status.readyReplicas}' # actual

# Check RS replica counts:
kubectl get rs -l app=myapp

# Check for failed pod creations (quota, etc.):
kubectl get events --field-selector reason=FailedCreate

# Check for Pending pods:
kubectl get pods -l app=myapp --field-selector status.phase=Pending

# Check resource quota:
kubectl describe resourcequota -n <namespace>

# Check if HPA is overriding your scale:
kubectl get hpa
kubectl describe hpa <name> | grep "current replicas"

# Controller manager logs (if pods aren't being created):
kubectl logs -n kube-system -l component=kube-controller-manager --tail=50
```

## Quick Reference

```bash
# Scale up/down:
kubectl scale deployment/app --replicas=5
kubectl scale statefulset/db --replicas=3

# Scale multiple:
kubectl scale deployment/app1 deployment/app2 --replicas=3

# Conditional scale (only if current replicas match):
kubectl scale deployment/app --current-replicas=3 --replicas=5

# Check scaling progress:
kubectl get deployment app -w
kubectl rollout status deployment/app

# Key internals:
# - Deployment controller patches RS replica count
# - RS controller creates/deletes pods (slow-start batching)
# - Scale down: unhealthy/newer pods deleted first
# - Scale up: slow-start (1, 2, 4, 8, 16... pods per batch)
# - StatefulSet: ordered by default (one at a time)
# - HPA overrides manual replicas field
```
