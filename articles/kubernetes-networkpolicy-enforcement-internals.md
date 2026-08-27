# How Kubernetes NetworkPolicies Are Enforced

How NetworkPolicy objects are translated into actual packet filtering — the CNI plugin's role, iptables vs eBPF implementation, default deny behavior, and traffic flow with and without policies.

## High-Level Flow

```
kubectl apply -f networkpolicy.yaml
        │
        ▼
┌───────────────┐     ┌──────────────────┐     ┌──────────────────────┐
│   API Server  │────▶│  CNI Plugin      │────▶│  Node-Level Rules    │
│  (stores      │     │  (policy agent)  │     │  (iptables/eBPF/     │
│   policy)     │     │  watches policies│     │   nftables)          │
└───────────────┘     └──────────────────┘     └──────────────────────┘
```

**Critical**: The API server does NOT enforce NetworkPolicies. The CNI plugin does. If your CNI doesn't support NetworkPolicies, they have no effect.

## Which CNI Plugins Support NetworkPolicies

| CNI Plugin | Enforcement Engine | NetworkPolicy Support |
|------------|--------------------|-----------------------|
| Calico | iptables or eBPF | Full (ingress + egress) |
| Cilium | eBPF | Full (+ L7 policies via CiliumNetworkPolicy) |
| Weave Net | iptables | Full |
| Antrea | Open vSwitch (OVS) | Full |
| Flannel | None | No support (policies are ignored) |
| AWS VPC CNI | iptables (via Calico add-on) | Requires separate policy agent |
| Azure CNI | iptables (via Azure NPM or Calico) | Requires policy add-on |

```bash
# Check if your cluster has a policy agent:
kubectl get pods -A | grep -i "calico\|cilium\|weave\|antrea"

# Test if policies are enforced (create a deny-all and verify):
kubectl apply -f deny-all.yaml
kubectl exec test-pod -- curl -s --connect-timeout 3 http://target-svc
# Should timeout if policies are enforced
```

## How a NetworkPolicy Works

### The Policy Object

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: production
spec:
  podSelector:           # Which pods this policy applies TO
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:               # What traffic is ALLOWED IN
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:                # What traffic is ALLOWED OUT
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:                  # Allow DNS
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

### Selection Logic

```
┌────────────────────────────────────────────────────────────────┐
│  NetworkPolicy "allow-frontend" in namespace "production"      │
│                                                                │
│  spec.podSelector: {app: backend}                              │
│  → Applies to all pods with label app=backend in production    │
│                                                                │
│  Once ANY policy selects a pod:                                │
│  → Default DENY for the specified policyTypes (ingress/egress) │
│  → Only traffic matching the allow rules gets through          │
│                                                                │
│  Pods with NO policy selecting them:                           │
│  → All traffic allowed (open by default)                       │
└────────────────────────────────────────────────────────────────┘
```

## Default Behavior — Open vs Deny

```
┌──────────────────────────────────────────────────────┐
│  No NetworkPolicy selecting a pod:                   │
│  → ALL ingress allowed                               │
│  → ALL egress allowed                                │
│  → Pod is "unprotected"                              │
│                                                      │
│  Any NetworkPolicy selects a pod (via podSelector):  │
│  → Default DENY for that policyType                  │
│  → Only explicitly allowed traffic gets through      │
│  → Multiple policies are ADDITIVE (union of rules)   │
└──────────────────────────────────────────────────────┘
```

### Default Deny All (Common Starting Point)

```yaml
# Deny all ingress in a namespace:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}       # Selects ALL pods in the namespace
  policyTypes:
  - Ingress
  # No ingress rules = deny everything
---
# Deny all egress in a namespace:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  # No egress rules = deny everything (including DNS!)
---
# Deny all (both directions):
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

## Enforcement: iptables Implementation (Calico)

When Calico enforces a NetworkPolicy, it translates rules into iptables chains on each node:

```
┌─────────────────────────────────────────────────────────────────┐
│  Calico Felix Agent (per node)                                  │
│                                                                 │
│  1. Watches NetworkPolicy objects via API server                │
│  2. Watches Pod objects (to resolve selectors to IPs)           │
│  3. Computes effective rules per pod endpoint                   │
│  4. Programs iptables chains on the node                        │
│                                                                 │
│  iptables structure:                                            │
│                                                                 │
│  FORWARD chain                                                  │
│    → cali-FORWARD                                               │
│      → cali-from-wl-dispatch (from workload)                    │
│        → cali-fw-<endpoint-id> (per pod interface)              │
│          → policy chain: allow TCP 8080 from <frontend-IPs>     │
│          → default: DROP                                        │
│      → cali-to-wl-dispatch (to workload)                        │
│        → cali-tw-<endpoint-id>                                  │
│          → policy chain: allow from <allowed sources>           │
│          → default: DROP                                        │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# On a node, see Calico's iptables chains:
sudo iptables-save | grep cali | head -30

# See rules for a specific pod endpoint:
sudo iptables -L cali-fw-<endpoint-hash> -n --line-numbers

# Calico stores endpoint info:
calicoctl get workloadendpoints -o yaml
```

## Enforcement: eBPF Implementation (Cilium)

Cilium uses eBPF programs attached to pod network interfaces:

```
┌─────────────────────────────────────────────────────────────────┐
│  Cilium Agent (per node)                                        │
│                                                                 │
│  1. Watches NetworkPolicy + CiliumNetworkPolicy objects         │
│  2. Compiles policies into eBPF bytecode                        │
│  3. Attaches eBPF programs to pod veth interfaces               │
│  4. eBPF maps store allowed identity → port combinations        │
│                                                                 │
│  Data path:                                                     │
│                                                                 │
│  Packet arrives at pod's veth                                   │
│    → eBPF program runs (TC hook or XDP)                         │
│    → Looks up source identity in eBPF map                       │
│    → Checks if identity + port is allowed                       │
│    → ALLOW (pass) or DROP                                       │
│                                                                 │
│  No iptables overhead — pure kernel-level filtering             │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# See Cilium endpoints and their policy status:
kubectl exec -n kube-system cilium-<node> -- cilium endpoint list

# See the policy verdict for an endpoint:
kubectl exec -n kube-system cilium-<node> -- cilium policy get <endpoint-id>

# Monitor policy decisions in real-time:
kubectl exec -n kube-system cilium-<node> -- cilium monitor --type policy-verdict
```

### iptables vs eBPF Comparison

| Feature | iptables (Calico default) | eBPF (Cilium, Calico eBPF) |
|---------|--------------------------|---------------------------|
| Performance at scale | Degrades with many rules (O(n) traversal) | Consistent (hash map lookups) |
| Rule update speed | Chain rewrite (can be slow) | Map update (instant) |
| Observability | iptables counters, LOG target | Rich metrics, Hubble flow logs |
| L7 policy support | No (L3/L4 only) | Yes (HTTP, gRPC, DNS) |
| Connection tracking | Kernel conntrack | Custom eBPF conntrack |
| Resource overhead | Moderate | Lower (no conntrack table bloat) |

## Policy Resolution — Multiple Policies

When multiple NetworkPolicies select the same pod, their rules are **combined (OR logic)**:

```
Policy A: allow ingress from app=frontend on port 8080
Policy B: allow ingress from app=monitoring on port 9090

Combined effect on the selected pod:
  → Allow from frontend:8080 OR from monitoring:9090
  → Deny everything else
```

Policies are purely additive. There is no way to create a "deny" rule that overrides another policy's "allow."

```
┌────────────────────────────────────────────────────────────┐
│  Multiple policies selecting pod "backend":                │
│                                                            │
│  Policy 1: ingress from frontend:8080  ─┐                  │
│  Policy 2: ingress from monitoring:9090 ─┼─▶ UNION = allow │
│  Policy 3: ingress from admin:*         ─┘                 │
│                                                            │
│  Result: traffic allowed if it matches ANY policy's rules  │
│  Everything else: DENIED (because pod IS selected)         │
└────────────────────────────────────────────────────────────┘
```

## Selector Types in Rules

### podSelector (Same Namespace)

```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
  # Only pods with app=frontend in the SAME namespace
```

### namespaceSelector

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: production
  # All pods in namespaces labeled environment=production
```

### Combined (AND Logic Within a Rule)

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: production
    podSelector:
      matchLabels:
        app: frontend
  # Pods with app=frontend AND in namespaces with environment=production
```

### Separate Entries (OR Logic Between Rules)

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: production
  - podSelector:
      matchLabels:
        app: frontend
  # Pods in production namespaces OR pods with app=frontend (in same ns)
```

**The single-dash vs two-entries distinction is the #1 NetworkPolicy gotcha.**

### ipBlock (External CIDR)

```yaml
ingress:
- from:
  - ipBlock:
      cidr: 10.0.0.0/8
      except:
      - 10.0.1.0/24
  # Allow from 10.0.0.0/8 except 10.0.1.0/24
```

## What NetworkPolicies Cannot Do

| Limitation | Explanation |
|------------|-------------|
| No deny rules | Policies are additive-only (allow rules that combine as OR) |
| No cluster-wide policies | Must be namespaced (use CiliumClusterWideNetworkPolicy or Calico GlobalNetworkPolicy) |
| No L7 filtering | Standard policies are L3/L4 only (IP + port). Use CNI-specific CRDs for HTTP/gRPC |
| No log-only mode | Traffic is either allowed or dropped — no "audit" mode in the standard API |
| No node-level rules | Can't block traffic from/to nodes — only pod-to-pod |
| No rule ordering | All matching rules are combined (no priority or ordering) |

## Egress Policy — Don't Forget DNS

When you add an egress policy, you block all outgoing traffic including DNS:

```yaml
# This breaks DNS resolution:
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - port: 5432

# Fix: also allow DNS egress:
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - port: 5432
  - to:                           # Allow DNS to CoreDNS
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

## Debugging NetworkPolicies

```bash
# List policies in a namespace:
kubectl get networkpolicies -n production

# See which pods a policy selects:
kubectl get pods -n production -l app=backend

# Describe a policy (see the rules):
kubectl describe networkpolicy allow-frontend -n production

# Test connectivity:
kubectl exec test-pod -- curl -s --connect-timeout 3 http://backend:8080
kubectl exec test-pod -- wget -qO- --timeout=3 http://backend:8080

# Check if CNI supports network policies:
kubectl get pods -A -l k8s-app=calico-node    # Calico
kubectl get pods -A -l k8s-app=cilium         # Cilium

# Calico: check computed policy for a pod:
calicoctl get workloadendpoint -o yaml | grep -A 20 "policy"

# Cilium: check policy verdict:
kubectl exec -n kube-system cilium-<node> -- cilium monitor --type policy-verdict

# Common test: deploy a test pod and curl:
kubectl run curl-test --image=curlimages/curl --rm -it -- \
  curl -s --connect-timeout 3 http://<service>.<namespace>.svc.cluster.local
```

### Troubleshooting Flow

```
Traffic blocked unexpectedly:
  1. Is the pod selected by any NetworkPolicy? (check podSelector)
  2. Is the policyType (Ingress/Egress) declared? (default deny activates)
  3. Does any rule in any selecting policy allow the traffic?
  4. Is the source/destination matching correctly (selector AND vs OR)?
  5. Is DNS allowed in egress policies?
  6. Is the CNI actually enforcing policies?

Traffic NOT blocked (should be):
  1. Does any policy actually select the target pod?
  2. Is the CNI installed and healthy?
  3. Are policies in the correct namespace?
  4. Is the podSelector matching the pod labels?
```

## Quick Reference

```bash
# NetworkPolicy enforcement:
# - API server stores the policy
# - CNI plugin (Calico/Cilium) watches and enforces
# - No CNI support = policies are ignored silently

# Default behavior:
# - No policy selecting a pod = all traffic allowed
# - Any policy selecting a pod = default deny for that policyType
# - Rules are additive (OR across policies)

# Selector gotcha:
# Single array entry = AND (both must match)
# Separate array entries = OR (either matches)

# Don't forget:
# - DNS (UDP/TCP 53) in egress policies
# - NetworkPolicies are namespaced
# - No deny rules exist — only allow
# - ipBlock doesn't match pod IPs (only external CIDRs)

# Key commands:
kubectl get networkpolicies -n <namespace>
kubectl describe networkpolicy <name> -n <namespace>
kubectl get pods -l <selector> -n <namespace>  # Verify policy targets

# Test connectivity:
kubectl exec <pod> -- curl -s --connect-timeout 3 http://<target>
```
