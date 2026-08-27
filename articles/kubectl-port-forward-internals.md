# How kubectl port-forward Works

The connection lifecycle of `kubectl port-forward` — how kubectl opens a local listener, tunnels traffic through the API server to the kubelet, and why connections sometimes drop.

## High-Level Flow

```
┌──────────────┐     ┌──────────┐     ┌───────────────┐     ┌──────────┐     ┌─────────┐
│ Local Client │────▶│  kubectl │────▶│   API Server  │────▶│  Kubelet │────▶│   Pod   │
│ (browser,    │     │ (local   │     │  (proxy)      │     │ (on node)│     │ (target │
│  curl, app)  │     │ listener)│     │               │     │          │     │  port)  │
└──────────────┘     └──────────┘     └───────────────┘     └──────────┘     └─────────┘
                     localhost:8080                                            pod:80
```

## What port-forward Does

```bash
kubectl port-forward pod/my-app 8080:80
```

This creates:
1. A TCP listener on `localhost:8080` (your machine)
2. A tunnel through the API server to the pod
3. Any connection to `localhost:8080` gets forwarded to `pod-ip:80`

```
Your machine:8080 ═══tunnel═══▶ Pod's container:80
```

## Step 1: kubectl Opens a Local Listener

```
┌──────────────────────────────────────────────────┐
│  kubectl port-forward                            │
│                                                  │
│  1. Resolve pod name to pod object               │
│  2. Confirm pod is Running                       │
│  3. Open TCP listener on localhost:8080          │
│  4. Print: "Forwarding from 127.0.0.1:8080 → 80" │
│  5. Wait for incoming connections                │
└──────────────────────────────────────────────────┘
```

```bash
# After running port-forward, verify the listener:
lsof -i :8080
# COMMAND   PID  USER   FD   TYPE  DEVICE  NODE NAME
# kubectl  1234  jane   8u   IPv4  0x1234  TCP  *:8080 (LISTEN)

# Or:
ss -tlnp | grep 8080
```

## Step 2: Connection Arrives — Tunnel Established

When a local client connects to `localhost:8080`:

```
┌────────────────────────────────────────────────────────────────────┐
│  Per-Connection Flow                                               │
│                                                                    │
│  1. Client connects to localhost:8080                              │
│  2. kubectl accepts the TCP connection                             │
│  3. kubectl sends POST to API server:                              │
│     POST /api/v1/namespaces/default/pods/my-app/portforward        │
│     Upgrade: SPDY/3.1 (or WebSocket)                               │
│  4. API server upgrades connection                                 │
│  5. API server proxies to kubelet on pod's node (:10250)           │
│  6. Kubelet connects to pod's container port                       │
│  7. Data flows bidirectionally through the tunnel                  │
│  8. When either side closes, the tunnel tears down                 │
└────────────────────────────────────────────────────────────────────┘
```

### The portforward Subresource

```
POST /api/v1/namespaces/default/pods/my-app/portforward
Upgrade: SPDY/3.1
Connection: Upgrade
```

Response:

```
HTTP/1.1 101 Switching Protocols
Upgrade: SPDY/3.1
Connection: Upgrade
```

### Stream Multiplexing

Each forwarded port gets two SPDY streams (data + error):

```
Stream 0: port 80 data (bidirectional)
Stream 1: port 80 error channel

Stream 2: port 443 data (if forwarding multiple ports)
Stream 3: port 443 error channel
```

The streams carry raw TCP bytes — kubectl doesn't inspect or modify the traffic.

## Step 3: Kubelet Connects to the Pod

The kubelet uses `nsenter` or CRI to connect to the pod's network namespace and open a TCP connection to the target port:

```
┌──────────┐                              ┌─────────────────────────┐
│  Kubelet │                              │  Pod Network Namespace  │
│          │── enter netns ──────────────▶│                         │
│          │── connect(127.0.0.1:80) ────▶│  Container listening    │
│          │                              │  on port 80             │
│          │◀═══ TCP data ═══════════════▶│                         │
└──────────┘                              └─────────────────────────┘
```

The connection is to `localhost:port` inside the pod's network namespace — same as if you were inside the pod running `curl localhost:80`.

## Port Forwarding Variations

### Pod Port-Forward

```bash
# Forward local 8080 to pod port 80:
kubectl port-forward pod/my-app 8080:80

# Same local and remote port:
kubectl port-forward pod/my-app 80

# Multiple ports:
kubectl port-forward pod/my-app 8080:80 8443:443

# Random local port (useful in scripts):
kubectl port-forward pod/my-app :80
# Forwarding from 127.0.0.1:43721 -> 80

# Listen on all interfaces (not just localhost):
kubectl port-forward --address 0.0.0.0 pod/my-app 8080:80

# Listen on specific IP:
kubectl port-forward --address 192.168.1.10 pod/my-app 8080:80
```

### Service Port-Forward

```bash
# Forward to a Service (kubectl picks a backing pod):
kubectl port-forward svc/my-app 8080:80
```

When targeting a Service:
1. kubectl resolves the Service to its Endpoints
2. Picks one pod from the endpoint list
3. Forwards directly to that pod (NOT through the Service ClusterIP)
4. If that pod dies, the connection breaks (no failover)

### Deployment Port-Forward

```bash
# Forward to a Deployment (kubectl picks one pod):
kubectl port-forward deployment/my-app 8080:80
```

Same behavior — kubectl picks one pod owned by the Deployment's active ReplicaSet.

## Connection Lifecycle

```
Time ──────────────────────────────────────────────────────────────────▶

Browser         kubectl          API Server       Kubelet        Pod
   │               │               │               │              │
   │               │ listen :8080  │               │              │
   │               │               │               │              │
   │ connect ─────▶│               │               │              │
   │               │ POST /portfwd▶│               │              │
   │               │               │── connect ──▶ │              │
   │               │               │               │── connect ──▶│
   │               │ ◀─ 101 ─────  │               │              │
   │               │               │               │              │
   │═ HTTP GET ═══▶│══════════════▶│══════════════▶│═════════════▶│
   │◀═ response ══ │◀══════════════│◀══════════════│◀═════════════│
   │               │               │               │              │
   │═ more data ══▶│══════════════▶│══════════════▶│═════════════▶│
   │               │               │               │              │
   │ close ───────▶│               │               │              │
   │               │── close ─────▶│── close ────▶ │── close ────▶│
   │               │               │               │              │
   │               │ (still listening on :8080     │              │
   │               │  for next connection)         │              │
```

## Why Connections Drop

### Idle Timeout

The tunnel has no TCP keepalive by default. Load balancers or firewalls between you and the API server may kill idle connections:

```
kubectl ←── 60s idle ──→ API server
                │
        ALB drops the connection
        (default idle timeout: 60s)
```

**Fix**: Send periodic traffic, or configure LB idle timeout higher.

### Pod Restart

If the target pod restarts, all port-forward connections to it break:

```
Pod crashes → new pod starts (different IP) → tunnel invalid → connection refused
```

**Fix**: Use a wrapper script that reconnects, or use a tool like `kubefwd`.

### Node Network Issues

If the node becomes unreachable, the kubelet connection drops:

```
API server ←× lost connection ×→ Kubelet
    │
    ▼
All port-forward sessions to pods on that node die
```

### API Server Restart / Upgrade

Control plane upgrades tear down all active port-forward tunnels.

## Multiple Simultaneous Connections

kubectl handles multiple concurrent connections on the same forwarded port:

```
Browser tab 1 → localhost:8080 ═══▶ Pod:80 (connection A)
Browser tab 2 → localhost:8080 ═══▶ Pod:80 (connection B)
curl          → localhost:8080 ═══▶ Pod:80 (connection C)
```

Each connection gets its own SPDY stream pair through the same tunnel. They're independent — one can close without affecting others.

## port-forward vs Service/Ingress

| Feature | port-forward | Service (NodePort/LB) | Ingress |
|---------|-------------|----------------------|---------|
| Setup | Instant (no config) | Requires manifest | Requires controller + manifest |
| Persistence | Dies with kubectl process | Permanent | Permanent |
| Access | Only from your machine | Cluster-wide or external | External (HTTP/HTTPS) |
| Load balancing | Single pod | Yes (all backends) | Yes |
| Survives pod restart | No | Yes (endpoints update) | Yes |
| Use case | Debugging, dev access | Production traffic | Production HTTP routing |

## Security / RBAC

```yaml
# RBAC needed for port-forward:
rules:
- apiGroups: [""]
  resources: ["pods/portforward"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get"]    # To resolve pod and check it's Running
```

Port-forward exposes pod ports to your local machine. Anyone with access to your machine can reach the forwarded port. Be careful with `--address 0.0.0.0` in shared environments.

## Debugging port-forward Issues

```bash
# See what kubectl is doing:
kubectl port-forward pod/my-app 8080:80 -v=8

# Verify pod is running and has the port:
kubectl get pod my-app -o jsonpath='{.status.phase}'
kubectl get pod my-app -o jsonpath='{.spec.containers[*].ports[*].containerPort}'

# Verify something is listening inside the pod:
kubectl exec my-app -- ss -tlnp | grep 80
kubectl exec my-app -- netstat -tlnp | grep 80

# Test connectivity without port-forward (from another pod):
kubectl run curl --image=curlimages/curl --rm -it -- curl http://<pod-ip>:80

# Check if port-forward process is still alive:
ps aux | grep "port-forward"

# Common error messages:
# "error: unable to forward port because pod is not running"
#   → Pod crashed or was deleted
#
# "error: an error occurred forwarding 8080 -> 80"
#   → Nothing listening on port 80 in the container
#   → Or connection was refused
#
# "Handling connection for 8080" (printed per connection)
#   → Normal — shows each new connection being tunneled
```

## Quick Reference

```bash
# Basic port-forward:
kubectl port-forward pod/my-app 8080:80

# Service (picks one pod):
kubectl port-forward svc/my-app 8080:80

# Multiple ports:
kubectl port-forward pod/my-app 8080:80 9090:9090

# Listen on all interfaces:
kubectl port-forward --address 0.0.0.0 pod/my-app 8080:80

# Random local port:
kubectl port-forward pod/my-app :80

# Background it:
kubectl port-forward pod/my-app 8080:80 &

# Connection path:
# localhost:8080 → kubectl → API server → kubelet → pod netns → container:80

# Protocol: SPDY/3.1 or WebSocket (same as exec)
# Each connection = separate stream pair through the tunnel

# RBAC required:
# resources: ["pods/portforward"], verbs: ["create"]
# resources: ["pods"], verbs: ["get"]

# Limitations:
# - Single pod (no load balancing)
# - Dies with kubectl process
# - No auto-reconnect on pod restart
# - Subject to idle timeouts on proxies/LBs
```
