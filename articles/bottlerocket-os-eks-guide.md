# Bottlerocket OS for EKS

Bottlerocket is a minimal, purpose-built Linux OS designed by AWS specifically for running containers. It strips away everything that isn't needed for container workloads — no shell by default, no package manager, immutable root filesystem, and atomic updates.

## What Makes Bottlerocket Different

```
┌────────────────────────────────────────────────────────────────────────┐
│  Traditional Linux (AL2, Ubuntu)    vs    Bottlerocket                 │
│                                                                        │
│  ✓ Full package manager (yum/apt)    ✗ No package manager              │
│  ✓ SSH access (by default)           ✗ No SSH (use SSM or API)         │
│  ✓ General-purpose shell             ✗ No shell on host                │
│  ✓ Mutable filesystem                ✗ Immutable root (dm-verity)      │
│  ✓ In-place package upda             ✗ Atomic image-based updates      │
│  ✓ Many running services             ✗ Minimal (only container runtime)│
│  ✓ Broad attack surface              ✗ Tiny attack surface             │
│                                                                        │
│  Bottlerocket has:                                                     │
│    containerd, kubelet, aws-iam-authenticator                          │
│  That's it. Nothing else.                                              │
└────────────────────────────────────────────────────────────────────────┘
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Bottlerocket Node                                          │
│                                                             │
│  ┌───────────────────────────────────────────────────┐      │
│  │  Partition A (active)    Partition B (standby)    │      │
│  │  ┌─────────────────┐    ┌─────────────────┐       │      │
│  │  │ Root filesystem │    │ Root filesystem │       │      │
│  │  │ (dm-verity,     │    │ (previous or    │       │      │
│  │  │  read-only)     │    │  next version)  │       │      │
│  │  └─────────────────┘    └─────────────────┘       │      │
│  └───────────────────────────────────────────────────┘      │
│                                                             │
│  ┌───────────────────────────────────────────────────┐      │
│  │  Data Partition (persistent, writable)            │      │
│  │  - Container images                               │      │
│  │  - containerd state                               │      │
│  │  - kubelet data                                   │      │
│  │  - Settings (TOML configuration)                  │      │
│  └───────────────────────────────────────────────────┘      │
│                                                             │
│  Running processes:                                         │
│    - containerd                                             │
│    - kubelet                                                │
│    - apiserver (Bottlerocket settings API, localhost only)  │
│    - storewolf (settings daemon)                            │
│    - host-ctr (admin container management)                  │
└─────────────────────────────────────────────────────────────┘
```

### Dual Root Partition (A/B Update)

Bottlerocket uses an A/B partition scheme for atomic, rollback-safe updates:

```
Before update:
  Partition A: v1.20.0 (active, booted)
  Partition B: v1.19.0 (previous, inactive)

Update applied:
  Partition A: v1.20.0 (still active)
  Partition B: v1.21.0 (new version written)

After reboot:
  Partition A: v1.20.0 (now inactive, rollback target)
  Partition B: v1.21.0 (now active, booted)

If v1.21.0 fails:
  Automatic rollback to Partition A (v1.20.0)
```

## Security Model

| Feature | How It Works |
|---------|-------------|
| Immutable root filesystem | dm-verity ensures integrity; tampered files cause boot failure |
| No shell | No `/bin/bash`, `/bin/sh` on the host. Removes entire class of exploits |
| No package manager | Can't install software. Nothing to `yum install` or `apt-get` |
| No SSH by default | Access via SSM or admin container only |
| SELinux enforcing | Mandatory access control enabled by default |
| Minimal CVE surface | Fewer packages = fewer vulnerabilities to patch |
| Signed updates | OS images are signed; can't install unsigned builds |
| Read-only `/etc` | Configuration changes only via the settings API |

### How to Access a Bottlerocket Node

Since there's no shell, you access nodes through:

**1. Admin Container (interactive debugging):**

```bash
# Enable the admin container via userdata:
[settings.host-containers.admin]
enabled = true

# Connect via SSM:
aws ssm start-session --target <instance-id>
# This drops you into the admin container (Alpine-based)
# From there, use `sheltie` to enter the host namespace:
sudo sheltie
# Now you're in the Bottlerocket host with a shell
```

**2. Control Container (automated management):**

```bash
# The control container runs by default
# Used by SSM for agent communication
# Gives SSM Session Manager access without enabling admin container
aws ssm start-session --target <instance-id>
```

**3. Bottlerocket API (programmatic):**

```bash
# From the admin container:
apiclient get settings
apiclient set settings.kubernetes.node-labels.role=worker
apiclient reboot
```

## Configuration via Settings API

Bottlerocket doesn't use files for configuration. Everything goes through a TOML-based settings API:

### Userdata Format (TOML)

```toml
# Bottlerocket userdata is TOML, not bash:
[settings.kubernetes]
cluster-name = "my-cluster"
api-server = "https://ABC123.gr7.us-east-1.eks.amazonaws.com"
cluster-certificate = "LS0tLS1CRUdJTi..."
cluster-dns-ip = "10.100.0.10"
max-pods = 110

[settings.kubernetes.node-labels]
role = "worker"
environment = "production"
team = "platform"

[settings.kubernetes.node-taints]
"dedicated" = "gpu:NoSchedule"

[settings.kubernetes.eviction-hard]
"memory.available" = "500Mi"
"nodefs.available" = "10%"

[settings.host-containers.admin]
enabled = true
```

### Managed Node Groups — Userdata

For EKS managed node groups with Bottlerocket AMI:

```toml
# EKS auto-populates cluster-name, api-server, cluster-certificate
# You only need custom settings:

[settings.kubernetes]
max-pods = 110

[settings.kubernetes.node-labels]
nodegroup = "general"
environment = "production"

[settings.host-containers.admin]
enabled = true

[settings.ntp]
time-servers = ["169.254.169.123"]
```

### Common Settings

| Setting | Path | Example |
|---------|------|---------|
| Max pods | `settings.kubernetes.max-pods` | `110` |
| Node labels | `settings.kubernetes.node-labels.<key>` | `role = "worker"` |
| Node taints | `settings.kubernetes.node-taints.<key>` | `"gpu" = "true:NoSchedule"` |
| Cluster DNS | `settings.kubernetes.cluster-dns-ip` | `"10.100.0.10"` |
| Admin container | `settings.host-containers.admin.enabled` | `true` |
| Kernel parameters | `settings.kernel.sysctl.<key>` | See below |
| Container registry mirrors | `settings.container-registry.mirrors.<registry>` | See below |

### Kernel Sysctl Settings

```toml
[settings.kernel.sysctl]
"net.core.somaxconn" = "32768"
"net.ipv4.tcp_max_syn_backlog" = "32768"
"vm.max_map_count" = "262144"
"fs.inotify.max_user_watches" = "524288"
```

### Container Registry Mirrors (For Air-Gapped Clusters)

```toml
[settings.container-registry.mirrors]
"docker.io" = ["https://my-mirror.example.com"]
"public.ecr.aws" = ["https://my-mirror.example.com"]
```

## Updates — Atomic and Automatic

### How Updates Work

```
┌────────────────────────────────────────────────────────────────┐
│  Bottlerocket Update Process                                   │
│                                                                │
│  1. Check for update (calls update API)                        │
│  2. Download new OS image to inactive partition                │
│  3. Verify image signature                                     │
│  4. Mark inactive partition as "next boot"                     │
│  5. Reboot (node drains via Kubernetes)                        │
│  6. Boot into new partition                                    │
│  7. If healthy → success                                       │
│     If unhealthy → rollback to previous partition              │
│                                                                │
│  No package-by-package patching. Entire OS replaced atomically.│
└────────────────────────────────────────────────────────────────┘
```

### Bottlerocket Update Operator (brupop)

For automated, orchestrated updates across a fleet:

```bash
# Install the Bottlerocket Update Operator:
kubectl apply -f https://raw.githubusercontent.com/bottlerocket-os/bottlerocket-update-operator/develop/deploy/bottlerocket-update-operator.yaml
```

brupop:
- Watches for available Bottlerocket updates
- Cordons and drains nodes one at a time
- Triggers the update + reboot
- Waits for the node to rejoin as Ready
- Respects PodDisruptionBudgets
- Rolls back if the node doesn't recover

### Manual Update (via API)

```bash
# From the admin container:
apiclient update check
apiclient update apply
apiclient reboot

# Or via SSM RunCommand:
aws ssm send-command --instance-ids <id> \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["apiclient update apply && apiclient reboot"]'
```

## Bottlerocket vs AL2 vs AL2023 for EKS

| Feature | Bottlerocket | Amazon Linux 2 | Amazon Linux 2023 |
|---------|:------------:|:--------------:|:-----------------:|
| Purpose | Container-only OS | General-purpose | General-purpose (newer) |
| Shell access | No (admin container) | Yes (SSH, SSM) | Yes (SSH, SSM) |
| Package manager | No | yum | dnf |
| Root filesystem | Immutable (dm-verity) | Mutable | Mutable |
| Update mechanism | Atomic image swap | yum update (package) | dnf update (package) |
| Rollback | Automatic (A/B partition) | Manual (re-image) | Manual (re-image) |
| SELinux | Enforcing (default) | Permissive (default) | Enforcing (configurable) |
| Configuration | TOML settings API | Files + bootstrap.sh | YAML + nodeadm |
| Boot time | ~25s | ~40-60s | ~35-50s |
| CVEs to patch | Very few (minimal packages) | Many (full OS) | Moderate |
| Custom software on node | Not possible (no pkg mgr) | Easy (yum install) | Easy (dnf install) |
| GPU/custom kernel drivers | Via bootstrap containers | Via userdata scripts | Via userdata scripts |
| EKS managed node groups | Supported | Supported | Supported |
| Karpenter | Supported | Supported | Supported |

### When to Use Bottlerocket

| Use Case | Recommendation |
|----------|---------------|
| High-security environments | Bottlerocket (minimal surface, SELinux, immutable) |
| Compliance requirements (CIS, NIST) | Bottlerocket (hardened by default) |
| Large fleet with automatic updates | Bottlerocket (brupop handles updates safely) |
| Need custom agents/tools on node | AL2/AL2023 (can't install on Bottlerocket) |
| GPU workloads with custom drivers | AL2/AL2023 (easier driver installation) |
| Development/debugging with SSH | AL2/AL2023 (has shell access) |
| Standard EKS workloads, security-conscious | Bottlerocket (best security posture) |

## EKS Setup with Bottlerocket

### Managed Node Group (eksctl)

```bash
eksctl create nodegroup \
  --cluster my-cluster \
  --name bottlerocket-ng \
  --node-ami-family Bottlerocket \
  --instance-types m5.large \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 5
```

### Managed Node Group (Terraform)

```hcl
resource "aws_eks_node_group" "bottlerocket" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "bottlerocket-workers"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.private_subnet_ids
  ami_type        = "BOTTLEROCKET_x86_64"   # or BOTTLEROCKET_ARM_64

  scaling_config {
    desired_size = 3
    max_size     = 5
    min_size     = 2
  }

  instance_types = ["m5.large"]

  launch_template {
    id      = aws_launch_template.bottlerocket.id
    version = aws_launch_template.bottlerocket.latest_version
  }
}

resource "aws_launch_template" "bottlerocket" {
  name_prefix = "bottlerocket-"

  user_data = base64encode(<<-TOML
    [settings.kubernetes]
    max-pods = 110

    [settings.kubernetes.node-labels]
    environment = "production"

    [settings.host-containers.admin]
    enabled = true
  TOML
  )

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 2
  }
}
```

### Karpenter with Bottlerocket

```yaml
apiVersion: karpenter.sh/v1
kind: EC2NodeClass
metadata:
  name: bottlerocket
spec:
  amiFamily: Bottlerocket
  role: KarpenterNodeRole
  subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: my-cluster
  userData: |
    [settings.kubernetes]
    max-pods = 110

    [settings.kubernetes.node-labels]
    managed-by = "karpenter"

    [settings.host-containers.admin]
    enabled = true
```

## AMI Selection

```bash
# Find the latest Bottlerocket AMI for EKS:
aws ssm get-parameter \
  --name /aws/service/bottlerocket/aws-k8s-1.30/x86_64/latest/image_id \
  --query "Parameter.Value" --output text

# ARM64:
aws ssm get-parameter \
  --name /aws/service/bottlerocket/aws-k8s-1.30/arm64/latest/image_id \
  --query "Parameter.Value" --output text

# GPU (NVIDIA):
aws ssm get-parameter \
  --name /aws/service/bottlerocket/aws-k8s-1.30-nvidia/x86_64/latest/image_id \
  --query "Parameter.Value" --output text

# List all available variants:
aws ssm get-parameters-by-path \
  --path /aws/service/bottlerocket/ \
  --query "Parameters[].Name" --output table
```

### AMI Naming Pattern

```
/aws/service/bottlerocket/aws-k8s-<k8s-version>/<arch>/latest/image_id
/aws/service/bottlerocket/aws-k8s-<k8s-version>-nvidia/<arch>/latest/image_id
```

## Debugging Bottlerocket Nodes

### Access the Node

```bash
# Via SSM (control container is always running):
aws ssm start-session --target <instance-id>

# Once inside the control container:
# Enter the host namespace for full access:
sudo sheltie

# Now you have a privileged shell in the Bottlerocket host
```

### Common Debug Commands (Inside sheltie)

```bash
# Check kubelet status:
systemctl status kubelet

# View kubelet logs:
journalctl -u kubelet --since "5 minutes ago"

# Check containerd:
systemctl status containerd
ctr -n k8s.io containers list

# View Bottlerocket settings:
cat /proc/1/environ | tr '\0' '\n' | grep KUBERNETES

# Check current OS version:
cat /etc/os-release

# Check update status:
apiclient update check

# View network configuration:
ip addr
ip route

# Check disk usage:
df -h
```

### From Outside (kubectl debug)

```bash
# Debug pod on a Bottlerocket node:
kubectl debug node/<node-name> -it --image=ubuntu --profile=sysadmin

# Inside:
chroot /host
journalctl -u kubelet --since "5 min ago"
```

### Troubleshooting Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Node not joining cluster | Wrong cluster settings in userdata | Check TOML syntax in userdata |
| "admin container disabled" | Admin container not enabled | Add `[settings.host-containers.admin] enabled = true` |
| Can't pull images | No ECR access or registry mirror misconfigured | Check IAM role has ECR permissions |
| Node shows Wrong labels | TOML syntax error in node-labels | Validate TOML (quotes around values with special chars) |
| Update fails | Insufficient disk space on data partition | Check `/var/lib/bottlerocket/` usage |
| Rollback loop | New version consistently fails to boot | Pin to a specific known-good AMI version |

## Limitations

| Limitation | Workaround |
|-----------|-----------|
| Can't install packages on host | Use DaemonSet pods for monitoring agents |
| No SSH by default | Use SSM Session Manager or admin container |
| Custom kernel modules difficult | Use bootstrap containers (limited) |
| No init scripts (no cloud-init) | All config via TOML settings |
| Can't modify `/etc/` files | Use settings API for supported config |
| Limited GPU driver flexibility | Use the nvidia variant AMI |

## Quick Reference

```bash
# AMI selection:
aws ssm get-parameter --name /aws/service/bottlerocket/aws-k8s-1.30/x86_64/latest/image_id

# Userdata format: TOML (not bash)
# [settings.kubernetes]
# cluster-name = "my-cluster"
# max-pods = 110

# Access a Bottlerocket node:
aws ssm start-session --target <instance-id>
# Then: sudo sheltie (for host access)

# Update:
apiclient update check
apiclient update apply && apiclient reboot

# Automated updates: Bottlerocket Update Operator (brupop)

# Key properties:
# - Immutable root (dm-verity)
# - No shell, no package manager
# - Atomic A/B updates with automatic rollback
# - SELinux enforcing by default
# - Settings via TOML API (not config files)
# - Boot time: ~25s

# EKS ami_type values:
# BOTTLEROCKET_x86_64
# BOTTLEROCKET_ARM_64

# Karpenter amiFamily: Bottlerocket
```
