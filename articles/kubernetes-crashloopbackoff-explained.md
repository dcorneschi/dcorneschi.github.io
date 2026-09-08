# CrashLoopBackOff Explained

## What Is CrashLoopBackOff?

CrashLoopBackOff is a **temporary status** indicating that Kubernetes is waiting
before restarting a container that has failed multiple times.

It's not a permanent error state — it's Kubernetes saying "this container keeps
crashing, so I'll wait a bit before trying again."

## Why It's Temporary

When a container crashes repeatedly, the kubelet uses **exponential backoff** to
avoid hammering the system with constant restart attempts.

### Backoff timing

```text
Restart 1: immediate
Restart 2: wait 10s
Restart 3: wait 20s
Restart 4: wait 40s
Restart 5: wait 80s
Restart 6: wait 160s (2m40s)
Restart 7+: wait 300s (5m) — MAX
```

Maximum backoff is 5 minutes. After the delay expires the kubelet restarts the
container again; if it crashes again, the delay grows up to that 5-minute cap.

## Status Progression

```text
Running → Crashed → CrashLoopBackOff → Running → Crashed → CrashLoopBackOff → ...
          ^                            ^
          container exits              waiting during backoff
```

### What you'll see

```bash
$ kubectl get pods
NAME                 READY   STATUS             RESTARTS   AGE
myapp-7d8f9c5b-xyz   0/1     CrashLoopBackOff   5          10m

# after the backoff expires...
$ kubectl get pods
NAME                 READY   STATUS    RESTARTS   AGE
myapp-7d8f9c5b-xyz   0/1     Running   6          11m

# then it crashes again...
$ kubectl get pods
NAME                 READY   STATUS             RESTARTS   AGE
myapp-7d8f9c5b-xyz   0/1     CrashLoopBackOff   6          11m
```

## Common Causes

### 1. Application crashes on startup

Container starts, immediately exits, restart count climbs fast, logs show init
errors:

```text
Error: DATABASE_URL is not set
Error: Cannot read config file /etc/app/config.yaml
Error: Failed to connect to database at postgres:5432
```

Diagnose:

```bash
kubectl logs <pod-name> -n <namespace> --previous   # logs from the crashed instance
kubectl logs <pod-name> -n <namespace>              # current logs
kubectl describe pod <pod-name> -n <namespace>      # events
```

### 2. Liveness probe failures

Container runs a while, then gets killed at regular intervals (probe period ×
failure threshold):

```text
Warning  Unhealthy  kubelet  Liveness probe failed: HTTP probe failed with statuscode: 500
Normal   Killing    kubelet  Container myapp failed liveness probe, will be restarted
```

### 3. OOMKilled (out of memory)

Memory grows, then the container is killed; restarts may space out as memory
rebuilds.

```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].resources.limits.memory}'
kubectl top pod <pod-name> -n <namespace> --containers
```

Fix by raising limits (and finding the leak):

```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

### 4. Missing dependencies

App can't reach a required service (DB, cache, external API, or an unmounted
ConfigMap/Secret). Gate startup with an init container:

```yaml
initContainers:
- name: wait-for-db
  image: busybox:1.28
  command: ['sh', '-c', 'until nc -z postgres-service 5432; do echo waiting for db; sleep 2; done']
```

### 5. Incorrect command or entrypoint

Container exits immediately with code 126/127 ("permission denied" / "command
not found"):

```yaml
command: ["/app/start.sh"]   # file doesn't exist → 127
command: ["/scripts/start.sh"]  # not executable → 126
```

```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
```

### 6. Configuration errors

Bad or missing config values or files:

```bash
kubectl exec <pod-name> -n <namespace> -- env                 # env vars present?
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Mounts:"
kubectl get configmap <configmap-name> -n <namespace>
```

### 7. Volume mount issues

The app can't read/write a mounted volume — "permission denied" or "no such file
or directory" for a path that should be mounted. Common with an unbound PVC, a
wrong `mountPath`, or a fsGroup/ownership mismatch.

```bash
# Is the PVC bound?
kubectl get pvc -n <namespace>

# What does the pod actually mount, and did volume attach fail in events?
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 -E "Mounts:|Volumes:|Events:"
```

Fix: ensure the PVC is `Bound`, the `mountPath` matches what the app expects, and
the container user can access it (e.g. set `securityContext.fsGroup`).

## Diagnostic Workflow

```text
1. kubectl get pod <pod> -n <ns>                 # status + restart count
2. kubectl logs <pod> -n <ns> --previous         # crashed instance's logs
3. kubectl logs <pod> -n <ns>                    # current logs (if running)
4. kubectl describe pod <pod> -n <ns>            # events (probe failed, OOMKilled...)
5. ... -o jsonpath='{...lastState.terminated.exitCode}'   # exit code
6. ... -o jsonpath='{...lastState.terminated.reason}'     # termination reason
7. fix the root cause
```

## Quick Diagnostic Commands

```bash
kubectl get pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> -f
kubectl describe pod <pod-name> -n <namespace>

kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[0].lastState.terminated}' | jq

kubectl top pod <pod-name> -n <namespace> --containers
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name> --sort-by='.lastTimestamp'
```

## Interactive Debugging

Logs and events explain most crashes, but sometimes you need to get *inside* a
crashing container to inspect its filesystem, environment, or processes. A
crashing container is hard to `exec` into because it's rarely running — these
methods work around that.

### Method 1: Ephemeral debug container (`kubectl debug`, K8s 1.18+)

Attach a debug container that shares namespaces with the target, so you can
inspect the target container's process, filesystem, and environment via
`/proc/1/`:

```bash
# Basic ephemeral container targeting the crashing container
kubectl debug <pod-name> -it --image=busybox:latest --target=<container-name>

# With more privileges for full filesystem/env access
kubectl debug <pod-name> -it --image=nicolaka/netshoot --target=<container-name> --profile=general

# Maximum access if the general profile isn't enough
kubectl debug <pod-name> -it --image=nicolaka/netshoot --target=<container-name> --profile=sysadmin
```

Inside the debug container, inspect the target (PID 1) through `/proc`:

```bash
ps aux                                    # processes (needs shared PID namespace)
ls /proc/1/root/                          # the target container's filesystem
cat /proc/1/environ | tr '\0' '\n'        # its environment variables
cat /proc/1/root/etc/hosts                # its network config
cat /proc/1/root/var/log/*                # its application logs
```

If `/proc/1/root/` gives "permission denied", check whether the pod shares its
process namespace:

```bash
kubectl get pod <pod-name> -o jsonpath='{.spec.shareProcessNamespace}'
```

If that's empty/`false`, use the copy-pod method below, or enable it on the
controller (triggers a rollout):

```bash
kubectl patch deployment <deployment-name> --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/shareProcessNamespace","value":true}]'
```

### Method 2: Copy the pod (best for permission/PSS issues)

Create a copy of the pod with a shell as the entrypoint. Because it's a fresh
pod you control, there are no preemption/permission surprises, and it respects
Pod Security Standards:

```bash
# Copy with a shell in the target container
kubectl debug <pod-name> -it --copy-to=debug-pod --container=<container-name> -- sh

# Copy with process-namespace sharing
kubectl debug <pod-name> -it --copy-to=debug-pod --container=<container-name> --share-processes -- sh

# Copy but swap in a debug image
kubectl debug <pod-name> -it --copy-to=debug-pod --container=<container-name> --image=busybox -- sh

# Clean up when done
kubectl delete pod debug-pod
```

### Method 3: Override the entrypoint to keep it alive

Temporarily replace the command so the container idles instead of crashing, then
`exec` in and investigate:

```bash
kubectl patch deployment <deployment-name> --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/command","value":["/bin/sh","-c","sleep infinity"]}]'

kubectl exec -it <new-pod-name> -- /bin/sh
```

Revert the command once you're done debugging.

### Method 4: Exec during the brief running window

If the container survives a few seconds before crashing, keep retrying until you
land a shell:

```bash
while ! kubectl exec -it <pod-name> -- /bin/sh; do sleep 1; done
```

### Inspect Ports and Probes from Inside

Once you're in (via any method above), verify the app is actually listening and
that its health endpoint responds — a common liveness/readiness failure cause:

```bash
# Listening ports
netstat -tuln
ss -tuln

# Hit the health endpoint the probe uses
curl -s http://localhost:80/
wget -qO- http://localhost:80/
nc -zv localhost 80

# Probe/readiness status from outside the container
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].ready}'
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].restartCount}'
kubectl get pod <pod-name> -o json | jq '.status.containerStatuses[] | {name, ready, restartCount, state}'
```

## Exit Codes Reference

| Exit code | Meaning | Common cause |
|-----------|---------|--------------|
| 0 | Success | Completed cleanly (shouldn't loop) |
| 1 | General error | App error, uncaught exception |
| 2 | Shell misuse | Invalid command syntax |
| 126 | Cannot execute | Permission denied, not executable |
| 127 | Command not found | Binary/script missing |
| 130 | SIGINT | Terminated by Ctrl+C |
| 137 | SIGKILL | OOMKilled or force-killed |
| 143 | SIGTERM | Graceful shutdown |
| 255 | Out of range | Invalid exit code |

> Codes above 128 follow the `128 + N` convention, where `N` is the fatal signal
> number: `137 = 128 + 9` (SIGKILL, e.g. OOMKilled), `143 = 128 + 15` (SIGTERM),
> `139 = 128 + 11` (SIGSEGV).

## How to Fix It

1. **Identify the root cause** with the diagnostics above.
2. **Apply the fix:**
   - App crashes → fix code/config, add missing env vars, ensure dependencies.
   - Liveness failures → raise `initialDelaySeconds`, add a startup probe, fix the
     health endpoint.
   - OOMKilled → raise memory limits, fix leaks.
   - Missing dependencies → init containers, verify services and connectivity.
3. **Redeploy:**

```bash
kubectl apply -f deployment.yaml
kubectl rollout restart deployment <deployment-name> -n <namespace>
```

## Temporary Workarounds

You can't set the backoff time directly, but you can reset the restart counter:

```bash
# Delete the pod (a controller recreates it)
kubectl delete pod <pod-name> -n <namespace>

# Or scale a deployment down and back up
kubectl scale deployment <deployment-name> -n <namespace> --replicas=0
kubectl scale deployment <deployment-name> -n <namespace> --replicas=1
```

These are stopgaps — always fix the root cause.

## Prevention

1. Test the container locally first: `docker run --rm <image>`.
2. Use proper probes — startup (slow starts), liveness (deadlocks), readiness (traffic).
3. Set sensible resource requests/limits.
4. Use init containers to wait on dependencies.
5. Handle errors gracefully in the app (retries with backoff, clear logging).
6. Validate ConfigMaps/Secrets exist before deploying.

## How the Backoff Resets

The restart backoff resets after a container runs successfully for about
**10 minutes**. That's why a pod that eventually stabilizes can show a high
restart count yet no longer be in CrashLoopBackOff — after the next crash it
starts again from the 10-second delay.

## Related

- [Troubleshooting CrashLoopBackOff with No Logs](articles/kubernetes-crashloopbackoff-no-logs.md)
- [CrashLoopBackOff Internals](articles/kubernetes-crashloopbackoff-internals.md)
- [Kubernetes Pod Conditions Flow](articles/kubernetes-pod-conditions-flow.md)

## Skills Practiced

- Understanding CrashLoopBackOff as a backoff state, not a fatal error
- Reading exit codes and termination reasons to pinpoint the cause
- Running the standard diagnostic workflow (logs `--previous`, describe, jsonpath)
- Applying targeted fixes and knowing how the 10-minute backoff reset works
