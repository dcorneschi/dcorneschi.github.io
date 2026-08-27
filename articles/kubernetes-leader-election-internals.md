# How Leader Election Works in Kubernetes Controllers

How Kubernetes ensures only one instance of a controller is active at a time — Lease-based leader election, split-brain prevention, failover timing, and how kube-controller-manager and kube-scheduler use it for HA.

## Why Leader Election Exists

Controllers that modify cluster state (Deployment controller, scheduler) must run as a single active instance to prevent conflicts. But for HA, you run multiple replicas. Leader election ensures:

- Only ONE replica is the **leader** (actively reconciling)
- Other replicas are **standby** (idle, ready to take over)
- If the leader crashes, a standby becomes the new leader

```
┌──────────────────────────────────────────────────────────────┐
│  3 replicas of kube-controller-manager                       │
│                                                              │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                  │
│  │ Leader   │   │ Standby  │   │ Standby  │                  │
│  │ (active) │   │ (idle)   │   │ (idle)   │                  │
│  │          │   │          │   │          │                  │
│  │ Runs all │   │ Watches  │   │ Watches  │                  │
│  │ control  │   │ Lease    │   │ Lease    │                  │ 
│  │ loops    │   │ object   │   │ object   │                  │
│  └──────────┘   └──────────┘   └──────────┘                  │
│       │                                                      │
│       └── Renews Lease every 2s                              │
│           If renewal stops → standby takes over              │
└──────────────────────────────────────────────────────────────┘
```

## The Lease Object

Leader election uses a **Lease** object (in `coordination.k8s.io/v1`) as a distributed lock:

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  holderIdentity: "controller-manager-node-1_abc123"   # Current leader
  leaseDurationSeconds: 15                              # Lock validity
  acquireTime: "2024-03-15T10:00:00Z"                   # When acquired
  renewTime: "2024-03-15T10:05:30Z"                     # Last renewal
  leaseTransitions: 3                                   # Times leadership changed
```

```bash
# Check who is the current leader:
kubectl get lease -n kube-system kube-controller-manager -o yaml
kubectl get lease -n kube-system kube-scheduler -o yaml

# See holder:
kubectl get lease -n kube-system kube-controller-manager \
  -o jsonpath='{.spec.holderIdentity}'
```

## Election Algorithm

```
┌────────────────────────────────────────────────────────────────┐
│  Leader Election Algorithm                                     │
│                                                                │
│  Each replica runs this loop:                                  │
│                                                                │
│  1. Try to acquire the Lease:                                  │
│     a. Read the Lease object                                   │
│     b. If no holder OR lease expired:                          │
│        → Update Lease with my identity (optimistic lock)       │
│        → If update succeeds: I am the leader                   │
│        → If 409 conflict: someone else won, retry later        │
│                                                                │
│  2. If I am the leader:                                        │
│     → Start running controller loops                           │
│     → Renew Lease every renewDeadline (default: 10s)           │
│     → If renewal fails: stop controller loops, re-enter step 1 │
│                                                                │
│  3. If I am NOT the leader:                                    │
│     → Sleep for retryPeriod (default: 2s)                      │
│     → Check if Lease is expired                                │
│     → If expired: attempt to acquire (go to step 1)            │
│     → If not expired: continue watching                        │
└────────────────────────────────────────────────────────────────┘
```

### Timing Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `leaseDuration` | 15s | How long the lock is valid without renewal |
| `renewDeadline` | 10s | How often the leader must renew (must be < leaseDuration) |
| `retryPeriod` | 2s | How often non-leaders check if lease is available |

```bash
# kube-controller-manager flags:
# --leader-elect=true (default)
# --leader-elect-lease-duration=15s
# --leader-elect-renew-deadline=10s
# --leader-elect-retry-period=2s
```

## Failover Timeline

```
Time ──────────────────────────────────────────────────────────────────▶

Leader (node-1)      Standby (node-2)     Lease Object
    │                    │                    │
    │ renew ──────────── ┼───────────────────▶│ renewTime updated
    │                    │ check: not expired │
    │                    │                    │
    │ renew ──────────── ┼───────────────────▶│ renewTime updated
    │                    │ check: not expired │
    │                    │                    │
    ╳ node-1 crashes     │                    │
    │ (no more renews)   │                    │
    │                    │                    │
    │                    │ check: not expired │ (still within 15s)
    │                    │ (wait 2s)          │
    │                    │                    │
    │                    │ check: not expired │ (still within 15s)
    │                    │ (wait 2s)          │
    │                    │                    │
    │         +15s       │                    │
    │                    │ check: EXPIRED!    │
    │                    │ attempt acquire ──▶│
    │                    │                    │ holderIdentity = node-2
    │                    │ ◀── success ───────│
    │                    │                    │
    │                    │ START controller   │
    │                    │ loops              │
    │                    │                    │
```

**Worst-case failover time**: `leaseDuration + retryPeriod` = 15s + 2s = **17 seconds**

During this window, no controller is actively reconciling. This is acceptable because:
- Existing objects continue to function (pods keep running)
- Events queue up and are processed when new leader starts
- Level-triggered design means nothing is lost (just delayed)

## What Uses Leader Election

### kube-controller-manager

```bash
# Contains ~40 controllers, all run by the single leader:
# Deployment, ReplicaSet, StatefulSet, DaemonSet, Job, CronJob,
# Endpoint, EndpointSlice, Namespace, ServiceAccount, Node,
# PV, GarbageCollector, etc.

kubectl get lease kube-controller-manager -n kube-system
```

### kube-scheduler

```bash
# Only one scheduler should assign pods to nodes at a time
kubectl get lease kube-scheduler -n kube-system
```

### cloud-controller-manager

```bash
# Cloud operations (LB provisioning, node management)
kubectl get lease cloud-controller-manager -n kube-system
```

### Custom Controllers / Operators

```bash
# Any controller running multiple replicas:
kubectl get leases -n <controller-namespace>

# Common examples:
# cert-manager, ArgoCD, ingress-nginx (status updates)
```

## How It Prevents Split-Brain

The Lease object uses Kubernetes **optimistic concurrency** (resourceVersion) to prevent two leaders:

```
┌────────────────────────────────────────────────────────────────┐
│  Split-Brain Prevention                                        │
│                                                                │
│  Scenario: Leader's renewal is slow (network partition)        │
│                                                                │
│  T+0s:  Leader renews (rv=100) → success                       │
│  T+10s: Leader tries to renew (rv=100) → slow network...       │
│  T+15s: Lease expired (no renewal received)                    │
│  T+16s: Standby acquires (rv=100 → rv=101) → success           │
│  T+17s: Old leader's slow renewal arrives (rv=100)             │
│         → 409 Conflict (rv mismatch: expected 100, got 101)    │
│         → Old leader KNOWS it lost leadership                  │
│         → Old leader stops all controller loops                │
│                                                                │
│  Result: exactly one leader at all times                       │
│  (brief gap during transition, but never two active leaders)   │
└────────────────────────────────────────────────────────────────┘
```

### The leader MUST stop on renewal failure

```
Leader renewal fails (409 or timeout)
    │
    ▼
Leader immediately:
  1. Stops all reconciliation loops
  2. Returns to "not leader" state
  3. Starts trying to re-acquire

This is critical — if the leader kept running after losing the Lease,
you'd have two controllers making conflicting decisions.
```

## Lease vs ConfigMap vs Endpoints (Historical)

Leader election used to be done via ConfigMap or Endpoints annotations. Lease is the modern approach:

| Method | Resource | Performance | Current Status |
|--------|----------|-------------|----------------|
| Endpoints | `endpoints/{name}` in kube-system | Poor (large objects, triggers endpoint watches) | Deprecated |
| ConfigMap | `configmaps/{name}` in kube-system | Moderate | Deprecated |
| Lease | `leases/{name}` in kube-system | Best (small, dedicated API) | Default (1.17+) |

```bash
# Old-style (might still see in legacy clusters):
kubectl get endpoints -n kube-system kube-controller-manager -o yaml
kubectl get configmap -n kube-system kube-controller-manager -o yaml

# Modern:
kubectl get lease -n kube-system kube-controller-manager -o yaml
```

## Leader Election in Custom Controllers

### Using client-go LeaderElector

```go
// Simplified leader election setup:
leaderElector, _ := leaderelection.NewLeaderElector(leaderelection.LeaderElectionConfig{
    Lock: &resourcelock.LeaseLock{
        LeaseMeta: metav1.ObjectMeta{
            Name:      "my-controller",
            Namespace: "my-system",
        },
        Client: client.CoordinationV1(),
        LockConfig: resourcelock.ResourceLockConfig{
            Identity: hostname + "_" + uuid,
        },
    },
    LeaseDuration: 15 * time.Second,
    RenewDeadline: 10 * time.Second,
    RetryPeriod:   2 * time.Second,
    Callbacks: leaderelection.LeaderCallbacks{
        OnStartedLeading: func(ctx context.Context) {
            // Start controller loops
            runControllers(ctx)
        },
        OnStoppedLeading: func() {
            // Stop everything, log, exit
            log.Fatal("lost leadership")
        },
        OnNewLeader: func(identity string) {
            log.Printf("new leader: %s", identity)
        },
    },
})

leaderElector.Run(ctx)
```

### Helm/YAML for Controller with Leader Election

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-controller
spec:
  replicas: 2                    # Multiple replicas for HA
  template:
    spec:
      containers:
      - name: controller
        args:
        - --leader-elect=true
        - --leader-elect-lease-duration=15s
        - --leader-elect-renew-deadline=10s
        - --leader-elect-retry-period=2s
---
# RBAC: controller needs permission to manage its Lease
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-controller-leader-election
  namespace: my-system
rules:
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["get", "create", "update"]
```

## Leader Election for the Data Plane

Some controllers separate leader election by function:

```
┌────────────────────────────────────────────────────────────────┐
│  Ingress Controller Example (NGINX)                            │
│                                                                │
│  Data Plane (all replicas):                                    │
│    - Serve traffic (no leader election needed)                 │
│    - All replicas handle requests independently                │
│                                                                │
│  Control Plane (leader only):                                  │
│    - Update Ingress status (set external IP)                   │
│    - Only leader writes to avoid conflicts                     │
│                                                                │
│  Result: high-throughput data path + single-writer control     │
└────────────────────────────────────────────────────────────────┘
```

## Monitoring Leader Election

```bash
# Check current leaders:
kubectl get leases -n kube-system
# NAME                      HOLDER                              AGE
# kube-controller-manager   ctrl-mgr-node-1_abc123              5d
# kube-scheduler            scheduler-node-2_def456             5d
# cloud-controller-manager  cloud-ctrl-node-1_ghi789            5d

# Check lease health (recent renewTime):
kubectl get lease -n kube-system kube-controller-manager \
  -o jsonpath='{.spec.renewTime}'
# If renewTime is stale (>15s ago), leader may be unhealthy

# Watch for leadership changes:
kubectl get lease -n kube-system -w

# Check leaseTransitions (how many times leadership changed):
kubectl get lease -n kube-system kube-controller-manager \
  -o jsonpath='{.spec.leaseTransitions}'
# High number = instability (leader keeps crashing)

# Controller manager logs on leadership:
kubectl logs -n kube-system kube-controller-manager-<node> | grep -i "leader\|elected\|lost"
```

### Prometheus Metrics

```promql
# Leader election metric (1 = leader, 0 = not leader):
leader_election_master_status{name="kube-controller-manager"}

# Time since last successful leader renewal:
# (derive from renewTime in Lease object)
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Frequent leaseTransitions | Leader pod keeps crashing or restarting | Check pod logs, resource limits, node health |
| renewTime stale but pod running | Leader can't reach API server | Check network, API server health |
| No leader (empty holderIdentity) | All replicas failing to acquire | Check RBAC (needs create/update on leases), API server availability |
| Two instances think they're leader | Should not happen (CAS prevents it) | If it does: likely clock skew or etcd inconsistency |
| Controllers not reconciling | Leader lost and failover hasn't happened | Wait up to 17s, check if standby is healthy |

## Quick Reference

```bash
# Leader election mechanism:
# - Lease object in coordination.k8s.io/v1
# - Optimistic concurrency (resourceVersion) prevents split-brain
# - Only holder can renew; others acquire when expired

# Timing:
# leaseDuration: 15s (lock validity)
# renewDeadline: 10s (renewal frequency)
# retryPeriod:   2s (non-leader check frequency)
# Worst-case failover: ~17s

# Check leaders:
kubectl get leases -n kube-system
kubectl get lease <name> -n kube-system -o jsonpath='{.spec.holderIdentity}'

# Key components using leader election:
# kube-controller-manager, kube-scheduler, cloud-controller-manager
# Custom operators (cert-manager, ArgoCD, etc.)

# RBAC needed:
# resources: ["leases"], verbs: ["get", "create", "update"]
# in coordination.k8s.io apiGroup

# The leader MUST stop reconciling on renewal failure
# (this is what prevents split-brain)

# Data plane (traffic serving) does NOT need leader election
# Only control plane (state mutations) needs it
```
