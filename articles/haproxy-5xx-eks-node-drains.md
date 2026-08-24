# HAProxy 5xx Errors During EKS Node Drains

Why HAProxy returns 502/503/504 errors when EKS nodes are drained — the race condition between pod termination, endpoint removal, and HAProxy backend updates.

## The Problem

During EKS node group updates or autoscaler scale-downs, nodes are cordoned and drained. Pods receive SIGTERM and start shutting down, but HAProxy may still route traffic to those pods for a few seconds — causing 5xx errors.

```
Timeline:
T+0s:  kubectl drain starts
T+0s:  Pod receives SIGTERM
T+0s:  Pod begins graceful shutdown
T+1s:  Pod removed from Service endpoints
T+2s:  Endpoints controller updates
T+3s:  HAProxy picks up new backend list
T+0s-3s: HAProxy still sends traffic to dying pod → 502/503
```

## Why It Happens

### The Race Condition

Three systems must coordinate, but they don't happen simultaneously:

1. **Pod termination** — Pod receives SIGTERM and starts shutting down
2. **Endpoint removal** — kube-proxy/endpoints controller removes the pod IP from the Service
3. **HAProxy config reload** — HAProxy Ingress Controller detects the endpoint change and updates its backend list

The gap between step 1 (pod starts dying) and step 3 (HAProxy stops routing to it) is where 5xx errors occur.

```
┌─────────────────────────────────────────────────────────────┐
│                     The Race Condition                      │
│                                                             │
│  kubelet           endpoints controller     HAProxy         │
│     │                     │                    │            │
│     │ SIGTERM pod         │                    │            │
│     ├─────────────►       │                    │            │
│     │                     │                    │            │
│     │              removes pod IP              │            │
│     │              from endpoints              │            │
│     │                     ├─────────────►      │            │
│     │                     │              detects change     │
│     │                     │              reloads config     │
│     │                     │                    │            │
│     │◄────── 2-5 seconds gap ──────────────────►            │
│     │        (5xx errors happen here)          │            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### HAProxy-Specific Factors

- HAProxy reloads are not instantaneous — config generation + reload takes 1-3 seconds
- During reload, existing connections to the old backend continue until timeout
- If the backend pod closes the connection before HAProxy realizes, you get a 502
- High-traffic services are more likely to hit the window

## Error Types During Drains

| Error | Cause |
|-------|-------|
| 502 Bad Gateway | HAProxy sent request to pod, but pod closed the connection (already shutting down) |
| 503 Service Unavailable | HAProxy has no available backends (all pods on draining node) |
| 504 Gateway Timeout | Pod received request but didn't respond before timeout (busy shutting down) |

## The Fix: preStop Hook + Graceful Shutdown

### 1. Add a preStop Hook (Delay Before SIGTERM)

The preStop hook runs **before** SIGTERM is sent to the container. By sleeping, you give endpoints time to propagate before the pod starts shutting down.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: app
        image: myapp:latest
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
```

**How this fixes the race:**

```
T+0s:   Pod marked for deletion
T+0s:   preStop hook starts (sleep 15)
T+0s:   Pod removed from endpoints (happens in parallel)
T+2s:   HAProxy detects endpoint removal, reloads config
T+3s:   HAProxy stops routing to this pod
T+15s:  preStop completes, SIGTERM sent to container
T+15s+: Container performs graceful shutdown
```

By the time the container receives SIGTERM, HAProxy has already stopped sending traffic to it.

### 2. Application-Level Graceful Shutdown

Your application should handle SIGTERM properly:

```go
// Go example
func main() {
    srv := &http.Server{Addr: ":8080"}
    
    go func() {
        sigCh := make(chan os.Signal, 1)
        signal.Notify(sigCh, syscall.SIGTERM)
        <-sigCh
        
        // Stop accepting new connections
        // Finish in-flight requests (with timeout)
        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        srv.Shutdown(ctx)
    }()
    
    srv.ListenAndServe()
}
```

```python
# Python (Flask/Gunicorn)
# Gunicorn handles SIGTERM gracefully by default:
# - Stops accepting new connections
# - Waits for workers to finish (graceful_timeout)
gunicorn --graceful-timeout 30 --timeout 60 app:app
```

### 3. Readiness Probe Fails Fast on Shutdown

Make your readiness probe return unhealthy immediately when SIGTERM is received:

```go
var shuttingDown atomic.Bool

func readinessHandler(w http.ResponseWriter, r *http.Request) {
    if shuttingDown.Load() {
        w.WriteHeader(503)
        return
    }
    w.WriteHeader(200)
}

func main() {
    go func() {
        sigCh := make(chan os.Signal, 1)
        signal.Notify(sigCh, syscall.SIGTERM)
        <-sigCh
        shuttingDown.Store(true) // Readiness fails immediately
        // ... graceful shutdown
    }()
}
```

This makes the pod get removed from endpoints faster (next readiness check).

## Complete Solution

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0        # Never reduce capacity during update
      maxSurge: 1
  template:
    spec:
      terminationGracePeriodSeconds: 60   # Total budget: preStop + shutdown

      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080

        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          periodSeconds: 2          # Fast detection of unready state
          failureThreshold: 1       # Remove from endpoints after 1 failure

        livenessProbe:
          httpGet:
            path: /live
            port: 8080
          periodSeconds: 10
          failureThreshold: 3

        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]

        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            memory: 512Mi
```

### Why Each Setting Matters

| Setting | Purpose |
|---------|---------|
| `terminationGracePeriodSeconds: 60` | Total time allowed for preStop + shutdown |
| `preStop: sleep 15` | Gives endpoints 15s to propagate before SIGTERM |
| `maxUnavailable: 0` | Never kills old pod before new pod is ready |
| `readinessProbe.periodSeconds: 2` | Fast detection (pod removed from endpoints quickly) |
| `readinessProbe.failureThreshold: 1` | Immediate removal on first failure |

## PodDisruptionBudget

Prevent all pods from being drained simultaneously:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2           # Always keep at least 2 pods running
  selector:
    matchLabels:
      app: myapp
```

Or use `maxUnavailable`:

```yaml
spec:
  maxUnavailable: 1         # Drain at most 1 pod at a time
```

Without a PDB, a node drain can evict all pods of a deployment at once if they're all on the same node.

## HAProxy-Specific Mitigations

### HAProxy Pod Graceful Shutdown

When the HAProxy Ingress Controller pods themselves are on a draining node, configure their lifecycle too:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: haproxy-ingress
spec:
  replicas: 3
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: haproxy
        image: haproxy:2.8
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                # Stop accepting new connections
                kill -SIGUSR1 $(pidof haproxy)
                # Wait for existing connections to finish
                sleep 20
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          periodSeconds: 2
          failureThreshold: 2
```

**What happens during drain:**
1. Node drain initiated → HAProxy pod marked for termination
2. Readiness probe fails → Pod removed from NLB target group (no new traffic from LB)
3. preStop hook executes → HAProxy receives SIGUSR1, stops accepting new connections
4. Sleep period → In-flight requests complete
5. SIGTERM sent → HAProxy shuts down gracefully
6. Grace period expires → SIGKILL if still running (after 60s)

### Increase HAProxy Retry on Connection Failure

```yaml
# Ingress annotation
annotations:
  haproxy.org/check: "true"
  haproxy.org/check-interval: "2s"
  haproxy.org/retries: "3"             # Retry failed connections
  haproxy.org/option-redispatch: "true" # Send to different server on retry
```

### Connection Draining Timeouts

```yaml
annotations:
  haproxy.org/timeout-server: "60s"
  haproxy.org/timeout-server-fin: "30s"  # Wait for server to close gracefully
  haproxy.org/timeout-client-fin: "30s"  # Wait for client to close gracefully
```

### Backend Health Checks

Enable active health checks so HAProxy detects dying pods faster:

```yaml
annotations:
  haproxy.org/check: "true"
  haproxy.org/check-http: "/health"
  haproxy.org/check-interval: "2s"
  haproxy.org/check-fall: "2"           # Mark down after 2 failed checks
```

With a 2-second check interval and 2 failures required, HAProxy marks the backend as down within 4 seconds — even before endpoint propagation.

## HAProxy Controller Configuration

### Faster Backend Updates

In the controller's ConfigMap or Helm values:

```yaml
controller:
  config:
    backend-server-slots-increment: "4"
    # Reduces config reload frequency by pre-allocating server slots
    
    # Use dynamic server management (avoids full reload)
    config-backend: |
      option httpchk GET /health
      http-check expect status 200
```

### Enable Dynamic Configuration (No Reload Needed)

HAProxy supports updating backends via the Runtime API without a full reload:

```yaml
controller:
  config:
    dynamic-scaling: "true"     # Update backends via socket, not reload
```

This significantly reduces the time between endpoint change and HAProxy routing update.

## Monitoring 5xx During Drains

### Prometheus Queries

```promql
# 5xx rate during drain windows
rate(haproxy_backend_http_responses_total{code=~"5.."}[1m])

# Compare with pod termination events
rate(kube_pod_container_status_terminated_reason_total{reason="OOMKilled"}[5m])

# Track connection errors
rate(haproxy_backend_connection_errors_total[1m])
```

### Datadog

```
# 5xx error rate
sum:haproxy.backend.response.5xx{*}.as_rate()

# During node drain windows — correlate with:
sum:kubernetes.pods.running{*} by {node}
```

### CloudWatch (via Container Insights)

```bash
# Check NLB 5xx metrics during drain
aws cloudwatch get-metric-statistics \
  --namespace AWS/NetworkELB \
  --metric-name TCP_Target_Reset_Count \
  --dimensions Name=LoadBalancer,Value=<nlb-arn-suffix> \
  --start-time <drain-start> \
  --end-time <drain-end> \
  --period 60 \
  --statistics Sum
```

## Testing the Fix

### Simulate a Drain

```bash
# 1. Start a load test
kubectl run load --image=busybox:1.36 --restart=Never --rm -it -- sh -c \
  'while true; do wget -qO- --timeout=5 http://myapp.default.svc.cluster.local/health || echo "FAIL"; sleep 0.1; done'

# 2. In another terminal, drain a node
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 3. Watch for failures in the load test output

# 4. Check HAProxy stats for 5xx
kubectl exec -n haproxy-ingress deploy/haproxy-ingress-kubernetes-ingress -- \
  sh -c 'echo "show stat" | socat /var/run/haproxy.sock -' | grep myapp | cut -d, -f1,2,14,40
```

### Validate preStop Works

```bash
# Watch pod termination timing
kubectl get pods -w -l app=myapp

# Check events during drain
kubectl get events --sort-by='.lastTimestamp' | grep -E "Killing|Evict"
```

## Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| No preStop hook | Traffic sent to dying pod during 2-5s gap | Add `sleep 15` preStop |
| preStop too short (< 5s) | Endpoint propagation not complete | Use 10-15s minimum |
| preStop longer than `terminationGracePeriodSeconds` | Pod is killed (SIGKILL) before shutdown completes | Ensure: preStop + shutdown < terminationGracePeriod |
| No PDB | All replicas drained at once | Add PDB with minAvailable |
| `maxUnavailable: 1` in deployment | One less pod during update (acceptable) but combined with drain = 0 pods | Set `maxUnavailable: 0` for critical services |
| readinessProbe too slow | Pod stays in endpoints 10-30s after SIGTERM | Use `periodSeconds: 2, failureThreshold: 1` |
| Application doesn't handle SIGTERM | Connections dropped immediately | Implement graceful shutdown |

## Checklist

```
□ preStop hook with sleep 10-15s on all application containers
□ terminationGracePeriodSeconds > preStop + expected shutdown time
□ Application handles SIGTERM (graceful connection draining)
□ Readiness probe with fast detection (periodSeconds: 2-5)
□ PodDisruptionBudget configured (minAvailable or maxUnavailable)
□ maxUnavailable: 0 in Deployment strategy for critical services
□ HAProxy health checks enabled (check-interval: 2s)
□ HAProxy retries + redispatch enabled
□ Tested with simulated drain under load
□ Monitoring for 5xx spikes during maintenance windows
```

## Summary

| Component | Setting | Purpose |
|-----------|---------|---------|
| Pod | `preStop: sleep 15` | Delay SIGTERM until HAProxy stops routing |
| Pod | `terminationGracePeriodSeconds: 60` | Budget for preStop + shutdown |
| Deployment | `maxUnavailable: 0` | Never reduce capacity |
| Readiness | `periodSeconds: 2, failureThreshold: 1` | Fast endpoint removal |
| PDB | `minAvailable: 2` | Prevent all pods draining at once |
| HAProxy | `check-interval: 2s, check-fall: 2` | Detect dying backends in 4s |
| HAProxy | `retries: 3, option-redispatch` | Retry on different backend |
| App | Graceful SIGTERM handler | Finish in-flight requests |
