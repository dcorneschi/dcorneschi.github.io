# EKS Node Bootstrap Deep Dive — bootstrap.sh, nodeadm, AL2 vs AL2023

How EKS worker nodes join the cluster — the bootstrap process from EC2 launch to node Ready, bootstrap.sh parameters (AL2), nodeadm (AL2023), custom userdata patterns, and troubleshooting join failures.

Note: For validating cloud-init completion and checking bootstrap logs, see the cloud-init validation article. This article covers the bootstrap tools themselves, their parameters, and the differences between AMI generations.

## Bootstrap Overview

```
EC2 Instance Launches
    │
    ▼
cloud-init runs userdata
    │
    ├── AL2 AMI: calls /etc/eks/bootstrap.sh
    │
    └── AL2023 AMI: calls nodeadm (via cloud-init nodeConfig)
    │
    ▼
kubelet configured and started
    │
    ▼
kubelet registers with API server
    │
    ▼
Node appears as Ready (after CNI setup)
```

## AL2 vs AL2023 — Key Differences

| Feature | Amazon Linux 2 (AL2) | Amazon Linux 2023 (AL2023) |
|---------|:--------------------:|:--------------------------:|
| Bootstrap tool | `/etc/eks/bootstrap.sh` (bash script) | `nodeadm` (Go binary) |
| Configuration format | CLI flags in userdata | YAML (NodeConfig document) |
| Init system | systemd | systemd |
| Container runtime | containerd | containerd |
| Kernel | 5.10 | 6.1 |
| Default kubelet config | Via bootstrap.sh flags | Via NodeConfig YAML |
| Prefix delegation support | Via bootstrap.sh flag | Built-in |
| Support timeline | Maintenance mode (no new features) | Actively developed |
| EKS versions supported | All current | 1.28+ |

## AL2: bootstrap.sh

The EKS-optimized AL2 AMI includes `/etc/eks/bootstrap.sh`, a bash script that configures kubelet and joins the cluster.

### Minimal Userdata (Managed Node Groups)

For managed node groups, EKS injects bootstrap arguments automatically. Your userdata only needs custom additions:

```bash
#!/bin/bash
# Managed node groups: EKS prepends bootstrap call automatically
# Your userdata runs BEFORE the EKS-injected bootstrap
# Use this for pre-bootstrap customization only

# Example: install monitoring agent before bootstrap
yum install -y amazon-cloudwatch-agent
```

### Explicit Bootstrap (Self-Managed / Launch Template)

```bash
#!/bin/bash
set -o xtrace

/etc/eks/bootstrap.sh my-cluster-name \
  --kubelet-extra-args '--node-labels=role=worker,environment=production --max-pods=110'
```

### bootstrap.sh Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `$1` (positional) | (required) | Cluster name |
| `--b64-cluster-ca` | Auto-detected | Base64-encoded cluster CA certificate |
| `--apiserver-endpoint` | Auto-detected | API server endpoint URL |
| `--kubelet-extra-args` | None | Extra flags passed to kubelet |
| `--container-runtime` | `containerd` | Container runtime (only containerd supported now) |
| `--use-max-pods` | true | Use ENI-based max-pods calculation |
| `--ip-family` | `ipv4` | IP family (`ipv4` or `ipv6`) |
| `--service-ipv4-cidr` | Auto-detected | Service CIDR (for kube-proxy config) |
| `--dns-cluster-ip` | Auto-derived | CoreDNS ClusterIP (derived from service CIDR) |
| `--enable-local-outpost` | false | For EKS on Outposts |

```bash
# Full example with common options:
/etc/eks/bootstrap.sh my-cluster \
  --b64-cluster-ca "$B64_CA" \
  --apiserver-endpoint "https://ABC123.gr7.us-east-1.eks.amazonaws.com" \
  --kubelet-extra-args '--node-labels=nodegroup=gpu,instance-type=p3.2xlarge --register-with-taints=nvidia.com/gpu=present:NoSchedule --max-pods=110' \
  --use-max-pods false
```

### What bootstrap.sh Does Internally

```
┌─────────────────────────────────────────────────────────────────────┐
│  /etc/eks/bootstrap.sh execution flow:                              │
│                                                                     │
│  1. Parse arguments                                                 │
│  2. Retrieve cluster info (if not provided):                        │
│     - Calls IMDS for region                                         │
│     - Calls EKS DescribeCluster API for CA + endpoint               │
│  3. Write /etc/kubernetes/kubelet/kubelet-config.json               │
│  4. Write /var/lib/kubelet/kubeconfig                               │
│  5. Write /etc/systemd/system/kubelet.service.d/10-kubelet-args.conf│
│  6. Calculate max-pods (based on ENI count + IPs per ENI)           │
│  7. Configure containerd (/etc/containerd/config.toml)              │
│  8. Enable and start kubelet.service                                │
└─────────────────────────────────────────────────────────────────────┘
```

### Common kubelet-extra-args

```bash
# Node labels:
--kubelet-extra-args '--node-labels=role=worker,team=platform'

# Taints:
--kubelet-extra-args '--register-with-taints=dedicated=gpu:NoSchedule'

# Max pods (override ENI calculation):
--kubelet-extra-args '--max-pods=110'

# Reserve resources for system:
--kubelet-extra-args '--system-reserved=cpu=100m,memory=256Mi --kube-reserved=cpu=100m,memory=256Mi'

# Eviction thresholds:
--kubelet-extra-args '--eviction-hard=memory.available<500Mi,nodefs.available<10%'

# Combined:
--kubelet-extra-args '--node-labels=role=worker --register-with-taints=spot=true:PreferNoSchedule --max-pods=110 --system-reserved=cpu=200m,memory=512Mi'
```

### Private Cluster Bootstrap (No Internet)

For private clusters (no NAT Gateway), the node can't reach the EKS API to discover cluster info. You must provide it explicitly:

```bash
#!/bin/bash
/etc/eks/bootstrap.sh my-cluster \
  --b64-cluster-ca "LS0tLS1CRUdJTi..." \
  --apiserver-endpoint "https://ABC123.gr7.us-east-1.eks.amazonaws.com" \
  --dns-cluster-ip "10.100.0.10" \
  --kubelet-extra-args '--node-labels=role=worker'
```

## AL2023: nodeadm

AL2023 EKS AMIs use `nodeadm`, a Go binary that replaces bootstrap.sh with a structured YAML configuration.

### NodeConfig Format

```yaml
# Passed as userdata (MIME multipart or plain YAML):
---
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    name: my-cluster
    apiServerEndpoint: https://ABC123.gr7.us-east-1.eks.amazonaws.com
    certificateAuthority: LS0tLS1CRUdJTi...
    cidr: 10.100.0.0/16
  kubelet:
    config:
      maxPods: 110
      clusterDNS:
      - 10.100.0.10
    flags:
    - --node-labels=role=worker,environment=production
    - --register-with-taints=dedicated=gpu:NoSchedule
  containerd:
    config: |
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
        runtime_type = "io.containerd.runtime.v1.linux"
```

### Managed Node Groups (AL2023)

For managed node groups on AL2023, EKS handles the NodeConfig automatically. Your userdata extends it using MIME multipart:

```bash
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="BOUNDARY"

--BOUNDARY
Content-Type: application/node.eks.aws

---
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  kubelet:
    config:
      maxPods: 110
      shutdownGracePeriod: 30s
      shutdownGracePeriodCriticalPods: 10s
    flags:
    - --node-labels=role=worker

--BOUNDARY--
```

### nodeadm vs bootstrap.sh Mapping

| bootstrap.sh | nodeadm (NodeConfig) |
|-------------|---------------------|
| `--kubelet-extra-args '--max-pods=110'` | `spec.kubelet.config.maxPods: 110` |
| `--kubelet-extra-args '--node-labels=a=b'` | `spec.kubelet.flags: [--node-labels=a=b]` |
| `--b64-cluster-ca` | `spec.cluster.certificateAuthority` |
| `--apiserver-endpoint` | `spec.cluster.apiServerEndpoint` |
| `--dns-cluster-ip` | `spec.kubelet.config.clusterDNS: [ip]` |
| `--use-max-pods false` | `spec.kubelet.config.maxPods: <value>` |
| `--container-runtime` | Always containerd (no flag) |
| Custom containerd config | `spec.containerd.config` |

### nodeadm Execution Flow

```
┌────────────────────────────────────────────────────────────────────┐
│  nodeadm execution flow:                                           │
│                                                                    │
│  1. Read NodeConfig from:                                          │
│     - IMDS userdata (cloud-init delivers it)                       │
│     - Or /etc/eks/nodeadm/nodeConfig.yaml                          │
│                                                                    │
│  2. Validate NodeConfig schema                                     │
│                                                                    │
│  3. If cluster info not provided:                                  │
│     - Query IMDS for region + instance identity                    │
│     - Call EKS DescribeCluster API                                 │
│                                                                    │
│  4. Configure kubelet:                                             │
│     - Write /etc/kubernetes/kubelet/config.json                    │
│     - Write /var/lib/kubelet/kubeconfig                            │
│     - Apply kubelet flags + config                                 │
│                                                                    │
│  5. Configure containerd:                                          │
│     - Write /etc/containerd/config.toml                            │
│     - Apply any custom containerd config                           │
│                                                                    │
│  6. Configure VPC CNI / networking                                 │
│                                                                    │
│  7. Start kubelet.service                                          │
│                                                                    │
│  8. Run credential provider setup (ECR auth)                       │
└────────────────────────────────────────────────────────────────────┘
```

### nodeadm Commands

```bash
# On the node, check nodeadm status:
nodeadm status

# View effective configuration:
nodeadm config view

# Validate a NodeConfig:
nodeadm validate --config /path/to/nodeConfig.yaml
```

## Managed Node Group Bootstrap Behavior

### What EKS Injects Automatically

When using managed node groups (no custom launch template), EKS generates bootstrap configuration:

```
EKS-generated for AL2:
  /etc/eks/bootstrap.sh <cluster-name> --b64-cluster-ca <CA> --apiserver-endpoint <URL>

EKS-generated for AL2023:
  NodeConfig with cluster info pre-populated

Your userdata (if any):
  AL2: runs BEFORE the EKS-injected bootstrap call
  AL2023: merged with EKS-generated NodeConfig (MIME multipart)
```

### Launch Template Override

When you provide a custom launch template with userdata:
- **AL2**: EKS does NOT inject its bootstrap call. You MUST call bootstrap.sh yourself.
- **AL2023**: EKS merges your NodeConfig with its own (your values take precedence for conflicts).

```bash
# AL2 with custom launch template — YOU must bootstrap:
#!/bin/bash
# Your custom setup first
yum install -y my-custom-package

# Then bootstrap (required!):
/etc/eks/bootstrap.sh my-cluster \
  --kubelet-extra-args '--node-labels=custom=true'
```

## Common Bootstrap Customizations

### Enable Prefix Delegation (More Pods per Node)

**AL2:**
```bash
/etc/eks/bootstrap.sh my-cluster \
  --use-max-pods false \
  --kubelet-extra-args '--max-pods=110'
```

**AL2023:**
```yaml
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  kubelet:
    config:
      maxPods: 110
```

(Also requires VPC CNI to have `ENABLE_PREFIX_DELEGATION=true`)

### GPU Node Configuration

**AL2:**
```bash
#!/bin/bash
/etc/eks/bootstrap.sh my-cluster \
  --kubelet-extra-args '--node-labels=nvidia.com/gpu.present=true --register-with-taints=nvidia.com/gpu=present:NoSchedule'
```

**AL2023:**
```yaml
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  kubelet:
    flags:
    - --node-labels=nvidia.com/gpu.present=true
    - --register-with-taints=nvidia.com/gpu=present:NoSchedule
```

### Custom Kubelet Configuration

**AL2023 (structured kubelet config):**
```yaml
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  kubelet:
    config:
      maxPods: 58
      shutdownGracePeriod: 60s
      shutdownGracePeriodCriticalPods: 20s
      imageGCHighThresholdPercent: 85
      imageGCLowThresholdPercent: 80
      evictionHard:
        memory.available: "500Mi"
        nodefs.available: "10%"
        imagefs.available: "15%"
      evictionSoft:
        memory.available: "750Mi"
      evictionSoftGracePeriod:
        memory.available: "1m30s"
      systemReserved:
        cpu: "100m"
        memory: "256Mi"
      kubeReserved:
        cpu: "100m"
        memory: "512Mi"
```

### IMDSv2 Enforcement

```bash
# In launch template (Terraform):
metadata_options {
  http_endpoint               = "enabled"
  http_tokens                 = "required"   # Enforce IMDSv2
  http_put_response_hop_limit = 2            # 2 for containers to reach IMDS
}
```

The hop limit must be 2 (not 1) for pods to access IMDS through the VPC CNI's veth pair.

## Node Join Sequence (What Happens After Bootstrap)

```
┌────────────────────────────────────────────────────────────────────┐
│  After kubelet starts:                                             │
│                                                                    │
│  1. kubelet sends registration request to API server               │
│     (includes node labels, taints, capacity)                       │
│                                                                    │
│  2. API server creates Node object (status: NotReady)              │
│                                                                    │
│  3. aws-node (VPC CNI) DaemonSet pod starts on the node            │
│     - Attaches ENIs                                                │
│     - Allocates secondary IPs to warm pool                         │
│     - Sets up routing                                              │
│                                                                    │
│  4. kube-proxy DaemonSet pod starts                                │
│     - Programs iptables/IPVS rules                                 │
│                                                                    │
│  5. Node reports Ready condition                                   │
│     (kubelet confirms runtime + network are functional)            │
│                                                                    │
│  6. Node is schedulable (unless tainted)                           │
│                                                                    │
│  Typical time from EC2 launch to Ready: 60-120 seconds             │
└────────────────────────────────────────────────────────────────────┘
```

## Troubleshooting Bootstrap Failures

### Node Never Joins Cluster

```bash
# On the node (via SSM):

# 1. Check cloud-init completed:
cloud-init status
# If "error" → check /var/log/cloud-init-output.log

# 2. Check kubelet is running:
systemctl status kubelet

# 3. Check kubelet logs:
journalctl -u kubelet --since "5 minutes ago" | tail -50

# 4. Check kubelet can reach API server:
curl -k https://<api-server-endpoint>/healthz
# If timeout → security group, route table, or DNS issue

# 5. Check node can resolve DNS:
nslookup <api-server-endpoint>

# 6. Check IAM instance profile:
curl -s http://169.254.169.254/latest/meta-data/iam/info
# Node role needs: AmazonEKSWorkerNodePolicy, AmazonEKS_CNI_Policy, AmazonEC2ContainerRegistryReadOnly

# 7. Check bootstrap.sh output:
cat /var/log/cloud-init-output.log | grep -A 20 "bootstrap.sh"

# 8. AL2023 — check nodeadm:
nodeadm status
journalctl -u nodeadm
```

### Common Failure Causes

| Symptom | Cause | Fix |
|---------|-------|-----|
| kubelet can't reach API server | Security group missing rule (port 443 outbound) | Add outbound 443 to cluster SG or endpoint |
| "Unauthorized" in kubelet logs | Node IAM role not in aws-auth ConfigMap or Access Entries | Add role mapping |
| Node joins but stays NotReady | VPC CNI can't attach ENIs (subnet full, limits reached) | Check subnet IPs, ENI limits |
| cloud-init timeout | Userdata script hangs (blocking command) | Check for interactive prompts, add timeouts |
| "certificate signed by unknown authority" | Wrong CA certificate in bootstrap args | Verify `--b64-cluster-ca` value |
| kubelet crashloops | Max pods too high for instance type | Lower `--max-pods` or enable prefix delegation |
| Node joins then disappears | Instance terminated by ASG (health check failure) | Check EC2 health checks, extend grace period |

### Key Log Files

| File | Contents |
|------|----------|
| `/var/log/cloud-init.log` | Cloud-init execution (all stages) |
| `/var/log/cloud-init-output.log` | Userdata script stdout/stderr |
| `/var/log/messages` or `journalctl -u kubelet` | Kubelet logs |
| `/etc/kubernetes/kubelet/kubelet-config.json` | Generated kubelet config |
| `/var/lib/kubelet/kubeconfig` | Kubelet's kubeconfig (cluster endpoint + CA) |
| `/var/log/aws-routed-eni/ipamd.log` | VPC CNI logs |

## Quick Reference

```bash
# AL2 bootstrap (self-managed):
/etc/eks/bootstrap.sh <cluster-name> \
  --kubelet-extra-args '--node-labels=role=x --max-pods=110'

# AL2023 nodeadm (via userdata):
---
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    name: my-cluster
  kubelet:
    config:
      maxPods: 110
    flags:
    - --node-labels=role=x

# Key differences:
# AL2: bash script, CLI flags, --kubelet-extra-args for everything
# AL2023: Go binary, YAML config, structured kubelet settings

# Managed node groups:
# AL2: EKS injects bootstrap call (unless custom launch template)
# AL2023: EKS injects NodeConfig (merged with your userdata)

# Private clusters (no internet):
# Must provide --b64-cluster-ca and --apiserver-endpoint explicitly

# Troubleshooting:
cloud-init status                              # Did userdata finish?
systemctl status kubelet                       # Is kubelet running?
journalctl -u kubelet --since "5 min ago"      # Kubelet errors
curl -k https://<endpoint>/healthz             # Can node reach API?
cat /var/log/cloud-init-output.log             # Bootstrap script output

# Time from launch to Ready: ~60-120s (healthy cluster)
```
