# Cluster Autoscaler vs Karpenter for EKS

A comparison of the two node auto-scaling solutions for Amazon EKS — architecture, behavior, configuration, and when to use each.

## Overview

| | Cluster Autoscaler (CA) | Karpenter |
|---|---|---|
| **Maintained by** | Kubernetes SIG Autoscaling | AWS (open source) |
| **Scaling unit** | Auto Scaling Group (ASG) | Individual EC2 instances |
| **Instance selection** | You pre-define instance types per ASG | Karpenter chooses dynamically from constraints |
| **Scale-up speed** | 30–60 seconds (ASG launch) | 15–30 seconds (direct EC2 API) |
| **Scale-down** | Based on utilization threshold + timers | Consolidation — actively moves pods to fewer nodes |
| **Configuration** | ASG tags + CA flags | NodePool + EC2NodeClass CRDs |
| **Multi-arch/GPU** | Separate ASGs per type | Single NodePool with flexible requirements |
| **EKS Auto Mode** | Not used | Built-in (Karpenter-based) |

## Architecture Differences

### Cluster Autoscaler

```
Pending pods → CA evaluates ASGs → picks ASG → calls ASG API → ASG launches EC2
                                                                      ↓
                                                              Node joins cluster
                                                                      ↓
                                                              Pods scheduled
```

- CA doesn't launch instances directly — it adjusts the ASG desired count
- ASG handles instance launch, AZ placement, and lifecycle
- You pre-configure ASGs with specific instance types and sizes
- CA picks which ASG to scale based on the expander strategy

### Karpenter

```
Pending pods → Karpenter evaluates pod requirements → calls EC2 API directly → instance launches
                                                                                      ↓
                                                                              Node joins cluster
                                                                                      ↓
                                                                              Pods scheduled
```

- Karpenter calls the EC2 Fleet API directly (no ASG intermediary)
- Chooses instance type dynamically based on pending pod requirements
- Can mix instance types, architectures, and purchase options per launch
- No ASGs needed

## Scale-Up Behavior

### Cluster Autoscaler

1. CA scans for unschedulable pods every `--scan-interval` (default 10s)
2. Simulates scheduling against each ASG's node template
3. Uses the expander to pick which ASG to scale (random, least-waste, most-pods, priority)
4. Increments ASG desired capacity
5. ASG launches instance (20–40s)
6. Node registers with cluster (10–20s)
7. Pod schedules on new node

**Total time: 30–90 seconds**

### Karpenter

1. Karpenter watches for unschedulable pods (real-time via informers)
2. Groups pending pods by constraints (node selectors, affinity, topology)
3. Selects optimal instance type(s) from the NodePool's allowed set
4. Calls EC2 CreateFleet API directly
5. Instance launches (10–20s)
6. Node registers with cluster (10–15s)
7. Pod schedules on new node

**Total time: 15–45 seconds**

Karpenter is faster because it skips the ASG layer and makes a single API call for the exact instance it needs.

## Scale-Down Behavior

### Cluster Autoscaler

- Checks node utilization every `--scan-interval`
- Node is candidate for removal if utilization < `--scale-down-utilization-threshold` (default 0.5)
- Must remain underutilized for `--scale-down-unneeded-time` (default 10m) before removal
- Respects PDBs during drain
- Decrements ASG desired capacity

**Behavior:** Passive — waits for nodes to become underutilized, then removes them after a delay.

### Karpenter

- **Consolidation** — actively looks for opportunities to reduce costs:
  - **Delete:** Remove empty nodes immediately
  - **Replace:** Replace a node with a cheaper instance if pods would fit
  - **Merge:** Move pods from multiple underutilized nodes onto fewer nodes
- Runs continuously, not on a timer
- Respects PDBs and NodePool disruption budgets
- Can be configured per NodePool

**Behavior:** Active — continuously optimizes node fleet for cost and efficiency.

```yaml
# Karpenter consolidation policy
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
    budgets:
    - nodes: "10%"
```

## Configuration Comparison

### Cluster Autoscaler Setup

```bash
# 1. Create ASGs with specific instance types
# (via eksctl, Terraform, or console)

# 2. Tag ASGs for auto-discovery
# k8s.io/cluster-autoscaler/enabled = true
# k8s.io/cluster-autoscaler/<cluster-name> = owned

# 3. Deploy CA with auto-discovery
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=us-west-2 \
  --set extraArgs.scale-down-utilization-threshold=0.5 \
  --set extraArgs.scale-down-unneeded-time=10m \
  --set extraArgs.expander=least-waste \
  -n kube-system
```

You need separate ASGs for:
- Different instance families (m5 vs c5 vs r5)
- Different AZs (if you want control)
- GPU vs non-GPU
- Spot vs On-Demand
- ARM vs x86

### Karpenter Setup

```bash
# 1. Install Karpenter
helm install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --namespace kube-system \
  --set clusterName=my-cluster \
  --set clusterEndpoint=$(aws eks describe-cluster --name my-cluster --query "cluster.endpoint" --output text)
```

```yaml
# 2. Create a NodePool (replaces ASG configuration)
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64", "arm64"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["on-demand", "spot"]
      - key: node.kubernetes.io/instance-type
        operator: In
        values: ["m5.large", "m5.xlarge", "m5.2xlarge", "c5.large", "c5.xlarge", "r5.large"]
  limits:
    cpu: "100"
    memory: 400Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
    budgets:
    - nodes: "10%"
---
# 3. Create EC2NodeClass (networking, AMI, security)
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiSelectorTerms:
  - alias: al2023@latest
  subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: my-cluster
  role: KarpenterNodeRole-my-cluster
```

One NodePool handles what would require 5+ ASGs with Cluster Autoscaler.

## Instance Type Selection

### Cluster Autoscaler

- You define fixed instance types per ASG at creation time
- CA can only launch what's in the ASG launch template
- To add new instance types, you modify the ASG or create a new one
- No runtime flexibility

### Karpenter

- You define constraints (families, sizes, architectures) in the NodePool
- Karpenter picks the best-fit instance at launch time based on:
  - Pending pod resource requests
  - Spot availability and pricing
  - AZ balance
  - Instance type availability
- Can use `karpenter.k8s.aws/instance-family`, `karpenter.k8s.aws/instance-size`, etc.

```yaml
# Karpenter: flexible instance selection
requirements:
- key: karpenter.k8s.aws/instance-family
  operator: In
  values: ["m5", "m6i", "m7i", "c5", "c6i", "r5", "r6i"]
- key: karpenter.k8s.aws/instance-size
  operator: NotIn
  values: ["nano", "micro", "small"]    # Exclude tiny instances
- key: karpenter.sh/capacity-type
  operator: In
  values: ["spot", "on-demand"]
- key: topology.kubernetes.io/zone
  operator: In
  values: ["us-west-2a", "us-west-2b", "us-west-2c"]
```

## Spot Instance Handling

| Aspect | Cluster Autoscaler | Karpenter |
|--------|-------------------|-----------|
| Configuration | Separate Spot ASG with mixed instances policy | `capacity-type: spot` in NodePool requirements |
| Interruption handling | Requires AWS Node Termination Handler (NTH) | Built-in interruption handling |
| Fallback to On-Demand | Separate ASG needed | Automatic — includes both in same NodePool |
| Instance diversification | ASG mixed instances policy | Karpenter diversifies automatically across 15+ types |

## Cost Optimization

### Cluster Autoscaler

- Scale-down is reactive (waits for underutilization)
- No instance replacement for cost savings
- You must manually choose instance types
- Spot savings require separate ASGs

### Karpenter

- Active consolidation — replaces expensive nodes with cheaper ones
- Detects when a smaller instance would fit the current workload
- Automatically uses Spot where allowed
- Right-sizes nodes to actual pod requests (no oversized nodes sitting idle)

Example: If a `m5.4xlarge` is running only 2 small pods, Karpenter will:
1. Launch a `m5.large` (cheaper)
2. Move the pods to the new node (respecting PDBs)
3. Terminate the `m5.4xlarge`

## Real-World Scenarios

### Batch Job Processing

With CA, batch jobs requesting 2 CPU / 4Gi memory get scheduled on m5.large nodes (2 CPU, 8GB) — 50% memory waste. Scaling 50 parallel pods takes 3-5 minutes as ASGs incrementally add nodes.

With Karpenter, the same job gets c5.large instances (2 CPU, 4GB) — perfect fit. Nodes launch in 30 seconds via a single CreateFleet API call for the entire batch.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processing
spec:
  parallelism: 50
  template:
    spec:
      containers:
      - name: processor
        image: my-batch-job
        resources:
          requests:
            cpu: 2
            memory: 4Gi
      # Karpenter provisions c5.large (2 CPU, 4GB) — exact fit
      # CA provisions m5.large (2 CPU, 8GB) — 4GB wasted per pod
```

### Mixed Workloads (Small + Large on Same Cluster)

Karpenter handles diverse pod sizes in one NodePool — it packs small pods on shared nodes and launches large instances only for big pods:

```yaml
# Small web pods → Karpenter bins multiple onto one m5.xlarge
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 20
  template:
    spec:
      containers:
      - name: web
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
---
# Large ML pod → Karpenter launches a dedicated c5.4xlarge
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-training
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: trainer
        resources:
          requests:
            cpu: 8
            memory: 32Gi
```

With CA, you'd need separate ASGs (small-instance ASG + large-instance ASG) and nodeSelectors to route pods.

## Workload-Specific NodePools

### GPU Workloads

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu-pool
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: gpu
      requirements:
      - key: node.kubernetes.io/instance-type
        operator: In
        values: ["p3.2xlarge", "p3.8xlarge", "g4dn.xlarge", "g4dn.2xlarge"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["on-demand"]    # GPU rarely available as Spot
      taints:
      - key: nvidia.com/gpu
        value: "true"
        effect: NoSchedule
  limits:
    cpu: "64"
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 60s
```

Pods must tolerate the taint to schedule on GPU nodes:

```yaml
spec:
  tolerations:
  - key: nvidia.com/gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  containers:
  - name: training
    resources:
      limits:
        nvidia.com/gpu: 1
```

### ARM64 Cost-Optimized Pool

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: arm64-spot
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: ["arm64"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot"]
      - key: karpenter.k8s.aws/instance-family
        operator: In
        values: ["m6g", "m7g", "c6g", "c7g", "r6g"]
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
```

ARM64 Graviton instances are ~20% cheaper than x86 equivalents. Combine with Spot for up to 70% savings.

## Migration: CA to Karpenter

### Step 1: Install Karpenter alongside CA

Both can run simultaneously. Karpenter will handle pods that CA can't schedule (and vice versa).

### Step 2: Create NodePools matching your ASG configuration

Map your ASG instance types, subnets, and security groups to NodePool/EC2NodeClass.

### Step 3: Taint CA-managed nodes

```bash
# Prevent new pods from scheduling on CA nodes
kubectl taint nodes -l eks.amazonaws.com/nodegroup=old-ng karpenter-migration=true:NoSchedule
```

### Step 4: Drain CA-managed nodes

```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

Pods will reschedule → Karpenter launches new nodes for them.

### Step 5: Remove CA and ASGs

```bash
helm uninstall cluster-autoscaler -n kube-system
# Delete ASGs via eksctl/Terraform/console
```

## When to Use Each

### Use Cluster Autoscaler when:

- You have an existing stable setup with ASGs that works fine
- You need tight control over exactly which instance types launch
- Your organization requires ASGs for compliance/auditing
- You're using EKS managed node groups and want AWS-managed AMI updates
- Simple scaling needs — predictable workloads, few instance types

### Use Karpenter when:

- You need fast scale-up (sub-30-second node provisioning)
- You want automatic cost optimization (consolidation, right-sizing)
- You run diverse workloads (GPU, ARM, Spot, different sizes)
- You want one configuration to handle multiple instance families
- You need better Spot instance handling (automatic diversification)
- You're starting a new cluster or EKS Auto Mode

## Limitations

### Cluster Autoscaler

- Slower scale-up (ASG adds latency)
- No active cost optimization (only removes idle nodes)
- ASG sprawl — many ASGs needed for diverse workloads
- No instance replacement for cheaper alternatives
- Requires separate Node Termination Handler for Spot

### Karpenter

- No managed node group integration (uses unmanaged instances)
- AMI updates are your responsibility (via EC2NodeClass)
- Consolidation can cause pod disruption (mitigate with PDBs)
- Less mature ecosystem (fewer community guides, newer)
- AWS-specific (not multi-cloud like CA)
- No direct visibility in EKS console node groups view

## Quick Reference

```bash
# --- Cluster Autoscaler ---

# Check CA logs
kubectl -n kube-system logs -l app=cluster-autoscaler --tail=100

# Check CA status
kubectl -n kube-system get configmap cluster-autoscaler-status -o yaml

# Check which ASGs CA manages
kubectl -n kube-system logs -l app=cluster-autoscaler | grep "node group"

# --- Karpenter ---

# Check Karpenter logs
kubectl -n kube-system logs -l app.kubernetes.io/name=karpenter --tail=100

# List NodePools
kubectl get nodepools

# List EC2NodeClasses
kubectl get ec2nodeclasses

# Check provisioned nodes by Karpenter
kubectl get nodes -l karpenter.sh/nodepool

# Check Karpenter's view of node capacity
kubectl get nodeclaims

# Force consolidation check
kubectl annotate nodeclaim <name> karpenter.sh/voluntary-disruption=underutilized
```
