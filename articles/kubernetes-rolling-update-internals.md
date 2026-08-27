# What Happens During a Kubernetes Rolling Update

The internal controller mechanics when you update a Deployment — how the Deployment controller creates ReplicaSets, scales them up and down, manages revisions, and coordinates with the scheduler and kubelet.

Note: For strategy comparison (Recreate, Blue-Green, Canary, A/B), see the deployment strategies guide. This article focuses on the Deployment controller's internal reconciliation loop.

## High-Level Flow

```
kubectl set image deployment/app app=v2
        │
        ▼
┌───────────────┐     ┌────────────────────┐     ┌──────────────────┐
│   API Server  │────▶│ Deployment         │────▶│ ReplicaSet       │
│  (updates     │     │ Controller         │     │ Controller       │
│   Deployment) │     │ (creates new RS)   │     │ (creates Pods)   │
└───────────────┘     └────────────────────┘     └──────────────────┘
                                                          │
                                                          ▼
                                                 ┌──────────────────┐
                                                 │   Scheduler +    │
                                                 │   Kubelet        │
                                                 │  (run new Pods)  │
                                                 └──────────────────┘
```

## The Deployment Controller's Reconciliation

When you change a Deployment's pod template (image, env, resources, etc.), the Deployment controller detects the spec change and orchestrates the rollout:

```
┌───────────────────────────────────────────────────────────────────┐
│  Deployment Controller Logic                                      │
│                                                                   │
│  1. Detect: pod template hash changed                             │
│  2. Create new ReplicaSet (or find existing one with same hash)   │
│  3. Scale new RS up by maxSurge                                   │
│  4. Wait for new pods to be Ready                                 │
│  5. Scale old RS down by maxUnavailable                           │
│  6. Repeat steps 3-5 until:                                       │
│     - New RS has desired replicas                                 │
│     - Old RS has 0 replicas                                       │
│  7. Update Deployment status and revision                         │
└───────────────────────────────────────────────────────────────────┘
```

## ReplicaSet Creation

The Deployment controller doesn't manage pods directly. It manages ReplicaSets, which manage pods:

```
Deployment (desired: 3 replicas, image: v2)
    │
    ├── ReplicaSet-v1 (image: v1) — scaling DOWN
    │     ├── Pod-a (v1) — Terminating
    │     ├── Pod-b (v1) — Running
    │     └── Pod-c (v1) — Running
    │
    └── ReplicaSet-v2 (image: v2) — scaling UP
          ├── Pod-d (v2) — Running (Ready)
          └── Pod-e (v2) — ContainerCreating
```

Each unique pod template gets its own ReplicaSet, identified by a **pod-template-hash** label:

```bash
# See ReplicaSets and their pod-template-hash:
kubectl get rs -l app=myapp -o custom-columns=\
  NAME:.metadata.name,\
  DESIRED:.spec.replicas,\
  READY:.status.readyReplicas,\
  HASH:.metadata.labels.pod-template-hash
```

## maxSurge and maxUnavailable — The Math

These two parameters control rollout speed vs resource usage:

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `maxSurge` | 25% | Max pods ABOVE desired count during rollout |
| `maxUnavailable` | 25% | Max pods BELOW desired count during rollout |

For a Deployment with `replicas: 4`:

```
maxSurge: 25%         → ceil(4 * 0.25) = 1  → max 5 pods total
maxUnavailable: 25%   → floor(4 * 0.25) = 1 → min 3 pods available

Available range during rollout: [3, 5]
```

### Rollout Progression (replicas=4, maxSurge=1, maxUnavailable=1)

```
Step   Old RS   New RS   Total   Available   Action
─────  ──────   ──────   ─────   ─────────   ──────
  0      4        0        4        4        Initial state
  1      4        1        5        4        Scale new RS +1 (maxSurge)
  2      3        1        4        3*       Scale old RS -1 (new pod Ready)
  3      3        2        5        4        Scale new RS +1
  4      2        2        4        3*       Scale old RS -1
  5      2        3        5        4        Scale new RS +1
  6      1        3        4        3*       Scale old RS -1
  7      1        4        5        4        Scale new RS +1
  8      0        4        4        4        Scale old RS -1 (done)

* Available = total - pods not yet Ready - pods Terminating
```

### Aggressive Rollout (maxSurge=50%, maxUnavailable=50%)

```
replicas: 4
maxSurge: 2       → max 6 pods total
maxUnavailable: 2 → min 2 pods available

Step   Old RS   New RS   Total   Action
─────  ──────   ──────   ─────   ──────
  0      4        0        4     Initial
  1      4        2        6     Scale new RS +2 (hit maxSurge)
  2      2        2        4     Scale old RS -2 (new pods Ready)
  3      2        4        6     Scale new RS +2
  4      0        4        4     Scale old RS -2 (done)
```

Faster rollout, but uses more resources during the transition.

### Zero-Downtime Rollout (maxSurge=1, maxUnavailable=0)

```
replicas: 4
maxSurge: 1       → max 5 pods total
maxUnavailable: 0 → min 4 pods available (never below desired)

Step   Old RS   New RS   Total   Available   Action
─────  ──────   ──────   ─────   ─────────   ──────
  0      4        0        4        4        Initial
  1      4        1        5        4        Scale new RS +1
  2      3        1        4        4        Scale old -1 (only after new is Ready)
  3      3        2        5        4        Scale new RS +1
  ...continues one at a time...
```

Slowest but safest — always at full capacity.

## Pod Template Hash

The Deployment controller generates a hash of the pod template spec and uses it to:
1. Label pods so they're uniquely associated with a ReplicaSet
2. Name ReplicaSets (e.g., `myapp-6d4f7b8c9`)
3. Detect whether a matching ReplicaSet already exists

```bash
# The hash is a label on the RS and its pods:
kubectl get rs -l app=myapp --show-labels
# NAME              DESIRED   CURRENT   READY   LABELS
# myapp-6d4f7b8c9   4         4         4       app=myapp,pod-template-hash=6d4f7b8c9
# myapp-7a5e8f2d1   0         0         0       app=myapp,pod-template-hash=7a5e8f2d1
```

If you update a Deployment back to a previous image, the controller finds the existing old ReplicaSet (by hash match) and scales it back up instead of creating a new one.

## Revision History

Each ReplicaSet represents a revision. The Deployment stores revision metadata in annotations:

```bash
# See all revisions:
kubectl rollout history deployment/myapp

# See a specific revision's pod template:
kubectl rollout history deployment/myapp --revision=3

# The revision number is stored on the ReplicaSet:
kubectl get rs -l app=myapp -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.deployment\.kubernetes\.io/revision}{"\n"}{end}'
```

The `revisionHistoryLimit` field (default: 10) controls how many old ReplicaSets are kept:

```yaml
spec:
  revisionHistoryLimit: 10  # Keep 10 old ReplicaSets (for rollback)
```

Old ReplicaSets with 0 replicas are kept for rollback history. Once you exceed the limit, the oldest are garbage collected.

## Readiness Gates and Rollout Progression

The Deployment controller only scales down the old RS when new pods are **Ready**:

```
New Pod lifecycle during rollout:
  1. Created by ReplicaSet controller
  2. Scheduled by scheduler
  3. Container started by kubelet
  4. Startup probe passes (if configured)
  5. Readiness probe passes
  6. Pod condition Ready=True
  7. ──▶ Deployment controller sees this, proceeds with rollout
```

If a new pod never becomes Ready (bad image, crash, failing probe), the rollout stalls:

```bash
# Deployment shows progressing=False after deadline:
kubectl get deployment myapp -o jsonpath='{.status.conditions[?(@.type=="Progressing")]}'

# Default progressDeadlineSeconds is 600 (10 minutes)
```

## Rollout Stall and Deadline

```yaml
spec:
  progressDeadlineSeconds: 600  # 10 min default
```

If no progress is made for `progressDeadlineSeconds`:
- The Deployment's `Progressing` condition becomes `False`
- The reason is `ProgressDeadlineExceeded`
- The rollout does NOT automatically roll back — it stays stalled
- Pods from the old RS continue serving traffic

```bash
# Check if rollout is stalled:
kubectl rollout status deployment/myapp
# error: deployment "myapp" exceeded its progress deadline

# Manual rollback:
kubectl rollout undo deployment/myapp
```

## Rollback Mechanics

```bash
kubectl rollout undo deployment/myapp
```

What happens internally:
1. Deployment controller finds the previous ReplicaSet (revision N-1)
2. Updates the Deployment's pod template to match that RS's template
3. This triggers a new rollout (same mechanics as above)
4. The old RS becomes the "new" RS (already has pods from the previous revision)
5. If the RS still has pods, scaling is faster (no image pull needed)

```bash
# Rollback to specific revision:
kubectl rollout undo deployment/myapp --to-revision=5

# This copies the pod template from revision 5's RS into the Deployment spec
# Then a normal rolling update proceeds
```

## Pause and Resume

```bash
# Pause — stop the rollout mid-way
kubectl rollout pause deployment/myapp

# Make multiple changes while paused:
kubectl set image deployment/myapp app=v3
kubectl set resources deployment/myapp -c app --limits=memory=512Mi

# Resume — all changes applied as a single rollout
kubectl rollout resume deployment/myapp
```

When paused:
- The Deployment controller stops scaling operations
- Already-running pods remain (no rollback)
- Changes accumulate until resume triggers a single combined rollout

## The Controller Watch Loop

```
┌────────────────────────────────────────────────────────────┐
│  Deployment Controller (in kube-controller-manager)        │
│                                                            │
│  Watches:                                                  │
│    - Deployments (spec changes trigger reconciliation)     │
│    - ReplicaSets (owned by this Deployment)                │
│    - Pods (to track Ready count)                           │
│                                                            │
│  On each reconciliation:                                   │
│    1. List all RS owned by this Deployment                 │
│    2. Identify the "new" RS (matches current pod template) │
│    3. Calculate desired replica counts for new/old RS      │
│    4. Scale RS accordingly (respecting surge/unavailable)  │
│    5. Update Deployment status (replicas, ready, etc.)     │
│                                                            │
│  Triggers:                                                 │
│    - Deployment spec change                                │
│    - Pod becomes Ready                                     │
│    - Pod is deleted or fails                               │
│    - ReplicaSet status change                              │
└────────────────────────────────────────────────────────────┘
```

## Complete Timeline

```
Time ──────────────────────────────────────────────────────────────────▶

User            API Server      Deployment Ctrl    RS Controller    Scheduler+Kubelet
 │                │                │                │                │
 │ set image v2 ─▶│                │                │                │
 │                │ update Deploy  │                │                │
 │                │ ─── watch ────▶│                │                │
 │                │                │ create RS-v2   │                │
 │                │                │ (replicas=1)   │                │
 │                │                │ ─── watch ────▶│                │
 │                │                │                │ create Pod     │
 │                │                │                │ ─── watch ────▶│
 │                │                │                │                │ schedule + start
 │                │                │                │                │
 │                │                │                │                │Pod Ready ──┐
 │                │                │ ◀───────────── ┼────────────────┼────────────┘
 │                │                │                │                │
 │                │                │ scale RS-v2=2  │                │
 │                │                │ scale RS-v1=3  │                │
 │                │                │                │                │
 │                │                │  ...continues until...          │
 │                │                │                │                │
 │                │                │ RS-v2=4, RS-v1=0                │
 │                │                │ update Deploy status            │
 │                │                │ (conditions: Available=True)    │
```

## Observing a Rollout

```bash
# Watch the rollout in real-time:
kubectl rollout status deployment/myapp

# Watch RS scaling:
kubectl get rs -l app=myapp -w

# Watch pods being created/terminated:
kubectl get pods -l app=myapp -w

# See events during rollout:
kubectl describe deployment myapp | tail -20

# Check deployment conditions:
kubectl get deployment myapp -o jsonpath='{.status.conditions[*].type}'
# Available Progressing
```

## Deployment Status Fields

```yaml
status:
  replicas: 5              # Total pods (old + new RS)
  updatedReplicas: 3       # Pods running the new template
  readyReplicas: 4         # Pods passing readiness
  availableReplicas: 4     # Pods Ready for minReadySeconds
  unavailableReplicas: 1   # Pods not yet available
  conditions:
  - type: Available        # At least minAvailable pods ready
    status: "True"
  - type: Progressing      # Rollout making progress
    status: "True"
    reason: NewReplicaSetAvailable
```

## minReadySeconds

```yaml
spec:
  minReadySeconds: 30  # Pod must be Ready for 30s before considered Available
```

This adds a delay between a pod becoming Ready and the controller counting it as "available". Useful for catching pods that pass probes but crash shortly after:

```
Pod Ready ─── 30s wait ──── Pod Available ──── Controller scales down old RS
```

Without `minReadySeconds` (default: 0), the controller proceeds the moment the readiness probe passes.

## Quick Reference

```bash
# Trigger rollout
kubectl set image deployment/app container=image:v2
kubectl apply -f deployment.yaml  # if template changed

# Watch rollout
kubectl rollout status deployment/app

# Pause/resume
kubectl rollout pause deployment/app
kubectl rollout resume deployment/app

# Rollback
kubectl rollout undo deployment/app
kubectl rollout undo deployment/app --to-revision=3

# History
kubectl rollout history deployment/app
kubectl rollout history deployment/app --revision=2

# Check RS scaling during rollout
kubectl get rs -l app=myapp -w

# Check why rollout is stuck
kubectl describe deployment app | grep -A5 Conditions
kubectl get pods -l app=myapp --sort-by=.metadata.creationTimestamp
```
