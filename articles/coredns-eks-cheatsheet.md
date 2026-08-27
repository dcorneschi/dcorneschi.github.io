# CoreDNS on EKS — Cheatsheet

Commands, one-liners, tips, and troubleshooting for CoreDNS running as the cluster DNS in Amazon EKS.

## CoreDNS Basics on EKS

CoreDNS runs as a Deployment in `kube-system` and provides in-cluster DNS resolution. EKS manages it as an add-on.

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide

# Check CoreDNS deployment
kubectl get deployment coredns -n kube-system

# Check CoreDNS service (cluster IP)
kubectl get svc kube-dns -n kube-system
```

Default cluster DNS IP: `10.100.0.10` (or `172.20.0.10` depending on service CIDR).

## DNS Record Syntax

### Service Records

```
<service-name>.<namespace>.svc.cluster.local
```

Examples:

```bash
# Service "web" in namespace "default"
web.default.svc.cluster.local

# Service "vpa-webhook" in namespace "kube-system"
vpa-webhook.kube-system.svc.cluster.local
```

### Pod Records

```
<pod-ip-with-dashes>.<namespace>.pod.cluster.local
```

Replace dots in the pod IP with hyphens:

```bash
# Pod with IP 10.36.0.2 in namespace "kube-system"
10-36-0-2.kube-system.pod.cluster.local

# Pod with IP 10.244.1.15 in namespace "default"
10-244-1-15.default.pod.cluster.local
```

```bash
# Resolve a pod by its hyphened IP
kubectl exec dnsutils -- nslookup 10-36-0-2.kube-system.pod.cluster.local
```

## DNS Resolution Flow

```
Pod → /etc/resolv.conf → kube-dns Service (ClusterIP)
                              │
                              ▼
                     CoreDNS Pods (port 53)
                              │
                 ┌────────────┼────────────┐
                 ▼            ▼            ▼
           Cluster DNS    Node DNS     External DNS
           (services,     (VPC DNS     (forward to
            pods)          resolver)    upstream)
```

Pods get this `/etc/resolv.conf`:

```
nameserver 10.100.0.10
search default.svc.cluster.local svc.cluster.local cluster.local ec2.internal
options ndots:5
```

### How Search Domains Work (ndots:5)

Any name with fewer than 5 dots is tried with each search domain before the final bare query:

**Service in same namespace** (`my-service`, 0 dots):
```
1. my-service.default.svc.cluster.local → Found!
```

**Service in different namespace** (`my-service.production`, 1 dot):
```
1. my-service.production.default.svc.cluster.local → NXDOMAIN
2. my-service.production.svc.cluster.local → Found!
```

**External domain** (`google.com`, 1 dot):
```
1. google.com.default.svc.cluster.local → NXDOMAIN
2. google.com.svc.cluster.local → NXDOMAIN
3. google.com.cluster.local → NXDOMAIN
4. google.com.ec2.internal → NXDOMAIN
5. google.com. → Found! (forwarded upstream)
```

That's 4 wasted queries for external names. Use FQDN (trailing dot) or reduce ndots.

### kube-proxy Traffic Forwarding

kube-proxy intercepts traffic to the kube-dns ClusterIP and load-balances to CoreDNS pod IPs.

**iptables mode:**

```bash
# View iptables rules for kube-dns (on a node)
sudo iptables-save | grep kube-dns
sudo iptables -t nat -L KUBE-SERVICES | grep kube-dns
```

Example rules:
```
# Traffic to kube-dns service is sent to service chain
-A KUBE-SERVICES -d 10.100.0.10/32 -p udp --dport 53 -j KUBE-SVC-TCOU7JCQXEZGVUNU

# Service chain load balances (50/50 for 2 pods)
-A KUBE-SVC-TCOU7JCQXEZGVUNU -m statistic --mode random --probability 0.50 -j KUBE-SEP-AAAA
-A KUBE-SVC-TCOU7JCQXEZGVUNU -j KUBE-SEP-BBBB

# DNAT to actual CoreDNS pod IPs
-A KUBE-SEP-AAAA -p udp -j DNAT --to-destination 192.168.1.10:53
-A KUBE-SEP-BBBB -p udp -j DNAT --to-destination 192.168.2.20:53
```

**IPVS mode:**

```bash
# Check IPVS rules (on a node)
sudo ipvsadm -Ln | grep -A 5 10.100.0.10
```

```
UDP  10.100.0.10:53 rr
  -> 192.168.1.10:53              Masq    1      0          0
  -> 192.168.2.20:53              Masq    1      0          0
```

## View and Edit Corefile

```bash
# View current Corefile
kubectl get configmap coredns -n kube-system -o yaml

# Edit Corefile
kubectl edit configmap coredns -n kube-system

# View just the Corefile content
kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}'
```

### Default EKS Corefile

```
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
```

## DNS Testing and Debugging

### Quick DNS Test from a Pod

```bash
# Run a DNS test pod
kubectl run dnsutils --image=gcr.io/kubernetes-e2e-test-images/dnsutils:1.3 --restart=Never -- sleep infinity

# Or use busybox
kubectl run dns-test --image=busybox:1.36 --restart=Never -- sleep 3600

# Test resolution
kubectl exec dnsutils -- nslookup kubernetes.default
kubectl exec dnsutils -- nslookup google.com

# Clean up
kubectl delete pod dnsutils dns-test
```

### Resolve Service Names

```bash
# Service in same namespace
kubectl exec <pod> -- nslookup <service-name>

# Service in different namespace
kubectl exec <pod> -- nslookup <service-name>.<namespace>.svc.cluster.local

# Full FQDN resolution
kubectl exec <pod> -- nslookup <service-name>.<namespace>.svc.cluster.local

# Headless service (returns all pod IPs)
kubectl exec <pod> -- nslookup <headless-service>.<namespace>.svc.cluster.local
```

### Dig for Detailed DNS Info

```bash
# A record
kubectl exec dnsutils -- dig <service>.default.svc.cluster.local A +short

# SRV record (shows port)
kubectl exec dnsutils -- dig <service>.default.svc.cluster.local SRV +short

# Reverse lookup
kubectl exec dnsutils -- dig -x 10.0.1.15 +short

# Query specific DNS server
kubectl exec dnsutils -- dig @10.100.0.10 kubernetes.default.svc.cluster.local

# External resolution through CoreDNS
kubectl exec dnsutils -- dig @10.100.0.10 google.com +short

# Check response time
kubectl exec dnsutils -- dig google.com +stats | grep "Query time"
```

### Check resolv.conf in Pods

```bash
kubectl exec <pod> -- cat /etc/resolv.conf
```

## CoreDNS Logs

```bash
# View CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100

# Follow logs
kubectl logs -n kube-system -l k8s-app=kube-dns -f

# Logs from specific pod
kubectl logs -n kube-system coredns-<hash> --tail=200

# Filter for errors
kubectl logs -n kube-system -l k8s-app=kube-dns | grep -i "error\|fail\|timeout"
```

### Enable Debug Logging

Add `log` plugin to the Corefile:

```bash
kubectl edit configmap coredns -n kube-system
```

```
.:53 {
    log        # <-- Add this line for all query logging
    errors
    health {
        lameduck 5s
    }
    ...
}
```

CoreDNS reloads automatically (the `reload` plugin handles it). Check logs after:

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns -f
```

**Remove `log` after debugging** — it generates a lot of output and affects performance.

## CoreDNS Metrics

CoreDNS exposes Prometheus metrics on port 9153.

```bash
# Port-forward to CoreDNS pod
kubectl port-forward -n kube-system deploy/coredns 9153:9153

# Scrape metrics
curl http://localhost:9153/metrics

# Key metrics to check
curl -s http://localhost:9153/metrics | grep -E "coredns_dns_requests_total|coredns_dns_responses_total|coredns_forward_requests_total|coredns_cache_hits_total"
```

### Important Metrics

| Metric | Description |
|--------|-------------|
| `coredns_dns_requests_total` | Total DNS queries received |
| `coredns_dns_responses_total` | Total DNS responses sent (by rcode) |
| `coredns_forward_requests_total` | Queries forwarded upstream |
| `coredns_forward_responses_total` | Responses from upstream |
| `coredns_cache_hits_total` | Cache hit count (by type) |
| `coredns_cache_misses_total` | Cache miss count |
| `coredns_dns_request_duration_seconds` | Query latency histogram |
| `coredns_panics_total` | CoreDNS panics (should be 0) |

### Prometheus Queries

```promql
# Request rate
rate(coredns_dns_requests_total[5m])

# Error rate (NXDOMAIN, SERVFAIL)
rate(coredns_dns_responses_total{rcode=~"NXDOMAIN|SERVFAIL"}[5m])

# Cache hit ratio
rate(coredns_cache_hits_total[5m]) / (rate(coredns_cache_hits_total[5m]) + rate(coredns_cache_misses_total[5m]))

# Forward latency (p99)
histogram_quantile(0.99, rate(coredns_forward_request_duration_seconds_bucket[5m]))

# Requests per CoreDNS pod
sum by (pod) (rate(coredns_dns_requests_total[5m]))
```

## Custom DNS Entries

### Forward Specific Domains to Custom DNS

```
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

# Forward company domain to internal DNS
corp.example.com:53 {
    errors
    cache 30
    forward . 10.0.1.100 10.0.1.101
}

# Forward another domain
internal.mycompany.io:53 {
    errors
    cache 30
    forward . 172.16.0.53
}
```

### Add Static DNS Entries (hosts plugin)

```
.:53 {
    errors
    hosts {
        10.0.1.50 myservice.internal
        10.0.1.51 database.internal
        fallthrough
    }
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
```

### Rewrite DNS Queries

```
.:53 {
    rewrite name old-service.default.svc.cluster.local new-service.default.svc.cluster.local
    ...
}
```

## Scaling CoreDNS

### Check Current Replicas

```bash
kubectl get deployment coredns -n kube-system
kubectl get hpa -n kube-system  # Check if HPA exists
```

### Manual Scaling

```bash
# Scale up
kubectl scale deployment coredns -n kube-system --replicas=5

# Note: EKS add-on may revert manual scaling. Use the add-on config instead.
```

### Enable Cluster Proportional Autoscaler

For large clusters, use proportional autoscaler (scales CoreDNS based on cluster size):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dns-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: dns-autoscaler
  template:
    metadata:
      labels:
        k8s-app: dns-autoscaler
    spec:
      containers:
      - name: autoscaler
        image: registry.k8s.io/cpa/cluster-proportional-autoscaler:v1.8.9
        command:
        - /cluster-proportional-autoscaler
        - --namespace=kube-system
        - --configmap=dns-autoscaler
        - --target=deployment/coredns
        - --default-params={"linear":{"coresPerReplica":256,"nodesPerReplica":16,"min":2,"max":10,"preventSinglePointFailure":true}}
        - --logtostderr=true
        - --v=2
```

### Sizing Guidelines

| Cluster Size | Recommended Replicas | Notes |
|-------------|---------------------|-------|
| < 50 nodes | 2 (default) | Sufficient for most workloads |
| 50-100 nodes | 3-4 | Monitor latency |
| 100-500 nodes | 5-8 | Use proportional autoscaler |
| 500+ nodes | 8-15+ | NodeLocal DNSCache recommended |

## NodeLocal DNSCache

Runs a DNS cache on every node to reduce latency and CoreDNS load:

```bash
# Check if NodeLocal DNSCache is running
kubectl get daemonset node-local-dns -n kube-system

# Install (if not present)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
```

With NodeLocal DNSCache:
- Pods query the local cache (link-local IP `169.254.20.10`)
- Cache misses go to CoreDNS
- Reduces cross-node DNS traffic
- Eliminates conntrack race conditions

## EKS Add-on Management

```bash
# Check CoreDNS add-on status
aws eks describe-addon --cluster-name <cluster> --addon-name coredns

# List available versions
aws eks describe-addon-versions --addon-name coredns --kubernetes-version 1.30

# Update CoreDNS add-on
aws eks update-addon --cluster-name <cluster> --addon-name coredns \
  --addon-version v1.11.1-eksbuild.8 \
  --resolve-conflicts OVERWRITE

# Update with custom configuration
aws eks update-addon --cluster-name <cluster> --addon-name coredns \
  --configuration-values '{"replicaCount": 3}'

# Delete add-on (manage CoreDNS yourself)
aws eks delete-addon --cluster-name <cluster> --addon-name coredns
```

## Troubleshooting

### DNS Resolution Fails Completely

```bash
# 1. Check CoreDNS pods are running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. Check kube-dns service has endpoints
kubectl get endpoints kube-dns -n kube-system

# 3. Test from a pod
kubectl run dns-test --image=busybox:1.36 --restart=Never --rm -it -- nslookup kubernetes.default

# 4. Check CoreDNS logs for errors
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# 5. Check if pod can reach DNS service IP
kubectl run net-test --image=busybox:1.36 --restart=Never --rm -it -- nc -zvu 10.100.0.10 53
```

### Verify kube-proxy Health

kube-proxy creates the iptables/IPVS rules that route traffic to CoreDNS pods. If kube-proxy can't reach the API server, DNS routing breaks.

```bash
# Check kube-proxy logs for timeout or auth errors
kubectl logs -n kube-system --selector 'k8s-app=kube-proxy' | grep -i "error\|timeout\|403"

# Check kube-proxy pods are running
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

Look for connection timeouts to the control plane or `403 Unauthorized` errors — both indicate kube-proxy can't sync endpoint rules.

### CoreDNS CPU Starvation

The EKS CoreDNS add-on sets a 170 MiB memory limit but **no CPU limit**. If the node's CPU is saturated, CoreDNS gets starved and DNS queries time out.

```bash
# Check CoreDNS pod CPU and memory usage
kubectl top pods -n kube-system -l k8s-app=kube-dns

# Check node CPU pressure
kubectl top nodes
```

If CoreDNS CPU is consistently high or the node is at 100%, either:
- Move CoreDNS pods to less loaded nodes via taints/tolerations
- Add explicit CPU requests to guarantee scheduling priority
- Scale up replicas to distribute load

### Intermittent DNS Failures

Common causes on EKS:
- **Conntrack table full** — UDP DNS uses conntrack, table fills under high load
- **Race condition with DNAT** — multiple DNS packets from same source port
- **CoreDNS overloaded** — too few replicas for cluster size
- **VPC DNS resolver throttling** — 1024 packets/sec per ENI limit hit

```bash
# Check conntrack table on node
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# Check for conntrack drops
dmesg | grep conntrack
cat /proc/net/stat/nf_conntrack | awk '{print $2}' # drops column
```

Fixes:
- Scale up CoreDNS replicas
- Deploy NodeLocal DNSCache
- Increase `nf_conntrack_max` on nodes

### VPC DNS Resolver Throttling (1024 pps per ENI)

The Amazon VPC DNS resolver accepts a maximum of 1024 packets per second per elastic network interface. If multiple CoreDNS pods land on the same node, external domain queries forwarded upstream can exceed this limit, causing intermittent SERVFAIL or timeouts.

Use PodAntiAffinity to spread CoreDNS pods across separate nodes:

```yaml
# Add to CoreDNS deployment spec.template.spec
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - podAffinityTerm:
      labelSelector:
        matchExpressions:
        - key: k8s-app
          operator: In
          values:
          - kube-dns
      topologyKey: kubernetes.io/hostname
    weight: 100
```

This ensures each CoreDNS pod uses a different node's ENI, effectively multiplying the available VPC DNS bandwidth.

### Slow DNS Resolution (High Latency)

```bash
# Measure DNS latency from a pod
kubectl exec dnsutils -- time nslookup google.com

# Check ndots setting (high ndots = many search domain attempts)
kubectl exec <pod> -- cat /etc/resolv.conf
```

The `ndots:5` default means any name with fewer than 5 dots is tried with each search domain first. For `google.com`:
1. `google.com.default.svc.cluster.local` → NXDOMAIN
2. `google.com.svc.cluster.local` → NXDOMAIN
3. `google.com.cluster.local` → NXDOMAIN
4. `google.com.ec2.internal` → NXDOMAIN
5. `google.com.` → SUCCESS

That's 4 wasted queries. Fix with FQDN (trailing dot) or reduce ndots:

```yaml
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "2"
```

Or use FQDN in application code:

```yaml
env:
- name: DB_HOST
  value: "postgres.database.svc.cluster.local."  # trailing dot = FQDN
```

### NXDOMAIN for External Domains

```bash
# Check forward plugin configuration
kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}' | grep forward

# Check if VPC DNS resolver is reachable from CoreDNS pod
kubectl exec -n kube-system <coredns-pod> -- cat /etc/resolv.conf

# Test upstream resolution
kubectl exec -n kube-system <coredns-pod> -- nslookup google.com 169.254.169.253
```

VPC DNS resolver is at the VPC CIDR base + 2 (e.g., `10.0.0.2` for a `10.0.0.0/16` VPC) or `169.254.169.253`.

### CoreDNS CrashLoopBackOff

```bash
# Check events
kubectl describe pod -n kube-system -l k8s-app=kube-dns

# Common causes:
# - Invalid Corefile syntax
# - Loop detection (forwarding to itself)
# - Resource limits too low (OOMKill)

# Check for OOMKill
kubectl get pods -n kube-system -l k8s-app=kube-dns -o jsonpath='{.items[*].status.containerStatuses[*].lastState}'

# Validate Corefile syntax
kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}'
```

### DNS Works in Some Pods but Not Others

```bash
# Check dnsPolicy on the failing pod
kubectl get pod <pod-name> -o jsonpath='{.spec.dnsPolicy}'

# Possible values:
# ClusterFirst (default) — uses CoreDNS
# Default — uses node's /etc/resolv.conf
# None — uses custom dnsConfig
# ClusterFirstWithHostNet — for hostNetwork pods
```

If a pod uses `hostNetwork: true`, it needs `dnsPolicy: ClusterFirstWithHostNet` to use CoreDNS.

## Performance Tuning

### Increase Cache Size

```
.:53 {
    ...
    cache 300 {          # Cache TTL 300 seconds
        success 9984 300 # Cache up to 9984 success responses
        denial 9984 5    # Cache up to 9984 NXDOMAIN for 5s
    }
    ...
}
```

### Adjust Resource Requests

```bash
# Check current resources
kubectl get deployment coredns -n kube-system -o jsonpath='{.spec.template.spec.containers[0].resources}'
```

Recommended resources for large clusters:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 256Mi
    # No CPU limit — avoid throttling DNS!
```

### Autopath Plugin (Reduce Search Domain Queries)

```
.:53 {
    ...
    autopath @kubernetes
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    ...
}
```

`autopath` resolves search domain queries server-side, reducing round trips from 5 to 1 for in-cluster names.

## One-Liners

```bash
# Check CoreDNS version
kubectl get deployment coredns -n kube-system -o jsonpath='{.spec.template.spec.containers[0].image}'

# Count DNS queries per second (from metrics)
kubectl exec -n kube-system deploy/coredns -- wget -qO- http://localhost:9153/metrics 2>/dev/null | grep "coredns_dns_requests_total"

# Find pods with custom dnsPolicy
kubectl get pods -A -o json | jq '.items[] | select(.spec.dnsPolicy != "ClusterFirst" and .spec.dnsPolicy != null) | {name: .metadata.name, ns: .metadata.namespace, policy: .spec.dnsPolicy}'

# Find pods with custom dnsConfig
kubectl get pods -A -o json | jq '.items[] | select(.spec.dnsConfig != null) | {name: .metadata.name, ns: .metadata.namespace, config: .spec.dnsConfig}'

# Test DNS from every node (via DaemonSet approach)
kubectl run dns-test --image=busybox:1.36 --restart=Never --rm -it -- sh -c 'for i in 1 2 3 4 5; do nslookup kubernetes.default && echo "OK" || echo "FAIL"; done'

# Check CoreDNS pod distribution across nodes
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide --no-headers | awk '{print $7}' | sort | uniq -c

# Restart CoreDNS (rollout)
kubectl rollout restart deployment coredns -n kube-system

# Watch CoreDNS pod status
kubectl get pods -n kube-system -l k8s-app=kube-dns -w
```

## Clear DNS Cache

CoreDNS has no built-in flush command. Options:

```bash
# Rolling restart (fastest, no downtime)
kubectl rollout restart deployment coredns -n kube-system
```

Or lower the cache TTL in the Corefile:

```
cache 30 {
  success 9984 30   # Cache up to 9984 successful responses for 30s
  denial 9984 5     # Cache up to 9984 NXDOMAIN/errors for 5s
}
```

CoreDNS picks up ConfigMap changes automatically if the `reload` plugin is present. Otherwise, restart the pods.

## Pod DNS Policies

| Policy | Behavior |
|--------|----------|
| `ClusterFirst` (default) | Pod → CoreDNS → VPC DNS |
| `Default` | Pod → Node DNS → VPC DNS (bypasses CoreDNS) |
| `ClusterFirstWithHostNet` | Same as ClusterFirst, for `hostNetwork: true` pods |
| `None` | Pod uses only custom `dnsConfig` — no automatic config |

```bash
# Check a pod's DNS policy
kubectl get pod <pod-name> -o jsonpath='{.spec.dnsPolicy}'
```

Example with custom dnsConfig:

```yaml
spec:
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 10.0.0.2
    searches:
      - my-ns.svc.cluster.local
      - svc.cluster.local
      - cluster.local
    options:
      - name: ndots
        value: "5"
```

## Query Individual CoreDNS Pods

Useful when getting mixed results — helps identify which replica has stale cache.

```bash
# Get CoreDNS pod IPs
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide

# Query each pod directly from a debug pod
kubectl run dns-test --image=busybox:1.36 --restart=Never --rm -it -- sh
nslookup sts.us-east-1.amazonaws.com 10.244.0.15
nslookup sts.us-east-1.amazonaws.com 10.244.1.22

# Query all CoreDNS pods in one go
for ip in $(kubectl get pods -n kube-system -l k8s-app=kube-dns -o jsonpath='{.items[*].status.podIP}'); do
  echo "=== CoreDNS pod: $ip ==="
  kubectl run dns-probe --image=busybox:1.36 --restart=Never --rm -it -- nslookup sts.us-east-1.amazonaws.com $ip
done
```

Mixed results usually mean CoreDNS replicas have stale cache entries, or pods are on nodes in different subnets where the VPC endpoint isn't available. Fix: restart CoreDNS and ensure VPC endpoint has ENIs in all subnets.

## VPC Endpoint Private DNS Not Resolving in Pods

Symptom: `nslookup sts.us-east-1.amazonaws.com` from a pod returns public IPs instead of VPC endpoint private IPs.

### Checklist

```bash
# 1. Confirm private DNS is enabled on the endpoint
aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.us-east-1.sts \
  --query 'VpcEndpoints[].[VpcEndpointId,PrivateDnsEnabled,State]'

# 2. Confirm VPC DNS settings (both must be true)
aws ec2 describe-vpc-attribute --vpc-id vpc-xxxx --attribute enableDnsSupport
aws ec2 describe-vpc-attribute --vpc-id vpc-xxxx --attribute enableDnsHostnames

# 3. Check CoreDNS forward directive (should use /etc/resolv.conf)
kubectl get configmap coredns -n kube-system -o jsonpath='{.data.Corefile}' | grep forward

# 4. Check node resolv.conf (nameserver should be VPC DNS)
kubectl debug node/<node-name> -it --image=busybox -- cat /etc/resolv.conf

# 5. Test from the node (should return private IPs)
kubectl debug node/<node-name> -it --image=busybox -- nslookup sts.us-east-1.amazonaws.com

# 6. Check pod DNS policy
kubectl get pod <pod-name> -o jsonpath='{.spec.dnsPolicy}'

# 7. Restart CoreDNS to flush cache
kubectl rollout restart deployment coredns -n kube-system

# 8. Check DHCP option set (custom DNS bypasses VPC private DNS)
aws ec2 describe-vpcs --vpc-ids vpc-xxxx --query 'Vpcs[].DhcpOptionsId'
aws ec2 describe-dhcp-options --dhcp-options-ids <id> --query 'DhcpOptions[].DhcpConfigurations'
# domain-name-servers should be AmazonProvidedDNS

# 9. Check Private Hosted Zone association
aws route53 list-hosted-zones-by-vpc --vpc-id vpc-xxxx --vpc-region us-east-1
```

If the PHZ is missing, toggle private DNS:

```bash
aws ec2 modify-vpc-endpoint --vpc-endpoint-id vpce-xxxx --no-private-dns-enabled
# wait a minute
aws ec2 modify-vpc-endpoint --vpc-endpoint-id vpce-xxxx --private-dns-enabled
```

## VPC DNS Forwarding

### DHCP Option Sets

Sets the default DNS for all EC2 instances in a VPC:

```bash
aws ec2 create-dhcp-options --dhcp-configurations \
  "Key=domain-name-servers,Values=AmazonProvidedDNS" \
  "Key=domain-name,Values=example.internal"

aws ec2 associate-dhcp-options --dhcp-options-id dopt-xxxx --vpc-id vpc-xxxx
```

- Use `AmazonProvidedDNS` for VPC endpoint private DNS to work
- Custom DNS servers bypass VPC endpoint private DNS resolution
- Nodes need DHCP lease renewal to pick up changes

### Route 53 Resolver Rules

Conditional forwarding per domain:

```bash
# Forward corp.internal to on-prem DNS
aws route53resolver create-resolver-rule \
  --rule-type FORWARD \
  --domain-name corp.internal \
  --resolver-endpoint-id rslvr-out-xxxx \
  --target-ips Ip=10.200.0.53 Ip=10.200.1.53
```

**Gotcha:** Resolver forwarding rules take precedence over VPC endpoint Private Hosted Zones. If a broad rule (like `.` or `amazonaws.com`) forwards to external servers, VPC endpoint private DNS won't work.

Fix with a `SYSTEM` rule for the specific domain:

```bash
aws route53resolver create-resolver-rule \
  --rule-type SYSTEM \
  --domain-name sts.us-east-1.amazonaws.com

aws route53resolver associate-resolver-rule \
  --resolver-rule-id rslvr-rr-xxxx \
  --vpc-id vpc-xxxx
```

## Capture DNS Traffic with tcpdump

### From a CoreDNS Pod

```bash
# Exec into a CoreDNS pod and capture DNS packets
kubectl exec -it -n kube-system <coredns-pod> -- tcpdump -i any port 53 -A

# Save to file
kubectl exec -it -n kube-system <coredns-pod> -- tcpdump -i any port 53 -w /tmp/dns.pcap
```

### From an EKS Node (via SSM)

```bash
aws ssm start-session --target <instance-id>

# Capture all DNS traffic on the node
sudo tcpdump -i any port 53 -A

# Filter by specific pod IP
sudo tcpdump -i any 'port 53 and (src host <pod-ip> or dst host <pod-ip>)'

# Save to pcap file
sudo tcpdump -i any port 53 -w /tmp/dns_capture.pcap
```

### From a Privileged Debug Pod

```bash
kubectl run debug --image=nicolaka/netshoot --privileged --restart=Never --rm -it -- bash

# Inside the pod
tcpdump -i eth0 port 53 -A
```

### Via nsenter into CoreDNS Network Namespace

Target the CoreDNS process directly by entering its network namespace using the PID. Useful when you want to capture only traffic hitting a specific CoreDNS replica without noise from other pods on the node.

```bash
# SSH/SSM into the node running the CoreDNS pod
aws ssm start-session --target <instance-id>

# Find the CoreDNS process ID
ps ax | grep coredns

# Enter the CoreDNS network namespace and capture DNS traffic
sudo nsenter -n -t <PID> tcpdump udp port 53

# Save capture to file
sudo nsenter -n -t <PID> tcpdump udp port 53 -w /tmp/coredns_capture.pcap
```

Then from a separate terminal, run `nslookup` against the CoreDNS service IP and each pod IP to confirm which replica handles the query. If the CoreDNS pod experiences timeouts and you don't see the query in the capture, the issue is network reachability between the client pod's node and the CoreDNS node.

### Monitor via conntrack

```bash
# On the node — see active DNS connections
sudo conntrack -L | grep :53
```

## Node-Level DNS Monitoring

### VPC Flow Logs

Enable VPC Flow Logs to capture all network traffic including DNS queries to CoreDNS.

### CloudWatch Container Insights

```bash
# Tail performance logs (if Container Insights is enabled)
aws logs tail /aws/containerinsights/<cluster-name>/performance --follow
```

### Check iptables DNS rules

```bash
# On the node — verify traffic routing for port 53
sudo iptables -L -n -v | grep 53
```

## Datadog Integration

### Automatic Collection

If the Datadog Agent is deployed on EKS, it can scrape CoreDNS Prometheus metrics on port 9153.

### Annotate the kube-dns Service

```bash
kubectl annotate service kube-dns -n kube-system \
  ad.datadoghq.com/service.check_names='["prometheus"]' \
  ad.datadoghq.com/service.init_configs='[{}]' \
  ad.datadoghq.com/service.instances='[{"prometheus_url":"http://%%host%%:9153/metrics"}]'
```

### Key Datadog Metrics

| Metric | Description |
|--------|-------------|
| `coredns.dns.request.count` | DNS requests |
| `coredns.dns.response.rcode.count` | DNS response codes |
| `coredns.dns.request.duration.seconds` | Query latency |
| `coredns.cache.hits.total` | Cache hits |
| `coredns.cache.misses.total` | Cache misses |

### Alerts to Set Up

- High DNS query latency (p99 > 100ms)
- DNS request rate anomalies
- High SERVFAIL rate (resolution failures)
- CoreDNS pod restarts

```bash
# Verify Datadog agent is scraping CoreDNS
kubectl logs -n datadog -l app=datadog-agent | grep coredns
```

## Quick Reference

```bash
# Status
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
kubectl get svc kube-dns -n kube-system
kubectl get endpoints kube-dns -n kube-system

# Config
kubectl get configmap coredns -n kube-system -o yaml
kubectl edit configmap coredns -n kube-system

# Logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100
kubectl logs -n kube-system -l k8s-app=kube-dns -f

# Test
kubectl run dns-test --image=busybox:1.36 --restart=Never --rm -it -- nslookup kubernetes.default
kubectl exec <pod> -- nslookup <service>.<namespace>.svc.cluster.local

# Scale
kubectl scale deployment coredns -n kube-system --replicas=5

# EKS add-on
aws eks describe-addon --cluster-name <cluster> --addon-name coredns
aws eks update-addon --cluster-name <cluster> --addon-name coredns --addon-version <version>

# Restart (flush cache)
kubectl rollout restart deployment coredns -n kube-system

# Query individual CoreDNS pod
kubectl run dns-test --image=busybox:1.36 --restart=Never --rm -it -- nslookup <domain> <coredns-pod-ip>

# Capture DNS traffic
kubectl exec -it -n kube-system <coredns-pod> -- tcpdump -i any port 53 -A
```
