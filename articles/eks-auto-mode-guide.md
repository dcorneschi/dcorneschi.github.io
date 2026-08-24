# Amazon EKS Auto Mode

Related: [EKS Auto Mode Documentation](https://docs.aws.amazon.com/eks/latest/userguide/automode.html)

## What Is EKS Auto Mode?

EKS Auto Mode extends AWS management of Kubernetes clusters beyond the control plane to also manage the data plane infrastructure — compute, networking, storage, and load balancing. It automates infrastructure provisioning, instance selection, dynamic scaling, cost optimization, OS patching, and security integration.

You deploy workloads; AWS handles the underlying infrastructure decisions.

## What EKS Auto Mode Manages

| Component | What's Automated |
|-----------|-----------------|
| **Compute** | Node provisioning, instance type selection, auto scaling (Karpenter-based), Spot interruption handling, GPU support |
| **Networking** | VPC CNI (managed), pod IP assignment, network policies (eBPF-based), DNS (node-level caching), SNAT policy |
| **Load Balancing** | ALB and NLB provisioning via Ingress/Service resources — no separate LB controller needed |
| **Storage** | EBS CSI (managed), ephemeral storage configuration, encryption |
| **Upgrades** | Automatic node patching, 21-day max node lifetime, automated rollbacks |
| **Identity** | Pod Identity Agent built-in (no add-on install needed) |

## EKS Auto Mode vs Standard EKS

| Aspect | Standard EKS | EKS Auto Mode |
|--------|-------------|---------------|
| Node provisioning | You manage (managed node groups, self-managed, Fargate) | AWS manages (Karpenter-based auto scaling) |
| VPC CNI | You install and configure the add-on | Fully managed, optimized settings |
| Load balancer controller | You install AWS LB Controller | Built-in, no controller needed |
| EBS CSI driver | You install the add-on + IAM roles | Built-in |
| Node upgrades | You trigger and monitor | Automatic with rollback |
| Pod Identity Agent | You install the add-on | Built-in |
| DNS | CoreDNS add-on you manage | Managed with node-level caching |
| Cost optimization | You configure Karpenter/CA | Built-in consolidation and right-sizing |
| SSH/SSM access to nodes | Available | Disabled (nodes are immutable appliances) |

## Key Features

### Compute

- Nodes are treated as immutable appliances — no SSH, no SSM, read-only root filesystem, SELinux enforcing mode
- Auto scaling via Karpenter — watches for unschedulable pods, launches appropriate instances
- Automatic consolidation — terminates underutilized nodes, moves workloads to fewer nodes
- 21-day maximum node lifetime (configurable lower) — ensures regular patching
- GPU support for NVIDIA and Neuron (separate kernel drivers and plugins)
- Handles EC2 Spot interruption notices, health events, and instance status check failures automatically

### Networking

- **VPC CNI** — pods get native VPC IP addresses (no overlay network). Fully managed with automatic upgrades and rollback.
- **Prefix delegation** — enabled by default. Assigns /28 prefixes (16 IPs each) to ENIs instead of individual IPs, increasing pod density per node (e.g., c5.4xlarge: 110 pods vs 58 with secondary IP mode).
- **Network Policies** — eBPF-based enforcement (higher performance than iptables). Supports standard Kubernetes NetworkPolicy plus Admin Network Policies and DNS Network Policies (FQDN-based egress filtering).
- **Pod subnet isolation** — pods can run in different subnets than nodes via `NodeClass.podSubnetSelectorTerms`.
- **SNAT policy** — configurable via NodeClass. Default is `Random` (pods use the node's primary IP). Set to `Disabled` for pod-level traceability.
- **DNS** — managed cluster DNS with node-level caching for reduced latency.

### Load Balancing

No separate load balancer controller is needed. EKS Auto Mode provisions load balancers directly from Kubernetes resources:

**ALB (Layer 7)** — create an Ingress with IngressClass controller `eks.amazonaws.com/alb`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
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
            name: my-app
            port:
              number: 80
```

**NLB (Layer 4)** — create a Service of type LoadBalancer with class `eks.amazonaws.com/nlb`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  loadBalancerClass: eks.amazonaws.com/nlb
  selector:
    app: my-app
  ports:
  - port: 443
    targetPort: 8443
```

### Storage

- EBS CSI driver is built-in (no add-on installation)
- Ephemeral storage configured automatically (volume type, size, encryption, deletion policy)
- Default StorageClass available out of the box

### Security

- Nodes use locked-down AMIs with SELinux enforcing mode
- Read-only root filesystem on nodes
- No SSH/SSM access (debug via ephemeral containers or kubectl)
- 21-day node rotation ensures regular patching
- Network Policies with eBPF enforcement
- Admin Network Policies for cluster-wide security rules
- DNS Network Policies for FQDN-based egress control
- Integration with AWS Shield Advanced for DDoS protection

## NodePool and NodeClass

EKS Auto Mode uses two custom resources for compute and networking configuration:

### NodePool

Controls which instance types, availability zones, and capacity types (on-demand/spot) are available:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general-purpose
spec:
  template:
    spec:
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      requirements:
      - key: node.kubernetes.io/instance-type
        operator: In
        values: ["m5.large", "m5.xlarge", "m5.2xlarge"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["on-demand", "spot"]
      - key: topology.kubernetes.io/zone
        operator: In
        values: ["us-west-2a", "us-west-2b"]
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    budgets:
    - nodes: "10%"
```

### NodeClass

Controls networking (subnets, security groups, SNAT) and storage:

```yaml
apiVersion: eks.amazonaws.com/v1
kind: NodeClass
metadata:
  name: default
spec:
  subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: my-cluster
  podSubnetSelectorTerms:
  - tags:
      kubernetes.io/role/pod: "1"
  snatPolicy: Random
```

> **Note:** Don't edit the default NodePool/NodeClass. Create custom ones alongside the defaults for specific workload requirements.

## Subnet Planning

For large clusters, use dedicated pod subnets to avoid IP exhaustion:

- Add a secondary CIDR block (e.g., `100.64.0.0/10` from RFC 6598) for pod addressing
- Keep primary VPC CIDR for nodes and load balancers
- Configure via `NodeClass.podSubnetSelectorTerms`
- A /20 per AZ provides ~4,000 addresses — enough for rolling replacements and surge

## Hybrid Connectivity

EKS Auto Mode pods receive native VPC IPs, making them fully routable through:

- **AWS Site-to-Site VPN** or **AWS Direct Connect** for on-premises
- **AWS Transit Gateway** for multi-VPC architectures
- **Route 53 Resolver rules** for on-premises DNS resolution
- Standard VPC route tables and security groups

## Getting Started

### Create a new cluster with Auto Mode

```bash
# Using eksctl
eksctl create cluster --name my-cluster --region us-west-2 --auto-mode

# Using AWS CLI
aws eks create-cluster \
  --name my-cluster \
  --region us-west-2 \
  --compute-config enabled=true \
  --kubernetes-networking-config elasticLoadBalancing=enabled=true \
  --storage-config blockStorage=enabled=true
```

### Enable Auto Mode on an existing cluster

```bash
eksctl update cluster --name my-cluster --region us-west-2 --auto-mode
```

### Verify

```bash
kubectl get nodepools
kubectl get nodeclasses
kubectl get nodes
```

## Limitations

- No SSH/SSM access to nodes — use `kubectl debug` or ephemeral containers for troubleshooting
- Cannot install DaemonSets that require privileged access to the host (some monitoring agents may need adjustments)
- Custom AMIs are not supported — nodes use AWS-managed AMIs
- Node lifetime capped at 21 days (not configurable higher)
- Some Karpenter features may have different defaults or restrictions compared to self-managed Karpenter

## Links

- [EKS Auto Mode Documentation](https://docs.aws.amazon.com/eks/latest/userguide/automode.html)
- [EKS Auto Mode Best Practices](https://docs.aws.amazon.com/eks/latest/best-practices/automode.html)
- [Enterprise Networking with EKS Auto Mode](https://aws.amazon.com/blogs/containers/navigating-enterprise-networking-challenges-with-amazon-eks-auto-mode/)
- [Under the Hood: EKS Auto Mode](https://aws.amazon.com/blogs/containers/under-the-hood-amazon-eks-auto-mode/)
