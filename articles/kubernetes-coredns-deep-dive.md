# Kubernetes DNS Deep Dive — CoreDNS Architecture, Plugins, and Configuration

How CoreDNS works inside Kubernetes — plugin architecture, Corefile syntax, the kubernetes plugin internals, autopath, stub domains, forwarding, caching strategies, and advanced patterns beyond basic troubleshooting.

Note: For EKS-specific CoreDNS troubleshooting (debug logging, metrics, scaling, VPC endpoint issues), see the CoreDNS on EKS cheatsheet. This article covers the generic CoreDNS architecture and plugin system.

## CoreDNS Architecture

CoreDNS is a general-purpose DNS server written in Go, built on a plugin-based architecture. In Kubernetes, it runs as a Deployment and serves as the cluster DNS:

```
┌─────────────────────────────────────────────────────────────────────┐
│  CoreDNS Architecture                                               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────┐            │
│  │  Corefile (configuration)                           │            │
│  │                                                     │            │
│  │  .:53 {                                             │            │
│  │    errors                                           │            │
│  │    health                                           │            │
│  │    kubernetes cluster.local ...                     │            │
│  │    forward . /etc/resolv.conf                       │            │
│  │    cache 30                                         │            │
│  │    loop                                             │            │
│  │    reload                                           │            │
│  │    loadbalance                                      │            │
│  │  }                                                  │            │
│  └─────────────────────────────────────────────────────┘            │
│                                                                     │
│  DNS query arrives on port 53                                       │
│      │                                                              │
│      ▼                                                              │
│  Plugin Chain (executed in order listed in Corefile):               │
│      errors → health → kubernetes → forward → cache → ...           │
│                                                                     │
│  Each plugin either:                                                │
│    - Handles the query (returns a response) OR                      │
│    - Passes to the next plugin (fallthrough)                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## The Plugin Chain

CoreDNS processes DNS queries by passing them through a chain of plugins. Order in the Corefile determines execution order:

```
Query: "my-service.default.svc.cluster.local"
    │
    ▼ errors plugin → (logs errors, passes through)
    ▼ health plugin → (not a health query, passes through)
    ▼ kubernetes plugin → MATCH! Returns the ClusterIP
    ✓ Response sent to client

Query: "google.com"
    │
    ▼ errors plugin → (passes through)
    ▼ health plugin → (passes through)
    ▼ kubernetes plugin → not cluster.local, fallthrough
    ▼ forward plugin → forwards to upstream DNS
    ▼ cache plugin → caches the response
    ✓ Response sent to client
```

## Core Plugins Reference

### kubernetes — Cluster DNS Records

The `kubernetes` plugin is what makes CoreDNS work for Service/Pod discovery:

```
kubernetes cluster.local in-addr.arpa ip6.arpa {
    pods insecure          # Enable pod DNS records (A records for pod IPs)
    fallthrough in-addr.arpa ip6.arpa   # Pass reverse lookups to next plugin if not found
    ttl 30                 # Cache TTL for responses (seconds)
}
```

**What it resolves:**

| Query Pattern | Returns | Example |
|--------------|---------|---------|
| `<svc>.<ns>.svc.cluster.local` | Service ClusterIP (A record) | `nginx.default.svc.cluster.local → 10.100.0.50` |
| `<svc>.<ns>.svc.cluster.local` | SRV record (port info) | `_http._tcp.nginx.default.svc.cluster.local` |
| `<pod-ip-dashed>.<ns>.pod.cluster.local` | Pod IP (A record) | `10-244-1-5.default.pod.cluster.local → 10.244.1.5` |
| `<svc>.<ns>.svc.cluster.local` (headless) | All pod IPs | Returns multiple A records |
| `<hostname>.<svc>.<ns>.svc.cluster.local` | StatefulSet pod IP | `web-0.nginx.default.svc.cluster.local` |

**How it works internally:**

```
┌────────────────────────────────────────────────────────────────┐
│  kubernetes plugin internals:                                  │
│                                                                │
│  1. Watches API server for Services and EndpointSlices         │
│  2. Maintains in-memory cache of all Service/Endpoint data     │
│  3. On DNS query:                                              │
│     a. Parse the FQDN to extract service, namespace            │
│     b. Look up in the local cache                              │
│     c. Return ClusterIP (normal) or pod IPs (headless)         │
│  4. No live API calls per query — everything from cache        │
│                                                                │
│  pods insecure:                                                │
│    Enables A record for pods (10-244-1-5.ns.pod.cluster.local) │
│    "insecure" = no verification that pod actually exists       │
│    (returns the IP embedded in the name, no lookup)            │
│                                                                │
│  pods verified:                                                │
│    Same but verifies the pod exists in the API (slower)        │
└────────────────────────────────────────────────────────────────┘
```

### forward — Upstream DNS

Forwards queries that CoreDNS can't resolve locally:

```
# Forward everything not handled by kubernetes plugin:
forward . /etc/resolv.conf

# Forward to specific upstream servers:
forward . 8.8.8.8 8.8.4.4

# Forward to VPC DNS resolver:
forward . 169.254.169.253

# Forward with options:
forward . 8.8.8.8 8.8.4.4 {
    max_concurrent 1000      # Max concurrent queries to upstream
    policy round_robin       # Or: random, sequential
    health_check 5s          # Probe upstream every 5s
    expire 10s               # Drop stale connections after 10s
    tls_servername dns.google # TLS server name for DoT
}
```

### cache — Response Caching

```
cache 30 {
    success 9984 30     # Cache up to 9984 positive responses, 30s TTL
    denial 9984 5       # Cache up to 9984 NXDOMAIN responses, 5s TTL
    prefetch 10 10m 20% # Prefetch when entry hits 10 queries, within 10m of expiry, at 20% TTL remaining
}
```

Cache behavior:
- Caches responses from ALL plugins (kubernetes, forward, etc.)
- Respects the original TTL if it's lower than the configured max
- On cache hit: returns immediately without calling downstream plugins
- Prefetch: proactively refreshes popular entries before they expire

### errors — Error Logging

```
errors {
    consolidate 5m ".* i]]]" # Consolidate repeated errors
}
```

Logs all errors to stdout (visible in `kubectl logs`).

### health — Health Endpoint

```
health {
    lameduck 5s    # Stay healthy for 5s after receiving SIGTERM (graceful shutdown)
}
```

Exposes `http://:8080/health` — returns 200 if CoreDNS is functioning. Used by Kubernetes liveness probes.

### ready — Readiness Endpoint

```
ready
```

Exposes `http://:8181/ready` — returns 200 when all plugins are ready (kubernetes plugin has synced its cache). Used by readiness probes.

### loop — Loop Detection

```
loop
```

Detects forwarding loops (CoreDNS forwarding to itself). If detected, CoreDNS crashes intentionally to alert the operator.

### reload — Hot Reload

```
reload
```

Watches the Corefile ConfigMap for changes and reloads without restart (checks every 30s by default).

### loadbalance — Shuffle DNS Records

```
loadbalance round_robin
```

Randomizes the order of A records in responses. Provides basic client-side load balancing for headless services (clients typically use the first record).

### log — Query Logging

```
log
```

Logs every query to stdout. **Generates a lot of output — use only for debugging, then remove.**

```
# Example log output:
[INFO] 10.244.1.5:43210 - 12345 "A IN google.com.default.svc.cluster.local. udp 54 false 512" NXDOMAIN qr,aa,rd 147 0.000473s
[INFO] 10.244.1.5:43210 - 12346 "A IN google.com. udp 28 false 512" NOERROR qr,rd,ra 106 0.001711s
```

## Advanced Plugins

### autopath — Reduce Search Domain Queries

The biggest CoreDNS performance optimization for external DNS lookups:

```
autopath @kubernetes
```

**The Problem (without autopath):**

```
Pod queries: google.com (ndots:5, fewer than 5 dots)

Without autopath, CoreDNS receives 5 queries:
  1. google.com.default.svc.cluster.local → NXDOMAIN
  2. google.com.svc.cluster.local         → NXDOMAIN
  3. google.com.cluster.local             → NXDOMAIN
  4. google.com.ec2.internal              → NXDOMAIN
  5. google.com.                          → NOERROR (success)

= 4 wasted queries × every external DNS lookup × every pod
```

**With autopath:**

```
Pod queries: google.com

CoreDNS receives: google.com.default.svc.cluster.local
autopath plugin:
  1. Knows this pod is in namespace "default"
  2. Checks: is google.com a Service in "default"? No.
  3. Checks: is google.com a Service anywhere? No.
  4. Resolves google.com. directly (upstream)
  5. Returns the answer with a CNAME rewrite:
     google.com.default.svc.cluster.local → google.com. → 142.250.x.x

Client receives the answer on the FIRST query (no wasted NXDOMAIN queries)
```

**Impact**: Reduces DNS query volume by 60-80% for workloads calling external services.

**Requirement**: The `kubernetes` plugin must be listed BEFORE `autopath` in the Corefile (autopath needs `@kubernetes` as its source).

```
.:53 {
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    autopath @kubernetes    # AFTER kubernetes plugin
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

### hosts — Static DNS Entries

```
hosts {
    10.0.1.50 myservice.internal
    10.0.1.51 database.internal
    fallthrough
}
```

Returns static A records. `fallthrough` passes unmatched queries to the next plugin.

### rewrite — Query Rewriting

```
# Rewrite a service name:
rewrite name old-service.default.svc.cluster.local new-service.default.svc.cluster.local

# Rewrite based on pattern:
rewrite name regex (.*)\.old-domain\.com {1}.new-domain.com

# Rewrite AAAA to A (disable IPv6 lookups):
rewrite stop type AAAA A
```

### template — Synthetic DNS Records

```
template IN A internal.example.com {
    match "^(?P<host>[a-z]+)\.internal\.example\.com\.$"
    answer "{{ .Name }} 60 IN A 10.0.1.{{ .Group.host | hashIP }}"
    fallthrough
}
```

Generates DNS responses from templates — useful for dynamic internal DNS without a separate zone file.

## Stub Domains and Conditional Forwarding

Route specific domains to different DNS servers:

```
.:53 {
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}

# Conditional forward: corp.example.com to on-premises DNS
corp.example.com:53 {
    errors
    cache 30
    forward . 10.200.0.53 10.200.1.53
}

# Another domain to a different server:
partner.io:53 {
    errors
    cache 60
    forward . 172.16.0.53
}
```

Each server block (zone) is independent. CoreDNS selects the most specific matching zone for each query:
- `myapp.corp.example.com` → matches `corp.example.com:53` block
- `google.com` → matches `.:53` block (catch-all)

### Configuring Stub Domains via ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
    corp.internal:53 {
        errors
        cache 30
        forward . 10.0.1.100 10.0.1.101
    }
```

```bash
# Apply changes:
kubectl edit configmap coredns -n kube-system
# CoreDNS reloads automatically (reload plugin watches for changes)
```

## How Pods Get DNS Configuration

The kubelet configures each pod's `/etc/resolv.conf` based on the pod's `dnsPolicy`:

| dnsPolicy | resolv.conf contents | Use Case |
|-----------|---------------------|----------|
| `ClusterFirst` (default) | Points to CoreDNS ClusterIP + search domains | Standard pods |
| `Default` | Copies from the node's resolv.conf | Bypass CoreDNS (use node DNS) |
| `ClusterFirstWithHostNet` | Same as ClusterFirst (for hostNetwork pods) | System pods with hostNetwork |
| `None` | Empty (you must provide `dnsConfig`) | Custom DNS setup |

```yaml
# Custom DNS configuration:
spec:
  dnsPolicy: None
  dnsConfig:
    nameservers:
    - 10.0.0.53
    searches:
    - myns.svc.cluster.local
    - svc.cluster.local
    - cluster.local
    options:
    - name: ndots
      value: "3"
```

## ndots and Search Domains

```
# Default pod resolv.conf:
nameserver 10.100.0.10
search default.svc.cluster.local svc.cluster.local cluster.local ec2.internal
options ndots:5
```

`ndots:5` means: if the query name has fewer than 5 dots, try appending each search domain BEFORE querying the bare name.

| Query | Dots | Behavior (ndots:5) |
|-------|:----:|-------------------|
| `my-service` | 0 | Try all search domains first, then bare |
| `my-service.production` | 1 | Try all search domains first |
| `google.com` | 1 | Try all search domains first (4 wasted NXDOMAIN!) |
| `a.b.c.d.google.com` | 4 | Still tries search domains first |
| `a.b.c.d.e.google.com` | 5 | Query as-is (absolute, no search domains) |

**Optimization**: Lower ndots for pods that primarily call external services:

```yaml
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "2"    # Only append search domains for names with < 2 dots
```

Or use FQDNs with a trailing dot in your app: `google.com.` (trailing dot = absolute, skips search).

## CoreDNS Metrics (Prometheus)

CoreDNS exposes metrics on port 9153:

```
prometheus :9153
```

```bash
# Key metrics:
coredns_dns_requests_total          # Total queries received
coredns_dns_responses_total         # Responses sent (by rcode)
coredns_dns_request_duration_seconds # Query latency histogram
coredns_cache_hits_total            # Cache hits
coredns_cache_misses_total          # Cache misses
coredns_forward_requests_total      # Queries forwarded upstream
coredns_forward_responses_total     # Upstream responses
coredns_kubernetes_dns_programming_duration_seconds # Time to sync from API
coredns_panics_total                # CoreDNS panics (should be 0)
```

```promql
# Useful queries:
rate(coredns_dns_requests_total[5m])                    # Query rate
rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m]) # Error rate
histogram_quantile(0.99, rate(coredns_dns_request_duration_seconds_bucket[5m]))  # p99 latency
rate(coredns_cache_hits_total[5m]) / (rate(coredns_cache_hits_total[5m]) + rate(coredns_cache_misses_total[5m]))  # Cache hit ratio
```

## Scaling CoreDNS

### Horizontal Scaling (More Replicas)

```bash
# Scale manually:
kubectl scale deployment coredns -n kube-system --replicas=5

# Or use Cluster Proportional Autoscaler:
# Scales CoreDNS based on cluster size (nodes/cores)
```

### NodeLocal DNSCache

Runs a DNS cache on every node, reducing cross-node DNS traffic and CoreDNS load:

```
Pod → NodeLocal DNSCache (169.254.20.10, on same node)
         │
         ├── Cache hit → respond immediately (no network hop)
         └── Cache miss → forward to CoreDNS ClusterIP
```

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
```

Benefits:
- Eliminates conntrack race conditions (uses TCP to CoreDNS, UDP locally)
- Reduces cross-AZ DNS traffic
- Lower latency for cached entries
- Reduces CoreDNS pod load

## Debugging CoreDNS

```bash
# Check CoreDNS pods:
kubectl get pods -n kube-system -l k8s-app=kube-dns

# View logs:
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=30

# Enable query logging (temporary — high volume!):
kubectl edit configmap coredns -n kube-system
# Add "log" to the Corefile, then wait for reload

# Test resolution from a pod:
kubectl run dns-test --image=busybox:1.36 --rm -it -- nslookup kubernetes.default
kubectl run dns-test --image=busybox:1.36 --rm -it -- nslookup google.com

# Check CoreDNS config:
kubectl get configmap coredns -n kube-system -o yaml

# Check CoreDNS is the nameserver in pods:
kubectl exec <pod> -- cat /etc/resolv.conf

# Metrics:
kubectl port-forward -n kube-system deploy/coredns 9153:9153
curl localhost:9153/metrics | grep coredns_dns_requests_total
```

## Quick Reference

```bash
# CoreDNS = plugin-based DNS server, configured via Corefile (ConfigMap)
# Plugin chain: queries pass through plugins in Corefile order
# First plugin to handle the query wins

# Key plugins:
# kubernetes    — resolves Service/Pod names from cluster state
# forward       — forwards to upstream DNS
# cache         — caches responses (success + denial)
# autopath      — eliminates wasted search domain queries
# hosts         — static DNS entries
# rewrite       — query rewriting
# log           — query logging (debug only)

# Stub domains: separate server blocks for custom forwarding
# corp.example.com:53 { forward . 10.200.0.53 }

# ndots:5 = queries with < 5 dots try all search domains first
# Fix: lower ndots, use FQDN (trailing dot), or enable autopath

# Scaling: more replicas, NodeLocal DNSCache, or autopath

# Config location:
kubectl get configmap coredns -n kube-system -o yaml
# Changes auto-reload (reload plugin)

# Test: kubectl run dns-test --image=busybox:1.36 --rm -it -- nslookup <name>
# Logs: kubectl logs -n kube-system -l k8s-app=kube-dns
# Metrics: port 9153 (Prometheus)
```
