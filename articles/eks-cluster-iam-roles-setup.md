# EKS Cluster IAM Roles Setup

IAM roles and policies required to create and operate an EKS cluster — cluster role, worker node role, and the three ways to deploy.

## Cluster Creation Options

There are three popular options to create an EKS cluster:

| Method | Best For |
|--------|----------|
| AWS Console (web interface) | Learning, one-off clusters |
| eksctl CLI | Quick setup, development, managed config |
| Terraform / IaC | Production, repeatable, version-controlled |

## Required IAM Roles and Policies

You need two IAM roles: one for the EKS control plane and one for worker nodes.

| Role | Policy Name | Policy ARN |
|------|-------------|------------|
| EKS Cluster | AmazonEKSClusterPolicy | arn:aws:iam::aws:policy/AmazonEKSClusterPolicy |
| EKS Cluster | AmazonEKSServicePolicy | arn:aws:iam::aws:policy/AmazonEKSServicePolicy |
| Worker Nodes | AmazonEKSWorkerNodePolicy | arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy |
| Worker Nodes | AmazonEC2ContainerRegistryReadOnly | arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly |
| Worker Nodes | AmazonEKS_CNI_Policy | arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy |

## EKS Cluster Policies

### AmazonEKSClusterPolicy

Provides Kubernetes the permissions it requires to manage resources on your behalf. Kubernetes requires `ec2:CreateTags` permissions to place identifying information on EC2 resources including Instances, Security Groups, and Elastic Network Interfaces.

### AmazonEKSServicePolicy

Allows Amazon Elastic Kubernetes Service to create and manage the necessary resources to operate EKS clusters (load balancers, ENIs, security groups).

## Worker Node Policies

### AmazonEKSWorkerNodePolicy

Allows EKS worker nodes to connect to EKS clusters. Includes permissions for the kubelet to register the node and report status.

### AmazonEC2ContainerRegistryReadOnly

Provides read-only access to Amazon ECR repositories. Required for pulling container images from ECR.

### AmazonEKS_CNI_Policy

Provides the Amazon VPC CNI Plugin (amazon-vpc-cni-k8s) the permissions to modify IP address configuration on worker nodes. Enables the CNI to list, describe, and modify Elastic Network Interfaces.

> **Note:** For production, consider moving the CNI policy to a dedicated service account using IRSA instead of attaching it to the node role. This follows least-privilege principles.

## Creating the EKS Cluster Role

```bash
# Create the trust policy
cat > eks-cluster-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name EKSClusterRole \
  --assume-role-policy-document file://eks-cluster-trust-policy.json

# Attach policies
aws iam attach-role-policy \
  --role-name EKSClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

aws iam attach-role-policy \
  --role-name EKSClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSServicePolicy
```

## Creating the Worker Node Role

```bash
# Create the trust policy
cat > node-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name EKSWorkerNodeRole \
  --assume-role-policy-document file://node-trust-policy.json

# Attach policies
aws iam attach-role-policy \
  --role-name EKSWorkerNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy \
  --role-name EKSWorkerNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

aws iam attach-role-policy \
  --role-name EKSWorkerNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
```

## Creating the Cluster

### Using eksctl (Simplest)

eksctl creates both roles automatically:

```bash
eksctl create cluster \
  --name my-cluster \
  --region us-west-2 \
  --version 1.30 \
  --nodegroup-name workers \
  --node-type m5.large \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 5
```

### Using AWS CLI (Manual)

```bash
# Create cluster (uses the EKS cluster role created above)
aws eks create-cluster \
  --name my-cluster \
  --region us-west-2 \
  --kubernetes-version 1.30 \
  --role-arn arn:aws:iam::123456789012:role/EKSClusterRole \
  --resources-vpc-config subnetIds=subnet-abc,subnet-def,securityGroupIds=sg-123

# Wait for cluster to be active
aws eks wait cluster-active --name my-cluster

# Create node group (uses the worker node role created above)
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name workers \
  --node-role arn:aws:iam::123456789012:role/EKSWorkerNodeRole \
  --subnets subnet-abc subnet-def \
  --instance-types m5.large \
  --scaling-config minSize=2,maxSize=5,desiredSize=3
```

### Using Terraform

```hcl
resource "aws_iam_role" "eks_cluster" {
  name = "EKSClusterRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "eks.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster.name
}

resource "aws_eks_cluster" "main" {
  name     = "my-cluster"
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.30"

  vpc_config {
    subnet_ids = var.subnet_ids
  }
}

resource "aws_iam_role" "node_group" {
  name = "EKSWorkerNodeRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "node_worker" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.node_group.name
}

resource "aws_iam_role_policy_attachment" "node_ecr" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.node_group.name
}

resource "aws_iam_role_policy_attachment" "node_cni" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.node_group.name
}

resource "aws_eks_node_group" "workers" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "workers"
  node_role_arn   = aws_iam_role.node_group.arn
  subnet_ids      = var.subnet_ids

  instance_types = ["m5.large"]

  scaling_config {
    desired_size = 3
    min_size     = 2
    max_size     = 5
  }
}
```

## Additional Roles for Common Add-Ons

| Add-On | Additional IAM Needed | Method |
|--------|----------------------|--------|
| AWS Load Balancer Controller | Elasticloadbalancing, EC2, WAF, Shield policies | IRSA (service account) |
| EBS CSI Driver | EC2 volume create/attach/delete | IRSA or Pod Identity |
| EFS CSI Driver | EFS describe/mount | IRSA or Pod Identity |
| Cluster Autoscaler | Autoscaling group describe/modify | IRSA or Pod Identity |
| External DNS | Route53 change/list | IRSA or Pod Identity |
| cert-manager | Route53 (for DNS-01 challenge) | IRSA or Pod Identity |

> **Best practice:** Use IRSA (IAM Roles for Service Accounts) or EKS Pod Identity for add-on permissions instead of attaching them to the node role. This limits blast radius if a pod is compromised.

## Verify Setup

```bash
# Update kubeconfig
aws eks update-kubeconfig --name my-cluster --region us-west-2

# Check cluster
kubectl cluster-info
kubectl get nodes

# Verify node role policies
aws iam list-attached-role-policies --role-name EKSWorkerNodeRole

# Verify cluster role policies
aws iam list-attached-role-policies --role-name EKSClusterRole
```
