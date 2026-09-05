# Troubleshooting DNS Resolution Issues on EKS

DNS problems are maddening because they rarely announce themselves as DNS problems. Services
can't reach each other, image pulls stall, external API calls time out, health checks flap —
and the common root cause underneath all of it is name resolution. It just doesn't look that
way at first.

On EKS, resolution spans several layers: **CoreDNS** for cluster-internal names, the **VPC
DNS resolver** for AWS resources, and upstream resolvers for the public internet. A fault at
any layer breaks a different-looking set of things. This guide walks a systematic path — from
the Pod outward — so you can localize the failure instead of guessing.

> For the per-Pod DNS settings referenced below (`dnsPolicy`, `dnsConfig`, `ndots`), see
> [DNS Policies for Pods in Kubernetes](articles/kubernetes-pod-dns-policies.md). For CoreDNS
> internals, see the [CoreDNS deep dive](articles/kubernetes-coredns-deep-dive.md) and the
> [CoreDNS on EKS cheatsheet](articles/coredns-eks-cheatsheet.md).

---

## How DNS resolution works on EKS

When a Pod issues a lookup, the request flows through distinct hops:

1. The Pod sends the query to **CoreDNS** at the cluster DNS ClusterIP (commonly
   `10.100.0.10`).
2. CoreDNS answers **cluster-internal** names (Services, Pods) from the Kubernetes API via
   its `kubernetes` plugin.
3. For **external** names, CoreDNS forwards to the **VPC DNS resolver** — the VPC's base CIDR
   address **plus two** (e.g. `10.0.0.2` for a `10.0.0.0/16` VPC).
4. The VPC resolver handles Route 53 private/public hosted zones and upstream internet
   resolution.

```
Pod ── mysvc.default.svc.cluster.local ─▶ CoreDNS ─▶ Kubernetes API   (internal)
Pod ── rds.amazonaws.com ───────────────▶ CoreDNS ─▶ VPC DNS (.2) ─▶ Route 53 / internet
```

Knowing these hops is what makes the rest of this guide fast: each symptom points at one hop.

## Step 1: Identify the symptom

Before touching config, find out *what* fails. Run a throwaway Pod and try each query type:

```bash
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- sh

# Inside the pod:
nslookup kubernetes.default.svc.cluster.local   # cluster service (internal)
nslookup kubernetes                              # short name (relies on search domains)
nslookup google.com                              # external domain
nslookup s3.us-west-2.amazonaws.com              # AWS service endpoint
```

The pattern of what breaks tells you which layer to inspect:

| Symptom | Likely layer |
|---------|--------------|
| Internal names fail, external work | CoreDNS (`kubernetes` plugin / API access) |
| External names fail, internal work | Upstream forwarding / VPC DNS |
| Everything fails | CoreDNS unreachable or down |
| Short names fail, FQDNs work | Search domains / `ndots` / `dnsPolicy` |

## Step 2: Check CoreDNS pods

CoreDNS runs as a Deployment in `kube-system`. Confirm it's healthy and not crash-looping:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide

# Restart counts (high counts hint at OOMKills or crashes)
kubectl get pods -n kube-system -l k8s-app=kube-dns \
  -o custom-columns=NAME:.metadata.name,READY:.status.containerStatuses[0].ready,RESTARTS:.status.containerStatuses[0].restartCount,NODE:.spec.nodeName

kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100
```

Log lines worth recognizing:

| Log message | Meaning |
|-------------|---------|
| `i/o timeout` | Can't reach the upstream resolver |
| `SERVFAIL` | Upstream returned an error |
| `connection refused` | Can't connect to the Kubernetes API |
| `plugin/loop: Loop detected` | DNS forwarding loop — a config problem (see Step 9) |

## Step 3: Verify the CoreDNS Service and endpoints

If CoreDNS pods are healthy but Pods still can't reach them, check the Service has ready
backends:

```bash
kubectl get svc kube-dns -n kube-system
kubectl get endpointslice -n kube-system -l kubernetes.io/service-name=kube-dns
```

No ready addresses in the EndpointSlices means the CoreDNS pods aren't `Ready` or the Service
selector doesn't match the pods.

## Step 4: Check the CoreDNS ConfigMap

The Corefile lives in a ConfigMap:

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

A healthy default looks like this:

```text
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

Watch for: a `forward` pointing at the wrong upstream, a missing `kubernetes` plugin block
(kills internal resolution), or typos in any custom zone entries.

## Step 5: Check the Pod's DNS configuration

Confirm the Pod is actually pointed at CoreDNS:

```bash
kubectl exec my-pod -- cat /etc/resolv.conf
```

Expected on EKS (note the extra `*.compute.internal` search domain the VPC injects):

```text
nameserver 10.100.0.10
search default.svc.cluster.local svc.cluster.local cluster.local us-west-2.compute.internal
options ndots:5
```

If `nameserver` doesn't point at the cluster DNS IP, the Pod's `dnsPolicy` is likely wrong.
`ClusterFirst` is the default and the one you want for most workloads:

```yaml
spec:
  dnsPolicy: ClusterFirst   # default — uses CoreDNS
```

The other values (`Default` uses the node's resolver and skips CoreDNS entirely; `None`
requires a manual `dnsConfig`; `ClusterFirstWithHostNet` for host-network pods) are covered in
[DNS Policies for Pods in Kubernetes](articles/kubernetes-pod-dns-policies.md).

## Step 6: ndots and DNS query amplification

The default `options ndots:5` means any name with fewer than 5 dots gets the search domains
appended **first**. On EKS a lookup for `api.example.com` (2 dots) expands to:

```text
1. api.example.com.default.svc.cluster.local   → NXDOMAIN
2. api.example.com.svc.cluster.local           → NXDOMAIN
3. api.example.com.cluster.local               → NXDOMAIN
4. api.example.com.us-west-2.compute.internal  → NXDOMAIN
5. api.example.com                             → finally resolves
```

That's 5x the queries per external lookup, and it can overwhelm CoreDNS in high-traffic
clusters. Two fixes:

```yaml
# Lower ndots for external-heavy workloads
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"
```

Or append a trailing dot to external names in your app config (`api.example.com.`) to make
them absolute and skip search-domain expansion entirely.

## Step 7: Check VPC DNS

If internal resolution works but external doesn't, suspect the VPC resolver. Test it directly
from a node, then verify the VPC attributes:

```bash
# From a node — query the VPC resolver directly (VPC base CIDR + 2)
nslookup google.com 10.0.0.2

# Both of these must be enabled
aws ec2 describe-vpc-attribute --vpc-id vpc-0abc123 --attribute enableDnsSupport
aws ec2 describe-vpc-attribute --vpc-id vpc-0abc123 --attribute enableDnsHostnames
```

Both `enableDnsSupport` and `enableDnsHostnames` must be `true`. For private clusters, also
confirm VPC endpoints have **private DNS** enabled so AWS service endpoints resolve.

## Step 8: Scale CoreDNS

Slow or timing-out lookups under load often mean CoreDNS is under-provisioned:

```bash
kubectl top pods -n kube-system -l k8s-app=kube-dns
kubectl scale deployment coredns -n kube-system --replicas=5
```

For hands-off scaling, add an HPA:

```yaml
# coredns-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: coredns
  namespace: kube-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: coredns
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

Better still, deploy **NodeLocal DNSCache** — a per-node DNS cache that slashes CoreDNS load
and cuts tail latency:

```bash
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
kubedns=$(kubectl get svc kube-dns -n kube-system -o jsonpath='{.spec.clusterIP}')
domain=cluster.local
localdns=169.254.20.10
sed -i "s/__PILLAR__LOCAL__DNS__/$localdns/g; s/__PILLAR__DNS__DOMAIN__/$domain/g; s/__PILLAR__DNS__SERVER__/$kubedns/g" nodelocaldns.yaml
kubectl create -f nodelocaldns.yaml
```

## Step 9: Common fixes

**CoreDNS OOMKilled** — bump the memory limit:

```bash
kubectl patch deployment coredns -n kube-system --type json \
  -p '[{"op":"replace","path":"/spec/template/spec/containers/0/resources/limits/memory","value":"256Mi"}]'
```

**DNS loop detected** — CoreDNS is forwarding to itself. Its `forward . /etc/resolv.conf`
follows the node's `resolv.conf`; if that points at `127.0.0.1` (a local stub resolver),
CoreDNS loops. Ensure the node's `/etc/resolv.conf` points at the VPC DNS resolver, not
loopback.

**Intermittent failures** — frequently conntrack table exhaustion on busy nodes:

```bash
sudo sysctl net.netfilter.nf_conntrack_count
sudo sysctl net.netfilter.nf_conntrack_max
```

If `count` is near `max`, raise the limit or cut DNS query volume (NodeLocal DNSCache and a
lower `ndots` both help).

**AWS service endpoints won't resolve** — verify VPC DNS is enabled (Step 7) and, for private
clusters, that the relevant VPC endpoints have private DNS enabled.

## The mental model

DNS troubleshooting on EKS feels mysterious only until you fix the layers in your head. Work
outward:

1. **Pod** — can it reach CoreDNS at all? (`resolv.conf`, `dnsPolicy`)
2. **CoreDNS** — is it running, scheduled, and configured correctly? (pods, endpoints,
   Corefile)
3. **Upstream** — can the VPC resolver reach Route 53 and the internet? (VPC attributes,
   direct `.2` query)

At each layer the diagnostic points you to the next. Start narrow, confirm each hop, and the
"mysterious" failure usually resolves into one misconfiguration.

---

### Sources

- [DNS for Services and Pods (Kubernetes docs)](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Customizing DNS Service / NodeLocal DNSCache](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/)
- [Amazon EKS networking and DNS](https://docs.aws.amazon.com/eks/latest/userguide/network_reqs.html)
- [VPC DNS attributes (enableDnsSupport / enableDnsHostnames)](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns.html)
- [CoreDNS forward plugin](https://coredns.io/plugins/forward/)
