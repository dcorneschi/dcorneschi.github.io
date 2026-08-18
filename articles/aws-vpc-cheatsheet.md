# AWS VPC Cheatsheet

Amazon Virtual Private Cloud (VPC) — subnets, route tables, internet/NAT gateways, security groups, NACLs, VPC endpoints, peering, and Transit Gateway.

## VPC Basics

```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=my-vpc}]'

# List VPCs
aws ec2 describe-vpcs --query 'Vpcs[].{ID:VpcId,CIDR:CidrBlock,Name:Tags[?Key==`Name`].Value|[0]}' --output table

# Enable DNS hostnames
aws ec2 modify-vpc-attribute --vpc-id vpc-123 --enable-dns-hostnames

# Enable DNS resolution
aws ec2 modify-vpc-attribute --vpc-id vpc-123 --enable-dns-support

# Delete VPC
aws ec2 delete-vpc --vpc-id vpc-123
```

## Subnets

```bash
# Create public subnet
aws ec2 create-subnet \
  --vpc-id vpc-123 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone eu-west-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a}]'

# Create private subnet
aws ec2 create-subnet \
  --vpc-id vpc-123 \
  --cidr-block 10.0.10.0/24 \
  --availability-zone eu-west-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1a}]'

# Enable auto-assign public IP (public subnets)
aws ec2 modify-subnet-attribute --subnet-id subnet-aaa --map-public-ip-on-launch

# List subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-123" \
  --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Name:Tags[?Key==`Name`].Value|[0]}' --output table

# Delete subnet
aws ec2 delete-subnet --subnet-id subnet-aaa
```

## Internet Gateway

```bash
# Create and attach IGW
aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=my-igw}]'
aws ec2 attach-internet-gateway --internet-gateway-id igw-123 --vpc-id vpc-123

# Detach and delete
aws ec2 detach-internet-gateway --internet-gateway-id igw-123 --vpc-id vpc-123
aws ec2 delete-internet-gateway --internet-gateway-id igw-123
```

## NAT Gateway

```bash
# Allocate Elastic IP for NAT
aws ec2 allocate-address --domain vpc

# Create NAT gateway in public subnet
aws ec2 create-nat-gateway \
  --subnet-id subnet-public \
  --allocation-id eipalloc-123 \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-1a}]'

# List NAT gateways
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=vpc-123" \
  --query 'NatGateways[].{ID:NatGatewayId,State:State,Subnet:SubnetId,IP:NatGatewayAddresses[0].PublicIp}' --output table

# Delete NAT gateway
aws ec2 delete-nat-gateway --nat-gateway-id nat-123
```

## Route Tables

```bash
# Create route table
aws ec2 create-route-table --vpc-id vpc-123 --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]'

# Add route to IGW (public subnets)
aws ec2 create-route --route-table-id rtb-123 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-123

# Add route to NAT gateway (private subnets)
aws ec2 create-route --route-table-id rtb-456 --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-123

# Associate route table with subnet
aws ec2 associate-route-table --route-table-id rtb-123 --subnet-id subnet-aaa

# List routes
aws ec2 describe-route-tables --route-table-ids rtb-123

# Delete route
aws ec2 delete-route --route-table-id rtb-123 --destination-cidr-block 0.0.0.0/0
```

## Security Groups

```bash
# Create security group
aws ec2 create-security-group \
  --group-name web-sg \
  --description "Web server security group" \
  --vpc-id vpc-123

# Allow inbound HTTP/HTTPS
aws ec2 authorize-security-group-ingress --group-id sg-123 --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-123 --protocol tcp --port 443 --cidr 0.0.0.0/0

# Allow inbound SSH from specific IP
aws ec2 authorize-security-group-ingress --group-id sg-123 --protocol tcp --port 22 --cidr 10.0.0.0/8

# Allow traffic from another security group
aws ec2 authorize-security-group-ingress --group-id sg-456 --protocol tcp --port 5432 --source-group sg-123

# List security groups
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=vpc-123" \
  --query 'SecurityGroups[].{ID:GroupId,Name:GroupName}' --output table

# List rules for a security group
aws ec2 describe-security-group-rules --filters "Name=group-id,Values=sg-123"

# Revoke a rule
aws ec2 revoke-security-group-ingress --group-id sg-123 --protocol tcp --port 22 --cidr 0.0.0.0/0

# Delete security group
aws ec2 delete-security-group --group-id sg-123
```

## Network ACLs

```bash
# Create NACL
aws ec2 create-network-acl --vpc-id vpc-123

# Add inbound rule (allow HTTP)
aws ec2 create-network-acl-entry \
  --network-acl-id acl-123 \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow \
  --ingress

# Add outbound rule (allow all)
aws ec2 create-network-acl-entry \
  --network-acl-id acl-123 \
  --rule-number 100 \
  --protocol -1 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow \
  --egress

# Associate with subnet
aws ec2 replace-network-acl-association --association-id aclassoc-123 --network-acl-id acl-123
```

### Security Groups vs NACLs

| Feature | Security Group | NACL |
|---------|---------------|------|
| Level | Instance (ENI) | Subnet |
| State | Stateful (return traffic auto-allowed) | Stateless (must allow both directions) |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules evaluated | Rules evaluated in order (lowest number first) |
| Default | Deny all inbound, allow all outbound | Allow all (default NACL) |

## VPC Endpoints

### Gateway Endpoints (S3, DynamoDB)

```bash
# Create gateway endpoint for S3
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-123 \
  --service-name com.amazonaws.eu-west-1.s3 \
  --route-table-ids rtb-123 rtb-456

# Create gateway endpoint for DynamoDB
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-123 \
  --service-name com.amazonaws.eu-west-1.dynamodb \
  --route-table-ids rtb-123
```

### Interface Endpoints (PrivateLink)

```bash
# Create interface endpoint (e.g., for ECR)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-123 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.eu-west-1.ecr.api \
  --subnet-ids subnet-aaa subnet-bbb \
  --security-group-ids sg-123 \
  --private-dns-enabled

# Common interface endpoints for EKS/ECS
# com.amazonaws.<region>.ecr.api
# com.amazonaws.<region>.ecr.dkr
# com.amazonaws.<region>.logs
# com.amazonaws.<region>.sts
# com.amazonaws.<region>.ssm
# com.amazonaws.<region>.ssmmessages
# com.amazonaws.<region>.ec2messages
```

```bash
# List endpoints
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=vpc-123" \
  --query 'VpcEndpoints[].{ID:VpcEndpointId,Service:ServiceName,Type:VpcEndpointType,State:State}' --output table

# Delete endpoint
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids vpce-123
```

## VPC Peering

```bash
# Create peering request
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-111 \
  --peer-vpc-id vpc-222 \
  --peer-region eu-west-1

# Accept peering (from the other account/VPC)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-123

# Add routes for peering
aws ec2 create-route --route-table-id rtb-111 --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id pcx-123
aws ec2 create-route --route-table-id rtb-222 --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id pcx-123

# List peering connections
aws ec2 describe-vpc-peering-connections

# Delete peering
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id pcx-123
```

## Transit Gateway

```bash
# Create Transit Gateway
aws ec2 create-transit-gateway --description "Central hub" \
  --options "AmazonSideAsn=64512,AutoAcceptSharedAttachments=enable,DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable,DnsSupport=enable"

# Attach VPC
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-123 \
  --vpc-id vpc-111 \
  --subnet-ids subnet-aaa subnet-bbb

# List attachments
aws ec2 describe-transit-gateway-attachments --filters "Name=transit-gateway-id,Values=tgw-123"

# Add route
aws ec2 create-transit-gateway-route \
  --transit-gateway-route-table-id tgw-rtb-123 \
  --destination-cidr-block 10.2.0.0/16 \
  --transit-gateway-attachment-id tgw-attach-123
```

## VPC Flow Logs

```bash
# Enable flow logs to CloudWatch
aws ec2 create-flow-log \
  --resource-type VPC \
  --resource-ids vpc-123 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /vpc/flow-logs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flowlogsRole

# Enable flow logs to S3
aws ec2 create-flow-log \
  --resource-type VPC \
  --resource-ids vpc-123 \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::my-flow-logs-bucket/vpc-logs/

# List flow logs
aws ec2 describe-flow-logs --filter "Name=resource-id,Values=vpc-123"

# Delete flow log
aws ec2 delete-flow-logs --flow-log-ids fl-123
```

## CIDR Planning

| Subnet Type | CIDR | Hosts | Use |
|-------------|------|-------|-----|
| /16 VPC | 10.0.0.0/16 | 65,531 | Entire VPC |
| /24 subnet | 10.0.1.0/24 | 251 | Standard subnet |
| /20 subnet | 10.0.0.0/20 | 4,091 | Large subnet (EKS) |
| /28 subnet | 10.0.1.0/28 | 11 | Minimal (NAT GW, firewall) |

> AWS reserves 5 IPs per subnet (first 4 + last 1).

## Troubleshooting

| Issue | Check |
|-------|-------|
| Can't reach internet from private subnet | Route table has 0.0.0.0/0 → NAT GW; NAT GW is in public subnet with IGW route |
| Can't SSH to instance | Security group allows port 22; instance has public IP or you're using SSM |
| Instances can't talk to each other | Security groups allow traffic between them; same VPC or peering/TGW routes exist |
| VPC endpoint not working | DNS resolution enabled; security group on endpoint allows traffic; route table associated |
| Peering not routing | Routes added in BOTH VPCs; CIDRs don't overlap |

## Best Practices

- Use /16 for VPCs, /24 for most subnets, /20 for EKS node subnets
- Minimum 2 AZs for HA, prefer 3
- Separate public/private/data subnets per AZ
- Use VPC endpoints to avoid NAT costs for AWS service traffic
- Enable VPC Flow Logs for security and troubleshooting
- Use Transit Gateway instead of peering mesh when connecting 3+ VPCs
- Tag everything for cost allocation and automation
