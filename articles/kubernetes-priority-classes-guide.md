<img src="/articles/images/kubernetes-logo.svg" alt="Kubernetes" width="150">

# Kubernetes PriorityClasses — A Practical Guide

## What Are PriorityClasses?

PriorityClasses define the relative importance of Pods in a Kubernetes cluster. When the scheduler cannot find room for a pending Pod, it uses priority to decide which existing Pods to preempt (evict). Higher priority Pods can displace lower priority ones.

Priority values range from **-2,147,483,648** to **1,000,000,000** (values above 1 billion are reserved for system components).

Every cluster ships with two built-in system PriorityClasses:

| Name | Value | Purpose |
|------|-------|---------|
| `system-node-critical` | 2000001000 | Core node components (kubelet, kube-proxy) |
| `system-cluster-critical` | 2000000000 | Cluster-level components (CoreDNS, etcd, API server) |

If you don't assign a PriorityClass to your Pods, they get a priority value of **0** — the lowest possible.


## Why This Matters

With only the two default system classes, your cluster looks like this:

```
Priority Scale
═══════════════════════════════════════════════════════

  2,000,001,000 ─── system-node-critical (kubelet, kube-proxy)
  2,000,000,000 ─── system-cluster-critical (CoreDNS, etcd)
          ...
          ...       (massive gap — nothing here)
          ...
            0 ───── ALL your workloads (default)

```

**Problems with this setup:**

1. **All your workloads are equal** — the scheduler has no preference between your critical production API and a throwaway batch job.
2. **No preemption hierarchy** — if the cluster is full, a batch job won't be preempted to make room for your revenue-critical service.
3. **Node pressure eviction treats them equally** — kubelet uses priority as a tiebreaker within QoS classes.
4. **Starvation risk** — low-importance workloads can starve high-importance ones by consuming all available resources first.


## Recommended Priority Tiers

A well-structured cluster typically has 3–5 custom tiers between the system classes and the default:

```
Priority Scale (Recommended)
═══════════════════════════════════════════════════════

  2,000,001,000 ─── system-node-critical      (DO NOT USE for your workloads)
  2,000,000,000 ─── system-cluster-critical   (DO NOT USE for your workloads)
    1,000,000 ───── platform-critical         (ingress, monitoring, service mesh)
      500,000 ───── production-high           (revenue-critical APIs, databases)
      250,000 ───── production-medium         (internal services, workers)
      100,000 ───── production-low            (non-critical services, dashboards)
            0 ───── default                   (dev workloads, experiments, batch)
     -100,000 ───── best-effort               (preemptible jobs, cost-optimized)

```


## Creating PriorityClasses

### Platform-Critical (ingress controllers, monitoring, service mesh)

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: platform-critical
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Platform infrastructure: ingress, monitoring, service mesh, cert-manager"
```

### Production-High (revenue-critical services)

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-high
value: 500000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Revenue-critical production services: APIs, databases, payment processing"
```

### Production-Medium (internal services)

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-medium
value: 250000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Internal production services: background workers, internal APIs"
```

### Production-Low (non-critical services)

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-low
value: 100000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Non-critical production: dashboards, admin tools, reports"
```

### Best-Effort (preemptible, cost-optimized)

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: best-effort
value: -100000
globalDefault: false
preemptionPolicy: Never    # These pods will NOT preempt others
description: "Best-effort workloads: batch jobs, dev environments, experiments"
```


## Using PriorityClasses in Your Pods

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
spec:
  replicas: 3
  template:
    spec:
      priorityClassName: production-high    # <-- assign here
      containers:
      - name: api
        image: myapp/payment-api:latest
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            memory: "1Gi"
```


## How Priority Interacts with Eviction

Priority affects multiple eviction paths:

| Eviction Type | How Priority Affects It |
|---------------|------------------------|
| **Scheduler preemption** | Lower-priority pods are evicted to make room for higher-priority pending pods |
| **Node pressure eviction** | Within the same QoS class, lower-priority pods are evicted first |
| **Taint-based eviction** | No effect — all non-tolerating pods are evicted regardless of priority |
| **Eviction API (drain)** | No effect — PDBs control this, not priority |

### Preemption in Action

```
Scenario: Cluster is full. A production-high Pod is pending.
═══════════════════════════════════════════════════════════════

Scheduler evaluates nodes:

  Node A:                          Node B:
  ┌──────────────────────┐        ┌──────────────────────┐
  │ pod-1: prod-high     │        │ pod-3: best-effort   │ ◄── victim
  │ pod-2: prod-medium   │        │ pod-4: prod-low      │ ◄── victim
  └──────────────────────┘        └──────────────────────┘

  Scheduler picks Node B (can evict lower-priority pods).
  pod-3 and pod-4 are terminated gracefully.
  Pending high-priority pod is scheduled on Node B.
```

### Preemption Decision Flow

```
                    ┌─────────────────────────┐
                    │  High-priority Pod (P)  │
                    │ enters scheduling queue │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │  Can P be scheduled on  │
                    │  any node as-is?        │
                    └────────────┬────────────┘
                          │            │
                         YES          NO
                          │            │
                          ▼            ▼
                    ┌──────────┐  ┌──────────────────────────┐
                    │ Schedule │  │Find nodes where removing │
                    │ normally │  │lower-priority pods would │
                    └──────────┘  │make room for P           │
                                  └─────────────┬────────────┘
                                          │           │
                                        FOUND      NOT FOUND
                                          │           │
                                          ▼           ▼
                              ┌────────────────┐  ┌──────────────┐
                              │ Check:         │  │ P stays in   │
                              │ • PDB impact   │  │pending queue │
                              │ • Pod affinity │  └──────────────┘
                              │ • Preemption   │
                              │   policy       │
                              └───────┬────────┘
                                      │
                                      ▼
                              ┌────────────────────────────────┐
                              │ Select victim pods             │
                              │ (lowest priority first)        │
                              └───────┬────────────────────────┘
                                      │
                                      ▼
                              ┌────────────────────────────────┐
                              │ Set P.nominatedNodeName = Node │
                              │ Send graceful termination to   │
                              │ victims (default 30s)          │
                              └───────┬────────────────────────┘
                                      │
                                      ▼
                              ┌────────────────────────────────┐
                              │ Victims terminated?            │
                              │                                │
                              │ • If yes → schedule P on node  │
                              │ • If higher-priority Pod Q     │
                              │   arrives → Q may take the     │
                              │   node, P.nominatedNodeName    │
                              │   is cleared, P retries        │
                              └────────────────────────────────┘
```

### Key Preemption Constraints

```
┌─────────────────────────────────────────────────────────────────────┐
│ PREEMPTION RULES                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✓ Only evicts pods on the SAME candidate node (no cross-node)      │
│  ✓ Picks the node with lowest-priority victims first                │
│  ✓ Respects PDB where possible (best-effort, not guaranteed)        │
│  ✓ Won't preempt if pod has affinity to a lower-priority victim     │
│  ✓ nominatedNodeName ≠ guaranteed final placement                   │
│  ✓ preemptionPolicy: Never pods queue first but never evict others  │
│  ✓ Victims get graceful termination (30s default or pod's setting)  │
│  ✓ Minimum victims evicted (not necessarily all lower-priority)     │ 
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```


## The `preemptionPolicy` Field

Two options:

| Policy | Behavior |
|--------|----------|
| `PreemptLowerPriority` (default) | This pod CAN trigger eviction of lower-priority pods |
| `Never` | This pod has high scheduling priority (queued first) but will NOT evict others |

**When to use `Never`:**
- Batch jobs that should be scheduled before best-effort work but shouldn't disrupt running services
- Development workloads that you want prioritized over experiments but that shouldn't affect production

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-high
value: 200000
preemptionPolicy: Never    # High priority in queue, but won't evict others
description: "Important batch jobs that should not preempt production workloads"
```


## The `globalDefault` Field

Setting `globalDefault: true` on a PriorityClass makes it the default for all Pods that don't specify a `priorityClassName`. Only one PriorityClass can be the global default.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-low
value: 100000
globalDefault: true    # All pods without explicit priorityClassName get this
preemptionPolicy: PreemptLowerPriority
description: "Default priority for all workloads"
```

**Recommendation:** Consider setting `production-low` or a similar mid-tier as the global default so that pods without explicit priority aren't sitting at 0.


## Common Pitfalls

### 1. Values too close to system classes

```yaml
# BAD — too close to system range
value: 1999999999

# GOOD — stay well below 1 billion
value: 1000000
```

System priority classes use values >= 2 billion. Keep your custom classes well below 1 billion to avoid accidentally competing with system components.

### 2. Forgetting to set priority on critical workloads

If your monitoring stack (Prometheus, Grafana) doesn't have a high PriorityClass, it could be evicted right when you need it most — during a resource crunch.

### 3. Every workload at the same priority

If everything is `production-high`, nothing is. The scheduler can't make meaningful preemption decisions when all pods have equal priority.

### 4. Not using `preemptionPolicy: Never` for batch

Batch jobs with `PreemptLowerPriority` will evict your lower-priority running services. Unless the batch job is truly more important than the running service, use `Never`.

### 5. Ignoring PDB + Priority interaction

During preemption, the scheduler tries to avoid PDB violations but will violate them for sufficiently high-priority pods. Don't rely solely on PDBs to protect low-priority workloads from preemption.

Pair your high-priority deployments with a PodDisruptionBudget to guard against excessive preemption:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      priorityClassName: production-high
      containers:
      - name: api
        image: myapp/payment-api:latest
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-api-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: payment-api
```

### 6. Not accounting for graceful termination during preemption

When a pod is preempted, it gets a default 30-second graceful termination period. If your pod has `terminationGracePeriodSeconds` or `preStop` lifecycle hooks, those override the default. Long-running cleanup tasks can delay preemption and leave high-priority pods waiting longer than expected.

```yaml
# If your pod needs time to drain connections:
spec:
  terminationGracePeriodSeconds: 60  # Preempted pods get up to 60s to shut down
```

### 7. Confusing QoS-based eviction with priority-based preemption

These are two separate mechanisms:

- **Scheduler preemption** — triggered when a high-priority pod is in the scheduling queue and no resources are available. The scheduler ignores QoS class entirely — it only looks at pod priority values.
- **Kubelet eviction** — triggered by node resource pressure (memory/disk/PID). The kubelet considers QoS class *first* (`BestEffort` → `Burstable` → `Guaranteed`), then pod priority *within* the same QoS tier.

A `Guaranteed` pod with low priority won't be preempted by kubelet eviction before a `BestEffort` pod, but it *will* be preempted by the scheduler if a higher-priority pod needs the resources.

### 8. Ignoring `nominatedNodeName` behavior

When a pod triggers preemption, its `status.nominatedNodeName` is set to the target node. However, the pod is **not guaranteed** to land on that node:

- A higher-priority pod may arrive and take the nominated node instead.
- Another node might free up while victims are terminating, and the scheduler will use that instead.
- The scheduler clears `nominatedNodeName` if the pod loses its spot, making it eligible to preempt elsewhere.

Don't assume `nominatedNodeName == nodeName` — they can differ.

### 9. Inter-pod affinity towards lower-priority pods

If a pending high-priority pod has inter-pod affinity to a lower-priority pod on a node, the scheduler **won't preempt from that node** — evicting the lower-priority pod would break the affinity rule and make scheduling impossible anyway.

Best practice: only define inter-pod affinity towards pods of equal or higher priority.

### 10. No cross-node preemption

The scheduler won't evict pods on Node B to satisfy topology or anti-affinity constraints for scheduling on Node A. Preemption only considers pods on the same candidate node.

Example: if Pod P has zone-wide anti-affinity with Pod Q on another node in the same zone, the scheduler won't preempt Pod Q to make room. Pod P will be deemed unschedulable on that node.

### 11. Not using ResourceQuota to limit PriorityClass usage

In multi-tenant clusters, any user can create pods with high PriorityClasses unless restricted. Use `ResourceQuota` with `scopeSelector` to limit which namespaces can consume specific priority classes:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: limit-critical-priority
  namespace: dev-team
spec:
  hard:
    pods: "0"
  scopeSelector:
    matchExpressions:
    - scopeName: PriorityClass
      operator: In
      values:
      - platform-critical
      - production-high
```

This prevents the `dev-team` namespace from creating pods using `platform-critical` or `production-high` classes.

### 12. Deleting a PriorityClass that's still in use

If you delete a PriorityClass:

- Existing pods **keep their priority value** — they're unaffected.
- New pods referencing the deleted class name will be **rejected** by the admission controller.

Always verify no workloads reference a PriorityClass before deleting it:

```bash
kubectl get pods -A -o jsonpath='{range .items[?(@.spec.priorityClassName=="<class-name>")]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'
```


## Best Practices

### Keep the priority model small and deliberate

3–5 application-facing priority levels is the sweet spot. More levels create confusion without improving scheduling behavior. Use wide numeric gaps between values (e.g. 100000, 500000, 1000000) to leave room for future classes without renumbering.

### Require resource requests for elevated priority

The scheduler makes placement decisions using resource *requests*, not actual usage. A high-priority pod without requests can be scheduled in ways that create node pressure later. If a team can't define reasonable requests, they shouldn't receive preemption rights yet.

### Treat frequent preemption as a capacity problem

If high-priority pods are constantly preempting lower-priority pods, your cluster is telling you something: capacity, quotas, or workload placement rules are wrong. Don't solve this by raising more priorities — fix the underlying capacity issue.

Watch for these signals:

- Lower-priority Deployments that can't return to their desired replica count
- Pods with frequent `Preempted` events after high-priority deploys
- Autoscaling events that show the cluster can't add nodes fast enough
- Namespaces repeatedly hitting high-priority ResourceQuotas

### Enforce priority assignment with admission policies

Without enforcement, anyone can assign any PriorityClass to their workload. Use admission policies to control which namespaces or teams can use which priority classes:

- **ValidatingAdmissionPolicy** (Kubernetes 1.26+) — built-in, no external tooling needed
- **Kyverno** or **Gatekeeper** — for more complex rules

Example Kyverno policy restricting `platform-critical` usage:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-platform-critical
spec:
  validationFailureAction: Enforce
  rules:
  - name: deny-platform-critical-outside-infra
    match:
      any:
      - resources:
          kinds:
          - Pod
    exclude:
      any:
      - resources:
          namespaces:
          - kube-system
          - monitoring
          - ingress-nginx
    validate:
      message: "Only infrastructure namespaces can use platform-critical PriorityClass"
      pattern:
        spec:
          =(priorityClassName): "!platform-critical"
```

### Restart workloads after changing a PriorityClass value

Priority is resolved at pod admission time. Changing a PriorityClass value does not update running pods — they keep their old priority until recreated:

```bash
kubectl rollout restart deployment/payment-api -n production
```

Verify the effective priority:

```bash
kubectl get pod <pod-name> -o jsonpath='{.spec.priorityClassName} {.spec.priority}'
```

### Test preemption before enabling it broadly

Don't introduce high-priority preemption directly into a busy production cluster. Test in a dedicated namespace:

1. Create your PriorityClasses
2. Deploy lower-priority pods that consume most available resources
3. Deploy a higher-priority pod that can't fit
4. Observe which pods get preempted
5. Check events and verify recovery behavior

```bash
# Watch preemption in action
kubectl get events -n priority-test --sort-by='.lastTimestamp' --field-selector reason=Preempted
kubectl get pods -n priority-test -w
```

### Rollout checklist

Before merging PriorityClasses into a shared cluster:

- [ ] No more than 3–5 application-facing priority levels
- [ ] `system-*` classes reserved for Kubernetes system components only
- [ ] One clear `globalDefault` set (typically a normal workload class)
- [ ] Resource requests required for every workload with elevated priority
- [ ] ResourceQuotas scoped by PriorityClass per namespace
- [ ] `preemptionPolicy: Never` used where queue ordering is sufficient
- [ ] Preemption tested in a non-production namespace
- [ ] Admission policy enforcing which namespaces can use high-priority classes
- [ ] Monitoring in place for preemptions, pending pods, and quota failures
- [ ] Documentation of who can approve new high-priority workloads


## Quick Setup — Apply All Classes at Once

```yaml
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: platform-critical
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Platform infrastructure: ingress, monitoring, service mesh"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-high
value: 500000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Revenue-critical production services"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-medium
value: 250000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Internal production services"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-low
value: 100000
globalDefault: true
preemptionPolicy: PreemptLowerPriority
description: "Default priority for non-critical services"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: best-effort
value: -100000
globalDefault: false
preemptionPolicy: Never
description: "Preemptible batch, dev, and experimental workloads"
```

Apply with:

```bash
kubectl apply -f priority-classes.yaml
```

Verify:

```bash
kubectl get priorityclasses
```


## Monitoring Priority and Preemption

```bash
# Check current priority classes
kubectl get pc

# Create a priority class imperatively (quick one-liner)
kubectl create priorityclass production-high --value=500000 --description="Revenue-critical production services"

# Patch an existing deployment to assign a priority class
kubectl patch deployment <deployment-name> -p '{"spec":{"template":{"spec":{"priorityClassName":"production-high"}}}}'

# Check a single pod's priority value
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.priority}'

# See which pods have which priority
kubectl get pods -A -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,NODE:.spec.nodeName,PRIORITY:.spec.priority,CLASS:.spec.priorityClassName | (read -r header; echo "$header"; sort -k4 -n -r)

# Check for recent preemption events
kubectl get events -A --field-selector reason=Preempted

# Find pods pending due to resource constraints (potential preemption triggers)
kubectl get pods -A --field-selector=status.phase=Pending -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,PRIORITY:.spec.priority,NOMINATED:.status.nominatedNodeName
```


## Validation Script

A quick script to audit priority classes and preemption events across the cluster:

```bash
#!/bin/bash
set -e

echo "=== Priority Class Validation ==="

echo -e "\nAvailable Priority Classes:"
kubectl get priorityclasses --sort-by='.value' \
  -o custom-columns=NAME:.metadata.name,VALUE:.value,DEFAULT:.globalDefault,PREEMPTION:.preemptionPolicy

echo -e "\n=== Pods with Priority Classes ==="
kubectl get pods --all-namespaces \
  -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,PRIORITY-CLASS:.spec.priorityClassName,PRIORITY-VALUE:.spec.priority,STATUS:.status.phase \
  | (read -r header; echo "$header"; sort -k4 -n -r)

echo -e "\n=== Node Resource Pressure ==="
kubectl top nodes 2>/dev/null || echo "(metrics-server not available)"

echo -e "\n=== Recent Preemption Events ==="
kubectl get events --all-namespaces --field-selector=reason=Preempted --sort-by='.lastTimestamp' 2>/dev/null | tail -10 || echo "No preemption events found"
```
