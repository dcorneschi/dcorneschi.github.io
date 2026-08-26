# EKS Auto Mode

EKS Auto Mode extends AWS management to the data plane — AWS handles compute, networking, storage, load balancing, and upgrades so you can focus on deploying workloads. Announced December 2024, it's the most managed EKS experience available.

For commands, YAML manifests, and operational reference, see [EKS Auto Mode Cheatsheet](articles/eks-auto-mode-cheatsheet.md).

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
| Network policies | Add-on required (Calico/Cilium) | Built-in (eBPF-based) |
| Load balancer controller | You deploy | Built-in |
| EBS CSI driver | You deploy | Built-in |
| Pod Identity Agent | You deploy | Built-in |
| CoreDNS | Deployment (you manage replicas) | System service per node |
| Node max lifetime | Unlimited (you control) | 21 days (configurable lower) |
| Pricing | EC2 per-instance + $0.10/hr control plane | EKS Auto Mode compute pricing + $0.10/hr control plane |

## Auto Mode vs Fargate vs Self-Managed Karpenter

| | EKS Auto Mode | EKS with Fargate | Self-Managed Karpenter |
|---|---|---|---|
| **Compute** | EC2 (managed by AWS) | Serverless (no instances) | EC2 (you manage) |
| **Management fee** | 12% on EC2 | Fargate pricing (vCPU + memory/s) | None |
| **DaemonSets** | Supported | Not supported | Supported |
| **Privileged containers** | Supported | Not supported | Supported |
| **Persistent storage** | EBS (auto-managed CSI) | EFS only | EBS/EFS (you install CSI) |
| **GPU / accelerators** | Supported (auto driver install) | Not supported | Supported (manual driver setup) |
| **Custom AMI** | No (Bottlerocket only) | N/A | Yes |
| **SSH to nodes** | No | N/A | Yes |
| **Pod limit per node** | 110 (fixed) | 1 pod per Fargate instance | Configurable |
| **Autoscaling** | Built-in Karpenter | Per-pod scaling | You install & configure |
| **OS patching** | Automatic (21-day rotation) | AWS-managed | You manage |
| **Add-on management** | AWS-managed (CoreDNS, kube-proxy, VPC CNI, LB controller) | AWS-managed | You install & update |
| **Spot instances** | Supported (separate pool) | Not supported | Supported (flexible config) |
| **Operational overhead** | Low | Lowest | Highest |
| **Control / flexibility** | Medium | Low | High |
| **Best for** | Teams wanting managed infra with K8s flexibility | Simple stateless workloads | Teams needing full control |

Choose Fargate when you want zero node management and your workloads are stateless without DaemonSets. Choose Auto Mode when you need DaemonSets, stateful storage, GPUs, or more control over scheduling.

## EKS Auto Mode vs GKE Autopilot vs AKS Automatic

| | EKS Auto Mode | GKE Autopilot | AKS Automatic |
|---|---|---|---|
| **Provider** | AWS | Google Cloud | Microsoft Azure |
| **Compute** | Managed EC2 (Bottlerocket) | Managed GCE (COS) | Managed VMSS |
| **Pricing model** | EC2 cost + 12% management fee | Flat cluster fee + per-pod (vCPU/mem/s) | Standard AKS pricing |
| **Autoscaler** | Karpenter (built-in) | GKE NAP (built-in) | Karpenter + Azure workload autoscaling |
| **Node access** | No SSH, no SSM, no public IP | No SSH | Limited (Azure Serial Console) |
| **Custom AMI/image** | No (Bottlerocket only) | No (COS only) | No |
| **DaemonSets** | Supported | Supported (with restrictions) | Supported |
| **Privileged containers** | Supported | Not supported | Supported (with PSS) |
| **GPU / accelerators** | Supported (auto driver install) | Supported | Supported |
| **Pod limit per node** | 110 (fixed) | 110 (default, configurable) | Configurable |
| **Network policy** | VPC CNI (custom config limited) | GKE Dataplane V2 (enforced) | Azure CNI (Cilium) |
| **Control plane upgrades** | Manual (`aws eks update-cluster-version`) | Automatic (with maintenance windows) | Automatic (with maintenance windows) |
| **Node OS upgrades** | Automatic (21-day rotation) | Automatic | Automatic |
| **Add-on management** | AWS-managed (systemd services) | Google-managed | Azure-managed |
| **Spot/preemptible** | Supported (separate NodePool) | Supported (Spot pods) | Supported |
| **Persistent storage** | EBS (auto-managed CSI) | PD CSI (auto-managed) | Azure Disk CSI (auto-managed) |
| **Security model** | Immutable AMI, SELinux enforcing, read-only rootfs | Hardened COS, Workload Identity | Azure Policy, Entra ID integration |
| **Maturity** | GA since re:Invent 2024 | GA since 2021 | GA since 2024 |

**Key takeaways:**
- **GKE Autopilot** is the most opinionated — pod-level billing, enforced network policies, no privileged containers
- **AKS Automatic** has the broadest automation scope — control plane + nodes + networking + security policies
- **EKS Auto Mode** sits in the middle — strong node automation while preserving Kubernetes API flexibility and Spot support

## Architecture: NodePool → NodeClass → NodeClaim → Node

```
┌─────────────────────────────────────────────────────────────────────┐
│                        EKS Auto Mode                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐         ┌──────────────────┐                      │
│  │  NodePool    │────────▶│   NodeClass      │                      │
│  │              │  refs   │   (default)      │                      │
│  │ - constraints│         │                  │                      │
│  │ - limits     │         │ - role (IAM)     │                      │
│  │ - disruption │         │ - subnets        │                      │
│  │              │         │ - security groups│                      │
│  └──────┬───────┘         │ - storage config │                      │
│         │                 │ - instance prof. │                      │
│         │ creates         └──────────────────┘                      │
│         ▼                                                           │
│  ┌──────────────┐         ┌──────────────────┐                      │
│  │  NodeClaim   │────────▶│   EC2 Instance   │                      │
│  │              │launches │   (Node)         │                      │
│  │ - instance   │         │                  │                      │
│  │   type       │         │ - joins cluster  │                      │
│  │ - zone       │         │ - runs pods      │                      │
│  │ - capacity   │         │                  │                      │
│  └──────────────┘         └──────────────────┘                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

Flow:
  Pod Pending → NodePool matches → NodeClaim created → NodeClass provides
  config (subnets, SG, IAM) → EC2 launched → Node joins → Pod scheduled
```

### Default Node Pools

Auto Mode creates two default NodePools:

| Pool | Purpose | Instance Types | Capacity |
|------|---------|---------------|----------|
| `system` | System workloads (kube-proxy, CoreDNS) | Optimized for small, reliable | On-demand |
| `general-purpose` | Application workloads | C, M, R families (gen 5+) | On-demand |

You cannot edit default NodePools, but you can create custom ones alongside them.

> Note: Auto Mode uses `eks.amazonaws.com/v1 NodeClass` instead of Karpenter's `karpenter.k8s.aws/v1 EC2NodeClass`.

## Networking in Auto Mode

- **VPC CNI** — pods get native VPC IP addresses (no overlay network). Fully managed with automatic upgrades and rollback.
- **Prefix delegation** — enabled by default. Assigns /28 prefixes (16 IPs each) to ENIs, increasing pod density per node.
- **Network Policies** — eBPF-based enforcement (higher performance than iptables). Supports standard Kubernetes NetworkPolicy plus Admin Network Policies and DNS Network Policies (FQDN-based egress filtering).
- **Pod subnet isolation** — pods can run in different subnets than nodes via `NodeClass.podSubnetSelectorTerms`.
- **SNAT policy** — configurable via NodeClass. Default is `Random` (pods use the node's primary IP). Set to `Disabled` for pod-level traceability.
- **DNS** — managed cluster DNS with node-level caching for reduced latency.
- **Load balancing** — automatic ALB/NLB for Services and Ingress, no controller install needed.
- **IPv4 and IPv6** — both supported.
- **Secondary CIDRs** — supported for extending pod IP space.

### Subnet Planning

For large clusters, use dedicated pod subnets to avoid IP exhaustion:

- Add a secondary CIDR block (e.g., `100.64.0.0/10` from RFC 6598) for pod addressing
- Keep primary VPC CIDR for nodes and load balancers
- Configure via `NodeClass.podSubnetSelectorTerms`
- A /20 per AZ provides ~4,000 addresses — enough for rolling replacements and surge

### Hybrid Connectivity

EKS Auto Mode pods receive native VPC IPs, making them fully routable through:

- **AWS Site-to-Site VPN** or **AWS Direct Connect** for on-premises
- **AWS Transit Gateway** for multi-VPC architectures
- **Route 53 Resolver rules** for on-premises DNS resolution
- Standard VPC route tables and security groups

## Security Model

### What's Locked Down

- **No SSH/SSM**: You cannot access nodes directly. Use `kubectl exec`, `kubectl debug`, or DaemonSets for troubleshooting.
- **SELinux enforcing**: Mandatory access controls are enabled. Pods get unique MCS labels.
- **Read-only root filesystem**: AMI is immutable.
- **Bottlerocket-based**: Minimal OS, container-optimized, host containers disabled.
- **21-day node lifetime**: Nodes are automatically replaced, ensuring fresh images.
- **IMDS locked down**: IMDSv2 only with hop limit 1 — blocks non-host-network pods from accessing node credentials.
- **EBS volumes encrypted**: Root and data volumes encrypted by default, deleted on termination.

### What You Still Control

- Pod security (PSS/PSA, OPA, Kyverno)
- RBAC
- Network policies
- VPC security groups
- IAM (cluster role, Pod Identity)

For a full security deep dive, see [EKS Auto Mode Security Deep Dive](articles/eks-auto-mode-security.md).

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

### Consolidation Methods

| Method | When Applied | What Happens |
|--------|-------------|-------------|
| **Node deletion** | All pods on a node can fit on available capacity of other nodes | Node is drained and terminated |
| **Node replacement** | Pods can be redistributed across existing nodes + one cheaper replacement | Old node replaced with a lower-cost instance |

### Automatic Node Health Repair

EKS Auto Mode includes built-in health monitoring that can automatically repair nodes for:
- Unresponsive kubelet
- Exhausted process IDs (PID pressure)
- Other failure modes identified by AWS from operating EKS at scale

Unhealthy nodes are replaced (not repaired in-place) to minimize disruption.

## Limitations

- **No SSH/SSM access** to nodes — cannot run arbitrary commands on the host
- **No custom AMIs** — Bottlerocket only, managed by AWS
- **Restricted privileged containers** — `hostNetwork`, `hostPID`, `hostIPC` require specific configuration
- **No node-level package installation** — everything runs in containers
- **21-day maximum node lifetime** — cannot keep nodes running longer (but can set shorter)
- **110 pods per node** — not configurable, can lead to more nodes than self-managed Karpenter
- **Cannot mix Spot and On-Demand** in the same NodePool — separate pools required
- **EC2 instances hidden from console** (since April 2026) — managed instances don't show in default EC2 views
- **Custom NodeClass uses `eks.amazonaws.com/v1`** — not the standard Karpenter `EC2NodeClass`
- **Control plane upgrades still manual** — node OS/components auto-update, but control plane K8s version requires `aws eks update-cluster-version`
- **Min K8s version** — requires Kubernetes 1.29 or greater

## Disadvantages

### Cost

- 12% management fee on top of EC2 instance costs
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

## When to Use Auto Mode vs Standard EKS

### Use EKS Auto Mode when:

- **Greenfield projects** — start here, avoid building node management tech debt
- **Dynamic/bursty workloads** — AI/ML training, CI/CD runners, batch jobs that scale 0→100 nodes
- **Small Platform teams** — spend time on developer experience, not node patching
- **Day 2 operations are painful** — if your backlog is full of upgrades, AMI rotations, and add-on version conflicts
- **Cost optimization without tuning** — let Karpenter bin-pack and consolidate automatically

### Use EKS Standard when:

- **Custom kernel requirements** — proprietary kernel modules, custom sysctl parameters, specific hardened AMIs (CIS benchmarks)
- **Instance store NVMe** — workloads mounting local NVMe drives in ways the CSI driver doesn't support
- **Node-level debugging required** — compliance frameworks requiring SSH/SSM for forensic analysis
- **Legacy applications** — apps needing host-level configuration, specific OS packages, or writable root filesystem
- **Full Karpenter control** — need custom consolidation logic, pod limits >110, or unsupported instance configurations

### Hybrid approach

You can run **both modes on the same cluster** — Managed Node Groups alongside Auto Mode nodes. Use `eks.amazonaws.com/compute-type: auto` nodeSelector to target Auto Mode, and standard node labels for your managed groups.

## Pricing

| Component | Cost |
|-----------|------|
| Control plane | $0.10/hour (same as standard) |
| Compute (Auto Mode nodes) | EC2 instance cost + 12% EKS Auto Mode management fee |
| Storage (EBS) | Standard EBS pricing |
| Load balancers | Standard ALB/NLB pricing |
| Data transfer | Standard VPC pricing |

The management fee covers all the automation (scaling, patching, upgrades, health management). Check the [EKS pricing page](https://aws.amazon.com/eks/pricing/) for current rates.

## Best Practices

- **Always set resource requests**: Auto Mode's cost optimization (Karpenter-based consolidation) relies on accurate pod resource requests. Without them, right-sizing and bin-packing don't work effectively.
- **Use cost allocation tags**: Apply AWS tags consistently across namespaces and workloads for detailed cost visibility.
- **Start with default NodePools**: Only create custom NodePools when the defaults don't meet your needs (GPU, Spot, specific instance types).
- **Set PDBs on all production workloads**: Auto Mode respects PDBs during consolidation and node rotation. Without PDBs, pods may be disrupted during the 21-day refresh cycle.
- **Monitor consolidation behavior**: Watch for excessive pod movement. If consolidation is too aggressive, increase `consolidateAfter` in a custom NodePool.
- **Use `eks.amazonaws.com/compute-type: auto` selector**: When migrating from managed node groups, this ensures new workloads target Auto Mode nodes specifically.
- **Use `securityContext`** — nodes enforce SELinux, so set `runAsUser`, `runAsGroup`, and `allowPrivilegeEscalation: false`.

## Links

- [EKS Auto Mode Documentation](https://docs.aws.amazon.com/eks/latest/userguide/automode.html)
- [EKS Auto Mode Best Practices](https://docs.aws.amazon.com/eks/latest/best-practices/automode.html)
- [EKS Auto Mode Cheatsheet](articles/eks-auto-mode-cheatsheet.md) — Commands, YAML manifests, kubectl one-liners, Terraform, and troubleshooting
- [EKS Auto Mode Security Deep Dive](articles/eks-auto-mode-security.md) — IMDS, EBS encryption, IAM, pod networking segregation, SCPs, GuardDuty
- [Deploy 2048 Game on EKS Auto Mode](articles/eks-auto-mode-2048-game.md) — Hands-on tutorial
- [Enterprise Networking with EKS Auto Mode](https://aws.amazon.com/blogs/containers/navigating-enterprise-networking-challenges-with-amazon-eks-auto-mode/)
- [Under the Hood: EKS Auto Mode](https://aws.amazon.com/blogs/containers/under-the-hood-amazon-eks-auto-mode/)
- [Cost Optimization](https://docs.aws.amazon.com/eks/latest/userguide/auto-cost-control.html)
- [Migrate from Karpenter to EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/auto-migrate-karpenter.html)
- [Migrate from Managed Node Groups to EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/auto-migrate-mng.html)
