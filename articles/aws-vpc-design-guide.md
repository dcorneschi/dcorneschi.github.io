# AWS VPC Design Guide

Architecture patterns, CIDR planning, subnet strategies, and connectivity options for production VPC deployments.

## CIDR Planning

### VPC CIDR Limits

- Minimum: /28 (16 IPs, 11 usable)
- Maximum: /16 (65,536 IPs)
- Up to 5 CIDRs per VPC (1 primary + 4 secondary)

### Choosing a VPC CIDR

```bash
# RFC 1918 private ranges available for VPCs
10.0.0.0/8       # 16,777,216 IPs — largest block
172.16.0.0/12    # 1,048,576 IPs
192.168.0.0/16   # 65,536 IPs

# AWS limits: VPC CIDR must be between /16 and /28
# Recommended: /16 for production (65,536 IPs)

# Check existing VPCs to avoid overlap
aws ec2 describe-vpcs --query 'Vpcs[].CidrBlock' --output table
```

### Multi-VPC CIDR Strategy

Avoid overlap when peering or connecting via Transit Gateway:

| VPC | Purpose | CIDR | IPs |
|-----|---------|------|-----|
| Production | Workloads | 10.0.0.0/16 | 65,536 |
| Staging | Pre-prod | 10.1.0.0/16 | 65,536 |
| Development | Dev/test | 10.2.0.0/16 | 65,536 |
| Shared Services | DNS, AD, tools | 10.3.0.0/16 | 65,536 |
| On-premises | Corporate DC | 172.16.0.0/12 | Varies |

### Secondary CIDRs

```bash
# Add secondary CIDR to an existing VPC (max 5 CIDRs)
aws ec2 associate-vpc-cidr-block \
    --vpc-id vpc-0123456789abcdef0 \
    --cidr-block 100.64.0.0/16

# Useful for:
# - Running out of IPs
# - EKS pod networking (secondary CIDR for pods)
# - Separating workload types
```

## Subnet Design

### Three-Tier Architecture (Standard)

| Tier | Subnet Type | Purpose | Route |
|------|------------|---------|-------|
| Public | Public | Load balancers, NAT GW, bastion | IGW |
| Private | Private | Application servers, EKS nodes | NAT GW |
| Data | Isolated | RDS, ElastiCache, internal services | No internet |

### Subnet Sizing Per AZ

For a /16 VPC across 3 AZs:

| Subnet | CIDR | IPs Available | Notes |
|--------|------|--------------|-------|
| Public-AZ1 | 10.0.0.0/20 | 4,091 | 5 IPs reserved by AWS |
| Public-AZ2 | 10.0.16.0/20 | 4,091 | |
| Public-AZ3 | 10.0.32.0/20 | 4,091 | |
| Private-AZ1 | 10.0.48.0/19 | 8,187 | Larger for workloads |
| Private-AZ2 | 10.0.80.0/19 | 8,187 | |
| Private-AZ3 | 10.0.112.0/19 | 8,187 | |
| Data-AZ1 | 10.0.144.0/20 | 4,091 | |
| Data-AZ2 | 10.0.160.0/20 | 4,091 | |
| Data-AZ3 | 10.0.176.0/20 | 4,091 | |
| Spare | 10.0.192.0/18 | 16,379 | Future use |

### AWS Reserved IPs (Per Subnet)

AWS reserves 5 IPs in every subnet. For a `10.0.0.0/24` subnet:

| IP | Purpose |
|----|---------|
| 10.0.0.0 | Network address |
| 10.0.0.1 | VPC router |
| 10.0.0.2 | DNS server (base of VPC network range + 2) |
| 10.0.0.3 | Reserved for future use |
| 10.0.0.255 | Broadcast (not supported in VPC, but reserved) |

The DNS server IP is always the base of the primary VPC CIDR + 2. For VPCs with multiple CIDRs, the DNS server is in the primary CIDR. AWS also reserves base + 2 in each subnet range for all CIDR blocks.

A /24 subnet provides 251 usable IPs, not 256.

### List IPs in Use

```bash
# List all private IPs in a subnet
aws ec2 describe-network-interfaces \
    --filters Name=subnet-id,Values=subnet-xxx | \
    jq -r '.NetworkInterfaces[].PrivateIpAddress' | sort
```

## VPC for EKS

### EKS VPC Requirements

```bash
# Minimum: 2 AZs (3 recommended)
# Subnets need sufficient IPs for pods

# EKS uses IPs from node subnets for pods (VPC CNI)
# Each node consumes: 1 IP (node) + N IPs (pods)
# Example: m5.large supports 29 pods = 30 IPs per node

# Calculate subnet size:
# 100 nodes × 30 IPs = 3,000 IPs minimum
# Use /19 subnets (8,187 IPs) for room to grow
```

### EKS Subnet Tagging

```bash
# Public subnets (for external load balancers)
aws ec2 create-tags --resources subnet-xxx \
    --tags Key=kubernetes.io/role/elb,Value=1

# Private subnets (for internal load balancers and nodes)
aws ec2 create-tags --resources subnet-xxx \
    --tags Key=kubernetes.io/role/internal-elb,Value=1

# Cluster ownership tag
aws ec2 create-tags --resources subnet-xxx \
    --tags Key=kubernetes.io/cluster/my-cluster,Value=shared
```

### EKS with Secondary CIDR (100.64.0.0/16 for Pods)

```bash
# Add secondary CIDR for pod networking
aws ec2 associate-vpc-cidr-block \
    --vpc-id vpc-xxx \
    --cidr-block 100.64.0.0/16

# Create subnets from secondary CIDR
aws ec2 create-subnet \
    --vpc-id vpc-xxx \
    --cidr-block 100.64.0.0/19 \
    --availability-zone us-east-1a

# Configure VPC CNI to use custom networking
kubectl set env daemonset aws-node -n kube-system \
    AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true

# Benefits:
# - Node IPs from primary CIDR (routable)
# - Pod IPs from secondary CIDR (not routable externally)
# - Massively increases available pod IPs
```

## Connectivity Patterns

### VPC Peering

```bash
# Create peering connection
aws ec2 create-vpc-peering-connection \
    --vpc-id vpc-111111 \
    --peer-vpc-id vpc-222222 \
    --peer-region us-west-2

# Accept peering (from peer account/region)
aws ec2 accept-vpc-peering-connection \
    --vpc-peering-connection-id pcx-xxx

# Add routes in both VPCs
aws ec2 create-route \
    --route-table-id rtb-111 \
    --destination-cidr-block 10.1.0.0/16 \
    --vpc-peering-connection-id pcx-xxx
```

**Limitations:**
- Not transitive (A↔B and B↔C doesn't mean A↔C)
- Max 125 peering connections per VPC
- No overlapping CIDRs
- No edge-to-edge routing through peering

### Transit Gateway

```bash
# Create Transit Gateway
aws ec2 create-transit-gateway \
    --description "Central hub" \
    --options "AmazonSideAsn=64512,AutoAcceptSharedAttachments=enable,DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable,DnsSupport=enable"

# Attach VPC
aws ec2 create-transit-gateway-vpc-attachment \
    --transit-gateway-id tgw-xxx \
    --vpc-id vpc-xxx \
    --subnet-ids subnet-aaa subnet-bbb subnet-ccc

# Add route pointing to TGW
aws ec2 create-route \
    --route-table-id rtb-xxx \
    --destination-cidr-block 10.0.0.0/8 \
    --transit-gateway-id tgw-xxx
```

**Use Transit Gateway when:**
- Connecting 3+ VPCs
- Need transitive routing
- Connecting to on-premises via VPN/Direct Connect
- Centralized egress/ingress

### VPN Connections

```bash
# Create Customer Gateway (your on-prem device)
aws ec2 create-customer-gateway \
    --type ipsec.1 \
    --public-ip 203.0.113.1 \
    --bgp-asn 65000

# Create Virtual Private Gateway
aws ec2 create-vpn-gateway --type ipsec.1

# Attach VGW to VPC
aws ec2 attach-vpn-gateway \
    --vpn-gateway-id vgw-xxx \
    --vpc-id vpc-xxx

# Create VPN connection
aws ec2 create-vpn-connection \
    --type ipsec.1 \
    --customer-gateway-id cgw-xxx \
    --vpn-gateway-id vgw-xxx \
    --options "{\"StaticRoutesOnly\":false}"
```

### VPC Endpoints

```bash
# Gateway endpoint (S3, DynamoDB) — free
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-xxx \
    --service-name com.amazonaws.us-east-1.s3 \
    --route-table-ids rtb-xxx

# Interface endpoint (most other services) — per-hour + data cost
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-xxx \
    --vpc-endpoint-type Interface \
    --service-name com.amazonaws.us-east-1.ecr.api \
    --subnet-ids subnet-xxx \
    --security-group-ids sg-xxx

# Common endpoints for private EKS clusters
# com.amazonaws.<region>.ecr.api
# com.amazonaws.<region>.ecr.dkr
# com.amazonaws.<region>.s3 (gateway)
# com.amazonaws.<region>.ec2
# com.amazonaws.<region>.sts
# com.amazonaws.<region>.logs
# com.amazonaws.<region>.elasticloadbalancing
```

## NAT Gateway Patterns

### Single NAT Gateway (Cost-Optimized)

```bash
# One NAT GW in one AZ — all private subnets route through it
aws ec2 create-nat-gateway \
    --subnet-id subnet-public-az1 \
    --allocation-id eipalloc-xxx

# Route from all private route tables
aws ec2 create-route \
    --route-table-id rtb-private \
    --destination-cidr-block 0.0.0.0/0 \
    --nat-gateway-id nat-xxx
```

**Trade-off:** Single point of failure. If AZ1 goes down, all private subnets lose internet.

### HA NAT Gateway (One Per AZ)

```bash
# NAT GW in each AZ — private subnets use their local NAT
# AZ1
aws ec2 create-nat-gateway --subnet-id subnet-public-az1 --allocation-id eipalloc-1
# AZ2
aws ec2 create-nat-gateway --subnet-id subnet-public-az2 --allocation-id eipalloc-2
# AZ3
aws ec2 create-nat-gateway --subnet-id subnet-public-az3 --allocation-id eipalloc-3

# Each private subnet routes to its AZ's NAT GW
# Costs 3× but survives AZ failures
```

## Security Design

### Security Groups Strategy

```bash
# Layered security groups (reference other SGs, not CIDRs)

# ALB security group
aws ec2 create-security-group --group-name alb-sg --description "ALB" --vpc-id vpc-xxx
aws ec2 authorize-security-group-ingress --group-id sg-alb \
    --protocol tcp --port 443 --cidr 0.0.0.0/0

# App security group (only from ALB)
aws ec2 create-security-group --group-name app-sg --description "App" --vpc-id vpc-xxx
aws ec2 authorize-security-group-ingress --group-id sg-app \
    --protocol tcp --port 8080 --source-group sg-alb

# DB security group (only from App)
aws ec2 create-security-group --group-name db-sg --description "DB" --vpc-id vpc-xxx
aws ec2 authorize-security-group-ingress --group-id sg-db \
    --protocol tcp --port 5432 --source-group sg-app
```

### Network ACLs (Stateless)

```bash
# NACLs are stateless — need both inbound and outbound rules
# Use sparingly (security groups are usually sufficient)

# Deny specific IP ranges (e.g., known malicious)
aws ec2 create-network-acl-entry \
    --network-acl-id acl-xxx \
    --rule-number 50 \
    --protocol -1 \
    --rule-action deny \
    --ingress \
    --cidr-block 203.0.113.0/24
```

## VPC Flow Logs

```bash
# Enable flow logs to CloudWatch
aws ec2 create-flow-log \
    --resource-type VPC \
    --resource-id vpc-xxx \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name /aws/vpc/flowlogs \
    --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flowlogsRole

# Enable flow logs to S3
aws ec2 create-flow-log \
    --resource-type VPC \
    --resource-id vpc-xxx \
    --traffic-type ALL \
    --log-destination-type s3 \
    --log-destination arn:aws:s3:::my-flowlogs-bucket/vpc-logs/

# Query flow logs with Athena or CloudWatch Insights
# Useful fields: srcaddr, dstaddr, srcport, dstport, protocol, action
```

## Common Architecture Patterns

### Pattern 1: Simple Web App

```
Internet → IGW → ALB (public subnet)
                   ↓
              App servers (private subnet)
                   ↓
              RDS (isolated subnet)
                   ↓
              NAT GW → Internet (for updates)
```

### Pattern 2: Multi-Account with Transit Gateway

```
Production VPC (10.0.0.0/16) ──┐
Staging VPC (10.1.0.0/16) ─────┼── Transit Gateway ── Direct Connect ── On-premises
Shared Services (10.3.0.0/16) ─┘
```

### Pattern 3: Private EKS Cluster

```
Private subnets: EKS nodes + pods
No IGW attached
VPC Endpoints: ECR, S3, STS, EC2, ELB, CloudWatch
NAT GW: Only if pods need outbound internet
API endpoint: Private (accessible only within VPC or via VPN)
```

## Cost Optimization

| Resource | Cost | Tip |
|----------|------|-----|
| NAT Gateway | ~$32/month + $0.045/GB | Use single NAT GW in non-prod |
| Interface Endpoints | ~$7.20/month per AZ | Share across VPCs via TGW |
| Transit Gateway | $0.05/hour per attachment | Cheaper than many peering connections |
| VPN Connection | ~$36/month | Consider Direct Connect for high throughput |
| Elastic IP (unattached) | $3.60/month | Release unused EIPs |
| IPv4 Public IP | $3.60/month per IP (since Feb 2024) | Use private IPs where possible |

## Troubleshooting

### Cannot Reach Internet from Private Subnet

```bash
# Check route table has NAT GW route
aws ec2 describe-route-tables --route-table-id rtb-xxx \
    --query 'RouteTables[].Routes[?DestinationCidrBlock==`0.0.0.0/0`]'

# Check NAT GW is in a public subnet with IGW route
aws ec2 describe-nat-gateways --nat-gateway-ids nat-xxx

# Check security group allows outbound
aws ec2 describe-security-groups --group-ids sg-xxx \
    --query 'SecurityGroups[].IpPermissionsEgress'

# Check NACL allows traffic
aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=subnet-xxx
```

### VPC Peering Not Working

```bash
# Check peering status is "active"
aws ec2 describe-vpc-peering-connections --vpc-peering-connection-ids pcx-xxx

# Check routes exist in BOTH VPCs
aws ec2 describe-route-tables --filters Name=route.vpc-peering-connection-id,Values=pcx-xxx

# Check security groups allow traffic from peer CIDR
# Check NACLs allow traffic from peer CIDR
# Verify no overlapping CIDRs
```

### Running Out of IPs

```bash
# Check available IPs per subnet
aws ec2 describe-subnets --subnet-ids subnet-xxx \
    --query 'Subnets[].AvailableIpAddressCount'

# Options:
# 1. Add secondary CIDR
# 2. Create larger subnets
# 3. Use prefix delegation (EKS VPC CNI)
# 4. Clean up unused ENIs
aws ec2 describe-network-interfaces \
    --filters Name=status,Values=available \
    --query 'NetworkInterfaces[].NetworkInterfaceId'
```

## IPv4 Subnet Reference Table

| Prefix | Network Mask | Usable Hosts |
|--------|-------------|:------------:|
| /16 | 255.255.0.0 | 65,534 |
| /17 | 255.255.128.0 | 32,766 |
| /18 | 255.255.192.0 | 16,382 |
| /19 | 255.255.224.0 | 8,190 |
| /20 | 255.255.240.0 | 4,094 |
| /21 | 255.255.248.0 | 2,046 |
| /22 | 255.255.252.0 | 1,022 |
| /23 | 255.255.254.0 | 510 |
| /24 | 255.255.255.0 | 254 |
| /25 | 255.255.255.128 | 126 |
| /26 | 255.255.255.192 | 62 |
| /27 | 255.255.255.224 | 30 |
| /28 | 255.255.255.240 | 14 |

> **Note:** AWS reserves 5 IPs per subnet, so actual usable IPs in AWS = usable hosts - 5. For example, a /24 in AWS gives 251 usable IPs (254 - 5 reserved + 2 adjustment = 251).

### AWS-Specific Usable IPs

| Prefix | Standard Usable | AWS Usable (minus 5 reserved) |
|--------|:--------------:|:-----------------------------:|
| /16 | 65,534 | 65,531 |
| /20 | 4,094 | 4,091 |
| /24 | 254 | 251 |
| /25 | 126 | 123 |
| /26 | 62 | 59 |
| /27 | 30 | 27 |
| /28 | 14 | 11 |
