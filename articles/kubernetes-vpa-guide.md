# Kubernetes Vertical Pod Autoscaler (VPA)

Related: [VPA Documentation](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) | [HPA with scaleDown Behavior](articles/kubernetes-hpa-scaledown-behavior.md)

## What VPA Does

The Vertical Pod Autoscaler automatically adjusts CPU and memory **requests** (and optionally limits) for pods based on observed usage. Instead of adding more pods (HPA), VPA right-sizes each pod.

```
VPA observes usage → recommends new requests → evicts pod → pod restarts with updated requests
```

## VPA vs HPA

| Aspect | VPA | HPA |
|--------|-----|-----|
| **Scales** | Pod resources (CPU/memory requests) | Pod count (replicas) |
| **Direction** | Vertical — bigger/smaller pods | Horizontal — more/fewer pods |
| **Disruption** | Evicts and recreates pods | No pod disruption |
| **Best for** | Workloads that can't scale horizontally (stateful, single-instance) | Stateless workloads with variable traffic |
| **Metric source** | Historical usage from metrics-server | Real-time metrics |

> **Important:** Don't use VPA and HPA on the same metric (e.g., both on CPU). They'll fight each other. You can combine them if HPA uses custom metrics and VPA manages CPU/memory.

## Components

VPA consists of three components:

| Component | Role |
|-----------|------|
| **Recommender** | Watches resource usage and computes recommended requests |
| **Updater** | Evicts pods that need resource updates (when `updateMode: Auto`) |
| **Admission Controller** | Mutates pod specs at creation time to apply recommended resources |

```
┌─────────────┐     ┌──────────┐     ┌──────────────────────┐
│ Recommender │────▶│ Updater  │────▶│ Evicts pods needing  │
│ (observes)  │     │          │     │ resource changes     │
└─────────────┘     └──────────┘     └──────────────────────┘
        │
        ▼
┌──────────────────────┐
│ Admission Controller │──▶ Mutates new pods with recommended resources
└──────────────────────┘
```

## Installation

VPA is not built into Kubernetes — you need to install it separately.

```bash
# Clone the autoscaler repository
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# Install VPA components
./hack/vpa-up.sh

# Verify installation
kubectl get pods -n kube-system | grep vpa
```

On managed clusters:
- **EKS** — Install manually or use the [EKS add-on](https://docs.aws.amazon.com/eks/latest/userguide/vertical-pod-autoscaler.html)
- **GKE** — Built-in, enable via cluster settings
- **AKS** — Available as an add-on

## VPA Resource Structure

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"            # Off | Initial | Recreate | Auto
  resourcePolicy:
    containerPolicies:
    - containerName: "*"          # Apply to all containers
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: 2000m
        memory: 4Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits   # RequestsOnly | RequestsAndLimits
```

## Update Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| `Off` | Only generates recommendations, never applies them | Monitoring — see what VPA would do without risk |
| `Initial` | Applies recommendations only when pods are created | Safe for production — no evictions, only new pods get updated resources |
| `Recreate` | Evicts and recreates pods when recommendations change significantly | When you need auto-tuning but want explicit eviction |
| `Auto` | Currently same as `Recreate`, may support in-place updates in the future | Full automation |
| `InPlaceOrRecreate` | Patches running pods via `/resize` subresource without eviction; falls back to recreate if the node lacks capacity | Stateful apps, long-running jobs, or anything sensitive to restarts (requires K8s 1.35+) |

> **Production tip:** Start with `Off` to observe recommendations, then move to `Initial` or `Auto` once you trust the values.

## In-Place Pod Resize (Kubernetes 1.35+)

Starting with Kubernetes 1.35, the In-Place Pod Resize feature is stable. It allows changing CPU and memory requests/limits on a running pod without deleting or recreating it — the kubelet updates the container's cgroup directly.

### How It Works with VPA

```
metrics-server → VPA Recommender (calculates target) → VPA Updater (patches pod via /resize subresource) → Kubelet (updates cgroup) → Pod runs with new resources
```

With `updateMode: InPlaceOrRecreate`:
1. VPA Recommender detects the pod is over/under-provisioned
2. VPA Updater sends a patch request to the `/resize` subresource (instead of evicting)
3. Kubelet updates the container's cgroup on the node
4. If the node doesn't have enough capacity, VPA falls back to eviction and reschedules on another node

### Container resizePolicy

The `resizePolicy` field tells the kubelet what to do when a resource changes:

```yaml
spec:
  containers:
  - name: my-app
    image: my-image
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 100m
        memory: 256Mi
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired        # Update cgroup without restart
    - resourceName: memory
      restartPolicy: RestartContainer   # Restart container after memory change
```

| restartPolicy | Behavior |
|---------------|----------|
| `NotRequired` | Cgroup is updated while the container keeps running (default) |
| `RestartContainer` | Container is restarted after the resource change (pod stays running) |

> **When to use `RestartContainer`:** JVM-based apps read max heap size at startup. Increasing memory cgroup without restart means the JVM won't use the extra memory. Same applies to any runtime that reads resource limits only once at init.

### VPA with InPlaceOrRecreate Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-resize
  template:
    metadata:
      labels:
        app: nginx-resize
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 50m
            memory: 64Mi
        resizePolicy:
        - resourceName: cpu
          restartPolicy: NotRequired
        - resourceName: memory
          restartPolicy: NotRequired
---
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  updatePolicy:
    updateMode: "InPlaceOrRecreate"
    minReplicas: 1
  resourcePolicy:
    containerPolicies:
    - containerName: nginx
      minAllowed:
        cpu: 100m
        memory: 200Mi
      maxAllowed:
        cpu: 500m
        memory: 500Mi
      controlledResources: ["cpu", "memory"]
```

> **Important:** Always set initial `requests` and `limits` on your pods. Without them, Kubernetes assigns QoS class `BestEffort`. When VPA later adds a request, it tries to change the QoS to `Burstable` — but QoS class is immutable on a running pod, forcing a recreate instead of in-place resize.

### Downsizing Considerations

- **CPU downsizing** — CPU is compressible. The kernel throttles it without killing the process. Safe to downsize.
- **Memory downsizing** — Memory is non-compressible. If the container is using more memory than the new limit, the pod gets OOMKilled. Use `restartPolicy: RestartContainer` for memory to handle this gracefully.

### Prerequisites for In-Place Resize

```bash
# Verify nodes use cgroup v2 (required for in-place resize)
# SSH into a node and run:
stat -fc %T /sys/fs/cgroup/
# Output should be: cgroup2fs
# If it says tmpfs, nodes are on cgroup v1 — in-place resize won't work
```

Source: [VPA In-Place Pod Resize (DevOpsCube)](https://devopscube.com/vpa-in-place-pod-resize/)

## Reading Recommendations

```bash
# View VPA recommendations
kubectl describe vpa my-app-vpa
```

The output includes:

```yaml
recommendation:
  containerRecommendations:
  - containerName: my-app
    lowerBound:                    # Minimum resources to avoid OOM/throttling
      cpu: 100m
      memory: 128Mi
    target:                        # Recommended values (what VPA would set)
      cpu: 250m
      memory: 256Mi
    uncappedTarget:                # Ideal values ignoring min/max constraints
      cpu: 250m
      memory: 256Mi
    upperBound:                    # Maximum resources VPA might recommend
      cpu: 500m
      memory: 512Mi
```

| Field | Meaning |
|-------|---------|
| `lowerBound` | Minimum to prevent resource pressure |
| `target` | What VPA recommends (constrained by `minAllowed`/`maxAllowed`) |
| `uncappedTarget` | What VPA would recommend without constraints |
| `upperBound` | Upper estimate for occasional spikes |

## Examples

### Recommendation Only (Safe Start)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"
```

### Auto-Tuning with Guardrails

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: my-app
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4000m
        memory: 8Gi
    - containerName: sidecar
      mode: "Off"                 # Don't touch the sidecar container
```

### VPA for a StatefulSet

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: postgres-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: postgres
  updatePolicy:
    updateMode: "Initial"         # Only apply on new pods (safe for databases)
  resourcePolicy:
    containerPolicies:
    - containerName: postgres
      minAllowed:
        cpu: 500m
        memory: 1Gi
      maxAllowed:
        cpu: 8000m
        memory: 32Gi
      controlledValues: RequestsOnly   # Don't touch limits
```

## VPA + HPA Together (Multidimensional Autoscaling)

You can use both if they don't compete on the same metric:

```yaml
# HPA scales replicas based on custom metric (requests per second)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
---
# VPA right-sizes CPU/memory requests
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: my-app
      controlledResources: ["cpu", "memory"]
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2000m
        memory: 4Gi
```

> **Rule:** HPA on custom/external metrics + VPA on CPU/memory = safe combination.
> HPA on CPU + VPA on CPU = conflict (VPA changes requests, which changes HPA's utilization percentage).

## Common Pitfalls

1. **Pod eviction disruption** — In `Auto`/`Recreate` mode, VPA evicts pods to apply new resources. Use PodDisruptionBudgets to prevent all pods from being evicted simultaneously.
2. **No in-place resize (yet)** — VPA must restart pods to change resources. Kubernetes 1.27+ has in-place resource resize (alpha), but VPA doesn't use it yet.
3. **VPA + HPA on CPU** — They conflict. VPA changes requests, which changes the utilization percentage HPA uses. Pods scale up/down erratically.
4. **Missing metrics-server** — VPA needs metrics-server running. Without it, recommendations stay empty.
5. **History window** — VPA needs time to collect data (default: 8 days lookback). Recommendations may be inaccurate in the first hours.
6. **OOM before VPA reacts** — If memory usage spikes faster than VPA can react, pods get OOMKilled. Set `minAllowed.memory` high enough for startup peaks.
7. **Limits ratio** — With `controlledValues: RequestsAndLimits`, VPA maintains the original requests-to-limits ratio. If your ratio was wrong initially, VPA perpetuates it.

## Useful Commands

```bash
# List all VPA objects
kubectl get vpa

# View recommendations for a specific VPA
kubectl describe vpa my-app-vpa

# Check VPA status and conditions
kubectl get vpa my-app-vpa -o yaml | grep -A 20 status

# View which pods were evicted by VPA (check events)
kubectl get events --field-selector reason=EvictedByVPA

# Check VPA admission controller is running
kubectl get pods -n kube-system | grep vpa-admission

# Check VPA recommender logs
kubectl logs -n kube-system -l app=vpa-recommender --tail=50

# Get recommendations in JSON (useful for scripting)
kubectl get vpa my-app-vpa -o jsonpath='{.status.recommendation.containerRecommendations[0].target}'
```
