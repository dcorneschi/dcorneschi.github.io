# EKS AMI Comparison: Ubuntu vs Ubuntu Pro vs Amazon Linux

Comparing the available AMI options for EKS worker nodes — Ubuntu EKS, Ubuntu Pro EKS, Amazon Linux 2023, and Bottlerocket.

## AMI Overview

| AMI | Base OS | Maintained By | Cost | Use Case |
|-----|---------|---------------|------|----------|
| Amazon Linux 2023 (AL2023) | Amazon Linux | AWS | Free (included in EC2) | Default, general workloads |
| Bottlerocket | Purpose-built container OS | AWS | Free (included in EC2) | Minimal, security-focused |
| Ubuntu EKS | Ubuntu Minimal LTS | Canonical | Free | Ubuntu ecosystem, custom tooling |
| Ubuntu Pro EKS | Ubuntu Minimal LTS + Pro | Canonical | Per-hour premium via AWS Marketplace | Compliance, extended security, enterprise |

## Ubuntu EKS AMI (Standard)

The standard Ubuntu EKS AMI is a free, optimized image for running EKS worker nodes.

### What's Included

- Ubuntu Minimal LTS base (22.04 or 24.04)
- AWS-optimized kernel (jointly developed with AWS)
- EKS binaries: kubelet, kubectl, containerd
- AWS IAM Authenticator
- Optimized for performance on EC2

### Supported Versions

| Ubuntu LTS | EKS Versions |
|------------|-------------|
| 22.04 | 1.25 – 1.34 |
| 24.04 | 1.31+ |

### Deploy with eksctl

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-east-1
  version: '1.31'

nodeGroups:
  - name: ng-ubuntu
    instanceType: m5.large
    desiredCapacity: 3
    amiFamily: Ubuntu2404
    ssh:
      allow: true
      publicKeyName: myKeyPair
```

```bash
eksctl create cluster -f config.yaml
```

### When to Use

- Standard workloads without compliance requirements
- Teams familiar with Ubuntu/Debian ecosystem
- Need access to Ubuntu package repositories (apt)
- Custom kernel modules or drivers that work better on Ubuntu
- No additional cost beyond EC2 pricing

## Ubuntu Pro EKS AMI

Ubuntu Pro EKS adds enterprise security and compliance features on top of the standard Ubuntu EKS AMI.

### What's Included (Everything in Standard Plus)

- Kernel Livepatch — security patches without rebooting nodes
- Expanded Security Maintenance (ESM) — patches for 23,000+ packages beyond main (including Apache, Kafka, MySQL, PostgreSQL, MongoDB, Nginx, Node.js, and more)
- EKS extended support coverage — aligned with the full EKS lifecycle
- Pro container license — run unlimited Pro containers with ESM coverage
- FIPS 140-2/140-3 validated cryptographic modules (22.04 only, currently)
- CIS hardening tools and compliance profiles
- USG (Ubuntu Security Guide) for automated hardening

### Supported Versions

| Ubuntu Pro LTS | EKS Versions | FIPS Available |
|---------------|-------------|----------------|
| Pro 22.04 | 1.29 – 1.34 | Yes (NIST-validated) |
| Pro 24.04 | 1.31+ | Not yet |

### Deploy with eksctl

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-pro-cluster
  region: us-east-1
  version: '1.35'

iam:
  withOIDC: true

nodeGroups:
  - name: ng-ubuntu-pro
    instanceType: m5.large
    desiredCapacity: 3
    amiFamily: UbuntuPro2404
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
    ssh:
      allow: true
      publicKeyName: myKeyPair
```

```bash
eksctl create cluster -f config.yaml
```

### Verify Pro Subscription

```bash
# Check running instances show "Ubuntu Pro Linux" as platform
aws ec2 describe-instances \
    --region us-east-1 \
    --filters Name=instance-state-name,Values=running \
    --query 'Reservations[*].Instances[*].[InstanceType, LaunchTime, PlatformDetails]' \
    --output table
```

### Pricing

- Billed per-hour on top of EC2 instance cost
- Available via AWS Marketplace (metered billing)
- Alternatively, attach an existing Ubuntu Pro token from Canonical

### When to Use

- Regulated industries (finance, healthcare, government)
- FIPS 140-2/140-3 compliance requirements (FedRAMP, HIPAA)
- Need kernel livepatching to avoid reboot windows
- Running containers that need ESM security patches
- EKS extended support period alignment
- CIS benchmark compliance required

## Amazon Linux 2023 (AL2023) EKS AMI

The default and most common EKS AMI, maintained by AWS.

### What's Included

- Amazon Linux 2023 base
- AWS-optimized kernel
- EKS binaries: kubelet, containerd, AWS IAM Authenticator
- SELinux in permissive mode
- IMDSv2-only by default
- Deterministic upgrades via versioned repos

### Deploy with eksctl

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-east-1
  version: '1.31'

managedNodeGroups:
  - name: ng-al2023
    instanceType: m5.large
    desiredCapacity: 3
    amiFamily: AmazonLinux2023
```

### When to Use

- Default choice for most EKS clusters
- Tightest integration with AWS services
- Fastest AMI updates after new EKS versions
- No additional OS licensing cost
- Teams comfortable with RPM-based distributions (dnf/yum)

## Bottlerocket EKS AMI

A minimal, purpose-built container OS from AWS.

### What's Included

- Immutable root filesystem (read-only)
- No shell or package manager by default
- Automatic updates via update API
- Minimal attack surface
- SELinux enforcing mode
- dm-verity for boot integrity

### Deploy with eksctl

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-east-1
  version: '1.31'

managedNodeGroups:
  - name: ng-bottlerocket
    instanceType: m5.large
    desiredCapacity: 3
    amiFamily: Bottlerocket
```

### When to Use

- Maximum security posture (smallest attack surface)
- Don't need SSH access or custom packages on nodes
- Want automatic, API-driven OS updates
- Container-only workloads with no host-level customization

## Feature Comparison

| Feature | AL2023 | Bottlerocket | Ubuntu EKS | Ubuntu Pro EKS |
|---------|--------|-------------|-----------|---------------|
| **Cost** | Free | Free | Free | Per-hour premium |
| **Package Manager** | dnf | None (immutable) | apt | apt |
| **Shell Access** | Yes | Admin container only | Yes | Yes |
| **Kernel Livepatch** | No | N/A (auto-update) | No | Yes |
| **ESM (extended patches)** | No | N/A | No | Yes (23,000+ packages) |
| **FIPS 140-2/3** | No | No | No | Yes (22.04) |
| **CIS Hardening** | Manual | Built-in (minimal) | Manual | Automated (USG) |
| **SELinux** | Permissive | Enforcing | Not default | Not default |
| **AppArmor** | No | No | Yes | Yes |
| **EKS Extended Support** | Yes | Yes | Standard only | Yes |
| **IMDSv2 Default** | Yes | Yes | No | No |
| **Custom Node Software** | Easy | Difficult | Easy | Easy |
| **Node Group Support** | Managed + Self-managed | Managed + Self-managed | Self-managed | Self-managed |
| **Pro Containers** | N/A | N/A | N/A | Unlimited |
| **Auto OS Updates** | No (managed via AMI) | Yes (API-driven) | No (managed via AMI) | No (managed via AMI) |

## Security Comparison

| Security Feature | AL2023 | Bottlerocket | Ubuntu EKS | Ubuntu Pro EKS |
|-----------------|--------|-------------|-----------|---------------|
| Standard support | 5 years | Ongoing | 5 years (LTS) | 10 years (ESM) |
| CVE patching (main) | Yes | Yes | Yes | Yes |
| CVE patching (universe) | No | N/A | No | Yes |
| Rebootless patching | No | No | No | Yes (Livepatch) |
| FIPS crypto modules | No | No | No | Yes |
| Compliance profiles | Limited | Minimal | Manual | CIS, DISA-STIG |
| Read-only root | No | Yes | No | No |

## AMI Lookup Commands

### Amazon Linux 2023

```bash
aws ssm get-parameter \
    --name /aws/service/eks/optimized-ami/1.31/amazon-linux-2023/x86_64/standard/recommended/image_id \
    --query 'Parameter.Value' --output text
```

### Bottlerocket

```bash
aws ssm get-parameter \
    --name /aws/service/bottlerocket/aws-k8s-1.31/x86_64/latest/image_id \
    --query 'Parameter.Value' --output text
```

### Ubuntu EKS

```bash
aws ssm get-parameter \
    --name /aws/service/canonical/ubuntu/eks/24.04/1.31/stable/current/amd64/hvm/ebs-gp3/ami-id \
    --query 'Parameter.Value' --output text
```

### Ubuntu Pro EKS

```bash
aws ssm get-parameter \
    --name /aws/service/canonical/ubuntu/eks-pro/24.04/1.31/stable/current/amd64/hvm/ebs-gp3/ami-id \
    --query 'Parameter.Value' --output text
```

## Migration Considerations

### From Amazon Linux 2 (End of Support: November 2025)

AL2 EKS AMIs are no longer receiving new Kubernetes versions. Migration options:

```bash
# Check current node AMI
kubectl get nodes -o wide

# Options:
# 1. Amazon Linux 2023 (most similar, RPM-based)
# 2. Bottlerocket (if minimal/security-focused)
# 3. Ubuntu EKS (if moving to Debian ecosystem)
# 4. Ubuntu Pro EKS (if compliance needed)
```

### From Ubuntu EKS to Ubuntu Pro EKS

```bash
# Change amiFamily in eksctl config
# From:
#   amiFamily: Ubuntu2404
# To:
#   amiFamily: UbuntuPro2404

# Rolling update with new node group
eksctl create nodegroup -f config-pro.yaml
eksctl delete nodegroup -f config-old.yaml --approve
```

## Decision Guide

### Choose Amazon Linux 2023 if:

- You want the default, best-supported EKS experience
- No specific compliance requirements
- Comfortable with RPM/dnf package management
- Want fastest access to new EKS versions

### Choose Bottlerocket if:

- Security is the top priority
- No need for SSH or host-level customization
- Want automatic, API-driven updates
- Running purely containerized workloads

### Choose Ubuntu EKS if:

- Team expertise is in Ubuntu/Debian ecosystem
- Need specific Ubuntu packages or PPAs
- Running workloads that require Ubuntu compatibility
- AppArmor preferred over SELinux

### Choose Ubuntu Pro EKS if:

- Regulatory compliance required (FIPS, CIS, FedRAMP, HIPAA)
- Need kernel livepatching (zero-downtime security updates)
- Running containers that need extended security patches
- Want aligned EKS extended support coverage
- Enterprise support contract with Canonical

## See Also

- [EKS Architecture Deep Dive](articles/eks-architecture-deep-dive.md) — EKS internals and control plane
- [EKS Node Groups: With and Without Launch Templates](articles/eks-nodegroups-launch-templates.md) — Node group configuration
- [eksctl Cheatsheet](articles/eksctl-cheatsheet.md) — eksctl CLI commands
- [EKS Auto Mode](articles/eks-auto-mode.md) — Fully managed data plane
