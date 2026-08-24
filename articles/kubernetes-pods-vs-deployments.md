# Kubernetes Pods vs Deployments

Understanding the relationship between Pods and Deployments — when to use each, how they interact, and why bare Pods are almost never the right choice in production.

## The Relationship

```
Deployment → creates → ReplicaSet → creates → Pod(s)
```

A Deployment doesn't run your container directly. It creates a ReplicaSet, which in turn creates and manages the Pods. Each layer adds capabilities:

| Resource | What It Does |
|----------|-------------|
| **Pod** | Runs one or more containers. Smallest deployable unit. |
| **ReplicaSet** | Ensures N copies of a Pod are running at all times. |
| **Deployment** | Manages ReplicaSets. Handles rolling updates, rollbacks, scaling. |

## Quick Comparison

| | Pod | Deployment |
|---|---|---|
| **Created by** | `kubectl run` or YAML | `kubectl create deployment` or YAML |
| **Self-healing** | No — if it dies, it's gone | Yes — ReplicaSet recreates it |
| **Scaling** | No | Yes (`kubectl scale`) |
| **Rolling updates** | No | Yes (zero-downtime upgrades) |
| **Rollback** | No | Yes (`kubectl rollout undo`) |
| **Scheduling** | Scheduled once, never moved | New Pods scheduled if nodes fail |
| **Use case** | One-off tasks, debugging | Production workloads |

## Pod (Bare Pod)

A bare Pod is a single instance with no controller managing it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
```

```bash
# Create a bare pod
kubectl run nginx --image=nginx:1.25

# Or from YAML
kubectl apply -f pod.yaml
```

### What Happens When a Bare Pod Dies

- Node fails → Pod is **gone forever** (not rescheduled)
- Pod crashes → Kubelet restarts the container (if `restartPolicy: Always`)
- Pod is evicted (resource pressure) → **gone forever**
- You delete it → **gone forever**

No controller watches for it or recreates it.

## Deployment

A Deployment declares the desired state (image, replicas, resources) and Kubernetes continuously reconciles reality to match:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
```

```bash
# Create a deployment
kubectl create deployment nginx --image=nginx:1.25 --replicas=3

# Or from YAML
kubectl apply -f deployment.yaml
```

### What Happens When a Deployment's Pod Dies

- Node fails → ReplicaSet creates a new Pod on another node
- Pod crashes → ReplicaSet creates a replacement
- Pod is evicted → ReplicaSet creates a replacement
- You delete a Pod → ReplicaSet creates a replacement

The desired replica count is always maintained.

## When to Use a Bare Pod

Almost never in production. Valid use cases:

| Use Case | Why a Pod Works |
|----------|----------------|
| Quick debugging | `kubectl run debug --image=busybox --rm -it -- sh` |
| One-off commands | `kubectl run curl --image=curlimages/curl --rm -it -- curl http://...` |
| Static Pods (kubelet-managed) | Control plane components on kubeadm clusters |
| Learning/experimenting | Testing a container image quickly |

> If you need the workload to keep running, use a Deployment (or Job/CronJob for batch work).

## When to Use a Deployment

Any workload that should stay running:

| Use Case | Example |
|----------|---------|
| Web servers | nginx, Apache |
| APIs | REST/gRPC services |
| Microservices | Any long-running service |
| Workers | Queue consumers, stream processors |
| Databases (simple) | Single-replica stateless-ish DBs |

> For stateful workloads (databases with persistent identity), use a **StatefulSet** instead.

## Key Deployment Features

### Scaling

```bash
# Scale to 5 replicas
kubectl scale deployment nginx --replicas=5

# Autoscale based on CPU
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=70
```

### Rolling Updates

```bash
# Update the image (triggers rolling update)
kubectl set image deployment/nginx nginx=nginx:1.26

# Watch the rollout
kubectl rollout status deployment/nginx

# Check rollout history
kubectl rollout history deployment/nginx
```

### Rollback

```bash
# Undo the last update
kubectl rollout undo deployment/nginx

# Rollback to a specific revision
kubectl rollout undo deployment/nginx --to-revision=2
```

### Update Strategy

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Max pods above desired during update
      maxUnavailable: 0    # Zero-downtime (always have all replicas ready)
```

## The Hierarchy Visualized

```
kubectl get all -n my-app

NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-7c5ddbdf54-abc12   1/1     Running   0          5m
pod/nginx-7c5ddbdf54-def34   1/1     Running   0          5m
pod/nginx-7c5ddbdf54-ghi56   1/1     Running   0          5m

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   3/3     3            3           5m

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-7c5ddbdf54   3         3         3       5m
```

- **Deployment** `nginx` owns **ReplicaSet** `nginx-7c5ddbdf54`
- **ReplicaSet** owns the three **Pods**
- The hash `7c5ddbdf54` is derived from the pod template spec — it changes on updates

## Pod vs Deployment vs Other Controllers

| Controller | Use Case | Key Feature |
|-----------|----------|-------------|
| **Pod** (bare) | One-off, debugging | None — no management |
| **Deployment** | Stateless long-running services | Rolling updates, scaling, rollback |
| **StatefulSet** | Stateful apps (databases) | Stable network identity, ordered startup, persistent volumes |
| **DaemonSet** | One pod per node (agents, logging) | Runs on every node automatically |
| **Job** | Run-to-completion tasks | Runs once, tracks success |
| **CronJob** | Scheduled tasks | Creates Jobs on a schedule |
| **ReplicaSet** | (Don't use directly) | Use Deployment instead — it manages ReplicaSets for you |

## Common Mistakes

1. **Creating bare Pods in production** — They won't self-heal. Always use a Deployment.
2. **Editing Pods directly** — Pods are immutable (mostly). Change the Deployment spec and let it roll out new Pods.
3. **Using ReplicaSet directly** — Always use a Deployment. It adds rolling updates and rollback on top of ReplicaSet.
4. **Forgetting labels** — The Deployment's `selector.matchLabels` must match `template.metadata.labels`. Mismatches cause errors.
5. **Not setting resource requests** — Without them, the scheduler can't make good placement decisions and Pods may be evicted first under pressure.

## Useful Commands

```bash
# List deployments
kubectl get deployments -A

# Describe a deployment (shows events, conditions, strategy)
kubectl describe deployment <name>

# See which ReplicaSets a Deployment manages
kubectl get replicasets -l app=<label>

# See pods owned by a Deployment
kubectl get pods -l app=<label> -o wide

# Check why a pod isn't starting
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous

# Force restart all pods in a deployment (without changing spec)
kubectl rollout restart deployment/<name>

# Pause a rollout (for canary-style manual verification)
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>

# Delete deployment (also deletes ReplicaSets and Pods)
kubectl delete deployment <name>

# Delete a bare pod
kubectl delete pod <name>
```
