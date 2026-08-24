# Karpenter Guide: Intelligent Node Scaling for EKS

Karpenter is an open-source node provisioner for Kubernetes that launches the right-sized compute in seconds based on pending pod requirements. It replaces the Cluster Autoscaler with faster, more cost-efficient scaling.

## Why Karpenter Over Cluster Autoscaler

### Cost Savings

Teams typically report **20-40% compute cost reduction** after switching from CA to Karpenter. The savings come from:

| Mechanism | Cluster Autoscaler | Karpenter |
|-----------|-------------------|-----------|
| Instance selection | Fixed per node group | Picks cheapest fit from all allowed types |
| Bin-packing | Launches full-size nodes regardless of pod needs | Launches the smallest instance that fits |
| Consolidation | Only removes empty nodes | Replaces underutilized nodes with smaller ones |
| Spot optimization | One type per node group (or limited MIG) | Many types per NodePool, automatic fallback |
| Scale-down speed | 10-min cooldown, conservative | Minutes, aggressive (configurable) |

### Example: Why CA Over-Provisions

```
Pod requesting: 0.5 CPU, 1 GB RAM

Cluster Autoscaler (ASG with m5.xlarge):
  → Launches m5.xlarge (4 CPU, 16 GB RAM)
  → 87% of capacity wasted
  → Cost: $0.192/hour

Karpenter (allowed: t3.small through m5.2xlarge):
  → Launches t3.small (2 CPU, 2 GB RAM)
  → Fits the pod with room for 1-2 more
  → Cost: $0.0208/hour
  → 89% cheaper for this single pod
```

At scale with hundreds of pods, this right-sizing adds up significantly.

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Karpenter Controller                      │
│                  (Deployment in kube-system)                 │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │  Provisioner │  │ Consolidator │  │  Disruption Engine │  │
│  │  (watches    │  │ (optimizes   │  │  (drift, expiry,   │  │
│  │   pending    │  │  node usage) │  │   consolidation)   │  │
│  │   pods)      │  │              │  │                    │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬───────────┘  │
│         │                 │                   │              │
└─────────┼─────────────────┼───────────────────┼──────────────┘
          │                 │                   │
          ▼                 ▼                   ▼
    EC2 RunInstances    EC2 TerminateInstances   Cordon + Drain
    (direct API call)   (replace with smaller)   (before terminate)
```

Karpenter calls the EC2 API directly — it does NOT use Auto Scaling Groups.

## Installation

### Prerequisites

- EKS cluster with OIDC provider (for IRSA) or Pod Identity
- IAM role for Karpenter with EC2, SSM, and IAM permissions
- Subnets and security groups tagged for discovery

### Install with Helm

```sh
# Set variables
CLUSTER_NAME="my-cluster"
KARPENTER_VERSION="1.0.0"
CLUSTER_ENDPOINT=$(aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.endpoint" --output text)

# Install
helm install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version $KARPENTER_VERSION \
  --namespace kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set clusterEndpoint=$CLUSTER_ENDPOINT \
  --set settings.interruptionQueue=karpenter-$CLUSTER_NAME

# Verify
kubectl get pods -n kube-system -l app.kubernetes.io/name=karpenter
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter -f
```

### Tag Subnets and Security Groups

Karpenter discovers subnets and SGs via tags:

```sh
# Tag subnets
aws ec2 create-tags --resources subnet-xxx subnet-yyy subnet-zzz \
  --tags Key=karpenter.sh/discovery,Value=$CLUSTER_NAME

# Tag security groups
aws ec2 create-tags --resources sg-xxx \
  --tags Key=karpenter.sh/discovery,Value=$CLUSTER_NAME
```

## Configuration

### NodePool (What to Provision)

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      labels:
        team: platform
    spec:
      requirements:
        # Instance types
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.large", "m5.xlarge", "m5.2xlarge", "c5.large", "c5.xlarge", "r5.large"]

        # Capacity type (spot + on-demand fallback)
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]

        # Architecture
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]

        # Availability zones
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["us-east-1a", "us-east-1b", "us-east-1c"]

      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default

  # Resource limits (total across all nodes from this pool)
  limits:
    cpu: "1000"
    memory: 1000Gi

  # Disruption policy
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m

  # Node expiry (force rotation after 7 days)
  disruption:
    expireAfter: 168h
```

### EC2NodeClass (How to Configure Nodes)

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  # AMI selection
  amiSelectorTerms:
    - alias: al2023@latest   # Amazon Linux 2023, latest version

  # Subnet discovery
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster

  # Security group discovery
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster

  # IAM role for nodes
  role: KarpenterNodeRole-my-cluster

  # Instance profile (alternative to role)
  # instanceProfile: KarpenterNodeInstanceProfile-my-cluster

  # Block device mappings
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        iops: 3000
        throughput: 125
        deleteOnTermination: true
        encrypted: true

  # User data (bootstrap customization)
  userData: |
    #!/bin/bash
    echo "Custom setup script"

  # Tags applied to EC2 instances
  tags:
    Environment: production
    ManagedBy: karpenter
```

## Disruption and Consolidation

### How Consolidation Works

Karpenter continuously evaluates whether nodes can be consolidated:

1. **Empty node** — No pods running (after grace period) → terminate
2. **Underutilized node** — Pods can fit on other existing nodes → drain and terminate
3. **Replace with cheaper** — Two small nodes can be replaced by one larger (or vice versa) → replace

```
Before consolidation:
  Node A (m5.xlarge, 4 CPU): using 1 CPU
  Node B (m5.xlarge, 4 CPU): using 1 CPU

After consolidation:
  Node C (m5.large, 2 CPU): using 2 CPU
  Savings: 50% (one fewer node)
```

### Consolidation Policies

```yaml
disruption:
  # Consolidate empty and underutilized nodes
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 1m

  # Or: only consolidate empty nodes (less aggressive)
  # consolidationPolicy: WhenEmpty
  # consolidateAfter: 30s
```

### Drift Detection

Karpenter detects when nodes no longer match the desired state:

- AMI changed in EC2NodeClass → nodes are "drifted"
- Security group changed → drifted
- Subnet removed → drifted
- NodePool requirements changed → drifted

Drifted nodes are cordoned, drained, and replaced automatically.

### Node Expiry

Force node rotation (useful for security patching):

```yaml
disruption:
  expireAfter: 168h   # Nodes live max 7 days, then get replaced
```

### Budgets (Control Disruption Rate)

Limit how many nodes Karpenter can disrupt simultaneously:

```yaml
disruption:
  budgets:
    - nodes: "10%"      # Max 10% of nodes disrupted at once
    - nodes: "3"        # Or at most 3 nodes at a time
    - nodes: "0"        # Block all disruptions (maintenance window)
      schedule: "0 9 * * 1-5"   # During business hours
      duration: 8h
```

## Spot Instance Management

### Spot Configuration

```yaml
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
```

Karpenter handles Spot automatically:
- Diversifies across many instance types to reduce interruption risk
- Falls back to on-demand if no Spot capacity is available
- Handles Spot interruption notices (2-min warning) → cordons and drains

### Spot Best Practices

```yaml
# Allow many instance types for better Spot availability
requirements:
  - key: node.kubernetes.io/instance-type
    operator: In
    values:
      - m5.large
      - m5.xlarge
      - m5a.large
      - m5a.xlarge
      - m6i.large
      - m6i.xlarge
      - c5.large
      - c5.xlarge
      - c5a.large
      - r5.large
```

The more instance types you allow, the better Spot availability and pricing.

### Separate Pools for Spot and On-Demand

```yaml
# Critical workloads: on-demand only
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: critical
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
      taints:
        - key: workload-type
          value: critical
          effect: NoSchedule
---
# Batch/tolerant workloads: spot preferred
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: batch
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
```

## GPU and Accelerator Support

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu
spec:
  template:
    spec:
      requirements:
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["g5.xlarge", "g5.2xlarge", "p4d.24xlarge"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
      taints:
        - key: nvidia.com/gpu
          value: "true"
          effect: NoSchedule
  limits:
    nvidia.com/gpu: "8"
```

Karpenter provisions GPU instances only when pods request `nvidia.com/gpu` resources.

## Monitoring

### Key Metrics

Karpenter exposes Prometheus metrics:

```sh
# Scrape from the controller pod
kubectl port-forward -n kube-system deploy/karpenter 8080:8080
curl http://localhost:8080/metrics
```

| Metric | Description |
|--------|-------------|
| `karpenter_nodes_total` | Total nodes managed by Karpenter |
| `karpenter_pods_state` | Pod states (pending, running, scheduled) |
| `karpenter_provisioner_scheduling_duration_seconds` | Time to make scheduling decisions |
| `karpenter_nodes_allocatable` | Allocatable resources per node |
| `karpenter_interruption_actions_performed` | Spot interruption actions taken |
| `karpenter_disruption_actions_performed` | Consolidation/drift actions taken |
| `karpenter_cloudprovider_duration_seconds` | EC2 API call latency |

### Useful kubectl Commands

```sh
# See what Karpenter has provisioned
kubectl get nodeclaims

# See NodePool status
kubectl get nodepools

# Check pending pods (what Karpenter is working on)
kubectl get pods --field-selector status.phase=Pending -A

# Karpenter logs (scheduling decisions)
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter --tail=100

# Watch Karpenter in action
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter -f | grep -E "launched|terminated|consolidated"
```

## Migration from Cluster Autoscaler

### Step-by-Step

```sh
# 1. Install Karpenter alongside CA (they can coexist briefly)
helm install karpenter oci://public.ecr.aws/karpenter/karpenter ...

# 2. Create NodePools matching your current node groups
kubectl apply -f nodepool.yaml

# 3. Taint existing CA-managed nodes (so new pods go to Karpenter nodes)
kubectl taint nodes -l eks.amazonaws.com/nodegroup=old-ng karpenter-migration=true:PreferNoSchedule

# 4. Scale down CA node groups gradually
aws eks update-nodegroup-config --cluster-name <cluster> --nodegroup-name <ng> \
  --scaling-config minSize=0,desiredSize=0,maxSize=0

# 5. Remove Cluster Autoscaler
helm uninstall cluster-autoscaler -n kube-system

# 6. Delete empty node groups
aws eks delete-nodegroup --cluster-name <cluster> --nodegroup-name <ng>
```

### Keep One Managed Node Group

Best practice: keep a small managed node group for system components (Karpenter itself, CoreDNS, kube-proxy):

```sh
# Small node group for system pods
aws eks create-nodegroup --cluster-name <cluster> \
  --nodegroup-name system \
  --node-role-arn <role-arn> \
  --subnets subnet-aaa subnet-bbb \
  --instance-types t3.medium \
  --scaling-config minSize=2,desiredSize=2,maxSize=3 \
  --labels node-role=system \
  --taints "key=CriticalAddonsOnly,value=true,effect=NO_SCHEDULE"
```

This ensures Karpenter always has somewhere to run, even if all Karpenter-managed nodes are terminated.

## Cost Optimization Tips

### 1. Allow Many Instance Types

More instance types = better spot pricing + better right-sizing:

```yaml
# Bad: only one type
values: ["m5.xlarge"]

# Good: many types across families
values: ["m5.large", "m5.xlarge", "m5a.large", "m5a.xlarge", "m6i.large", "m6i.xlarge", "c5.large", "c5.xlarge", "c5a.large", "r5.large"]
```

### 2. Enable Consolidation

```yaml
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 1m
```

### 3. Use Node Expiry for Savings Plans

If you have EC2 Savings Plans, expired nodes get replaced with whatever is cheapest under your plan:

```yaml
disruption:
  expireAfter: 168h  # Weekly rotation catches new pricing
```

### 4. Set Resource Requests Accurately

Karpenter provisions based on pod requests. If requests are inflated, you get bigger (more expensive) nodes:

```yaml
# Bad: requesting 4 CPU when you use 0.5
resources:
  requests:
    cpu: "4"

# Good: request what you actually use
resources:
  requests:
    cpu: "500m"
```

### 5. Use Topology Spread for Even AZ Distribution

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
```

This prevents all pods landing in one AZ (which forces Karpenter to over-provision that AZ).

## Gotchas

- **PDBs block consolidation**: If a PDB prevents draining, Karpenter can't consolidate. This is by design (safety) but can prevent cost savings.
- **Node churn**: Aggressive consolidation means pods move frequently. Ensure proper graceful shutdown and readiness probes.
- **Startup time still matters**: Karpenter is fast (1 min), but if your app takes 5 min to start, that's still 5 min of unavailability for new pods.
- **Karpenter needs to run somewhere**: Don't let Karpenter schedule itself on Karpenter-managed nodes. Use a small managed node group or Fargate for system pods.
- **Limits prevent runaway spend**: Always set `limits` in your NodePool. Without them, a bad HPA config could launch unlimited nodes.
- **Spot interruptions + consolidation**: If Karpenter replaces a Spot node due to consolidation, and then the replacement also gets interrupted, pods can bounce. Set `consolidateAfter` higher for Spot-heavy pools.
- **Windows nodes**: Karpenter supports Windows, but ensure your EC2NodeClass specifies a Windows AMI.
- **Drift can be aggressive**: If you change the EC2NodeClass (e.g., new AMI), ALL nodes are marked as drifted and will be replaced. Use `budgets` to control the rollout rate.

## Node Pools

Node pools are groups of nodes with similar configurations in a Kubernetes cluster. They organize and manage different types of compute resources.

### Traditional Node Pool Components

- **Instance type** — CPU, memory, and storage specifications
- **Operating system** — Linux distributions, Windows, etc.
- **Kubernetes version** — control plane compatibility
- **Networking** — VPC, subnets, security groups
- **Scaling policies** — min/max node counts, scaling triggers

### EKS Node Group Example (eksctl)

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-west-2

nodeGroups:
  - name: general-purpose
    instanceType: t3.medium
    minSize: 1
    maxSize: 10
    desiredCapacity: 3
    volumeSize: 20
    labels:
      role: general-purpose
    tags:
      nodegroup-role: general-purpose

  - name: compute-optimized
    instanceType: c5.large
    minSize: 0
    maxSize: 5
    desiredCapacity: 0
    volumeSize: 20
    labels:
      role: compute-optimized
    taints:
      - key: compute-optimized
        value: "true"
        effect: NoSchedule

  - name: spot-instances
    instanceType: m5.large
    minSize: 0
    maxSize: 10
    desiredCapacity: 2
    spot: true
    labels:
      lifecycle: spot
    tags:
      nodegroup-type: spot
```

### Scheduling Workloads to Specific Pools

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spot-workload
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spot-workload
  template:
    metadata:
      labels:
        app: spot-workload
    spec:
      nodeSelector:
        karpenter.sh/capacity-type: spot
      tolerations:
        - key: spot
          operator: Equal
          value: "true"
          effect: NoSchedule
      containers:
        - name: app
          image: nginx
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
```

## Karpenter vs Traditional Node Pools

### Traditional Approach

- Pre-defined node pools with fixed instance types
- Slower scaling (Cluster Autoscaler + ASG)
- Manual configuration for different workload types
- Over-provisioning to handle demand spikes
- Each instance type needs its own node group

### Karpenter Approach

- Dynamic node provisioning based on actual pod requirements
- No pre-defined node pools — creates nodes on-demand
- Automatically selects from hundreds of instance types
- More efficient resource utilization
- Single NodePool can serve many workload profiles

### Side-by-Side Comparison

| Aspect | Traditional Node Pools | Karpenter NodePools |
|--------|----------------------|---------------------|
| Instance selection | Fixed per node group | Dynamic per scheduling decision |
| Scaling speed | Minutes (ASG + CA) | Seconds (direct EC2 API) |
| Node groups needed | One per instance type/config | One or few NodePools cover everything |
| Spot handling | Separate ASG or MIG | Built-in, multi-type diversification |
| Right-sizing | Manual, per node group | Automatic, per pod batch |
| Consolidation | Remove empty nodes only | Replace underutilized with smaller |
| AMI updates | Rolling update per ASG | Drift detection + automatic replacement |
| Operational overhead | High (many ASGs to manage) | Low (declarative NodePool + EC2NodeClass) |

### When to Use Traditional Node Groups

- System components (Karpenter itself, CoreDNS)
- Strict compliance requiring fixed infrastructure
- Windows nodes with specific AMI requirements
- GPU nodes with limited instance type options
- Fargate profiles for serverless workloads

### When to Use Karpenter

- General application workloads
- Bursty or unpredictable traffic patterns
- Cost optimization is a priority
- Many different workload profiles on one cluster
- Spot instance diversification

## Quick Reference

```sh
# Install
helm install karpenter oci://public.ecr.aws/karpenter/karpenter --namespace kube-system ...

# Check NodePools
kubectl get nodepools
kubectl describe nodepool default

# Check provisioned nodes
kubectl get nodeclaims
kubectl get nodes -l karpenter.sh/nodepool=default

# Watch Karpenter decisions
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter -f

# Force a node to be replaced
kubectl delete node <node-name>  # Karpenter replaces it automatically

# Pause disruptions (emergency)
kubectl annotate nodepool default karpenter.sh/do-not-disrupt="true"

# Resume disruptions
kubectl annotate nodepool default karpenter.sh/do-not-disrupt-
```
