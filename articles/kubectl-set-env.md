# kubectl set env: Managing Container Environment Variables

`kubectl set env` updates environment variables on existing workloads without editing YAML by hand. It patches the pod template, so for controllers like Deployments and StatefulSets it triggers a rolling update automatically. This makes it handy for quick config changes, but for anything you want to keep, prefer declarative manifests or GitOps — imperative `set env` changes drift from your source of truth.

> On a Deployment, StatefulSet, or DaemonSet, `set env` edits the pod template and rolls out new pods. On a bare Pod it can't change a running container's env — a Pod's spec is largely immutable, so the value is recorded but only takes effect if the Pod is recreated.

## Basic Syntax

```sh
kubectl set env <resource-type>/<resource-name> <KEY>=<value>
```

## Set Environment Variables

### Single Variable

```sh
kubectl set env deployment/myapp DATABASE_URL=postgres://localhost:5432/mydb
kubectl set env statefulset/mydb LOG_LEVEL=debug
kubectl set env daemonset/myds LOG_LEVEL=debug
```

### Multiple Variables

```sh
kubectl set env deployment/myapp \
  DATABASE_URL=postgres://localhost:5432/mydb \
  API_KEY=abc123 \
  LOG_LEVEL=info

kubectl set env deployment/myapp PORT=8080 DEBUG=true MAX_CONNECTIONS=100
```

## Remove Environment Variables

Append a trailing hyphen (`KEY-`) to remove a variable.

```sh
# Single
kubectl set env deployment/myapp DATABASE_URL-

# Multiple
kubectl set env deployment/myapp DATABASE_URL- API_KEY- LOG_LEVEL-
```

## Sourcing Values

### From a ConfigMap

```sh
# Import every key from the ConfigMap as env vars
kubectl set env deployment/myapp --from=configmap/myconfig

# Import all keys, prefixed
kubectl set env deployment/myapp --from=configmap/myconfig --prefix=CONFIG_
```

### From a Secret

```sh
# Import every key from the Secret
kubectl set env deployment/myapp --from=secret/mysecret

# Import all keys, prefixed
kubectl set env deployment/myapp --from=secret/mysecret --prefix=APP_
```

> `--from` imports keys as `valueFrom` references (`configMapKeyRef` / `secretKeyRef`), so the values stay linked to the source object rather than being copied inline. Use `--keys` to import only specific keys:
> ```sh
> kubectl set env deployment/myapp --from=configmap/myconfig --keys=redis-url,log-level
> ```

## Target Specific Containers

Without `-c`, `set env` applies to **all** containers in the pod template. Use `-c` (which accepts a glob/pattern) to scope it.

```sh
# One container
kubectl set env deployment/myapp -c web-container LOG_LEVEL=debug

# By pattern (all containers starting with "web")
kubectl set env deployment/myapp -c 'web*' LOG_LEVEL=debug
```

## Namespace and Selector Targeting

```sh
# Specific namespace
kubectl set env deployment/myapp -n production DATABASE_URL=postgres://prod:5432/mydb

# All matching resources by label
kubectl set env deployment -l app=myapp DATABASE_URL=postgres://localhost:5432/mydb

# Multiple resource types at once
kubectl set env deployment,statefulset -l tier=backend API_ENDPOINT=https://api.example.com

# Every resource of a type in the namespace
kubectl set env deployment --all LOG_LEVEL=info
```

## Preview and Overwrite

```sh
# See the change without applying it
kubectl set env deployment/myapp DATABASE_URL=postgres://localhost:5432/mydb --dry-run=client -o yaml

# Overwrite an existing value (required when the key already exists)
kubectl set env deployment/myapp DATABASE_URL=postgres://new-host:5432/mydb --overwrite

# Print the resulting spec instead of applying (list current env)
kubectl set env deployment/myapp --list
```

## Supported Resource Types

Any resource with a pod template, plus bare Pods:

```sh
kubectl set env deployment/myapp   KEY=value
kubectl set env statefulset/myss   KEY=value
kubectl set env daemonset/myds     KEY=value
kubectl set env replicaset/myrs    KEY=value
kubectl set env job/myjob          KEY=value
kubectl set env cronjob/mycron     KEY=value
kubectl set env pod/mypod          KEY=value   # recorded, effective only on recreate
```

## Common Patterns

### Database Configuration

```sh
kubectl set env deployment/webapp \
  DB_HOST=postgres.default.svc.cluster.local \
  DB_PORT=5432 \
  DB_NAME=myapp
kubectl set env deployment/webapp --from=secret/db-credentials --prefix=DB_
```

### Application Configuration

```sh
kubectl set env deployment/api \
  PORT=8080 NODE_ENV=production LOG_LEVEL=info CONFIG_PATH=/app/config
kubectl set env deployment/api --from=configmap/app-config
```

### Remove All Environment Variables

```sh
kubectl get deployment myapp \
  -o jsonpath='{.spec.template.spec.containers[0].env[*].name}' \
  | xargs -n1 -I{} kubectl set env deployment/myapp {}-
```

## Verify and Roll Back

```sh
# What the template declares
kubectl set env deployment/myapp --list
kubectl get deployment myapp -o jsonpath='{.spec.template.spec.containers[*].env}'

# What the running container actually sees
kubectl exec -it <pod-name> -- env | sort

# Roll back the change
kubectl rollout undo deployment/myapp
kubectl rollout status deployment/myapp
```

## Best Practices

- **Secrets for sensitive values**: reference `secret/...` via `--from`; never pass passwords or tokens as plain `KEY=value` (they'd be visible in shell history and the object spec).
- **ConfigMaps for configuration**: keep tunables out of the workload spec and link them with `--from`.
- **Preview first**: run with `--dry-run=client -o yaml` before applying.
- **Prefer declarative for anything lasting**: imperative `set env` is great for a quick fix, but commit the change to your manifests/GitOps repo so it isn't lost on the next apply.
- **Validate the rollout**: confirm new pods start cleanly with `kubectl rollout status`, then check the actual env with `kubectl exec ... -- env`.

## Common Errors

| Message | Cause | Fix |
|---------|-------|-----|
| `environment variable "X" already has a value, and --overwrite is false` | Key exists and you didn't allow replacement | Add `--overwrite` |
| `error: at least one environment variable must be provided` | No `KEY=value`, `KEY-`, or `--from` given | Provide an assignment, removal, or source |
| No rollout happens | Applied to a bare `pod/` | Target the controller (`deployment/`, `statefulset/`, ...) |
| `secrets "X" not found` | `--from` references a missing object | Create the Secret/ConfigMap or fix the name/namespace |

## Quick Reference

```sh
# Set
kubectl set env deploy/app KEY=value
kubectl set env deploy/app K1=v1 K2=v2 K3=v3
kubectl set env deploy/app KEY=value --overwrite

# Remove
kubectl set env deploy/app KEY-

# From sources
kubectl set env deploy/app --from=configmap/cm
kubectl set env deploy/app --from=secret/sec --prefix=APP_
kubectl set env deploy/app --from=configmap/cm --keys=k1,k2

# Scope
kubectl set env deploy/app -c container KEY=value
kubectl set env deploy -l app=web KEY=value
kubectl set env deploy/app -n prod KEY=value

# Inspect / preview / undo
kubectl set env deploy/app --list
kubectl set env deploy/app KEY=value --dry-run=client -o yaml
kubectl rollout undo deploy/app
```
