# kubectl logs --previous: Debugging Restarted Containers

The `--previous` flag (`-p` for short) on `kubectl logs` retrieves logs from the *previous* instance of a container — the one that ran before the current restart. When a pod is crash-looping, the live container's logs are often empty or only show a fresh startup, while the crash evidence lives in the terminated instance. `--previous` is how you get to it.

> `--previous` only works if the previous container instance still exists on the node. Logs from terminated containers are subject to kubelet garbage collection and are lost when the pod is deleted or rescheduled to another node. Capture them early.

## Basic Syntax

```sh
kubectl logs <pod-name> --previous
kubectl logs <pod-name> -p                 # short form
kubectl logs <pod-name> -c <container> -p  # specific container
kubectl logs <pod-name> -n <namespace> -p  # specific namespace
```

## When to Reach for --previous

### CrashLoopBackOff

The live container may not have logged anything yet — the crash happened in the instance before it.

```sh
kubectl get pods
# NAME         READY   STATUS             RESTARTS   AGE
# my-app-pod   0/1     CrashLoopBackOff   5          10m

kubectl logs my-app-pod              # current instance — often minimal
kubectl logs my-app-pod --previous   # the instance that actually crashed
```

### OOMKilled Containers

When a container is killed for exceeding its memory limit, the previous instance's logs show what it was doing right before the kill.

```sh
kubectl describe pod my-app-pod | grep -A5 "Last State"
# Last State:     Terminated
#   Reason:       OOMKilled
#   Exit Code:    137

kubectl logs my-app-pod --previous
```

Exit code `137` (`128 + 9`, SIGKILL) is the classic OOM signature.

### Startup / Initialization Failures

```sh
kubectl logs my-app-pod --previous | grep -iE 'error|exception|fatal|panic'
```

## Command Variations

### Multi-Container Pods

You must name the container with `-c`, or kubectl defaults to the first container.

```sh
# List the containers first
kubectl get pod web-app-pod -o jsonpath='{.spec.containers[*].name}'

kubectl logs web-app-pod -c app --previous
kubectl logs web-app-pod -c nginx --previous
kubectl logs web-app-pod -c logging-sidecar --previous
```

### Combined with Other Flags

```sh
kubectl logs <pod> --previous --timestamps    # add timestamps
kubectl logs <pod> --previous --tail=50       # last 50 lines only
kubectl logs <pod> --previous --since=1h      # time-bounded (if retained)
```

### Init Containers

Init containers must finish before app containers start, so a failing init container blocks the whole pod. Its previous logs are found the same way.

```sh
kubectl logs my-app-pod -c init-db --previous
kubectl describe pod my-app-pod | grep -A20 "Init Containers:"
```

### Multiple Pods by Label

```sh
kubectl logs -l app=myapp --previous
kubectl logs -l app=myapp --previous --all-containers=true
```

## A Practical Debugging Workflow

```sh
# 1. Status and restart count
kubectl get pod <pod>
kubectl describe pod <pod> | grep -i "restart count"

# 2. Termination reason of the previous instance
kubectl describe pod <pod> | grep -A10 "Last State:"

# 3. Compare current vs previous
kubectl logs <pod> > current.log
kubectl logs <pod> --previous > previous.log 2>/dev/null
diff previous.log current.log
```

### Snapshot Script

Grabs status, restart count, both log sets, and the termination reason in one shot:

```sh
#!/bin/bash
POD=$1
NS=${2:-default}

echo "=== Pod Status ==="
kubectl get pod "$POD" -n "$NS"

echo -e "\n=== Restart Count ==="
kubectl describe pod "$POD" -n "$NS" | grep -i "restart count"

echo -e "\n=== Current Logs (last 20) ==="
kubectl logs "$POD" -n "$NS" --tail=20

echo -e "\n=== Previous Logs (last 20) ==="
kubectl logs "$POD" -n "$NS" --previous --tail=20 2>/dev/null \
  || echo "No previous logs available"

echo -e "\n=== Last Termination Reason ==="
kubectl describe pod "$POD" -n "$NS" | grep -A10 "Last State:" | head -10
```

## Common Crash Patterns

### Configuration Errors

```sh
kubectl logs my-app --previous | grep -iE 'config|invalid|parse'
# ERROR: Failed to parse config file /app/config.yaml
# FATAL: Invalid database connection string
```

### Resource Limits

```sh
kubectl logs my-app --previous | grep -iE 'memory|oom|killed'
kubectl describe pod my-app | grep -A10 "Limits:"
```

### Dependency Failures

```sh
kubectl logs my-app --previous | grep -iE 'connection|timeout|refused'
# Connection refused to database:5432
# Timeout connecting to redis:6379
# DNS resolution failed for api-service
```

## Errors You'll Hit

| Error | Cause | Fix |
|-------|-------|-----|
| `previous terminated container "x" ... not found` | Pod hasn't restarted, or the previous container was garbage collected / rescheduled | Confirm `RESTARTS > 0`; nothing to recover if the instance is gone |
| `container "x" in pod "y" not found` | Wrong container name in `-c` | List names: `kubectl get pod <pod> -o jsonpath='{.spec.containers[*].name}'` |
| `pods "x" is forbidden` | RBAC or wrong namespace | Check role bindings; pass the correct `-n <namespace>` |

## Best Practices

- **Save before it's lost**: terminated-container logs can be garbage collected — snapshot them immediately.
  ```sh
  kubectl logs my-app --previous > crash-$(date +%Y%m%d-%H%M%S).log 2>/dev/null
  ```
- **Always check both**: current logs show recovery attempts; `--previous` shows the failure.
- **Pair with `describe`**: the `Last State` block and `Events` give the reason and exit code the logs won't.
  ```sh
  kubectl describe pod my-app | grep -A20 "Events:"
  ```
- **Correlate with events**: build a timeline of what happened around the restart.
  ```sh
  kubectl get events --field-selector involvedObject.name=my-app --sort-by='.lastTimestamp'
  ```
- **Watch restart trends**: a rising count means the loop is ongoing.
  ```sh
  kubectl get pod my-app -o jsonpath='{.status.containerStatuses[0].restartCount}'
  ```
- **Don't depend on it in production**: `--previous` is best-effort recovery. Ship logs to Loki, ELK, or CloudWatch so crash logs survive pod deletion.

## Troubleshooting Checklist

- [ ] Confirm the pod has actually restarted (`RESTARTS > 0`)
- [ ] Use the correct container name in multi-container pods
- [ ] Check `Last State` / exit code in `kubectl describe pod`
- [ ] Compare current vs previous logs for patterns
- [ ] Save previous logs before they're garbage collected
- [ ] Correlate with pod events and resource limits
- [ ] Check init container logs when the pod never reaches Running

## Quick Reference

```sh
kubectl logs <pod> --previous                 # previous instance
kubectl logs <pod> -p                         # short form
kubectl logs <pod> -c <container> --previous  # specific container
kubectl logs <pod> --previous --timestamps    # with timestamps
kubectl logs <pod> --previous --tail=50       # last 50 lines
kubectl logs -l app=myapp --previous          # by label

kubectl describe pod <pod> | grep -A10 "Last State:"   # termination reason
kubectl logs <pod> --previous > crash.log 2>/dev/null  # save it
```
