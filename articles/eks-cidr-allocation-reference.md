# AWS EKS — CIDR Allocation Reference

A planning guide for IP address allocation in EKS clusters — VPC CIDRs, subnet sizing, pod networking, and strategies to avoid IP exhaustion.

## CIDR Summary

| Component | Default CIDR | Key IPs | Notes |
|-----------|-------------|---------|-------|
| **VPC (Primary)** | `10.0.0.0/16` | `10.0.0.1` – `10.0.255.254` | Nodes + Pods use this range |
| **VPC (Secondary)** | `100.64.0.0/16` | `100.64.0.1` – `100.64.255.254` | Custom networking (RFC 6598) |
| **Pod Network** | Same as VPC subnet | 1 IP per pod from subnet | VPC CNI — real routable IPs |
| **Service CIDR** | `10.100.0.0/16` | `10.100.0.1` (API Server) | Alt default: `172.20.0.0/16` |
| **CoreDNS (kube-dns)** | Within Service CIDR | `10.100.0.10` | Always 10th IP in service range |
| **API Server Endpoint** | Public/Private ENI | ELB or ENI in VPC | Managed by AWS control plane |

## How It Works

### Pod Networking (VPC CNI)

- EKS uses the **Amazon VPC CNI plugin** — pods get real VPC IP addresses (no overlay network)
- Each pod is assigned a private IPv4 address from the node's subnet
- IPs are allocated via Elastic Network Interfaces (ENIs) attached to EC2 nodes
- Pod IPs are directly routable within the VPC

### Service CIDR (ClusterIP)

- Kubernetes assigns **virtual IPs** from the service CIDR to `ClusterIP` services
- Default range: `10.100.0.0/16` (or `172.20.0.0/16` depending on VPC primary CIDR)
- These IPs are **not routable** outside the cluster — handled by iptables/IPVS on each node
- Set at cluster creation time and **cannot be changed** afterwards

### Well-Known Service IPs

| Service | ClusterIP | Purpose |
|---------|-----------|---------|
| `kubernetes` | `10.100.0.1` | Kubernetes API server (first IP in range) |
| `kube-dns` | `10.100.0.10` | CoreDNS service (10th IP in range) |

### CoreDNS

- Deployed as 2 replicas by default (regardless of node count)
- Service exposed on `10.100.0.10` (ports 53/UDP, 53/TCP, 9153/TCP for metrics)
- Pod IPs come from the VPC subnet (like any other pod)
- The kubelet on each node is configured with `--cluster-dns=10.100.0.10`

## IP Address Consumers in EKS

Every EKS cluster consumes IPs from multiple sources:

| Consumer | Where IPs Come From | Count |
|----------|-------------------|-------|
| Nodes (EC2 instances) | Node subnets | 1 per node (primary IP) |
| Pods | Pod subnets (or node subnets if not separated) | 1 per pod (VPC CNI default) |
| ENI secondary IPs | Node subnets | Varies by instance type |
| Load balancers (ALB/NLB) | Public or private subnets | 1+ per AZ per LB |
| EKS control plane ENIs | Cluster subnets | 2–4 cross-account ENIs |
| Services (ClusterIP) | Service CIDR (virtual, not from VPC) | 1 per Service |

> **Key insight:** Pods are the biggest IP consumer. With VPC CNI, each pod gets a real VPC IP. A cluster running 500 pods needs 500 IPs from VPC subnets.

## VPC CIDR Planning

### RFC 1918 Private Ranges

| Range | CIDR | Available IPs |
|-------|------|---------------|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | 16,777,216 |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | 1,048,576 |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | 65,536 |

### RFC 6598 Shared Address Space (for pods)

| Range | CIDR | Available IPs |
|-------|------|---------------|
| 100.64.0.0 – 100.127.255.255 | 100.64.0.0/10 | 4,194,304 |

> Use `100.64.0.0/10` as a secondary CIDR for pod subnets. These IPs are not routable on the internet and avoid conflicts with corporate RFC 1918 ranges.

### VPC CIDR Limits

- Primary CIDR: /16 to /28
- Secondary CIDRs: up to 4 additional (5 total)
- Cannot overlap with existing CIDRs in the VPC or peered VPCs

## Subnet Sizing Guide

### How Many IPs per Subnet?

AWS reserves 5 IPs per subnet (first 4 + last). Usable = total - 5.

| Subnet CIDR | Total IPs | Usable IPs | Good For |
|-------------|-----------|------------|----------|
| /28 | 16 | 11 | Tiny (control plane ENIs only) |
| /27 | 32 | 27 | Very small |
| /26 | 64 | 59 | Small node groups |
| /25 | 128 | 123 | Small-medium |
| /24 | 256 | 251 | Medium clusters (nodes) |
| /22 | 1,024 | 1,019 | Large clusters (nodes) |
| /20 | 4,096 | 4,091 | Pod subnets (recommended per AZ) |
| /19 | 8,192 | 8,187 | Large pod subnets |
| /18 | 16,384 | 16,379 | Very large pod subnets |
| /16 | 65,536 | 65,531 | Full VPC primary CIDR |

### AWS Reserved IPs (5 per subnet)

For example, in a `10.0.1.0/24` subnet:

| IP Address | Purpose |
|------------|---------|
| 10.0.1.0 | Network address |
| 10.0.1.1 | VPC router |
| 10.0.1.2 | DNS server |
| 10.0.1.3 | Reserved for future use |
| 10.0.1.255 | Broadcast address |

### Finding Used IPs in a Subnet

```bash
# List all network interfaces in a specific subnet
aws ec2 describe-network-interfaces \
  --filters "Name=subnet-id,Values=subnet-xxxxxxxx" \
  --query "NetworkInterfaces[].{IP:PrivateIpAddress,Description:Description,Status:Status}" \
  --output table
```

```bash
# Get the available IP count for a subnet
aws ec2 describe-subnets \
  --subnet-ids subnet-xxxxxxxx \
  --query "Subnets[].{CIDR:CidrBlock,AvailableIPs:AvailableIpAddressCount}" \
  --output table
```

> **EKS tip:** ENIs managed by EKS typically include `amazon-eks` or the node name in the description. Filter by that to see EKS-specific IP usage.

### Recommended Layout (3 AZs)

```
VPC Primary CIDR: 10.0.0.0/16 (65,536 IPs)

├── Public Subnets (for ALBs, NAT Gateways)
│   ├── 10.0.0.0/24   (us-east-1a)   — 251 usable
│   ├── 10.0.1.0/24   (us-east-1b)   — 251 usable
│   └── 10.0.2.0/24   (us-east-1c)   — 251 usable
│
├── Private Node Subnets (for EC2 instances)
│   ├── 10.0.10.0/22  (us-east-1a)   — 1,019 usable
│   ├── 10.0.14.0/22  (us-east-1b)   — 1,019 usable
│   └── 10.0.18.0/22  (us-east-1c)   — 1,019 usable
│
└── Secondary CIDR: 100.64.0.0/16 (for pods)
    ├── 100.64.0.0/18  (us-east-1a)  — 16,379 usable
    ├── 100.64.64.0/18 (us-east-1b)  — 16,379 usable
    └── 100.64.128.0/18 (us-east-1c) — 16,379 usable
```

## Pod IP Allocation: VPC CNI Modes

### Secondary IP Mode (Default)

Each pod gets an individual IP (/32) from the node's ENI secondary IPs.

```
Max pods per node = (ENIs × (IPs per ENI - 1)) + 2
```

Example: `m5.large` has 3 ENIs × 10 IPs each = (3 × 9) + 2 = **29 pods max**.

### Prefix Delegation Mode

Each ENI gets /28 prefixes (16 IPs each) instead of individual IPs. Significantly increases pod density.

```
Max pods per node = (ENIs × (prefixes per ENI) × 16) + 2
```

Example: `m5.large` with prefix delegation = (3 × 9 × 16) + 2 = **434 theoretical** (capped at 110 by default `max-pods`).

Enable prefix delegation:

```bash
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
kubectl set env daemonset aws-node -n kube-system WARM_PREFIX_TARGET=1
```

> **EKS Auto Mode** enables prefix delegation by default.

### Custom Networking (Separate Pod Subnets)

Pods use a different subnet than nodes. Useful when node subnets are small but you have large dedicated pod subnets.

```bash
kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORKING=true
```

Then create `ENIConfig` resources per AZ:

```yaml
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-east-1a
spec:
  subnet: subnet-0abc123pod-a
  securityGroups:
  - sg-0abc123
```

## EKS Service CIDR

The Kubernetes Service CIDR is separate from VPC — it's a virtual range used only for ClusterIP Services.

- Default: `10.100.0.0/16` or `172.20.0.0/16`
- Configurable at cluster creation (cannot change later)
- Must not overlap with your VPC CIDR or peered VPCs

```bash
# Set custom service CIDR at cluster creation
eksctl create cluster --name my-cluster --service-cidr 172.20.0.0/16
```

## IP Exhaustion: Signs and Fixes

### Signs of IP Exhaustion

```bash
# Pods stuck in Pending with this event:
# "failed to assign an IP address to container"

# Check available IPs in subnets
aws ec2 describe-subnets --subnet-ids subnet-abc123 \
  --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,Available:AvailableIpAddressCount,CIDR:CidrBlock}'

# Check VPC CNI metrics
kubectl get pods -n kube-system -l k8s-app=aws-node -o wide
kubectl logs -n kube-system -l k8s-app=aws-node --tail=20 | grep -i "ip"
```

### Fixes

| Problem | Solution |
|---------|----------|
| Pod subnet running out of IPs | Add a secondary CIDR, create larger pod subnets |
| Low pod density per node | Enable prefix delegation |
| Pods consuming node subnet IPs | Enable custom networking with dedicated pod subnets |
| Too many ENI secondary IPs warming | Tune `WARM_IP_TARGET` and `MINIMUM_IP_TARGET` |
| Small subnets | Migrate to larger subnets (or add secondary CIDR) |

### VPC CNI Environment Variables

```bash
# View current settings
kubectl get daemonset aws-node -n kube-system -o json | jq '.spec.template.spec.containers[0].env'

# Key variables
WARM_IP_TARGET=1                    # Keep 1 spare IP ready (reduces waste)
MINIMUM_IP_TARGET=3                 # Always have at least 3 IPs available
WARM_PREFIX_TARGET=1                # Keep 1 spare /28 prefix (prefix delegation)
ENABLE_PREFIX_DELEGATION=true       # Use /28 prefixes instead of individual IPs
AWS_VPC_K8S_CNI_CUSTOM_NETWORKING=true  # Pods use different subnets than nodes
```

## Instance Type IP Limits

| Instance Type | ENIs | IPs per ENI | Max Pods (Secondary IP) | Max Pods (Prefix Delegation) |
|--------------|------|-------------|------------------------|------------------------------|
| t3.medium | 3 | 6 | 17 | 110 |
| m5.large | 3 | 10 | 29 | 110 |
| m5.xlarge | 4 | 15 | 58 | 110 |
| m5.2xlarge | 4 | 15 | 58 | 110 |
| m5.4xlarge | 8 | 30 | 234 | 110 |
| c5.4xlarge | 8 | 30 | 234 | 110 |
| r5.2xlarge | 4 | 15 | 58 | 110 |

> The "110" cap with prefix delegation is the default `max-pods` setting. You can increase it via kubelet `--max-pods` flag, but consider node resource pressure.

```bash
# Check max pods for an instance type
aws ec2 describe-instance-types --instance-types m5.large \
  --query 'InstanceTypes[].NetworkInfo.{ENIs:MaximumNetworkInterfaces,IPv4PerENI:Ipv4AddressesPerInterface}'
```

## Planning Checklist

- [ ] Choose VPC primary CIDR (avoid overlaps with other VPCs, on-premises, peered networks)
- [ ] Add secondary CIDR for pod subnets if primary is small
- [ ] Size node subnets: at least /22 per AZ for growth
- [ ] Size pod subnets: at least /20 per AZ (or /18 for large clusters)
- [ ] Decide on prefix delegation (recommended for >30 pods/node)
- [ ] Set Service CIDR at cluster creation (can't change later)
- [ ] Tag subnets for EKS discovery (`kubernetes.io/cluster/<name>: shared`)
- [ ] Tag public subnets for ELB (`kubernetes.io/role/elb: 1`)
- [ ] Tag private subnets for internal ELB (`kubernetes.io/role/internal-elb: 1`)
- [ ] Monitor subnet IP availability with CloudWatch or AWS CLI

## Useful Commands

```bash
# Check available IPs per subnet
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-abc123" \
  --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Available:AvailableIpAddressCount}' \
  --output table

# List VPC CIDRs (primary + secondary)
aws ec2 describe-vpcs --vpc-ids vpc-abc123 \
  --query 'Vpcs[].CidrBlockAssociationSet[].CidrBlock'

# Check ENI limits for your instance type
aws ec2 describe-instance-types --instance-types m5.xlarge \
  --query 'InstanceTypes[].NetworkInfo.{MaxENIs:MaximumNetworkInterfaces,IPv4PerENI:Ipv4AddressesPerInterface}'

# Check VPC CNI version and config
kubectl describe daemonset aws-node -n kube-system | grep Image
kubectl get daemonset aws-node -n kube-system -o json | jq '.spec.template.spec.containers[0].env[] | select(.name | test("PREFIX|WARM|CUSTOM"))'

# Check current pod IP allocation on a node
kubectl get pods -A --field-selector spec.nodeName=<node-name> -o wide | wc -l
```
