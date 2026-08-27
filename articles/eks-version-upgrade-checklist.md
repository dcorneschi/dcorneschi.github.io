# EKS Version Upgrades — Checklist and Process

A complete checklist for upgrading EKS clusters — pre-upgrade validation, control plane upgrade, add-on compatibility, node group rolling updates, post-upgrade verification, and rollback limitations.

Note: For control plane patching internals (platform version updates, rolling API server replacement), see the EKS architecture deep-dive. This article is the operational checklist for Kubernetes minor version upgrades (e.g., 1.29 → 1.30).

## Upgrade Overview

```
┌────────────────────────────────────────────────────────────────────┐
│  EKS Upgrade Sequence (MUST follow this order)                     │
│                                                                    │
│  1. Pre-upgrade checks (APIs, deprecations, add-on compat)         │
│  2. Upgrade control plane (AWS-managed, zero-downtime)             │
│  3. Upgrade EKS add-ons (VPC CNI, CoreDNS, kube-proxy)             │
│  4. Upgrade worker nodes (rolling replacement)                     │
│  5. Post-upgrade verification                                      │
│                                                                    │
│    Cannot be rolled back. Test in staging first.                   │
│    Nodes can be at most N-2 versions behind control plane.         │
└────────────────────────────────────────────────────────────────────┘
```

## Pre-Upgrade Checklist

### 1. Check Current Versions

```bash
# Current cluster version:
aws eks describe-cluster --name <cluster> \
  --query "{K8sVersion:cluster.version,Platform:cluster.platformVersion}" --output table

# Current node versions:
kubectl get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion

# Current add-on versions:
aws eks list-addons --cluster-name <cluster> --output table
for addon in $(aws eks list-addons --cluster-name <cluster> --query "addons[]" --output text); do
  echo "$addon: $(aws eks describe-addon --cluster-name <cluster> --addon-name $addon --query "addon.addonVersion" --output text)"
done
```

### 2. Check for Deprecated/Removed APIs

Every K8s version removes APIs. If your manifests use removed APIs, they'll break.

```bash
# Use pluto to scan for deprecated APIs:
pluto detect-all-in-cluster

# Or check specific manifests:
pluto detect-files -d ./manifests/

# Check deprecated APIs still in use via audit logs:
# Look for "k8s.io/deprecated" annotation in audit events
```

### Common API Removals by Version

| Version | Removed API | Replacement |
|---------|------------|-------------|
| 1.25 | PodSecurityPolicy | Pod Security Admission |
| 1.25 | policy/v1beta1 PodDisruptionBudget | policy/v1 |
| 1.26 | flowcontrol.apiserver.k8s.io/v1beta1 | v1beta3 or v1 |
| 1.27 | storage.k8s.io/v1beta1 CSIStorageCapacity | storage.k8s.io/v1 |
| 1.29 | flowcontrol.apiserver.k8s.io/v1beta2 | v1beta3 or v1 |
| 1.32 | flowcontrol.apiserver.k8s.io/v1beta3 | v1 |

```bash
# Check if deprecated APIs are being called (from audit logs):
aws logs filter-log-events \
  --log-group-name /aws/eks/<cluster>/cluster \
  --log-stream-name-prefix kube-apiserver-audit \
  --filter-pattern '"k8s.io/deprecated":"true"' \
  --start-time $(date -u -d '24 hours ago' '+%s000')
```

### 3. Check Add-on Compatibility

```bash
# List compatible add-on versions for the TARGET K8s version:
aws eks describe-addon-versions \
  --kubernetes-version 1.30 \
  --addon-name vpc-cni \
  --query "addons[0].addonVersions[].{Version:addonVersion,Default:compatibilities[0].defaultVersion}" \
  --output table

# Check all add-ons:
for addon in vpc-cni kube-proxy coredns aws-ebs-csi-driver; do
  echo "=== $addon ==="
  aws eks describe-addon-versions --kubernetes-version 1.30 --addon-name $addon \
    --query "addons[0].addonVersions[0].addonVersion" --output text
done
```

### 4. Check Node Group AMI Availability

```bash
# Check available AMI for target version:
aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.30/amazon-linux-2023/x86_64/standard/recommended/image_id \
  --query "Parameter.Value" --output text

# For GPU:
aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.30/amazon-linux-2023/x86_64/nvidia/recommended/image_id \
  --query "Parameter.Value" --output text
```

### 5. Check Cluster Health

```bash
# Nodes all Ready:
kubectl get nodes --no-headers | grep -v " Ready " && echo "UNHEALTHY NODES!" || echo "All nodes Ready"

# No failing pods:
kubectl get pods -A --field-selector status.phase=Failed
kubectl get pods -A | grep -v "Running\|Completed" | grep -v "NAMESPACE"

# PDBs that could block drains:
kubectl get pdb -A -o custom-columns=\
  NS:.metadata.namespace,NAME:.metadata.name,\
  MIN:.spec.minAvailable,MAX:.spec.maxUnavailable,\
  ALLOWED:.status.disruptionsAllowed
# Any with ALLOWED=0 will block node rolling updates

# Check pending PVCs:
kubectl get pvc -A --field-selector status.phase!=Bound
```

### 6. Check Version Skew Policy

```
Control plane: 1.30 (after upgrade)
Nodes allowed: 1.28, 1.29, 1.30

If nodes are currently on 1.28:
  → Control plane can go to 1.30 (nodes are N-2, still valid)
  → But you MUST upgrade nodes before going to 1.31
```

### 7. Review Release Notes

```bash
# Check Kubernetes changelog for the target version:
# https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.30.md

# Check EKS-specific release notes:
# https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html
```

## Phase 1: Upgrade Control Plane

```bash
# Initiate the upgrade:
aws eks update-cluster-version \
  --name <cluster> \
  --kubernetes-version 1.30

# Monitor progress:
aws eks describe-update --name <cluster> --update-id <update-id>

# Or watch:
aws eks wait cluster-active --name <cluster>
# Takes 15-45 minutes typically
```

### What Happens During Control Plane Upgrade

```
1. AWS launches new API server instances with K8s 1.30
2. Rolling replacement behind the NLB (zero-downtime)
3. etcd is upgraded (Raft-based, quorum maintained)
4. Once all new instances healthy, old ones terminated
5. Cluster endpoint remains the same

During the upgrade:
- Existing workloads continue running (nodes unaffected)
- kubectl may see brief 5xx during API server switchover
- Webhooks may timeout briefly (retry logic needed)
- You can't create resources using NEW K8s 1.30 APIs until complete
```

```bash
# Check upgrade status:
aws eks describe-cluster --name <cluster> \
  --query "cluster.{Status:status,Version:version,Platform:platformVersion}"

# Status goes: ACTIVE → UPDATING → ACTIVE
```

## Phase 2: Upgrade Add-ons

**Always upgrade add-ons AFTER the control plane and BEFORE nodes.**

```bash
# Upgrade VPC CNI:
aws eks update-addon --cluster-name <cluster> --addon-name vpc-cni \
  --addon-version v1.18.5-eksbuild.1 \
  --resolve-conflicts OVERWRITE

# Upgrade kube-proxy:
aws eks update-addon --cluster-name <cluster> --addon-name kube-proxy \
  --addon-version v1.30.6-eksbuild.3 \
  --resolve-conflicts OVERWRITE

# Upgrade CoreDNS:
aws eks update-addon --cluster-name <cluster> --addon-name coredns \
  --addon-version v1.11.3-eksbuild.2 \
  --resolve-conflicts OVERWRITE

# Upgrade EBS CSI Driver:
aws eks update-addon --cluster-name <cluster> --addon-name aws-ebs-csi-driver \
  --addon-version v1.37.0-eksbuild.1 \
  --resolve-conflicts OVERWRITE
```

### resolve-conflicts Options

| Option | Behavior |
|--------|----------|
| `NONE` | Fail if any field was manually modified (safest) |
| `OVERWRITE` | Replace custom config with EKS defaults (common) |
| `PRESERVE` | Keep your custom config, add new fields only |

```bash
# Check add-on upgrade progress:
aws eks describe-addon --cluster-name <cluster> --addon-name vpc-cni \
  --query "addon.{Status:status,Version:addonVersion,Health:health}"
```

### Add-on Version Compatibility Matrix

```bash
# Get the default (recommended) version for each add-on:
for addon in vpc-cni kube-proxy coredns aws-ebs-csi-driver; do
  VERSION=$(aws eks describe-addon-versions --kubernetes-version 1.30 --addon-name $addon \
    --query "addons[0].addonVersions[?compatibilities[0].defaultVersion==\`true\`].addonVersion" --output text)
  echo "$addon → $VERSION"
done
```

## Phase 3: Upgrade Worker Nodes

### Managed Node Groups

```bash
# Update the node group:
aws eks update-nodegroup-version \
  --cluster-name <cluster> \
  --nodegroup-name <nodegroup> \
  --kubernetes-version 1.30

# Or update to latest AMI without changing K8s version:
aws eks update-nodegroup-version \
  --cluster-name <cluster> \
  --nodegroup-name <nodegroup> \
  --release-version 1.30.6-20241121

# Monitor rolling update:
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <nodegroup> \
  --query "nodegroup.{Status:status,Version:version,ReleaseVersion:releaseVersion}"
```

### Rolling Update Behavior

```
1. New EC2 instances launched with new AMI (K8s 1.30)
2. Wait for new nodes to join and become Ready
3. Old nodes cordoned (unschedulable)
4. Old nodes drained (pods evicted, PDBs respected)
5. Old nodes terminated

EKS respects:
- PodDisruptionBudgets (won't violate PDBs)
- maxUnavailable setting (how many nodes drain at once)
- Force timeout after ~15 minutes per node
```

### Node Group Update Configuration

```bash
# Control parallelism:
aws eks update-nodegroup-config --cluster-name <cluster> --nodegroup-name <nodegroup> \
  --update-config '{"maxUnavailable": 1}'
# OR
  --update-config '{"maxUnavailablePercentage": 25}'
```

| Setting | Default | Effect |
|---------|---------|--------|
| `maxUnavailable: 1` | Default | One node at a time (slow, safe) |
| `maxUnavailable: 3` | — | Three nodes simultaneously (faster) |
| `maxUnavailablePercentage: 33` | — | One-third of nodes at a time |

### Self-Managed Node Groups

For self-managed (ASG-based) nodes, you manage the upgrade yourself:

```bash
# Option 1: Update launch template with new AMI, then rolling replace
aws ec2 create-launch-template-version --launch-template-id lt-xxx \
  --source-version 1 \
  --launch-template-data '{"ImageId":"ami-new-eks-1.30"}'

# Trigger instance refresh:
aws autoscaling start-instance-refresh --auto-scaling-group-name <asg-name> \
  --preferences '{"MinHealthyPercentage":90,"InstanceWarmup":300}'

# Option 2: Double the ASG, drain old nodes, scale down
```

## Phase 4: Post-Upgrade Verification

```bash
# 1. Verify control plane version:
aws eks describe-cluster --name <cluster> --query "cluster.version"

# 2. Verify all nodes on new version:
kubectl get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion
# All should show v1.30.x

# 3. Verify add-on health:
for addon in $(aws eks list-addons --cluster-name <cluster> --query "addons[]" --output text); do
  STATUS=$(aws eks describe-addon --cluster-name <cluster> --addon-name $addon --query "addon.status" --output text)
  VERSION=$(aws eks describe-addon --cluster-name <cluster> --addon-name $addon --query "addon.addonVersion" --output text)
  echo "$addon: $STATUS ($VERSION)"
done

# 4. Verify workload health:
kubectl get pods -A | grep -v "Running\|Completed" | grep -v "NAMESPACE"

# 5. Verify DNS:
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup kubernetes.default

# 6. Verify networking:
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl -s http://kubernetes.default.svc/healthz

# 7. Check events for errors:
kubectl get events -A --sort-by=.lastTimestamp | tail -30 | grep -i "error\|fail\|warning"

# 8. Verify cluster autoscaler/Karpenter still functions:
# (trigger a scale test or check logs)
```

## Rollback Limitations

```
┌────────────────────────────────────────────────────────────────┐
│    YOU CANNOT DOWNGRADE THE CONTROL PLANE                      │
│                                                                │
│  Once upgraded to 1.30, you CANNOT go back to 1.29.            │
│  This is a one-way operation.                                  │
│                                                                │
│  If the upgrade breaks something:                              │
│  - Fix forward (patch your manifests/configs)                  │
│  - Or create a new cluster on the old version and migrate      │
│                                                                │
│  Worker nodes CAN be rolled back:                              │
│  - Point node group at old AMI                                 │
│  - Rolling replacement brings back old kubelet version         │
│  - Only valid if version skew is within N-2                    │
└────────────────────────────────────────────────────────────────┘
```

## Extended Support and End-of-Life

```bash
# Check support status for versions:
# Standard support: 14 months after release
# Extended support: Additional 12 months ($0.60/cluster/hour)
# End of extended support: Forced upgrade

# Check your cluster's support type:
aws eks describe-cluster --name <cluster> --query "cluster.upgradePolicy"
```

| Phase | Duration | Cost | Action Needed |
|-------|:--------:|:----:|--------------|
| Standard support | 14 months | $0.10/hr | None |
| Extended support | +12 months | $0.60/hr | Plan upgrade |
| End of extended | — | — | AWS force-upgrades your cluster |

## Upgrade Automation Script

```bash
#!/bin/bash
set -e

CLUSTER=$1
TARGET_VERSION=$2

if [ -z "$CLUSTER" ] || [ -z "$TARGET_VERSION" ]; then
  echo "Usage: $0 <cluster-name> <target-version>"
  exit 1
fi

echo "=== Pre-flight checks ==="
CURRENT=$(aws eks describe-cluster --name $CLUSTER --query "cluster.version" --output text)
echo "Current: $CURRENT → Target: $TARGET_VERSION"

# Check nodes are healthy
UNHEALTHY=$(kubectl get nodes --no-headers | grep -v " Ready " | wc -l)
if [ "$UNHEALTHY" -gt 0 ]; then
  echo "ERROR: $UNHEALTHY unhealthy nodes. Fix before upgrading."
  exit 1
fi

echo "=== Upgrading control plane ==="
aws eks update-cluster-version --name $CLUSTER --kubernetes-version $TARGET_VERSION
echo "Waiting for control plane upgrade..."
aws eks wait cluster-active --name $CLUSTER
echo "Control plane upgraded to $TARGET_VERSION"

echo "=== Upgrading add-ons ==="
for addon in vpc-cni kube-proxy coredns; do
  LATEST=$(aws eks describe-addon-versions --kubernetes-version $TARGET_VERSION --addon-name $addon \
    --query "addons[0].addonVersions[?compatibilities[0].defaultVersion==\`true\`].addonVersion" --output text)
  echo "Upgrading $addon to $LATEST..."
  aws eks update-addon --cluster-name $CLUSTER --addon-name $addon \
    --addon-version $LATEST --resolve-conflicts PRESERVE
done

echo "=== Upgrading node groups ==="
for ng in $(aws eks list-nodegroups --cluster-name $CLUSTER --query "nodegroups[]" --output text); do
  echo "Upgrading node group: $ng"
  aws eks update-nodegroup-version --cluster-name $CLUSTER --nodegroup-name $ng
done

echo "=== Upgrade initiated. Monitor with: ==="
echo "aws eks describe-cluster --name $CLUSTER --query cluster.status"
echo "kubectl get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion"
```

## Timing Expectations

| Phase | Duration | Downtime |
|-------|:--------:|:--------:|
| Control plane upgrade | 15-45 min | Zero (rolling behind NLB) |
| Add-on upgrades | 2-5 min each | Brief (rolling restart) |
| Node group rolling (per node) | 5-15 min | Pods evicted (rescheduled) |
| Full cluster (50 nodes) | 2-4 hours | No overall downtime (rolling) |

## Quick Reference

```bash
# Pre-upgrade:
pluto detect-all-in-cluster              # Check deprecated APIs
kubectl get nodes                         # Verify all Ready
kubectl get pdb -A                        # Check PDBs won't block

# Upgrade control plane:
aws eks update-cluster-version --name <cluster> --kubernetes-version 1.30
aws eks wait cluster-active --name <cluster>

# Upgrade add-ons:
aws eks update-addon --cluster-name <cluster> --addon-name <addon> \
  --addon-version <version> --resolve-conflicts PRESERVE

# Upgrade nodes:
aws eks update-nodegroup-version --cluster-name <cluster> --nodegroup-name <ng>

# Verify:
kubectl get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion
kubectl get pods -A | grep -v "Running\|Completed"

# Order: control plane → add-ons → nodes
# Cannot rollback control plane
# Nodes can be N-2 behind control plane
# Test in staging first — always
```
