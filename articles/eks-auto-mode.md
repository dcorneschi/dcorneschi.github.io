# EKS Auto Mode

EKS Auto Mode extends AWS management to the data plane — AWS handles compute, networking, storage, load balancing, and upgrades so you can focus on deploying workloads. Announced December 2024, it's the most managed EKS experience available.

## What AWS Manages in Auto Mode

| Component | What Auto Mode Handles |
|-----------|----------------------|
| **Compute** | Node provisioning, scaling (Karpenter-based), instance selection, Spot interruption |
| **Networking** | Pod IP assignment, network policies, VPC CNI, IPv4/IPv6, secondary CIDRs |
| **Storage** | Ephemeral storage config, EBS CSI, volume encryption |
| **Load Balancing** | ALB/NLB provisioning for Services and Ingress |
| **DNS** | CoreDNS runs as a system service on each node (not a Deployment) |
| **GPU** | NVIDIA and Neuron drivers pre-installed, plugins managed |
| **Upgrades** | OS patching, component upgrades, automatic node rotation (21-day max lifetime) |
| **Security** | SELinux enforcing, read-only root filesystem, immutable AMIs, no SSH/SSM access |
| **Node health** | Automatic repair on EC2 status check failures, scheduled maintenance |
| **Identity** | Pod Identity Agent pre-installed |

## Auto Mode vs Standard Mode

| Feature | Standard Mode | Auto Mode |
|---------|--------------|-----------|
| Node management | You (node groups, ASGs, AMIs) | AWS |
| Scaling | You deploy Cluster Autoscaler or Karpenter | Built-in (Karpenter-based) |
| AMI updates | You trigger rolling updates | Automatic (21-day max node lifetime) |
| OS patching | You manage | AWS manages |
| SSH/SSM access to nodes | Yes | No (locked down) |
| DaemonSets | Yes | Yes |
| Privileged containers | Yes | Restricted |
| Custom AMI | Yes | No (Bottlerocket-based, immutable) |
| SELinux | Optional | Enforced |
| Root filesystem | Read-write | Read-only |
| Network policies | Add-on required (Calico/Cilium) | Built-in |
| Load balancer controller | You deploy | Built-in |
| EBS CSI driver | You deploy | Built-in |
| Pod Identity Agent | You deploy | Built-in |
| CoreDNS | Deployment (you manage replicas) | System service per node |
| Node max lifetime | Unlimited (you control) | 21 days (configurable lower) |
| Pricing | EC2 per-instance + $0.10/hr control plane | EKS Auto Mode compute pricing + $0.10/hr control plane |

## Creating an Auto Mode Cluster

### AWS CLI

```sh
aws eks create-cluster --name my-auto-cluster \
  --region us-east-1 \
  --kubernetes-version 1.30 \
  --role-arn arn:aws:iam::123456789012:role/EKSAutoModeClusterRole \
  --resources-vpc-config subnetIds=subnet-aaa,subnet-bbb,subnet-ccc,securityGroupIds=sg-xxx \
  --compute-config enabled=true,nodePools=["general-purpose","system"] \
  --kubernetes-network-config elasticLoadBalancing=enabled \
  --storage-config blockStorage=enabled
```

### eksctl

```sh
eksctl create cluster --name my-auto-cluster --region us-east-1 \
  --version 1.30 --auto-mode
```

Or with a config file:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-auto-cluster
  region: us-east-1
  version: "1.30"

autoModeConfig:
  enabled: true
```

### AWS Console

1. Open EKS Console → **Add cluster** → **Create**
2. Under **Configuration options**, select **Custom configuration**
3. Toggle **Use EKS Auto Mode** to ON
4. Configure VPC, subnets, and cluster IAM role
5. Review and create

### Terraform

```hcl
resource "aws_eks_cluster" "auto" {
  name     = "my-auto-cluster"
  role_arn = aws_iam_role.cluster.arn
  version  = "1.30"

  vpc_config {
    subnet_ids = var.subnet_ids
  }

  compute_config {
    enabled    = true
    node_pools = ["general-purpose", "system"]
  }

  kubernetes_network_config {
    elastic_load_balancing {
      enabled = true
    }
  }

  storage_config {
    block_storage {
      enabled = true
    }
  }
}
```

## Enable Auto Mode on Existing Cluster

```sh
aws eks update-cluster-config --name my-cluster --region us-east-1 \
  --compute-config enabled=true,nodePools=["general-purpose","system"]
```

> Existing managed node groups continue to work alongside Auto Mode nodes. You can migrate workloads gradually.

## Default Node Pools

Auto Mode creates two default NodePools:

| Pool | Purpose | Instance Types | Capacity |
|------|---------|---------------|----------|
| `system` | System workloads (kube-proxy, CoreDNS) | Optimized for small, reliable | On-demand |
| `general-purpose` | Application workloads | Broad selection (auto-optimized) | On-demand + Spot |

You cannot edit default NodePools, but you can create custom ones alongside them.

## Custom NodePools and NodeClasses

For specific requirements, create custom NodePools:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu-workloads
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
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: gpu
  limits:
    nvidia.com/gpu: "16"
---
apiVersion: eks.amazonaws.com/v1
kind: NodeClass
metadata:
  name: gpu
spec:
  ephemeralStorage:
    size: 200Gi
    iops: 5000
    throughput: 500
```

> Note: Auto Mode uses `eks.amazonaws.com/v1 NodeClass` instead of Karpenter's `karpenter.k8s.aws/v1 EC2NodeClass`.

## Networking in Auto Mode

- **Pod networking**: VPC CNI (built-in), pods get real VPC IPs
- **Network policies**: Built-in enforcement (no add-on required)
- **IPv4 and IPv6**: Both supported
- **Secondary CIDRs**: Supported for extending pod IP space
- **Load balancing**: Automatic ALB/NLB for Services and Ingress

### Using Load Balancers

```yaml
# NLB (default for Service type: LoadBalancer)
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: my-app
---
# ALB via Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  ingressClassName: alb
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

## Storage in Auto Mode

EBS CSI is built-in. Use standard StorageClasses:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "5000"
  throughput: "500"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
```

## Security Model

### What's Locked Down

- **No SSH/SSM**: You cannot access nodes directly. Use `kubectl exec`, `kubectl debug`, or DaemonSets for troubleshooting.
- **SELinux enforcing**: Mandatory access controls are enabled.
- **Read-only root filesystem**: AMI is immutable.
- **Bottlerocket-based**: Minimal OS, container-optimized.
- **21-day node lifetime**: Nodes are automatically replaced, ensuring fresh images.

### What You Still Control

- Pod security (PSS/PSA, OPA, Kyverno)
- RBAC
- Network policies
- VPC security groups
- IAM (cluster role, Pod Identity)

## Node Lifecycle

```
Node launched by Auto Mode
    │
    ├── Spot interruption notice → automatic drain + replace
    ├── EC2 status check failure → automatic repair/replace
    ├── Scheduled maintenance event → drain + replace
    ├── Consolidation (underutilized) → drain + replace with smaller
    └── Max lifetime reached (21 days) → drain + replace

All disruptions respect PDBs and NodePool Disruption Budgets (NDBs).
```

### Node Disruption Budgets (NDBs)

Control how many nodes can be disrupted simultaneously:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general-purpose-custom
spec:
  disruption:
    budgets:
      - nodes: "10%"
      - nodes: "0"
        schedule: "0 9 * * 1-5"  # No disruptions during business hours
        duration: 8h
```

## Monitoring and Troubleshooting

### No SSH — How to Debug

```sh
# kubectl debug (ephemeral container on a node)
kubectl debug node/<node-name> -it --image=busybox

# View node logs via kubectl
kubectl logs -n kube-system <pod-on-node>

# DaemonSet for monitoring
kubectl apply -f monitoring-daemonset.yaml
```

### Check Node Pool Status

```sh
kubectl get nodepools
kubectl get nodeclaims
kubectl get nodes -o wide
```

### Check Auto Mode Components

```sh
# Pods managed by Auto Mode
kubectl get pods -n kube-system

# Node status
kubectl describe node <node-name>
```

## Limitations

- **No SSH/SSM access** to nodes — cannot run arbitrary commands on the host
- **No custom AMIs** — Bottlerocket only, managed by AWS
- **Restricted privileged containers** — `hostNetwork`, `hostPID`, `hostIPC` require specific configuration
- **No node-level package installation** — everything runs in containers
- **21-day maximum node lifetime** — cannot keep nodes running longer (but can set shorter)
- **EC2 instances hidden from console** (since April 2026) — managed instances don't show in default EC2 views
- **Custom NodeClass uses `eks.amazonaws.com/v1`** — not the standard Karpenter `EC2NodeClass`

## Disadvantages

### Cost

- Higher per-pod pricing compared to self-managed node groups (management fee on top of EC2 cost)
- Less cost optimization control — you can't fine-tune instance selection as precisely as with raw Karpenter
- Potential for over-provisioning during traffic spikes before consolidation kicks in

### Control

- Reduced visibility into underlying EC2 instances (hidden from console by default)
- Cannot customize node-level configurations (no custom AMIs, no userdata, no kernel tuning)
- Limited control over instance placement strategies
- Cannot run alternative CNI plugins (Calico, Cilium) — locked to the built-in VPC CNI with network policies
- Cannot troubleshoot at the node level without `kubectl debug`

### Vendor Lock-in

- Tighter coupling to AWS services and pricing model than standard EKS
- Custom NodeClasses use `eks.amazonaws.com/v1` (not portable Karpenter `EC2NodeClass`)
- More difficult to migrate to other Kubernetes platforms or self-managed clusters
- Dependency on AWS's roadmap for feature development and K8s version support

### Feature Gaps

- Potential delays in supporting the newest Kubernetes versions (AWS tests compatibility first)
- Limited integration with third-party monitoring/security tools that require node-level agents (must use DaemonSets)
- Some specialized workloads (custom kernel modules, FUSE filesystems, eBPF programs) cannot run without host access

## When to Use Auto Mode

| Use Auto Mode When | Use Standard Mode When |
|-------------------|----------------------|
| You want minimal operational overhead | You need custom AMIs |
| Your team doesn't have deep EKS expertise | You need SSH access to nodes |
| Workloads are standard containers | You run specialized kernel modules |
| You want built-in network policies | You need full control over node configuration |
| Cost optimization via consolidation matters | You have strict compliance requiring specific OS versions |
| GPU workloads with managed drivers | You need to run non-containerized workloads on nodes |

## Pricing

Auto Mode pricing is based on compute consumed:

| Component | Cost |
|-----------|------|
| Control plane | $0.10/hour (same as standard) |
| Compute (Auto Mode nodes) | EC2 instance cost + EKS Auto Mode management fee |
| Storage (EBS) | Standard EBS pricing |
| Load balancers | Standard ALB/NLB pricing |
| Data transfer | Standard VPC pricing |

The management fee covers all the automation (scaling, patching, upgrades, health management). Check the [EKS pricing page](https://aws.amazon.com/eks/pricing/) for current rates.

## Migration from Standard to Auto Mode

```sh
# 1. Enable Auto Mode on existing cluster
aws eks update-cluster-config --name my-cluster --region us-east-1 \
  --compute-config enabled=true,nodePools=["general-purpose","system"]

# 2. Wait for Auto Mode to be active
aws eks describe-cluster --name my-cluster --query "cluster.computeConfig"

# 3. Taint old node groups (shift workloads to Auto Mode nodes)
kubectl taint nodes -l eks.amazonaws.com/nodegroup=old-ng migration=true:PreferNoSchedule

# 4. Verify workloads running on Auto Mode nodes
kubectl get pods -A -o wide | grep -v <old-node-prefix>

# 5. Scale down old node groups
aws eks update-nodegroup-config --cluster-name my-cluster --nodegroup-name old-ng \
  --scaling-config minSize=0,desiredSize=0,maxSize=0

# 6. Delete old node groups
aws eks delete-nodegroup --cluster-name my-cluster --nodegroup-name old-ng
```


## Targeting Auto Mode Nodes

Use the `eks.amazonaws.com/compute-type: auto` nodeSelector to ensure workloads land on Auto Mode nodes (useful when running Auto Mode alongside existing managed node groups):

```yaml
spec:
  nodeSelector:
    eks.amazonaws.com/compute-type: auto
  containers:
    - name: app
      image: my-app:latest
      resources:
        requests:
          cpu: "500m"
          memory: "512Mi"
```

## Default NodePool Configuration

The built-in `general-purpose` NodePool has these defaults:

```yaml
# kubectl describe nodepool general-purpose (key fields)
spec:
  disruption:
    budgets:
      nodes: 10%
    consolidateAfter: 30s
    consolidationPolicy: WhenEmptyOrUnderutilized
  template:
    spec:
      expireAfter: 336h    # 14 days (within the 21-day max)
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: eks.amazonaws.com/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: eks.amazonaws.com/instance-generation
          operator: Gt
          values: ["4"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
      terminationGracePeriod: 24h
```

Key takeaways:
- Only provisions from **c, m, r** instance families (compute, general-purpose, memory)
- Only generations **newer than 4** (m5+, c5+, r5+)
- On-demand capacity only (no Spot in the default pool)
- Consolidation enabled with 30-second delay
- 10% disruption budget

## EKS Auto Mode vs Fargate

| Feature | Auto Mode | Fargate |
|---------|-----------|---------|
| Compute layer | EC2 instances (managed) | Serverless (no instances) |
| DaemonSets | Yes | No |
| Stateful storage (EBS) | Yes | No (ephemeral only) |
| GPU support | Yes | No |
| Node visibility | Limited (hidden by default) | None (no nodes at all) |
| Networking | Full VPC CNI | VPC CNI (limited) |
| Scaling speed | Fast (Karpenter-based) | Slower (pod-level provisioning) |
| Pricing | EC2 + management fee | Per-pod vCPU + memory per second |
| Best for | General workloads, GPU, stateful | Stateless microservices, batch |

Choose Fargate when you want zero node management and your workloads are stateless without DaemonSets. Choose Auto Mode when you need DaemonSets, stateful storage, GPUs, or more control over scheduling.

## Best Practices

- **Always set resource requests**: Auto Mode's cost optimization (Karpenter-based consolidation) relies on accurate pod resource requests. Without them, right-sizing and bin-packing don't work effectively.
- **Use cost allocation tags**: Apply AWS tags consistently across namespaces and workloads for detailed cost visibility.
- **Start with default NodePools**: Only create custom NodePools when the defaults don't meet your needs (GPU, Spot, specific instance types).
- **Set PDBs on all production workloads**: Auto Mode respects PDBs during consolidation and node rotation. Without PDBs, pods may be disrupted during the 21-day refresh cycle.
- **Monitor consolidation behavior**: Watch for excessive pod movement. If consolidation is too aggressive, increase `consolidateAfter` in a custom NodePool.
- **Use `eks.amazonaws.com/compute-type: auto` selector**: When migrating from managed node groups, this ensures new workloads target Auto Mode nodes specifically.
