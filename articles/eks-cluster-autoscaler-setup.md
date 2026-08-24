# Cluster Autoscaler on EKS

How to deploy, configure, and operate the Kubernetes Cluster Autoscaler on Amazon EKS — IAM setup, Helm installation, tuning, and troubleshooting.

Related: [Cluster Autoscaler Tuning Guide](articles/kubernetes-cluster-autoscaler-tuning.md) | [Cluster Autoscaler Scale-Up Troubleshooting](articles/kubernetes-cluster-autoscaler-scale-up-troubleshooting.md) | [Cluster Autoscaler vs Karpenter](articles/eks-cluster-autoscaler-vs-karpenter.md)

## How It Works on EKS

```
Pending pod (unschedulable)
  │
  ▼
Cluster Autoscaler evaluates ASGs
  │
  │ 1. Filters ASGs that can satisfy pod requirements
  │ 2. Expander picks which ASG to scale
  │ 3. Increments ASG desired capacity
  │
  ▼
ASG launches new EC2 instance
  │
  ▼
Node joins cluster → pod schedules
```

The Cluster Autoscaler (CA) runs as a Deployment in your cluster. It watches for pods that can't schedule due to insufficient resources, then scales up the appropriate Auto Scaling Group. It also scales down nodes that are underutilized.

## Prerequisites

- EKS cluster with managed or self-managed node groups (ASGs)
- IAM permissions for the CA to manage ASGs
- Node groups tagged for auto-discovery

## Step 1: IAM Policy

Create an IAM policy that allows the CA to describe and modify Auto Scaling Groups:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeImages",
        "ec2:GetInstanceTypesFromInstanceRequirements",
        "eks:DescribeNodegroup"
      ],
      "Resource": "*"
    }
  ]
}
```

```bash
aws iam create-policy \
  --policy-name ClusterAutoscalerPolicy \
  --policy-document file://cluster-autoscaler-policy.json
```

## Step 2: IAM Role (IRSA or Pod Identity)

### Option A: IRSA (IAM Roles for Service Accounts)

```bash
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace kube-system \
  --name cluster-autoscaler \
  --attach-policy-arn arn:aws:iam::123456789012:policy/ClusterAutoscalerPolicy \
  --approve
```

### Option B: EKS Pod Identity (Recommended for new clusters)

```bash
# Create the Pod Identity association
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace kube-system \
  --service-account cluster-autoscaler \
  --role-arn arn:aws:iam::123456789012:role/ClusterAutoscalerRole
```

## Step 3: Tag ASGs for Auto-Discovery

The CA discovers which ASGs to manage via tags. Add these tags to your node group ASGs:

| Tag Key | Tag Value |
|---------|-----------|
| `k8s.io/cluster-autoscaler/enabled` | `true` |
| `k8s.io/cluster-autoscaler/<cluster-name>` | `owned` |

```bash
# Tag an existing ASG
aws autoscaling create-or-update-tags --tags \
  "ResourceId=my-asg-name,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/enabled,Value=true,PropagateAtLaunch=true" \
  "ResourceId=my-asg-name,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/my-cluster,Value=owned,PropagateAtLaunch=true"
```

> If you created node groups with eksctl, these tags are added automatically.

## Step 4: Install with Helm

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update

helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=us-west-2 \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.skip-nodes-with-system-pods=false \
  --set extraArgs.expander=least-waste
```

### Verify Installation

```bash
kubectl get deployment cluster-autoscaler -n kube-system
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-cluster-autoscaler --tail=50
```

## Step 5: Verify Auto-Discovery

```bash
# Check CA logs for discovered node groups
kubectl -n kube-system logs -l app.kubernetes.io/name=aws-cluster-autoscaler | grep "node group"

# Check CA status configmap
kubectl -n kube-system get configmap cluster-autoscaler-status -o yaml
```

You should see your ASGs listed under "NodeGroups" in the status.

## Configuration Options

### Key Flags

| Flag | Default | Purpose |
|------|---------|---------|
| `--scale-down-utilization-threshold` | 0.5 | Remove nodes below this utilization |
| `--scale-down-unneeded-time` | 10m | How long a node must be underutilized before removal |
| `--scale-up-delay-after-add` | 10m | Wait before scaling up after adding a node |
| `--max-node-provision-time` | 15m | Max wait for a node to become ready |
| `--expander` | random | Strategy for choosing which ASG to scale (random, least-waste, most-pods, priority) |
| `--balance-similar-node-groups` | false | Balance node count across similar ASGs |
| `--skip-nodes-with-local-storage` | true | Don't scale down nodes with local volumes |
| `--skip-nodes-with-system-pods` | true | Don't scale down nodes with kube-system pods |
| `--scan-interval` | 10s | How often CA evaluates the cluster |

### Helm Values Example (Production)

```yaml
# cluster-autoscaler-values.yaml
autoDiscovery:
  clusterName: my-cluster

awsRegion: us-west-2

rbac:
  serviceAccount:
    create: false
    name: cluster-autoscaler

extraArgs:
  balance-similar-node-groups: true
  skip-nodes-with-system-pods: false
  expander: least-waste
  scale-down-utilization-threshold: "0.5"
  scale-down-unneeded-time: 10m
  scale-up-delay-after-add: 5m
  max-node-provision-time: 10m

resources:
  requests:
    cpu: 100m
    memory: 300Mi
  limits:
    cpu: 500m
    memory: 500Mi

podAnnotations:
  cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
```

```bash
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  -n kube-system -f cluster-autoscaler-values.yaml
```

## Expander Strategies

The expander decides which ASG to scale when multiple can fit the pending pods:

| Expander | Behavior | Best For |
|----------|----------|----------|
| `random` | Pick any eligible ASG randomly | Simple setups |
| `least-waste` | Pick ASG with least idle resources after scaling | Cost optimization |
| `most-pods` | Pick ASG that can fit the most pending pods | Fast scheduling |
| `priority` | Use a priority ConfigMap to rank ASGs | Specific ordering (e.g., Spot first) |

### Priority Expander ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-priority-expander
  namespace: kube-system
data:
  priorities: |-
    10:
      - .*spot.*            # Try Spot ASGs first (lowest priority number = tried first... wait, no)
    50:
      - .*on-demand.*       # Fall back to On-Demand
```

> **Note:** Higher number = higher priority. ASGs matching priority 50 are preferred over those matching priority 10.

## Node Groups Best Practices for EKS

### Managed Node Groups (Recommended)

```bash
# Create with eksctl
eksctl create nodegroup \
  --cluster my-cluster \
  --name general-workers \
  --node-type m5.xlarge \
  --nodes-min 2 \
  --nodes-max 20 \
  --asg-access
```

eksctl automatically:
- Creates the ASG with proper tags
- Configures launch templates
- Sets up the IAM node role

### Multiple Node Groups for Different Workloads

```yaml
# eksctl config
managedNodeGroups:
  - name: general
    instanceTypes: ["m5.large", "m5.xlarge"]
    minSize: 2
    maxSize: 20
    labels:
      workload: general

  - name: compute
    instanceTypes: ["c5.xlarge", "c5.2xlarge"]
    minSize: 0
    maxSize: 10
    labels:
      workload: compute
    taints:
      - key: compute-only
        value: "true"
        effect: NoSchedule

  - name: spot
    instanceTypes: ["m5.large", "m5.xlarge", "m5a.large", "m5a.xlarge"]
    minSize: 0
    maxSize: 30
    spot: true
    labels:
      lifecycle: spot
```

## Scale-Down Protection

### Prevent Specific Nodes from Scale-Down

Annotate nodes you want to keep:

```bash
kubectl annotate node <node-name> cluster-autoscaler.kubernetes.io/scale-down-disabled=true
```

### Prevent Specific Pods from Blocking Scale-Down

Mark pods as safe to evict (e.g., on DaemonSets or replaceable workloads):

```yaml
metadata:
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
```

### What Blocks Scale-Down

The CA will NOT remove a node if it has:
- Pods with PDBs that would be violated
- Pods not managed by a controller (bare pods)
- Pods with local storage (unless `--skip-nodes-with-local-storage=false`)
- kube-system pods (unless `--skip-nodes-with-system-pods=false`)
- Pods with the annotation `cluster-autoscaler.kubernetes.io/safe-to-evict=false`

## Overprovisioning (Buffer Nodes)

Keep spare capacity so new pods don't wait for scale-up:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: overprovisioning
value: -1
globalDefault: false
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overprovisioning
  namespace: kube-system
spec:
  replicas: 3
  selector:
    matchLabels:
      run: overprovisioning
  template:
    metadata:
      labels:
        run: overprovisioning
    spec:
      priorityClassName: overprovisioning
      containers:
      - name: pause
        image: registry.k8s.io/pause:3.9
        resources:
          requests:
            cpu: "1"
            memory: 2Gi
```

How it works:
1. Pause pods consume resources but do nothing
2. When a real pod arrives and there's no room, it preempts a pause pod
3. Real pod starts immediately on the freed capacity
4. Evicted pause pod goes Pending → triggers CA to add a node
5. New node comes up → pause pod schedules → buffer restored

## Troubleshooting

### CA Not Scaling Up

```bash
# Check for pending pods
kubectl get pods -A --field-selector status.phase=Pending

# Check CA logs for scale-up decisions
kubectl -n kube-system logs -l app.kubernetes.io/name=aws-cluster-autoscaler --tail=200 | grep -iE "scale.up|unschedulable|could not"

# Check CA status
kubectl -n kube-system get configmap cluster-autoscaler-status -o yaml

# Verify ASG tags are correct
aws autoscaling describe-auto-scaling-groups \
  --query "AutoScalingGroups[?contains(Tags[?Key=='k8s.io/cluster-autoscaler/enabled'].Value, 'true')].{Name:AutoScalingGroupName,Min:MinSize,Max:MaxSize,Desired:DesiredCapacity}" \
  --output table
```

Common causes:
- ASG already at max size
- Pod requests more resources than any node in the ASG can provide
- Node selectors/affinity don't match any ASG
- Missing IAM permissions

### CA Not Scaling Down

```bash
# Check which nodes CA considers underutilized
kubectl -n kube-system logs -l app.kubernetes.io/name=aws-cluster-autoscaler | grep "underutilized"

# Check what's blocking scale-down on a specific node
kubectl -n kube-system logs -l app.kubernetes.io/name=aws-cluster-autoscaler | grep "<node-name>"

# List nodes with scale-down disabled annotation
kubectl get nodes -o json | jq '.items[] | select(.metadata.annotations["cluster-autoscaler.kubernetes.io/scale-down-disabled"]=="true") | .metadata.name'
```

Common causes:
- Pods with PDBs blocking eviction
- Bare pods (not managed by a Deployment/StatefulSet)
- Pods with local storage
- Node utilization above threshold
- `--scale-down-unneeded-time` hasn't elapsed yet

### CA Pod Itself Not Running

```bash
# Check the deployment
kubectl get deployment cluster-autoscaler -n kube-system

# Check events
kubectl describe deployment cluster-autoscaler -n kube-system

# Check if IRSA/Pod Identity is working
kubectl -n kube-system get sa cluster-autoscaler -o yaml | grep -A5 annotations
```

## Monitoring

### Key Metrics (Prometheus)

```
# Cluster Autoscaler specific
cluster_autoscaler_scaled_up_nodes_total
cluster_autoscaler_scaled_down_nodes_total
cluster_autoscaler_unschedulable_pods_count
cluster_autoscaler_nodes_count
cluster_autoscaler_function_duration_seconds

# Useful Kubernetes metrics
kube_pod_status_phase{phase="Pending"}
kube_node_status_condition{condition="Ready",status="true"}
```

### Datadog Queries

```
# Pending pods (should trigger scale-up)
avg:kubernetes.pods.running{phase:pending,kube_cluster_name:my-cluster}

# Node count over time
avg:kubernetes.nodes.count{kube_cluster_name:my-cluster}

# ASG desired vs actual
avg:aws.autoscaling.group_desired_capacity{autoscalinggroupname:my-asg} by {autoscalinggroupname}
avg:aws.autoscaling.group_in_service_instances{autoscalinggroupname:my-asg} by {autoscalinggroupname}
```

## Version Compatibility

Match the CA version to your Kubernetes version:

| EKS Kubernetes Version | Cluster Autoscaler Version |
|------------------------|---------------------------|
| 1.30 | 1.30.x |
| 1.29 | 1.29.x |
| 1.28 | 1.28.x |
| 1.27 | 1.27.x |

Always use the CA version that matches your cluster's minor version. Check the [releases page](https://github.com/kubernetes/autoscaler/releases) for the latest patch.

```bash
# Check current CA version
kubectl -n kube-system get deployment cluster-autoscaler -o jsonpath='{.spec.template.spec.containers[0].image}'
```

## Testing Scale-Up and Scale-Down

### Step-by-Step Validation

Open three terminals to observe the full cycle:

**Terminal 1 — Watch CA logs:**

```bash
kubectl -n kube-system logs -f deployment/cluster-autoscaler
```

**Terminal 2 — Create a resource-hungry deployment:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-scale
spec:
  replicas: 30
  selector:
    matchLabels:
      app: test-scale
  template:
    metadata:
      labels:
        app: test-scale
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
EOF
```

**Terminal 3 — Watch pods and nodes:**

```bash
watch -n 2 'kubectl get nodes && echo "---" && kubectl get pods -l app=test-scale | head -20'
```

### What Should Happen (Scale-Up)

1. CA detects pending pods within 10 seconds
2. Evaluates ASGs for fit
3. Increments ASG desired capacity
4. New nodes appear in 2–5 minutes
5. Pending pods schedule on new nodes

### Test Scale-Down

```bash
kubectl scale deployment test-scale --replicas=1
```

Watch logs and nodes. Scale-down takes 10–20 minutes (waits for `--scale-down-unneeded-time`).

### Clean Up

```bash
kubectl delete deployment test-scale
```

## Log Patterns — What to Look For

### Healthy Operation

```
No unschedulable pods
Calculating unneeded nodes
Scale down: no unneeded nodes
```

### Scale-Up Triggered

```
Expanding Node Group ... by 1
Best option to resize: ...
Scale-up: setting group ... size to ...
```

### Scale-Down Triggered

```
Scale-down: removing empty node ...
Scale-down: removing node ..., utilization ...
```

### IAM Permission Error

```
Failed to create AWS Manager: cannot autodiscover ASGs: AccessDenied: User: abc is not authorized to perform: autoscaling:DescribeTags
```

### Network/Credential Error

```
WebIdentityErr: failed to retrieve credentials caused by: RequestError: send request failed caused by: I/O timeout
```

## Stray Node: "No Node Group Config"

If you see this in CA logs:

```
Node ip-10-112-48-80.us-west-2.compute.internal should not be processed by cluster autoscaler (no node group config)
```

This means a node exists in the cluster but CA can't find which ASG it belongs to.

### Causes

- Node was created manually (outside any ASG)
- Node's ASG was deleted but the node still exists
- ASG is missing the auto-discovery tags
- Node was launched by a different tool (e.g., Karpenter, Terraform without proper tags)

### Investigation

```bash
# Check if the node belongs to an ASG
aws ec2 describe-instances \
  --filters "Name=private-dns-name,Values=ip-10-112-48-80.us-west-2.compute.internal" \
  --query 'Reservations[].Instances[].[InstanceId,Tags[?Key==`aws:autoscaling:groupName`].Value]' \
  --output table
```

### Fixes

```bash
# Option 1: Exclude from CA (keep the node, stop the warning)
kubectl annotate node ip-10-112-48-80.us-west-2.compute.internal \
  cluster-autoscaler.kubernetes.io/scale-down-disabled=true

# Option 2: Remove the stray node
kubectl drain ip-10-112-48-80.us-west-2.compute.internal --ignore-daemonsets --delete-emptydir-data
kubectl delete node ip-10-112-48-80.us-west-2.compute.internal

# Option 3: Add proper tags to its ASG (if it has one)
aws autoscaling create-or-update-tags --tags \
  "ResourceId=<asg-name>,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/enabled,Value=true,PropagateAtLaunch=true"
```

## Validation Checklist

- [ ] CA pod is running: `kubectl get pods -n kube-system -l app=cluster-autoscaler`
- [ ] Logs show no errors: `kubectl -n kube-system logs deployment/cluster-autoscaler --tail=20`
- [ ] ASG tags are present: `k8s.io/cluster-autoscaler/enabled` + `k8s.io/cluster-autoscaler/<cluster>`
- [ ] Service account has IAM role annotation: `kubectl -n kube-system describe sa cluster-autoscaler`
- [ ] ASGs are discovered in logs: `grep -i "asg" <logs>`
- [ ] CA version matches K8s version
- [ ] Test deployment triggers scale-up
- [ ] Scale-down works after reducing replicas

## Useful Aliases

```bash
# View CA logs (follow)
alias ca-logs='kubectl -n kube-system logs -f deployment/cluster-autoscaler'

# Check CA pod status
alias ca-status='kubectl get pods -n kube-system -l app=cluster-autoscaler'

# Pending pods
alias kpending='kubectl get pods -A --field-selector=status.phase=Pending'

# Watch nodes
alias knodes='watch kubectl get nodes -o wide'
```
