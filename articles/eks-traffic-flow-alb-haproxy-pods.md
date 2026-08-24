# EKS Traffic Flow: External → ALB → HAProxy → Services/Ingress → Pods

End-to-end request flow through an EKS cluster using AWS ALB as the external entry point and HAProxy as the ingress controller.

## Full Traffic Path

```
┌──────────┐     ┌─────────────┐     ┌──────────────────────┐     ┌─────────────┐     ┌──────┐
│  Client  │────▶│  AWS ALB    │────▶│  HAProxy Ingress     │────▶│  Service    │────▶│ Pod  │
│(internet)│     │(Layer 7 LB) │     │  Controller Pod      │     │ (ClusterIP) │     │      │
└──────────┘     └─────────────┘     └──────────────────────┘     └─────────────┘     └──────┘
     │                  │                       │                        │                  │
     │  HTTPS           │  HTTP/HTTPS           │  HTTP                  │  HTTP            │
     │  (TLS terminated │  (to target group     │  (routed by            │  (iptables       │
     │   or passed)     │   registration)       │   ingress rules)       │   DNAT)          │
```

## Layer-by-Layer Breakdown

### Layer 1: Client → AWS ALB

```
Client (browser/API)
  │
  │ DNS resolution: app.example.com → ALB DNS name → ALB IP(s)
  │ Protocol: HTTPS (port 443)
  │
  ▼
AWS Application Load Balancer
  │
  │ - Terminates TLS (using ACM certificate)
  │ - Evaluates listener rules (host/path-based routing)
  │ - Selects target group based on rules
  │ - Performs health checks against targets
  │ - Adds X-Forwarded-For, X-Forwarded-Proto headers
  │
  ▼
Target Group (HAProxy pods registered as targets)
```

**Key details:**
- ALB operates at Layer 7 (HTTP/HTTPS)
- TLS is typically terminated at the ALB (offloads crypto from the cluster)
- Target type can be `instance` (routes to NodePort) or `ip` (routes directly to pod IP)
- ALB is multi-AZ — has at least one IP per AZ in its configured subnets

### Layer 2: ALB → HAProxy Ingress Controller

```
ALB Target Group
  │
  │ Target type: IP (pod IP directly) or Instance (NodePort)
  │
  ├─── IP mode: ALB → HAProxy pod IP directly (bypasses kube-proxy)
  │    Traffic: ALB → VPC routing → Node ENI → veth → HAProxy pod
  │
  └─── Instance mode: ALB → Node IP:NodePort → kube-proxy → HAProxy pod
       Traffic: ALB → Node ENI → iptables DNAT → HAProxy pod
```

| Target Type | Path | Pros | Cons |
|-------------|------|------|------|
| `ip` | ALB → pod directly | Lower latency, no extra hop, preserves source IP | Requires VPC CNI, pods must be in ALB's subnet reach |
| `instance` | ALB → NodePort → pod | Works with any CNI, simpler setup | Extra hop through kube-proxy, SNAT hides source IP |

**HAProxy Ingress Controller deployment:**
- Runs as a Deployment or DaemonSet in the cluster
- Each HAProxy pod is registered in the ALB target group
- ALB health-checks HAProxy pods directly (typically on `/healthz`)

### Layer 3: HAProxy → Backend Service/Pod

```
HAProxy Ingress Controller Pod
  │
  │ 1. Receives request from ALB
  │ 2. Matches request against Ingress rules (host + path)
  │ 3. Selects backend based on matching rule
  │ 4. Forwards request to backend pod(s)
  │
  ├─── Direct to Pod IP (if endpoints are configured directly)
  │    HAProxy → veth → host routing → veth → backend pod
  │
  └─── Via ClusterIP Service
       HAProxy → kube-proxy iptables → DNAT → backend pod
```

**HAProxy routing decision:**
- Reads Ingress resources or custom CRDs
- Maps host + path → backend service name + port
- Resolves backend to pod endpoints (watches Endpoints/EndpointSlices)
- Load balances across healthy pods (round-robin, least-conn, etc.)

### Layer 4: Service → Pod

```
Service (ClusterIP)
  │
  │ kube-proxy maintains iptables/IPVS rules:
  │   ClusterIP:port → DNAT to one of the pod IPs
  │
  │ Endpoint selection:
  │   - Random (iptables mode)
  │   - Least connections (IPVS mode)
  │   - Only Ready pods are included
  │
  ▼
Pod (application container)
  │
  │ Receives request on targetPort
  │ Processes and returns response
  │ Response follows the reverse path
```

> **Note:** If HAProxy routes directly to pod IPs (common with modern ingress controllers), it bypasses the ClusterIP entirely. The Service still exists for discovery but traffic doesn't flow through kube-proxy.

## Complete Packet Journey (IP Mode)

```
1. Client sends HTTPS request to app.example.com
   └─ DNS resolves to ALB IP (e.g., 52.1.2.3)

2. ALB receives request on port 443
   └─ Terminates TLS
   └─ Evaluates listener rules: Host=app.example.com, Path=/api
   └─ Matches target group: haproxy-tg
   └─ Selects healthy target: 10.0.1.50:80 (HAProxy pod)
   └─ Adds headers: X-Forwarded-For: <client-ip>

3. Packet enters VPC
   └─ Src: ALB ENI IP (10.0.0.100)
   └─ Dst: 10.0.1.50 (HAProxy pod)
   └─ VPC routing → subnet route table → node ENI

4. Packet arrives at node hosting HAProxy pod
   └─ Node kernel receives on eth0/eth1
   └─ ip route: 10.0.1.50 dev eniXXX (HAProxy pod's veth)
   └─ Forwarded through veth pair into HAProxy pod netns

5. HAProxy pod receives request
   └─ Matches Ingress rule: host=app.example.com, path=/api
   └─ Backend: api-service → endpoints: [10.0.2.10, 10.0.2.11]
   └─ Selects backend pod: 10.0.2.10:8080
   └─ Forwards request

6. Packet to backend pod
   └─ Src: 10.0.1.50 (HAProxy pod)
   └─ Dst: 10.0.2.10 (backend pod)
   └─ HAProxy veth → host routing → backend pod veth
   └─ (may cross nodes if backend is on a different node)

7. Backend pod processes request, sends response
   └─ Response follows reverse path:
   └─ Backend pod → HAProxy pod → ALB → Client
```

## Complete Packet Journey (Instance/NodePort Mode)

```
1. Client → ALB (same as above)

2. ALB selects target: Node IP:30080 (NodePort)
   └─ Src: ALB ENI IP
   └─ Dst: 10.0.1.5:30080 (node IP)

3. Packet arrives at node
   └─ kube-proxy iptables rule matches dst port 30080
   └─ DNAT: 10.0.1.5:30080 → 10.0.1.50:80 (HAProxy pod)
   └─ SNAT: source IP becomes node IP (client IP lost without proxy protocol)

4. HAProxy pod receives request
   └─ Same as IP mode from here on
   └─ But X-Forwarded-For may show node IP instead of client IP
```

## Headers at Each Layer

| Header | Set By | Contains |
|--------|--------|----------|
| `X-Forwarded-For` | ALB | Original client IP |
| `X-Forwarded-Proto` | ALB | Original protocol (https) |
| `X-Forwarded-Port` | ALB | Original port (443) |
| `X-Real-IP` | HAProxy (optional) | Client IP from X-Forwarded-For |
| `Host` | Client | Target hostname (app.example.com) |

> **Important:** In instance mode with SNAT, the source IP seen by HAProxy is the node's IP, not the client's. Use `X-Forwarded-For` from ALB to get the real client IP. In IP mode, you can also use proxy protocol for true source preservation.

## Health Checks at Each Layer

```
ALB → HAProxy health check
  │   GET /healthz on HAProxy pods
  │   Unhealthy = removed from target group
  │
HAProxy → Backend health check
  │   Configurable per backend (TCP connect, HTTP GET)
  │   Unhealthy = removed from backend pool
  │
Kubernetes → Pod readiness probe
      kubelet checks readinessProbe
      Not ready = removed from Endpoints (HAProxy won't route to it)
```

| Layer | Who Checks | What's Checked | Failure Action |
|-------|-----------|----------------|----------------|
| ALB → HAProxy | ALB target group | HAProxy `/healthz` | Remove HAProxy pod from rotation |
| HAProxy → Backend | HAProxy | Backend pod health | Stop routing to that pod |
| kubelet → Pod | kubelet | readinessProbe | Remove from Endpoints/EndpointSlices |

## Where Things Break (and What You See)

| Symptom | Layer | Likely Cause |
|---------|-------|-------------|
| 502 Bad Gateway | ALB | HAProxy pod is down or not responding to health checks |
| 503 Service Unavailable | HAProxy | Backend has no healthy endpoints |
| 504 Gateway Timeout | ALB or HAProxy | Backend pod is too slow to respond |
| Connection refused | HAProxy → Backend | Pod crashed, service port mismatch |
| SSL handshake error | Client → ALB | Certificate mismatch or expired ACM cert |
| 404 Not Found | HAProxy | No Ingress rule matches the host/path |
| Intermittent 502 | ALB | HAProxy pod restarting (rolling update, OOM) |

## Debugging Commands

```bash
# Check ALB target group health
aws elbv2 describe-target-health --target-group-arn <tg-arn>

# Check HAProxy pods are running
kubectl get pods -n <ingress-namespace> -l app=haproxy-ingress -o wide

# Check HAProxy logs for routing errors
kubectl logs -n <ingress-namespace> -l app=haproxy-ingress --tail=50

# Check Ingress rules
kubectl get ingress -A
kubectl describe ingress <name> -n <namespace>

# Check if backend service has endpoints
kubectl get endpoints <service-name> -n <namespace>

# Check backend pod readiness
kubectl get pods -n <namespace> -l app=<backend-app> -o wide

# Trace request path (from inside cluster)
kubectl run curl --image=curlimages/curl --rm -it -- \
  curl -v -H "Host: app.example.com" http://<haproxy-pod-ip>/api

# Check HAProxy config (inside HAProxy pod)
kubectl exec -n <ingress-namespace> <haproxy-pod> -- cat /etc/haproxy/haproxy.cfg

# Check ALB access logs (if enabled)
# Logs are in S3: s3://<bucket>/AWSLogs/<account-id>/elasticloadbalancing/<region>/
```

## Architecture Variants

### Variant 1: ALB (IP mode) → HAProxy → Pods

```
Client → ALB → HAProxy pod IP → Backend pod IP
```
- Most efficient — fewest hops
- Source IP preserved via X-Forwarded-For
- Requires VPC CNI with routable pod IPs

### Variant 2: ALB (Instance mode) → NodePort → HAProxy → Pods

```
Client → ALB → Node:NodePort → kube-proxy → HAProxy → Backend pod
```
- Extra hop through kube-proxy
- SNAT loses source IP (use X-Forwarded-For)
- Works with any CNI

### Variant 3: NLB → HAProxy (DaemonSet with hostPort) → Pods

```
Client → NLB → Node:hostPort → HAProxy → Backend pod
```
- Layer 4 (preserves source IP with proxy protocol)
- HAProxy runs on every node via DaemonSet
- No kube-proxy hop

### Variant 4: ALB Ingress Controller (No HAProxy)

```
Client → ALB → Backend pod IP directly
```
- AWS ALB Ingress Controller manages ALB rules directly
- No HAProxy — ALB routes to pods via target groups
- Simpler but less configurable than HAProxy

## Security Groups Chain

```
SG: alb-sg
  Inbound:  0.0.0.0/0 :443 (internet)
  Outbound: node-sg :32080 (to nodes) or pod-sg :8080 (IP mode)
       │
       ▼
SG: node-sg (EKS node security group)
  Inbound:  alb-sg :32080 (from ALB, instance mode)
  Inbound:  node-sg (any) (pod-to-pod, inter-node)
  Inbound:  cluster-sg :443 (API server → kubelet)
  Outbound: 0.0.0.0/0 (internet via NAT)
       │
       ▼
With target-type: ip, ALB talks directly to pods:
  Inbound:  alb-sg :8080 (must allow on pod/node SG)
```

> **Key rule:** The node security group must allow inbound from the ALB security group on the target port. With IP mode, this applies to the pod's port directly.

## Egress Traffic Flow (Pod → External)

```
App Pod (10.0.1.20)
  │
  │ HTTP request to external API
  ▼
Pod eth0 → veth → host namespace
  │
  ▼
Linux routing: default via 10.0.1.1 dev eth0 (VPC gateway)
  │
  ▼
Node eth0 (primary ENI) → VPC Route Table
  │
  │ 0.0.0.0/0 → NAT Gateway
  ▼
NAT Gateway (Elastic IP: 52.x.x.x) → Internet Gateway → External API
```

- All pods on the same node share the same outbound path
- NAT Gateway has a bandwidth limit (~45 Gbps) and connection limit
- Cross-AZ egress incurs data transfer charges ($0.01/GB each direction)

## Connection State Through the Stack

Each layer creates a **separate TCP connection** — there's no single end-to-end connection:

```
Client ←── Connection 1 ──→ ALB        (HTTPS, kept alive)
ALB    ←── Connection 2 ──→ HAProxy    (HTTP, may be pooled)
HAProxy ←── Connection 3 ──→ Backend   (HTTP, may be pooled)
```

Each segment has independent TCP window sizes, congestion control, timeout settings, and retry logic.

## Latency Budget by Hop

| Hop | Typical Latency | Notes |
|-----|----------------|-------|
| Client → ALB | Varies (internet) | Geography, ISP dependent |
| ALB processing | 1–5 ms | Rule evaluation, TLS termination |
| ALB → Node (same AZ) | < 1 ms | VPC internal |
| ALB → Node (cross AZ) | 1–2 ms | Cross-AZ hop |
| kube-proxy DNAT | < 0.1 ms | iptables rule lookup |
| Node → Pod (veth) | < 0.05 ms | Memory copy via veth pair |
| HAProxy processing | 0.1–1 ms | ACL matching, backend selection |
| HAProxy → Backend (same node) | < 0.1 ms | veth + routing |
| HAProxy → Backend (cross node) | 0.5–2 ms | VPC routing |

**Total internal (ALB → response): typically 5–15 ms** depending on cross-AZ hops.

## Bandwidth Limits Per Layer

| Component | Limit | Notes |
|-----------|-------|-------|
| ALB | ~100 Gbps (auto-scales) | Soft limit, scales with traffic |
| EC2 Instance (e.g., m5.xlarge) | 1.25 Gbps baseline, 10 Gbps burst | Shared with EBS on small instances |
| NAT Gateway | ~45 Gbps | Single NAT GW can be a bottleneck |
| VPC peering / TGW | 50 Gbps per AZ | For cross-VPC traffic |
| Pod | No inherent limit | Shares node bandwidth with all pods |

## Timeout Chain Alignment

```
Client → ALB idle_timeout (60s) → HAProxy timeout server (120s) → Backend processing
```

**Critical rule:** ALB `idle_timeout` must be **less than** HAProxy `timeout http-keep-alive`. Otherwise you get a keep-alive race condition → 502.

```
# Recommended alignment:
ALB idle_timeout:              60s
HAProxy timeout http-keep-alive: 65s   (> ALB idle)
HAProxy timeout server:          120s  (enough for slow backends)
HAProxy timeout connect:         5s    (TCP handshake to backend)
HAProxy timeout client:          60s   (≥ ALB idle)
```

```bash
# Set ALB idle timeout
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn <arn> \
  --attributes Key=idle_timeout.timeout_seconds,Value=60
```

## Errors During Rolling Deployments

| Problem | Cause | Fix |
|---------|-------|-----|
| 502 during scale-down | Old pod receives SIGTERM, closes listener, ALB still routes to it | `preStop: sleep 15` to give ALB time to deregister |
| 503 during scale-up | New pod not ready yet, old pod terminated, no servers available | `maxSurge: 1`, fast readinessProbe |
| 502 keep-alive race | Pod closes keep-alive connections on shutdown, ALB sends request on closed connection | HAProxy `timeout http-keep-alive` > ALB `idle_timeout` |

Graceful shutdown pod spec:

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 15"]
    readinessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 2
```

## Cross-AZ Traffic Considerations

| Scenario | Cost | Fix |
|----------|------|-----|
| ALB in AZ-a sends to Node in AZ-b (NodePort) | $0.01/GB each direction | Use `target-type: ip` (ALB controller is AZ-aware) |
| kube-proxy forwards to pod on different node | Cross-AZ data transfer | `externalTrafficPolicy: Local` |
| HAProxy routes to backend pod in different AZ | Cross-AZ data transfer | Topology-aware routing (K8s 1.23+) |

**Optimizations:**
- `externalTrafficPolicy: Local` → ALB only sends to nodes with a local HAProxy pod
- Topology-aware routing → prefer same-AZ backends
- `target-type: ip` → ALB goes directly to pod, controller is AZ-aware

## Source IP Preservation

| Mode | Client IP Visible At | How |
|------|---------------------|-----|
| ALB + NodePort | Via header | `X-Forwarded-For` (ALB adds it) |
| ALB + target-type: ip | Via header | `X-Forwarded-For` |
| NLB (TCP mode) | Direct | Proxy Protocol v2 |
| externalTrafficPolicy: Local | Direct | Node doesn't SNAT |
| externalTrafficPolicy: Cluster | Lost | Source = node IP (use X-Forwarded-For) |
