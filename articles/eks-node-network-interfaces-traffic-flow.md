# EKS Node Network Interfaces and Traffic Flow

How pod networking works at the node level with AWS VPC CNI — network namespaces, veth pairs, ENIs, routing, and traffic capture.

## Network Architecture Diagram (AWS VPC CNI)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              EKS NODE (EC2 Instance)                            │
│                                                                                 │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐               │
│  │     Pod A        │  │     Pod B        │  │     Pod C        │               │
│  │  10.0.1.10       │  │  10.0.1.11       │  │  10.0.1.12       │               │
│  │                  │  │                  │  │                  │               │
│  │  eth0 (in netns) │  │  eth0 (in netns) │  │  eth0 (in netns) │               │
│  └───────┬──────────┘  └───────┬──────────┘  └───────┬──────────┘               │
│          │                     │                     │                          │
│          │ veth pair           │ veth pair           │ veth pair                │
│          │                     │                     │                          │
│  ┌───────┴──────────┐  ┌───────┴──────────┐  ┌───────┴──────────┐               │
│  │ eni1a2b3c4d@if5  │  │ eni5e6f7g8h@if6  │  │ eni9i0j1k2l@if7  │               │
│  │ (veth on host)   │  │ (veth on host)   │  │ (veth on host)   │               │
│  └───────┬──────────┘  └───────┬──────────┘  └───────┬──────────┘               │
│          │                     │                     │                          │
│          └─────────────────────┼─────────────────────┘                          │
│                                │                                                │
│                    ┌───────────┴───────────┐                                    │
│                    │   Linux Routing Table │                                    │
│                    │   + iptables/DNAT     │                                    │
│                    │   + conntrack         │                                    │
│                    └───────────┬───────────┘                                    │
│                                │                                                │
│          ┌─────────────────────┼─────────────────────┐                          │
│          │                     │                     │                          │
│  ┌───────┴─────────┐   ┌───────┴────────┐   ┌────────┴───────┐                  │
│  │   eth0 (primary)│   │   eth1 (ENI 2) │   │   eth2 (ENI 3) │                  │
│  │   10.0.1.5      │   │   10.0.1.20    │   │   10.0.1.30    │                  │
│  │   (node IP)     │   │   (secondary)  │   │   (secondary)  │                  │
│  └───────┬─────────┘   └───────┬────────┘   └───────┬────────┘                  │
│          │                     │                    │                           │
└──────────┼─────────────────────┼────────────────────┼───────────────────────────┘
           │                     │                    │
           └─────────────────────┼────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │      VPC Subnet         │
                    │      10.0.1.0/24        │
                    └────────────┬────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
   ┌──────────┴───┐  ┌───────────┴──┐  ┌────────────┴────────┐
   │ Other Nodes  │  │ NAT Gateway  │  │ 169.254.169.254     │
   │ / Pods       │  │ (internet)   │  │ (IMDS - link local) │
   └──────────────┘  └──────────────┘  └─────────────────────┘
```

## Traffic Flow: Pod to External (via Proxy)

```
Pod A (10.0.1.10)
  │
  │ 1. App resolves proxy from HTTP_PROXY env var
  │
  ▼
Pod A eth0 (network namespace)
  │
  │ 2. SYN to proxy-pod-ip:3128
  │
  ▼
veth pair (eni1a2b3c4d on host side)
  │
  │ 3. Packet enters host network namespace
  │
  ▼
Linux routing table
  │
  │ 4. Route lookup → destination is another pod IP
  │    ip route: 10.0.1.15 dev eniXXX (proxy pod's veth)
  │
  ▼
Proxy Pod veth (host side)
  │
  │ 5. Packet enters proxy pod's network namespace
  │
  ▼
Proxy Pod (squid/envoy on 10.0.1.15:3128)
  │
  │ 6. Proxy opens new connection to destination
  │    (external internet or internal service)
  │
  ▼
Proxy Pod eth0 → veth → host routing → eth0/ENI → VPC → NAT GW → Internet
```

## Traffic Flow: Pod to IMDS (Correct — Direct)

```
Pod A (10.0.1.10)
  │
  │ 1. SDK checks NO_PROXY → 169.254.169.254 is listed → skip proxy
  │
  ▼
Pod A eth0 (network namespace)
  │
  │ 2. PUT http://169.254.169.254/latest/api/token
  │
  ▼
veth pair → host namespace
  │
  │ 3. Route: 169.254.169.254 is link-local
  │    Handled directly by the instance's primary ENI
  │
  ▼
eth0 (primary ENI) → IMDS endpoint (hypervisor)
  │
  │ 4. Response returns with TTL=1 (single hop only)
  │
  ▼
Pod gets token → uses it for GET /latest/meta-data/...
```

## Traffic Flow: Pod to IMDS (Broken — Through Proxy)

```
Pod A (10.0.1.10)
  │
  │ 1. SDK checks NO_PROXY → 169.254.169.254 NOT listed → use proxy
  │
  ▼
Pod A eth0
  │
  │ 2. Connect to proxy:3128, send "PUT http://169.254.169.254/latest/api/token"
  │
  ▼
veth → host routing → proxy pod veth
  │
  ▼
Proxy Pod
  │
  │ 3. Proxy tries to reach 169.254.169.254
  │    FAILS because:
  │    - Link-local not routable from proxy's perspective, OR
  │    - Response TTL=1, can't survive extra hop through proxy
  │
  ▼
Proxy returns error / RST / timeout
  │
  ▼
Pod A SDK → retry immediately → back to step 1 → LOOP (16k packets)
```

## Interface Listing on a Typical EKS Node

```bash
$ ip link show
1: lo: <LOOPBACK,UP>          # loopback
2: eth0: <BROADCAST,UP>       # primary ENI (node IP, management, IMDS)
3: eth1: <BROADCAST,UP>       # secondary ENI (more pod IPs)
4: eth2: <BROADCAST,UP>       # secondary ENI (more pod IPs, if needed)
5: eni1a2b3c@if3: <UP>        # veth to Pod A
6: eni5e6f7g@if3: <UP>        # veth to Pod B
7: eni9i0j1k@if3: <UP>        # veth to Pod C

$ ip route
default via 10.0.1.1 dev eth0
10.0.1.0/24 dev eth0 proto kernel
10.0.1.10 dev eni1a2b3c scope link    # route to Pod A
10.0.1.11 dev eni5e6f7g scope link    # route to Pod B
10.0.1.12 dev eni9i0j1k scope link    # route to Pod C
169.254.169.254 dev eth0              # IMDS (always via primary ENI)
```

## Where to Capture Traffic

| Location | Interface | What you see |
|----------|-----------|-------------|
| Pod's own traffic | `nsenter -t <pid> -n tcpdump -i eth0` | Traffic as the pod sees it |
| Specific pod from host | `tcpdump -i eni1a2b3c` | One pod's traffic from node perspective |
| All pod traffic | `tcpdump -i any` | Everything (noisy) |
| External only | `tcpdump -i eth0` | Traffic leaving/entering the node |
| IMDS calls | `tcpdump -i eth0 host 169.254.169.254` | All IMDS traffic |
| Pod-to-pod on same node | `tcpdump -i any host <pod-ip>` | Stays within node, crosses veths |
| Pod-to-proxy | `tcpdump -i any dst port 3128` | All proxy-bound traffic |

## Key Points

1. **Each pod has its own network namespace** with its own `eth0`
2. **veth pairs** connect pod namespaces to the host namespace
3. **AWS VPC CNI** assigns real VPC IPs to pods (no overlay network)
4. **Secondary ENIs** provide additional IP addresses for pods
5. **IMDS (169.254.169.254)** is only reachable via the primary ENI (eth0) with TTL=1
6. **kube-proxy iptables/DNAT** handles Service → Pod translation in the host namespace
7. **Traffic between pods on the same node** goes: pod A veth → host routing → pod B veth (never leaves the node)
