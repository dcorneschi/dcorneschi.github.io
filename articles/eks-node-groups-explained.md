# EKS Node Groups Explained

How EKS node groups work — managed vs self-managed, launch templates, scaling, updates, and when to use each type.

## What Is a Node Group?

A node group is a collection of EC2 instances that serve as worker nodes in an EKS cluster. All nodes in a group share the same configuration: instance type, AMI, IAM role, subnets, and scaling settings.

EKS supports three compute options:

| Type | Managed By | Use Case |
|------|-----------|----------|
| Managed Node Groups | AWS (EKS service) | Standard workloads, simplest to operate |
| Self-Managed Node Groups | You (via ASG + launch template) | Full control, custom AMIs, special requirements |
| Fargate Profiles | AWS (serverless) | Isolated pods, no node management at all |

## Managed Node Groups

AWS handles provisioning, updating, and lifecycle management of the underlying EC2 instances.

### What AWS Manages

- EC2 instance provisioning
- AMI updates and rolling replacements
- Node draining during updates (respects PDBs)
- Auto Scaling Group creation and configuration
- aws-auth ConfigMap entries (automatic)
- Instance health checks and replacement

### What You Configure

- Instance types
- Scaling (min, max, desired)
- Subnets
- IAM role
- Labels and taints
- Launch template (optional overrides)
- Update strategy

### Create a Managed Node Group

```bash
# Using AWS CLI
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name general \
  --node-role-arn arn:aws:iam::123456789012:role/EKSNodeRole \
  --subnets subnet-aaa subnet-bbb subnet-ccc \
  --instance-types t3.large m5.large \
  --scaling-config minSize=2,maxSize=10,desiredSize=3 \
  --capacity-type ON_DEMAND \
  --labels environment=production,team=platform \
  --tags Environment=production
```

```bash
# Using eksctl
eksctl create nodegroup \
  --cluster my-cluster \
  --name general \
  --node-type t3.large \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 10 \
  --managed
```

### eksctl ClusterConfig

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-east-1

managedNodeGroups:
  - name: general
    instanceType: t3.large
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    volumeSize: 50
    volumeType: gp3
    labels:
      role: general
    tags:
      Environment: production
    iam:
      withAddonPolicies:
        autoScaler: true
        ebs: true

  - name: compute
    instanceType: c5.xlarge
    minSize: 0
    maxSize: 5
    desiredCapacity: 0
    labels:
      role: compute
    taints:
      - key: workload
        value: compute
        effect: NoSchedule
```

### Multi-Instance Type Support

Managed node groups support multiple instance types for better availability:

```bash
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name mixed \
  --instance-types t3.large t3.xlarge m5.large m5.xlarge \
  --capacity-type SPOT \
  --scaling-config minSize=2,maxSize=20,desiredSize=5
```

When using multiple instance types, EKS creates a mixed instances ASG with:
- Capacity-optimized allocation strategy for Spot
- Prioritized allocation for On-Demand

## Self-Managed Node Groups

You create and manage the ASG, launch template, and node lifecycle yourself.

### When to Use Self-Managed

- Custom AMIs with pre-baked software
- GPU instances with custom drivers
- Windows nodes (historically required, now supported in managed)
- BottleRocket or custom OS
- Specific ASG features (warm pools, instance refresh)
- Mixed instance policies with fine-grained control

### Components

```
┌─────────────────────────────────────────┐
│ You Manage:                             │
│                                         │
│  Launch Template                        │
│    ├─ AMI ID                            │
│    ├─ Instance type                     │
│    ├─ Security groups                   │
│    ├─ IAM instance profile              │
│    ├─ User data (bootstrap.sh)          │
│    └─ Block device mappings             │ 
│                                         │
│  Auto Scaling Group                     │
│    ├─ Min / Max / Desired               │
│    ├─ Subnets                           │
│    ├─ Health checks                     │
│    ├─ Scaling policies                  │
│    └─ Instance refresh                  │
│                                         │
│  aws-auth ConfigMap                     │
│    └─ IAM role → K8s group mapping      │
└─────────────────────────────────────────┘
```

### Launch Template

```bash
aws ec2 create-launch-template \
  --launch-template-name eks-nodes-lt \
  --launch-template-data '{
    "ImageId": "ami-0abcdef1234567890",
    "InstanceType": "m5.large",
    "SecurityGroupIds": ["sg-xxx"],
    "IamInstanceProfile": {"Arn": "arn:aws:iam::123456789012:instance-profile/EKSNodeRole"},
    "BlockDeviceMappings": [{
      "DeviceName": "/dev/xvda",
      "Ebs": {"VolumeSize": 50, "VolumeType": "gp3", "Encrypted": true}
    }],
    "UserData": "'"$(base64 -w0 <<'EOF'
#!/bin/bash
/etc/eks/bootstrap.sh my-cluster \
  --kubelet-extra-args '--node-labels=role=general --max-pods=110'
EOF
)"'"
  }'
```

### Auto Scaling Group

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name eks-general-asg \
  --launch-template LaunchTemplateName=eks-nodes-lt,Version='$Latest' \
  --min-size 2 --max-size 10 --desired-capacity 3 \
  --vpc-zone-identifier "subnet-aaa,subnet-bbb,subnet-ccc" \
  --tags "Key=kubernetes.io/cluster/my-cluster,Value=owned,PropagateAtLaunch=true" \
         "Key=k8s.io/cluster-autoscaler/enabled,Value=true,PropagateAtLaunch=true"
```

### aws-auth ConfigMap Entry

Self-managed nodes require manual mapping:

```bash
kubectl edit configmap aws-auth -n kube-system
```

```yaml
mapRoles:
  - rolearn: arn:aws:iam::123456789012:role/EKSNodeRole
    username: system:node:{{EC2PrivateDNSName}}
    groups:
      - system:bootstrappers
      - system:nodes
```

## Launch Templates with Managed Node Groups

Managed node groups can use custom launch templates for additional control while keeping AWS-managed lifecycle:

```yaml
# eksctl with launch template overrides
managedNodeGroups:
  - name: custom
    launchTemplate:
      id: lt-0123456789abcdef0
      version: "1"
    instanceTypes:
      - m5.large
      - m5.xlarge
    minSize: 2
    maxSize: 10
```

### What You Can Override

| Setting | Default (No LT) | With Launch Template |
|---------|-----------------|---------------------|
| AMI | EKS-optimized | Custom AMI |
| Root volume | 20 GiB gp3 | Custom size/type/encryption |
| Security groups | Cluster + node SG | Custom SGs |
| User data | EKS bootstrap | Custom bootstrap script |
| Instance metadata | IMDSv2 optional | Enforce IMDSv2 |
| Network interfaces | Default | Custom ENI config |

### Custom AMI with Managed Node Group

```bash
# Get latest EKS-optimized AMI
AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.30/amazon-linux-2023/x86_64/standard/recommended/image_id \
  --query 'Parameter.Value' --output text)

# Create launch template with custom AMI
aws ec2 create-launch-template \
  --launch-template-name eks-custom-lt \
  --launch-template-data "{
    \"ImageId\": \"$AMI_ID\",
    \"BlockDeviceMappings\": [{
      \"DeviceName\": \"/dev/xvda\",
      \"Ebs\": {\"VolumeSize\": 100, \"VolumeType\": \"gp3\", \"Encrypted\": true}
    }],
    \"MetadataOptions\": {
      \"HttpTokens\": \"required\",
      \"HttpPutResponseHopLimit\": 2
    }
  }"
```

## Node Group Scaling

### Cluster Autoscaler Integration

Tag your ASG for discovery:

```bash
# Required tags for Cluster Autoscaler
Key=k8s.io/cluster-autoscaler/enabled,Value=true
Key=k8s.io/cluster-autoscaler/my-cluster,Value=owned
```

### Manual Scaling

```bash
# Managed node group
aws eks update-nodegroup-config \
  --cluster-name my-cluster \
  --nodegroup-name general \
  --scaling-config minSize=3,maxSize=15,desiredSize=5

# Self-managed ASG
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name eks-general-asg \
  --min-size 3 --max-size 15 --desired-capacity 5
```

### Scale to Zero

Managed node groups support scaling to zero:

```bash
aws eks update-nodegroup-config \
  --cluster-name my-cluster \
  --nodegroup-name batch \
  --scaling-config minSize=0,maxSize=10,desiredSize=0
```

Useful for:
- GPU node groups used only during training
- Spot pools as overflow capacity
- Dev/test environments off-hours

## Node Group Updates

### Managed Node Group Update Strategies

```bash
# Update AMI (rolling update)
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name general

# With custom launch template version
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name general \
  --launch-template name=eks-custom-lt,version=3
```

### Update Configuration

| Setting | Can Update? |
|---------|------------|
| Scaling (min/max/desired) | Yes |
| Labels | Yes |
| Taints | Yes |
| Launch template version | Yes (triggers rolling update) |
| Instance types | Yes (for new nodes) |
| Subnets | No (recreate node group) |
| IAM role | No (recreate node group) |
| Capacity type (Spot/OD) | No (recreate node group) |

### Rolling Update Process

When you update a managed node group, EKS:

1. Launches new nodes with updated configuration
2. Waits for new nodes to be Ready
3. Cordons old nodes (prevents new pod scheduling)
4. Drains old nodes (evicts pods, respects PDBs)
5. Waits for pods to reschedule
6. Terminates old nodes

```bash
# Check update status
aws eks describe-update \
  --cluster-name my-cluster \
  --nodegroup-name general \
  --update-id <update-id>

# List recent updates
aws eks list-updates \
  --cluster-name my-cluster \
  --nodegroup-name general
```

### Update Configuration Options

```bash
# Force update (don't respect PDBs after timeout)
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name general \
  --force

# Update config with max unavailable
aws eks update-nodegroup-config \
  --cluster-name my-cluster \
  --nodegroup-name general \
  --update-config maxUnavailable=1
# Or percentage:
# --update-config maxUnavailablePercentage=25
```

### Self-Managed Node Group Updates

Use ASG instance refresh:

```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name eks-general-asg \
  --preferences '{
    "MinHealthyPercentage": 90,
    "InstanceWarmup": 300
  }'
```

Or manual rolling update:

```bash
# For each old node:
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
aws ec2 terminate-instances --instance-ids <instance-id>
# ASG launches replacement with new launch template
```

## Spot Instance Node Groups

### Managed Spot Node Group

```bash
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name spot-workers \
  --capacity-type SPOT \
  --instance-types m5.large m5.xlarge m5a.large c5.large c5.xlarge r5.large \
  --scaling-config minSize=0,maxSize=20,desiredSize=5 \
  --labels lifecycle=spot \
  --taints "key=spot,value=true,effect=PreferNoSchedule"
```

### Spot Best Practices

- Use many instance types (6+ recommended) for better availability
- Use `capacity-optimized` allocation strategy (default for managed)
- Don't rely on Spot for critical single-replica workloads
- Add tolerations to deployments that can run on Spot

```yaml
spec:
  template:
    spec:
      tolerations:
        - key: spot
          operator: Equal
          value: "true"
          effect: PreferNoSchedule
      nodeSelector:
        lifecycle: spot
```

## Taints and Labels

### Apply During Node Group Creation

```bash
# Managed node group with taints and labels
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name gpu \
  --instance-types p3.2xlarge \
  --scaling-config minSize=0,maxSize=4,desiredSize=0 \
  --labels "accelerator=nvidia,workload=ml" \
  --taints "key=nvidia.com/gpu,value=true,effect=NoSchedule"
```

### Update Labels and Taints

```bash
# Add/update labels
aws eks update-nodegroup-config \
  --cluster-name my-cluster \
  --nodegroup-name general \
  --labels addOrUpdateLabels={team=platform,env=prod},removeLabels=[old-label]

# Add/update taints
aws eks update-nodegroup-config \
  --cluster-name my-cluster \
  --nodegroup-name general \
  --taints addOrUpdateTaints=[{key=dedicated,value=special,effect=NoSchedule}]
```

## IAM Role for Node Groups

### Required Policies

```bash
# Create the node role
aws iam create-role --role-name EKSNodeRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach required policies
aws iam attach-role-policy --role-name EKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy --role-name EKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

aws iam attach-role-policy --role-name EKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

# Optional: SSM access
aws iam attach-role-policy --role-name EKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

| Policy | Purpose |
|--------|---------|
| AmazonEKSWorkerNodePolicy | kubelet to call EKS APIs |
| AmazonEKS_CNI_Policy | VPC CNI to manage ENIs and IPs |
| AmazonEC2ContainerRegistryReadOnly | Pull images from ECR |
| AmazonSSMManagedInstanceCore | Optional: SSM access without SSH |

## Common Patterns

### System Node Group + Application Node Groups

```yaml
managedNodeGroups:
  # System: runs Karpenter, CoreDNS, kube-proxy, monitoring
  - name: system
    instanceType: t3.medium
    minSize: 2
    maxSize: 3
    desiredCapacity: 2
    labels:
      role: system
    taints:
      - key: CriticalAddonsOnly
        value: "true"
        effect: NoSchedule

  # General: standard workloads
  - name: general
    instanceTypes: [m5.large, m5.xlarge]
    minSize: 2
    maxSize: 20
    desiredCapacity: 3
    labels:
      role: general

  # Spot: cost-tolerant workloads
  - name: spot
    instanceTypes: [m5.large, m5.xlarge, m5a.large, c5.large]
    minSize: 0
    maxSize: 30
    desiredCapacity: 5
    capacityType: SPOT
    labels:
      role: spot
      lifecycle: spot
    taints:
      - key: spot
        value: "true"
        effect: PreferNoSchedule
```

### Multi-AZ Distribution

Managed node groups automatically distribute nodes across specified subnets (AZs). Ensure subnets span multiple AZs:

```bash
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name multi-az \
  --subnets subnet-az-a subnet-az-b subnet-az-c \
  --instance-types m5.large \
  --scaling-config minSize=3,maxSize=12,desiredSize=6
```

## Troubleshooting

### Node Group Creation Failures

```bash
# Check node group status and health issues
aws eks describe-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name general \
  --query 'nodegroup.[status, health.issues]'
```

Common failures:

| Issue | Cause | Fix |
|-------|-------|-----|
| `Ec2SubnetNotFound` | Subnet ID doesn't exist | Verify subnet IDs |
| `NodeCreationFailure` | IAM or SG issues | Check role policies and SG rules |
| `InsufficientFreeAddresses` | Subnet IP exhaustion | Use larger subnets or add secondary CIDR |
| `Ec2SecurityGroupNotFound` | SG deleted | Recreate or update cluster SG config |
| `AsgInstanceLaunchFailures` | No capacity for instance type | Add more instance types or try different AZ |

### Nodes Not Joining

```bash
# Check if instances are running
aws ec2 describe-instances --filters \
  "Name=tag:eks:nodegroup-name,Values=general" \
  "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PrivateIpAddress]' --output table

# Check aws-auth (for self-managed)
kubectl get configmap aws-auth -n kube-system -o yaml

# Check node group status
aws eks describe-nodegroup --cluster-name my-cluster --nodegroup-name general \
  --query 'nodegroup.health'
```

### Update Stuck

```bash
# Check update status
aws eks list-updates --cluster-name my-cluster --nodegroup-name general
aws eks describe-update --cluster-name my-cluster --nodegroup-name general --update-id <id>

# Common blockers:
# - PDBs preventing drain
kubectl get pdb -A

# - Pods with no owner (standalone pods)
kubectl get pods -A --field-selector spec.nodeName=<old-node> -o json | \
  jq '.items[] | select(.metadata.ownerReferences == null) | .metadata.name'

# - Pods stuck terminating
kubectl get pods -A | grep Terminating
```

## Node Groups vs Node Selectors

Node groups create nodes → Nodes get labels → Node selectors target those labels.

### How It Works

**1. Node Group Creates Nodes**

When you create a node group, it provisions EC2 instances that register as Kubernetes nodes.

**2. Labels Are Applied**

Node groups automatically add labels:

```yaml
eks.amazonaws.com/nodegroup: my-node-group-name
node.kubernetes.io/instance-type: t3.medium
topology.kubernetes.io/zone: us-east-1a
```

You also add custom labels when creating the node group:

```yaml
labels:
  workload-type: compute-intensive
  environment: production
```

**3. Node Selectors Use Those Labels**

In pod specs, use node selectors to target specific nodes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  nodeSelector:
    workload-type: compute-intensive
    eks.amazonaws.com/nodegroup: my-node-group-name
  containers:
  - name: app
    image: my-app:latest
```

### Why Have Multiple Node Groups?

Different node groups serve different purposes:

| Node Group | Labels | Workloads |
|-----------|--------|-----------|
| GPU | `accelerator: gpu` | ML training, inference |
| Spot | `capacity-type: spot` | Batch jobs, fault-tolerant tasks |
| High-memory | `workload: memory-intensive` | Databases, caches |
| General | `workload: general` | Standard applications |

Pods use node selectors (or node affinity) to land on the right node group based on their requirements.

## Quick Reference

```bash
# List node groups
aws eks list-nodegroups --cluster-name my-cluster

# Describe node group
aws eks describe-nodegroup --cluster-name my-cluster --nodegroup-name general

# Scale
aws eks update-nodegroup-config --cluster-name my-cluster --nodegroup-name general \
  --scaling-config minSize=2,maxSize=10,desiredSize=5

# Update AMI (rolling)
aws eks update-nodegroup-version --cluster-name my-cluster --nodegroup-name general

# Update labels
aws eks update-nodegroup-config --cluster-name my-cluster --nodegroup-name general \
  --labels addOrUpdateLabels={key=value}

# Delete node group
aws eks delete-nodegroup --cluster-name my-cluster --nodegroup-name general

# Get nodes in a node group
kubectl get nodes -l eks.amazonaws.com/nodegroup=general

# Check EKS-optimized AMI
aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.30/amazon-linux-2023/x86_64/standard/recommended/image_id \
  --query 'Parameter.Value' --output text
```
