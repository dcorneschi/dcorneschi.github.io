# What Happens When a Pod Gets an IP Address

The generic CNI flow for pod IP assignment — how the kubelet calls the CNI plugin, network namespaces are created, veth pairs connect the pod to the node, and IP addresses are allocated.

Note: For AWS VPC CNI specifics (IPAMD, ENI management, warm pool), see the EKS VPC CNI IPAMD guide. This article covers the generic Kubernetes CNI mechanics.

## High-Level Flow

```
┌──────────┐     ┌───────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Kubelet │────▶│   Container   │────▶│  CNI Plugin  │────▶│ Network Config  │
│          │     │   Runtime     │     │  (binary)    │     │ (netns, veth,   │
│          │     │  (containerd) │     │              │     │  IP, routes)    │
└──────────┘     └───────────────┘     └──────────────┘     └─────────────────┘
```

## Step 1: Pod Sandbox Creation

When the kubelet starts a pod, the first thing it does is create a **pod sandbox** — an isolated network namespace:

```
┌────────────────────────────────────────────────────────────┐
│  Kubelet: Starting a Pod                                   │
│                                                            │
│  1. CRI: RunPodSandbox()                                   │
│     → Container runtime creates "pause" container          │
│     → Pause container holds the network namespace          │
│     → All other containers share this namespace            │
│                                                            │
│  2. CRI: Runtime calls CNI ADD                             │
│     → CNI plugin configures network for the namespace      │
│     → Returns IP address to runtime                        │
│     → Runtime returns IP to kubelet                        │
│                                                            │
│  3. CRI: CreateContainer() + StartContainer()              │
│     → App containers start in the already-networked sandbox│
└────────────────────────────────────────────────────────────┘
```

The pause container (`registry.k8s.io/pause:3.9`) exists solely to hold the network namespace alive. It does nothing else.

```bash
# See pause containers on a node:
crictl ps -a | grep pause

# Inspect a pod's sandbox:
crictl inspectp <sandbox-id> | jq '.status.network'
```

## Step 2: CNI Plugin Invocation

The container runtime calls the CNI plugin as an executable binary, passing configuration via stdin and environment variables:

```
┌───────────────┐                         ┌───────────────┐
│   Container   │                         │  CNI Binary   │
│   Runtime     │──── exec CNI binary ───▶│(/opt/cni/bin/ │
│  (containerd) │     stdin: config JSON  │ bridge,calico,│
│               │     env: CNI_COMMAND=ADD│ aws-cni...)   │
│               │◀─── stdout: result JSON─│               │
└───────────────┘                         └───────────────┘
```

### Environment Variables Passed to CNI

| Variable | Value | Purpose |
|----------|-------|---------|
| `CNI_COMMAND` | ADD, DEL, CHECK, VERSION | Operation to perform |
| `CNI_CONTAINERID` | `abc123...` | Sandbox container ID |
| `CNI_NETNS` | `/var/run/netns/cni-xyz` | Path to network namespace |
| `CNI_IFNAME` | `eth0` | Interface name inside the pod |
| `CNI_PATH` | `/opt/cni/bin` | Path to CNI plugin binaries |

### CNI Config (stdin)

```json
{
  "cniVersion": "1.0.0",
  "name": "k8s-pod-network",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "ranges": [[{"subnet": "10.244.1.0/24"}]],
    "routes": [{"dst": "0.0.0.0/0"}]
  }
}
```

### CNI Result (stdout)

```json
{
  "cniVersion": "1.0.0",
  "interfaces": [
    {"name": "veth1234", "mac": "aa:bb:cc:dd:ee:01"},
    {"name": "eth0", "mac": "aa:bb:cc:dd:ee:02", "sandbox": "/var/run/netns/cni-xyz"}
  ],
  "ips": [
    {"address": "10.244.1.15/24", "gateway": "10.244.1.1", "interface": 1}
  ],
  "routes": [
    {"dst": "0.0.0.0/0", "gw": "10.244.1.1"}
  ]
}
```

## Step 3: Network Namespace and veth Pair

The CNI plugin creates the actual network plumbing:

```
┌─────────────────────────────────────────────────────────────────┐
│  Node (host network namespace)                                  │
│                                                                 │
│  ┌─────────────────────────┐                                    │
│  │  Bridge (cni0/cbr0)     │                                    │
│  │  IP: 10.244.1.1/24      │                                    │
│  │                         │                                    │
│  │  vethXXXX ──────────────┼──┐                                 │
│  │  vethYYYY ──────────────┼──┼──┐                              │
│  └─────────────────────────┘  │  │                              │
│                               │  │                              │
│  ┌────────────────────────┐   │  │  ┌────────────────────────┐  │
│  │  Pod A (netns)         │   │  │  │  Pod B (netns)         │  │
│  │                        │   │  │  │                        │  │
│  │  eth0: 10.244.1.15/24 ◀┘   │  │  │  eth0: 10.244.1.16/24 ◀┘  │
│  │  default gw: 10.244.1.1    │  │  │  default gw: 10.244.1.1   │
│  └────────────────────────┘   │  │  └────────────────────────┘  │
│                               │  │                              │
└─────────────────────────────────────────────────────────────────┘
```

### What the CNI Plugin Does (bridge plugin example)

```
1. Create network namespace (or use the one CRI created)
   → ip netns add cni-xyz

2. Create veth pair
   → ip link add vethXXXX type veth peer name eth0
   
3. Move one end into pod namespace
   → ip link set eth0 netns cni-xyz
   
4. Attach host end to bridge
   → ip link set vethXXXX master cni0
   → ip link set vethXXXX up
   
5. Configure pod interface
   → ip netns exec cni-xyz ip addr add 10.244.1.15/24 dev eth0
   → ip netns exec cni-xyz ip link set eth0 up
   → ip netns exec cni-xyz ip route add default via 10.244.1.1
   
6. Enable bridge interface (if not already)
   → ip link set cni0 up
```

## Step 4: IPAM — IP Address Allocation

The IP address is managed by an IPAM (IP Address Management) plugin, separate from the network plugin:

```
CNI Plugin (bridge/calico/flannel)
    │
    │ Delegates IP allocation to:
    ▼
IPAM Plugin (host-local/calico-ipam/aws-cni)
    │
    │ Returns allocated IP
    ▼
CNI Plugin configures interface with IP
```

### IPAM Types

| IPAM Plugin | How It Allocates | Used By |
|-------------|-----------------|---------|
| `host-local` | File-based per-node allocation from subnet | bridge, flannel |
| `calico-ipam` | etcd/Kubernetes-backed, cluster-wide IP pool | Calico |
| `aws-cni` (IPAMD) | Pre-allocates IPs from VPC ENIs | AWS VPC CNI |
| `whereabouts` | Cluster-wide IPAM via CRDs | Generic overlay networks |
| `dhcp` | DHCP request to external server | Bare-metal integrations |

### Pod CIDR Allocation

The cluster allocates a pod CIDR to each node. The IPAM plugin then assigns individual IPs from that range:

```
Cluster pod CIDR: 10.244.0.0/16
    │
    ├── Node 1: 10.244.1.0/24 (256 IPs)
    │     ├── Pod A: 10.244.1.15
    │     ├── Pod B: 10.244.1.16
    │     └── Pod C: 10.244.1.17
    │
    ├── Node 2: 10.244.2.0/24
    │     ├── Pod D: 10.244.2.5
    │     └── Pod E: 10.244.2.6
    │
    └── Node 3: 10.244.3.0/24
          └── Pod F: 10.244.3.8
```

```bash
# See node's pod CIDR:
kubectl get node <name> -o jsonpath='{.spec.podCIDR}'

# See all nodes' CIDRs:
kubectl get nodes -o custom-columns=NAME:.metadata.name,CIDR:.spec.podCIDR
```

### host-local IPAM Storage

For `host-local`, allocated IPs are tracked in files on the node:

```bash
# On the node:
ls /var/lib/cni/networks/k8s-pod-network/
# 10.244.1.15  10.244.1.16  10.244.1.17  last_reserved_ip.0  lock
```

Each file contains the container ID that owns that IP.

## Step 5: Cross-Node Communication

Pods on different nodes need a way to reach each other. The CNI plugin (or overlay) handles this:

### Overlay Networks (Flannel VXLAN, Calico IPIP)

```
Pod A (10.244.1.15) on Node 1
    │
    │ packet dst: 10.244.2.5
    ▼
Node 1 routing table:
    10.244.2.0/24 → via VXLAN tunnel to Node 2
    │
    │ encapsulated in UDP:8472 (VXLAN)
    ▼
Node 2 decapsulates:
    │
    │ delivers to Pod D (10.244.2.5)
    ▼
Pod D receives packet
```

### Native Routing (Calico BGP, AWS VPC CNI)

```
Pod A (10.244.1.15) on Node 1
    │
    │ packet dst: 10.244.2.5
    ▼
Node 1 routing table:
    10.244.2.0/24 → via Node 2 IP (learned from BGP or VPC route table)
    │
    │ routed natively (no encapsulation)
    ▼
Node 2 routing:
    10.244.2.5 → via veth to Pod D
    ▼
Pod D receives packet
```

| Approach | Overhead | Setup | Example |
|----------|----------|-------|---------|
| VXLAN overlay | ~50 bytes per packet | Simple, any infrastructure | Flannel VXLAN, Calico VXLAN |
| IPIP tunnel | ~20 bytes per packet | Moderate | Calico IPIP |
| Native routing (BGP) | None | Complex, requires BGP support | Calico BGP |
| VPC native | None | Cloud-specific | AWS VPC CNI, Azure CNI |

## Step 6: Pod Sees Its Network

Inside the pod, the network looks like:

```bash
# From inside the pod:
ip addr show eth0
# 2: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     inet 10.244.1.15/24 scope global eth0

ip route
# default via 10.244.1.1 dev eth0
# 10.244.1.0/24 dev eth0 proto kernel scope link src 10.244.1.15

cat /etc/resolv.conf
# nameserver 10.100.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

## CNI Plugin Chain

CNI supports chaining multiple plugins. Each plugin in the chain adds functionality:

```json
{
  "cniVersion": "1.0.0",
  "name": "k8s-pod-network",
  "plugins": [
    {
      "type": "calico",
      "ipam": {"type": "calico-ipam"}
    },
    {
      "type": "bandwidth",
      "ingressRate": 1000000,
      "egressRate": 1000000
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true}
    }
  ]
}
```

| Plugin | Purpose |
|--------|---------|
| Main plugin (bridge/calico/flannel) | Creates interface, assigns IP |
| `bandwidth` | Traffic shaping (rate limiting) |
| `portmap` | HostPort mapping (iptables DNAT) |
| `tuning` | Sysctl settings for the pod namespace |
| `firewall` | iptables rules for pod isolation |

## CNI Configuration Location

```bash
# CNI config directory (kubelet reads from here):
ls /etc/cni/net.d/
# 10-calico.conflist
# 10-flannel.conflist
# 10-aws.conflist

# CNI binary directory:
ls /opt/cni/bin/
# bridge  calico  flannel  host-local  loopback  portmap  bandwidth  tuning
```

The kubelet uses the first config file (alphabetically) in `/etc/cni/net.d/`.

```bash
# Kubelet CNI flags:
# --cni-conf-dir=/etc/cni/net.d
# --cni-bin-dir=/opt/cni/bin
# --network-plugin=cni (deprecated in 1.24+, CNI is always used)
```

## Complete Timeline

```
Time ──────────────────────────────────────────────────────────────────▶

Kubelet         Container Runtime      CNI Plugin        IPAM Plugin
   │                │                     │                 │
   │ RunPodSandbox ▶│                     │                 │
   │                │ create pause        │                 │
   │                │ container +         │                 │
   │                │ network namespace   │                 │
   │                │                     │                 │
   │                │── exec CNI ADD ────▶│                 │
   │                │   (env + config)    │                 │
   │                │                     │── allocate IP ─▶│
   │                │                     │◀─ 10.244.1.15 ──│
   │                │                     │                 │
   │                │                     │ create veth pair│
   │                │                     │ attach to bridge│
   │                │                     │ configure IP    │
   │                │                     │ add routes      │
   │                │                     │                 │
   │                │◀── result JSON ─────│                 │
   │                │   (IP, routes, DNS) │                 │
   │                │                     │                 │
   │◀── sandbox ID ─│                     │                 │
   │   + pod IP     │                     │                 │
   │                │                     │                 │
   │ CreateContainer│                     │                 │
   │ StartContainer │                     │                 │
   │  (uses same    │                     │                 │
   │   network ns)  │                     │                 │
```

## Pod Deletion — CNI DEL

When a pod is deleted, the reverse happens:

```
Kubelet → CRI StopPodSandbox → Runtime calls CNI DEL
    │
    ▼
CNI Plugin:
    1. IPAM: release IP (remove from allocation)
    2. Delete veth pair (host side disappears, pod side gone with netns)
    3. Network namespace deleted with pause container
```

## hostNetwork Pods

Pods with `hostNetwork: true` skip the entire CNI flow:

```yaml
spec:
  hostNetwork: true   # Pod uses the node's network namespace directly
```

- No separate network namespace
- No CNI plugin called
- Pod sees all node interfaces
- Pod IP = Node IP
- No port isolation (conflicts possible)

Used by: kube-proxy, CNI DaemonSets, node monitoring agents.

## Debugging Pod Networking

```bash
# Check pod IP:
kubectl get pod <name> -o wide

# Check CNI config on node:
cat /etc/cni/net.d/*.conflist | jq .

# Check node's pod CIDR:
kubectl get node <name> -o jsonpath='{.spec.podCIDR}'

# Inspect pod's network namespace (on the node):
crictl inspectp <sandbox-id> | jq '.info.runtimeSpec.linux.namespaces[] | select(.type=="network")'

# Enter pod's network namespace:
PID=$(crictl inspect <container-id> | jq .info.pid)
nsenter -t $PID -n ip addr
nsenter -t $PID -n ip route

# Check veth pair mapping:
ip link show type veth
# Match ifindex: inside pod "eth0@ifN" corresponds to host "vethXXXX@ifM"

# Check bridge:
bridge link show
ip addr show cni0

# Check IPAM allocations (host-local):
ls /var/lib/cni/networks/

# CNI plugin logs:
journalctl -u kubelet | grep -i cni
# Or plugin-specific logs (e.g., /var/log/calico/cni/)
```

## Quick Reference

```bash
# Pod IP assignment flow:
# 1. Kubelet → CRI: RunPodSandbox (creates pause container + netns)
# 2. CRI → CNI binary: exec with CNI_COMMAND=ADD
# 3. CNI → IPAM: allocate IP from node's podCIDR
# 4. CNI: create veth pair, configure IP/routes, attach to bridge
# 5. CRI → Kubelet: returns pod IP
# 6. Kubelet: starts app containers in same netns

# CNI config: /etc/cni/net.d/ (first file alphabetically wins)
# CNI binaries: /opt/cni/bin/
# IPAM state (host-local): /var/lib/cni/networks/

# Pod CIDR per node:
kubectl get nodes -o custom-columns=NAME:.metadata.name,CIDR:.spec.podCIDR

# Verify pod connectivity:
kubectl exec <pod> -- ping <other-pod-ip>
kubectl exec <pod> -- ip addr
kubectl exec <pod> -- ip route
```
