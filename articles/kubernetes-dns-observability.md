# DNS Observability and Troubleshooting in Kubernetes

DNS is the quiet dependency underneath almost everything in a cluster. Pods find Services by
name, workloads reach external APIs by name, and the whole thing runs through CoreDNS and
whatever upstream resolvers it forwards to. When DNS misbehaves it rarely says "DNS" — it
shows up as latency, transaction timeouts, flaky health checks, or a bad end-user experience.
Because the failures are often intermittent, teams can burn days chasing symptoms.

The way out is **observability**: knowing which DNS signals to watch, what the response codes
mean, and how to capture the actual packets when logs aren't enough. This article is
tool-agnostic — it uses CoreDNS metrics, standard logging, and packet capture rather than any
one vendor's product — and it's organized around the failure patterns you'll actually meet.

> For the per-Pod resolver settings (`dnsPolicy`, `dnsConfig`, `ndots`), see
> [DNS Policies for Pods in Kubernetes](articles/kubernetes-pod-dns-policies.md). For a
> layer-by-layer EKS runbook, see
> [Troubleshooting DNS Resolution Issues on EKS](articles/eks-dns-troubleshooting.md), and for
> CoreDNS internals the [CoreDNS deep dive](articles/kubernetes-coredns-deep-dive.md).

---

## The DNS path, briefly

A Pod's resolver points at the cluster DNS Service (`kube-dns`, usually a ClusterIP like
`10.96.0.10` or `10.0.0.10`), which is backed by CoreDNS pods in `kube-system`:

```bash
kubectl get service kube-dns -n kube-system
kubectl get configmap coredns -n kube-system -o yaml
```

CoreDNS answers cluster-internal names (`*.svc.cluster.local`) from the Kubernetes API and
**forwards** everything else to the upstream resolvers in its Corefile (typically
`forward . /etc/resolv.conf`). That forwarding chain matters: a query can traverse several
recursive resolvers before an answer comes back, so an upstream problem **outside** the
cluster still surfaces as an in-cluster application failure. Good observability has to see
both the internal hop and the upstream one.

## What to observe: the four signals

You don't need a specific product to reason about DNS health — you need these four signals,
whatever tool surfaces them (CoreDNS Prometheus metrics, a logging pipeline, an eBPF/agent
tool, or packet capture).

| Signal | Why it matters | Where to get it |
|--------|----------------|-----------------|
| **Response codes** (`NOERROR`, `SERVFAIL`, `NXDOMAIN`, `REFUSED`) | The single most diagnostic signal — tells you *how* a lookup failed | CoreDNS metrics, query logs, packet capture |
| **Query types** (A, AAAA, CNAME, SRV, PTR) | Reveals dual-stack issues and whether the answer type matches the question | Query logs, packet capture |
| **Latency** (response time distribution) | Slow DNS = slow app; catches upstream/forwarding degradation | CoreDNS `coredns_dns_request_duration_seconds` histogram |
| **Query rate & distribution** | Spots amplification storms and uneven load across CoreDNS pods | CoreDNS metrics per pod, request counters |

CoreDNS exposes Prometheus metrics on `:9153` (the `prometheus` plugin, on by default). The
most useful series:

```text
coredns_dns_requests_total{server,zone,type}          # query volume by type
coredns_dns_responses_total{server,zone,rcode}        # response codes — watch SERVFAIL/NXDOMAIN
coredns_dns_request_duration_seconds_bucket{...}      # latency histogram
coredns_proxy_request_duration_seconds_count{         # upstream volume + rcode per resolver
    proxy_name="forward",to,rcode}                    #   (replaces the deprecated coredns_forward_* series)
coredns_cache_hits_total{type}                        # cache hits by type (success/denial)
coredns_cache_requests_total{}                        # total cache lookups (hit + miss)
coredns_cache_entries{type}                           # current cached entries
```

> Note: the older `coredns_forward_requests_total` / `coredns_forward_responses_total` and
> `coredns_cache_misses_total` series still exist but are **deprecated** — prefer the
> `coredns_proxy_*` metrics and derive misses from `requests - hits`.

Alerting on a rising `rcode="SERVFAIL"` rate, or a latency p99 climbing, catches most DNS
incidents before users do.

### Watch the cache hit ratio

The Corefile's `cache 30` line means CoreDNS caches answers for up to 30 seconds, and the
cache is a signal in its own right. A **collapsing hit ratio** sends every query through to
the upstream forwarder — which shows up as a sudden jump in forward volume, higher latency,
and sometimes upstream rate-limiting. Track it with the cache metrics above; a healthy
cluster serves a large fraction of queries straight from cache:

```text
# hit ratio over 5m — a sharp drop means something is bypassing the cache
sum(rate(coredns_cache_hits_total[5m])) / sum(rate(coredns_cache_requests_total[5m]))
```

Common causes of a low ratio: very short record TTLs from upstream, a flood of unique names
(the `ndots:5` search-domain expansion generates lots of one-off NXDOMAINs), or a restart that
cleared the cache.

## Turning on visibility

**CoreDNS error logging (leave on).** The `errors` plugin — already in the default Corefile —
logs only failures, not every query. It's cheap enough to run permanently and is the first
place to look for `SERVFAIL`s, upstream timeouts, and loops. Keep it enabled always.

**CoreDNS query logging (turn on to investigate).** The `log` plugin emits a line per query.
It's the opposite of `errors` — full visibility but very verbose, so enable it while
investigating rather than permanently:

```text
.:53 {
    log
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

```bash
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100
```

**Packet capture** when logs and metrics don't explain it — capture the raw DNS exchange on
port 53 from a debug pod or a node and open it in Wireshark:

```bash
# Throwaway debug pod with capture tools
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash

# Inside: capture DNS to a file
tcpdump -i any -w /tmp/dns.pcap 'udp port 53 or tcp port 53'
```

Useful Wireshark display filters:

| Filter | Shows |
|--------|-------|
| `dns.flags.rcode == 2` | SERVFAIL responses |
| `dns.flags.rcode == 3` | NXDOMAIN responses |
| `dns.flags.rcode == 0` | NOERROR (may still lack the record you asked for) |
| `dns.qry.type == 1` | A queries (28 = AAAA, 5 = CNAME) |

## Wiring the signals into Datadog

Metrics and logs are only useful if something is watching them continuously. CoreDNS exposes
Prometheus metrics, so any Prometheus-compatible platform can scrape them — the steps below use
Datadog as one concrete example, but the same `:9153` endpoint feeds Prometheus/Grafana,
Grafana Cloud, or others equally well.

The Datadog Agent has a built-in **CoreDNS integration** (OpenMetrics-based) that discovers
CoreDNS pods and scrapes `:9153` automatically. The cleanest way to configure it is
**Autodiscovery annotations** on the CoreDNS Deployment, so config travels with the workload:

```yaml
# In the coredns Deployment pod template (kube-system)
metadata:
  annotations:
    ad.datadoghq.com/coredns.checks: |
      {
        "coredns": {
          "init_config": {},
          "instances": [
            {
              "openmetrics_endpoint": "http://%%host%%:9153/metrics"
            }
          ]
        }
      }
```

`%%host%%` is a Datadog template variable resolved per-pod at discovery time. With the Agent
running as a DaemonSet (and the Cluster Agent deployed), each CoreDNS pod is picked up and its
metrics land under the `coredns.*` namespace. Editing the upstream `coredns` Deployment can be
awkward on managed control planes; on EKS the CoreDNS add-on lets you set pod annotations
through the add-on's advanced configuration instead of patching the Deployment directly.

The four signals map onto Datadog metrics like this:

| Signal | Datadog metric |
|--------|----------------|
| Response codes | `coredns.response_code_count` (tagged by `rcode`) |
| Query types | `coredns.request_type_count` / `coredns.request_count` (tagged by `type`) |
| Latency | `coredns.request_duration.seconds.*` (histogram → p50/p95/p99) |
| Query rate & distribution | `coredns.request_count` by pod; `coredns.forward_request_count` per upstream |
| Cache health | `coredns.cache_hits_count` / `coredns.cache_misses_count` / `coredns.cache_size.count` |

Once the metrics flow, build a couple of monitors that mirror the failure patterns below:

- **SERVFAIL rate** — alert when `coredns.response_code_count{rcode:servfail}` climbs above a
  baseline; points at a broken forwarder.
- **Latency p99** — alert on the `coredns.request_duration.seconds` p99; catches upstream
  degradation and the 5-second race described later.
- **Cache hit ratio** — a formula of `cache_hits_count / (cache_hits_count + cache_misses_count)`
  dropping sharply.
- **Per-pod query skew** — graph `coredns.request_count` split by pod to catch uneven load
  balancing.

Enable **DNS log collection** too (either the CoreDNS `log` plugin routed through Datadog's log
pipeline, or Datadog's own DNS monitoring via the network module) so you can pivot from a
metric spike to the exact queries behind it. For monitor and dashboard mechanics, see
[Datadog Monitor Scoping](articles/datadog-monitor-scoping.md) and the
[Datadog Dashboards Guide](articles/datadog-dashboards-guide.md).

## Failure patterns and how to spot them

### DNS traffic denied by network policy

Because Pod IPs are ephemeral, a workload that can't reach `kube-dns` can't discover anything.
A too-tight `NetworkPolicy` that forgets to allow egress to the DNS Service is a classic
silent breaker — lookups just hang or fail. The tell is that **everything** fails from the
affected pods while other namespaces are fine.

- Check for policies selecting the workload, and confirm they allow egress to `kube-dns` on
  UDP/TCP 53:

```bash
kubectl get networkpolicy -A
kubectl exec my-pod -- nslookup kubernetes.default   # hangs/fails if DNS egress is blocked
```

- The fix is an explicit allow rule for DNS egress (to the `kube-system` DNS pods / Service on
  port 53). Most deny-by-default policy sets ship a ready-made "allow DNS" rule for exactly
  this reason.

### SERVFAIL — the forwarder is broken

`SERVFAIL` means the resolver tried and failed, usually because the **upstream** CoreDNS
forwards to is down, unreachable, or misconfigured. Enterprises often forward specific zones
to designated nameservers; if one is decommissioned or fat-fingered in the Corefile, every
lookup for that zone fails.

- A rising `coredns_dns_responses_total{rcode="SERVFAIL"}` (or `coredns_proxy_request_duration_seconds_count{proxy_name="forward",rcode="SERVFAIL"}`
  against a specific `to=` upstream) points straight at the bad forwarder.
- Inspect any custom forwarding blocks for typos or dead servers:

```bash
kubectl get configmap coredns -n kube-system -o yaml
# Look for a forward block like:  forward github.com 8.8.8.9   (wrong/dead server → SERVFAIL)
```

A forwarder receiving queries but returning `0` bytes back is a dead giveaway it isn't
answering.

### NXDOMAIN — the name doesn't exist

`NXDOMAIN` means no record of any type exists for the queried name. Two common causes:

- A pod queries an in-cluster Service name that doesn't exist (typo, wrong namespace, Service
  deleted).
- An external domain was decommissioned or renamed but workloads keep asking for it.

Confirm with a direct lookup and, if needed, a capture filtered to `dns.flags.rcode == 3` to
see exactly which name is returning NXDOMAIN:

```bash
kubectl exec my-pod -- nslookup my-svc.my-namespace.svc.cluster.local
```

Note that with `ndots:5`, a short external name generates several NXDOMAINs against the search
domains *before* the real query succeeds — that's expected noise, not a fault. See the
[ndots trap](articles/kubernetes-pod-dns-policies.md).

### NOERROR but the record is missing

The sneakiest one. `NOERROR` (rcode 0) means the query succeeded — but the response can still
lack the record type you asked for. A query for an **A** record can come back `NOERROR` with
only a **CNAME** and no A record, and the app silently fails to connect.

- These are often intermittent and tied to **GSLB / delegated zones**, where an authoritative
  nameserver for a delegated zone is misconfigured, or a global load balancer returns
  different answers based on backend health.
- In a capture or logs, look for responses where the question type is `A` but the answer set
  contains only `CNAME` and no `A`. A healthy answer chains CNAME → A (e.g. `app.example.com`
  → `app-123.elb.amazonaws.com` → `3.98.4.166`); a broken one stops at the CNAME.

This is precisely why capturing **query type vs. returned record types** matters — a plain
"success rate" metric hides it.

### Intermittent ~5-second latency (the UDP conntrack race)

If you see occasional DNS lookups that take almost exactly **5 seconds** and then succeed,
you've likely hit the classic Linux DNS race, not a CoreDNS fault. The glibc resolver fires
the A and AAAA queries **in parallel** over UDP from the same socket; under load, the kernel's
conntrack can drop one of the two near-simultaneous inserts (a known netfilter race), so one
query is lost and the client waits for its **5-second timeout** before retrying.

The signature is distinctive: intermittent, ~5s (or ~10s for two misses), and it *resolves on
retry* — so metrics show normal rcodes but ugly latency tails, and apps report sporadic
timeouts. It gets worse as query volume and node density climb.

Mitigations, roughly in order of effectiveness:

- **NodeLocal DNSCache** — the standard fix. It answers from a per-node cache over a local
  link and uses **TCP** to talk to upstream CoreDNS, sidestepping the UDP conntrack race
  entirely. This is the recommended cluster-wide remedy.
- **Resolver options via `dnsConfig`** — `single-request-reopen` (use a fresh source port for
  the A and AAAA queries so they don't collide) or `use-vc` (force TCP). Per-Pod, useful when
  you can't roll out NodeLocal DNSCache:

```yaml
spec:
  dnsConfig:
    options:
      - name: single-request-reopen
```

- **Drop AAAA lookups** for IPv4-only workloads (e.g. `single-request` or app-level config) so
  there's only one query to race.

See [DNS Policies for Pods in Kubernetes](articles/kubernetes-pod-dns-policies.md) for how
`dnsConfig` options merge into a Pod's resolver.

### Uneven CoreDNS load balancing

`kube-dns` is backed by two or more CoreDNS pods for availability and throughput. If a node or
kube-proxy/conntrack issue funnels traffic to a single pod, that pod saturates while others sit
idle — latency climbs and you get sporadic timeouts even though "CoreDNS is up."

- Compare per-pod request rate using `coredns_dns_requests_total` labeled by pod/instance, or
  simply watch CPU across the CoreDNS pods:

```bash
kubectl top pods -n kube-system -l k8s-app=kube-dns
```

- Grossly uneven distribution points at node-level networking (conntrack, kube-proxy) rather
  than CoreDNS itself. **NodeLocal DNSCache** also helps by giving each node its own cache and
  reducing cross-node DNS traffic.

## A practical workflow

1. **Classify the failure by response code first.** SERVFAIL → upstream/forwarder. NXDOMAIN →
   wrong/missing name. NOERROR-with-nothing-useful → record-type or GSLB problem. Total
   failure from specific pods → network policy. Intermittent ~5s stalls that succeed on retry
   → the UDP conntrack race (reach for NodeLocal DNSCache).
2. **Confirm reach to `kube-dns`** from the affected pod (`nslookup kubernetes.default`).
3. **Check CoreDNS health and the Corefile** (pods ready, forward targets valid, no `loop`).
4. **Look at the signals** — rcode rates, latency, per-pod distribution — to localize.
5. **Capture packets** when logs/metrics are ambiguous, and filter by rcode/type in Wireshark.

Start from the response code and the rest of the investigation narrows itself.

## Summary

DNS problems in Kubernetes are hard mostly because they're invisible by default and often
intermittent. You don't need a specific vendor to fix that — you need the four signals
(response codes, query types, latency, distribution), CoreDNS's built-in metrics and `log`
plugin, and packet capture for the ambiguous cases. Learn what SERVFAIL, NXDOMAIN, and
NOERROR-with-missing-records each imply, watch the distribution across CoreDNS pods, and most
"mysterious" DNS incidents become a short, ordered investigation.

---

### Sources

- [CoreDNS metrics / prometheus plugin](https://coredns.io/plugins/metrics/)
- [CoreDNS log plugin](https://coredns.io/plugins/log/)
- [DNS for Services and Pods (Kubernetes docs)](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [DNS response codes (RFC 1035, RCODE)](https://www.rfc-editor.org/rfc/rfc1035)
- [Datadog CoreDNS integration](https://docs.datadoghq.com/integrations/coredns/)
- [Datadog Kubernetes Autodiscovery](https://docs.datadoghq.com/containers/kubernetes/integrations/)
