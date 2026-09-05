# DNS Policies for Pods in Kubernetes

Every Pod gets a `/etc/resolv.conf` that decides how its name lookups are resolved — which
nameservers it queries, what search domains get appended to short names, and options like
`ndots`. In Kubernetes, that file is generated from the Pod's **`dnsPolicy`** (and
optionally **`dnsConfig`**). Getting this right matters: the wrong policy is a common cause
of "my Pod can't resolve the service" and of surprisingly slow DNS.

This article explains the four DNS policies, how each shapes `resolv.conf`, when to use
each, how `dnsConfig` customizes or overrides them, and the classic `ndots` performance
trap.

> This is about the **per-Pod** DNS settings. For how the cluster DNS server itself works
> (CoreDNS internals, plugins, records), see the
> [CoreDNS deep dive](articles/kubernetes-coredns-deep-dive.md).

---

## The `dnsPolicy` field

`dnsPolicy` is set in the Pod spec (`spec.dnsPolicy`) and takes one of four values. **The
default is `ClusterFirst`**, *not* `Default` — a naming quirk that trips people up.

| Policy | What it does |
|--------|--------------|
| **`ClusterFirst`** (default) | Cluster DNS (CoreDNS) is the nameserver. Queries matching the cluster domain (e.g. `*.cluster.local`) resolve internally; everything else is forwarded upstream by CoreDNS. |
| **`Default`** | The Pod inherits the **node's** `/etc/resolv.conf`. It uses whatever the node uses — **no** cluster service discovery. |
| **`ClusterFirstWithHostNet`** | Same as `ClusterFirst`, but required for Pods running with `hostNetwork: true` (which would otherwise fall back to `Default`). |
| **`None`** | Ignore all Kubernetes-provided DNS settings; you supply everything via `dnsConfig`. |

### ClusterFirst (the default)

The Pod's `resolv.conf` points at the cluster DNS service (CoreDNS), with search domains
for the Pod's namespace and cluster:

```
# /etc/resolv.conf in a Pod in namespace "prod"
nameserver 10.96.0.10
search prod.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

- **In-cluster names resolve:** `my-svc` → `my-svc.prod.svc.cluster.local`, and FQDNs like
  `db.other-ns.svc.cluster.local` work.
- **External names still work:** CoreDNS forwards anything outside `cluster.local` to the
  upstream resolvers it was configured with.
- This is what you want for **almost all workloads** that talk to Services.

### Default (inherit the node's resolver)

```
# Mirrors the node's /etc/resolv.conf — e.g. the VPC/DHCP resolver
nameserver 169.254.169.253
search ec2.internal
```

- **No Kubernetes service discovery** — `my-svc` won't resolve to a ClusterIP.
- Use it only when a Pod should behave exactly like a process on the node (e.g. some
  node-level agents), or when you explicitly don't want cluster DNS.
- The name is misleading: `Default` is **not** the default policy.

### ClusterFirstWithHostNet

Pods with `hostNetwork: true` default to `Default` DNS, which usually isn't what you want —
they lose cluster service discovery. If a host-network Pod needs to resolve cluster
Services, set this explicitly:

```yaml
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
```

Forgetting this is a frequent "host-network Pod can't reach the API/Service by name" bug.

### None (bring your own DNS)

`dnsPolicy: None` tells Kubernetes to inject **nothing** — you must provide the full DNS
configuration through `dnsConfig`. Use it when you need a fully custom resolver setup that
none of the other policies produce.

## Customizing with `dnsConfig`

`spec.dnsConfig` lets you **add to** (with most policies) or **fully define** (with
`None`) the Pod's DNS settings. It has three fields:

- `nameservers` — list of IP addresses to use as DNS servers.
- `searches` — list of search domains for hostname lookup.
- `options` — list of `{name, value}` objects (e.g. `ndots`, `edns0`, `timeout`).

### Example: fully custom DNS with `None`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns
spec:
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 1.1.1.1
      - 8.8.8.8
    searches:
      - example.com
      - internal.example.com
    options:
      - name: ndots
        value: "2"
      - name: edns0
  containers:
    - name: app
      image: nginx
```

Produces exactly:

```
nameserver 1.1.1.1
nameserver 8.8.8.8
search example.com internal.example.com
options ndots:2 edns0
```

### Example: keep ClusterFirst but tune it

With a policy other than `None`, `dnsConfig` **merges** with the generated config —
nameservers/searches are appended (deduplicated), and options are merged (your values win
on conflicts). This is the common pattern for lowering `ndots` while keeping cluster DNS:

```yaml
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    options:
      - name: ndots
        value: "1"
```

Merge rules to keep in mind:

- **`nameservers`:** appended to the base; a duplicate is removed. With `None`, at least one
  is required. Kubernetes accepts **at most 3 nameservers** total — extra entries are
  rejected, so budget carefully when combining cluster DNS with custom resolvers.
- **`searches`:** appended to the base list.
- **`options`:** merged; an option you specify **overrides** the same option from the base.

### Corporate DNS and split-horizon lookups

A common ask is "resolve internal Service names via CoreDNS **and** corporate hostnames via
the company's DNS servers." The tempting-but-wrong fix is to list several nameservers in
`dnsConfig` and expect each to handle its own domains. A Pod resolver **doesn't route by
domain** — it tries nameservers **in order** until one answers, so whichever is first ends
up fielding everything.

The correct split-DNS approach is to configure **forwarding rules in CoreDNS** (a `forward`
block per zone), pointing the corporate zones at the corporate servers while `cluster.local`
stays internal. Keep the Pod on `ClusterFirst` and use `dnsConfig` only to append the
corporate **search** domain:

```yaml
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    searches:
      - corp.example.com
```

If you genuinely need the Pod to bypass CoreDNS entirely and talk to corporate resolvers,
use `dnsPolicy: None` with explicit `nameservers` — but then you only get cluster Service
resolution if those corporate servers forward the cluster zones back to CoreDNS.

## The `ndots:5` performance trap

By default, `ClusterFirst` sets **`options ndots:5`**. `ndots` means: "if a name has fewer
than N dots, try the search domains **first** before treating it as absolute."

With `ndots:5`, a lookup for an external name like `api.github.com` (2 dots, < 5) is first
tried against **every** search domain:

```
api.github.com.prod.svc.cluster.local   → NXDOMAIN
api.github.com.svc.cluster.local         → NXDOMAIN
api.github.com.cluster.local             → NXDOMAIN
api.github.com                           → finally resolves
```

That's several wasted round-trips per external lookup — a real latency and CoreDNS-load
problem for chatty apps calling external APIs.

**Fixes:**

- **Use a fully-qualified name with a trailing dot:** `api.github.com.` — the trailing dot
  makes it absolute, skipping the search list entirely. Great for external endpoints your
  app calls a lot.
- **Lower `ndots` for the Pod** via `dnsConfig` (e.g. `ndots:1` or `ndots:2`) if the
  workload mostly talks to external names. Trade-off: short *in-cluster* names that rely on
  the search list may then need more dots or FQDNs.
- **Cache with NodeLocal DNSCache** to blunt the cost of the extra queries cluster-wide.

### Tuning `timeout` and `attempts`

Two more resolver options shape how DNS failures are handled:

- **`timeout`** (seconds) — how long to wait for a nameserver to answer before retrying.
- **`attempts`** — how many times to retry across the nameserver list.

```yaml
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    options:
      - name: timeout
        value: "2"
      - name: attempts
        value: "3"
```

It's a balance: very short timeouts turn a momentarily slow resolver into hard lookup
failures, while long timeouts stall the application on every miss. Tune these together with
`ndots` once you've watched real query patterns.

## Reading and debugging a Pod's DNS

```bash
# What resolver config did the Pod actually get?
kubectl exec -it <pod> -- cat /etc/resolv.conf

# Confirm the policy/config on the spec
kubectl get pod <pod> -o jsonpath='{.spec.dnsPolicy}{"\n"}'
kubectl get pod <pod> -o jsonpath='{.spec.dnsConfig}{"\n"}'

# Resolve a service from inside the Pod (needs a shell/tools in the image)
kubectl exec -it <pod> -- nslookup my-svc
kubectl exec -it <pod> -- nslookup my-svc.prod.svc.cluster.local

# Is cluster DNS itself healthy?
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns

# Spin up a throwaway debug pod with DNS tools
kubectl run dnsutils --rm -it --image=registry.k8s.io/e2e-test-images/agnhost:2.39 -- /bin/sh
```

## Choosing a policy

| Situation | Policy |
|-----------|--------|
| Normal workload that talks to Services | `ClusterFirst` (default) |
| Pod on `hostNetwork: true` that needs cluster DNS | `ClusterFirstWithHostNet` |
| Pod that must behave like a node process (no cluster DNS) | `Default` |
| Fully custom resolver (specific upstreams/search/options) | `None` + `dnsConfig` |
| Keep cluster DNS but lower `ndots` / add a search domain | any + `dnsConfig` (merge) |

## Common mistakes

- **Assuming `Default` is the default.** It isn't — `ClusterFirst` is. Setting `Default`
  silently breaks Service name resolution.
- **hostNetwork Pods losing cluster DNS** because `ClusterFirstWithHostNet` wasn't set.
- **Ignoring `ndots:5`** and eating multi-query latency on every external call; fix with
  FQDN-with-trailing-dot or a lower `ndots`.
- **Using `None` without any `nameservers`** — the Pod will have no resolver at all.
- **Expecting `dnsConfig` to replace the base config** with a non-`None` policy — it
  *merges*; only `None` gives you a clean slate.

---

### Sources

- [DNS for Services and Pods (Kubernetes docs)](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Pod's DNS Policy](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy)
- [Pod's DNS Config](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-dns-config)
- [Customizing DNS Service / NodeLocal DNSCache](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)
