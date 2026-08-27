# What Happens When You Create a Kubernetes Service

The internal flow from Service object creation to traffic routing — how the endpoint controller populates EndpointSlices, kube-proxy programs iptables/IPVS rules, and CoreDNS creates DNS records.

## High-Level Flow

```
kubectl apply -f service.yaml
        │
        ▼
┌───────────────┐     ┌───────────────────┐     ┌───────────────┐     ┌──────────────┐
│   API Server  │────▶│ Endpoint Slice    │────▶│  kube-proxy   │────▶│  iptables/   │
│  (allocates   │     │ Controller        │     │  (per node)   │     │  IPVS rules  │
│   ClusterIP)  │     │ (finds matching   │     │  (programs    │     │  (traffic    │
│               │     │  pods)            │     │   rules)      │     │   routing)   │
└───────────────┘     └───────────────────┘     └───────────────┘     └──────────────┘
                                                                              │
┌───────────────┐                                                             ▼
│   CoreDNS     │◀──── watches Service object ────────────────────── Traffic reaches pods
│  (DNS record) │
└───────────────┘
```

## Step 1: API Server — ClusterIP Allocation

When you create a Service, the API server:

1. Validates the spec
2. Allocates a ClusterIP from the service CIDR range (if `type: ClusterIP`)
3. Persists to etcd

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: default
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
```

After creation:

```yaml
spec:
  clusterIP: 10.100.42.15    # Allocated from service CIDR
  clusterIPs:
  - 10.100.42.15
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: my-app
```

### ClusterIP Allocation

```
┌──────────────────────────────────────────────────┐
│  Service CIDR: 10.100.0.0/16                     │
│                                                  │
│  API server maintains a bitmap of allocated IPs  │
│  Stored in etcd as a RangeAllocation object      │
│                                                  │
│  10.100.0.1  — kubernetes (API server service)   │
│  10.100.0.10 — kube-dns (CoreDNS)                │
│  10.100.42.15 — my-app (just allocated)          │
│  ...                                             │
│                                                  │
│  Static allocation: spec.clusterIP set by user   │
│  Dynamic allocation: API server picks next free  │
└──────────────────────────────────────────────────┘
```

```bash
# See the service CIDR:
kubectl cluster-info dump | grep -m1 service-cluster-ip-range

# See allocated ClusterIP:
kubectl get svc my-app -o jsonpath='{.spec.clusterIP}'
```

**Important**: ClusterIP is a virtual IP — it doesn't exist on any network interface. It only exists in iptables/IPVS rules as a DNAT destination.

## Step 2: EndpointSlice Controller — Finding Backends

The EndpointSlice controller watches Services and Pods. When a Service with a `selector` is created:

```
┌────────────────────────────────────────────────────────────────┐
│  EndpointSlice Controller Logic                                │
│                                                                │
│  1. Watch: new Service "my-app" with selector {app: my-app}    │
│  2. List all Pods matching selector in same namespace          │
│  3. Filter: only pods that are Ready (readiness probe passes)  │
│  4. Create EndpointSlice with pod IPs and ports                │
│  5. Continuously update as pods come and go                    │
└────────────────────────────────────────────────────────────────┘
```

### EndpointSlice Object

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-app-abc12
  namespace: default
  labels:
    kubernetes.io/service-name: my-app
  ownerReferences:
  - apiVersion: v1
    kind: Service
    name: my-app
addressType: IPv4
ports:
- name: ""
  port: 8080        # targetPort from Service
  protocol: TCP
endpoints:
- addresses:
  - 10.244.1.5      # Pod IP
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: node-1
  targetRef:
    kind: Pod
    name: my-app-pod-a
    namespace: default
- addresses:
  - 10.244.2.8
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: node-2
  targetRef:
    kind: Pod
    name: my-app-pod-b
    namespace: default
```

### When Endpoints Change

The EndpointSlice controller updates the slice when:
- A new pod matching the selector becomes Ready
- A pod's readiness probe fails (removed from `ready` endpoints)
- A pod is deleted (removed entirely, or marked `terminating: true`)
- A pod's IP changes (unlikely but possible)

```bash
# See EndpointSlices for a service:
kubectl get endpointslices -l kubernetes.io/service-name=my-app

# Detailed view:
kubectl get endpointslices -l kubernetes.io/service-name=my-app -o yaml

# Legacy Endpoints object (still created for backward compatibility):
kubectl get endpoints my-app
```

### EndpointSlice vs Endpoints

| Feature | Endpoints (legacy) | EndpointSlices (current) |
|---------|-------------------|------------------------|
| Object size | One object per Service (can get huge) | Split into chunks of 100 endpoints |
| Scalability | Poor for large services (1000+ pods) | Good (smaller update payloads) |
| Topology hints | No | Yes (zone-aware routing) |
| Dual-stack | No | Yes (separate IPv4/IPv6 slices) |
| Used since | Kubernetes 1.0 | Default since 1.21 |

## Step 3: kube-proxy — Programming Network Rules

kube-proxy runs on every node as a DaemonSet. It watches Services and EndpointSlices via the API server and programs node-level forwarding rules:

```
┌───────────────────────────────────────────────────────────────┐
│  kube-proxy (on each node)                                    │
│                                                               │
│  Watches:                                                     │
│    - Services (ClusterIP, ports, sessionAffinity)             │
│    - EndpointSlices (backend pod IPs and ports)               │
│                                                               │
│  On change:                                                   │
│    - Compute diff between current rules and desired rules     │
│    - Update iptables/IPVS/nftables on the node                │
│                                                               │
│  Result:                                                      │
│    - Traffic to ClusterIP:port is DNAT'd to a pod IP:port     │
│    - Load balancing across healthy endpoints                  │
└───────────────────────────────────────────────────────────────┘
```

### iptables Mode (Default)

kube-proxy creates iptables chains for each Service:

```
Packet to 10.100.42.15:80 (ClusterIP)
    │
    ▼
KUBE-SERVICES chain
    │ match: -d 10.100.42.15/32 -p tcp --dport 80
    ▼
KUBE-SVC-XXXXX chain (Service-specific)
    │
    ├── 33% probability → KUBE-SEP-AAAA (Pod A: 10.244.1.5:8080)
    ├── 50% probability → KUBE-SEP-BBBB (Pod B: 10.244.2.8:8080)
    └── remainder       → KUBE-SEP-CCCC (Pod C: 10.244.3.2:8080)

KUBE-SEP-AAAA:
    DNAT to 10.244.1.5:8080
```

```bash
# View iptables rules for a service (on a node):
sudo iptables-save | grep my-app
sudo iptables -t nat -L KUBE-SERVICES | grep 10.100.42.15
```

### IPVS Mode

```bash
# View IPVS rules:
sudo ipvsadm -Ln | grep -A5 10.100.42.15

# Output:
# TCP  10.100.42.15:80 rr
#   -> 10.244.1.5:8080      Masq    1      0          0
#   -> 10.244.2.8:8080      Masq    1      0          0
#   -> 10.244.3.2:8080      Masq    1      0          0
```

| Feature | iptables | IPVS | nftables |
|---------|----------|------|----------|
| Load balancing | Random (probability chains) | Round-robin, least-conn, source-hash | Random |
| Performance at scale | Degrades with many services (O(n) rules) | Better (hash table lookups) | Better than iptables |
| Rule update | Full chain rewrite | Incremental add/remove | Incremental |
| Default since | Always (until nftables GA) | Must opt-in | Kubernetes 1.31+ (optional) |

### nftables Mode (Kubernetes 1.31+)

```bash
# View nftables rules:
sudo nft list ruleset | grep my-app
```

## Step 4: CoreDNS — DNS Record Creation

CoreDNS watches Service objects and automatically creates DNS records:

```
Service: my-app.default.svc.cluster.local
    │
    ├── A record → 10.100.42.15 (ClusterIP)
    │
    └── SRV record → _http._tcp.my-app.default.svc.cluster.local
                     → 0 100 80 my-app.default.svc.cluster.local
```

### DNS Records by Service Type

| Service Type | DNS A Record | Points To |
|-------------|-------------|-----------|
| ClusterIP | `<name>.<ns>.svc.cluster.local` | ClusterIP (virtual IP) |
| Headless (clusterIP: None) | `<name>.<ns>.svc.cluster.local` | All pod IPs directly |
| ExternalName | `<name>.<ns>.svc.cluster.local` | CNAME to external domain |
| NodePort | Same as ClusterIP | ClusterIP |
| LoadBalancer | Same as ClusterIP | ClusterIP (+ external DNS via cloud) |

### Headless Services — No ClusterIP

```yaml
spec:
  clusterIP: None    # Headless
  selector:
    app: my-db
```

For headless services:
- No ClusterIP is allocated
- No kube-proxy rules are created
- DNS returns all pod IPs directly (A records for each pod)
- Client does its own load balancing

```bash
# Headless DNS lookup returns individual pod IPs:
nslookup my-db.default.svc.cluster.local
# Address: 10.244.1.5
# Address: 10.244.2.8
# Address: 10.244.3.2
```

## Complete Timeline

```
Time ──────────────────────────────────────────────────────────────────▶

kubectl         API Server       EndpointSlice Ctrl   kube-proxy    CoreDNS
   │               │                   │                │             │
   │ create svc ──▶│                   │                │             │
   │               │ allocate          │                │             │
   │               │ ClusterIP         │                │             │
   │               │ persist to etcd   │                │             │
   │               │                   │                │             │
   │ ◀── 201 ───── │                   │                │             │
   │               │                   │                │             │
   │               │── watch event ───▶│                │             │
   │               │                   │ find matching  │             │
   │               │                   │ pods           │             │
   │               │                   │ create         │             │
   │               │                   │ EndpointSlice  │             │
   │               │                   │                │             │
   │               │── watch event ────┼───────────────▶│             │
   │               │  (Svc + EPS)      │                │ program     │
   │               │                   │                │ iptables    │
   │               │                   │                │             │
   │               │── watch event ────┼────────────────┼────────────▶│
   │               │  (Service)        │                │             │ create
   │               │                   │                │             │ DNS record
   │               │                   │                │             │
   │               │                   │                │             │
   │  ~1-3 seconds after creation, service is fully routable          │
```

## Service Types — What Changes

### NodePort

Adds a port on every node in addition to ClusterIP:

```
┌──────────┐
│  Client  │
│ (external)│
└──────────┘
     │
     │ NodeIP:30080
     ▼
┌──────────┐     ┌──────────────┐     ┌──────────┐
│  Node    │────▶│  kube-proxy  │────▶│  Pod     │
│(any node)│     │  DNAT rules  │     │(any node)│
└──────────┘     └──────────────┘     └──────────┘
```

kube-proxy adds rules on ALL nodes to accept traffic on the NodePort and forward to backends.

```bash
# NodePort range: 30000-32767 (default)
# See allocated NodePort:
kubectl get svc my-app -o jsonpath='{.spec.ports[0].nodePort}'
```

### LoadBalancer

Adds cloud load balancer on top of NodePort:

```
┌──────────┐     ┌──────────────┐     ┌──────────┐     ┌─────────┐
│  Client  │────▶│  Cloud LB    │────▶│   Node   │────▶│   Pod   │
│          │     │  (AWS ALB/NLB│     │ NodePort │     │         │
│          │     │   GCP LB)    │     │          │     │         │
└──────────┘     └──────────────┘     └──────────┘     └─────────┘
```

The cloud controller manager watches for `type: LoadBalancer` services and provisions the external LB via cloud APIs.

### ExternalName

No ClusterIP, no endpoints, no kube-proxy rules. Just a DNS CNAME:

```yaml
spec:
  type: ExternalName
  externalName: db.example.com   # CoreDNS returns CNAME
```

## Session Affinity

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800   # 3 hours default
```

kube-proxy programs different rules:
- **iptables**: Uses `recent` module to stick client IP to a backend
- **IPVS**: Uses source-hash scheduling algorithm

## Traffic Policies

### internalTrafficPolicy

```yaml
spec:
  internalTrafficPolicy: Local  # Only route to pods on same node
```

kube-proxy only includes local endpoints in the rules for this Service on each node. If no local endpoints exist, traffic is dropped.

### externalTrafficPolicy

```yaml
spec:
  externalTrafficPolicy: Local  # Preserve client IP, no cross-node hop
```

For NodePort/LoadBalancer services:
- `Cluster` (default): Forward to any pod on any node (may cross nodes)
- `Local`: Only forward to pods on the node receiving traffic (preserves source IP)

## Services Without Selectors

Services without `selector` don't get automatic EndpointSlices. You manage endpoints manually:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  ports:
  - port: 5432
  # No selector — no automatic endpoint discovery
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-db-manual
  labels:
    kubernetes.io/service-name: external-db
addressType: IPv4
ports:
- port: 5432
endpoints:
- addresses:
  - 192.168.1.100    # External database IP
```

Use case: Pointing a Kubernetes Service at external infrastructure.

## Debugging Services

```bash
# Check Service has ClusterIP:
kubectl get svc my-app

# Check EndpointSlices have backends:
kubectl get endpointslices -l kubernetes.io/service-name=my-app

# Check pods match the selector:
kubectl get pods -l app=my-app -o wide

# Check pods are Ready (only Ready pods are in endpoints):
kubectl get pods -l app=my-app -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Test DNS resolution:
kubectl run dns-test --image=busybox:1.36 --rm -it -- nslookup my-app.default.svc.cluster.local

# Test connectivity to ClusterIP:
kubectl run curl-test --image=curlimages/curl --rm -it -- curl -s http://10.100.42.15:80

# Check iptables rules on a node (for the service):
sudo iptables-save | grep <ClusterIP>

# Check kube-proxy is healthy:
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=20
```

### Common Issues

| Symptom | Cause | Check |
|---------|-------|-------|
| Service has no endpoints | No pods match selector, or pods not Ready | `kubectl get endpoints <svc>` |
| DNS doesn't resolve | CoreDNS pods not running | `kubectl get pods -n kube-system -l k8s-app=kube-dns` |
| Connection refused on ClusterIP | kube-proxy rules missing or pods not listening | `iptables-save \| grep <ClusterIP>` |
| Timeout to ClusterIP | Network policy blocking, or no healthy backends | Check NetworkPolicies and pod readiness |
| Intermittent failures | Some backends unhealthy | Check individual pod IPs directly |

## Quick Reference

```bash
# Service creation triggers:
# 1. API server: allocates ClusterIP, persists to etcd
# 2. EndpointSlice controller: finds matching pods, creates EndpointSlice
# 3. kube-proxy (all nodes): programs iptables/IPVS rules
# 4. CoreDNS: creates DNS A record

# Key objects:
kubectl get svc <name>
kubectl get endpointslices -l kubernetes.io/service-name=<name>
kubectl get endpoints <name>  # legacy

# ClusterIP is virtual — exists only in iptables/IPVS rules
# You cannot ping a ClusterIP (no ICMP rules, only TCP/UDP/SCTP)

# Headless service (clusterIP: None):
# - No ClusterIP allocated
# - No kube-proxy rules
# - DNS returns pod IPs directly

# Time from Service creation to routable: ~1-3 seconds
# (EndpointSlice creation + kube-proxy rule sync)
```
