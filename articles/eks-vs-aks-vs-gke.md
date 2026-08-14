# EKS vs AKS vs GKE: Managed Kubernetes Compared

A comparison of the three major managed Kubernetes services — Amazon EKS, Azure AKS, and Google GKE — covering architecture, networking, pricing, IAM integration, and operational differences.

## At a Glance

| Feature | EKS (AWS) | AKS (Azure) | GKE (Google Cloud) |
|---------|-----------|-------------|-------------------|
| Control plane cost | $0.10/hour ($73/month) | Free | Free (Standard); $0.10/hour (Enterprise) |
| Control plane SLA | 99.95% (Standard), 99.99% (Provisioned) | 99.95% (with AZs), 99.9% (without) | 99.95% (Regional), 99.5% (Zonal) |
| Max nodes per cluster | 100,000 | 5,000 | 15,000 |
| Max pods per node | 250 (110 with prefix delegation) | 250 | 110 (Standard), 256 (with GKE Dataplane V2) |
| Default CNI | AWS VPC CNI (real VPC IPs) | Azure CNI or kubenet | GKE VPC-native (Alias IPs) |
| Managed node upgrades | Rolling (surge-based) | Rolling (surge/max unavailable) | Rolling (surge, blue-green, or canary) |
| Serverless pods | Fargate | ACI (Virtual Nodes) | Autopilot mode |
| GPU support | Yes (P/G/Trn/Inf instances) | Yes (NC/ND series) | Yes (T4, A100, H100, TPU) |
| CLI tool | `eksctl`, `aws eks` | `az aks` | `gcloud container clusters` |

## Control Plane

### Architecture

| Aspect | EKS | AKS | GKE |
|--------|-----|-----|-----|
| Where it runs | AWS-managed account (hidden) | Azure-managed infra (hidden) | Google-managed infra (hidden) |
| etcd | 3 nodes across 3 AZs | Managed (CosmosDB-based etcd or native) | Managed (distributed etcd) |
| API server replicas | Min 2, auto-scaled | Managed, scaled by Azure | Managed, auto-scaled |
| etcd access | No | No | No |
| Control plane logging | CloudWatch Logs (opt-in) | Azure Monitor / Log Analytics (opt-in) | Cloud Logging (enabled by default) |
| Upgrades | Manual trigger, rolling | Manual or auto, rolling | Manual, auto, or maintenance windows |

### Control Plane Cost

| Service | Standard | Premium/Enterprise |
|---------|----------|-------------------|
| **EKS** | $0.10/hour per cluster | Provisioned: higher (tiered by size) |
| **AKS** | Free | Premium: $0.10/hour per cluster |
| **GKE** | Free (Autopilot compute charges apply) | Enterprise: $0.10/hour per cluster |

GKE and AKS both offer a free control plane for their standard tier. EKS charges from day one.

## Networking

### Pod Networking Model

| Feature | EKS | AKS | GKE |
|---------|-----|-----|-----|
| Default mode | VPC CNI (pods get real VPC IPs) | Azure CNI (pods get VNet IPs) or kubenet (overlay) | VPC-native (Alias IP ranges) |
| Pod IPs routable in VPC/VNet | Yes | Yes (Azure CNI) / No (kubenet) | Yes |
| Overlay option | Not default (can use Cilium) | kubenet, Azure CNI Overlay | GKE Dataplane V2 (Cilium-based) |
| IP exhaustion risk | High (plan large subnets) | Medium (Azure CNI) / Low (kubenet) | Medium (secondary ranges) |
| Network Policy | Calico or Cilium (add-on) | Azure NPM or Calico | GKE Dataplane V2 (Cilium, built-in) |

### How Pod IPs Work

**EKS:** Each pod gets a secondary IP from the node's ENI. IPs come from the same VPC subnet as the node (or secondary CIDR with custom networking). You must plan for large subnets because every pod consumes a real IP.

**AKS (Azure CNI):** Each pod gets a VNet IP from the node's subnet. Similar to EKS — IP planning required. Azure CNI Overlay mode uses an overlay network (10.244.0.0/16 by default) to avoid IP exhaustion.

**GKE:** Uses Alias IP ranges. Nodes get a primary IP from the node subnet, and pods get IPs from a separate secondary range (Pod CIDR). GCP routes know how to reach these IPs natively.

### Service Networking

| Feature | EKS | AKS | GKE |
|---------|-----|-----|-----|
| Service CIDR | `172.20.0.0/16` or `10.100.0.0/16` | `10.0.0.0/16` (configurable) | `10.96.0.0/12` (configurable) |
| Load Balancer integration | AWS LB Controller (ALB/NLB) | Azure Load Balancer (auto) | GCE LB (auto), Gateway API |
| Ingress | AWS LB Controller (ALB) | AGIC or nginx | GKE Ingress (GCLB) or Gateway API |
| Service Mesh | App Mesh (deprecated), Istio | Istio-based (Azure Service Mesh) | Istio (managed), Traffic Director |

## IAM and Authentication

### How Pods Get Cloud Credentials

| Feature | EKS | AKS | GKE |
|---------|-----|-----|-----|
| Mechanism | IRSA or Pod Identity | Workload Identity (Azure AD) | Workload Identity Federation (GCP SA) |
| How it works | Pod JWT → STS → temp creds | Pod JWT → Azure AD → token | Pod KSA → GCP SA → token |
| Per-pod granularity | Yes | Yes | Yes |
| Legacy (avoid) | Node instance profile | Pod-managed identity (deprecated) | Node SA (too broad) |

### User Authentication to API Server

| Feature | EKS | AKS | GKE |
|---------|-----|-----|-----|
| Auth method | IAM + aws-iam-authenticator | Azure AD (Entra ID) | Google IAM / gcloud |
| Token mechanism | Pre-signed STS URL | Azure AD bearer token | Google OAuth2 token |
| RBAC integration | aws-auth ConfigMap or Access Entries | Azure RBAC or Kubernetes RBAC | GKE RBAC or IAM-based |
| MFA support | Via IAM policies | Azure AD Conditional Access | Via Google Workspace |

## Node Management

### Node Pool / Node Group Comparison

| Feature | EKS | AKS | GKE |
|---------|-----|-----|-----|
| Terminology | Node Group (managed) or ASG (self-managed) | Node Pool | Node Pool |
| Auto-scaling | Cluster Autoscaler or Karpenter | Cluster Autoscaler or KEDA | Cluster Autoscaler or NAP (Node Auto-Provisioning) |
| Spot/Preemptible | Spot Instances | Spot VMs | Spot VMs |
| GPU scheduling | Via node selectors/taints | Via node selectors/taints | Via node selectors/taints, TPU support |
| OS options | Amazon Linux 2/2023, Bottlerocket, Ubuntu, Windows | Ubuntu, Azure Linux (Mariner), Windows | Container-Optimized OS (COS), Ubuntu, Windows |
| Auto-repair | No (use NTH) | Yes (built-in) | Yes (built-in) |
| Auto-upgrade | Yes (configurable) | Yes (configurable) | Yes (maintenance windows, channels) |

### Serverless / Nodeless Options

| Feature | EKS Fargate | AKS Virtual Nodes | GKE Autopilot |
|---------|-------------|-------------------|---------------|
| What it is | AWS manages nodes, per-pod billing | ACI-backed pods (burst) | Fully managed nodes, per-pod billing |
| Node visibility | No nodes visible | Virtual node visible | Nodes visible but managed |
| Use case | Steady-state workloads | Burst scaling | Default for new clusters |
| Limitations | No DaemonSets, no privileged pods | Limited networking, no persistent volumes | Restricted security contexts |
| GPU support | No | No | Yes |

## Upgrades

### Cluster Upgrade Strategies

| Feature | EKS | AKS | GKE |
|---------|-----|-----|-----|
| Control plane upgrade | Manual trigger (or platform auto-patches) | Manual or auto-upgrade | Manual, auto (rapid/regular/stable channels) |
| Node upgrade | Separate from control plane, manual trigger | Separate, manual or auto | Separate, auto with maintenance windows |
| Rolling strategy | Surge-based (maxUnavailable) | Surge or max unavailable | Surge, blue-green, or canary |
| Canary upgrades | Not built-in (use separate node groups) | Not built-in | Built-in (per-pool canary) |
| Version skew allowed | Control plane +2 minor ahead of nodes | Control plane +2 minor ahead | Control plane +2 minor ahead |
| Rollback | Cannot downgrade control plane | Cannot downgrade control plane | Cannot downgrade control plane |

### Release Channels / Support

| Feature | EKS | AKS | GKE |
|---------|-----|-----|-----|
| Standard support | 14 months per version | 12 months per version | ~14 months per version |
| Extended support | Yes (additional cost) | Yes (additional cost) | Yes (additional cost) |
| Release channels | N/A (you pick a version) | N/A (you pick a version) | Rapid, Regular, Stable |
| Auto-upgrade channels | N/A | N/A | Yes (tied to channel) |

## Observability

| Feature | EKS | AKS | GKE |
|---------|-----|-----|-----|
| Built-in monitoring | CloudWatch Container Insights (add-on) | Azure Monitor Container Insights (built-in) | Cloud Monitoring (built-in) |
| Managed Prometheus | Amazon Managed Prometheus (AMP) | Azure Monitor managed Prometheus | Google Managed Prometheus (GMP, built-in) |
| Logging | CloudWatch Logs (opt-in) | Azure Monitor Logs | Cloud Logging (enabled by default) |
| Cost visibility | AWS Cost Explorer + Kubecost | Azure Cost Analysis + AKS cost views | GKE cost allocation (built-in) |
| Tracing | AWS X-Ray | Azure Monitor Application Insights | Cloud Trace |

## Security

| Feature | EKS | AKS | GKE |
|---------|-----|-----|-----|
| etcd encryption | Envelope encryption with KMS (opt-in) | Encryption at rest (default) + KMS (opt-in) | Encryption at rest (default) + CMEK |
| Secrets encryption | KMS envelope encryption (opt-in) | Azure Key Vault CSI (add-on) | GCP KMS (application-layer encryption) |
| Network Policy | Calico, Cilium (add-on) | Azure NPM, Calico (built-in option) | GKE Dataplane V2 / Cilium (built-in) |
| Pod Security | PSS/PSA (Kubernetes native) | PSS/PSA + Azure Policy | PSS/PSA + GKE Policy Controller |
| Image signing | No built-in (use Sigstore) | Notary v2 / ORAS | Binary Authorization (built-in) |
| Private cluster | Private endpoint + VPC | Private cluster (no public IP on nodes) | Private cluster (private endpoint) |

## Multi-Cluster and Hybrid

| Feature | EKS | AKS | GKE |
|---------|-----|-----|-----|
| Multi-cluster management | EKS Connector (limited) | Azure Arc | GKE Enterprise (Anthos) |
| On-premises K8s | EKS Anywhere | AKS Arc (formerly AKS-HCI) | GKE on-prem (Anthos) |
| Edge | EKS Anywhere, Outposts | AKS Edge Essentials | GKE on Edge |
| Service mesh (multi-cluster) | Istio (manual) | Azure Service Mesh | Managed Istio, Traffic Director |

## When to Choose Which

### Choose EKS When

- You're already heavily invested in AWS (VPC, IAM, CloudWatch ecosystem)
- You need maximum flexibility (self-managed nodes, custom AMIs, Karpenter)
- You're running large-scale clusters (100K node support)
- You need tight integration with AWS services (ALB, EBS, EFS, Secrets Manager)
- Your team is comfortable managing more operational complexity

### Choose AKS When

- You're in the Azure ecosystem (Azure AD, Azure Monitor, Azure DevOps)
- Cost is a priority (free control plane at standard tier)
- You want built-in auto-repair and simpler node management
- You need Azure AD integration for enterprise identity
- You're running Windows containers heavily

### Choose GKE When

- You want the most managed, least-operational-overhead experience
- Autopilot mode fits your workload (per-pod billing, no node management)
- You need advanced upgrade strategies (canary, blue-green per pool)
- You want built-in network policy (Dataplane V2/Cilium) without add-ons
- You're running ML/AI workloads with TPU support
- Cost observability is critical (built-in cost allocation)

## Gotchas Per Platform

### EKS

- Control plane costs $73/month per cluster even when idle
- VPC CNI IP exhaustion requires careful subnet planning (/22 or larger)
- Many features are add-ons (LB controller, network policy, CSI drivers)
- More operational burden — you assemble the platform from pieces

### AKS

- `kubenet` doesn't support network policies or Windows containers
- Azure CNI without overlay has the same IP exhaustion problem as EKS
- Cluster autoscaler is slower than Karpenter (no equivalent in AKS yet)
- Azure AD integration can be complex for multi-tenant setups

### GKE

- Autopilot restricts workloads (no privileged containers, limited hostPath)
- GKE Standard free tier doesn't include enterprise features (Binary Auth, multi-cluster)
- Zonal clusters have a 99.5% SLA (use Regional for production)
- Google's release channels can auto-upgrade at inconvenient times if not configured properly


## Storage

| Feature | EKS | AKS | GKE |
|---------|-----|-----|-----|
| Block storage | EBS (gp3, io2) | Azure Disk (Premium SSD, Ultra) | Persistent Disk (pd-ssd, pd-balanced) |
| File storage | EFS (NFS) | Azure Files (SMB/NFS) | Filestore (NFS) |
| CSI driver | EBS CSI, EFS CSI (add-ons) | Azure Disk CSI, Azure Files CSI (built-in) | GCE PD CSI (built-in) |
| Dynamic provisioning | Yes | Yes | Yes |
| Volume snapshots | Yes (EBS CSI) | Yes (Azure Disk CSI) | Yes (GCE PD CSI) |
| ReadWriteMany (RWX) | EFS only | Azure Files | Filestore |
| Volume expansion | Online (EBS) | Online (Azure Disk) | Online (PD) |
| Storage classes (default) | `gp2` (legacy), `gp3` (manual) | `managed-premium`, `managed-csi` | `standard`, `premium-rwo` |

### Default StorageClass Behavior

- **EKS**: No default StorageClass unless you install the EBS CSI driver and create one. You must set this up yourself.
- **AKS**: Comes with `managed-csi` (Premium SSD) and `managed-csi-premium` pre-configured.
- **GKE**: Comes with `standard` (pd-balanced) and `premium-rwo` (pd-ssd) pre-configured.

## Container Registry Integration

| Feature | EKS + ECR | AKS + ACR | GKE + Artifact Registry |
|---------|-----------|-----------|------------------------|
| Authentication | IAM-based (automatic on EC2/Fargate) | Managed identity or SP attachment | Workload Identity or node SA |
| Cross-account pull | ECR cross-account policy | ACR RBAC + private endpoint | Cross-project IAM binding |
| Image scanning | ECR basic + Inspector | Defender for Containers | Container Analysis / On-Demand Scanning |
| Registry cost | $0.10/GB/month | $0.667/day (Basic), $1.667/day (Standard) | $0.10/GB/month |
| Geo-replication | Cross-region replication | ACR geo-replication (Premium) | Multi-region (automatic) |
| Attach to cluster | Automatic (same account) | `az aks update --attach-acr` | Automatic (same project) |

## DNS Integration

| Feature | EKS | AKS | GKE |
|---------|-----|-----|-----|
| Cloud DNS service | Route 53 | Azure DNS | Cloud DNS |
| external-dns support | Yes (Route 53 add-on) | Yes (Azure DNS + Azure Private DNS) | Yes (Cloud DNS) |
| Internal DNS | CoreDNS → VPC DNS (169.254.169.253) | CoreDNS → Azure DNS (168.63.129.16) | CoreDNS → metadata DNS (169.254.169.254) |
| Private DNS zones | Route 53 Private Hosted Zones | Azure Private DNS Zones | Cloud DNS Private Zones |
| DNS for services | Via AWS LB Controller annotations | Automatic with Azure DNS integration | Automatic with GKE NEG |

## Cost Comparison: Typical 10-Node Cluster

Estimated monthly cost for a 10-node cluster (m5.large / Standard_D2s_v3 / e2-standard-2, 3 AZs):

| Component | EKS | AKS | GKE |
|-----------|----:|----:|----:|
| Control plane | $73 | $0 | $0 |
| Compute (10 nodes, on-demand) | ~$700 | ~$700 | ~$500 |
| Load balancer (1 ALB/NLB) | ~$25 | ~$20 | ~$20 |
| NAT Gateway (1 per AZ) | ~$100 | ~$40 | ~$45 |
| Persistent storage (500 GB gp3/SSD) | ~$40 | ~$55 | ~$40 |
| Data transfer (100 GB inter-AZ) | ~$10 | ~$10 | ~$10 |
| **Total** | **~$948** | **~$825** | **~$615** |

> Approximate pricing for us-east-1 / East US / us-central1. GKE benefits from sustained use discounts (automatic). Spot/preemptible can reduce compute by 60-70% on all three.

Key cost differences:
- **EKS** charges for the control plane and NAT Gateways are expensive with 3 AZs
- **AKS** free control plane saves $73/month, but Azure egress and disk pricing can add up
- **GKE** benefits from automatic sustained use discounts (SUD) and cheaper VMs at similar specs

## Portability and Migration

### What's Portable (Kubernetes-Native)

These work on any provider without changes:
- Deployments, StatefulSets, DaemonSets, Jobs
- Services (ClusterIP, NodePort)
- ConfigMaps, Secrets
- RBAC (Roles, ClusterRoles, Bindings)
- Network Policies (if using a portable CNI like Calico/Cilium)
- Helm charts (if no cloud-specific dependencies)
- Horizontal Pod Autoscaler

### What's NOT Portable (Vendor-Specific)

| Resource | EKS | AKS | GKE |
|----------|-----|-----|-----|
| LoadBalancer annotations | `service.beta.kubernetes.io/aws-*` | `service.beta.kubernetes.io/azure-*` | `cloud.google.com/neg`, `networking.gke.io/*` |
| Ingress class | `alb` (AWS LB Controller) | `azure/application-gateway` | `gce` or `gke-l7-*` |
| StorageClass provisioner | `ebs.csi.aws.com` | `disk.csi.azure.com` | `pd.csi.storage.gke.io` |
| IAM annotations | `eks.amazonaws.com/role-arn` | `azure.workload.identity/client-id` | `iam.gke.io/gcp-service-account` |
| Node labels (cloud) | `node.kubernetes.io/instance-type` | `node.kubernetes.io/instance-type` | `node.kubernetes.io/instance-type` (same!) |
| Spot/preemptible taints | `eks.amazonaws.com/capacityType=SPOT` | `kubernetes.azure.com/scalesetpriority=spot` | `cloud.google.com/gke-spot=true` |

### Migration Strategies

| Approach | Risk | Effort | Best For |
|----------|------|--------|----------|
| Lift and shift (re-deploy manifests) | Low (if K8s-native) | Low | Simple stateless apps |
| Blue-green migration (run both, shift traffic) | Low | Medium | Production workloads |
| Re-architect storage layer | Medium | High | Stateful workloads |
| Use abstraction layer (Crossplane, Terraform) | Low | Medium | Multi-cloud strategy |

Key migration challenges:
- Storage: PVs are provider-specific. Data must be migrated via backup/restore (Velero)
- IAM: Every provider has a different pod identity mechanism
- Networking: Load balancer annotations and Ingress configs are provider-locked
- DNS: External DNS entries need updating

## Compliance Certifications

| Certification | EKS | AKS | GKE |
|---------------|:---:|:---:|:---:|
| SOC 1/2/3 | Yes | Yes | Yes |
| ISO 27001 | Yes | Yes | Yes |
| PCI DSS | Yes | Yes | Yes |
| HIPAA | Yes (BAA) | Yes (BAA) | Yes (BAA) |
| FedRAMP | Yes (High) | Yes (High) | Yes (High) |
| GDPR | Yes | Yes | Yes |
| C5 (Germany) | Yes | Yes | Yes |

All three are certified for the most common compliance frameworks. The differences are usually in how you configure the cluster to meet specific controls, not whether the platform supports them.
