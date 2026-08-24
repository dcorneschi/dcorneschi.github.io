# kubectl run & expose — Pod Creation and Service Exposure

Quick reference for creating pods and exposing them as services using `kubectl run` and `kubectl expose`.

## kubectl run — Create a Pod

`kubectl run` creates a bare pod (not a Deployment). Useful for quick tests, debugging, and one-off tasks.

### Basic Pod

```bash
kubectl run nginx --image=nginx
```

### With Port

```bash
kubectl run nginx --image=nginx --port=80
```

### With Environment Variables

```bash
kubectl run myapp --image=myapp:1.0 --env="APP_ENV=production" --env="LOG_LEVEL=info"
```

### With Resource Requests/Limits

```bash
kubectl run nginx --image=nginx --requests='cpu=100m,memory=128Mi' --limits='cpu=500m,memory=256Mi'
```

### With Labels

```bash
kubectl run nginx --image=nginx --labels="app=web,tier=frontend"
```

### With Command Override

```bash
kubectl run busybox --image=busybox --command -- /bin/sh -c "sleep 3600"
```

### With Restart Policy (for Jobs)

```bash
kubectl run job-pod --image=busybox --restart=Never -- /bin/sh -c "echo done"
```

| `--restart` | What It Creates |
|-------------|-----------------|
| `Always` (default) | Pod |
| `Never` | Pod that runs once (like a Job) |

### Dry Run (Generate YAML)

```bash
kubectl run nginx --image=nginx --port=80 --dry-run=client -o yaml > pod.yaml
```

### Interactive/Debugging Pods

```bash
# Start a shell in a temporary pod (deleted on exit)
kubectl run debug --image=busybox --rm -it -- /bin/sh

# curl from inside the cluster
kubectl run curl --image=curlimages/curl --rm -it -- curl http://my-service.default.svc.cluster.local

# DNS debugging
kubectl run dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 --rm -it -- nslookup kubernetes

# Network debugging with nicolaka/netshoot
kubectl run netshoot --image=nicolaka/netshoot --rm -it -- /bin/bash
```

## kubectl expose — Create a Service

`kubectl expose` creates a Service that routes traffic to pods matching the selector.

### Expose a Pod

```bash
kubectl expose pod nginx --port=80 --target-port=80 --name=nginx-svc
```

### Expose a Deployment

```bash
kubectl expose deployment my-app --port=80 --target-port=8080
```

### Service Types

```bash
# ClusterIP (default — internal only)
kubectl expose deployment my-app --port=80 --target-port=8080 --type=ClusterIP

# NodePort (accessible on every node's IP at a random high port)
kubectl expose deployment my-app --port=80 --target-port=8080 --type=NodePort

# LoadBalancer (provisions cloud LB)
kubectl expose deployment my-app --port=80 --target-port=8080 --type=LoadBalancer
```

### With a Specific NodePort

```bash
kubectl expose deployment my-app --port=80 --target-port=8080 --type=NodePort --overrides='{"spec":{"ports":[{"port":80,"targetPort":8080,"nodePort":30080}]}}'
```

### Dry Run (Generate YAML)

```bash
kubectl expose deployment my-app --port=80 --target-port=8080 --dry-run=client -o yaml > service.yaml
```

## Common Patterns

### Create Pod + Expose in One Flow

```bash
kubectl run nginx --image=nginx --port=80
kubectl expose pod nginx --port=80 --target-port=80 --type=ClusterIP
```

### Quick Test: Create, Expose, Verify, Cleanup

```bash
# Create
kubectl run test-web --image=nginx --port=80
kubectl expose pod test-web --port=80 --type=ClusterIP --name=test-svc

# Verify
kubectl run curl --image=curlimages/curl --rm -it -- curl http://test-svc

# Cleanup
kubectl delete pod test-web
kubectl delete svc test-svc
```

### Generate Full Manifests for Version Control

```bash
kubectl run my-app --image=my-app:1.0 --port=8080 \
  --labels="app=my-app,env=prod" \
  --requests='cpu=100m,memory=128Mi' \
  --limits='cpu=500m,memory=256Mi' \
  --dry-run=client -o yaml > pod.yaml

kubectl expose pod my-app --port=80 --target-port=8080 \
  --type=ClusterIP \
  --dry-run=client -o yaml > service.yaml
```

## Port Terminology

| Field | Where | Meaning |
|-------|-------|---------|
| `port` | Service | Port the Service listens on (what clients connect to) |
| `targetPort` | Service | Port on the pod the Service forwards to |
| `containerPort` | Pod | Port the container listens on (informational, not enforced) |
| `nodePort` | Service (NodePort type) | Port on every node's IP (30000–32767 range) |

```
Client → Service:port → Pod:targetPort (== containerPort)
Client → Node:nodePort → Service:port → Pod:targetPort
```

## Pod Reachability Without a Service

Every pod gets its own IP from the CNI. Any other pod in the cluster can reach it directly:

```bash
# Find the pod's IP
kubectl get pod hazelcast -o wide

# From another pod, reach it directly
kubectl run test --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- 10.244.1.23:5701
```

The catch: pod IPs are ephemeral. If the pod restarts or gets rescheduled, it gets a new IP. There's no DNS entry for a bare pod. Anyone talking to it by IP breaks the moment the pod moves.

That's exactly what a Service solves — it gives you a stable DNS name and IP that stays the same regardless of what happens to the pod behind it.

| Who | Can reach the pod? |
|-----|-------------------|
| Pods in same namespace | Yes, via pod IP |
| Pods in other namespaces | Yes, via pod IP |
| Nodes | Yes, via pod IP (CNI routes on every node) |
| External clients | No — no route into the pod network |

## Service Reachability by Type

| Service type | Cluster pods | Nodes | External clients |
|--------------|-------------|-------|------------------|
| `ClusterIP` | Yes | Yes | No |
| `NodePort` | Yes | Yes | Yes (via node IP:nodePort) |
| `LoadBalancer` | Yes | Yes | Yes (via LB IP) |

With a `ClusterIP` Service:
- Any pod in any namespace can reach it — Kubernetes has no network restrictions by default
- A pod in namespace `billing` can call `hazelcast.default.svc.cluster.local:5701`
- Nodes can reach the ClusterIP too (kube-proxy rules on every node)
- Nothing outside the cluster can reach it

## Traffic Flow

```
Other pod → hazelcast.default.svc.cluster.local:5701 → Service (ClusterIP) → Pod hazelcast:5701
```

The Service gets a DNS name automatically: `<service-name>.<namespace>.svc.cluster.local`.

## Restricting Access with NetworkPolicy

Lock down so only pods in the same namespace can reach port 5701:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: hazelcast-allow-same-namespace
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: hazelcast
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}         # only pods in the same namespace
    ports:
    - port: 5701
      protocol: TCP
```

> NetworkPolicies only work if the cluster's CNI plugin supports them (Calico, Cilium, Weave — yes. Flannel — no). Without a supporting CNI, the NetworkPolicy resource is accepted but has no effect.

## kubectl run vs kubectl create deployment

| | `kubectl run` | `kubectl create deployment` |
|---|---|---|
| Creates | Bare Pod | Deployment → ReplicaSet → Pod |
| Self-healing | No (pod dies = gone) | Yes (ReplicaSet recreates pod) |
| Scaling | No | `kubectl scale deployment` |
| Rolling updates | No | Yes |
| Use case | Quick tests, debug pods, one-off tasks | Production workloads |

```bash
# Create a Deployment (preferred for real workloads)
kubectl create deployment nginx --image=nginx --port=80 --replicas=3

# Then expose it
kubectl expose deployment nginx --port=80 --target-port=80 --type=LoadBalancer
```

## Useful Flags Reference

### kubectl run

| Flag | Purpose |
|------|---------|
| `--image` | Container image |
| `--port` | Container port to expose |
| `--env` | Set environment variables |
| `--labels` | Set labels (comma-separated) |
| `--requests` | Resource requests (`cpu=100m,memory=128Mi`) |
| `--limits` | Resource limits |
| `--command` | Override entrypoint (everything after `--` is the command) |
| `--restart=Never` | Don't restart on exit (run-to-completion) |
| `--rm` | Delete pod after it exits |
| `-it` | Interactive + TTY (for shells) |
| `--dry-run=client -o yaml` | Generate YAML without creating |
| `--overrides` | Inline JSON patch for advanced fields |

### kubectl expose

| Flag | Purpose |
|------|---------|
| `--port` | Service port |
| `--target-port` | Pod port to forward to |
| `--type` | Service type: ClusterIP, NodePort, LoadBalancer |
| `--name` | Service name (defaults to resource name) |
| `--selector` | Override label selector |
| `--dry-run=client -o yaml` | Generate YAML without creating |
