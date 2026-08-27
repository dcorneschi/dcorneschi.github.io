# kubectl wait and Condition-Based Scripting

Using `kubectl wait` to block until resources reach a desired state — conditions, jsonpath expressions, timeouts, and patterns for CI/CD pipelines and automation scripts.

## How kubectl wait Works

`kubectl wait` uses the watch API to monitor a resource and exits when the specified condition is met or the timeout expires:

```
┌──────────┐     ┌───────────────┐
│  kubectl │────▶│   API Server  │
│  wait    │     │               │
│          │     │  WATCH the    │
│  (blocks │◀────│  resource     │
│   until  │     │  until        │
│   match) │     │  condition    │
└──────────┘     └───────────────┘
        │
        ▼
   Exit code 0 (condition met)
   Exit code 1 (timeout)
```

## Basic Syntax

```bash
kubectl wait <resource> --for=<condition> [--timeout=<duration>]
```

### Wait for Conditions

```bash
# Wait for pod to be Ready:
kubectl wait pod/my-app --for=condition=Ready --timeout=120s

# Wait for deployment to be Available:
kubectl wait deployment/my-app --for=condition=Available --timeout=300s

# Wait for node to be Ready:
kubectl wait node/node-1 --for=condition=Ready --timeout=60s

# Wait for job to Complete:
kubectl wait job/my-job --for=condition=Complete --timeout=600s

# Wait for PVC to be Bound:
kubectl wait pvc/my-volume --for=jsonpath='{.status.phase}'=Bound --timeout=60s
```

### Wait for Deletion

```bash
# Wait for pod to be fully deleted:
kubectl wait pod/my-app --for=delete --timeout=60s

# Wait for all pods with a label to be deleted:
kubectl wait pods -l app=old-version --for=delete --timeout=120s
```

### Wait with JSONPath

```bash
# Wait for specific field value:
kubectl wait pod/my-app --for=jsonpath='{.status.phase}'=Running --timeout=60s

# Wait for a deployment's replicas to match:
kubectl wait deployment/my-app --for=jsonpath='{.status.readyReplicas}'=3 --timeout=120s

# Wait for a custom resource field:
kubectl wait certificate/my-cert --for=jsonpath='{.status.conditions[0].status}'=True --timeout=300s

# Wait for node to be schedulable:
kubectl wait node/node-1 --for=jsonpath='{.spec.unschedulable}'=false --timeout=60s
```

## Condition Types by Resource

### Pod Conditions

| Condition | Meaning | When True |
|-----------|---------|-----------|
| `PodScheduled` | Pod assigned to a node | Scheduler bound the pod |
| `Initialized` | Init containers completed | All init containers exited 0 |
| `ContainersReady` | All containers passing readiness | All readiness probes pass |
| `Ready` | Pod is fully ready | ContainersReady + readiness gates |

```bash
kubectl wait pod/my-app --for=condition=Ready --timeout=120s
kubectl wait pod/my-app --for=condition=PodScheduled --timeout=30s
kubectl wait pod/my-app --for=condition=Initialized --timeout=60s
```

### Deployment Conditions

| Condition | Meaning |
|-----------|---------|
| `Available` | MinAvailable pods are ready for minReadySeconds |
| `Progressing` | Rollout is making progress (or completed) |

```bash
kubectl wait deployment/my-app --for=condition=Available --timeout=300s
```

### Job Conditions

| Condition | Meaning |
|-----------|---------|
| `Complete` | Job finished successfully |
| `Failed` | Job failed (backoffLimit reached) |

```bash
# Wait for success:
kubectl wait job/my-job --for=condition=Complete --timeout=600s

# Wait for failure (useful in tests):
kubectl wait job/my-job --for=condition=Failed --timeout=600s
```

### Node Conditions

| Condition | Meaning |
|-----------|---------|
| `Ready` | Kubelet is healthy and ready to accept pods |
| `MemoryPressure` | Node has memory pressure |
| `DiskPressure` | Node has disk pressure |
| `PIDPressure` | Node has PID pressure |

```bash
kubectl wait node/node-1 --for=condition=Ready --timeout=300s
```

## Selecting Multiple Resources

```bash
# Wait for ALL pods with a label:
kubectl wait pods -l app=my-app --for=condition=Ready --timeout=120s

# Wait for all pods in a namespace:
kubectl wait pods --all -n production --for=condition=Ready --timeout=120s

# Wait for all nodes:
kubectl wait nodes --all --for=condition=Ready --timeout=300s

# Wait for all jobs to complete:
kubectl wait jobs --all -n batch --for=condition=Complete --timeout=600s
```

When waiting on multiple resources, `kubectl wait` exits successfully only when ALL matched resources satisfy the condition.

## Timeout Behavior

```bash
# Default: no timeout (waits forever)
kubectl wait pod/my-app --for=condition=Ready

# With timeout:
kubectl wait pod/my-app --for=condition=Ready --timeout=120s
# Exit code 0: condition met within 120s
# Exit code 1: timeout expired

# Zero timeout = check once and exit:
kubectl wait pod/my-app --for=condition=Ready --timeout=0s
# Useful for "is it ready right now?" checks
```

### Timeout Format

```bash
--timeout=30s     # 30 seconds
--timeout=5m      # 5 minutes
--timeout=1h      # 1 hour
--timeout=0       # No wait (instant check)
```

## CI/CD Pipeline Patterns

### Wait for Deployment Rollout

```bash
#!/bin/bash
set -e

# Apply new deployment
kubectl apply -f deployment.yaml

# Wait for rollout to complete
kubectl rollout status deployment/my-app --timeout=300s

# OR use kubectl wait:
kubectl wait deployment/my-app --for=condition=Available --timeout=300s

# Verify all pods are ready
kubectl wait pods -l app=my-app --for=condition=Ready --timeout=120s
```

### Wait for Job Completion

```bash
#!/bin/bash
set -e

# Run a database migration job
kubectl apply -f migration-job.yaml

# Wait for it to finish
if kubectl wait job/db-migration --for=condition=Complete --timeout=600s; then
  echo "Migration succeeded"
else
  echo "Migration failed or timed out"
  kubectl logs job/db-migration
  exit 1
fi
```

### Wait for Dependencies Before Starting

```bash
#!/bin/bash
set -e

# Wait for database to be ready
kubectl wait pod -l app=postgres --for=condition=Ready --timeout=120s

# Wait for redis
kubectl wait pod -l app=redis --for=condition=Ready --timeout=60s

# Now deploy the app that depends on them
kubectl apply -f app-deployment.yaml
kubectl wait deployment/my-app --for=condition=Available --timeout=180s
```

### Wait After Node Operations

```bash
#!/bin/bash
# After adding a node or uncordoning:
kubectl uncordon node-5

# Wait for node to be ready:
kubectl wait node/node-5 --for=condition=Ready --timeout=300s

# Wait for DaemonSet pods to be scheduled:
sleep 10  # Give scheduler time
kubectl wait pods -l app=monitoring-agent --field-selector spec.nodeName=node-5 \
  --for=condition=Ready --timeout=120s
```

### Helm + Wait Pattern

```bash
# Helm has built-in wait:
helm install my-app ./chart --wait --timeout 5m

# Or manual post-helm wait:
helm install my-app ./chart
kubectl wait deployment/my-app --for=condition=Available --timeout=300s
kubectl wait pods -l app.kubernetes.io/instance=my-app --for=condition=Ready --timeout=120s
```

## Advanced Patterns

### Wait with Retry Logic

```bash
#!/bin/bash
# Wait for a resource that might not exist yet:
MAX_ATTEMPTS=30
ATTEMPT=0

until kubectl wait pod -l app=my-app --for=condition=Ready --timeout=10s 2>/dev/null; do
  ATTEMPT=$((ATTEMPT + 1))
  if [ $ATTEMPT -ge $MAX_ATTEMPTS ]; then
    echo "Timed out waiting for pods"
    exit 1
  fi
  echo "Waiting for pods to exist... (attempt $ATTEMPT/$MAX_ATTEMPTS)"
  sleep 5
done

echo "All pods ready"
```

### Wait for CRD Resources

```bash
# Wait for a cert-manager Certificate to be ready:
kubectl wait certificate/my-cert \
  --for=condition=Ready --timeout=120s

# Wait for an ArgoCD Application to be synced:
kubectl wait application/my-app -n argocd \
  --for=jsonpath='{.status.sync.status}'=Synced --timeout=300s

# Wait for Karpenter NodeClaim to be ready:
kubectl wait nodeclaim/my-claim \
  --for=condition=Ready --timeout=300s
```

### Combine with rollout status

```bash
# kubectl rollout status is similar but with more detail:
kubectl rollout status deployment/my-app --timeout=300s
# Waiting for deployment "my-app" rollout to finish: 1 of 3 updated replicas are available...
# deployment "my-app" successfully rolled out

# Difference:
# - rollout status: shows progress, handles rollout-specific logic
# - wait: generic, works with any condition on any resource
```

### Wait for Negative Conditions

```bash
# Wait for a condition to be False (use jsonpath):
kubectl wait node/node-1 \
  --for=jsonpath='{.status.conditions[?(@.type=="MemoryPressure")].status}'=False \
  --timeout=60s

# Wait for pod NOT to be Ready (draining scenario):
# Not directly supported — use a loop:
until ! kubectl get pod my-app -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' | grep -q True; do
  sleep 2
done
```

### Parallel Waits

```bash
#!/bin/bash
# Wait for multiple independent resources in parallel:
kubectl wait deployment/api --for=condition=Available --timeout=300s &
PID1=$!
kubectl wait deployment/worker --for=condition=Available --timeout=300s &
PID2=$!
kubectl wait deployment/scheduler --for=condition=Available --timeout=300s &
PID3=$!

# Wait for all to finish:
wait $PID1 $PID2 $PID3
echo "All deployments available"
```

## Gotchas

| Issue | Explanation | Workaround |
|-------|-------------|-----------|
| Resource doesn't exist yet | `kubectl wait` fails immediately if the resource isn't found | Retry loop, or `kubectl wait` after `kubectl apply` |
| Condition never exists | If the condition key never appears, wait hangs | Use jsonpath instead, or ensure controller sets conditions |
| Race with apply | `kubectl apply` returns before object is watchable | Add a small sleep, or use `kubectl apply --wait` |
| Multiple condition values | `--for=condition=Ready` checks for `status: "True"` only | Use `--for=condition=Ready=False` to wait for False |
| Label selector matches 0 pods | Returns success immediately (nothing to wait for) | Verify selector matches before waiting |

## Exit Codes

| Exit Code | Meaning |
|-----------|---------|
| 0 | Condition met (success) |
| 1 | Timeout expired, or error |

```bash
# Use in scripts:
if kubectl wait pod/my-app --for=condition=Ready --timeout=60s; then
  echo "Pod is ready"
else
  echo "Pod failed to become ready in 60s"
  kubectl describe pod my-app
  exit 1
fi
```

## Quick Reference

```bash
# Wait for condition (True):
kubectl wait <resource> --for=condition=<ConditionType> --timeout=<duration>

# Wait for condition=False:
kubectl wait <resource> --for=condition=<ConditionType>=False --timeout=<duration>

# Wait for deletion:
kubectl wait <resource> --for=delete --timeout=<duration>

# Wait for jsonpath value:
kubectl wait <resource> --for=jsonpath='{.path.to.field}'=<value> --timeout=<duration>

# Wait for all matching resources:
kubectl wait <resource> -l <label>=<value> --for=condition=<type> --timeout=<duration>
kubectl wait <resource> --all --for=condition=<type> --timeout=<duration>

# Common patterns:
kubectl wait pod/x --for=condition=Ready --timeout=120s
kubectl wait deployment/x --for=condition=Available --timeout=300s
kubectl wait job/x --for=condition=Complete --timeout=600s
kubectl wait node/x --for=condition=Ready --timeout=300s
kubectl wait pvc/x --for=jsonpath='{.status.phase}'=Bound --timeout=60s
kubectl wait pods -l app=x --for=delete --timeout=120s

# Tips:
# - Always set --timeout in CI/CD (default is infinite wait)
# - Check label selector matches something before waiting
# - Use exit code for conditional logic in scripts
# - For rollouts, prefer: kubectl rollout status deployment/x
```
