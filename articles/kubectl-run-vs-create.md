<img src="/articles/images/kubernetes-logo.svg" alt="Kubernetes" width="150">

# kubectl run vs kubectl create — Quick Reference

## The Short Answer

- `kubectl run` — creates a **bare Pod**
- `kubectl create` — creates **other resources** (Deployments, Jobs, CronJobs, etc.), not bare Pods
- There is no `kubectl create pod` command

## kubectl run

Creates a standalone Pod imperatively. Mainly used for quick debugging or exam scenarios.

```bash
# Basic pod
kubectl run nginx --image=nginx

# Pod with a command
kubectl run test --image=busybox --command -- sleep 3600

# Interactive shell, auto-deleted on exit
kubectl run debug --rm -it --image=busybox -- sh

# With resource requests (use --overrides since --requests was removed)
kubectl run nginx --image=nginx --overrides='{"spec":{"containers":[{"name":"nginx","image":"nginx","resources":{"requests":{"cpu":"100m","memory":"128Mi"}}}]}}'

# With labels
kubectl run nginx --image=nginx --labels='app=web,env=dev'

# Dry-run to generate YAML (useful for exams)
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# With port exposed
kubectl run nginx --image=nginx --port=80

# With environment variables
kubectl run nginx --image=nginx --env="DB_HOST=postgres" --env="DB_PORT=5432"

# With restart policy (default: Always)
kubectl run job-pod --image=busybox --restart=Never --command -- echo "done"
```

### Useful flags

| Flag | Purpose |
|------|---------|
| `--rm` | Delete pod after it exits |
| `-it` | Interactive TTY (attach to shell) |
| `--restart=Never` | Run once, don't restart on exit |
| `--dry-run=client -o yaml` | Generate YAML without creating |
| `--command --` | Override entrypoint (everything after `--` is the command) |
| `--labels` | Set pod labels |
| `--env` | Set environment variables |
| `--overrides` | JSON patch for resource requests, limits, nodeSelector, etc. |

## kubectl create

Creates higher-level resources. Does NOT create bare Pods.

```bash
# Deployment
kubectl create deployment nginx --image=nginx --replicas=3

# Job
kubectl create job my-job --image=busybox -- echo hello

# CronJob
kubectl create cronjob my-cron --image=busybox --schedule="*/5 * * * *" -- echo hi

# Service
kubectl create service clusterip my-svc --tcp=80:8080

# ConfigMap
kubectl create configmap my-config --from-literal=key=value

# Secret
kubectl create secret generic my-secret --from-literal=password=s3cr3t

# Namespace
kubectl create namespace staging

# ServiceAccount
kubectl create serviceaccount my-sa

# Role / RoleBinding
kubectl create role pod-reader --verb=get,list --resource=pods
kubectl create rolebinding read-pods --role=pod-reader --serviceaccount=default:my-sa
```

## Creating Pods from YAML

For anything beyond quick debugging, use declarative YAML:

```bash
# Create (fails if resource exists)
kubectl create -f pod.yaml

# Apply (creates or updates — preferred for production)
kubectl apply -f pod.yaml
```

### Generating YAML templates

```bash
# Generate Pod YAML
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Generate Deployment YAML
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml

# Generate Job YAML
kubectl create job my-job --image=busybox --dry-run=client -o yaml -- echo hi > job.yaml
```

## When to Use What

| Scenario | Command |
|----------|---------|
| Quick debug shell | `kubectl run debug --rm -it --image=busybox -- sh` |
| Test an image quickly | `kubectl run test --image=myapp:latest` |
| Generate YAML for exam | `kubectl run ... --dry-run=client -o yaml` |
| Production workload | `kubectl apply -f deployment.yaml` |
| One-off task | `kubectl create job ...` |
| Scheduled task | `kubectl create cronjob ...` |

## Key Differences

| Aspect | `kubectl run` | `kubectl create` |
|--------|---------------|-------------------|
| Creates | Bare Pod | Deployment, Job, CronJob, Service, etc. |
| Pod management | No controller (pod dies = gone) | Managed by controller (auto-restart/recreate) |
| Use case | Debugging, testing, exams | Production resources |
| Rollback | Not possible | `kubectl rollout undo` (Deployments) |
| Scaling | Not possible | `kubectl scale` (Deployments) |
| Self-healing | None | Controller recreates failed pods |

## Pro Tips

```bash
# Quickly test network connectivity from inside the cluster
kubectl run curl --rm -it --image=curlimages/curl -- curl http://my-service:8080/health

# DNS debugging
kubectl run dnstest --rm -it --image=busybox -- nslookup kubernetes.default

# Test a specific node (bypass scheduler)
kubectl run test --image=busybox --overrides='{"spec":{"nodeName":"worker-1"}}' --command -- sleep 3600

# Run with a specific service account
kubectl run test --image=busybox --overrides='{"spec":{"serviceAccountName":"my-sa"}}' --command -- sleep 3600
```
