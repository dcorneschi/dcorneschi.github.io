# AWS CLI EKS Commands Cheatsheet

All `aws eks` commands for managing EKS clusters, node groups, add-ons, identity, and Fargate.

## Cluster Management

### Create a Cluster

```sh
aws eks create-cluster --name <cluster> --region <region> \
  --kubernetes-version 1.30 \
  --role-arn arn:aws:iam::123456789012:role/EKSClusterRole \
  --resources-vpc-config subnetIds=subnet-aaa,subnet-bbb,securityGroupIds=sg-xxx

# With private endpoint only
aws eks create-cluster --name <cluster> --region <region> \
  --kubernetes-version 1.30 \
  --role-arn <role-arn> \
  --resources-vpc-config subnetIds=subnet-aaa,subnet-bbb,endpointPublicAccess=false,endpointPrivateAccess=true

# With custom service CIDR
aws eks create-cluster --name <cluster> --region <region> \
  --kubernetes-version 1.30 \
  --role-arn <role-arn> \
  --resources-vpc-config subnetIds=subnet-aaa,subnet-bbb \
  --kubernetes-network-config serviceIpv4Cidr=172.20.0.0/16

# Auto Mode cluster
aws eks create-cluster --name <cluster> --region <region> \
  --kubernetes-version 1.30 \
  --role-arn <role-arn> \
  --resources-vpc-config subnetIds=subnet-aaa,subnet-bbb \
  --compute-config enabled=true,nodePools=["general-purpose","system"] \
  --kubernetes-network-config elasticLoadBalancing=enabled \
  --storage-config blockStorage=enabled
```

### Describe a Cluster

```sh
aws eks describe-cluster --name <cluster> --region <region>

# Specific fields
aws eks describe-cluster --name <cluster> \
  --query "cluster.{Version:version, Platform:platformVersion, Status:status, Endpoint:endpoint}" \
  --output table

# Get endpoint
aws eks describe-cluster --name <cluster> --query "cluster.endpoint" --output text

# Get certificate authority
aws eks describe-cluster --name <cluster> --query "cluster.certificateAuthority.data" --output text

# Get OIDC issuer
aws eks describe-cluster --name <cluster> --query "cluster.identity.oidc.issuer" --output text

# Get cluster security group
aws eks describe-cluster --name <cluster> --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text

# Get VPC config
aws eks describe-cluster --name <cluster> --query "cluster.resourcesVpcConfig" --output json

# Get service CIDR
aws eks describe-cluster --name <cluster> --query "cluster.kubernetesNetworkConfig.serviceIpv4Cidr" --output text
```

### List Clusters

```sh
aws eks list-clusters --region <region>
aws eks list-clusters --region <region> --output table
```

### Delete a Cluster

```sh
aws eks delete-cluster --name <cluster> --region <region>
```

### Update Cluster Version

```sh
aws eks update-cluster-version --name <cluster> --region <region> --kubernetes-version 1.31
```

### Update Cluster Config

```sh
# Enable private endpoint
aws eks update-cluster-config --name <cluster> --region <region> \
  --resources-vpc-config endpointPublicAccess=true,endpointPrivateAccess=true

# Restrict public access to specific CIDRs
aws eks update-cluster-config --name <cluster> --region <region> \
  --resources-vpc-config endpointPublicAccess=true,publicAccessCidrs="203.0.113.0/24,198.51.100.0/24"

# Enable control plane logging
aws eks update-cluster-config --name <cluster> --region <region> \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# Disable all logging
aws eks update-cluster-config --name <cluster> --region <region> \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":false}]}'

# Enable Auto Mode on existing cluster
aws eks update-cluster-config --name <cluster> --region <region> \
  --compute-config enabled=true,nodePools=["general-purpose","system"]
```

## kubeconfig

```sh
# Update kubeconfig (default ~/.kube/config)
aws eks update-kubeconfig --name <cluster> --region <region>

# Specify a different file
aws eks update-kubeconfig --name <cluster> --region <region> --kubeconfig /tmp/my-config

# Use a specific IAM role for authentication
aws eks update-kubeconfig --name <cluster> --region <region> --role-arn <role-arn>

# Set an alias for the context
aws eks update-kubeconfig --name <cluster> --region <region> --alias my-prod-cluster
```

## Node Groups

### Create a Node Group

```sh
aws eks create-nodegroup --cluster-name <cluster> --region <region> \
  --nodegroup-name workers \
  --node-role arn:aws:iam::123456789012:role/NodeRole \
  --subnets subnet-aaa subnet-bbb subnet-ccc \
  --instance-types m5.large \
  --scaling-config minSize=2,maxSize=10,desiredSize=3

# With labels and taints
aws eks create-nodegroup --cluster-name <cluster> --region <region> \
  --nodegroup-name gpu-workers \
  --node-role <role-arn> \
  --subnets subnet-aaa subnet-bbb \
  --instance-types g5.xlarge \
  --scaling-config minSize=0,maxSize=4,desiredSize=0 \
  --labels gpu=true,team=ml \
  --taints "key=nvidia.com/gpu,value=true,effect=NO_SCHEDULE"

# With Spot capacity
aws eks create-nodegroup --cluster-name <cluster> --region <region> \
  --nodegroup-name spot-workers \
  --node-role <role-arn> \
  --subnets subnet-aaa subnet-bbb subnet-ccc \
  --instance-types m5.large m5a.large m5.xlarge c5.large \
  --capacity-type SPOT \
  --scaling-config minSize=0,maxSize=20,desiredSize=5

# With launch template
aws eks create-nodegroup --cluster-name <cluster> --region <region> \
  --nodegroup-name workers \
  --node-role <role-arn> \
  --subnets subnet-aaa subnet-bbb \
  --launch-template name=my-lt,version=3 \
  --scaling-config minSize=2,maxSize=10,desiredSize=3

# With update config (max unavailable during rolling updates)
aws eks create-nodegroup --cluster-name <cluster> --region <region> \
  --nodegroup-name workers \
  --node-role <role-arn> \
  --subnets subnet-aaa subnet-bbb \
  --instance-types m5.large \
  --scaling-config minSize=2,maxSize=10,desiredSize=3 \
  --update-config maxUnavailable=1

# With SSH access
aws eks create-nodegroup --cluster-name <cluster> --region <region> \
  --nodegroup-name workers \
  --node-role <role-arn> \
  --subnets subnet-aaa subnet-bbb \
  --instance-types m5.large \
  --scaling-config minSize=2,maxSize=6,desiredSize=3 \
  --remote-access ec2SshKey=my-key
```

### List Node Groups

```sh
aws eks list-nodegroups --cluster-name <cluster> --region <region>
```

### Describe a Node Group

```sh
aws eks describe-nodegroup --cluster-name <cluster> --region <region> --nodegroup-name workers

# Key fields
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name workers \
  --query "nodegroup.{Status:status, Size:scalingConfig, Version:version, AMI:amiType, Health:health}" \
  --output json

# Get instance types
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name workers \
  --query "nodegroup.instanceTypes" --output text

# Get node role ARN
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name workers \
  --query "nodegroup.nodeRole" --output text

# Get Auto Scaling Group name
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name workers \
  --query "nodegroup.resources.autoScalingGroups[0].name" --output text
```

### Update Node Group Config

```sh
# Scale
aws eks update-nodegroup-config --cluster-name <cluster> --region <region> \
  --nodegroup-name workers \
  --scaling-config minSize=3,maxSize=15,desiredSize=5

# Update labels
aws eks update-nodegroup-config --cluster-name <cluster> --region <region> \
  --nodegroup-name workers \
  --labels addOrUpdateLabels={environment=prod,team=platform},removeLabels=["old-label"]

# Update taints
aws eks update-nodegroup-config --cluster-name <cluster> --region <region> \
  --nodegroup-name workers \
  --taints addOrUpdateTaints=[{key=dedicated,value=workers,effect=NO_SCHEDULE}]

# Update max unavailable for rolling updates
aws eks update-nodegroup-config --cluster-name <cluster> --region <region> \
  --nodegroup-name workers \
  --update-config maxUnavailable=2

# Update max unavailable percentage
aws eks update-nodegroup-config --cluster-name <cluster> --region <region> \
  --nodegroup-name workers \
  --update-config maxUnavailablePercentage=33
```

### Upgrade Node Group Version

```sh
# Upgrade to latest AMI (same K8s version)
aws eks update-nodegroup-version --cluster-name <cluster> --region <region> \
  --nodegroup-name workers

# Upgrade to specific K8s version
aws eks update-nodegroup-version --cluster-name <cluster> --region <region> \
  --nodegroup-name workers --kubernetes-version 1.31

# Upgrade with new launch template version
aws eks update-nodegroup-version --cluster-name <cluster> --region <region> \
  --nodegroup-name workers --launch-template name=my-lt,version=4

# Force upgrade (ignores PDBs)
aws eks update-nodegroup-version --cluster-name <cluster> --region <region> \
  --nodegroup-name workers --force
```

### Delete a Node Group

```sh
aws eks delete-nodegroup --cluster-name <cluster> --region <region> --nodegroup-name workers
```

## Add-ons

### List Add-ons

```sh
# Installed add-ons
aws eks list-addons --cluster-name <cluster> --region <region>

# Available add-on versions
aws eks describe-addon-versions --addon-name vpc-cni \
  --query "addons[0].addonVersions[].addonVersion" --output table

# All available add-ons
aws eks describe-addon-versions --query "addons[].addonName" --output table
```

### Create an Add-on

```sh
aws eks create-addon --cluster-name <cluster> --region <region> \
  --addon-name vpc-cni --addon-version latest

aws eks create-addon --cluster-name <cluster> --region <region> \
  --addon-name aws-ebs-csi-driver --addon-version latest \
  --service-account-role-arn arn:aws:iam::123456789012:role/EBSCSIRole

# With configuration values
aws eks create-addon --cluster-name <cluster> --region <region> \
  --addon-name vpc-cni \
  --configuration-values '{"env":{"ENABLE_PREFIX_DELEGATION":"true","WARM_PREFIX_TARGET":"1"}}'
```

### Describe an Add-on

```sh
aws eks describe-addon --cluster-name <cluster> --region <region> --addon-name vpc-cni

aws eks describe-addon --cluster-name <cluster> --addon-name vpc-cni \
  --query "{Version:addonVersion, Status:status, Health:health.issues}" --output json
```

### Update an Add-on

```sh
aws eks update-addon --cluster-name <cluster> --region <region> \
  --addon-name vpc-cni --addon-version latest

# With resolve conflicts (overwrite manual changes)
aws eks update-addon --cluster-name <cluster> --region <region> \
  --addon-name vpc-cni --addon-version latest --resolve-conflicts OVERWRITE
```

### Delete an Add-on

```sh
aws eks delete-addon --cluster-name <cluster> --region <region> --addon-name vpc-cni

# Preserve (keep resources but remove EKS management)
aws eks delete-addon --cluster-name <cluster> --region <region> --addon-name vpc-cni --preserve
```

## Access Entries (Authentication)

### Create an Access Entry

```sh
# Standard access entry
aws eks create-access-entry --cluster-name <cluster> --region <region> \
  --principal-arn arn:aws:iam::123456789012:role/DevRole \
  --type STANDARD

# With Kubernetes groups
aws eks create-access-entry --cluster-name <cluster> --region <region> \
  --principal-arn arn:aws:iam::123456789012:role/DevRole \
  --type STANDARD \
  --kubernetes-groups dev-team
```

### Associate an Access Policy

```sh
# Cluster admin
aws eks associate-access-policy --cluster-name <cluster> --region <region> \
  --principal-arn arn:aws:iam::123456789012:role/AdminRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster

# Namespace-scoped
aws eks associate-access-policy --cluster-name <cluster> --region <region> \
  --principal-arn arn:aws:iam::123456789012:role/DevRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy \
  --access-scope type=namespace,namespaces=dev,staging
```

### List Access Entries

```sh
aws eks list-access-entries --cluster-name <cluster> --region <region>
```

### Describe an Access Entry

```sh
aws eks describe-access-entry --cluster-name <cluster> --region <region> \
  --principal-arn arn:aws:iam::123456789012:role/DevRole
```

### List Associated Access Policies

```sh
aws eks list-associated-access-policies --cluster-name <cluster> --region <region> \
  --principal-arn arn:aws:iam::123456789012:role/DevRole
```

### Delete an Access Entry

```sh
aws eks delete-access-entry --cluster-name <cluster> --region <region> \
  --principal-arn arn:aws:iam::123456789012:role/DevRole
```

## Pod Identity

### Create a Pod Identity Association

```sh
aws eks create-pod-identity-association --cluster-name <cluster> --region <region> \
  --namespace my-namespace \
  --service-account my-service-account \
  --role-arn arn:aws:iam::123456789012:role/MyPodRole
```

### List Pod Identity Associations

```sh
aws eks list-pod-identity-associations --cluster-name <cluster> --region <region>

# Filter by namespace
aws eks list-pod-identity-associations --cluster-name <cluster> --region <region> \
  --namespace my-namespace
```

### Describe a Pod Identity Association

```sh
aws eks describe-pod-identity-association --cluster-name <cluster> --region <region> \
  --association-id <association-id>
```

### Delete a Pod Identity Association

```sh
aws eks delete-pod-identity-association --cluster-name <cluster> --region <region> \
  --association-id <association-id>
```

## Fargate

### Create a Fargate Profile

```sh
aws eks create-fargate-profile --cluster-name <cluster> --region <region> \
  --fargate-profile-name my-profile \
  --pod-execution-role-arn arn:aws:iam::123456789012:role/FargatePodRole \
  --subnets subnet-aaa subnet-bbb \
  --selectors namespace=my-namespace

# With label selectors
aws eks create-fargate-profile --cluster-name <cluster> --region <region> \
  --fargate-profile-name my-profile \
  --pod-execution-role-arn <role-arn> \
  --subnets subnet-aaa subnet-bbb \
  --selectors namespace=production,labels={app=web,env=prod}
```

### List Fargate Profiles

```sh
aws eks list-fargate-profiles --cluster-name <cluster> --region <region>
```

### Describe a Fargate Profile

```sh
aws eks describe-fargate-profile --cluster-name <cluster> --region <region> \
  --fargate-profile-name my-profile
```

### Delete a Fargate Profile

```sh
aws eks delete-fargate-profile --cluster-name <cluster> --region <region> \
  --fargate-profile-name my-profile
```

## Updates and Upgrades

### List Updates

```sh
aws eks list-updates --name <cluster> --region <region>

# For a node group
aws eks list-updates --name <cluster> --region <region> --nodegroup-name workers
```

### Describe an Update

```sh
aws eks describe-update --name <cluster> --region <region> --update-id <update-id>

# Node group update
aws eks describe-update --name <cluster> --region <region> \
  --nodegroup-name workers --update-id <update-id>
```

## Token and OIDC

### Get a Token (for kubectl)

```sh
aws eks get-token --cluster-name <cluster> --region <region>

# Output just the token value
aws eks get-token --cluster-name <cluster> --query "status.token" --output text
```

### Describe OIDC Identity Provider

```sh
aws eks describe-identity-provider-config --cluster-name <cluster> --region <region> \
  --identity-provider-config type=oidc,name=my-oidc
```

### Associate OIDC Identity Provider

```sh
aws eks associate-identity-provider-config --cluster-name <cluster> --region <region> \
  --oidc name=my-oidc,issuerUrl=https://token.actions.githubusercontent.com,clientId=sts.amazonaws.com
```

## EKS Insights

```sh
# List insights for a cluster
aws eks list-insights --cluster-name <cluster> --region <region>

# Describe a specific insight
aws eks describe-insight --cluster-name <cluster> --region <region> --id <insight-id>
```

## One-Liners

```sh
# Get all clusters with versions
aws eks list-clusters --output text | tr '\t' '\n' | while read c; do
  v=$(aws eks describe-cluster --name $c --query "cluster.version" --output text)
  echo "$c: v$v"
done

# Get all node groups across all clusters
for cluster in $(aws eks list-clusters --output text | tr '\t' '\n'); do
  echo "=== $cluster ==="
  aws eks list-nodegroups --cluster-name $cluster --output text
done

# Check health of all node groups
aws eks list-nodegroups --cluster-name <cluster> --output text | tr '\t' '\n' | while read ng; do
  status=$(aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name $ng --query "nodegroup.status" --output text)
  echo "$ng: $status"
done

# Get all add-on versions installed
aws eks list-addons --cluster-name <cluster> --output text | tr '\t' '\n' | while read addon; do
  ver=$(aws eks describe-addon --cluster-name <cluster> --addon-name $addon --query "addon.addonVersion" --output text)
  echo "$addon: $ver"
done

# Find clusters with outdated K8s version
for cluster in $(aws eks list-clusters --output text | tr '\t' '\n'); do
  ver=$(aws eks describe-cluster --name $cluster --query "cluster.version" --output text)
  echo "$cluster: $ver"
done | sort -t: -k2 -V

# Get API server endpoint for scripting
ENDPOINT=$(aws eks describe-cluster --name <cluster> --query "cluster.endpoint" --output text)
CA_DATA=$(aws eks describe-cluster --name <cluster> --query "cluster.certificateAuthority.data" --output text)
```

## Waiter Commands

```sh
# Wait for cluster to be active
aws eks wait cluster-active --name <cluster> --region <region>

# Wait for cluster to be deleted
aws eks wait cluster-deleted --name <cluster> --region <region>

# Wait for node group to be active
aws eks wait nodegroup-active --cluster-name <cluster> --region <region> --nodegroup-name workers

# Wait for node group to be deleted
aws eks wait nodegroup-deleted --cluster-name <cluster> --region <region> --nodegroup-name workers

# Wait for add-on to be active
aws eks wait addon-active --cluster-name <cluster> --region <region> --addon-name vpc-cni
```


## Create Cluster with Encryption

```sh
# Encrypt Kubernetes Secrets with a KMS key
aws eks create-cluster --name <cluster> --region <region> \
  --kubernetes-version 1.30 \
  --role-arn <role-arn> \
  --resources-vpc-config subnetIds=subnet-aaa,subnet-bbb \
  --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"arn:aws:kms:<region>:<account>:key/<key-id>"}}]'
```

## Create Cluster with Logging Enabled

```sh
# Enable logging at creation time
aws eks create-cluster --name <cluster> --region <region> \
  --kubernetes-version 1.30 \
  --role-arn <role-arn> \
  --resources-vpc-config subnetIds=subnet-aaa,subnet-bbb \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```

## Add-on Configuration Schema

```sh
# Get the configuration schema for an add-on (what values you can set)
aws eks describe-addon-configuration --addon-name vpc-cni --addon-version v1.18.0-eksbuild.1

# List all versions for a specific add-on
aws eks describe-addon-versions --addon-name vpc-cni \
  --query "addons[0].addonVersions[].{Version:addonVersion, Default:compatibilities[0].defaultVersion}" --output table
```

## Identity Provider — Disassociate

```sh
aws eks disassociate-identity-provider-config --cluster-name <cluster> --region <region> \
  --identity-provider-config type=oidc,name=my-oidc
```

## Fargate — Multiple Selectors

```sh
# Create Fargate profile matching multiple namespaces/labels
aws eks create-fargate-profile --cluster-name <cluster> --region <region> \
  --fargate-profile-name multi-selector \
  --pod-execution-role-arn <role-arn> \
  --subnets subnet-aaa subnet-bbb \
  --selectors namespace=default \
  --selectors namespace=kube-system,labels={k8s-app=kube-dns}
```

## Batch Operations

```sh
# Get all clusters across all regions
for region in $(aws ec2 describe-regions --query "Regions[].RegionName" --output text); do
  clusters=$(aws eks list-clusters --region $region --query "clusters[]" --output text 2>/dev/null)
  if [ -n "$clusters" ]; then
    echo "=== $region ==="
    echo "$clusters" | tr '\t' '\n'
  fi
done

# Update kubeconfig for all clusters in current region
for cluster in $(aws eks list-clusters --query "clusters[]" --output text); do
  aws eks update-kubeconfig --name $cluster --alias $cluster
done

# Check deletion status of a node group
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng> \
  --query "nodegroup.status" --output text
```

## Useful kubectl Aliases

```sh
# Add to ~/.bashrc or ~/.zshrc
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias kex='kubectl exec -it'
alias ka='kubectl apply -f'
alias kdel='kubectl delete'
alias kgp='kubectl get pods'
alias kgn='kubectl get nodes'
alias kgs='kubectl get svc'

# Switch context quickly
kctx() { kubectl config use-context "$1"; }

# Switch namespace quickly
kns() { kubectl config set-context --current --namespace="$1"; }

# Get current context and namespace
alias kc='kubectl config current-context'
alias kns-current='kubectl config view --minify --output "jsonpath={..namespace}"'
```

## Common AWS CLI Parameters

| Parameter | Description |
|-----------|-------------|
| `--cluster-name` | EKS cluster name |
| `--nodegroup-name` | Node group name |
| `--region` | AWS region |
| `--output` | Output format (`json`, `table`, `text`, `yaml`) |
| `--query` | JMESPath query string |
| `--profile` | AWS CLI named profile |
| `--role-arn` | IAM role ARN to assume |
| `--no-paginate` | Disable pagination |
| `--max-items` | Max items to return |
| `--dry-run` | Preview without executing (eksctl) |

## Environment Variables

```sh
export AWS_DEFAULT_REGION=us-east-1
export AWS_DEFAULT_OUTPUT=json
export AWS_PROFILE=my-profile
export KUBECONFIG=~/.kube/config
```


## Enable Secrets Encryption on Existing Cluster

```sh
# Associate encryption config (requires existing KMS key)
aws eks associate-encryption-config --cluster-name <cluster> --region <region> \
  --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"arn:aws:kms:<region>:<account>:key/<key-id>"}}]'
```

## Stream CloudWatch Logs

```sh
# Tail control plane logs in real-time
aws logs tail /aws/eks/<cluster>/cluster --follow

# Filter by log stream (e.g., audit logs)
aws logs tail /aws/eks/<cluster>/cluster --follow --log-stream-name-prefix kube-apiserver-audit

# Last 30 minutes
aws logs tail /aws/eks/<cluster>/cluster --since 30m
```

## VPC CNI Configuration

```sh
# Enable prefix delegation (more pods per node)
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
kubectl set env daemonset aws-node -n kube-system WARM_PREFIX_TARGET=1

# Enable network policy enforcement
kubectl set env daemonset aws-node -n kube-system ENABLE_NETWORK_POLICY=true

# Check current CNI settings
kubectl get daemonset aws-node -n kube-system -o jsonpath='{.spec.template.spec.containers[0].env}' | jq
```

## ALB Ingress with SSL

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:<region>:<account>:certificate/<cert-id>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

## Container Insights (Quick Install)

```sh
# Install CloudWatch agent + Fluent Bit for container monitoring
curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml \
  | sed "s/{{cluster_name}}/<cluster>/;s/{{region_name}}/<region>/" \
  | kubectl apply -f -
```

## Node Maintenance

```sh
# Drain a node (evict pods before maintenance/termination)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Uncordon a node (allow scheduling again)
kubectl uncordon <node-name>

# Cordon without draining (stop new pods, keep existing)
kubectl cordon <node-name>
```

## Context Switching

```sh
# Update kubeconfig with aliases for multiple clusters
aws eks update-kubeconfig --name prod-cluster --region us-east-1 --alias prod
aws eks update-kubeconfig --name dev-cluster --region us-east-1 --alias dev

# Switch between clusters
kubectl config use-context prod
kubectl config use-context dev

# View current context
kubectl config current-context

# List all contexts
kubectl config get-contexts
```

## Cluster Autoscaler Annotation

```sh
# Prevent the autoscaler pod from being evicted by itself
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler \
  cluster-autoscaler.kubernetes.io/safe-to-evict="false"
```
