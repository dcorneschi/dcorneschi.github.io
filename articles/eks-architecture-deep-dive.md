# EKS Deep Dive: Architecture, Internals, and What's Behind the Curtain

A technical deep-dive into how Amazon EKS actually works: control plane architecture, networking internals, IAM integration, scalability limits, and the decisions AWS made that affect your day-to-day operations.

## What AWS Manages vs What You Manage

| Component | Who Manages | Where It Runs |
|-----------|-------------|---------------|
| etcd | AWS | AWS-managed VPC (hidden) |
| kube-apiserver | AWS | AWS-managed VPC (hidden) |
| kube-scheduler | AWS | AWS-managed VPC (hidden) |
| kube-controller-manager | AWS | AWS-managed VPC (hidden) |
| cloud-controller-manager | AWS | AWS-managed VPC (hidden) |
| ENI for cross-account access | AWS (in your VPC) | Your VPC subnets |
| kubelet | You | Your worker nodes |
| kube-proxy | You (DaemonSet) | Your worker nodes |
| CoreDNS | You (EKS addon) | Your worker nodes |
| VPC CNI (aws-node) | You (EKS addon) | Your worker nodes |
| CSI drivers (EBS, EFS) | You (EKS addon) | Your worker nodes |
| Your workloads | You | Your worker nodes |

## Control Plane Architecture

The EKS control plane runs in an AWS-managed account, not in your VPC. You never see the underlying EC2 instances or etcd storage.

```
┌──────────────────────────────────────────────────────────────────────┐
│                        AWS-MANAGED ACCOUNT                           │
│                     (You never see this)                             │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
│  │ AZ-a         │  │ AZ-b         │  │ AZ-c         │                │
│  │              │  │              │  │              │                │
│  │ API Server   │  │ API Server   │  │ etcd         │                │
│  │ etcd         │  │ etcd         │  │              │                │
│  │ NAT GW       │  │ NAT GW       │  │ NAT GW       │                │
│  │ Scheduler    │  │              │  │              │                │
│  │ CCM          │  │              │  │              │                │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                │
│         │                 │                 │                        │
│   ┌─────┴─────────────────┴─────────────────┴─────┐                  │
│   │             NLB (API endpoint)                │                  |
│   └────────────────────────┬──────────────────────┘                  │
│                            │                                         │
└────────────────────────────┼─────────────────────────────────────────┘
          │ X-ENI            │ X-ENI            │ X-ENI
          │                  │                  │
┌─────────┼──────────────────┼──────────────────┼──────────────────────┐
│         ▼                  ▼                  ▼                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
│  │ Subnet AZ-a  │  │ Subnet AZ-b  │  │ Subnet AZ-c  │                │
│  │              │  │              │  │              │                │
│  │ Node 1       │  │ Node 3       │  │ Node 5       │                │
│  │ Node 2       │  │ Node 4       │  │ Node 6       │                │
│  └──────────────┘  └──────────────┘  └──────────────┘                │
│                                                                      │
│                       YOUR AWS ACCOUNT                               │
│                    (Your VPC, your nodes)                            │
└──────────────────────────────────────────────────────────────────────┘
```

EKS splits into two accounts:
- **AWS-managed account**: Runs the control plane (API servers, etcd, controllers)
- **Your account**: Runs the data plane (worker nodes, pods, your workloads)

The bridge between them: **Cross-Account ENIs (X-ENIs)**.

### Control Plane Details

- **etcd**: Runs across 3 AZs in the AWS-managed account. Encrypted at rest with AWS-managed keys. You cannot access etcd directly.
- **API servers**: At least 2 replicas behind an NLB. Automatically scaled based on load.
- **SLA**: 99.95% uptime for the API server endpoint.
- **Upgrades**: AWS handles control plane patches and upgrades within a minor version. You initiate minor version upgrades.

### Control Plane Component Details

| Component | Instances | Spread | Notes |
|-----------|:---------:|--------|-------|
| kube-apiserver | Minimum 2 | Across distinct AZs | In an Auto Scaling Group |
| etcd | 3 | Across 3 AZs | In an Auto Scaling Group |
| kube-scheduler | Runs on API server nodes | — | Leader-elected |
| kube-controller-manager | Runs on API server nodes | — | Leader-elected |
| cloud-controller-manager | Runs on API server nodes | — | AWS-specific (ELB, routes) |
| NAT Gateway | 1 per AZ | Each AZ | For control plane egress |

### API Server Scaling Modes

EKS supports two control plane modes:

| Mode | Behavior | SLA |
|------|----------|:---:|
| **Standard** (default) | Auto-scales API servers based on load | 99.95% |
| **Provisioned** | Pre-allocated capacity, fixed size | 99.99% |

With Provisioned mode:
- Enhanced SLA measured in 1-minute intervals (vs 5-minute for Standard)
- Capacity is pre-allocated — no cold start during traffic spikes
- Recommended for large or latency-sensitive clusters

### Control Plane Egress Routing

Since 2024, EKS supports two egress modes:

| Mode | How It Works |
|------|-------------|
| **AWS_MANAGED** (default) | API server egress goes through AWS-managed NAT Gateways |
| **CUSTOMER_ROUTED** | API server egress (webhooks, OIDC) goes through X-ENIs in your VPC |

With CUSTOMER_ROUTED, webhook calls from the API server use your VPC's routing — meaning they respect your VPC endpoints, NAT Gateways, and firewall rules.

## API Server Endpoint Access

The API server has two communication paths:

### Public Endpoint (Default)

```
kubectl (internet) ──► NLB (public) ──► API server (AWS VPC)
                                              │
                                              ▼
                                      ENI in your VPC ──► kubelet
```

- API server is reachable from the internet
- Worker nodes communicate via the public endpoint or ENI (depending on configuration)
- Can be restricted by CIDR blocks

### Private Endpoint

```
kubectl (your VPC) ──► ENI ──► API server (AWS VPC)
                                      │
                                      ▼
                               ENI in your VPC ──► kubelet
```

- API server only reachable from within the VPC (or peered VPCs)
- All traffic stays on AWS network
- Requires VPN/Direct Connect for external access

### Configuration

```sh
# Enable private, disable public
aws eks update-cluster-config \
  --name <cluster> \
  --resources-vpc-config endpointPublicAccess=false,endpointPrivateAccess=true

# Public + private (recommended for most setups)
aws eks update-cluster-config \
  --name <cluster> \
  --resources-vpc-config endpointPublicAccess=true,endpointPrivateAccess=true

# Restrict public access to specific CIDRs
aws eks update-cluster-config \
  --name <cluster> \
  --resources-vpc-config endpointPublicAccess=true,publicAccessCidrs="203.0.113.0/24,198.51.100.0/24"

# Check current config
aws eks describe-cluster --name <cluster> \
  --query "cluster.resourcesVpcConfig.{Public:endpointPublicAccess, Private:endpointPrivateAccess, CIDRs:publicAccessCidrs}"
```

## Cross-Account ENIs

When you create an EKS cluster, AWS places Elastic Network Interfaces (ENIs) in your VPC subnets. These are the bridge between the AWS-managed control plane and your worker nodes.

- ENIs are owned by the AWS EKS service account, placed in your cluster subnets
- They appear in your VPC with a description like `Amazon EKS <cluster-name>`
- Security group on these ENIs controls API server → kubelet communication
- The control plane uses these ENIs to:
  - Push commands to kubelets (logs, exec, port-forward)
  - Communicate with admission webhooks running on worker nodes

```sh
# Find the cross-account ENIs
aws ec2 describe-network-interfaces \
  --filters "Name=description,Values=Amazon EKS*" \
  --query "NetworkInterfaces[].{ID:NetworkInterfaceId, Subnet:SubnetId, SG:Groups[].GroupId, Description:Description}" \
  --output table

# Find the cluster security group (attached to ENIs and nodes)
aws eks describe-cluster --name <cluster> \
  --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text
```

## The Cluster Security Group

EKS creates a single security group (the "cluster security group") that is shared between the control plane ENIs and the worker nodes:

```
Control Plane ENI ◄──── cluster SG ────► Worker Nodes
```

Rules (created automatically):
- **Inbound**: All traffic from itself (allows node-to-node and node-to-control-plane)
- **Outbound**: All traffic (allows control plane to reach nodes)

```sh
# Get cluster security group
CLUSTER_SG=$(aws eks describe-cluster --name <cluster> \
  --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)

# View its rules
aws ec2 describe-security-groups --group-ids $CLUSTER_SG \
  --query "SecurityGroups[0].{Inbound:IpPermissions, Outbound:IpPermissionsEgress}" --output json
```

If you use a custom node security group, you must add rules to allow:
- Inbound from cluster SG on port 443 (webhooks)
- Inbound from cluster SG on ports 1025-65535 (kubelet, pods)
- Outbound to cluster SG on port 443 (API server)

## Networking: VPC CNI (aws-node)

The VPC CNI plugin assigns real VPC IP addresses to pods — each pod gets an ENI secondary IP from the subnet:

```
Node (primary ENI: 10.0.1.10)
├── Pod A: 10.0.1.15 (secondary IP on ENI)
├── Pod B: 10.0.1.16 (secondary IP on ENI)
└── Pod C: 10.0.1.17 (secondary IP on ENI)
```

### How It Works

1. Each node gets one or more ENIs attached
2. Each ENI can hold multiple secondary IP addresses
3. Pods are assigned IPs from the node's ENI secondary IPs
4. Pod traffic is routed natively through the VPC — no overlay network

### IP Address Capacity Per Node

The number of pods a node can run is limited by ENI and IP capacity:

```
Max pods = (number of ENIs) × (IPs per ENI - 1) + 2
```

Example for `m5.large` (3 ENIs, 10 IPs per ENI):

```
Max pods = 3 × (10 - 1) + 2 = 29
```

```sh
# Check max pods for an instance type
aws ec2 describe-instance-types --instance-types m5.large \
  --query "InstanceTypes[0].NetworkInfo.{MaxENI:MaximumNetworkInterfaces, IPv4PerENI:Ipv4AddressesPerInterface}" --output table
```

### VPC CNI Configuration

```sh
# Check current CNI settings (on a node)
kubectl get daemonset aws-node -n kube-system -o jsonpath='{.spec.template.spec.containers[0].env}' | jq

# Key environment variables
# WARM_IP_TARGET — how many IPs to pre-allocate
# MINIMUM_IP_TARGET — minimum IPs to maintain
# WARM_ENI_TARGET — how many spare ENIs to keep warm
# ENABLE_PREFIX_DELEGATION — assign /28 prefixes instead of individual IPs (more pods per node)
```

### Prefix Delegation (Higher Pod Density)

With prefix delegation, each ENI slot gets a /28 prefix (16 IPs) instead of 1 IP:

```sh
# Enable prefix delegation
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
kubectl set env daemonset aws-node -n kube-system WARM_PREFIX_TARGET=1
```

This dramatically increases pods per node (e.g., m5.large goes from 29 to 110 pods).

### Common Instance Types: Max Pods

| Instance Type | Max ENIs | IPs per ENI | Max Pods (default) | Max Pods (prefix delegation) |
|:-------------|:--------:|:-----------:|:------------------:|:---------------------------:|
| t3.small | 3 | 4 | 11 | 35 |
| t3.medium | 3 | 6 | 17 | 55 |
| t3.large | 3 | 12 | 35 | 110 |
| m5.large | 3 | 10 | 29 | 110 |
| m5.xlarge | 4 | 15 | 58 | 110 |
| m5.2xlarge | 4 | 15 | 58 | 110 |
| m5.4xlarge | 8 | 30 | 234 | 250 |
| c5.large | 3 | 10 | 29 | 110 |
| c5.2xlarge | 4 | 15 | 58 | 110 |
| r5.xlarge | 4 | 15 | 58 | 110 |

### IP Warm Pool

The VPC CNI pre-allocates IPs to reduce pod startup latency:

| Environment Variable | Default | Purpose |
|---------------------|:-------:|---------|
| `WARM_ENI_TARGET` | 1 | Keep 1 spare ENI attached |
| `WARM_IP_TARGET` | — | Keep N spare IPs allocated |
| `MINIMUM_IP_TARGET` | — | Minimum IPs to maintain |
| `WARM_PREFIX_TARGET` | 1 | Keep N spare prefixes (prefix mode) |

Tradeoff: More warm IPs = faster pod startup, but more IP address consumption.

### Custom Networking

By default, pods get IPs from the same subnet as the node. With custom networking:
- Nodes use one set of subnets (e.g., from your primary CIDR)
- Pods use a different set of subnets (e.g., from a secondary CIDR, 100.64.0.0/16)
- Solves IP exhaustion without re-architecting the VPC

```sh
kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
```

Then define ENIConfig resources per AZ to specify which subnets pods should use.

### Subnet Planning for EKS

EKS uses multiple subnet types for different purposes. Plan CIDR allocations carefully:

| Purpose | Subnet Type | Recommended CIDR | Notes |
|---------|-------------|------------------|-------|
| Worker nodes | Private | Primary VPC CIDR (e.g., 10.0.0.0/16) | Nodes get IPs from these |
| Pods (default) | Same as nodes | Primary VPC CIDR | Pods share the node subnet IPs |
| Pods (custom networking) | Private (secondary CIDR) | 100.64.0.0/16 | RFC 6598 shared address space, not routable outside VPC |
| EKS control plane ENIs | Private | Same subnets as nodes (or dedicated) | 2-4 ENIs placed by EKS |
| Load balancers (public) | Public | Primary VPC CIDR | ALB/NLB public-facing, needs IGW |
| Load balancers (internal) | Private | Primary VPC CIDR | Internal ALB/NLB |
| NAT Gateway | Public | Primary VPC CIDR | For node outbound internet access |

#### Common CIDR Allocations

```
VPC Primary CIDR: 10.0.0.0/16 (65,536 IPs)

├── Public subnets (for LBs, NAT GW)
│   ├── 10.0.0.0/24   (AZ-a) — 251 IPs
│   ├── 10.0.1.0/24   (AZ-b) — 251 IPs
│   └── 10.0.2.0/24   (AZ-c) — 251 IPs
│
├── Private subnets (for nodes + control plane ENIs)
│   ├── 10.0.10.0/23  (AZ-a) — 509 IPs
│   ├── 10.0.12.0/23  (AZ-b) — 509 IPs
│   └── 10.0.14.0/23  (AZ-c) — 509 IPs
│
VPC Secondary CIDR: 100.64.0.0/16 (65,536 IPs) — for pods with custom networking

├── Pod subnets (secondary CIDR)
│   ├── 100.64.0.0/18  (AZ-a) — 16,382 IPs
│   ├── 100.64.64.0/18 (AZ-b) — 16,382 IPs
│   └── 100.64.128.0/18 (AZ-c) — 16,382 IPs
```

#### Why 100.64.0.0/16 for Pods

- RFC 6598 "Carrier-Grade NAT" space — reserved for shared address space
- Not routable on the internet, safe to use internally
- Doesn't conflict with common RFC 1918 ranges (10.x, 172.16.x, 192.168.x)
- Gives you 65,536 pod IPs without consuming your primary CIDR
- Pods can still reach the internet via NAT Gateway (VPC routes handle it)

Other secondary CIDRs that work:
- `198.19.0.0/16` — reserved for benchmarking
- Additional 10.x/172.x ranges if they don't conflict with your network

#### Subnet Sizing Guidelines

| Resource | Minimum Subnet | Recommendation |
|----------|---------------|----------------|
| Nodes (small cluster, < 50 nodes) | /24 (251 IPs) | /23 (509 IPs) |
| Nodes (large cluster, 100+ nodes) | /22 (1,021 IPs) | /21 (2,045 IPs) |
| Pods (without custom networking) | Same as nodes | Use /19 or larger per AZ |
| Pods (with custom networking) | /19 (8,190 IPs) | /18 (16,382 IPs) per AZ |
| Control plane ENIs | Only needs 2-4 IPs | No dedicated subnet needed |
| Public (LBs) | /24 per AZ | Enough for ELB IPs |

#### Why /22 or Larger Instead of /24

A /24 subnet gives you 251 usable IPs. That sounds like plenty, but with EKS and the VPC CNI, IPs are consumed much faster than you expect:

- **Each pod consumes a VPC IP** — not just nodes. A 3-node cluster running 29 pods per node uses 87 pod IPs + 3 node IPs = 90 IPs from a single subnet.
- **Warm IPs are pre-allocated** — the VPC CNI reserves spare IPs for fast pod startup (`WARM_ENI_TARGET=1` means an entire ENI worth of IPs sit unused but allocated).
- **You don't control where pods land** — the Kubernetes scheduler picks nodes based on resource requests, affinity, and taints. It does NOT consider subnet IP availability. A single AZ can end up with most of the pods.
- **DaemonSets consume one IP per node** — kube-proxy, aws-node, CSI drivers, monitoring agents all take pod IPs.
- **Scaling bursts consume IPs instantly** — an HPA scaling from 5 to 50 pods allocates 45 IPs in seconds. If the subnet is nearly full, pods stay in `Pending` with `FailedCreatePodSandBox: insufficient IPs`.

Real-world example with a /24 subnet (251 IPs):
```
Nodes:                    10 IPs (10 nodes)
Node warm ENIs:           90 IPs (10 nodes × 1 spare ENI × ~9 IPs)
Running pods:             ~100 IPs
DaemonSets (5 per node):  50 IPs
────────────────────────────────
Total:                    250 IPs ← subnet is full
```

With a /22 (1,021 IPs) or /21 (2,045 IPs), you have headroom for scaling, warm pools, and unpredictable scheduling distribution.

> **Key insight:** The Kubernetes scheduler optimizes for CPU/memory fit, not IP availability. It will happily schedule 80% of pods into one AZ if that's where the resources are — exhausting that subnet's IPs while other AZ subnets sit empty. Larger subnets protect against this imbalance.

#### Subnet Tags Required by EKS

```sh
# Private subnets (for internal LBs and nodes)
aws ec2 create-tags --resources subnet-xxx \
  --tags Key=kubernetes.io/role/internal-elb,Value=1

# Public subnets (for public-facing LBs)
aws ec2 create-tags --resources subnet-xxx \
  --tags Key=kubernetes.io/role/elb,Value=1

# All cluster subnets (for cluster discovery)
aws ec2 create-tags --resources subnet-xxx \
  --tags Key=kubernetes.io/cluster/<cluster-name>,Value=shared
```

#### Internal IP Ranges Used by EKS

EKS uses several IP ranges for internal cluster networking beyond your VPC CIDRs:

| CIDR | Used By | Configurable | Notes |
|------|---------|:---:|-------|
| `172.20.0.0/16` | Kubernetes Service CIDR (default) | At creation only | ClusterIP Services get IPs from here |
| `10.100.0.0/16` | Kubernetes Service CIDR (alternative) | At creation only | Used if VPC overlaps with 172.20.x |
| `172.20.0.1` | `kubernetes` Service (API server) | No | Always the first IP in the Service CIDR |
| `172.20.0.10` | CoreDNS (`kube-dns` Service) | No | Always the 10th IP in the Service CIDR |
| `169.254.169.254` | IMDS (instance metadata) | No | Link-local, available on every node |
| `169.254.169.253` | VPC DNS resolver | No | Where CoreDNS forwards external queries |
| Your VPC CIDR | Nodes and pods | Yes | Pod IPs are real VPC IPs |
| `100.64.0.0/16` | Pods (custom networking) | Yes | Secondary CIDR for pod subnets |

The Service CIDR is set at cluster creation and **cannot be changed afterward**:

```sh
# Check the cluster's Service CIDR
aws eks describe-cluster --name <cluster> \
  --query "cluster.kubernetesNetworkConfig.serviceIpv4Cidr" --output text

# Verify CoreDNS ClusterIP
kubectl get svc kube-dns -n kube-system -o jsonpath='{.spec.clusterIP}'
# 172.20.0.10 (or 10.100.0.10 depending on your Service CIDR)

# Verify the kubernetes API Service
kubectl get svc kubernetes -o jsonpath='{.spec.clusterIP}'
# 172.20.0.1 (first IP in Service CIDR)
```

> **Important:** If your VPC CIDR overlaps with `172.20.0.0/16`, EKS will use `10.100.0.0/16` as the Service CIDR instead. You can also specify a custom Service CIDR at cluster creation with `--kubernetes-network-config serviceIpv4Cidr=<cidr>`.

#### IP Exhaustion: Warning Signs and Solutions

```sh
# Check available IPs in a subnet
aws ec2 describe-subnets --subnet-ids subnet-xxx \
  --query "Subnets[0].AvailableIpAddressCount" --output text

# Check all subnets used by the cluster
aws ec2 describe-subnets --filters "Name=tag:kubernetes.io/cluster/<cluster>,Values=*" \
  --query "Subnets[].{ID:SubnetId, AZ:AvailabilityZone, Available:AvailableIpAddressCount, CIDR:CidrBlock}" \
  --output table
```

If subnets run low:
1. Add a secondary CIDR to the VPC (`aws ec2 associate-vpc-cidr-block --vpc-id vpc-xxx --cidr-block 100.64.0.0/16`)
2. Create new subnets in the secondary CIDR
3. Enable custom networking to route pod IPs to the new subnets
4. Or: create larger subnets in the primary CIDR and migrate node groups

### kube-proxy Modes

| Mode | How It Works | Scalability | Notes |
|------|-------------|:-----------:|-------|
| **iptables** (default) | Inserts NAT rules for each Service | O(n) per packet | Default in EKS |
| **IPVS** | Uses kernel IPVS load balancer | O(1) per packet | Better for 1000+ services |
| **eBPF (Cilium)** | Kernel-bypass via BPF programs | O(1) per packet | Replace kube-proxy entirely |
| **nftables** | Next-gen iptables replacement | Better than iptables | Available in newer kernels |

At scale (1000+ services), iptables rules become a bottleneck. Consider IPVS or Cilium.

## Node Communication Flow

### Pod-to-Pod (Same Node)

```
Pod A (10.0.1.15) ──► Linux bridge/routing ──► Pod B (10.0.1.16)
```

No network hop — handled by the kernel.

### Pod-to-Pod (Different Nodes)

```
Pod A (10.0.1.15) ──► Node 1 ENI ──► VPC routing ──► Node 2 ENI ──► Pod C (10.0.2.20)
```

Direct VPC routing — no encapsulation, no overlay.

### Pod-to-API Server

```
Pod ──► kube-proxy (ClusterIP) ──► Node ENI ──► VPC ──► API server ENI ──► Control Plane
```

Or via the public endpoint (if private endpoint is disabled).

### kubectl to API Server

```
kubectl ──► DNS (EKS endpoint) ──► NLB ──► API server
```

## etcd: What AWS Manages

- 3 nodes across 3 AZs (within the AWS-managed account)
- Encrypted at rest
- Automated backups (you cannot access or restore them directly)
- Automatic compaction (every 5 minutes) and defragmentation
- Storage: EBS volumes (gp3 or io2, AWS-managed)
- You interact with etcd only through the API server

### etcd Limits That Affect You

| Limit | Value | Impact |
|-------|-------|--------|
| Max object size | 1.5 MB | Large ConfigMaps/Secrets fail |
| Max request size | 1.5 MB | Large apply operations fail |
| Total etcd DB size | ~8 GB | Too many objects = cluster degradation |
| Watch connections | Thousands | Too many controllers can overwhelm the API server |

If etcd exceeds 8 GB:
1. API server starts rejecting writes (`NOSPACE` alarm)
2. The cluster becomes effectively read-only
3. You must delete Kubernetes objects to reduce size
4. Common causes: excessive ConfigMaps, Secrets, CRDs, or events not being garbage collected

```sh
# Check API server metrics for etcd health (if metrics endpoint available)
kubectl get --raw /metrics | grep etcd_db_total_size
kubectl get --raw /metrics | grep apiserver_storage_objects
```

## Add-ons and Their Roles

| Add-on | What It Does | Where It Runs |
|--------|-------------|---------------|
| `vpc-cni` (aws-node) | Assigns VPC IPs to pods | DaemonSet on every node |
| `kube-proxy` | Implements Services (iptables/IPVS rules) | DaemonSet on every node |
| `coredns` | Cluster DNS | Deployment (2+ replicas) |
| `aws-ebs-csi-driver` | EBS volume provisioning | Controller + DaemonSet |
| `aws-efs-csi-driver` | EFS mount provisioning | Controller + DaemonSet |
| `aws-load-balancer-controller` | Provisions ALB/NLB for Ingress/Service | Deployment |

```sh
# List installed add-ons
aws eks list-addons --cluster-name <cluster> --output table

# Check add-on versions
aws eks describe-addon --cluster-name <cluster> --addon-name vpc-cni \
  --query "{Version:addonVersion, Status:status, Health:health}" --output json

# List available add-on versions
aws eks describe-addon-versions --addon-name vpc-cni \
  --query "addons[0].addonVersions[].addonVersion" --output table
```

## Authentication Flow

How kubectl authenticates to EKS:

```
1. kubectl reads ~/.kube/config
2. Finds the exec-based credential plugin: "aws eks get-token"
3. aws eks get-token calls AWS STS (GetCallerIdentity) signed with your IAM creds
4. Returns a bearer token (presigned STS URL, base64-encoded)
5. kubectl sends this token to the API server
6. API server calls the aws-iam-authenticator webhook
7. Webhook validates the token against STS
8. Maps the IAM identity to a Kubernetes user/group (via aws-auth ConfigMap or Access Entries)
9. RBAC evaluates permissions
```

```sh
# Manually get a token (for debugging)
aws eks get-token --cluster-name <cluster>

# The token is valid for 15 minutes
# Token format: k8s-aws-v1.<base64-encoded-presigned-url>
```

### aws-auth ConfigMap vs Access Entries

| Method | Where | EKS Version |
|--------|-------|-------------|
| aws-auth ConfigMap | In the cluster (kube-system namespace) | All versions |
| Access Entries (API) | AWS API (managed by EKS) | 1.23+ with platform version ≥ eks.6 |

Access Entries are the newer, recommended approach — they don't require a ConfigMap that can accidentally be deleted.

### IRSA vs EKS Pod Identity

#### IRSA (IAM Roles for Service Accounts) — Since 2019

```
Pod starts → kubelet mounts projected JWT → AWS SDK calls STS AssumeRoleWithWebIdentity → credentials returned
```

Requires:
- An OIDC provider per cluster (registered in IAM)
- A trust policy on each IAM role referencing the specific cluster's OIDC issuer
- Service account annotations in Kubernetes

#### EKS Pod Identity — Since 2023 (Recommended)

```
Pod starts → webhook injects env vars → AWS SDK calls local agent on node → agent calls EKS API → credentials returned
```

No OIDC provider needed. No per-cluster trust policies.

| Feature | IRSA | Pod Identity |
|---------|------|-------------|
| OIDC provider per cluster | Required | Not needed |
| Trust policy changes per cluster | Required | Not needed |
| Session tags | Manual | Automatic |
| Cross-account | Complex setup | Simpler |
| Maturity | Since 2019, battle-tested | Since 2023, now recommended |

## What Happens During a Cluster Upgrade

```
1. AWS upgrades the control plane (API servers, etcd, controllers)
   - Zero downtime — rolling upgrade behind the NLB
   - You can't use the new K8s APIs until control plane upgrade completes

2. AWS updates the platform version (internal patches)
   - Happens automatically within a minor version
   - No action required

3. YOU upgrade node groups (worker nodes)
   - Triggered separately after control plane upgrade
   - Rolling replacement: new nodes launch → old nodes drain → old nodes terminate
   - Nodes can be up to 2 minor versions behind the control plane
```

```sh
# Check current versions
aws eks describe-cluster --name <cluster> \
  --query "{Version:cluster.kubernetesVersion, Platform:cluster.platformVersion, Status:cluster.status}"

# Check available upgrades
aws eks describe-update --name <cluster> --update-id <update-id>
```

## Control Plane Patching: The Rolling Update Mechanism

### API Server Patching

AWS uses a rolling replacement strategy for API server instances:

1. **Preparation** — New API server instances are launched with the patched version across different AZs
2. **Gradual Replacement** — During patching, kube-apiserver instances are replaced, resulting in different IP addresses returned when resolving the cluster endpoint FQDN. Old instances remain available during transition. DNS propagates new IPs (TTL = 60 seconds).
3. **Completion** — Once all new instances are healthy, old instances are terminated. The cluster endpoint remains stable via the NLB.

### etcd Patching

etcd is updated separately with extra care:

1. **Quorum Maintenance** — Always maintains minimum 3 instances across AZs (quorum = 2)
2. **Leader Election** — Elects a new leader if the current leader is being patched
3. **Data Replication** — Ensures no data loss during patching
4. **Incremental** — One instance at a time to maintain cluster stability

### Potential Brief Disruptions During Patching

The operation is zero-downtime for the API endpoint but briefly disrupts webhook calls. Controllers with aggressive timeouts may need retries.

| Issue | Cause | Duration |
|-------|-------|----------|
| Webhook timeouts | Admission controllers briefly unreachable | Seconds |
| DNS serving stale IPs | Client-side DNS caches | Up to 60s (TTL) |
| Connection resets | Long-lived connections interrupted | Instant |
| API server temporarily unresponsive | During instance cutover | 1-5 seconds |

### Mitigation Strategies

- Implement client-side retry logic in applications and controllers
- Set appropriate timeouts for API calls (not too aggressive)
- Don't cache cluster endpoint IPs — always use DNS resolution
- Handle reconnection scenarios gracefully in custom operators
- Use exponential backoff in custom controllers

### Timeline

- **Patch detection**: AWS notifies via service announcements
- **Automatic application**: Platform versions applied within AWS-determined maintenance windows
- **Typical duration**: 10-15 minutes for patch application
- **Worker node patching**: Separate process, you control the timing

## Control Plane vs Worker Node Patching

| Aspect | Control Plane | Worker Nodes |
|--------|:-------------:|:------------:|
| Responsibility | AWS managed | Your responsibility |
| Automatic | Yes (platform patches) | No (you trigger AMI updates) |
| Downtime | Zero-downtime (rolling) | Requires planning (rolling update) |
| Your action | Monitor notifications | Trigger upgrades |
| Version control | You choose K8s version only | You choose node AMI + K8s version |
| Scheduling | AWS-managed windows | Your schedule |
| Rollback | Not possible | Not possible (replace with old version) |
| PDB respected | N/A (no pods on control plane) | Yes (during node rolling update) |

### What AWS Does Automatically

- Monitors control plane load and scales API server instances
- Patches operating system security updates on control plane
- Replaces unhealthy control plane instances
- Manages etcd backups and replication
- Applies Kubernetes security patches (within platform version)
- Maintains API server availability via NLB
- Handles DNS/networking transitions during patching

### What You Must Do

**For platform version updates (patches):**
- No action required — patches apply automatically
- Monitor for brief webhook timeouts during the window
- Be aware of DNS TTL (60s) for API server endpoint

**For Kubernetes version upgrades (e.g., 1.29 → 1.30):**
- Initiate the upgrade manually
- Upgrade worker nodes separately afterward
- Update add-ons (CoreDNS, VPC CNI, kube-proxy) to compatible versions
- Test in staging first — upgrades cannot be rolled back

### Important Notes

- **No rollback**: Control plane version cannot be downgraded once upgraded
- **Minimum support**: Always maintain within supported Kubernetes versions (standard or extended support)
- **Network assumptions**: Requires sufficient IP addresses in cluster subnets for new API server ENIs
- **Stateful applications**: May experience brief connectivity hiccups during patching — design for resilience

## Pricing: What You're Actually Paying For

| Component | Cost | Notes |
|-----------|:----:|-------|
| EKS control plane (Standard) | $0.10/hour (~$73/month) | Per cluster |
| EKS control plane (Provisioned) | Higher (tiered) | Pre-allocated capacity, 99.99% SLA |
| EC2 nodes | Standard EC2 pricing | Instance type dependent |
| Fargate pods | vCPU + memory per second | No node management |
| Extended support | Additional per cluster/hour | After standard support ends |
| EKS Auto Mode | Includes compute | Managed nodes pricing |
| NAT Gateway (if used) | $0.045/hour + $0.045/GB | Your account NAT |
| Load Balancers | Standard ALB/NLB pricing | — |
| Data transfer (X-ENI) | Free within AZ | Cross-AZ standard rates |
| EKS add-ons | Free | You pay for the EC2 they run on |

> The control plane cost is fixed per cluster. Most of the bill is EC2 instances and data transfer.

## Debugging the Control Plane

You can't SSH into the control plane, but you can:

### Enable Control Plane Logging

```sh
# Enable all log types
aws eks update-cluster-config --name <cluster> \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# Logs go to CloudWatch Logs group: /aws/eks/<cluster-name>/cluster
```

### View Logs in CloudWatch

```sh
# API server logs
aws logs filter-log-events \
  --log-group-name /aws/eks/<cluster>/cluster \
  --log-stream-name-prefix kube-apiserver \
  --start-time $(date -u -d '1 hour ago' '+%s000')

# Authenticator logs (auth failures)
aws logs filter-log-events \
  --log-group-name /aws/eks/<cluster>/cluster \
  --log-stream-name-prefix authenticator \
  --filter-pattern "ERROR"

# Audit logs (who did what)
aws logs filter-log-events \
  --log-group-name /aws/eks/<cluster>/cluster \
  --log-stream-name-prefix kube-apiserver-audit \
  --filter-pattern '"verb":"delete"'
```

### API Server Metrics

```sh
# Check API server health
kubectl get --raw /healthz
kubectl get --raw /livez
kubectl get --raw /readyz

# API server metrics (Prometheus format)
kubectl get --raw /metrics | head -50

# Check API request latency
kubectl get --raw /metrics | grep apiserver_request_duration_seconds

# Check etcd health from API server perspective
kubectl get --raw /metrics | grep etcd_request_duration_seconds

# Watch cache size
kubectl get --raw /metrics | grep apiserver_watch_events_sizes

# Inflight requests
kubectl get --raw /metrics | grep apiserver_current_inflight_requests
```

### Audit Logs with CloudWatch Insights

```
# In CloudWatch Logs Insights for /aws/eks/<cluster>/cluster:

# Find failed API requests
fields @timestamp, verb, requestURI, user.username, responseStatus.code
| filter @logStream like /kube-apiserver-audit/
| filter responseStatus.code >= 400
| sort @timestamp desc
| limit 50

# Find delete operations
fields @timestamp, verb, requestURI, user.username
| filter @logStream like /kube-apiserver-audit/
| filter verb = "delete"
| sort @timestamp desc
| limit 20
```

## What Happens During Cluster Creation

When you run `aws eks create-cluster`, behind the scenes:

1. AWS provisions EC2 instances for API servers (minimum 2, across AZs)
2. AWS provisions EC2 instances for etcd (3, across 3 AZs)
3. AWS creates EBS volumes for etcd data
4. AWS creates NAT Gateways in the control plane VPC (one per AZ)
5. AWS creates Cross-Account ENIs in YOUR subnets (2-4 ENIs)
6. AWS creates a Network Load Balancer for the public API endpoint (if enabled)
7. AWS configures the cluster security group
8. AWS registers an OIDC provider (for IRSA)
9. AWS installs default add-ons (kube-proxy, CoreDNS, VPC CNI)
10. Cluster enters ACTIVE state (~10-15 minutes)

## What Happens When an AZ Goes Down

| Component | Impact | Recovery |
|-----------|--------|----------|
| API server in that AZ | Other AZ(s) serve traffic | Automatic |
| etcd node in that AZ | 2/3 quorum maintained | Automatic (read+write still work) |
| X-ENI in that AZ | Nodes in other AZs still communicate | Nodes in failed AZ lose API access |
| Nodes in that AZ | Pods reschedule (after node timeout) | Depends on PDB and resource availability |

If 2 of 3 etcd nodes go down → cluster becomes read-only until quorum is restored.

## Add-on Internals

### CoreDNS

- Runs as a Deployment (default 2 replicas)
- Serves cluster DNS (`.cluster.local`)
- Forwards external queries to VPC DNS (169.254.169.253)
- At scale: increase replicas, enable NodeLocal DNSCache
- Common issue: 5-second DNS timeout on new nodes (conntrack race condition)

### kube-proxy

- Runs as a DaemonSet on every node
- Programs iptables/IPVS rules for Service → Pod routing
- Watches API server for Service/EndpointSlice changes
- At scale: switch to IPVS mode or replace with Cilium

### VPC CNI (aws-node)

- Runs as a DaemonSet on every node
- Manages ENI attachment and IP allocation
- Communicates with EC2 API (DescribeNetworkInterfaces, AssignPrivateIpAddresses)
- Maintains a local IP pool (WARM_IP_TARGET)
- At scale: uses prefix delegation, custom networking

### AWS Load Balancer Controller

- Runs as a Deployment
- Watches Ingress/Service resources
- Creates ALBs (Ingress) and NLBs (Service type: LoadBalancer)
- Calls EC2, ELBv2, WAFv2, ACM APIs
- Uses pod readiness gates to prevent traffic to unready pods

## Security Architecture

### Network Security Layers

```
Internet
    │
    ▼
┌─────────────────┐
│ NLB/ALB (public)│  ← Security Groups, WAF, Shield
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Node SG         │  ← Node-level Security Group
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Pod SG          │  ← Security Groups for Pods (optional)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Network Policy  │  ← Kubernetes NetworkPolicy (Calico/Cilium)
└─────────────────┘
```

### Security Groups for Pods

EKS can assign individual security groups to pods (not just nodes):
- Requires VPC CNI with `ENABLE_POD_ENI=true`
- Pod gets its own "branch ENI" with its own SG
- Allows pod-level firewall rules using VPC Security Groups
- Tradeoff: reduces max pods per node (branch ENIs consume ENI slots)

### Envelope Encryption (etcd Secrets)

- EKS supports envelope encryption of Kubernetes Secrets using a customer-managed KMS key
- Without it: Secrets are base64-encoded in etcd (not truly encrypted at the application layer)
- With it: Secrets are encrypted with a DEK, which is encrypted with your KMS CMK
- Performance cost: ~few ms per Secret read/write (KMS call)

## Common Failure Modes

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| `Unable to connect to the server` | API server unreachable | Check endpoint access, security groups, X-ENI |
| `etcd NOSPACE` alarm | etcd > 8 GB | Delete old objects, especially events and CRDs |
| Pods stuck in `Pending` | No IPs available | Check subnet CIDR space, VPC CNI logs |
| Node `NotReady` | kubelet can't reach API server | Check X-ENI SG, node SG, NACLs |
| Webhook timeouts | API server can't reach webhook pod | Check egress routing, pod SG |
| `Unauthorized` | Token expired or ConfigMap missing | Check aws-auth / Access Entries |
| Slow `kubectl` | API server under load | Check inflight requests, enable APF |
| Cluster upgrade stuck | Webhook blocking API server | Check ValidatingWebhookConfigurations |

## EKS Auto Mode

EKS Auto Mode (2024+) is a fully managed data plane where AWS manages nodes, scaling, and updates — similar to GKE Autopilot:

| Feature | Standard Mode | Auto Mode |
|---------|--------------|-----------|
| Node management | You (node groups, ASGs) | AWS |
| Scaling | Karpenter or CA (you deploy) | Built-in (Karpenter-based) |
| AMI updates | You trigger rolling updates | Automatic |
| OS patching | You manage | AWS manages |
| Billing | EC2 per-instance | Per-pod compute charges |
| GPU support | Yes | Yes |
| DaemonSets | Yes | Limited |
| Privileged containers | Yes | Restricted |

```sh
# Create an Auto Mode cluster
aws eks create-cluster --name my-cluster \
  --compute-config enabled=true,nodePools=["general-purpose","system"] \
  --kubernetes-network-config elasticLoadBalancing=enabled \
  --storage-config blockStorage=enabled

# Or enable on existing cluster
aws eks update-cluster-config --name <cluster> \
  --compute-config enabled=true
```

Auto Mode is best for teams that want minimal operational burden and are willing to accept restrictions on workload types (similar to Fargate but with more flexibility).

## Karpenter vs Cluster Autoscaler

Both solve node scaling, but they work fundamentally differently:

| Feature | Cluster Autoscaler (CA) | Karpenter |
|---------|------------------------|-----------|
| Scaling unit | Node Group (ASG) | Individual nodes |
| Instance selection | Fixed per node group | Dynamic (picks optimal from all types) |
| Scaling speed | 2-5 minutes (ASG scale-out) | 30-60 seconds (direct EC2 API) |
| Consolidation | No (only scale-down of empty nodes) | Yes (replaces underutilized nodes) |
| Spot management | Per node group | Automatic mixed (spot + on-demand) |
| Disruption handling | Basic (respects PDB) | Advanced (drift, consolidation, expiry) |
| Configuration | Per-ASG annotations | NodePool + EC2NodeClass CRDs |
| Multi-AZ awareness | Via ASG config | Built-in topology spread |
| GPU/accelerator | Separate node groups | Dynamic provisioning based on pod requests |
| Maintained by | Kubernetes SIG | AWS (open source) |

### When to Use Each

**Use Cluster Autoscaler when:**
- Simple, predictable workloads with few instance types
- You need node groups for organizational reasons (teams, environments)
- You're on AKS or GKE (Karpenter is EKS-only currently)

**Use Karpenter when:**
- You want fastest possible scaling (sub-minute)
- Mixed instance types and Spot optimization matter
- Workloads have diverse resource requirements (GPU, high-memory, ARM)
- You want automatic bin-packing and cost optimization
- You're running on EKS

### Karpenter Quick Start

```sh
# Install Karpenter (Helm)
helm install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --namespace kube-system \
  --set clusterName=<cluster> \
  --set clusterEndpoint=$(aws eks describe-cluster --name <cluster> --query "cluster.endpoint" --output text)
```

```yaml
# NodePool: what Karpenter can provision
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand", "spot"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.large", "m5.xlarge", "m5.2xlarge", "c5.large", "c5.xlarge"]
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["us-east-1a", "us-east-1b", "us-east-1c"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: "1000"
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
---
# EC2NodeClass: how nodes are configured
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiSelectorTerms:
    - alias: al2023@latest
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: <cluster>
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: <cluster>
  role: KarpenterNodeRole-<cluster>
```

### Karpenter vs CA: Scaling Timeline

```
Pod goes Pending:

Cluster Autoscaler:
  T+0s   Pod pending
  T+30s  CA detects pending pod (scan interval)
  T+35s  CA decides to scale ASG
  T+40s  ASG launches instance
  T+120s Instance boots, joins cluster
  T+150s Pod scheduled
  Total: ~2.5 minutes

Karpenter:
  T+0s   Pod pending
  T+1s   Karpenter detects pending pod (immediate watch)
  T+2s   Karpenter calls EC2 RunInstances directly
  T+60s  Instance boots, joins cluster
  T+65s  Pod scheduled
  Total: ~1 minute
```

## Limits to Be Aware Of

### Hard Limits

| Resource | Limit | Notes |
|----------|:-----:|-------|
| Nodes per cluster | 100,000 | Raised from 5,000 in July 2025 |
| Pods per node | 250 | Kubernetes upstream limit |
| Pods per node (EKS default) | 110 | With prefix delegation |
| Services per namespace | 5,000 | Kubernetes limit |
| Services per cluster | 10,000 | Kubernetes limit |
| Namespaces per cluster | 10,000 | Practical limit |
| etcd database size | 8 GB | Kubernetes recommended max |
| ConfigMap/Secret size | 1 MB | etcd object size limit |

### Soft Limits (Adjustable via Service Quotas)

| Resource | Default | Notes |
|----------|:-------:|-------|
| Clusters per region | 100 | Can be increased |
| Managed node groups per cluster | 30 | Can be increased |
| Nodes per managed node group | 450 | Can be increased |
| Fargate profiles per cluster | 10 | Can be increased |
| Selectors per Fargate profile | 5 | Can be increased |
| Label pairs per selector | 5 | Hard limit |

### Practical Scaling Thresholds

| Threshold | What to Consider |
|-----------|-----------------|
| 300 nodes | Review API server load, consider API Priority and Fairness |
| 1,000 nodes | Enable Provisioned control plane, reduce watch cardinality |
| 5,000 pods | Review kube-proxy mode, consider IPVS or Cilium |
| 1,000 services | iptables performance degrades, switch to IPVS |
| 100 namespaces | Review RBAC complexity, watch cache memory |
| 4 GB etcd | Review stored objects, enable event TTLs, compact |

## Gotchas

- **You cannot access etcd directly**: No backup/restore, no compaction tuning, no direct queries. Use the Kubernetes API for everything.
- **Control plane upgrades are one-way**: You cannot downgrade a cluster. Test upgrades in a non-production cluster first.
- **Subnet IP exhaustion**: The VPC CNI assigns VPC IPs to pods. A /24 subnet (251 usable IPs) fills up fast with many pods. Plan subnet sizing accordingly.
- **Cross-account ENIs need subnet space**: EKS places 2-4 ENIs in your subnets. Don't use subnets that are nearly full.
- **coredns on Fargate is tricky**: If all nodes are Fargate, coredns needs a Fargate profile matching its namespace/labels.
- **API server throttling**: High-frequency controllers (custom operators, ArgoCD with many apps) can hit API rate limits. Check for 429 responses.
- **Private endpoint DNS**: When private endpoint is enabled, the cluster DNS name resolves to private IPs only from within the VPC. External tools need VPN/Direct Connect.
- **Platform version ≠ Kubernetes version**: AWS patches the platform version independently. Two clusters on the same K8s version may have different platform versions.
