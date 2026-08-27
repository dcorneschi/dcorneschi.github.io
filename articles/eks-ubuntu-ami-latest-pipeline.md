# Pipeline to Get the Latest Ubuntu EKS AMI

How to automatically discover the latest Ubuntu EKS-optimized AMI for your current cluster version — SSM parameters, AWS CLI queries, Terraform data sources, and CI/CD integration patterns.

## SSM Parameter Store — The Official Source

Canonical publishes EKS-optimized Ubuntu AMIs and registers them in AWS SSM Parameter Store:

```bash
# Pattern:
/aws/service/canonical/ubuntu/eks/20.04/<k8s-version>/stable/current/<arch>/hvm/ebs-gp2/ami-id
/aws/service/canonical/ubuntu/eks-pro/22.04/<k8s-version>/stable/current/<arch>/hvm/ebs-gp2/ami-id

# Ubuntu 22.04 (EKS):
aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/eks/22.04/1.30/stable/current/amd64/hvm/ebs-gp2/ami-id \
  --query "Parameter.Value" --output text

# Ubuntu 22.04 Pro (EKS):
aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/eks-pro/22.04/1.30/stable/current/amd64/hvm/ebs-gp2/ami-id \
  --query "Parameter.Value" --output text

# ARM64:
aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/eks/22.04/1.30/stable/current/arm64/hvm/ebs-gp2/ami-id \
  --query "Parameter.Value" --output text
```

### SSM Parameter Path Structure

```
/aws/service/canonical/ubuntu/eks/<ubuntu-version>/<k8s-version>/stable/current/<arch>/hvm/ebs-gp2/ami-id
                                  │                 │                            │
                                  │                 │                            └── amd64 or arm64
                                  │                 └── 1.28, 1.29, 1.30, 1.31
                                  └── 20.04, 22.04
```

## Get AMI for Current Cluster Version (Dynamic)

```bash
#!/bin/bash
# Automatically resolve AMI for the cluster's current K8s version

CLUSTER_NAME="my-cluster"
REGION="us-east-1"
ARCH="amd64"
UBUNTU_VERSION="22.04"

# Get current cluster K8s version:
K8S_VERSION=$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$REGION" \
  --query "cluster.version" --output text)

echo "Cluster $CLUSTER_NAME is on K8s $K8S_VERSION"

# Get the latest Ubuntu EKS AMI for that version:
AMI_ID=$(aws ssm get-parameter \
  --name "/aws/service/canonical/ubuntu/eks/${UBUNTU_VERSION}/${K8S_VERSION}/stable/current/${ARCH}/hvm/ebs-gp2/ami-id" \
  --region "$REGION" \
  --query "Parameter.Value" --output text)

echo "Latest Ubuntu $UBUNTU_VERSION EKS AMI for K8s $K8S_VERSION: $AMI_ID"
```

## Alternative: aws ec2 describe-images (Filter by Name)

If SSM parameters aren't available or you want more metadata:

```bash
# Find latest Ubuntu EKS AMI by name pattern:
aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu-eks/k8s_1.30/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
    "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].{ID:ImageId,Name:Name,Created:CreationDate}" \
  --output table
```

Owner `099720109477` is Canonical's AWS account ID.

```bash
# For Ubuntu Pro EKS:
aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu-eks-pro/k8s_1.30/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
    "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text
```

## Terraform Data Source

### Using SSM Parameter

```hcl
data "aws_eks_cluster" "current" {
  name = var.cluster_name
}

data "aws_ssm_parameter" "ubuntu_eks_ami" {
  name = "/aws/service/canonical/ubuntu/eks/22.04/${data.aws_eks_cluster.current.version}/stable/current/amd64/hvm/ebs-gp2/ami-id"
}

resource "aws_launch_template" "workers" {
  name_prefix = "eks-ubuntu-"
  image_id    = data.aws_ssm_parameter.ubuntu_eks_ami.value

  # ...
}

output "ubuntu_ami_id" {
  value = data.aws_ssm_parameter.ubuntu_eks_ami.value
}
```

### Using aws_ami Data Source

```hcl
data "aws_ami" "ubuntu_eks" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu-eks/k8s_${data.aws_eks_cluster.current.version}/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "state"
    values = ["available"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
}

output "ami_id" {
  value = data.aws_ami.ubuntu_eks.id
}

output "ami_name" {
  value = data.aws_ami.ubuntu_eks.name
}
```

## CI/CD Pipeline — Auto-Detect and Update

### GitHub Actions: Check for New AMI and Update Launch Template

```yaml
name: Update EKS Node AMI

on:
  schedule:
    - cron: '0 8 * * 1'  # Every Monday at 8am
  workflow_dispatch:       # Manual trigger

env:
  CLUSTER_NAME: production
  AWS_REGION: us-east-1
  UBUNTU_VERSION: "22.04"
  ARCH: amd64

jobs:
  check-and-update:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: write
      pull-requests: write

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/GitHubActionsAMICheck
        aws-region: ${{ env.AWS_REGION }}

    - name: Get cluster version
      id: cluster
      run: |
        VERSION=$(aws eks describe-cluster --name ${{ env.CLUSTER_NAME }} \
          --query "cluster.version" --output text)
        echo "k8s_version=$VERSION" >> $GITHUB_OUTPUT

    - name: Get latest Ubuntu EKS AMI
      id: ami
      run: |
        AMI_ID=$(aws ssm get-parameter \
          --name "/aws/service/canonical/ubuntu/eks/${{ env.UBUNTU_VERSION }}/${{ steps.cluster.outputs.k8s_version }}/stable/current/${{ env.ARCH }}/hvm/ebs-gp2/ami-id" \
          --query "Parameter.Value" --output text)
        echo "ami_id=$AMI_ID" >> $GITHUB_OUTPUT
        echo "Latest AMI: $AMI_ID"

    - name: Get current AMI from Terraform
      id: current
      run: |
        CURRENT=$(grep -oP 'ami-[a-z0-9]+' terraform/environments/production/terraform.tfvars | head -1)
        echo "current_ami=$CURRENT" >> $GITHUB_OUTPUT
        echo "Current AMI: $CURRENT"

    - name: Compare and update
      if: steps.ami.outputs.ami_id != steps.current.outputs.current_ami
      run: |
        echo "New AMI available: ${{ steps.ami.outputs.ami_id }}"
        echo "Current AMI: ${{ steps.current.outputs.current_ami }}"

        # Update Terraform tfvars:
        sed -i "s/${{ steps.current.outputs.current_ami }}/${{ steps.ami.outputs.ami_id }}/g" \
          terraform/environments/production/terraform.tfvars

    - name: Create Pull Request
      if: steps.ami.outputs.ami_id != steps.current.outputs.current_ami
      uses: peter-evans/create-pull-request@v6
      with:
        title: "Update EKS Ubuntu AMI to ${{ steps.ami.outputs.ami_id }}"
        body: |
          New Ubuntu EKS AMI available for K8s ${{ steps.cluster.outputs.k8s_version }}:
          - Old: `${{ steps.current.outputs.current_ami }}`
          - New: `${{ steps.ami.outputs.ami_id }}`

          This was detected by the weekly AMI check pipeline.
        branch: update-eks-ami-${{ steps.ami.outputs.ami_id }}
        commit-message: "Update EKS Ubuntu AMI to ${{ steps.ami.outputs.ami_id }}"
```

### Shell Script (Cron / Jenkins)

```bash
#!/bin/bash
# check-ami-update.sh — Run weekly via cron or Jenkins
set -e

CLUSTER_NAME="${1:-production}"
REGION="${2:-us-east-1}"
UBUNTU_VERSION="22.04"
ARCH="amd64"
NODEGROUP_NAME="workers"

# Get cluster K8s version
K8S_VERSION=$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$REGION" \
  --query "cluster.version" --output text)

# Get latest AMI
LATEST_AMI=$(aws ssm get-parameter \
  --name "/aws/service/canonical/ubuntu/eks/${UBUNTU_VERSION}/${K8S_VERSION}/stable/current/${ARCH}/hvm/ebs-gp2/ami-id" \
  --region "$REGION" --query "Parameter.Value" --output text)

# Get current AMI on the node group
CURRENT_AMI=$(aws eks describe-nodegroup --cluster-name "$CLUSTER_NAME" \
  --nodegroup-name "$NODEGROUP_NAME" --region "$REGION" \
  --query "nodegroup.releaseVersion" --output text 2>/dev/null || echo "unknown")

# Or get from launch template:
LT_ID=$(aws eks describe-nodegroup --cluster-name "$CLUSTER_NAME" \
  --nodegroup-name "$NODEGROUP_NAME" --region "$REGION" \
  --query "nodegroup.launchTemplate.id" --output text)

if [ "$LT_ID" != "None" ] && [ -n "$LT_ID" ]; then
  CURRENT_AMI=$(aws ec2 describe-launch-template-versions --launch-template-id "$LT_ID" \
    --versions '$Latest' --region "$REGION" \
    --query "LaunchTemplateVersions[0].LaunchTemplateData.ImageId" --output text)
fi

echo "Cluster: $CLUSTER_NAME (K8s $K8S_VERSION)"
echo "Current AMI: $CURRENT_AMI"
echo "Latest AMI:  $LATEST_AMI"

if [ "$CURRENT_AMI" == "$LATEST_AMI" ]; then
  echo "✓ Already on latest AMI"
  exit 0
fi

echo "⚠ New AMI available!"
echo ""
echo "To update (managed node group):"
echo "  aws eks update-nodegroup-version --cluster-name $CLUSTER_NAME --nodegroup-name $NODEGROUP_NAME --region $REGION"
echo ""
echo "To update (launch template):"
echo "  aws ec2 create-launch-template-version --launch-template-id $LT_ID --source-version '\$Latest' --launch-template-data '{\"ImageId\":\"$LATEST_AMI\"}'"
```

## Comparing AMI Versions

```bash
# List all available Ubuntu EKS AMIs for a version (sorted by date):
aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu-eks/k8s_1.30/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
  --query "sort_by(Images, &CreationDate)[].{Created:CreationDate,ID:ImageId,Name:Name}" \
  --output table

# Compare what's running on your nodes vs latest available:
echo "=== Nodes ==="
kubectl get nodes -o custom-columns=NAME:.metadata.name,AMI:.status.nodeInfo.osImage,KUBELET:.status.nodeInfo.kubeletVersion

echo "=== Latest Available ==="
aws ssm get-parameter \
  --name "/aws/service/canonical/ubuntu/eks/22.04/1.30/stable/current/amd64/hvm/ebs-gp2/ami-id" \
  --query "Parameter.Value" --output text
```

## AMI SSM Parameters — All Variants

```bash
# List all available Ubuntu EKS SSM parameters:
aws ssm get-parameters-by-path \
  --path /aws/service/canonical/ubuntu/eks/ \
  --query "Parameters[].Name" --output table

# Common paths:
# Ubuntu 20.04 EKS:
/aws/service/canonical/ubuntu/eks/20.04/<k8s-ver>/stable/current/amd64/hvm/ebs-gp2/ami-id

# Ubuntu 22.04 EKS:
/aws/service/canonical/ubuntu/eks/22.04/<k8s-ver>/stable/current/amd64/hvm/ebs-gp2/ami-id

# Ubuntu 22.04 Pro EKS:
/aws/service/canonical/ubuntu/eks-pro/22.04/<k8s-ver>/stable/current/amd64/hvm/ebs-gp2/ami-id

# ARM64 variants (same path, swap arch):
/aws/service/canonical/ubuntu/eks/22.04/<k8s-ver>/stable/current/arm64/hvm/ebs-gp2/ami-id
```

## Triggering a Node Group Rolling Update

After identifying a new AMI, trigger the node replacement:

### Managed Node Group (EKS API)

```bash
# Updates to the latest AMI release for the current K8s version:
aws eks update-nodegroup-version \
  --cluster-name production \
  --nodegroup-name workers \
  --region us-east-1

# This triggers a rolling update:
# 1. New nodes with latest AMI launched
# 2. Old nodes cordoned + drained
# 3. Old nodes terminated
```

### Custom Launch Template

```bash
# Create new launch template version with updated AMI:
aws ec2 create-launch-template-version \
  --launch-template-id lt-0abc123 \
  --source-version '$Latest' \
  --launch-template-data "{\"ImageId\":\"$LATEST_AMI\"}"

# Then update the node group to use the new version:
aws eks update-nodegroup-version \
  --cluster-name production \
  --nodegroup-name workers \
  --launch-template '{"id":"lt-0abc123","version":"$Latest"}'
```

### Self-Managed (ASG Instance Refresh)

```bash
# Update ASG launch template, then trigger refresh:
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name eks-workers-asg \
  --preferences '{"MinHealthyPercentage":90,"InstanceWarmup":300}'
```

## Quick Reference

```bash
# Get latest Ubuntu EKS AMI for current cluster version:
K8S_VERSION=$(aws eks describe-cluster --name <cluster> --query "cluster.version" --output text)
aws ssm get-parameter \
  --name "/aws/service/canonical/ubuntu/eks/22.04/${K8S_VERSION}/stable/current/amd64/hvm/ebs-gp2/ami-id" \
  --query "Parameter.Value" --output text

# Terraform:
data "aws_ssm_parameter" "ami" {
  name = "/aws/service/canonical/ubuntu/eks/22.04/${data.aws_eks_cluster.current.version}/stable/current/amd64/hvm/ebs-gp2/ami-id"
}

# Filter by name (alternative):
aws ec2 describe-images --owners 099720109477 \
  --filters "Name=name,Values=ubuntu-eks/k8s_${K8S_VERSION}/*amd64*" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" --output text

# Check what nodes are running:
kubectl get nodes -o custom-columns=NAME:.metadata.name,OS:.status.nodeInfo.osImage

# Trigger node group update (after AMI change):
aws eks update-nodegroup-version --cluster-name <cluster> --nodegroup-name <ng>

# Canonical owner ID: 099720109477
# SSM path: /aws/service/canonical/ubuntu/eks/<ubuntu-ver>/<k8s-ver>/stable/current/<arch>/hvm/ebs-gp2/ami-id
```
