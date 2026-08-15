# Creating an AWS EKS Cluster with eksctl

A step-by-step guide to creating production-ready EKS clusters using `eksctl` — from a minimal dev cluster to a multi-AZ production setup with IRSA, add-ons, and private networking.

## Prerequisites

```sh
# eksctl
brew install eksctl         # macOS
# or: https://eksctl.io/installation

# AWS CLI (configured with credentials)
aws sts get-caller-identity   # Verify access

# kubectl
brew install kubectl

# Required IAM permissions (minimum)
# - eks:*
# - ec2:* (VPC, subnets, security groups, instances)
# - iam:* (roles, policies, OIDC)
# - cloudformation:* (eksctl uses CloudFormation)
# - ssm:GetParameter (for AMI lookup)
```

## Quick Start: Minimal Dev Cluster

```sh
eksctl create cluster --name dev-cluster --region us-east-1 \
  --version 1.30 --nodes 2 --node-type t3.medium
```

This creates:
- A new VPC with public and private subnets across 2 AZs
- A managed node group with 2 `t3.medium` nodes
- Default add-ons (vpc-cni, kube-proxy, CoreDNS)
- OIDC provider (for IRSA)

Takes ~15-20 minutes.

## Production Cluster: Full Config File

For production, always use a config file for reproducibility:

```yaml
# cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: prod-cluster
  region: us-east-1
  version: "1.30"
  tags:
    environment: production
    team: platform

# Use existing VPC (recommended for production)
vpc:
  id: vpc-0abc123def456
  subnets:
    private:
      us-east-1a: { id: subnet-aaa111 }
      us-east-1b: { id: subnet-bbb222 }
      us-east-1c: { id: subnet-ccc333 }
    public:
      us-east-1a: { id: subnet-ddd444 }
      us-east-1b: { id: subnet-eee555 }
      us-east-1c: { id: subnet-fff666 }
  clusterEndpoints:
    publicAccess: true
    privateAccess: true

# IAM
iam:
  withOIDC: true
  serviceAccounts:
    - metadata:
        name: aws-load-balancer-controller
        namespace: kube-system
      wellKnownPolicies:
        awsLoadBalancerController: true
    - metadata:
        name: ebs-csi-controller-sa
        namespace: kube-system
      wellKnownPolicies:
        ebsCSIController: true
    - metadata:
        name: external-dns
        namespace: kube-system
      wellKnownPolicies:
        externalDNS: true

# Add-ons
addons:
  - name: vpc-cni
    version: latest
    configurationValues: '{"env":{"ENABLE_PREFIX_DELEGATION":"true"}}'
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver
    version: latest
    serviceAccountRoleARN: arn:aws:iam::123456789012:role/EBSCSIRole

# CloudWatch logging
cloudWatch:
  clusterLogging:
    enableTypes:
      - api
      - audit
      - authenticator
      - controllerManager
      - scheduler

# Managed node groups
managedNodeGroups:
  - name: system
    instanceType: t3.large
    minSize: 2
    maxSize: 3
    desiredCapacity: 2
    privateNetworking: true
    labels:
      role: system
    taints:
      - key: CriticalAddonsOnly
        value: "true"
        effect: NoSchedule
    tags:
      nodegroup-role: system
    iam:
      withAddonPolicies:
        ebs: true
        efs: true
        cloudWatch: true

  - name: workers-az-a
    instanceType: m5.xlarge
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    privateNetworking: true
    availabilityZones: ["us-east-1a"]
    labels:
      role: worker
      topology.kubernetes.io/zone: us-east-1a
    tags:
      nodegroup-role: worker
    iam:
      withAddonPolicies:
        ebs: true
        cloudWatch: true

  - name: workers-az-b
    instanceType: m5.xlarge
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    privateNetworking: true
    availabilityZones: ["us-east-1b"]
    labels:
      role: worker
      topology.kubernetes.io/zone: us-east-1b
    tags:
      nodegroup-role: worker
    iam:
      withAddonPolicies:
        ebs: true
        cloudWatch: true

  - name: workers-az-c
    instanceType: m5.xlarge
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    privateNetworking: true
    availabilityZones: ["us-east-1c"]
    labels:
      role: worker
      topology.kubernetes.io/zone: us-east-1c
    tags:
      nodegroup-role: worker
    iam:
      withAddonPolicies:
        ebs: true
        cloudWatch: true
```

```sh
# Create the cluster
eksctl create cluster -f cluster.yaml

# Verify
kubectl get nodes
kubectl get pods -A
```

## Private Cluster (No Public Endpoint)

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: private-cluster
  region: us-east-1
  version: "1.30"

vpc:
  id: vpc-0abc123
  subnets:
    private:
      us-east-1a: { id: subnet-aaa }
      us-east-1b: { id: subnet-bbb }
      us-east-1c: { id: subnet-ccc }
  clusterEndpoints:
    publicAccess: false
    privateAccess: true

# Private cluster requires these for node bootstrap
privateCluster:
  enabled: true
  additionalEndpointServices:
    - "autoscaling"
    - "logs"

managedNodeGroups:
  - name: workers
    instanceType: m5.large
    minSize: 2
    maxSize: 6
    desiredCapacity: 3
    privateNetworking: true
```

> Private clusters require VPC endpoints for ECR, S3, STS, EC2, and logs for nodes to bootstrap. eksctl creates some automatically with `privateCluster.enabled: true`.

## Cluster with Karpenter

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: karpenter-cluster
  region: us-east-1
  version: "1.30"

iam:
  withOIDC: true

# Use Karpenter instead of managed node groups for workloads
# Keep a small managed node group for system pods (Karpenter itself)
managedNodeGroups:
  - name: system
    instanceType: t3.medium
    minSize: 2
    maxSize: 3
    desiredCapacity: 2
    privateNetworking: true
    labels:
      role: system
    taints:
      - key: CriticalAddonsOnly
        value: "true"
        effect: NoSchedule

# Karpenter IAM setup
karpenter:
  version: "1.0.0"
  createServiceAccount: true
  withSpotInterruptionQueue: true
```

After creation, deploy NodePool and EC2NodeClass resources for Karpenter to provision worker nodes.

## Cluster with Fargate

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: fargate-cluster
  region: us-east-1
  version: "1.30"

fargateProfiles:
  - name: default
    selectors:
      - namespace: default
      - namespace: kube-system
  - name: apps
    selectors:
      - namespace: production
        labels:
          compute: fargate
```

## Spot Instances Configuration

```yaml
managedNodeGroups:
  - name: spot-workers
    instanceTypes: ["m5.large", "m5a.large", "m5.xlarge", "m5a.xlarge", "m6i.large"]
    spot: true
    minSize: 0
    maxSize: 20
    desiredCapacity: 5
    privateNetworking: true
    labels:
      role: worker
      capacity-type: spot
    taints:
      - key: spot
        value: "true"
        effect: PreferNoSchedule
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/prod-cluster: "owned"
```

## GPU Node Group

```yaml
managedNodeGroups:
  - name: gpu-workers
    instanceType: g5.xlarge
    minSize: 0
    maxSize: 4
    desiredCapacity: 0
    privateNetworking: true
    labels:
      role: gpu
      nvidia.com/gpu.present: "true"
    taints:
      - key: nvidia.com/gpu
        value: "true"
        effect: NoSchedule
    tags:
      nodegroup-role: gpu
```

## Post-Creation Setup

### Verify the Cluster

```sh
# Check cluster status
eksctl get cluster --name prod-cluster --region us-east-1

# Check nodes
kubectl get nodes -o wide

# Check system pods
kubectl get pods -A

# Check add-ons
eksctl get addon --cluster prod-cluster --region us-east-1

# Check OIDC provider
eksctl utils associate-iam-oidc-provider --cluster prod-cluster --region us-east-1 --approve
```

### Install AWS Load Balancer Controller

```sh
# If IRSA was set up in the config file, the SA already exists
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=prod-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Install Cluster Autoscaler

```sh
# Create IRSA for autoscaler
eksctl create iamserviceaccount --cluster prod-cluster --region us-east-1 \
  --name cluster-autoscaler --namespace kube-system \
  --attach-policy-arn arn:aws:iam::123456789012:policy/ClusterAutoscalerPolicy \
  --approve

# Deploy
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  -n kube-system \
  --set autoDiscovery.clusterName=prod-cluster \
  --set awsRegion=us-east-1 \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler
```

### Install Metrics Server

```sh
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl top nodes
```

## Updating a Cluster

### Upgrade Kubernetes Version

```sh
# 1. Upgrade control plane
eksctl upgrade cluster --name prod-cluster --region us-east-1 --version 1.31 --approve

# 2. Update default add-ons
eksctl update addon --cluster prod-cluster --region us-east-1 --name vpc-cni --version latest
eksctl update addon --cluster prod-cluster --region us-east-1 --name coredns --version latest
eksctl update addon --cluster prod-cluster --region us-east-1 --name kube-proxy --version latest

# 3. Upgrade node groups (one at a time for zero-downtime)
eksctl upgrade nodegroup --cluster prod-cluster --region us-east-1 --name workers-az-a
eksctl upgrade nodegroup --cluster prod-cluster --region us-east-1 --name workers-az-b
eksctl upgrade nodegroup --cluster prod-cluster --region us-east-1 --name workers-az-c
```

### Add a New Node Group (Blue-Green Node Replacement)

```sh
# 1. Create new node group with updated config
eksctl create nodegroup --cluster prod-cluster --region us-east-1 \
  --name workers-v2 --node-type m5.xlarge --nodes-min 2 --nodes-max 10 --nodes 3 \
  --node-private-networking

# 2. Verify pods migrate to new nodes
kubectl get pods -A -o wide

# 3. Delete old node group
eksctl delete nodegroup --cluster prod-cluster --region us-east-1 --name workers-v1 --drain
```

## Cleanup

```sh
# Delete node groups first (optional, eksctl handles it)
eksctl delete nodegroup --cluster prod-cluster --region us-east-1 --name workers-az-a --drain
eksctl delete nodegroup --cluster prod-cluster --region us-east-1 --name workers-az-b --drain
eksctl delete nodegroup --cluster prod-cluster --region us-east-1 --name workers-az-c --drain

# Delete the entire cluster (including VPC if eksctl created it)
eksctl delete cluster --name prod-cluster --region us-east-1
```

## Troubleshooting

### Cluster Creation Fails

```sh
# Check CloudFormation events
eksctl utils describe-stacks --cluster prod-cluster --region us-east-1

# Check CloudFormation in AWS Console
# Stack name: eksctl-<cluster-name>-cluster
```

### Nodes Not Joining

```sh
# Check node group status
eksctl get nodegroup --cluster prod-cluster --region us-east-1

# Check EC2 instances
aws ec2 describe-instances --filters "Name=tag:eks:cluster-name,Values=prod-cluster" \
  --query "Reservations[].Instances[].{ID:InstanceId, State:State.Name}" --output table

# SSH into node (if SSH key configured) and check kubelet
journalctl -u kubelet -f
```

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `ResourceInUseException` | Cluster name already exists | Use a different name or delete the existing cluster |
| `UnsupportedAvailabilityZoneException` | AZ doesn't support EKS | Remove that AZ from your config |
| `Insufficient capacity` | Instance type not available in AZ | Try a different instance type or AZ |
| `Maximum number of VPCs reached` | VPC limit (5 default) | Request a limit increase or use existing VPC |
| Node `NotReady` | Node can't reach API server | Check security groups, NACLs, and routing |
| Pods `Pending` after creation | No nodes or insufficient resources | Check node group status, scale up |

## Tips

- **Always use a config file** for production clusters — it's version-controllable and reproducible.
- **Use existing VPC** in production — don't let eksctl create one (harder to manage networking later).
- **One node group per AZ** gives you control over per-AZ patching and rollouts.
- **Enable private networking** on all node groups — nodes don't need public IPs.
- **Set up IRSA from day one** — avoid using node instance profiles for pod permissions.
- **Enable control plane logging** immediately — you'll need audit logs when debugging auth issues.
- **Tag everything** — use tags for cost allocation and resource identification.
- **Store the config file in Git** — treat cluster configuration as code.


## Installing Prerequisites on Linux

### AWS CLI v2

```sh
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install --bin-dir /usr/bin --install-dir /usr/bin/aws-cli --update
aws --version

# Configure credentials
aws configure
```

### kubectl (from Amazon S3)

```sh
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.0/2024-05-12/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/
kubectl version --client
```

### eksctl (Linux/ARM)

```sh
# For AMD64
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# Verify checksum (optional)
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

## Connect to an Existing Cluster

```sh
# Update kubeconfig for an existing cluster
aws eks update-kubeconfig --name my-cluster --region us-east-1

# Verify connection
kubectl get nodes
kubectl cluster-info
```

## Verify the Cluster with a Test Deployment

After creating a cluster, deploy a simple app to confirm everything works:

```sh
# Deploy nginx
kubectl create deployment nginx --image=nginx --replicas=2

# Expose it via a LoadBalancer
kubectl expose deployment nginx --type=LoadBalancer --port=80

# Wait for the external IP/DNS
kubectl get svc nginx -w

# Test (once EXTERNAL-IP is assigned)
curl <EXTERNAL-IP>

# Cleanup
kubectl delete svc nginx
kubectl delete deployment nginx
```

## Dry Run (Preview Without Creating)

```sh
# Preview what eksctl would create (outputs YAML)
eksctl create cluster --name my-cluster --region us-east-1 \
  --version 1.30 --node-type t3.medium --nodes 2 --dry-run

# Useful to generate a starter config file
eksctl create cluster --name my-cluster --region us-east-1 \
  --version 1.30 --spot --node-type t3.medium --nodes 2 --dry-run > cluster.yaml
```

## Quick Spot Cluster (One-Liner)

```sh
eksctl create cluster --name spot-cluster --region us-east-1 \
  --version 1.30 --spot --node-type t3.medium --nodes 2 --nodes-min 1 --nodes-max 4
```

## Unmanaged Node Groups (Legacy)

Older eksctl configs use `nodeGroups` instead of `managedNodeGroups`. These create self-managed ASGs:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: legacy-cluster
  region: eu-west-2

nodeGroups:
  - name: workers
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 3
    volumeSize: 20
    ssh:
      publicKeyPath: ~/.ssh/id_rsa.pub
```

> For new clusters, always use `managedNodeGroups`. Managed node groups are easier to upgrade and integrate with EKS rolling updates.

## Granting Cluster Access (aws-auth ConfigMap)

After cluster creation, only the creator has access. To grant access to other IAM users or roles:

### Using eksctl (Recommended)

```sh
# Map an IAM role
eksctl create iamidentitymapping --cluster my-cluster --region us-east-1 \
  --arn arn:aws:iam::123456789012:role/AdminRole \
  --group system:masters --username admin

# Map an IAM user
eksctl create iamidentitymapping --cluster my-cluster --region us-east-1 \
  --arn arn:aws:iam::123456789012:user/john \
  --group system:masters --username john

# Verify
eksctl get iamidentitymapping --cluster my-cluster --region us-east-1
```

### Manual Edit (Direct ConfigMap)

```sh
kubectl edit configmap aws-auth -n kube-system
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::123456789012:role/eksctl-my-cluster-nodegroup-NodeInstanceRole-XXXXX
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/john
      username: john
      groups:
      - system:masters
```

```sh
# View current aws-auth
kubectl get configmap aws-auth -n kube-system -o yaml
```

> Be careful editing aws-auth directly — a syntax error can lock you out. Use `eksctl create iamidentitymapping` when possible.

## Delete Cluster

```sh
# Delete and wait for completion
eksctl delete cluster --name my-cluster --region us-east-1 --wait

# Delete using config file
eksctl delete cluster -f cluster.yaml --wait

# Force delete (if stuck)
eksctl delete cluster --name my-cluster --region us-east-1 --force --disable-nodegroup-eviction
```

## Verify IAM Policies Before Creation

```sh
# Check that the EKS cluster policy exists
aws iam get-policy --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

# Check the current account identity
aws sts get-caller-identity
```
