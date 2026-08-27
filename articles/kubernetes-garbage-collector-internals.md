# How the Kubernetes Garbage Collector Works

How Kubernetes automatically deletes dependent objects — ownerReferences, cascading delete modes (foreground, background, orphan), finalizers, and the garbage collector controller's graph-based approach.

## High-Level Flow

```
kubectl delete deployment my-app
        │
        ▼
┌───────────────┐     ┌──────────────────────┐     ┌──────────────────┐
│   API Server  │────▶│  Garbage Collector   │────▶│ Delete dependent │
│  (marks owner │     │  (GC controller in   │     │ objects:         │
│  for deletion)│     │   kube-controller-   │     │ - ReplicaSets    │
│               │     │   manager)           │     │ - Pods           │
└───────────────┘     └──────────────────────┘     └──────────────────┘
```

## ownerReferences — The Dependency Graph

Every Kubernetes object can declare owners via `metadata.ownerReferences`. This forms a directed acyclic graph:

```
Deployment
    │ owns
    ▼
ReplicaSet
    │ owns
    ▼
Pod (×3)
```

```yaml
# Pod's ownerReferences (set by ReplicaSet controller):
metadata:
  ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: my-app-6d4f7b8c9
    uid: 12345-abcde-67890
    controller: true           # This is the managing controller
    blockOwnerDeletion: true   # Block owner deletion until this object is gone
```

### How ownerReferences Get Set

Controllers automatically set ownerReferences when they create child objects:

| Parent (Owner) | Child (Dependent) | Set By |
|---------------|-------------------|--------|
| Deployment | ReplicaSet | Deployment controller |
| ReplicaSet | Pod | ReplicaSet controller |
| StatefulSet | Pod | StatefulSet controller |
| Job | Pod | Job controller |
| CronJob | Job | CronJob controller |
| DaemonSet | Pod | DaemonSet controller |
| Node | CSINode | node-driver-registrar |

```bash
# See a pod's owners:
kubectl get pod <name> -o jsonpath='{.metadata.ownerReferences}' | jq .

# Find all objects owned by a ReplicaSet:
kubectl get pods -o json | jq -r '.items[] | select(.metadata.ownerReferences[]?.name == "my-app-6d4f7b8c9") | .metadata.name'

# Visualize the ownership chain:
kubectl get deployment my-app -o jsonpath='{.metadata.uid}'    # Owner UID
kubectl get rs -o json | jq -r '.items[] | select(.metadata.ownerReferences[]?.uid == "<deploy-uid>") | .metadata.name'
```

## Three Cascading Delete Modes

When you delete an owner object, Kubernetes offers three options for handling dependents:

### Background Cascading Delete (Default)

```bash
kubectl delete deployment my-app
# OR explicitly:
kubectl delete deployment my-app --cascade=background
```

```
┌────────────────────────────────────────────────────────────────┐
│  Background Delete                                             │
│                                                                │
│  1. API server deletes the Deployment immediately              │
│  2. Deployment disappears from kubectl get deployments         │
│  3. Garbage Collector (asynchronously) finds orphaned RS       │
│  4. GC deletes the ReplicaSet                                  │
│  5. GC finds orphaned Pods                                     │
│  6. GC deletes the Pods                                        │
│                                                                │
│  Owner deleted FIRST, then dependents cleaned up in background │
└────────────────────────────────────────────────────────────────┘
```

Timeline:
```
T+0s:  Deployment gone
T+1s:  GC picks up orphaned ReplicaSet, deletes it
T+2s:  GC picks up orphaned Pods, deletes them
T+30s: Pods finish graceful termination, fully gone
```

### Foreground Cascading Delete

```bash
kubectl delete deployment my-app --cascade=foreground
```

```
┌────────────────────────────────────────────────────────────────┐
│  Foreground Delete                                             │
│                                                                │
│  1. API server sets deletionTimestamp on Deployment            │
│  2. API server adds finalizer:                                 │
│     foregroundDeletion                                         │
│  3. Deployment is visible but has deletionTimestamp            │
│  4. GC deletes all dependents FIRST (RS, then Pods)            │
│  5. After ALL dependents are gone:                             │
│     GC removes the foregroundDeletion finalizer                │
│  6. API server deletes the Deployment from etcd                │
│                                                                │
│  Dependents deleted FIRST, then owner removed last             │
└────────────────────────────────────────────────────────────────┘
```

Timeline:
```
T+0s:   Deployment gets deletionTimestamp (still visible)
T+1s:   GC starts deleting Pods
T+30s:  All Pods terminated
T+31s:  GC deletes ReplicaSet
T+32s:  GC removes foregroundDeletion finalizer
T+32s:  Deployment finally removed from etcd
```

During foreground delete, the owner shows as `Terminating` but dependents are being cleaned up.

### Orphan (No Cascade)

```bash
kubectl delete deployment my-app --cascade=orphan
```

```
┌────────────────────────────────────────────────────────────────┐
│  Orphan Delete                                                 │
│                                                                │
│  1. API server deletes the Deployment                          │
│  2. GC removes ownerReferences from dependents                 │
│     (strips the ownership link)                                │
│  3. ReplicaSet and Pods continue running                       │
│     (but are now "orphaned" — no controller managing them)     │
│                                                                │
│  Owner deleted, dependents survive as standalone objects       │
└────────────────────────────────────────────────────────────────┘
```

After orphan delete:
- ReplicaSet still exists (with pods) but has no owning Deployment
- No controller will scale it up/down or roll it out
- You can adopt it later by creating a new Deployment with matching selector

### Comparison

| Mode | Owner Deletion | Dependent Deletion | Use Case |
|------|:-------------:|:-----------------:|----------|
| `background` (default) | Immediate | Async (GC cleans up) | Normal operations |
| `foreground` | After dependents gone | First (before owner) | Want to confirm cleanup is complete |
| `orphan` | Immediate | Never (orphaned) | Keep pods running during migration |

## The Garbage Collector Controller

The GC controller runs in `kube-controller-manager` and maintains an in-memory graph of all object ownership:

```
┌────────────────────────────────────────────────────────────────┐
│  Garbage Collector Controller                                  │
│                                                                │
│  Components:                                                   │
│                                                                │
│  1. Graph Builder (Scanner)                                    │
│     - Watches ALL objects in the cluster                       │
│     - Builds in-memory ownership graph                         │
│     - Detects when an owner is deleted (orphaned dependents)   │
│                                                                │
│  2. Garbage Processor (Worker)                                 │
│     - Processes items from the "dirty" queue                   │
│     - Determines if object should be deleted                   │
│     - Issues delete requests for orphaned objects              │
│                                                                │
│  Graph structure:                                              │
│    node = object (UID, kind, namespace, name)                  │
│    edge = ownerReference (parent → child)                      │
│                                                                │
│  When a node is deleted from the graph:                        │
│    - All children (dependents) become "dirty"                  │
│    - GC processor checks each dirty item                       │
│    - If all owners are gone → delete the dependent             │
│    - If some owners remain → keep (multi-owner)                │
└────────────────────────────────────────────────────────────────┘
```

### Multi-Owner Objects

An object can have multiple owners. It's only garbage collected when ALL owners are gone:

```yaml
metadata:
  ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: rs-blue
    uid: aaa-111
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: rs-green
    uid: bbb-222
```

If `rs-blue` is deleted but `rs-green` still exists, the pod is NOT garbage collected.

## Finalizers and the GC

Finalizers interact with garbage collection to control deletion timing:

```
┌────────────────────────────────────────────────────────────────┐
│  Finalizer Flow                                                │
│                                                                │
│  Object with finalizer ["example.com/cleanup"]:                │
│                                                                │
│  1. User: kubectl delete object                                │
│  2. API server: sets deletionTimestamp (NOT removed from etcd) │
│  3. Object is now "Terminating" but still exists               │
│  4. Controller sees deletionTimestamp, performs cleanup        │
│  5. Controller removes "example.com/cleanup" from finalizers   │
│  6. API server: finalizers list is empty → delete from etcd    │
│                                                                │
│  The GC uses finalizers internally:                            │
│  - "foregroundDeletion" finalizer: blocks owner deletion       │
│    until dependents are deleted                                │
│  - "orphan" finalizer: triggers ownerRef removal from          │
│    dependents before owner is deleted                          │
└────────────────────────────────────────────────────────────────┘
```

### GC-Specific Finalizers

| Finalizer | Set When | Removed When |
|-----------|----------|-------------|
| `foregroundDeletion` | Foreground cascade delete requested | All dependents are deleted |
| `orphan` | Orphan delete requested | ownerReferences stripped from all dependents |

```bash
# Check if an object has GC finalizers:
kubectl get deployment my-app -o jsonpath='{.metadata.finalizers}'
# ["foregroundDeletion"]  ← waiting for dependents to be deleted
```

## blockOwnerDeletion

The `blockOwnerDeletion: true` field in ownerReferences means: "In foreground delete mode, the owner cannot be fully deleted until I'm gone."

```yaml
ownerReferences:
- apiVersion: apps/v1
  kind: ReplicaSet
  name: my-app-rs
  uid: xyz-123
  blockOwnerDeletion: true   # Owner waits for me in foreground delete
```

Without `blockOwnerDeletion`, in foreground mode, the GC might delete the owner before all dependents are gone.

## What the GC Does NOT Do

| Not GC's Job | Who Handles It |
|-------------|---------------|
| Delete completed Pods from Jobs | Job controller (via `ttlSecondsAfterFinished`) |
| Delete old ReplicaSets | Deployment controller (via `revisionHistoryLimit`) |
| Delete old Events | API server (TTL-based, default 1 hour) |
| Clean up completed Jobs | CronJob controller (via `successfulJobsHistoryLimit`) |
| Delete PVs when PVC is gone | PV controller (reclaimPolicy, not GC) |
| Node removal | Admin or cloud controller |

The GC only handles **ownerReference-based** dependency cleanup.

## Common GC Scenarios

### Delete a Deployment (Normal)

```
kubectl delete deployment my-app
    │ (background cascade)
    ▼
Deployment deleted immediately
    │
    ▼ GC detects orphaned ReplicaSets
    │
    ▼ GC deletes ReplicaSet(s)
    │
    ▼ GC detects orphaned Pods
    │
    ▼ GC deletes Pods (graceful termination)
```

### Delete a Namespace

```
kubectl delete namespace staging
    │
    ▼
Namespace controller (NOT GC) handles this:
  - Deletes ALL objects in the namespace
  - Uses namespace finalizer: "kubernetes"
  - Namespace stays Terminating until all objects are gone
  - Then namespace itself is removed
```

Namespace deletion is a special case — it's the namespace controller, not the GC.

### Adopt Orphaned Objects

If you create a Deployment with a label selector matching existing orphaned pods/ReplicaSets, the controllers will **adopt** them (set ownerReferences):

```bash
# Orphan a ReplicaSet:
kubectl delete deployment my-app --cascade=orphan

# Create a new Deployment with the same selector:
kubectl apply -f deployment.yaml
# → Deployment controller finds matching RS, adopts it via ownerReference
```

## Debugging GC Issues

```bash
# Check if an object has ownerReferences:
kubectl get <resource> <name> -o jsonpath='{.metadata.ownerReferences}' | jq .

# Find orphaned objects (no ownerReferences, not top-level):
kubectl get pods -o json | jq -r '.items[] | select(.metadata.ownerReferences == null) | .metadata.name'

# Check for stuck objects with finalizers:
kubectl get all -A -o json | jq -r '.items[] | select(.metadata.deletionTimestamp != null and .metadata.finalizers != null) | "\(.kind)/\(.metadata.name) in \(.metadata.namespace): \(.metadata.finalizers)"'

# Check GC controller logs:
kubectl logs -n kube-system -l component=kube-controller-manager --tail=50 | grep -i "garbage\|gc\|orphan"

# Force remove a stuck finalizer (use carefully):
kubectl patch <resource> <name> -p '{"metadata":{"finalizers":null}}'

# See deletion propagation in action:
kubectl delete deployment my-app -v=6
# Watch the cascade happen:
kubectl get all -l app=my-app -w
```

### Common Stuck Scenarios

| Symptom | Cause | Fix |
|---------|-------|-----|
| Namespace stuck Terminating | Resources with finalizers not being cleared | Find and fix or remove stuck finalizers |
| Pods remain after Deployment delete | GC hasn't processed yet (race) | Wait a few seconds, or check GC health |
| Object stuck Terminating | Finalizer not removed by responsible controller | Check controller logs, or manually remove finalizer |
| Orphaned ReplicaSets accumulate | `revisionHistoryLimit` set too high | Lower the limit, or manually delete old RS |

## Quick Reference

```bash
# Cascading delete modes:
kubectl delete deployment x                   # background (default)
kubectl delete deployment x --cascade=background
kubectl delete deployment x --cascade=foreground  # wait for children first
kubectl delete deployment x --cascade=orphan      # keep children, strip ownerRef

# ownerReferences:
# - Set by controllers when creating child objects
# - Forms a dependency graph (DAG)
# - GC deletes objects when all owners are gone
# - Multi-owner: object survives until ALL owners deleted

# GC controller:
# - Runs in kube-controller-manager
# - Maintains in-memory ownership graph
# - Watches all objects for changes
# - Processes "dirty" (potentially orphaned) objects

# Finalizers:
# - Block deletion until removed
# - GC uses: "foregroundDeletion" and "orphan"
# - Custom controllers use their own finalizers for cleanup

# Key debugging:
kubectl get <resource> <name> -o jsonpath='{.metadata.ownerReferences}'
kubectl get <resource> <name> -o jsonpath='{.metadata.finalizers}'
kubectl get <resource> <name> -o jsonpath='{.metadata.deletionTimestamp}'

# GC does NOT handle:
# - Namespace cleanup (namespace controller)
# - Old RS pruning (Deployment controller + revisionHistoryLimit)
# - Event TTL (API server)
# - PV reclaim (PV controller)
```
