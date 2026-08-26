# EKS Auto Mode Security Deep Dive

Security architecture of Amazon EKS Auto Mode — covering the shared responsibility model, node hardening, IAM, networking isolation, encryption, and runtime monitoring.

## Shared Responsibility Model

With EKS Auto Mode, AWS takes responsibility for significantly more of the stack compared to standard EKS:

| Layer | AWS Responsibility | Customer Responsibility |
|-------|-------------------|------------------------|
| Control plane | API server, etcd, scheduler, controller-manager | — |
| Cluster capabilities | Compute autoscaling, pod networking, load balancing, storage drivers | — |
| EC2 instances | Lifecycle, OS, patching, health, monitoring | — |
| VPC & cluster config | — | VPC, subnets, security groups, cluster settings |
| Application containers | — | Availability, security, monitoring |

## EC2 Managed Instances

EKS Auto Mode nodes are EC2 managed instances with built-in IAM-enforced restrictions that block operations which could compromise AWS's ability to operate the nodes.

**Restrictions (applied regardless of IAM identity — even the AWS account root user cannot bypass):**

- Cannot change the instance profile
- Cannot attach or detach ENIs
- Cannot modify the instance at runtime
- Restrictions extend to EBS volumes attached at launch, ENIs, and launch templates

**What still works:**

- Capacity reservations and savings plans
- Full range of EC2 instance types (including accelerated types for ML inferencing/training)
- Standard EC2 billing and cost allocation

## Instance Configuration

### IMDS Lockdown

| Setting | Value | Security Benefit |
|---------|-------|-----------------|
| IMDS version | IMDSv2 only (token required) | Blocks SSRF-based credential theft |
| Hop limit | 1 | Blocks non-host-network pods from reaching IMDS |

Because the hop limit is 1, only processes running in the host network namespace can access the metadata service. Regular pods cannot reach IMDS to steal the node's IAM credentials.

### EBS Volume Encryption

Nodes launch with **two attached EBS volumes**, both encrypted by default:

| Volume | Purpose | Encrypted | Deleted on termination |
|--------|---------|-----------|----------------------|
| Root volume | Bottlerocket OS | Yes (AWS managed key) | Yes |
| Data volume | Pod logs, container images, ephemeral data | Yes (AWS managed key) | Yes |

**Custom encryption key:**

```yaml
apiVersion: eks.amazonaws.com/v1
kind: NodeClass
metadata:
  name: custom-encrypted
spec:
  ephemeralStorage:
    size: "80Gi"
    iops: 3000
    throughput: 125
    kmsKeyID: arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012
  role: my-node-role
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
```

**Persistent volumes (EBS CSI)** can also be configured to encrypt by default, including with a CMK.

## Node Operating System

The OS is a **custom variant of Bottlerocket** hardened beyond the standard distribution:

| Feature | Standard Bottlerocket | EKS Auto Mode Variant |
|---------|----------------------|----------------------|
| Host containers | Enabled | Disabled |
| SSH server | Available | Not available |
| SSM agent | Available | Not available |
| SELinux | Enforcing | Enforcing |
| Root filesystem | Read-only | Read-only |
| Cryptographic integrity checks | Yes | Yes |
| Accelerator drivers | Manual install | Automatically included for GPU/Neuron instances |

### SELinux MCS Labels

Most non-privileged pods automatically get a unique SELinux multi-category security (MCS) label. This protects against:

- A process in one pod manipulating a process in another pod or on the host
- Even if a pod runs as root and has access to the host filesystem, it cannot manipulate files, make sensitive system calls, or access the container runtime

## Node IAM — Minimal Permissions

### Node Role Policies

EKS Auto Mode introduces a new **minimal node IAM policy** that removes nine permissions from the previous `AmazonEKSWorkerNodePolicy`:

| Policy | Purpose | Notes |
|--------|---------|-------|
| `AmazonEKSWorkerNodeMinimalPolicy` | Core node operations | Only permission: `eks-auth:AssumeRoleForPodIdentity` |
| `AmazonEC2ContainerRegistryPullOnly` | Pull images from ECR | Fewer permissions than `AmazonEC2ContainerRegistryReadOnly` |

**Why fewer permissions are needed:**
- Auto Mode nodes use the EC2 instance ID as the Kubernetes node name
- Instance ID is reliably determined through IMDS → no need for `ec2:DescribeInstances`
- ENI management is handled by the AWS-managed networking capability → no VPC CNI permissions on the node

### Node Access Entry

| Field | Value |
|-------|-------|
| Access entry type | `EC2` |
| Kubernetes username | `system:node:{{SessionName}}` (resolves to instance ID) |
| Kubernetes group | `system:nodes` |
| Access policy | `AmazonEKSAutoNodePolicy` |
| Node name | EC2 instance ID (e.g. `i-0285feeceecfa12af`) |

```bash
# Verify the node access entry
aws eks list-access-entries --cluster-name my-cluster

# Describe a specific access entry
aws eks describe-access-entry --cluster-name my-cluster \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/my-node-role
```

## Node Component RBAC — Impersonation Model

Instead of granting all permissions to the kubelet's RBAC identity, Auto Mode uses **Kubernetes impersonation** for least-privilege:

1. Components start using the kubelet's identity
2. They impersonate a specific identity with only the permissions they need
3. After impersonation, they lose the kubelet's broader permissions

**Visible in audit logs:**

```json
{
  "impersonatedUser": {
    "username": "eks-auto:component-name",
    "groups": [
      "system:authenticated"
    ]
  }
}
```

This means you'll see `eks-auto:dns`, `eks-auto:node-monitoring`, etc. in audit logs — each with only the permissions that specific component requires.

## Networking Security

### Pod Networking Segregation

The NodeClass supports separating node and pod network traffic using different subnets and security groups:

**Default (shared):** Only `subnetSelectorTerms` and `securityGroupSelectorTerms` configured:
- Node IP and pod IPs come from the same ENIs
- Same security groups apply to node and pod traffic

**Segregated:** Also configure `podSubnetSelectorTerms` and `podSecurityGroupSelectorTerms`:
- Node IP comes from the primary ENI only
- Pod IPs come from secondary ENIs with separate security groups
- Allows different firewall rules for node vs pod traffic
- Trade-off: reduced pod density (primary ENI reserved for node IP only)

```yaml
apiVersion: eks.amazonaws.com/v1
kind: NodeClass
metadata:
  name: segregated-networking
spec:
  role: my-node-role
  # Node networking
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
        network-tier: node
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
        network-tier: node
  # Pod networking (separate subnets and security groups)
  podSubnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
        network-tier: pod
  podSecurityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
        network-tier: pod
```

### NetworkPolicy Enforcement

Pod-to-Pod traffic is controlled using standard Kubernetes NetworkPolicies, enforced by a networking component on the node using **eBPF**.

```bash
# Enable network policy on the cluster (if not already enabled)
# Network policy is a cluster-level setting

# Verify network policies
kubectl get networkpolicies -A

# Example: deny all ingress to a namespace
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
EOF
```

### Advanced Networking Options

| NodeClass Field | Purpose |
|-----------------|---------|
| `advancedNetworking.httpsProxy` | HTTPS_PROXY setting for containerd and kubelet |
| `advancedNetworking.noProxy` | NO_PROXY setting for containerd and kubelet |
| `certificateBundles` | Custom CA certificates (e.g. private registry with self-signed certs) |
| `advancedNetworking.associatePublicIPAddress` | Controls public IP on launch template (set to `false` if SCPs require it) |

## Encryption at Rest — Kubernetes API Data

EKS Auto Mode clusters get **default envelope encryption** for Kubernetes API data:

| What's encrypted | Method | Key |
|-----------------|--------|-----|
| ConfigMaps, Secrets, and other K8s API objects | Envelope encryption via KMS provider v2 | AWS owned key (default) or customer-managed key (CMK) |
| Root and data EBS volumes on nodes | EBS encryption | AWS managed key or CMK via `ephemeralStorage.kmsKeyID` |
| Persistent volumes (EBS CSI) | EBS encryption | Configurable per StorageClass |

**Not encrypted by envelope encryption:** Data on nodes or EBS volumes (those use standard EBS encryption).

```bash
# Use a customer-managed key for K8s API encryption
aws eks create-cluster \
  --name my-cluster \
  --encryption-config '[{
    "resources": ["secrets"],
    "provider": {"keyArn": "arn:aws:kms:us-east-1:123456789012:key/my-key-id"}
  }]' \
  ...
```

## Service Control Policies (SCPs)

SCPs apply to actions performed by the EKS Auto Mode cluster role. This means organizations can restrict Auto Mode capabilities:

- Limit which instance types can be launched
- Enforce EBS encryption requirements
- Block public IP assignment
- Restrict regions or AZs

**Key design decision:** Auto Mode uses a **service role** (not a service-linked role), so SCPs are respected. Auto Mode minimizes use of SLRs specifically so that organizational controls work.

```json
{
  "Effect": "Deny",
  "Action": "ec2:RunInstances",
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "ForAnyValue:StringNotLike": {
      "ec2:InstanceType": ["c6i.*", "m6i.*", "r6i.*"]
    }
  }
}
```

> See AWS docs: [Update organization controls for EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/auto-scp.html)

## Node Patching Process

### How nodes get updated:

1. AWS builds new Auto Mode AMI
2. AMI undergoes rigorous testing (CVE scanning, K8s conformance tests, accelerator compatibility)
3. AMI deploys to a small subset of clusters in one Region with bake time
4. Gradually rolls out to more clusters in larger waves across more Regions
5. A new AMI is made available **no more than once per week**
6. Nodes are replaced through **drift** — not patched in-place

### Node expiration and forced replacement:

| Setting | Default Value | Effect |
|---------|--------------|--------|
| NodePool `expireAfter` | 14 days (built-in pools) | Node becomes eligible for replacement |
| Maximum node lifetime | 21 days (hard limit) | Node is disrupted **regardless** of PDBs or disruption controls |
| `disruption.budgets[].schedule` | Not set | Restricts time windows for node replacement (optional) |

If PDBs or NodePool disruption controls prevent a node from being replaced before 21 days, the node is disrupted anyway. This ensures a misconfigured PDB cannot indefinitely block security patches.

## Runtime Monitoring

AWS recommends **Amazon GuardDuty** for runtime monitoring of Auto Mode nodes. Because nodes are Kubernetes conformant, third-party runtime monitoring solutions that work with standard K8s nodes also work with Auto Mode.

**GuardDuty detects:**
- Container breakouts
- Reverse shell creation
- Privilege escalation
- Cryptocurrency mining
- Anomalous network activity

```bash
# Enable GuardDuty EKS Runtime Monitoring (via AWS CLI)
aws guardduty update-detector \
  --detector-id <detector-id> \
  --features '[{"Name": "RUNTIME_MONITORING", "Status": "ENABLED",
    "AdditionalConfiguration": [{"Name": "EKS_ADDON_MANAGEMENT", "Status": "ENABLED"}]}]'
```

## Workload Security Best Practices

Even with Auto Mode's hardened foundation, follow these practices for application pods:

- **Use `securityContext`** — Set `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`
- **Use Pod Identity** — Preferred over IRSA for vending IAM credentials to pods
- **Use policy enforcement** — Deploy Kyverno or OPA Gatekeeper to limit pod-level configuration
- **Avoid `hostNetwork: true`** — Pods with host networking can bypass the IMDS hop limit
- **Set resource requests/limits** — Required for proper bin-packing and cost optimization
- **Use NetworkPolicies** — Restrict pod-to-pod communication to only what's needed

## Useful Links

- [Security Overview of Amazon EKS Auto Mode (PDF)](https://docs.aws.amazon.com/whitepapers/latest/security-overview-amazon-eks-auto-mode/security-overview-amazon-eks-auto-mode.html)
- [EKS Auto Mode User Guide](https://docs.aws.amazon.com/eks/latest/userguide/automode.html)
- [EKS Best Practices Guide — Security](https://docs.aws.amazon.com/eks/latest/best-practices/security.html)
- [Update Organization Controls for EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/auto-scp.html)
- [Shared Responsibility Model](https://aws.amazon.com/compliance/shared-responsibility-model/)
- [Security Best Practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [Amazon GuardDuty Runtime Monitoring](https://docs.aws.amazon.com/guardduty/latest/ug/runtime-monitoring.html)
- [Under the Hood: Amazon EKS Auto Mode](https://aws.amazon.com/blogs/containers/under-the-hood-amazon-eks-auto-mode/)
