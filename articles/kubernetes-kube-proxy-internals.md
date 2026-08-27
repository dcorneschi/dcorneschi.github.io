# How kube-proxy Works — iptables vs IPVS vs nftables

The internal mechanics of kube-proxy — how it translates Service objects into packet forwarding rules, the three proxy modes, performance at scale, and when to switch modes.

Note: For the Service creation flow (ClusterIP allocation, EndpointSlice controller, DNS), see the Service creation internals article. This article focuses on kube-proxy's packet-level implementation.

## What kube-proxy Does

kube-proxy runs as a DaemonSet on every node. It watches Services and EndpointSlices, then programs node-level rules to intercept traffic destined for ClusterIPs and forward it to backend pod IPs:

```
┌─────────────────────────────────────────────────────────────────┐
│  kube-proxy (per node)                                          │
│                                                                 │
│  Input:                                                         │
│    - Service objects (ClusterIP, ports, sessionAffinity)        │
│    - EndpointSlice objects (pod IPs, ports, readiness)          │
│                                                                 │
│  Output:                                                        │
│    - Packet forwarding rules (iptables, IPVS, or nftables)      │
│    - DNAT from ClusterIP:port → PodIP:targetPort                │
│    - Load balancing across healthy backends                     │
│                                                                 │
│  ClusterIP doesn't exist on any interface — it only exists      │
│  as a DNAT rule in the forwarding path                          │
└─────────────────────────────────────────────────────────────────┘
```

## Mode Comparison

| Feature | iptables (default) | IPVS | nftables |
|---------|:------------------:|:----:|:--------:|
| Default since | Always | Opt-in | 1.31 (opt-in) |
| Rule structure | Chains of sequential rules | Hash table + kernel module | nft rules (sets) |
| Load balancing | Random (probability) | Round-robin, least-conn, source-hash, and more | Random (like iptables) |
| Rule update | Full chain rewrite | Incremental add/remove | Incremental |
| Performance with 1000 services | Degrades (O(n) traversal) | Consistent (O(1) lookup) | Better than iptables |
| Performance with 10,000 services | Poor (latency visible) | Good | Good |
| Connection tracking | Kernel conntrack | Kernel conntrack | Kernel conntrack |
| Session affinity | `recent` module (client IP) | Built-in (multiple algorithms) | Similar to iptables |
| Metrics | Basic (via iptables counters) | Rich (`ipvsadm` stats per backend) | Basic |
| Kernel modules needed | None (iptables built-in) | `ip_vs`, `ip_vs_rr`, `ip_vs_wrr`, etc. | `nf_tables` |

## iptables Mode (Default)

### How It Works

kube-proxy creates iptables chains in the `nat` table that DNAT traffic from ClusterIP to a random backend pod:

```
┌────────────────────────────────────────────────────────────────────┐
│  Packet path (iptables mode):                                      │
│                                                                    │
│  Packet dst: 10.100.42.15:80 (ClusterIP)                          │
│      │                                                             │
│      ▼                                                             │
│  PREROUTING chain → KUBE-SERVICES chain                            │
│      │                                                             │
│      │ Match: -d 10.100.42.15/32 -p tcp --dport 80                 │
│      ▼                                                             │
│  KUBE-SVC-XXXXX chain (Service-level)                              │
│      │                                                             │
│      ├── 33% → KUBE-SEP-AAAA (endpoint A)                         │
│      ├── 50% → KUBE-SEP-BBBB (endpoint B)                         │
│      └── 100% → KUBE-SEP-CCCC (endpoint C)                        │
│                                                                    │
│  KUBE-SEP-AAAA:                                                    │
│      -j DNAT --to-destination 10.244.1.5:8080                      │
│                                                                    │
│  Result: packet rewritten from 10.100.42.15:80 → 10.244.1.5:8080  │
│  conntrack entry created for return traffic                        │
└────────────────────────────────────────────────────────────────────┘
```

### Chain Structure

```bash
# Top-level hook:
-A PREROUTING -j KUBE-SERVICES
-A OUTPUT -j KUBE-SERVICES       # For traffic originating on this node

# Per-Service chain (one KUBE-SVC-* per Service):
-A KUBE-SERVICES -d 10.100.42.15/32 -p tcp --dport 80 -j KUBE-SVC-XXXXX

# Load balancing (probability-based):
-A KUBE-SVC-XXXXX -m statistic --mode random --probability 0.33333 -j KUBE-SEP-AAAA
-A KUBE-SVC-XXXXX -m statistic --mode random --probability 0.50000 -j KUBE-SEP-BBBB
-A KUBE-SVC-XXXXX -j KUBE-SEP-CCCC

# Per-Endpoint chain (one KUBE-SEP-* per pod backend):
-A KUBE-SEP-AAAA -p tcp -j DNAT --to-destination 10.244.1.5:8080
-A KUBE-SEP-BBBB -p tcp -j DNAT --to-destination 10.244.2.8:8080
-A KUBE-SEP-CCCC -p tcp -j DNAT --to-destination 10.244.3.2:8080

# Masquerade (SNAT for hairpin/external traffic):
-A KUBE-POSTROUTING -j MASQUERADE
```

### Viewing iptables Rules

```bash
# All kube-proxy rules:
sudo iptables-save | grep -c "KUBE-"    # Count rules
sudo iptables-save | grep "KUBE-SVC"    # Service chains
sudo iptables-save | grep "KUBE-SEP"    # Endpoint chains

# Rules for a specific ClusterIP:
sudo iptables-save | grep "10.100.42.15"

# Rule counts (scale indicator):
sudo iptables-save | wc -l              # Total rules in all tables
```

### Why iptables Degrades at Scale

```
┌────────────────────────────────────────────────────────────────┐
│  iptables Performance Problem                                  │
│                                                                │
│  Each packet traverses rules SEQUENTIALLY (O(n)):              │
│                                                                │
│  1000 Services × 3 endpoints each = ~6000 KUBE-SEP rules      │
│  + 1000 KUBE-SVC rules + KUBE-SERVICES entries                 │
│  = ~8000+ rules traversed per packet (worst case)              │
│                                                                │
│  Rule update is ATOMIC (full replace):                         │
│  - kube-proxy generates entire iptables-restore input          │
│  - Replaces ALL chains atomically                              │
│  - With 10,000+ rules: takes 100ms+ to apply                  │
│  - During apply: brief packet loss possible                    │
│                                                                │
│  At ~5,000 services: noticeable CPU overhead on nodes          │
│  At ~10,000 services: significant latency + CPU spikes         │
└────────────────────────────────────────────────────────────────┘
```

## IPVS Mode

IPVS (IP Virtual Server) is a kernel-level Layer 4 load balancer. It uses hash tables for O(1) rule lookup regardless of service count.

### How It Works

```
┌────────────────────────────────────────────────────────────────────┐
│  Packet path (IPVS mode):                                          │
│                                                                    │
│  Packet dst: 10.100.42.15:80 (ClusterIP)                          │
│      │                                                             │
│      ▼                                                             │
│  IPVS kernel module intercepts (hash table lookup, O(1))           │
│      │                                                             │
│      │  Virtual Server: 10.100.42.15:80 (scheduling: rr)           │
│      │    → Real Server: 10.244.1.5:8080  weight=1                 │
│      │    → Real Server: 10.244.2.8:8080  weight=1                 │
│      │    → Real Server: 10.244.3.2:8080  weight=1                 │
│      │                                                             │
│      ▼                                                             │
│  DNAT to selected backend (round-robin by default)                 │
│  conntrack entry created                                           │
└────────────────────────────────────────────────────────────────────┘
```

### Enabling IPVS Mode

```yaml
# kube-proxy ConfigMap:
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |
    mode: "ipvs"
    ipvs:
      scheduler: "rr"          # Round-robin (default)
      # Other options: lc (least connections), dh (dest hash),
      #               sh (source hash), sed (shortest expected delay),
      #               nq (never queue)
```

```bash
# Apply and restart kube-proxy:
kubectl edit configmap kube-proxy -n kube-system
# Change mode: "" to mode: "ipvs"
kubectl rollout restart daemonset kube-proxy -n kube-system
```

### IPVS Scheduling Algorithms

| Algorithm | Flag | Behavior | Use Case |
|-----------|------|----------|----------|
| Round Robin | `rr` | Rotate through backends equally | Default, stateless services |
| Least Connections | `lc` | Send to backend with fewest active connections | Long-lived connections |
| Destination Hash | `dh` | Hash destination IP → consistent backend | Caching layers |
| Source Hash | `sh` | Hash source IP → same backend | Session affinity without cookies |
| Shortest Expected Delay | `sed` | Weighted least connections | Mixed-weight backends |
| Never Queue | `nq` | Like sed but never queues | Low-latency requirements |

### Viewing IPVS Rules

```bash
# List all virtual servers:
sudo ipvsadm -Ln
# TCP  10.100.42.15:80 rr
#   -> 10.244.1.5:8080            Masq    1      0          0
#   -> 10.244.2.8:8080            Masq    1      0          0
#   -> 10.244.3.2:8080            Masq    1      0          0

# With stats:
sudo ipvsadm -Ln --stats
# Shows: Conns, InPkts, OutPkts, InBytes, OutBytes per backend

# With rates:
sudo ipvsadm -Ln --rate
# Shows: CPS, InPPS, OutPPS, InBPS, OutBPS per backend

# Count virtual servers:
sudo ipvsadm -Ln | grep -c "^TCP\|^UDP"
```

### IPVS Dummy Interface

IPVS needs the ClusterIPs bound to a network interface for the kernel to accept traffic. kube-proxy creates a dummy interface:

```bash
# The dummy interface:
ip addr show kube-ipvs0
# 4: kube-ipvs0: <BROADCAST,NOARP> mtu 1500
#     inet 10.100.42.15/32 scope global kube-ipvs0
#     inet 10.100.0.1/32 scope global kube-ipvs0
#     inet 10.100.0.10/32 scope global kube-ipvs0
#     ... (one IP per Service ClusterIP)
```

### IPVS + iptables (Still Needs Some iptables)

IPVS mode still uses iptables for:
- Masquerading (SNAT for pods accessing services)
- NodePort handling
- External traffic policy enforcement

```bash
# IPVS mode still has some iptables rules:
sudo iptables-save | grep -c "KUBE"
# Much fewer than pure iptables mode (typically <100 vs thousands)
```

## nftables Mode (Kubernetes 1.31+)

nftables is the modern successor to iptables in the Linux kernel. kube-proxy nftables mode uses nft sets for efficient rule lookup.

### How It Works

```
┌────────────────────────────────────────────────────────────────────┐
│  nftables mode:                                                    │
│                                                                    │
│  - Uses nft sets (hash-based) instead of sequential chains         │
│  - Incremental updates (no full chain rewrite)                     │
│  - Better than iptables at scale                                   │
│  - Same random load balancing as iptables mode                     │
│  - Native nftables, doesn't translate to iptables                  │
└────────────────────────────────────────────────────────────────────┘
```

### Enabling nftables Mode

```yaml
# kube-proxy ConfigMap:
data:
  config.conf: |
    mode: "nftables"
```

### Viewing nftables Rules

```bash
# List kube-proxy nftables rules:
sudo nft list ruleset | grep -A 5 "kube-proxy"

# Count rules:
sudo nft list ruleset | wc -l
```

### nftables Requirements

- Linux kernel 5.13+ (for full nft features kube-proxy uses)
- nft userspace tools installed
- Kubernetes 1.31+ (beta)
- Cannot run alongside iptables mode (mutually exclusive)

## Performance Comparison at Scale

```
┌────────────────────────────────────────────────────────────────────┐
│  Rule update time (adding/removing one endpoint):                   │
│                                                                    │
│  Services    iptables         IPVS           nftables              │
│  ──────────  ──────────       ──────────     ──────────            │
│  100         ~5ms             ~1ms           ~2ms                  │
│  1,000       ~50ms            ~1ms           ~5ms                  │
│  5,000       ~250ms           ~1ms           ~10ms                 │
│  10,000      ~500ms+          ~1ms           ~15ms                 │
│  20,000      ~2s+             ~1ms           ~20ms                 │
│                                                                    │
│  First-packet latency (per new connection):                        │
│                                                                    │
│  Services    iptables         IPVS           nftables              │
│  ──────────  ──────────       ──────────     ──────────            │
│  100         ~negligible      ~negligible    ~negligible           │
│  1,000       ~tens of μs      ~negligible    ~negligible           │
│  10,000      ~hundreds of μs  ~negligible    ~negligible           │
│                                                                    │
│  iptables: O(n) rule traversal per packet                          │
│  IPVS: O(1) hash lookup                                           │
│  nftables: O(1) set lookup                                         │
└────────────────────────────────────────────────────────────────────┘
```

## When to Switch Modes

| Scenario | Recommended Mode |
|----------|-----------------|
| < 1,000 Services | iptables (default, simple, well-tested) |
| 1,000 - 5,000 Services | Consider IPVS (if seeing latency) |
| > 5,000 Services | IPVS or nftables (iptables will struggle) |
| Need advanced LB (least-conn, source hash) | IPVS (only mode with LB algorithms) |
| Want modern kernel path, fewer legacy deps | nftables (1.31+) |
| Replace kube-proxy entirely | Cilium kube-proxy replacement (eBPF) |

## Cilium — Replacing kube-proxy with eBPF

Cilium can replace kube-proxy entirely using eBPF programs:

```bash
# Install Cilium without kube-proxy:
helm install cilium cilium/cilium \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<api-server-ip> \
  --set k8sServicePort=443
```

```
┌────────────────────────────────────────────────────────────────┐
│  eBPF (Cilium) advantages over kube-proxy:                     │
│                                                                │
│  - O(1) lookup (BPF hash maps)                                │
│  - No conntrack table overhead (BPF manages its own state)    │
│  - Direct packet rewrite (no traversing chains)               │
│  - Socket-level load balancing (skips network stack entirely) │
│  - DSR (Direct Server Return) mode                            │
│  - Maglev consistent hashing                                  │
│  - No dummy interface needed                                  │
│  - Lower latency, higher throughput                           │
└────────────────────────────────────────────────────────────────┘
```

## NodePort and LoadBalancer Handling

### NodePort (All Modes)

```
External traffic → Node:30080 (any node)
    │
    ▼
kube-proxy intercepts (DNAT to a backend pod)
    │
    ├── externalTrafficPolicy: Cluster → any pod on any node
    └── externalTrafficPolicy: Local  → only pods on THIS node
```

### LoadBalancer

The cloud load balancer sends to NodePorts. kube-proxy on each node forwards to backends. The rules are the same as for ClusterIP + NodePort combined.

## Session Affinity

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

| Mode | Implementation |
|------|---------------|
| iptables | `recent` module tracks client IP → same backend for timeout duration |
| IPVS | Native session persistence (`--persistent <timeout>`) |
| nftables | Similar to iptables (meter/set based) |

## Conntrack and Connection Draining

All modes use the kernel's conntrack table to track established connections:

```bash
# View conntrack entries:
sudo conntrack -L | grep 10.100.42.15
# Shows: src, dst, sport, dport, state, timeout

# Count entries:
sudo conntrack -C

# Check max (if full, new connections drop):
cat /proc/sys/net/netfilter/nf_conntrack_max
cat /proc/sys/net/netfilter/nf_conntrack_count
```

When a backend is removed (pod deleted), existing connections continue until they close naturally or timeout. New connections go to remaining backends.

## kube-proxy Configuration

```bash
# View current kube-proxy config:
kubectl get configmap kube-proxy -n kube-system -o yaml

# Key settings:
# mode: "" (iptables), "ipvs", or "nftables"
# iptables.syncPeriod: 30s (how often rules are refreshed)
# ipvs.scheduler: "rr" (scheduling algorithm)
# ipvs.syncPeriod: 30s
# conntrack.maxPerCore: 32768
# conntrack.min: 131072
```

## Debugging kube-proxy

```bash
# Check kube-proxy pods are running:
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Check mode:
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=5 | grep "Using"
# "Using iptables proxier" / "Using ipvs proxier" / "Using nftables proxier"

# Check kube-proxy health:
kubectl exec -n kube-system <kube-proxy-pod> -- curl -s localhost:10256/healthz

# iptables mode — verify rules exist for a service:
sudo iptables-save | grep <ClusterIP>

# IPVS mode — verify virtual server:
sudo ipvsadm -Ln | grep <ClusterIP>

# Check for stale rules (after service deletion):
sudo iptables-save | grep "KUBE-SVC" | wc -l  # Should match service count

# kube-proxy metrics:
kubectl port-forward -n kube-system <kube-proxy-pod> 10249:10249
curl localhost:10249/metrics | grep kubeproxy
# kubeproxy_sync_proxy_rules_duration_seconds (how long rule sync takes)
# kubeproxy_sync_proxy_rules_iptables_total (total rules managed)
```

## Quick Reference

```bash
# Three modes: iptables (default), ipvs, nftables (1.31+)

# iptables: sequential chain traversal, O(n), degrades at scale
# IPVS: hash table lookup, O(1), advanced scheduling algorithms
# nftables: nft sets, O(1), modern kernel path, incremental updates

# Switch mode:
kubectl edit configmap kube-proxy -n kube-system  # Change mode field
kubectl rollout restart daemonset kube-proxy -n kube-system

# Check current mode:
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=5 | grep "Using"

# View rules:
sudo iptables-save | grep KUBE        # iptables mode
sudo ipvsadm -Ln                       # IPVS mode
sudo nft list ruleset | grep kube      # nftables mode

# Performance thresholds:
# < 1,000 services: iptables is fine
# 1,000-5,000: consider IPVS
# > 5,000: IPVS or nftables required
# > 10,000: Cilium eBPF recommended

# IPVS scheduling: rr, lc, dh, sh, sed, nq
# iptables LB: random (probability-based, no algorithm choice)

# Key metric:
# kubeproxy_sync_proxy_rules_duration_seconds
# If this is >1s, you have too many services for iptables mode
```
