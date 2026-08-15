# kubectl logs Guide

## Basic Commands

### Get Current Logs

```sh
# Basic log retrieval
kubectl logs <pod-name>

# Logs from specific container in multi-container pod
kubectl logs <pod-name> -c <container-name>

# Logs from specific namespace
kubectl logs <pod-name> -n <namespace>
```

### Follow Live Logs

```sh
# Follow logs in real-time
kubectl logs <pod-name> -f

# Follow with timestamps
kubectl logs <pod-name> -f --timestamps
```

### Limit Log Output

```sh
# Get last 100 lines
kubectl logs <pod-name> --tail=100

# Get logs from the last hour
kubectl logs <pod-name> --since=1h

# Get logs from last 30 minutes
kubectl logs <pod-name> --since=30m

# Get logs since a specific timestamp
kubectl logs <pod-name> --since-time=2024-01-15T10:00:00Z
```

## Multiple Pods and Containers

```sh
# Get logs from all pods with a specific label
kubectl logs -l app=myapp

# All containers in all matching pods
kubectl logs -l app=myapp --all-containers=true

# Follow logs from all pods with a label
kubectl logs -l app=myapp -f --all-containers

# Previous instance (after restart)
kubectl logs <pod-name> --previous

# Previous logs from all matching pods
kubectl logs -l app=myapp --previous

# Logs from a deployment's pods
kubectl logs deploy/my-deployment

# Logs from a specific container in a deployment
kubectl logs deploy/my-deployment -c my-container
```

## Output and Formatting

```sh
# Include timestamps
kubectl logs <pod-name> --timestamps

# Output to file
kubectl logs <pod-name> > pod-logs.txt

# Combine tail + follow + timestamps
kubectl logs <pod-name> --tail=100 --timestamps -f

# Save both current and previous logs
kubectl logs <pod-name> > current.log
kubectl logs <pod-name> --previous > previous.log 2>/dev/null
```

## Troubleshooting Scenarios

### Pod Won't Start

```sh
# Check pod events first
kubectl describe pod <pod-name>

# Then check logs (may be empty if container never started)
kubectl logs <pod-name>

# Check init container logs
kubectl logs <pod-name> -c <init-container-name>
```

### Application Crashes (CrashLoopBackOff)

```sh
# Check restart count
kubectl describe pod <pod-name> | grep "Restart Count"

# Get logs from the current (crashing) container
kubectl logs <pod-name>

# Get logs from the previous instance (before crash)
kubectl logs <pod-name> --previous
```

### Multi-Container Debugging

```sh
# List all containers in a pod
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].name}'

# Get logs from each container
kubectl logs <pod-name> -c container1
kubectl logs <pod-name> -c container2

# All containers at once
kubectl logs <pod-name> --all-containers=true
```

### Correlate with Events and Metrics

```sh
# Logs with timestamps for metric correlation
kubectl logs <pod-name> --timestamps --since=1h

# Pod events around the same time
kubectl get events --field-selector involvedObject.name=<pod-name> --sort-by='.lastTimestamp'

# Resource usage
kubectl top pod <pod-name>
```

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `container "x" not found` | Wrong container name | List containers: `kubectl describe pod <pod>` |
| No output (empty logs) | Container didn't start or writes to file | Check `kubectl describe pod`, check if app logs to file instead of stdout |
| `pods "x" is forbidden` | RBAC permission issue | Check role bindings, use correct namespace |
| `previous terminated container not found` | No previous instance exists | Pod hasn't restarted yet |

## Debug Bundle Script

Collect all relevant debugging information in one go:

```sh
#!/bin/bash
POD_NAME=$1
NAMESPACE=${2:-default}

OUTPUT_DIR="debug-$POD_NAME-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$OUTPUT_DIR"

echo "Collecting debug info for $POD_NAME in $NAMESPACE..."

# Pod description and spec
kubectl describe pod "$POD_NAME" -n "$NAMESPACE" > "$OUTPUT_DIR/describe.txt"
kubectl get pod "$POD_NAME" -n "$NAMESPACE" -o yaml > "$OUTPUT_DIR/pod-spec.yaml"

# Events
kubectl get events -n "$NAMESPACE" --field-selector involvedObject.name="$POD_NAME" \
  --sort-by='.lastTimestamp' > "$OUTPUT_DIR/events.txt"

# Logs from all containers
CONTAINERS=$(kubectl get pod "$POD_NAME" -n "$NAMESPACE" -o jsonpath='{.spec.containers[*].name}')
for container in $CONTAINERS; do
  echo "  Collecting: $container"
  kubectl logs "$POD_NAME" -c "$container" -n "$NAMESPACE" > "$OUTPUT_DIR/${container}-current.log"
  kubectl logs "$POD_NAME" -c "$container" -n "$NAMESPACE" --previous > "$OUTPUT_DIR/${container}-previous.log" 2>/dev/null
done

# Init container logs
INIT_CONTAINERS=$(kubectl get pod "$POD_NAME" -n "$NAMESPACE" -o jsonpath='{.spec.initContainers[*].name}' 2>/dev/null)
for container in $INIT_CONTAINERS; do
  echo "  Collecting init: $container"
  kubectl logs "$POD_NAME" -c "$container" -n "$NAMESPACE" > "$OUTPUT_DIR/init-${container}.log"
done

echo "Done → $OUTPUT_DIR/"
ls -la "$OUTPUT_DIR/"
```

Usage:

```sh
chmod +x debug-pod.sh
./debug-pod.sh my-pod production
```

## Best Practices

- **Check status first**: Always run `kubectl describe pod` before looking at logs — events often tell you more than logs
- **Use `--tail`**: Don't dump entire log history; use `--tail=200 --since=1h` to focus on recent output
- **Use `--previous`**: When a pod is in CrashLoopBackOff, `--previous` shows what happened before the crash
- **Timestamps matter**: Use `--timestamps` when correlating logs with metrics or events
- **Save logs early**: If you're about to delete a pod or it's crashing, save logs to a file immediately
- **Use labels over pod names**: `kubectl logs -l app=web` survives pod restarts and is deployment-friendly
- **Log aggregation for production**: Don't rely on `kubectl logs` in production — use Fluentd/Fluent Bit → CloudWatch, Loki, or ELK

## Quick Reference

```sh
# Current logs
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl logs <pod> -n <namespace>

# Follow
kubectl logs <pod> -f
kubectl logs <pod> -f --timestamps

# History
kubectl logs <pod> --previous
kubectl logs <pod> --tail=100
kubectl logs <pod> --since=1h

# Multiple pods
kubectl logs -l app=myapp
kubectl logs -l app=myapp --all-containers
kubectl logs deploy/my-deployment

# Save
kubectl logs <pod> > logs.txt
kubectl logs <pod> --previous > previous.txt
```
