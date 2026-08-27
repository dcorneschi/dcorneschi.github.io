# Cross-AZ Traffic in EKS — Patterns, Costs, and Optimization

All the ways traffic crosses availability zones in an EKS cluster — pod-to-pod, service routing, load balancer forwarding, storage replication — how much it costs, and how to minimize it.

Note: For ALB target modes and source IP preservation, see the EKS traffic flow article. This article focuses specifically on cross-AZ patterns and cost optimization.

## Why Cross-AZ Traffic Matters

AWS charges for data transfer between availability zones:

```
┌────────────────────────────────────────────────────────────────┐
│  Cross-AZ Data Transfer Pricing (as of 2024):                  │
│                                                                │
│  $0.01/GB in EACH direction = $0.02/GB round-trip              │
│                                                                │
│  Example: 10 TB/month cross-AZ traffic                         │
│  = 10,000 GB × $0.02 = $200/month                              │
│                                                                │
│  For a busy cluster doing 100 TB/month:                        │
│  = $2,000/month just in cross-AZ data transfer                 │
│                                                                │
│  Same-AZ traffic: FREE                                         │
└────────────────────────────────────────────────────────────────┘
```

## All Cross-AZ Traffic Sources in EKS

```
┌────────────────────────────────────────────────────────────────┐
│  Traffic Source                   │ Crosses AZ?  │ Avoidable?  │
│───────────────────────────────────│──────────────│─────────────│
│  Pod → Pod (different node/AZ)    │ Yes          │ Partially   │
│  Service → Pod (kube-proxy DNAT)  │ Yes          │ Yes         │
│  ALB → Node (NodePort mode)       │ Often        │ Yes         │
│  ALB → Pod (IP mode)              │ Rarely       │ Yes         │
│  NLB → Node                       │ Often        │ Yes         │
│  Pod → External (NAT GW)          │ Maybe        │ Yes         │
│  Pod → EBS (must be same AZ)      │ Never        │ N/A         │
│  Pod → EFS                        │ Yes (always) │ No          │
│  CoreDNS query                    │ Maybe        │ Yes         │
│  Kubelet → API server (X-ENI)     │ Maybe        │ No          │
│  Metrics/Logs → collectors        │ Maybe        │ Partially   │
└────────────────────────────────────────────────────────────────┘
```

## Pattern 1: Service Routing (kube-proxy)

By default, kube-proxy load-balances to ALL pods backing a Service, regardless of zone:

```
┌─────────────────────────────────────────────────────────────────┐
│  Default: externalTrafficPolicy: Cluster                        │
│                                                                 │
│  AZ-a                          AZ-b                             │
│  ┌──────────────┐             ┌──────────────┐                  │
│  │ Client Pod   │             │              │                  │
│  │              │─── Service ─┼──────────────┼──▶ Backend Pod   │
│  │ (calls svc)  │  ClusterIP  │              │   (cross-AZ!)    │
│  └──────────────┘             └──────────────┘                  │
│                                                                 │
│  kube-proxy randomly picks a backend — may be in any AZ         │
│  $0.01/GB each way                                              │
└─────────────────────────────────────────────────────────────────┘
```

### Fix: internalTrafficPolicy: Local

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  internalTrafficPolicy: Local    # Only route to pods on same node
  selector:
    app: my-app
  ports:
  - port: 80
```

| Policy | Behavior | Cross-AZ | Risk |
|--------|----------|:--------:|------|
| `Cluster` (default) | Route to any pod on any node | Yes | None |
| `Local` | Route only to pods on the same node | No | Connection fails if no local pod |

**Warning**: `internalTrafficPolicy: Local` drops traffic if no backend pod exists on the requesting node. Only use when your app has pods on every node (DaemonSet pattern) or combined with topology spread.

### Fix: Topology-Aware Routing (Topology Aware Hints)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    service.kubernetes.io/topology-mode: Auto    # K8s 1.27+
spec:
  selector:
    app: my-app
  ports:
  - port: 80
```

With topology-aware routing:
- EndpointSlice controller adds zone hints to endpoints
- kube-proxy prefers same-zone backends
- Falls back to cross-zone if same-zone pods are overloaded or unavailable

```bash
# Check if hints are active:
kubectl get endpointslices -l kubernetes.io/service-name=my-service -o yaml | grep -A2 hints

# Hints look like:
# endpoints:
# - addresses: [10.0.1.5]
#   zone: us-east-1a
#   hints:
#     forZones:
#     - name: us-east-1a
```

**Requirements for topology hints to activate:**
- At least 3 endpoints per zone (for even distribution)
- Zones must be reasonably balanced (< 2x imbalance)
- If imbalanced, the controller disables hints (falls back to random)

## Pattern 2: Load Balancer → Nodes

### ALB with target-type: instance (NodePort)

```
┌─────────────────────────────────────────────────────────────────┐
│  ALB → NodePort (default)                                       │
│                                                                 │
│  ALB (AZ-a listener)                                            │
│       │                                                         │
│       ▼                                                         │
│  Node in AZ-a (NodePort 30080)                                  │
│       │                                                         │
│       ▼ kube-proxy DNAT                                         │
│  Pod in AZ-b  ← CROSS-AZ!                                       │
│                                                                 │
│  Two hops: ALB → Node → Pod (may cross zones twice)             │
└─────────────────────────────────────────────────────────────────┘
```

### Fix: ALB with target-type: ip

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/target-type: ip    # Direct to pod IP
spec:
  ...
```

```
ALB (AZ-a listener) → Pod in AZ-a (direct, same-AZ)
ALB (AZ-b listener) → Pod in AZ-b (direct, same-AZ)
```

The ALB controller registers pod IPs directly as targets. ALB is AZ-aware and prefers sending traffic to targets in the same AZ as the listener.

### Fix: externalTrafficPolicy: Local (for NLB/NodePort)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local    # Only forward to local pods
  ports:
  - port: 80
```

With `Local`:
- NLB health checks detect which nodes have backend pods
- NLB only sends traffic to nodes WITH local pods
- No cross-node/cross-AZ forwarding by kube-proxy

**Tradeoff**: Traffic distribution may be uneven if pods aren't evenly spread.

## Pattern 3: Pod-to-Pod Communication

Direct pod-to-pod traffic crosses AZs when pods are on nodes in different zones:

```
Pod A (10.0.1.5, AZ-a) → Pod B (10.0.2.8, AZ-b)
= $0.01/GB each way (VPC native routing, no overlay)
```

### Fix: TopologySpreadConstraints

Ensure communicating pods are co-located in the same AZ:

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-app
```

### Fix: Pod Affinity (Co-locate Related Services)

```yaml
# Frontend pods prefer to be in the same AZ as backend pods:
spec:
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: backend
          topologyKey: topology.kubernetes.io/zone
```

## Pattern 4: NAT Gateway

If your pods access the internet and your NAT Gateway is in a single AZ:

```
Pod in AZ-b → NAT GW in AZ-a → Internet
= Cross-AZ traffic for ALL egress from AZ-b pods!
```

### Fix: NAT Gateway Per AZ

Deploy a NAT Gateway in each AZ with route tables pointing to the local one:

```
AZ-a route table: 0.0.0.0/0 → nat-gw-a
AZ-b route table: 0.0.0.0/0 → nat-gw-b
AZ-c route table: 0.0.0.0/0 → nat-gw-c
```

Cost: $0.045/hour per NAT GW × 3 = $0.135/hour ($97/month)
But saves cross-AZ transfer costs for all egress traffic.

## Pattern 5: CoreDNS Queries

CoreDNS runs as a Deployment (typically 2 replicas). Pods on nodes without a local CoreDNS replica send DNS queries cross-AZ:

```
Pod (AZ-c) → CoreDNS pod (AZ-a) = cross-AZ for every DNS query
```

### Fix: NodeLocal DNSCache

```bash
# Deploy NodeLocal DNSCache (DaemonSet on every node):
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
```

NodeLocal DNSCache:
- Runs on every node (link-local IP 169.254.20.10)
- Caches DNS responses locally
- Only cache misses go to CoreDNS (potentially cross-AZ)
- Dramatically reduces cross-AZ DNS traffic

## Pattern 6: Monitoring and Logging

Centralized collectors (Prometheus, Datadog Agent, Fluentd) pull metrics/logs from all nodes:

```
Prometheus (AZ-a) scrapes:
  Node in AZ-a → same-AZ (free)
  Node in AZ-b → cross-AZ ($0.01/GB)
  Node in AZ-c → cross-AZ ($0.01/GB)
```

### Fix: Per-AZ Prometheus + Thanos/Cortex

Run a Prometheus instance per AZ, aggregate with Thanos or Cortex for a unified view.

### Fix: Datadog Agent (Already Per-Node)

The Datadog Agent runs as a DaemonSet — metrics collection is local to each node. Cross-AZ traffic only occurs when agents send to Datadog intake (external, not billed as cross-AZ).

## Measuring Cross-AZ Traffic

### VPC Flow Logs

```bash
# Enable flow logs with AZ information:
aws ec2 create-flow-log \
  --resource-type VPC \
  --resource-id vpc-xxx \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc-flow-logs

# Query for cross-AZ patterns:
# Compare source/destination AZ fields in flow logs
```

### Cost Explorer

```bash
# Filter by "Data Transfer" cost category:
# Service: EC2, Usage Type: DataTransfer-Regional-Bytes
# This shows inter-AZ transfer costs

aws ce get-cost-and-usage \
  --time-period Start=2024-03-01,End=2024-03-31 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --filter '{"Dimensions":{"Key":"USAGE_TYPE","Values":["USE1-DataTransfer-Regional-Bytes"]}}'
```

### Kubernetes Metrics

```bash
# If using Cilium, Hubble shows per-flow traffic with zone labels
# If using Datadog NPM, it tracks cross-AZ flows

# Manual: label nodes with AZ and check pod placement
kubectl get pods -o wide | awk '{print $7}' | sort | uniq -c
# Shows pod distribution across nodes (and therefore AZs)
```

## Optimization Decision Tree

```
Is the traffic cross-AZ?
│
├── Service → Pod routing?
│   ├── Internal: use internalTrafficPolicy: Local or topology-aware hints
│   └── External (LB): use target-type: ip or externalTrafficPolicy: Local
│
├── Pod → Pod direct?
│   ├── Co-locate with podAffinity (same zone)
│   └── Spread evenly with topologySpreadConstraints
│
├── DNS queries?
│   └── Deploy NodeLocal DNSCache
│
├── NAT Gateway egress?
│   └── NAT GW per AZ
│
├── Monitoring/scraping?
│   └── Per-AZ collectors or DaemonSet-based collection
│
└── Storage (EFS)?
    └── Cannot avoid — EFS is always cross-AZ by design
```

## Cost Estimation

```bash
# Estimate monthly cross-AZ cost:
# 1. Check inter-AZ bytes in VPC Flow Logs or Cost Explorer
# 2. Multiply by $0.02/GB (both directions)

# Quick estimate from Datadog/Prometheus:
# sum(rate(node_network_transmit_bytes_total[24h])) by (instance)
# Cross-reference with node AZ labels
# Traffic between nodes in different AZs = your cross-AZ cost driver
```

### Example Cost Breakdown

| Source | Monthly Traffic | Cost ($0.02/GB) |
|--------|:--------------:|:---------------:|
| Service routing (kube-proxy random) | 5 TB | $100 |
| ALB → NodePort (cross-AZ) | 2 TB | $40 |
| CoreDNS queries | 100 GB | $2 |
| Prometheus scraping | 500 GB | $10 |
| NAT GW (single AZ) | 3 TB | $60 |
| **Total** | **~10.6 TB** | **$212/month** |

After optimization (topology hints, IP mode, NodeLocal DNS, per-AZ NAT):

| Source | Monthly Traffic | Cost |
|--------|:--------------:|:----:|
| Service routing (zone-aware) | 500 GB | $10 |
| ALB → Pod IP (same-AZ) | 200 GB | $4 |
| CoreDNS (NodeLocal cache) | 10 GB | $0.20 |
| Prometheus (per-AZ) | 50 GB | $1 |
| NAT GW (per-AZ) | 0 | $0 |
| **Total** | **~760 GB** | **$15/month** |

## Gotchas

| Gotcha | Explanation |
|--------|-------------|
| Topology hints disabled if zones are imbalanced | If AZ-a has 10 pods and AZ-b has 2, hints turn off automatically |
| `internalTrafficPolicy: Local` drops traffic | If no local backend exists, connection fails (no fallback) |
| Single-AZ failures with zone-affinity | If all pods are in one AZ and it goes down, you lose everything |
| EFS is always cross-AZ | EFS mount targets are per-AZ but data replication crosses AZs |
| ALB cross-zone load balancing | Enabled by default — can be disabled but causes uneven distribution |
| Topology key must be consistent | Use `topology.kubernetes.io/zone` (standard label) |

## Quick Reference

```bash
# Key optimizations:
# 1. Service routing: topology-aware hints or internalTrafficPolicy: Local
# 2. Load balancer: target-type: ip + externalTrafficPolicy: Local
# 3. DNS: NodeLocal DNSCache DaemonSet
# 4. NAT: one NAT GW per AZ
# 5. Pod placement: topologySpreadConstraints + podAffinity

# Check zone labels on nodes:
kubectl get nodes -o custom-columns=NAME:.metadata.name,ZONE:.metadata.labels.topology\\.kubernetes\\.io/zone

# Check pod distribution across zones:
kubectl get pods -l app=my-app -o json | jq -r '.items[].spec.nodeName' | \
  xargs -I{} kubectl get node {} -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}{"\n"}' | sort | uniq -c

# Enable topology-aware hints:
kubectl annotate svc my-service service.kubernetes.io/topology-mode=Auto

# Check if hints are active:
kubectl get endpointslices -l kubernetes.io/service-name=my-service -o yaml | grep -c "forZones"

# Pricing: $0.01/GB per direction = $0.02/GB round-trip
# Same-AZ: free
# EFS: always cross-AZ (can't avoid)
# EBS: always same-AZ (zone-locked)
```
