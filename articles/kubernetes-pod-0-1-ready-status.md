# Why Pod Shows 0/1 Ready Status

Diagnosing pods stuck at `0/1 Ready` — common causes, troubleshooting flow, and fixes for containers that start but never become ready.

## What 0/1 Ready Means

```
NAME        READY   STATUS    RESTARTS   AGE
myapp-abc   0/1     Running   0          5m
```

- `0/1` means 0 out of 1 containers are reporting ready
- The format is `<ready_containers>/<total_containers>`
- The pod is running (container started) but Kubernetes considers it not ready to receive traffic
- The pod is removed from Service endpoints and won't get requests

## Is 0/1 Because of Readiness or Liveness?

The `0/1` status is **only** affected by the readiness probe, not liveness.

### Key Differences

| Aspect | Readiness Probe | Liveness Probe |
|--------|----------------|----------------|
| Controls | READY column (`0/1` or `1/1`) | Container **restarts** |
| Failure action | Pod removed from Service endpoints | Container killed and restarted |
| Pod status | Stays `Running` but `0/1` ready | RESTARTS count increases |
| Symptom | No traffic routed to pod | `CrashLoopBackOff` or restart count climbing |

### Visual Example

```bash
# Readiness failing (stays running, not ready):
NAME       READY   STATUS    RESTARTS   AGE
my-pod     0/1     Running   0          5m

# Liveness failing (gets restarted):
NAME       READY   STATUS    RESTARTS   AGE
my-pod     1/1     Running   3          5m
                             ↑ increasing restarts

# Both failing:
NAME       READY   STATUS             RESTARTS   AGE
my-pod     0/1     CrashLoopBackOff   5          5m
```

**Bottom line:** If you see `0/1` with `Running` status and **no restarts**, it's definitely a **readiness probe issue**.

## Quick Diagnosis Flow

```
Pod 0/1 Running
  │
  ├─ Readiness probe defined?
  │    ├─ Yes → Probe is failing (most common cause)
  │    └─ No  → Container itself has a problem (see below)
  │
  ├─ kubectl describe pod <name>
  │    ├─ Check Events section for probe failures
  │    ├─ Check Conditions section for False entries
  │    └─ Check container State and Last State
  │
  └─ kubectl logs <name>
       └─ Application-level errors preventing startup
```

## No Readiness Probe Configured But Still 0/1

Without a readiness probe, Kubernetes assumes the container is ready **as soon as it's running**. If it shows `0/1`, the container itself has a problem starting or staying alive — not a readiness issue.

### 1. Container Not Running Yet

The container might still be in `Waiting` or `ContainerCreating` state:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look for:
- **State**: `Waiting` instead of `Running`
- **Reason**: `ContainerCreating`, `PodInitializing`, `ImagePullBackOff`, `CrashLoopBackOff`, `ErrImagePull`

### 2. Init Containers Still Running or Failed

Init containers must complete before the main container starts:

```bash
# Check init container status
kubectl get pod <pod-name> -o jsonpath='{.status.initContainerStatuses[*].state}'

# See init container logs
kubectl logs <pod-name> -c <init-container-name>
```

Example:

```bash
$ kubectl get pod mypod -o jsonpath='{.status.initContainerStatuses[*].state.waiting.reason}'
CrashLoopBackOff
```

Fix: Debug init container with `kubectl logs <pod> -c <init-container-name>`

### 3. Container Crashed Immediately

Container starts but exits immediately:

```bash
# Check container state
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].state}'

# View logs (even if container exited)
kubectl logs <pod-name>

# View previous logs if restarting
kubectl logs <pod-name> --previous
```

Example:

```bash
$ kubectl describe pod mypod
...
State:          Waiting
Reason:         CrashLoopBackOff
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
```

Fix: Check logs for application errors.

### 4. Image Pull Issues

```bash
kubectl describe pod <pod-name> | grep -A 5 Events
```

Common errors: `ErrImagePull`, `ImagePullBackOff`, `manifest unknown`, `unauthorized`

Fix: Correct the image name/tag or push the image to the registry.

### 5. Resource Constraints

Pod can't be scheduled due to insufficient resources:

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.conditions[*]}'
```

Look for: `Unschedulable`, `Insufficient memory`, `Insufficient cpu`

## Common Causes (With Readiness Probe)

### 1. Readiness Probe Failing

The most frequent reason. The application is running but the probe endpoint returns non-200 or the check times out.

```bash
kubectl describe pod myapp | grep -A 10 "Readiness"
```

Look for:

```
Warning  Unhealthy  Readiness probe failed: Get "http://10.0.1.5:8080/health": dial tcp 10.0.1.5:8080: connect: connection refused
```

**Common probe issues:**

| Issue | Cause | Fix |
|-------|-------|-----|
| Connection refused | App not listening on probe port | Match port in probe to app's listening port |
| Timeout | App too slow to respond | Increase `timeoutSeconds` or fix app performance |
| 404 | Wrong path | Fix `path` in probe definition |
| 503 | App returns unhealthy | Check app logs for dependency failures |
| Wrong protocol | HTTP probe hitting HTTPS endpoint | Switch to `httpGet.scheme: HTTPS` |

**Example probe mismatch:**

```yaml
# App listens on port 3000, but probe checks 8080
readinessProbe:
  httpGet:
    path: /health
    port: 8080  # Wrong — should be 3000
```

### 2. Application Still Starting Up

The app needs time to initialize (loading data, connecting to databases) but the probe fires before it's ready.

**Fix — add initialDelaySeconds:**

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 5
```

**Better fix — use a startup probe (Kubernetes 1.20+):**

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
# Gives the app up to 300s (30 × 10) to start

readinessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 5
```

Startup probe behavior:
- While startup probe is running, liveness/readiness are **disabled**
- Once startup succeeds, liveness/readiness begin
- If startup fails after all attempts, container restarts

### 3. Dependency Not Available

The app starts but can't reach a database, cache, or external service it depends on.

```bash
kubectl logs myapp
```

Look for connection errors:

```
Error: connect ECONNREFUSED 10.0.2.15:5432
Failed to connect to Redis at redis-svc:6379
```

**Fixes:**
- Verify the dependent service is running and has endpoints
- Check NetworkPolicy rules aren't blocking traffic
- Use init containers to wait for dependencies before the main container starts

### 4. OOMKilled During Startup

The container starts but gets killed before it can respond to probes.

```bash
kubectl describe pod myapp | grep -A 3 "Last State"
```

```
    Last State:  Terminated
      Reason:    OOMKilled
      Exit Code: 137
```

**Fix — increase memory limits:**

```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

### 5. Readiness Gate Not Satisfied

Custom readiness gates (used by ALB Ingress Controller, Istio, etc.) block readiness even when the container is healthy.

```bash
kubectl get pod myapp -o jsonpath='{.spec.readinessGates}'
kubectl get pod myapp -o jsonpath='{.status.conditions}' | jq
```

Look for conditions with `status: "False"` that match a readiness gate:

```json
{
  "type": "target-health.elbv2.k8s.aws/my-tg",
  "status": "False"
}
```

Pod is ready only when **all** conditions are met:
- All readiness probes succeed
- All readiness gates are True

**Fixes:**
- Verify the controller managing the gate is running
- Check the controller's logs for registration errors
- Ensure Security Groups allow health check traffic from the load balancer

### 6. ConfigMap or Secret Not Mounted

If a required ConfigMap or Secret doesn't exist, the pod may start with an empty or failed mount, causing the app to crash or fail its probe.

```bash
kubectl describe pod myapp | grep -A 5 "Events"
```

```
Warning  FailedMount  Unable to attach or mount volumes: unmounted volumes=[config-vol]
```

Fix: Create the missing resource or fix the volume reference.

### 7. Wrong Container Command

The container starts but the entrypoint/command doesn't launch the actual application.

```bash
kubectl logs myapp
kubectl get pod myapp -o jsonpath='{.spec.containers[0].command}' | jq
kubectl get pod myapp -o jsonpath='{.spec.containers[0].args}' | jq
```

## Probe Types & Use Cases

### HTTP GET Probe

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
    httpHeaders:
    - name: X-Custom-Header
      value: probe
    scheme: HTTPS
  timeoutSeconds: 3
```

Response codes: `200-399` = success, `400+` = failure, timeout/connection refused = failure.

Best for: REST APIs, web services.

### TCP Socket Probe

```yaml
readinessProbe:
  tcpSocket:
    port: 3306
  periodSeconds: 5
```

Attempts TCP connection — established = success, refused/timeout = failure.

Best for: Databases, message queues, any TCP service.

### Exec Probe

```yaml
readinessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - pg_isready -U postgres && test -f /tmp/app-ready
  timeoutSeconds: 5
```

Exit code 0 = success, non-zero = failure.

Best for: Complex checks, database readiness, file-based signals.

### gRPC Probe (v1.24+)

```yaml
readinessProbe:
  grpc:
    port: 9090
    service: my.service.Health
```

Best for: gRPC services using health checking protocol.

## Probe Timing Parameters

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5    # Wait before first probe
  periodSeconds: 10         # Probe every 10 seconds
  timeoutSeconds: 1         # Probe must respond within 1s
  successThreshold: 1       # 1 success = ready (default)
  failureThreshold: 3       # 3 failures = not ready (default)
```

**State transitions:**
- **Ready → Not Ready**: Requires `failureThreshold` consecutive failures
- **Not Ready → Ready**: Requires `successThreshold` consecutive successes
- Each probe is independent — partial success doesn't count

## How Readiness Affects Service Endpoints

```
┌─────────────────┐
│   kubelet       │ ← Runs readiness probe on schedule
└────────┬────────┘
         │ Probe Result
         ↓
┌─────────────────┐
│ Pod Status      │ ← Updates ContainersReady & Ready conditions
└────────┬────────┘
         │ Watch Events
         ↓
┌─────────────────┐
│ Endpoints       │ ← Service controller adds/removes pod IP
│ Controller      │
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│ Service         │ ← Traffic routing updated
│ Endpoints       │
└─────────────────┘
```

### Endpoint Controller Behavior

- **Probe succeeds**: Pod IP added to `addresses[]` within ~2 seconds
- **Probe fails**: Pod IP moved to `notReadyAddresses[]`
- **Pod deleted**: IP removed from endpoints immediately
- **Service selector**: Only includes pods matching labels **AND** ready

### Traffic Impact Timeline

```
T+0s:  Readiness probe fails
T+1s:  kubelet updates pod status (Ready: False)
T+2s:  Endpoints controller updates Service endpoints
T+2s:  kube-proxy updates iptables/IPVS rules
T+2s:  New connections stop routing to pod
T+2s+: Existing connections may continue (depends on proxy mode)
```

### View Endpoint State

```bash
# See which pods are ready/not ready
kubectl get endpoints myservice -o yaml
```

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: myservice
subsets:
- addresses:
  - ip: 10.244.1.10         # Ready pods
    targetRef:
      kind: Pod
      name: pod-1
  notReadyAddresses:         # Not ready pods (tracked but not routed to)
  - ip: 10.244.1.12
    targetRef:
      kind: Pod
      name: pod-3
  ports:
  - port: 8080
```

```bash
# Quick check
kubectl get endpoints myservice -o jsonpath='{.subsets[*].addresses[*].ip}'          # Ready
kubectl get endpoints myservice -o jsonpath='{.subsets[*].notReadyAddresses[*].ip}'  # Not ready
```

## Zero-Downtime Deployments

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0       # Always maintain capacity
      maxSurge: 1             # Max 1 extra pod during update
  template:
    spec:
      containers:
      - name: app
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          periodSeconds: 2      # Fast detection
          failureThreshold: 1   # Immediate removal
```

**Flow during rolling update:**
1. New pod created, enters Pending
2. New pod starts, readiness probe begins
3. After `successThreshold` successes, pod added to endpoints
4. Old pod receives SIGTERM (still in endpoints briefly)
5. Old pod preStop hook runs (sleep allows connection draining)
6. Old pod removed from endpoints
7. Old pod terminates

## Advanced Pattern: Graceful Startup with All Probes

```yaml
spec:
  containers:
  - name: app
    image: myapp
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
      failureThreshold: 2

    livenessProbe:
      httpGet:
        path: /live
        port: 8080
      periodSeconds: 10
      failureThreshold: 3
```

## Readiness Gates

Custom readiness conditions beyond probes:

```yaml
spec:
  readinessGates:
  - conditionType: "example.com/feature-flag-enabled"
  - conditionType: "example.com/warmup-complete"

  containers:
  - name: app
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
```

External controller must update these conditions:

```bash
kubectl patch pod mypod --type='json' -p='[{
  "op": "add",
  "path": "/status/conditions/-",
  "value": {
    "type": "example.com/warmup-complete",
    "status": "True"
  }
}]'
```

## Application-Level Health Endpoint Example

```go
func readinessHandler(w http.ResponseWriter, r *http.Request) {
    // Check database connection
    if err := db.Ping(); err != nil {
        w.WriteHeader(503)
        return
    }

    // Check required dependencies
    if !cache.IsConnected() {
        w.WriteHeader(503)
        return
    }

    // Check if still accepting requests (shutdown signal received?)
    if shuttingDown {
        w.WriteHeader(503)
        return
    }

    w.WriteHeader(200)
    w.Write([]byte("ready"))
}
```

## Common Gotchas

| Gotcha | Fix |
|--------|-----|
| Probe timeout too short — app responds slowly under load | Increase `timeoutSeconds` (default is 1s) |
| `initialDelaySeconds` too short — app not fully started | Increase delay or use a startup probe |
| Probing wrong port — container exposes multiple ports | Check with `kubectl exec mypod -- netstat -tlnp` |
| `hostNetwork: true` confusion | Set `host: 127.0.0.1` explicitly in probe |
| Readiness depends on external service — cascading failures | Avoid external dependency checks in readiness; use circuit breakers |
| Requests higher than node capacity | Pod stays Pending, never reaches Running |

## Troubleshooting Commands

```bash
# 1. Overall pod status
kubectl get pod <pod-name> -o wide

# 2. Detailed description (most useful)
kubectl describe pod <pod-name>

# 3. Container logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous

# 4. Container states
kubectl get pod <pod-name> -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.state}{"\n"}{end}'

# 5. Pod conditions
kubectl get pod <pod-name> -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\n"}{end}'

# 6. Events for pod
kubectl get events --field-selector involvedObject.name=<pod-name> --sort-by='.lastTimestamp'

# 7. Check probe configuration
kubectl get pod <pod-name> -o json | jq '.spec.containers[].readinessProbe'

# 8. Monitor probe events in real-time
kubectl get events --watch --field-selector involvedObject.name=<pod-name>

# 9. Filter for readiness events
kubectl get events --field-selector reason=Unhealthy,involvedObject.name=<pod-name>
```

### Test Probes Manually

```bash
# HTTP probe
kubectl exec <pod-name> -- curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health

# TCP probe
kubectl exec <pod-name> -- nc -zv localhost 8080

# Exec probe
kubectl exec <pod-name> -- /bin/sh -c 'pg_isready -U postgres'

# From another pod
kubectl run curl-test --image=curlimages/curl --restart=Never --rm -it -- \
  curl -s http://<pod-ip>:8080/health
```

## Pod Conditions Reference

```bash
kubectl get pod myapp -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\n"}{end}'
```

| Condition | Meaning |
|-----------|---------|
| PodScheduled | Pod assigned to a node |
| Initialized | Init containers completed |
| ContainersReady | All containers passed readiness |
| Ready | Pod is fully ready (containers + readiness gates) |

A pod shows `0/1` when `ContainersReady` or `Ready` is `False`.

## Fix Checklist

```
□ Check events: kubectl describe pod <name>
□ Check logs: kubectl logs <name>
□ Is a readiness probe configured? If not — container has a startup problem
□ Verify probe config matches app (port, path, scheme)
□ Test probe endpoint manually from inside the pod
□ Check if app needs more startup time (initialDelaySeconds / startupProbe)
□ Check for dependency failures (DB, cache, external APIs)
□ Check for OOMKill in last state
□ Check readiness gates (ALB, Istio, custom controllers)
□ Verify ConfigMaps/Secrets exist
□ Check container command/args
□ Check Service endpoints to confirm pod is removed
```
