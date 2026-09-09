# Kubernetes Health Checks: Liveness, Readiness, and Startup Probes

Kubernetes only knows three container states on its own: waiting, running, and terminated. "Running" just means the process (PID 1) exists — not that the application inside is actually serving requests. Probes close that gap. Without them, a pod is marked Ready the instant its container starts and stays Ready even if the app inside has hung. This article explains what happens without health checks, how each probe works, and how to configure them.

Related reading: [Pod Conditions Flow](articles/kubernetes-pod-conditions-flow.md) for the Ready condition lifecycle, [Why Pod Shows 0/1 Ready](articles/kubernetes-pod-0-1-ready-status.md) for readiness debugging, and [Time to Ready](articles/kubectl-pod-time-to-ready.md) for startup timing.

## What Happens Without Probes

When a container starts with no probes configured:

1. The kubelet pulls the image; the runtime creates and starts the container.
2. **Immediately** the pod phase becomes `Running` and the container state becomes `Running`.
3. **Immediately** the pod's `Ready` condition flips `True` (a container with no readiness probe is ready as soon as it is running).
4. The EndpointSlice controller adds the pod IP to the Service's endpoints.
5. kube-proxy programs iptables/IPVS rules and traffic starts flowing.

The problem: nobody verified the application is actually responding.

```
Container runtime checks: is the process alive (PID exists)?
Kubernetes checks:        is the container state "Running"?
Nobody checks:            is the app answering requests?
```

### The Startup Race

If the app needs a few seconds to initialize (parse config, load TLS certs, bind its port), there's a window where the pod is Ready but requests fail:

```
T+0.0s  Container starts, PID 1 spawned
T+0.1s  Pod marked Ready=True
T+0.2s  Pod IP added to Service endpoints
T+0.3s  kube-proxy updates iptables
T+0.5s  First request arrives -> app still loading -> 502/503 or connection refused
T+2.0s  App finally ready
```

A readiness probe eliminates this window by withholding `Ready` until the app answers.

### What Kubernetes Cannot Detect Without a Liveness Probe

A process can be alive but useless. Without liveness checks, none of these trigger a restart:

- Worker threads deadlocked or thread pool exhausted
- Port bound but not accepting connections
- App returning 500s for every request
- Memory or file descriptors exhausted while PID 1 lingers
- Upstream dependency down and the app wedged waiting on it

```sh
# Process exists...
$ ps aux | grep nginx
root  1234  nginx: master process

# ...but it's not serving
$ curl http://localhost
curl: (7) Connection refused

# Kubernetes still thinks all is well — and keeps sending traffic
$ kubectl get pod web
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   0          30m
```

> A crashed process (PID 1 exits) is restarted by the pod's `restartPolicy` on its own — you don't need a liveness probe for that. Liveness probes earn their keep only for the **alive-but-hung** case, where the process never exits.

### Does Kubernetes Monitor PID 1?

Yes — the kubelet watches the container's main process (PID 1). If PID 1 exits, the container terminates and is restarted per the pod's `restartPolicy`, with or without any probe. That's why a hard crash is handled for free.

What PID 1 monitoring can't see is a process that stays alive while the app inside is broken — a deadlock, a full thread pool, a port that's bound but not accepting. PID 1 is running, so Kubernetes considers the container healthy and keeps routing traffic. A liveness probe is the only thing that catches this.

| App state | With no probe | What a liveness probe adds |
|-----------|---------------|----------------------------|
| PID 1 exits (crash) | Container restarts automatically | Nothing extra — already handled |
| PID 1 alive but hung/unresponsive | Stays "Running", keeps traffic | Detects it and restarts the container |

Readiness gets no PID-1 signal at all. There is no automatic readiness equivalent: without a readiness probe a pod is considered ready the moment PID 1 is running, so it joins Service endpoints immediately — even mid-initialization. For production workloads a readiness probe is effectively mandatory.

## The Three Probes

| Probe | When it runs | Action on failure |
|-------|--------------|-------------------|
| `startupProbe` | During startup only, until it first succeeds | Kill and restart the container |
| `livenessProbe` | Continuously, after startup completes | Kill and restart the container |
| `readinessProbe` | Continuously, after startup completes | Remove pod from Service endpoints (no restart) |

The startup probe gates the other two: liveness and readiness don't begin until the startup probe passes. This lets you give a slow-starting app a long startup budget without making liveness lenient forever.

## Readiness Probes

Readiness controls **traffic**, not restarts. A failing readiness probe pulls the pod out of Service endpoints; the container keeps running and can rejoin once it recovers.

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  successThreshold: 1
  failureThreshold: 3
```

### What Happens on Readiness Failure

The removal propagates through several components in order:

1. kubelet sets `containerStatus.ready = false`.
2. kubelet updates the pod's `Ready` condition to `False`.
3. The API server persists the new status.
4. The EndpointSlice controller sees `Ready=False` and removes the pod from the Service's EndpointSlice.
5. kube-proxy observes the change and updates iptables/IPVS rules.
6. New connections stop routing to the pod.

> Because iptables/IPVS act on new connections, **existing** established connections may continue until they close. Removal from endpoints stops *new* traffic; it doesn't forcibly cut live connections. Budget for that when reasoning about drain timing.

## Liveness Probes

Liveness controls **restarts**. After `failureThreshold` consecutive failures, the kubelet kills the container and lets `restartPolicy` recreate it.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 80
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

Restart sequence when a hung app trips the liveness probe:

```
T+30s  Liveness probe fails (timeout / non-2xx)
T+40s  Fails again
T+50s  Third failure hits failureThreshold
T+50s  kubelet sends SIGTERM; pod marked Ready=False, removed from endpoints
T+50s+ After terminationGracePeriodSeconds, SIGKILL if still alive
       New container starts -> runs readiness probe -> rejoins endpoints when ready
```

## Startup Probes

For apps with long or variable startup, a startup probe is better than a large `initialDelaySeconds` on liveness. It disables liveness/readiness checks until the app has started, then hands off.

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 80
  periodSeconds: 5
  failureThreshold: 30    # up to 30 x 5s = 150s to start before giving up
```

## Probe Types

All three probes support the same handlers.

### HTTP (`httpGet`)

```yaml
httpGet:
  path: /healthz
  port: 80
  scheme: HTTP
  httpHeaders:
    - name: X-Probe
      value: kube
```

- The kubelet makes the request from the **node**, directly to the pod IP — it bypasses Services.
- Success is any status `200 <= code < 400`; anything else (or a timeout) is a failure.
- Requests carry a `User-Agent: kube-probe/<version>` header, handy for filtering probe traffic in access logs.

### TCP (`tcpSocket`)

```yaml
tcpSocket:
  port: 80
```

Succeeds if the kubelet can open a TCP connection. It sends no data, so it proves the port is accepting connections but not that the app is healthy behind it. Useful when there's no HTTP endpoint.

### Exec (`exec`)

```yaml
exec:
  command: ["cat", "/tmp/healthy"]
```

Runs a command inside the container; exit code `0` is success. Most flexible (custom scripts, file checks, local queries) but also the most expensive since it forks a process each period.

### gRPC (`grpc`)

```yaml
grpc:
  port: 9090
```

Uses the standard gRPC health checking protocol (Kubernetes 1.24+). The service must implement the gRPC health service.

## When Your App Has No Health Endpoint

Plenty of images (legacy apps, third-party binaries, an app baked into an AMI) expose no `/health` route. You still have options — roughly in order of least effort to most reliable:

1. **TCP probe** — if the app already listens on a port, `tcpSocket` is the quickest win. It confirms the port accepts connections, which is a decent readiness signal for many servers.
   ```yaml
   readinessProbe:
     tcpSocket:
       port: 8080
     initialDelaySeconds: 5
     periodSeconds: 10
   ```

2. **Exec probe** — check a process, file, or run a small script inside the container when there's no network endpoint to hit.
   ```yaml
   livenessProbe:
     exec:
       command: ["/bin/sh", "-c", "pgrep -x app_process >/dev/null"]
     periodSeconds: 10
   ```

3. **Startup probe** — wrap a slow or unpredictable boot so liveness doesn't fire prematurely, even with just a TCP check.
   ```yaml
   startupProbe:
     tcpSocket:
       port: 8080
     failureThreshold: 30
     periodSeconds: 10
   ```

4. **Add a real HTTP endpoint** — the best long-term fix. A dedicated `/health/ready` and `/health/live` in the app can check what actually matters (DB connection, cache warmed, dependencies reachable) rather than just "port is open."

A TCP or exec probe is a reasonable stopgap, but be honest about what it proves: "the port is open" or "the process exists" is not the same as "the app can serve requests." Move to an HTTP/gRPC endpoint for anything production-critical.

## Configuration: Getting the Timing Right

**Startup budget** — set `initialDelaySeconds` (or a startup probe) to cover typical startup plus a buffer:

```
initialDelaySeconds ≈ average startup time + margin
# app starts in ~5s -> initialDelaySeconds: 10
```

**Failure detection time** — how long a fault takes to register:

```
detection time = periodSeconds × failureThreshold
# 10s × 3 = 30s before a liveness restart triggers
```

**Total recovery time** for a liveness restart:

```
total ≈ detection time + terminationGracePeriodSeconds + startup time
# 30s + 30s + 10s ≈ 70s of impact
```

## Common Mistakes

**`initialDelaySeconds` too low** — probe fails before the app finishes starting, causing a restart loop. Use a startup probe for slow starts.

**`failureThreshold: 1` on liveness** — a single transient blip restarts the container. Keep liveness lenient.

**Same probe for liveness and readiness** — the biggest anti-pattern. A brief hiccup that should only remove the pod from rotation instead restarts it.

```yaml
# Anti-pattern: identical probes
livenessProbe: &probe
  httpGet: { path: /, port: 80 }
readinessProbe: *probe
```

Split them — lenient liveness (is it alive at all?), stricter readiness (is it fit to serve?):

```yaml
livenessProbe:
  httpGet: { path: /healthz, port: 80 }
  failureThreshold: 5      # tolerant — only restart on real hangs
  periodSeconds: 10
readinessProbe:
  httpGet: { path: /ready, port: 80 }
  failureThreshold: 2      # strict — pull from traffic quickly
  periodSeconds: 5
```

## Complete Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: web
          image: nginx:1.25
          ports:
            - containerPort: 80
          # Gate liveness/readiness until the app has started
          startupProbe:
            httpGet: { path: /, port: 80 }
            periodSeconds: 5
            failureThreshold: 30      # up to 150s to start
          # Restart only on genuine hangs
          livenessProbe:
            httpGet: { path: /, port: 80 }
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 5
          # Protect users: no traffic until ready, pulled fast on failure
          readinessProbe:
            httpGet: { path: /, port: 80 }
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 500m, memory: 512Mi }
```

## Monitoring and Debugging

```sh
# Probe config and recent events
kubectl describe pod <pod>
kubectl get events --field-selector involvedObject.name=<pod>

# Restart count (liveness failures) and readiness
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].restartCount}'
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].ready}'

# Inspect just the probe blocks
kubectl get pod <pod> -o json | jq '.spec.containers[].readinessProbe'
```

Typical failure events:

```
Warning  Unhealthy  kubelet  Liveness probe failed: HTTP probe failed with statuscode: 500
Warning  Unhealthy  kubelet  Readiness probe failed: Get "http://10.0.0.1:80/": context deadline exceeded
Warning  BackOff    kubelet  Back-off restarting failed container
```

Metrics worth alerting on (via kube-state-metrics):

- `kube_pod_container_status_restarts_total` — climbing indicates liveness restarts
- `kube_pod_status_ready` / `kube_pod_container_status_ready` — readiness at pod/container level

## Summary

- Without a **readiness** probe, pods receive traffic before they're ready and keep receiving it when they break.
- Without a **liveness** probe, hung-but-alive containers are never restarted (crashed ones still are, via `restartPolicy`).
- A **startup** probe gives slow starters room without weakening liveness.
- Keep liveness lenient and readiness strict, and never reuse one probe for both.
- Detection time is `periodSeconds × failureThreshold`; total recovery adds the grace period and startup time.
```
