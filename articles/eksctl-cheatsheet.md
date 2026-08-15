# eksctl Cheatsheet

## Installation

```sh
# macOS
brew install eksctl

# Linux
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz"
tar -xzf eksctl_Linux_amd64.tar.gz -C /usr/local/bin

# Verify
eksctl version
```

## Cluster Management

### Create a Cluster

```sh
# Basic cluster (default settings)
eksctl create cluster --name my-cluster --region us-east-1

# With specific version and node count
eksctl create cluster --name my-cluster --region us-east-1 \
  --version 1.30 --nodes 3 --node-type m5.large

# Without node group (control plane only)
eksctl create cluster --name my-cluster --region us-east-1 \
  --version 1.30 --without-nodegroup

# With private subnets only
eksctl create cluster --name my-cluster --region us-east-1 \
  --vpc-private-subnets subnet-aaa,subnet-bbb,subnet-ccc \
  --without-nodegroup

# With existing VPC
eksctl create cluster --name my-cluster --region us-east-1 \
  --vpc-public-subnets subnet-aaa,subnet-bbb \
  --vpc-private-subnets subnet-ccc,subnet-ddd

# Dry run (preview what would be created)
eksctl create cluster --name my-cluster --region us-east-1 --dry-run
```

### Create from Config File

```sh
eksctl create cluster -f cluster.yaml
```

```yaml
# cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: us-east-1
  version: "1.30"

vpc:
  subnets:
    private:
      us-east-1a: { id: subnet-aaa }
      us-east-1b: { id: subnet-bbb }
      us-east-1c: { id: subnet-ccc }

managedNodeGroups:
  - name: workers
    instanceType: m5.large
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    privateNetworking: true
    labels:
      role: worker
    tags:
      environment: production
```

### List Clusters

```sh
eksctl get cluster
eksctl get cluster --region us-east-1
eksctl get cluster --name my-cluster --region us-east-1
```

### Delete a Cluster

```sh
eksctl delete cluster --name my-cluster --region us-east-1

# Force delete (skip drain)
eksctl delete cluster --name my-cluster --region us-east-1 --force

# Wait for deletion to complete
eksctl delete cluster --name my-cluster --region us-east-1 --wait
```

### Update Cluster

```sh
# Upgrade Kubernetes version
eksctl upgrade cluster --name my-cluster --region us-east-1 --version 1.31

# Approve the upgrade (after review)
eksctl upgrade cluster --name my-cluster --region us-east-1 --version 1.31 --approve
```

### Get Cluster Info

```sh
# Show cluster details
eksctl get cluster --name my-cluster --region us-east-1 -o yaml

# Get kubeconfig
eksctl utils write-kubeconfig --cluster my-cluster --region us-east-1
```

## Node Group Management

### Create a Managed Node Group

```sh
# Basic node group
eksctl create nodegroup --cluster my-cluster --region us-east-1 \
  --name workers --node-type m5.large --nodes 3

# With min/max scaling
eksctl create nodegroup --cluster my-cluster --region us-east-1 \
  --name workers --node-type m5.large \
  --nodes-min 2 --nodes-max 10 --nodes 3

# Spot instances
eksctl create nodegroup --cluster my-cluster --region us-east-1 \
  --name spot-workers --node-type m5.large --spot

# Mixed instances (spot + on-demand)
eksctl create nodegroup --cluster my-cluster --region us-east-1 \
  --name mixed-workers --instance-types m5.large,m5a.large,m5.xlarge --spot

# With labels and taints
eksctl create nodegroup --cluster my-cluster --region us-east-1 \
  --name gpu-workers --node-type g4dn.xlarge \
  --node-labels "gpu=true,team=ml" \
  --node-taints "nvidia.com/gpu=true:NoSchedule"

# Private networking (no public IP on nodes)
eksctl create nodegroup --cluster my-cluster --region us-east-1 \
  --name private-workers --node-type m5.large --node-private-networking

# With SSH access
eksctl create nodegroup --cluster my-cluster --region us-east-1 \
  --name workers --node-type m5.large --ssh-access --ssh-public-key my-key

# ARM/Graviton instances
eksctl create nodegroup --cluster my-cluster --region us-east-1 \
  --name graviton-workers --node-type m6g.large --node-ami-family AmazonLinux2023

# Bottlerocket OS
eksctl create nodegroup --cluster my-cluster --region us-east-1 \
  --name bottlerocket --node-type m5.large --node-ami-family Bottlerocket
```

### Create from Config File

```sh
eksctl create nodegroup -f nodegroup.yaml
```

```yaml
# nodegroup.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: us-east-1

managedNodeGroups:
  - name: workers
    instanceType: m5.large
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    privateNetworking: true
    labels:
      role: worker
      environment: production
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
    iam:
      withAddonPolicies:
        ebs: true
        efs: true
        albIngress: true
        cloudWatch: true

  - name: spot-workers
    instanceTypes: ["m5.large", "m5a.large", "m5.xlarge", "m5a.xlarge"]
    spot: true
    minSize: 0
    maxSize: 20
    desiredCapacity: 5
    labels:
      role: worker
      capacity-type: spot
    taints:
      - key: spot
        value: "true"
        effect: PreferNoSchedule
```

### List Node Groups

```sh
eksctl get nodegroup --cluster my-cluster --region us-east-1
eksctl get nodegroup --cluster my-cluster --region us-east-1 -o yaml
```

### Scale a Node Group

```sh
# Scale to specific count
eksctl scale nodegroup --cluster my-cluster --region us-east-1 \
  --name workers --nodes 5

# Scale with new min/max
eksctl scale nodegroup --cluster my-cluster --region us-east-1 \
  --name workers --nodes 5 --nodes-min 3 --nodes-max 10
```

### Delete a Node Group

```sh
eksctl delete nodegroup --cluster my-cluster --region us-east-1 --name workers

# Drain nodes before deletion
eksctl delete nodegroup --cluster my-cluster --region us-east-1 --name workers --drain

# Skip drain (faster but risky)
eksctl delete nodegroup --cluster my-cluster --region us-east-1 --name workers --disable-eviction
```

### Upgrade Node Group

```sh
# Upgrade node group to match cluster version
eksctl upgrade nodegroup --cluster my-cluster --region us-east-1 --name workers

# Upgrade to specific version
eksctl upgrade nodegroup --cluster my-cluster --region us-east-1 \
  --name workers --kubernetes-version 1.31
```

## Labels and Taints

### Set Labels

```sh
eksctl set labels --cluster my-cluster --region us-east-1 \
  --nodegroup workers --labels environment=prod,team=platform
```

### Unset Labels

```sh
eksctl unset labels --cluster my-cluster --region us-east-1 \
  --nodegroup workers --labels environment,team
```

## IAM and OIDC

### Associate OIDC Provider

```sh
# Required for IRSA (IAM Roles for Service Accounts)
eksctl utils associate-iam-oidc-provider --cluster my-cluster --region us-east-1 --approve
```

### Create IAM Service Account

```sh
# Create a service account with an IAM role
eksctl create iamserviceaccount --cluster my-cluster --region us-east-1 \
  --name my-app-sa --namespace my-namespace \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# With multiple policies
eksctl create iamserviceaccount --cluster my-cluster --region us-east-1 \
  --name my-app-sa --namespace default \
  --attach-policy-arn arn:aws:iam::123456789012:policy/MyPolicy1 \
  --attach-policy-arn arn:aws:iam::123456789012:policy/MyPolicy2 \
  --approve --override-existing-serviceaccounts
```

### List IAM Service Accounts

```sh
eksctl get iamserviceaccount --cluster my-cluster --region us-east-1
```

### Delete IAM Service Account

```sh
eksctl delete iamserviceaccount --cluster my-cluster --region us-east-1 \
  --name my-app-sa --namespace my-namespace
```

## Add-ons

### List Add-ons

```sh
eksctl get addon --cluster my-cluster --region us-east-1
```

### Create Add-on

```sh
eksctl create addon --cluster my-cluster --region us-east-1 \
  --name vpc-cni --version latest

eksctl create addon --cluster my-cluster --region us-east-1 \
  --name coredns --version latest

eksctl create addon --cluster my-cluster --region us-east-1 \
  --name kube-proxy --version latest

eksctl create addon --cluster my-cluster --region us-east-1 \
  --name aws-ebs-csi-driver --version latest \
  --service-account-role-arn arn:aws:iam::123456789012:role/EBSCSIRole
```

### Update Add-on

```sh
eksctl update addon --cluster my-cluster --region us-east-1 \
  --name vpc-cni --version latest
```

### Delete Add-on

```sh
eksctl delete addon --cluster my-cluster --region us-east-1 --name vpc-cni
```

## Cluster Access (Identity Mappings)

### Add IAM Role Mapping

```sh
# Map an IAM role to a Kubernetes group
eksctl create iamidentitymapping --cluster my-cluster --region us-east-1 \
  --arn arn:aws:iam::123456789012:role/AdminRole \
  --group system:masters --username admin

# Map with specific Kubernetes groups
eksctl create iamidentitymapping --cluster my-cluster --region us-east-1 \
  --arn arn:aws:iam::123456789012:role/DevRole \
  --group dev-group --username dev-user
```

### Add IAM User Mapping

```sh
eksctl create iamidentitymapping --cluster my-cluster --region us-east-1 \
  --arn arn:aws:iam::123456789012:user/john \
  --group system:masters --username john
```

### List Identity Mappings

```sh
eksctl get iamidentitymapping --cluster my-cluster --region us-east-1
```

### Delete Identity Mapping

```sh
eksctl delete iamidentitymapping --cluster my-cluster --region us-east-1 \
  --arn arn:aws:iam::123456789012:role/DevRole
```

## Utilities

### Enable CloudWatch Logging

```sh
eksctl utils update-cluster-logging --cluster my-cluster --region us-east-1 \
  --enable-types api,audit,authenticator,controllerManager,scheduler --approve
```

### Disable CloudWatch Logging

```sh
eksctl utils update-cluster-logging --cluster my-cluster --region us-east-1 \
  --disable-types all --approve
```

### Update Cluster Endpoint Access

```sh
# Private + public
eksctl utils update-cluster-endpoints --cluster my-cluster --region us-east-1 \
  --private-access=true --public-access=true --approve

# Private only
eksctl utils update-cluster-endpoints --cluster my-cluster --region us-east-1 \
  --private-access=true --public-access=false --approve
```

### Update kubeconfig

```sh
eksctl utils write-kubeconfig --cluster my-cluster --region us-east-1

# Set a specific kubeconfig file
eksctl utils write-kubeconfig --cluster my-cluster --region us-east-1 \
  --kubeconfig ~/.kube/my-cluster-config
```

### Describe Cluster Stacks (CloudFormation)

```sh
eksctl utils describe-stacks --cluster my-cluster --region us-east-1
```

## Fargate

### Create Fargate Profile

```sh
eksctl create fargateprofile --cluster my-cluster --region us-east-1 \
  --name my-fargate --namespace my-namespace

# With label selectors
eksctl create fargateprofile --cluster my-cluster --region us-east-1 \
  --name my-fargate --namespace my-namespace --labels app=web,env=prod
```

### List Fargate Profiles

```sh
eksctl get fargateprofile --cluster my-cluster --region us-east-1
```

### Delete Fargate Profile

```sh
eksctl delete fargateprofile --cluster my-cluster --region us-east-1 --name my-fargate
```

## Drain and Cordon

```sh
# Drain a node group (before deletion or maintenance)
eksctl drain nodegroup --cluster my-cluster --region us-east-1 --name workers

# Undrain (uncordon) a node group
eksctl drain nodegroup --cluster my-cluster --region us-east-1 --name workers --undo
```

## Useful Flags

| Flag | Description |
|------|-------------|
| `--region` | AWS region |
| `--profile` | AWS CLI profile to use |
| `--verbose` | Set verbosity level (0-5) |
| `--timeout` | Timeout for operations (default 25m) |
| `--dry-run` | Preview without creating resources |
| `--approve` | Auto-approve changes (no prompt) |
| `-f` | Use a config file |
| `-o yaml` / `-o json` | Output format |
| `--cfn-role-arn` | IAM role for CloudFormation |

## Quick Reference

```sh
# Full lifecycle
eksctl create cluster -f cluster.yaml              # Create
eksctl get cluster --name my-cluster               # Verify
eksctl create nodegroup -f nodegroup.yaml          # Add nodes
eksctl get nodegroup --cluster my-cluster          # Check nodes
eksctl scale nodegroup --cluster my-cluster --name workers --nodes 5   # Scale
eksctl upgrade cluster --name my-cluster --version 1.31 --approve      # Upgrade CP
eksctl upgrade nodegroup --cluster my-cluster --name workers           # Upgrade nodes
eksctl delete nodegroup --cluster my-cluster --name workers --drain    # Remove nodes
eksctl delete cluster --name my-cluster                                # Destroy

# Common operations
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve
eksctl create iamserviceaccount --cluster my-cluster --name sa --namespace ns --attach-policy-arn <arn> --approve
eksctl utils update-cluster-logging --cluster my-cluster --enable-types all --approve
eksctl set labels --cluster my-cluster --nodegroup workers --labels key=value
```


## Additional Cluster Creation Options

```sh
# Specify availability zones explicitly
eksctl create cluster --name my-cluster --region us-east-1 \
  --zones us-east-1a,us-east-1b,us-east-1c --without-nodegroup

# With a specific CIDR for the VPC
eksctl create cluster --name my-cluster --region us-east-1 \
  --vpc-cidr 10.10.0.0/16

# With NAT Gateway mode (single or highly-available)
eksctl create cluster --name my-cluster --region us-east-1 \
  --vpc-nat-mode HighlyAvailable

# With Kubernetes service CIDR
eksctl create cluster --name my-cluster --region us-east-1 \
  --service-cidr 172.20.0.0/16
```

## Additional Node Group Options

```sh
# Override max pods per node (useful with prefix delegation)
eksctl create nodegroup --cluster my-cluster --region us-east-1 \
  --name workers --node-type m5.large --max-pods-per-node 110

# With custom AMI
eksctl create nodegroup --cluster my-cluster --region us-east-1 \
  --name custom-workers --node-type m5.large \
  --node-ami ami-0abc123def456 --node-ami-family AmazonLinux2023

# With pre-bootstrap and post-bootstrap commands
eksctl create nodegroup --cluster my-cluster --region us-east-1 \
  --name workers --node-type m5.large \
  --pre-bootstrap-commands "yum install -y amazon-ssm-agent && systemctl enable amazon-ssm-agent"

# With a specific volume size and type
eksctl create nodegroup --cluster my-cluster --region us-east-1 \
  --name workers --node-type m5.large \
  --node-volume-size 100 --node-volume-type gp3

# Windows node group
eksctl create nodegroup --cluster my-cluster --region us-east-1 \
  --name windows-workers --node-type m5.large --node-ami-family WindowsServer2022FullContainer
```

## Node Group Health

```sh
# Check node group health status
eksctl utils nodegroup-health --cluster my-cluster --region us-east-1 --name workers
```

## VPC Controllers (Security Groups for Pods)

```sh
# Install VPC admission webhook (required for SG on pods)
eksctl utils install-vpc-controllers --cluster my-cluster --region us-east-1 --approve
```

## Register External Cluster

Connect a non-EKS Kubernetes cluster to the EKS console:

```sh
# Register an external cluster
eksctl register cluster --name external-cluster --provider OTHER --region us-east-1

# Deregister
eksctl deregister cluster --name external-cluster --region us-east-1
```

## EKS Anywhere

```sh
# Create an EKS Anywhere cluster (on-prem / bare metal)
eksctl anywhere create cluster -f cluster-config.yaml

# Upgrade an EKS Anywhere cluster
eksctl anywhere upgrade cluster -f cluster-config.yaml

# Delete an EKS Anywhere cluster
eksctl anywhere delete cluster -f cluster-config.yaml
```

## GitOps (Flux Integration)

```sh
# Enable GitOps with Flux on cluster creation
eksctl enable flux --cluster my-cluster --region us-east-1 \
  --git-url git@github.com:org/fleet-repo.git \
  --git-branch main \
  --namespace flux-system
```

## Pod Identity Associations

```sh
# Create a pod identity association (newer alternative to IRSA)
eksctl create podidentityassociation --cluster my-cluster --region us-east-1 \
  --namespace my-namespace --service-account-name my-sa \
  --role-arn arn:aws:iam::123456789012:role/MyPodRole

# List pod identity associations
eksctl get podidentityassociation --cluster my-cluster --region us-east-1

# Delete a pod identity association
eksctl delete podidentityassociation --cluster my-cluster --region us-east-1 \
  --namespace my-namespace --service-account-name my-sa
```

## Access Entries (Newer Auth Method)

```sh
# Create an access entry
eksctl create accessentry --cluster my-cluster --region us-east-1 \
  --principal-arn arn:aws:iam::123456789012:role/DevRole \
  --kubernetes-groups dev-group \
  --type STANDARD

# List access entries
eksctl get accessentry --cluster my-cluster --region us-east-1

# Delete an access entry
eksctl delete accessentry --cluster my-cluster --region us-east-1 \
  --principal-arn arn:aws:iam::123456789012:role/DevRole
```
