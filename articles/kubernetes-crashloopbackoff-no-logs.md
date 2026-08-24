# Troubleshooting CrashLoopBackOff with No Logs

When a pod is in `CrashLoopBackOff` and `kubectl logs` (even with `--previous`) shows nothing, the container is crashing before it can write any output.

## Check Previous Container Logs

```bash
# Current container logs (may be empty if it crashed instantly)
kubectl logs <pod-name> -n <namespace>

# Previous container's logs (from the last crash)
kubectl logs <pod-name> -n <namespace> --previous
```

> If `--previous` also shows nothing, the process crashed before writing to stdout/stderr. Move to the steps below.

## Understanding CrashLoopBackOff Timing

CrashLoopBackOff isn't a state — it's a back-off delay the kubelet applies between restart attempts:

```
Crash → 10s wait → restart → crash → 20s wait → restart → crash → 40s → 80s → 160s → 300s (max)
```

The delay doubles each time, capping at 5 minutes. To reset the back-off timer, delete the pod (it gets recreated by the controller with a fresh counter).

```bash
# Reset back-off by deleting the pod
kubectl delete pod <pod-name> -n <namespace>
```

## 1. Describe the Pod

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look at:
- **State / Last State** — check the exit code and reason (OOMKilled, Error, etc.)
- **Events** — scheduling issues, image pull errors, mount failures, probe failures
- **Exit Code** — `137` = OOMKilled, `126` = permission denied, `127` = command not found

## 2. Check Events Cluster-Wide

```bash
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | grep <pod-name>
```

## 3. Override the Entrypoint to Keep the Container Alive

If the container crashes immediately, override the command so you can exec in:

```yaml
command: ["sleep", "infinity"]
# or
command: ["/bin/sh", "-c", "while true; do sleep 3600; done"]
```

Then inspect manually:

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

## 4. Check if OOMKilled

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
```

If it returns `OOMKilled`, increase memory limits.

## 5. Check Init Containers

```bash
kubectl logs <pod-name> -c <init-container-name>
```

Init containers may succeed but leave broken state for the main container.

## 6. Inspect the Image Directly

```bash
docker run --rm -it --entrypoint /bin/sh <image>
```

Run the actual entrypoint command manually to see errors not captured by the log driver.

## 7. Check Volume Mounts and Secrets

Missing ConfigMaps, Secrets, or PVCs can cause instant crashes with no logs. `kubectl describe` will show these as mount errors in the events section.

## 8. Ephemeral Debug Container (K8s 1.23+)

```bash
kubectl debug -it <pod-name> --image=busybox --target=<container-name>
```

Attaches a debug container sharing the same process namespace so you can inspect the crashing container's filesystem.

## 9. Liveness Probe Killing the Container

A misconfigured liveness probe can kill a healthy container before it finishes starting — no OOM, no crash, but the pod keeps restarting.

```bash
# Check if the container was killed by a probe
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Last State"
```

If you see `Reason: Completed` or `State: Waiting` with `Reason: CrashLoopBackOff` but the exit code is `0` or `143` (SIGTERM), the liveness probe is likely the cause.

Common fixes:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30    # Give the app time to start
  periodSeconds: 10
  failureThreshold: 5        # Allow more failures before killing
  timeoutSeconds: 5          # Increase if the endpoint is slow
```

> **Tip:** If the app takes a long time to start, use a `startupProbe` instead of increasing `initialDelaySeconds`. The startup probe disables liveness checks until the app is ready.

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30       # 30 × 10s = 5 minutes to start
  periodSeconds: 10
```

## Quick Decision Tree

| Exit Code | Meaning | Fix |
|-----------|---------|-----|
| 137 | OOMKilled | Raise memory limits |
| 127 | Command not found | Check image entrypoint |
| 126 | Permission denied | Fix binary permissions |
| 1 (no logs) | Config/env issue | Override entrypoint and inspect |
| None visible | Mount/scheduling failure | Check events |
