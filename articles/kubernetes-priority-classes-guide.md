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

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

## Monitoring Priority and Preemption

```bash
# Check current priority classes
kubectl get pc

# Create a priority class imperatively (quick one-liner)
kubectl create priorityclass production-high --value=500000 --description="Revenue-critical production services"

# Patch an existing deployment to assign a priority class
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"priorityClassName":"production-high"}}}}'

# Check a single pod's priority value
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.priority}'

# View pod priority with node placement
kubectl get pods -A -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,PRIORITY:.spec.priority,NODE:.spec.nodeName,STATUS:.status.phase

# See which pods have which priority
kubectl get pods -A -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,PRIORITY:.spec.priority,CLASS:.spec.priorityClassName | sort -k3 -n -r

# Check for recent preemption events
kubectl get events -A --field-selector reason=Preempted
kubectl get events -A --field-selector reason=Preempting

# Find pods pending due to resource constraints (potential preemption triggers)
kubectl get pods -A --field-selector=status.phase=Pending -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,PRIORITY:.spec.priority,NOMINATED:.status.nominatedNodeName
```

---

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

---

## Summary

| What to do | Why |
|------------|-----|
| Create 3–5 custom PriorityClasses | Give the scheduler a hierarchy to make intelligent preemption decisions |
| Keep values well below 1 billion | Avoid conflicting with system-critical components |
| Set a `globalDefault` | Ensure pods without explicit priority aren't at 0 |
| Use `preemptionPolicy: Never` for batch | Prevent batch jobs from killing running services |
| Assign priority to monitoring/platform | Ensure observability survives resource pressure |
| Combine with PDBs | Priority handles preemption; PDBs handle voluntary disruptions (drains) |
