# EKS Self-Managed Nodes: EC2 Tags vs Node Selectors and Why Draining Is Instance-Based

## The Two Worlds

In EKS self-managed clusters, you operate at two layers that don't automatically talk to each other:

| Layer | Identifier | Used For |
|-------|-----------|----------|
| AWS (EC2) | Instance ID, EC2 tags | Auto Scaling Groups, lifecycle hooks, instance termination |
| Kubernetes | Node name, labels, annotations | Scheduling (nodeSelector, affinity), taints, tolerations |

Understanding where each layer starts and stops is critical for correctly managing node lifecycle.

## EC2 Instance Tags

EC2 tags are AWS-level metadata on the instance. They are used by:

- Auto Scaling Groups (ASG) for grouping and filtering
- AWS CLI / API for targeting instances
- Cost allocation and organization
- EKS for identifying which instances belong to a cluster

```sh
# Tag an instance
aws ec2 create-tags --resources i-0abc123 --tags Key=team,Value=platform Key=env,Value=prod

# List instances by tag
aws ec2 describe-instances --filters "Name=tag:env,Values=prod" --query "Reservations[].Instances[].InstanceId" --output text

# EKS required tags for self-managed nodes
# kubernetes.io/cluster/<cluster-name> = owned
```

EC2 tags are **not visible inside Kubernetes** unless explicitly propagated.

## Kubernetes Node Labels and nodeSelector

Node labels are Kubernetes-level metadata. They are used by:

- `nodeSelector` for basic scheduling constraints
- Node affinity/anti-affinity rules
- Taints and tolerations (indirectly)

```yaml
# Pod with nodeSelector
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  nodeSelector:
    team: platform
    instance-type: compute-optimized
  containers:
    - name: app
      image: my-app:latest
```

```sh
# Label a node
kubectl label node ip-10-0-1-42.ec2.internal team=platform

# List nodes with a label
kubectl get nodes -l team=platform
```

## How EC2 Tags and Node Labels Relate

They don't — at least not automatically. This is the key insight.

| EC2 Tag | Kubernetes Label | Auto-synced? |
|---------|-----------------|--------------|
| `team=platform` | `team=platform` | No |
| `Name=worker-1` | Node name is set from private DNS | Partially (name only) |
| `kubernetes.io/cluster/my-cluster=owned` | N/A (used for discovery only) | No |
| N/A | `node.kubernetes.io/instance-type=m5.xlarge` | Yes (kubelet sets this) |
| N/A | `topology.kubernetes.io/zone=us-east-1a` | Yes (kubelet sets this) |

### What kubelet adds automatically

Kubelet populates well-known labels from instance metadata:

- `node.kubernetes.io/instance-type` — from EC2 instance type
- `topology.kubernetes.io/zone` — from availability zone
- `topology.kubernetes.io/region` — from region
- `kubernetes.io/os` — operating system
- `kubernetes.io/arch` — CPU architecture

### What kubelet does NOT add

- Custom EC2 tags are NOT synced to Kubernetes labels
- ASG tags are NOT synced to Kubernetes labels

If you need custom EC2 tags available as node labels, you must either:

1. Set `--node-labels` in kubelet arguments at launch time (via userdata)
2. Use a controller/operator that watches EC2 tags and syncs them to labels
3. Manually label nodes after they join

### Setting labels via kubelet at boot

In the EC2 userdata (bootstrap script):

```sh
/etc/eks/bootstrap.sh my-cluster \
  --kubelet-extra-args '--node-labels=team=platform,env=prod,workload=compute'
```

This is the most common approach for self-managed nodes.

## Why Draining Is Based on EC2 Instances

### The problem

When you need to terminate a node (scaling down, patching, replacing), the trigger comes from the AWS layer:

1. ASG decides to terminate an instance (scale-in, spot interruption, lifecycle hook)
2. AWS identifies the target by **EC2 Instance ID** (e.g., `i-0abc123def456`)
3. Kubernetes knows the node by its **node name** (e.g., `ip-10-0-1-42.ec2.internal`)

Kubernetes has no concept of "EC2 Instance ID." AWS has no concept of "drain" or "cordon."

### The mapping

To drain a node before termination, you must:

1. Map the EC2 Instance ID → Kubernetes node name
2. Cordon the node (prevent new pods)
3. Drain the node (evict existing pods)
4. Then allow the instance to terminate

```sh
# Get the Kubernetes node name from an EC2 instance ID
INSTANCE_ID="i-0abc123def456"
NODE_NAME=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].PrivateDnsName" --output text)

# Cordon and drain
kubectl cordon $NODE_NAME
kubectl drain $NODE_NAME --ignore-daemonsets --delete-emptydir-data --grace-period=60
```

Reverse mapping (node name → instance ID):

```sh
# From Node Name → Instance ID via AWS API
NODE_NAME="ip-10-0-1-42.ec2.internal"
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=private-dns-name,Values=$NODE_NAME" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)
```

Or without calling AWS at all — Kubernetes stores the instance ID in the `providerID` field:

```sh
# Get Instance ID from the node object
kubectl get node ip-10-0-1-42.ec2.internal -o jsonpath='{.spec.providerID}'
# aws:///us-east-1a/i-0abc123def456

# List all nodes with their instance IDs
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.providerID}{"\n"}{end}'
```

> For managed node groups, EKS handles drain automatically during upgrades and scale-in. For self-managed nodes, you need NTH or lifecycle hooks.

### Why it's instance-based (not label-based)

- **ASG scale-in picks specific instances**: When an ASG scales down, it selects specific instance IDs to terminate based on its termination policy (oldest, newest, closest to billing hour). You can't predict which instance it'll choose based on labels.

- **Spot interruptions target instance IDs**: AWS sends a 2-minute notice for a specific instance ID, not "any node with label X."

- **Lifecycle hooks operate on instance IDs**: ASG lifecycle hooks fire for a specific instance transitioning states. The hook payload contains the instance ID.

- **Node replacement is 1:1 with instances**: Each Kubernetes node IS an EC2 instance. The node name is derived from the instance's private DNS name.

### Lifecycle hook drain pattern

For self-managed nodes, the standard pattern is:

```
ASG Scale-In Event
    ↓
Lifecycle Hook fires (Terminating:Wait)
    ↓
Lambda/controller receives Instance ID
    ↓
Maps Instance ID → Node Name (via private DNS)
    ↓
kubectl cordon + drain
    ↓
Complete lifecycle action (allow termination)
    ↓
Instance terminates
```

Example Lambda logic (pseudocode):

```sh
# From the lifecycle hook event
INSTANCE_ID=$1

# Get the node name
NODE_NAME=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].PrivateDnsName" --output text)

# Drain the node
kubectl cordon $NODE_NAME
kubectl drain $NODE_NAME --ignore-daemonsets --delete-emptydir-data --timeout=120s

# Signal ASG to proceed with termination
aws autoscaling complete-lifecycle-action \
  --lifecycle-hook-name my-drain-hook \
  --auto-scaling-group-name my-asg \
  --lifecycle-action-result CONTINUE \
  --instance-id $INSTANCE_ID
```

### aws-node-termination-handler

AWS provides the [Node Termination Handler](https://github.com/aws/aws-node-termination-handler) (NTH) to automate this:

```sh
# Deploy with Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-node-termination-handler eks/aws-node-termination-handler \
  --namespace kube-system \
  --set enableSpotInterruptionDraining=true \
  --set enableScheduledEventDraining=true
```

NTH handles:
- Spot interruption notices (2-min warning)
- Scheduled maintenance events
- ASG lifecycle hooks
- Rebalance recommendations

It watches EC2 metadata/SQS and automatically cordons + drains the target node before termination.

## Summary

| Concept | Where it lives | What uses it |
|---------|---------------|--------------|
| EC2 tags | AWS layer | ASG, billing, AWS CLI filtering |
| Node labels | Kubernetes layer | Scheduling (nodeSelector, affinity) |
| Instance ID | AWS layer | ASG termination, lifecycle hooks, spot notices |
| Node name | Kubernetes layer | kubectl cordon/drain |

Key takeaways:

- EC2 tags and Kubernetes labels are separate systems — don't assume one knows about the other
- Set node labels at boot time via kubelet `--node-labels` if you need scheduling based on custom metadata
- Draining is triggered by AWS events (instance-level) and executed in Kubernetes (node-level)
- The bridge between the two is the private DNS name mapping
- Use Node Termination Handler to automate the drain-before-terminate workflow
