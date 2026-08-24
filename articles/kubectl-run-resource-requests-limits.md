# kubectl run with Resource Requests & Limits

How to set CPU and memory requests and limits when creating pods with `kubectl run`. The dedicated `--requests` and `--limits` flags were removed — use `--overrides` to inject resource specs inline.

## Why --overrides?

`kubectl run` no longer has built-in flags for CPU/memory requests and limits. The `--overrides` flag accepts a JSON patch that merges into the generated pod spec, giving full control over the resource block.

## CPU and Memory Units

| Resource | Unit | Examples |
|----------|------|----------|
| CPU | millicores or cores | `100m`, `250m`, `0.5`, `1`, `2` |
| Memory | bytes (with suffix) | `64Mi`, `128Mi`, `256Mi`, `1Gi`, `2Gi` |

- `1` CPU = `1000m` (millicores)
- `1Gi` = 1024 MiB, `1G` = 1000 MB

## One-Liner with --overrides

```bash
kubectl run limit-test --image=nginx \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "limit-test",
        "image": "nginx",
        "resources": {
          "requests": {
            "cpu": "500m",
            "memory": "128Mi"
          },
          "limits": {
            "cpu": "1",
            "memory": "256Mi"
          }
        }
      }]
    }
  }'
```

The container `name` and `image` inside `--overrides` must match the pod name and `--image` flag.

## Common Patterns

### Requests Only

```bash
kubectl run web --image=nginx \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "web",
        "image": "nginx",
        "resources": {
          "requests": {
            "cpu": "100m",
            "memory": "128Mi"
          }
        }
      }]
    }
  }'
```

### Limits Only

```bash
kubectl run web --image=nginx \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "web",
        "image": "nginx",
        "resources": {
          "limits": {
            "cpu": "500m",
            "memory": "256Mi"
          }
        }
      }]
    }
  }'
```

### Guaranteed QoS (Requests = Limits)

```bash
kubectl run critical --image=nginx \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "critical",
        "image": "nginx",
        "resources": {
          "requests": {
            "cpu": "1",
            "memory": "1Gi"
          },
          "limits": {
            "cpu": "1",
            "memory": "1Gi"
          }
        }
      }]
    }
  }'
```

### One-Off Debug Pod

```bash
kubectl run debug --image=busybox --restart=Never \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "debug",
        "image": "busybox",
        "command": ["sleep", "3600"],
        "resources": {
          "requests": {
            "cpu": "50m",
            "memory": "32Mi"
          },
          "limits": {
            "cpu": "100m",
            "memory": "64Mi"
          }
        }
      }]
    }
  }'
```

## Reusable Shell Function

Wrap the override pattern in a function for repeated use:

```bash
run_pod() {
  kubectl run "$1" --image="$2" \
    --overrides="{\"spec\":{\"containers\":[{\"name\":\"$1\",\"image\":\"$2\",\"resources\":{\"requests\":{\"cpu\":\"$3\",\"memory\":\"$4\"},\"limits\":{\"cpu\":\"$5\",\"memory\":\"$6\"}}}]}}"
}

# Usage: run_pod <name> <image> <cpu-req> <mem-req> <cpu-limit> <mem-limit>
run_pod limit-test nginx 500m 128Mi 1 256Mi
run_pod worker python:3.12 250m 256Mi 2 2Gi
```

## Dry Run — Generate YAML

Use `--dry-run=client -o yaml` to preview the manifest with resources applied:

```bash
kubectl run limit-test --image=nginx \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "limit-test",
        "image": "nginx",
        "resources": {
          "requests": {
            "cpu": "500m",
            "memory": "128Mi"
          },
          "limits": {
            "cpu": "1",
            "memory": "256Mi"
          }
        }
      }]
    }
  }' \
  --dry-run=client -o yaml
```

Output:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: limit-test
  name: limit-test
spec:
  containers:
  - image: nginx
    name: limit-test
    resources:
      requests:
        cpu: 500m
        memory: 128Mi
      limits:
        cpu: "1"
        memory: 256Mi
  restartPolicy: Always
```

## Verify

```bash
kubectl get pod limit-test
kubectl describe pod limit-test | grep -A 5 "Limits"
```

Example output:

```
    Limits:
      cpu:     1
      memory:  256Mi
    Requests:
      cpu:     500m
      memory:  128Mi
```

Or use jsonpath:

```bash
kubectl get pod limit-test -o jsonpath='{.spec.containers[0].resources}' | jq
```

## How Requests and Limits Affect Scheduling

| Field | Purpose | Effect |
|-------|---------|--------|
| `requests` | Minimum guaranteed resources | Used by the scheduler to find a node with enough capacity |
| `limits` | Maximum allowed resources | Enforced by kubelet — CPU is throttled, memory causes OOMKill |

### QoS Classes

The combination of requests and limits determines the pod's Quality of Service class:

| QoS Class | Condition |
|-----------|-----------|
| **Guaranteed** | All containers have requests = limits for both CPU and memory |
| **Burstable** | At least one container has a request or limit set, but not Guaranteed |
| **BestEffort** | No requests or limits set on any container |

## Interaction with LimitRange

If a `LimitRange` exists in the namespace:

- Missing requests/limits are filled with LimitRange defaults
- Values below the minimum or above the maximum are rejected

```bash
kubectl get limitrange -n <namespace>
kubectl describe limitrange -n <namespace>
```

## Interaction with ResourceQuota

If a `ResourceQuota` exists in the namespace:

- Pods must specify requests and limits for resources tracked by the quota
- Pod creation fails if it would exceed the quota

```bash
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota -n <namespace>
```

## Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Container `name` in overrides doesn't match pod name | Creates a second container or fails |
| Forgetting `image` inside the overrides JSON | Image field may be empty, pod fails to start |
| Setting requests higher than limits | Pod creation rejected by the API server |
| Memory limit too low | Container gets OOMKilled |
| CPU limit too low | Container is throttled, slow performance |
| No resources in a namespace with ResourceQuota | Pod creation rejected |
| Using removed `--requests`/`--limits` flags | Unknown flag error |

## Quick Reference

```bash
# Basic pod with resource boundaries
kubectl run web --image=nginx \
  --overrides='{"spec":{"containers":[{"name":"web","image":"nginx","resources":{"requests":{"cpu":"100m","memory":"128Mi"},"limits":{"cpu":"500m","memory":"256Mi"}}}]}}'

# Generate YAML for further editing
kubectl run web --image=nginx \
  --overrides='{"spec":{"containers":[{"name":"web","image":"nginx","resources":{"requests":{"cpu":"100m","memory":"128Mi"},"limits":{"cpu":"500m","memory":"256Mi"}}}]}}' \
  --dry-run=client -o yaml > web-pod.yaml

# Guaranteed QoS (requests = limits)
kubectl run critical --image=nginx \
  --overrides='{"spec":{"containers":[{"name":"critical","image":"nginx","resources":{"requests":{"cpu":"1","memory":"1Gi"},"limits":{"cpu":"1","memory":"1Gi"}}}]}}'

# Shell function for convenience
run_pod() {
  kubectl run "$1" --image="$2" \
    --overrides="{\"spec\":{\"containers\":[{\"name\":\"$1\",\"image\":\"$2\",\"resources\":{\"requests\":{\"cpu\":\"$3\",\"memory\":\"$4\"},\"limits\":{\"cpu\":\"$5\",\"memory\":\"$6\"}}}]}}"
}
run_pod myapp nginx 500m 128Mi 1 256Mi
```
