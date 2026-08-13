# EKS Managed Node Groups: With and Without Launch Templates

## Overview

Amazon EKS managed node groups automate the provisioning and lifecycle management of worker nodes (EC2 instances) for your Kubernetes cluster. You don't need to separately provision or register EC2 instances — EKS handles creation, updates, and termination with a single operation.

A common question is whether you need a launch template for your node groups. The short answer: **no, a launch template is not required**. EKS creates one automatically behind the scenes. But providing your own gives you more control.

---

## How It Works Under the Hood

Managed node groups are **always** deployed with a launch template for the underlying Auto Scaling group. The difference is who creates it:

- **Without a custom launch template** — the EKS API creates one automatically with default values in your account
- **With a custom launch template** — you provide your own, giving you control over the configuration

The auto-generated launch template is fully managed by EKS. You should **never modify it manually** or errors will occur.

---

## Without a Launch Template (EKS Defaults)

When you create a node group without specifying a launch template, EKS handles everything:

```bash
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name my-nodes \
  --node-role arn:aws:iam::123456789012:role/eks-node-role \
  --subnets "subnet-abc123" "subnet-def456" \
  --instance-types "t3.medium" \
  --scaling-config minSize=2,maxSize=5,desiredSize=3
```

### What EKS manages automatically

- **AMI** — uses the latest EKS optimized AMI for your cluster's Kubernetes version
- **User data** — injects the bootstrap script to join nodes to the cluster
- **Root volume** — default 20 GiB EBS volume
- **Security groups** — attaches the cluster security group for node-to-control-plane communication
- **Instance metadata** — configured with IMDSv2 recommended settings
- **Kubernetes labels** — adds `eks.amazonaws.com` prefixed labels
- **Cluster Autoscaler tags** — tags ASG for auto-discovery

### What you configure through the node group API

- Instance types (defaults to `t3.medium` if not specified)
- Scaling configuration (min, max, desired)
- Subnets
- Node IAM role
- AMI type (AL2, AL2023, Bottlerocket, Windows)
- Disk size
- SSH key pair and source security groups for remote access
- Kubernetes labels and taints
- Capacity type (On-Demand or Spot)

### Why a launch template isn't required

For most standard workloads, the EKS defaults are perfectly fine:

1. **EKS handles bootstrapping** — the user data to join nodes to the cluster is generated automatically. You don't need to worry about the `bootstrap.sh` script, API server endpoint, or certificate authority.

2. **AMI updates are automatic** — EKS uses the latest optimized AMI for your cluster version. When you update the node group, it picks up the newest AMI.

3. **Security groups are managed** — EKS attaches the cluster security group, ensuring nodes can communicate with the control plane.

4. **Less to maintain** — no launch template versions to track, no drift between template and node group config.

---

## With a Launch Template

You need a custom launch template when the EKS defaults aren't enough:

```bash
aws ec2 create-launch-template \
  --launch-template-name eks-custom-template \
  --launch-template-data '{
    "BlockDeviceMappings": [{
      "DeviceName": "/dev/xvda",
      "Ebs": {
        "VolumeSize": 100,
        "VolumeType": "gp3",
        "Iops": 3000,
        "Throughput": 125,
        "Encrypted": true
      }
    }],
    "TagSpecifications": [{
      "ResourceType": "instance",
      "Tags": [{"Key": "Environment", "Value": "production"}]
    }]
  }'
```

Then reference it in the node group:

```bash
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name my-custom-nodes \
  --node-role arn:aws:iam::123456789012:role/eks-node-role \
  --subnets "subnet-abc123" "subnet-def456" \
  --launch-template name=eks-custom-template,version=1 \
  --scaling-config minSize=2,maxSize=10,desiredSize=3
```

### When you need a launch template

- **Custom AMI** — using a hardened or pre-baked image instead of the EKS optimized AMI
- **Custom user data** — passing extra kubelet arguments, installing packages, or configuring proxies
- **Larger or encrypted root volumes** — need more than 20 GiB or require EBS encryption
- **Custom security groups** — attaching additional security groups beyond the cluster SG
- **Instance tags** — tagging EC2 instances with custom metadata
- **Custom CNI** — deploying an alternative CNI plugin
- **Pod IP from different CIDR** — using custom networking for pod IPs
- **Private clusters** — bootstrapping nodes without internet access
- **Prefix delegation** — enabling more IPs per node with VPC CNI prefix assignment
- **Metadata options** — enforcing IMDSv2 or setting hop limit to 2 for containers using IMDS

### What you CAN'T set in the launch template

These must be set in the node group configuration, not the template:

- **Subnets** — use the node group's network configuration
- **IAM instance profile** — use the node group's IAM role setting
- **Shutdown/Hibernate behavior** — EKS must control instance lifecycle

### What you CAN'T set in the node group configuration (if using a template)

These move to the launch template and are hidden from the node group API:

- **AMI type** — if you specify a custom AMI in the template, the node group shows "Specified in launch template"
- **Disk size** — must be set in the template's block device mapping
- **SSH key pair** — must be set in the template
- **Security groups** — must be set in the template

---

## Key Differences

| Feature | Without Launch Template | With Launch Template |
|---|---|---|
| AMI | EKS optimized (auto-selected) | Custom or EKS optimized |
| Root volume | 20 GiB (configurable via API) | Full EBS customization |
| User data | Fully managed by EKS | Custom (merged with EKS bootstrap) |
| Security groups | Cluster SG only | Custom SGs (you manage connectivity) |
| Instance tags | Limited (via node group tags) | Full tag specifications |
| Iterative updates | Must create new node group | Update launch template version |
| EBS encryption | Not enabled by default | Configurable |
| IMDSv2 enforcement | Default settings | Configurable |
| Kubelet arguments | Limited (labels/taints via API) | Full control via user data |

---

## Important: Immutable Settings and the Recreation Problem

Several node group settings are **immutable** after creation — changing them requires destroying the node group and creating a new one:

- **Instance types** (without a launch template)
- **AMI type**
- **Disk size**
- **Subnets**
- **Node IAM role**
- **Capacity type** (On-Demand vs Spot)
- **Adding a launch template** to a node group created without one

This is a critical reason to **start with a launch template**. When you use a launch template, changing instance types is simply a launch template version update — EKS performs a rolling update (new nodes up, old nodes drained). Without a launch template, changing instance types forces full node group recreation: destroy the old group, create a new one, and migrate workloads.

### With launch template (rolling update)

```bash
# Update the launch template with new instance type
aws ec2 create-launch-template-version \
  --launch-template-id lt-0123456789abcdef \
  --source-version 1 \
  --launch-template-data '{"InstanceType": "m5.2xlarge"}'

# Update node group to use new version — triggers rolling update
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name my-nodes \
  --launch-template id=lt-0123456789abcdef,version=2
```

### Without launch template (forced recreation)

```bash
# No API to change instance types in-place
# You must: create new node group → drain old → delete old
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name my-nodes-v2 \
  --node-role arn:aws:iam::123456789012:role/eks-node-role \
  --subnets "subnet-abc123" "subnet-def456" \
  --instance-types "m5.2xlarge" \
  --scaling-config minSize=2,maxSize=5,desiredSize=3

# Wait for new nodes to be ready, then delete old group
aws eks delete-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name my-nodes
```

This is the strongest argument for always using a launch template, even if you don't need customization today.

---

## Custom AMI Considerations

When you specify a custom AMI in your launch template:

- **You own patching** — EKS is responsible for building patched AMIs only for EKS optimized AMIs. With custom AMIs, you build and deploy patches yourself.
- **You handle user data** — EKS does not merge its bootstrap user data with yours. You must provide the full bootstrap configuration.
- **AL2023 requires additional metadata** — unlike AL2 which called `DescribeCluster`, AL2023 requires you to explicitly provide `apiServerEndpoint`, `certificateAuthority`, and service `cidr` in user data.

### AL2 bootstrap user data (custom AMI)

```bash
#!/bin/bash
/etc/eks/bootstrap.sh my-cluster \
  --kubelet-extra-args '--node-labels=env=production'
```

### AL2023 bootstrap user data (custom AMI)

```yaml
---
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    name: my-cluster
    apiServerEndpoint: https://XXXXXXXX.gr7.us-east-1.eks.amazonaws.com
    certificateAuthority: BASE64_ENCODED_CA
    cidr: 10.100.0.0/16
```

### Bottlerocket user data (custom settings)

```toml
[settings.kubernetes.system-reserved]
cpu = "10m"
memory = "100Mi"

[settings.kubernetes.node-labels]
"env" = "production"
```

---

## Custom Security Groups

When you provide custom security groups in a launch template, EKS **does not** automatically add the cluster security group. You must ensure your security groups allow:

- **Inbound** from control plane to nodes (port 443, 10250, and ephemeral ports)
- **Outbound** from nodes to control plane (port 443)
- **Node-to-node** communication (all ports)

If your security group rules are incorrect, nodes will not join the cluster.

---

## Terraform Examples

### Without launch template

```hcl
resource "aws_eks_node_group" "default" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "default-nodes"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.private_subnet_ids

  instance_types = ["t3.large"]
  capacity_type  = "ON_DEMAND"

  scaling_config {
    desired_size = 3
    max_size     = 5
    min_size     = 2
  }
}
```

### With launch template

```hcl
resource "aws_launch_template" "eks_nodes" {
  name_prefix   = "eks-nodes-"
  instance_type = "m5.xlarge"

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size = 100
      volume_type = "gp3"
      encrypted   = true
    }
  }

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 2
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Environment = "production"
    }
  }
}

resource "aws_eks_node_group" "custom" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "custom-nodes"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.private_subnet_ids

  capacity_type = "ON_DEMAND"

  launch_template {
    id      = aws_launch_template.eks_nodes.id
    version = aws_launch_template.eks_nodes.latest_version
  }

  scaling_config {
    desired_size = 3
    max_size     = 10
    min_size     = 2
  }
}
```

> **Note:** When using a launch template, define instance type inside the launch template (not on the node group). This way, changing the instance type creates a new launch template version and triggers a rolling update — instead of forcing node group recreation.

---

## When to Use What

**Use node groups WITHOUT a launch template when:**
- You're running a quick proof of concept or dev cluster
- You're certain instance types, volumes, and AMI won't change
- You want the absolute simplest setup with no maintenance

**Use node groups WITH a launch template when (recommended for production):**
- You need a custom or hardened AMI
- You need larger or encrypted root volumes
- You need additional security groups attached to nodes
- You need custom instance tags
- You need custom kubelet arguments or user data
- You want to change instance types without recreating the node group
- You want flexibility for future changes (this alone is reason enough)
- You're running in a private cluster without internet access
- You need to enforce IMDSv2 with a specific hop limit

---

## Summary

A launch template is not required for EKS managed node groups because EKS creates one automatically with sensible defaults. For quick proofs of concept, this is fine.

However, for production workloads, **always use a launch template** — even a minimal one. The key reason: without a launch template, changing instance types (or disk size, or many other settings) forces full node group recreation. With a launch template, these changes become a simple version update with a rolling replacement. The small upfront effort of creating a launch template saves significant operational pain later.
