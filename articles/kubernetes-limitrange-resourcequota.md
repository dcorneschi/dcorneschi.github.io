# LimitRange and ResourceQuota in Kubernetes

Two mechanisms for controlling resource consumption at the namespace level — LimitRange sets per-object defaults and constraints, ResourceQuota caps total consumption for the namespace.

## LimitRange vs ResourceQuota

| Feature | LimitRange | ResourceQuota |
|---------|:----------:|:-------------:|
| Scope | Per-object (pod, container, PVC) | Entire namespace (aggregate) |
| Purpose | Set defaults and enforce min/max per resource | Cap total resources consumed |
| What it controls | Individual pod/container requests and limits | Sum of all requests/limits in the namespace |
| Enforcement | Admission controller (at creation time) | Admission controller (at creation time) |
| Applies to existing objects | No (only new/updated) | Yes (counts all existing objects) |

## LimitRange

A LimitRange defines:
- **Default** requests and limits for containers that don't specify them
- **Min/Max** allowed values per container or pod
- **MaxLimitRequestRatio** — the maximum ratio between limits and requests

### Basic LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: dev
spec:
  limits:
    - type: Container
      default:          # Applied when container has no limits set
        cpu: 500m
        memory: 512Mi
      defaultRequest:   # Applied when container has no requests set
        cpu: 100m
        memory: 128Mi
      min:              # Minimum allowed
        cpu: 50m
        memory: 64Mi
      max:              # Maximum allowed
        cpu: "2"
        memory: 2Gi
      maxLimitRequestRatio:   # limits / requests must be <= this
        cpu: "4"
        memory: "4"
```

### LimitRange Types

| Type | Controls |
|------|----------|
| `Container` | Per-container defaults, min, max |
| `Pod` | Per-pod totals (sum of all containers) |
| `PersistentVolumeClaim` | Per-PVC min/max storage size |

### Container Type (Most Common)

```yaml
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      min:
        cpu: 50m
        memory: 64Mi
      max:
        cpu: "4"
        memory: 4Gi
```

What happens when a pod is created without resources:

```yaml
# User submits:
containers:
  - name: app
    image: nginx
    # No resources specified

# After LimitRange admission:
containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: 100m        # defaultRequest applied
        memory: 128Mi    # defaultRequest applied
      limits:
        cpu: 500m        # default applied
        memory: 256Mi    # default applied
```

### Pod Type

Controls the total resources across all containers in a pod:

```yaml
spec:
  limits:
    - type: Pod
      max:
        cpu: "8"
        memory: 8Gi
      min:
        cpu: 100m
        memory: 128Mi
```

### PersistentVolumeClaim Type

```yaml
spec:
  limits:
    - type: PersistentVolumeClaim
      min:
        storage: 1Gi
      max:
        storage: 100Gi
```

Prevents teams from creating excessively large or tiny PVCs.

### How LimitRange Defaults Interact with Partial Specs

| User Sets | Result |
|-----------|--------|
| Nothing | `defaultRequest` and `default` (limits) applied |
| Only requests | User's requests kept, `default` limits applied |
| Only limits | User's limits kept, `defaultRequest` applied (or limits used as requests if defaultRequest not set) |
| Both requests and limits | Both kept as-is, validated against min/max |

### LimitRange Validation Errors

```sh
# Pod rejected — exceeds max
Error: pods "my-pod" is forbidden: maximum cpu usage per Container is 2, but limit is 4

# Pod rejected — below min
Error: pods "my-pod" is forbidden: minimum memory usage per Container is 64Mi, but request is 32Mi

# Pod rejected — ratio exceeded
Error: pods "my-pod" is forbidden: cpu max limit to request ratio per Container is 4, but provided ratio is 10
```

## ResourceQuota

A ResourceQuota limits the **total** resources a namespace can consume.

### Basic ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "10"         # Total CPU requests across all pods
    requests.memory: 20Gi      # Total memory requests
    limits.cpu: "20"           # Total CPU limits
    limits.memory: 40Gi        # Total memory limits
    pods: "50"                 # Maximum number of pods
    services: "10"             # Maximum services
    persistentvolumeclaims: "20"
    configmaps: "50"
    secrets: "50"
```

### Resource Types You Can Quota

| Resource | What It Limits |
|----------|---------------|
| `requests.cpu` | Sum of CPU requests across all pods |
| `requests.memory` | Sum of memory requests |
| `limits.cpu` | Sum of CPU limits |
| `limits.memory` | Sum of memory limits |
| `pods` | Total number of pods |
| `services` | Total services |
| `services.loadbalancers` | Total LoadBalancer services |
| `services.nodeports` | Total NodePort services |
| `persistentvolumeclaims` | Total PVCs |
| `requests.storage` | Total storage requested across PVCs |
| `<storageclass>.storageclass.storage.k8s.io/requests.storage` | Storage per StorageClass |
| `configmaps` | Total ConfigMaps |
| `secrets` | Total Secrets |
| `replicationcontrollers` | Total ReplicationControllers |
| `resourcequotas` | Total ResourceQuotas |
| `count/<resource>.<group>` | Count of any resource (e.g., `count/deployments.apps`) |

### ResourceQuota with Object Counts

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
  namespace: dev
spec:
  hard:
    count/deployments.apps: "20"
    count/statefulsets.apps: "5"
    count/jobs.batch: "100"
    count/cronjobs.batch: "10"
    count/ingresses.networking.k8s.io: "10"
```

### ResourceQuota with Storage Classes

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: dev
spec:
  hard:
    requests.storage: 500Gi
    persistentvolumeclaims: "30"
    gp3.storageclass.storage.k8s.io/requests.storage: 200Gi
    gp3.storageclass.storage.k8s.io/persistentvolumeclaims: "10"
```

### ResourceQuota Scoped by PriorityClass

Limit resources per priority tier within a namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: high-priority-quota
  namespace: production
spec:
  hard:
    requests.cpu: "50"
    requests.memory: 100Gi
    pods: "100"
  scopeSelector:
    matchExpressions:
      - scopeName: PriorityClass
        operator: In
        values:
          - critical
          - production-high
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: low-priority-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    pods: "50"
  scopeSelector:
    matchExpressions:
      - scopeName: PriorityClass
        operator: In
        values:
          - batch
          - best-effort
```

### ResourceQuota with Scopes

| Scope | Matches Pods That... |
|-------|---------------------|
| `Terminating` | Have `activeDeadlineSeconds` set |
| `NotTerminating` | Don't have `activeDeadlineSeconds` |
| `BestEffort` | Have QoS class BestEffort |
| `NotBestEffort` | Have QoS class Guaranteed or Burstable |
| `PriorityClass` | Match the specified priority classes |

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: besteffort-quota
  namespace: dev
spec:
  hard:
    pods: "20"
  scopes:
    - BestEffort
```

This limits BestEffort pods (no requests/limits) to 20 in the namespace.

## Using LimitRange + ResourceQuota Together

Best practice is to use both:

1. **LimitRange** ensures every pod has requests/limits (even if the developer forgets)
2. **ResourceQuota** ensures the namespace doesn't consume more than its fair share

Without LimitRange, ResourceQuota becomes hard to enforce — pods without requests/limits can bypass compute quotas.

### Complete Example: Dev Namespace

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      min:
        cpu: 50m
        memory: 64Mi
      max:
        cpu: "2"
        memory: 2Gi
    - type: PersistentVolumeClaim
      min:
        storage: 1Gi
      max:
        storage: 50Gi
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "30"
    services: "10"
    persistentvolumeclaims: "15"
    requests.storage: 200Gi
```

### Complete Example: Production Namespace

```yaml
---
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "1"
        memory: 1Gi
      defaultRequest:
        cpu: 250m
        memory: 256Mi
      min:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "8"
        memory: 8Gi
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    pods: "200"
    services: "50"
    services.loadbalancers: "5"
    persistentvolumeclaims: "100"
```

## Checking Quota Usage

```sh
# View quota usage in a namespace
kubectl get resourcequota -n dev
kubectl describe resourcequota compute-quota -n dev

# Example output:
# Name:            compute-quota
# Namespace:       dev
# Resource         Used    Hard
# --------         ----    ----
# limits.cpu       3500m   16
# limits.memory    4Gi     32Gi
# pods             7       30
# requests.cpu     1200m   8
# requests.memory  2Gi     16Gi

# View LimitRange
kubectl describe limitrange default-limits -n dev
```

## What Happens When Quota Is Exceeded

```sh
# Pod creation rejected:
Error from server (Forbidden): pods "my-pod" is forbidden:
  exceeded quota: compute-quota,
  requested: requests.cpu=500m,
  used: requests.cpu=7800m,
  limited: requests.cpu=8
```

The pod is simply not created. Existing pods are NOT evicted.

## ResourceQuota and Requests: The Gotcha

When a ResourceQuota for compute resources (cpu/memory) exists in a namespace, **every pod must specify requests and limits** — otherwise the admission controller rejects it.

```sh
# Without LimitRange, this fails when ResourceQuota exists:
kubectl run nginx --image=nginx -n dev
# Error: pods "nginx" is forbidden: failed quota: compute-quota:
#   must specify limits.cpu, limits.memory, requests.cpu, requests.memory
```

Solution: Always pair ResourceQuota with a LimitRange that sets defaults.

## Multi-Tenant Cluster Example

```yaml
# Team A namespace (large allocation)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "50"
    requests.memory: 100Gi
    pods: "100"
---
# Team B namespace (smaller allocation)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-b-quota
  namespace: team-b
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    pods: "50"
---
# Shared services (constrained)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: shared-quota
  namespace: shared
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    pods: "30"
    services.loadbalancers: "2"
```

## kubectl Commands

```sh
# Create a LimitRange
kubectl apply -f limitrange.yaml -n dev

# Create a ResourceQuota
kubectl apply -f resourcequota.yaml -n dev

# View all quotas in a namespace
kubectl get resourcequota -n dev -o yaml

# View all limit ranges
kubectl get limitrange -n dev -o yaml

# Check current usage vs limits
kubectl describe resourcequota -n dev

# Check what defaults are applied
kubectl describe limitrange -n dev

# Test if a pod would be accepted (dry-run)
kubectl run test --image=nginx --dry-run=server -n dev -o yaml

# Find namespaces approaching quota limits
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  echo "=== $ns ==="
  kubectl describe resourcequota -n $ns 2>/dev/null | grep -E "Used|Hard|Resource" || echo "No quota"
done
```

## Gotchas

- **LimitRange only applies to new pods**: Changing a LimitRange doesn't update existing running pods. You must recreate them.
- **ResourceQuota requires requests on every pod**: If a compute quota exists, pods without explicit requests/limits are rejected (unless LimitRange provides defaults).
- **Quota counts init containers separately**: Init container resource requests count toward the quota, using `max(init containers)` rather than `sum`.
- **DaemonSet pods count toward quota**: DaemonSets in a quota'd namespace can fail if the namespace quota is full.
- **Quota doesn't prevent overcommit**: ResourceQuota limits the total requests/limits declared, not actual usage. Pods can still OOM if limits are set too high.
- **Multiple quotas in one namespace**: You can have multiple ResourceQuota objects — all are enforced (most restrictive wins per resource).
- **Quota blocks Deployments silently**: If a Deployment can't create pods due to quota, the ReplicaSet scales to 0 with an event — but the Deployment itself shows no error.
- **BestEffort scope + compute quota**: A pod with no requests/limits is `BestEffort`. If you have a compute quota, BestEffort pods are rejected. Either add requests or use a LimitRange.
- **LimitRange max vs ResourceQuota**: LimitRange max is per-container, ResourceQuota hard is per-namespace total. Both must be satisfied.
