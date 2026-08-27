# EKS Add-on Management — Conflicts, Custom Config, and Self-Managed vs Managed

How EKS manages add-ons, what happens when you customize them, conflict resolution strategies, when to self-manage, and the hidden behaviors that catch teams off-guard.

## How EKS Add-on Management Works

When you install an add-on through EKS (API or console), AWS manages the lifecycle:

```
┌────────────────────────────────────────────────────────────────────┐
│  EKS Add-on Management                                             │
│                                                                    │
│  1. AWS maintains a catalog of add-on versions per K8s version     │
│  2. When you install/update, EKS applies a Kubernetes manifest     │
│  3. EKS tracks which fields it "owns" (managed fields)             │
│  4. On update, EKS reconciles its fields back to desired state     │
│  5. Custom changes to managed fields may be OVERWRITTEN            │
│                                                                    │
│  EKS uses Server-Side Apply with field manager "eks"               │
└────────────────────────────────────────────────────────────────────┘
```

### What EKS Actually Deploys

Each add-on is a set of standard Kubernetes resources:

| Add-on | Resources Created |
|--------|-------------------|
| `vpc-cni` | DaemonSet (aws-node), ServiceAccount, ConfigMap, ClusterRole, ClusterRoleBinding |
| `kube-proxy` | DaemonSet (kube-proxy), ServiceAccount, ConfigMap, ClusterRole, ClusterRoleBinding |
| `coredns` | Deployment (coredns), Service (kube-dns), ServiceAccount, ConfigMap, ClusterRole, ClusterRoleBinding |
| `aws-ebs-csi-driver` | Deployment (controller), DaemonSet (node), ServiceAccount, StorageClass (optional) |
| `eks-pod-identity-agent` | DaemonSet, ServiceAccount, ClusterRole, ClusterRoleBinding |

```bash
# See what resources an add-on manages:
kubectl get all -n kube-system -l app.kubernetes.io/managed-by=EKS

# Or check by add-on name:
kubectl get all -n kube-system -l app.kubernetes.io/name=aws-node
kubectl get all -n kube-system -l k8s-app=kube-proxy
kubectl get all -n kube-system -l k8s-app=kube-dns
```

## The Conflict Problem

### What Happens When You Customize an EKS-Managed Add-on

```
┌────────────────────────────────────────────────────────────────┐
│  Scenario: You edit the aws-node DaemonSet directly            │
│                                                                │
│  kubectl set env daemonset aws-node -n kube-system             │
│    WARM_IP_TARGET=5                                            │
│                                                                │
│  This works immediately. But later...                          │
│                                                                │
│  aws eks update-addon --addon-name vpc-cni                     │
│    --resolve-conflicts OVERWRITE                               │
│                                                                │
│  Result: YOUR CHANGE IS GONE. EKS overwrote the DaemonSet      │
│  with its default config. WARM_IP_TARGET is back to default.   │
└────────────────────────────────────────────────────────────────┘
```

### Field Ownership (Server-Side Apply)

EKS uses Server-Side Apply with field manager name `"eks"`. You can see what EKS owns:

```bash
# Check which fields EKS manages on the aws-node DaemonSet:
kubectl get daemonset aws-node -n kube-system -o json | \
  jq '.metadata.managedFields[] | select(.manager == "eks") | .fieldsV1'
```

Fields owned by `"eks"` will be overwritten on add-on update. Fields NOT owned by EKS are safe from overwrites.

## resolve-conflicts — The Three Strategies

When updating an add-on, you choose how to handle conflicts between EKS defaults and your customizations:

### NONE (Fail on Conflict)

```bash
aws eks update-addon --cluster-name <cluster> --addon-name vpc-cni \
  --addon-version v1.18.5-eksbuild.1 \
  --resolve-conflicts NONE
```

- If you modified ANY field that EKS manages → update **fails**
- Safest option: forces you to explicitly decide
- Use when you want to know if you have customizations before proceeding

### OVERWRITE (EKS Wins)

```bash
aws eks update-addon --cluster-name <cluster> --addon-name vpc-cni \
  --addon-version v1.18.5-eksbuild.1 \
  --resolve-conflicts OVERWRITE
```

- EKS replaces ALL managed fields with its defaults
- Your customizations to managed fields are **lost**
- Quick and clean, but destructive to custom config
- **Common mistake**: running this without realizing you had custom env vars

### PRESERVE (Your Changes Win)

```bash
aws eks update-addon --cluster-name <cluster> --addon-name vpc-cni \
  --addon-version v1.18.5-eksbuild.1 \
  --resolve-conflicts PRESERVE
```

- EKS keeps your existing field values
- Only adds NEW fields from the updated version
- Your customizations survive
- **Risk**: your old values might be incompatible with the new add-on version

### Decision Matrix

| Scenario | Use |
|----------|-----|
| No customizations, just want latest | `OVERWRITE` |
| Custom env vars on aws-node (WARM_IP_TARGET, etc.) | `PRESERVE` |
| Unsure if team made changes | `NONE` (check first) |
| Custom ConfigMap (CoreDNS Corefile) | `PRESERVE` |
| Fresh install after self-managed removal | `OVERWRITE` |

## Custom Configuration — The Right Way

### Using configurationValues (Recommended)

Instead of editing resources directly, pass configuration through the add-on API:

```bash
# Set custom configuration via EKS API:
aws eks update-addon --cluster-name <cluster> --addon-name vpc-cni \
  --addon-version v1.18.5-eksbuild.1 \
  --configuration-values '{"env":{"WARM_IP_TARGET":"5","MINIMUM_IP_TARGET":"2"}}' \
  --resolve-conflicts OVERWRITE
```

```bash
# CoreDNS custom config:
aws eks update-addon --cluster-name <cluster> --addon-name coredns \
  --configuration-values '{"replicaCount":3,"resources":{"limits":{"memory":"256Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}}'

# kube-proxy custom config:
aws eks update-addon --cluster-name <cluster> --addon-name kube-proxy \
  --configuration-values '{"mode":"ipvs"}'

# EBS CSI Driver:
aws eks update-addon --cluster-name <cluster> --addon-name aws-ebs-csi-driver \
  --configuration-values '{"controller":{"replicaCount":3}}'
```

### Viewing Current Configuration

```bash
# See what custom config is set:
aws eks describe-addon --cluster-name <cluster> --addon-name vpc-cni \
  --query "addon.configurationValues"

# See the full configuration schema (what's configurable):
aws eks describe-addon-configuration \
  --addon-name vpc-cni \
  --addon-version v1.18.5-eksbuild.1 \
  --query "configurationSchema" --output text | jq .
```

### Configuration Schema

Each add-on publishes a JSON schema of what you can configure:

```bash
# Get the schema:
aws eks describe-addon-configuration \
  --addon-name coredns \
  --addon-version v1.11.3-eksbuild.2 | jq -r '.configurationSchema' | jq .
```

This shows all supported fields — anything outside the schema isn't officially supported via `configurationValues`.

## EKS-Managed vs Self-Managed Add-ons

### EKS-Managed (via aws eks create-addon)

```bash
# Install:
aws eks create-addon --cluster-name <cluster> --addon-name vpc-cni \
  --addon-version v1.18.5-eksbuild.1

# Update:
aws eks update-addon --cluster-name <cluster> --addon-name vpc-cni \
  --addon-version v1.19.0-eksbuild.1

# Delete:
aws eks delete-addon --cluster-name <cluster> --addon-name vpc-cni
# ⚠️ This REMOVES the add-on from the cluster! Pods may lose networking!
```

**Pros:**
- Automatic compatibility validation (version vs K8s version)
- Managed lifecycle (install, update, health monitoring)
- Configuration via API (survives updates)
- Health status monitoring via EKS API
- Integration with Terraform/CloudFormation

**Cons:**
- Limited configuration flexibility (schema-constrained)
- May overwrite custom changes on update
- Version lag (latest upstream may not be available immediately)
- Can't apply arbitrary patches

### Self-Managed (Helm, kubectl, Kustomize)

```bash
# Install VPC CNI via Helm (self-managed):
helm repo add aws-vpc-cni https://aws.github.io/amazon-vpc-cni-k8s
helm install aws-vpc-cni aws-vpc-cni/aws-vpc-cni \
  --namespace kube-system \
  --set env.WARM_IP_TARGET=5 \
  --set env.MINIMUM_IP_TARGET=2
```

**Pros:**
- Full control over every field
- Can use latest upstream versions immediately
- Helm values for structured configuration
- GitOps-friendly (ArgoCD, Flux can manage)
- No EKS API dependency for changes

**Cons:**
- YOU own compatibility (must check version vs K8s version)
- No automatic health monitoring from EKS
- Must track updates yourself
- Risk of misconfiguration
- Terraform/EKS API can't see or manage it

### When to Self-Manage

| Scenario | Self-Manage? | Why |
|----------|:------------:|-----|
| Standard cluster, default config | No | EKS-managed is simpler |
| Custom CoreDNS plugins (e.g., forward to on-prem) | Maybe | configurationValues might suffice |
| Need latest VPC CNI before EKS publishes it | Yes | EKS lags upstream by weeks |
| Heavy customization (many env vars, tolerations, affinity) | Yes | Schema may not expose what you need |
| GitOps (ArgoCD manages everything) | Yes | Consistency with other resources |
| Cluster bootstrapping via Terraform only | Depends | EKS Terraform provider supports add-ons |

## Migrating Between Managed and Self-Managed

### EKS-Managed → Self-Managed

```bash
# 1. Document current configuration:
aws eks describe-addon --cluster-name <cluster> --addon-name vpc-cni

# 2. Remove EKS management (WITHOUT deleting the resources):
aws eks delete-addon --cluster-name <cluster> --addon-name vpc-cni --preserve
# --preserve keeps the DaemonSet running, just removes EKS management

# 3. Now manage it with Helm/kubectl/ArgoCD
```

### Self-Managed → EKS-Managed

```bash
# 1. Ensure your running version is compatible:
aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.30

# 2. Create the EKS add-on (adopts existing resources):
aws eks create-addon --cluster-name <cluster> --addon-name vpc-cni \
  --addon-version v1.18.5-eksbuild.1 \
  --resolve-conflicts OVERWRITE
# This takes ownership of existing DaemonSet

# ⚠️ OVERWRITE will reset to defaults. Use PRESERVE to keep your config.
```

## Add-on Health Monitoring

```bash
# Check add-on health:
aws eks describe-addon --cluster-name <cluster> --addon-name vpc-cni \
  --query "addon.{Status:status,Health:health,Version:addonVersion}"

# Possible statuses:
# ACTIVE — healthy
# CREATING — being installed
# UPDATING — being upgraded
# DELETING — being removed
# CREATE_FAILED — installation failed
# UPDATE_FAILED — upgrade failed
# DEGRADED — running but unhealthy
```

### Health Issues

| Status | Cause | Fix |
|--------|-------|-----|
| `DEGRADED` | Pods crashing, wrong IAM role, image pull failure | Check pod logs: `kubectl logs -n kube-system -l app.kubernetes.io/name=aws-node` |
| `CREATE_FAILED` | Incompatible version, insufficient permissions | Check IAM role, check version compatibility |
| `UPDATE_FAILED` | Conflict with manual changes, webhook blocking | Use `--resolve-conflicts OVERWRITE` or fix the conflict |

## Hidden Behaviors

### 1. EKS Periodically Reconciles Managed Fields

Even without an explicit update, EKS may reconcile add-on state during platform version patches. If you edited a managed field directly, it could revert.

### 2. delete-addon Without --preserve Removes Everything

```bash
# DANGEROUS: This removes the DaemonSet/Deployment from the cluster:
aws eks delete-addon --cluster-name <cluster> --addon-name vpc-cni
# All pods lose networking!

# SAFE: Remove EKS management but keep resources running:
aws eks delete-addon --cluster-name <cluster> --addon-name vpc-cni --preserve
```

### 3. CoreDNS ConfigMap Is Partially Managed

The `coredns` ConfigMap has a special annotation:

```yaml
metadata:
  annotations:
    # This tells EKS to NOT overwrite the Corefile:
    eks.amazonaws.com/configmap-override-protected: "true"
```

If this annotation is present, EKS won't overwrite your custom Corefile on updates. But it WILL update other fields (replicas, image, resource limits).

### 4. IAM Role for Add-ons

Some add-ons need IAM permissions (VPC CNI, EBS CSI). The recommended approach is Pod Identity or IRSA:

```bash
# Create add-on with a specific IAM role:
aws eks create-addon --cluster-name <cluster> --addon-name vpc-cni \
  --service-account-role-arn arn:aws:iam::123456789:role/AmazonEKS_CNI_Role

# Or use Pod Identity association:
aws eks create-pod-identity-association \
  --cluster-name <cluster> \
  --namespace kube-system \
  --service-account aws-node \
  --role-arn arn:aws:iam::123456789:role/AmazonEKS_CNI_Role
```

### 5. Add-on Version Doesn't Match Cluster Version

You can run add-on versions from different K8s versions (within compatibility bounds). EKS won't force-upgrade add-ons when you upgrade the control plane.

```bash
# Cluster on 1.30 but vpc-cni still on 1.29-compatible version:
# This is VALID but not recommended. Upgrade add-ons after control plane.
```

## Terraform Integration

```hcl
resource "aws_eks_addon" "vpc_cni" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "vpc-cni"
  addon_version               = "v1.18.5-eksbuild.1"
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  configuration_values = jsonencode({
    env = {
      WARM_IP_TARGET     = "5"
      MINIMUM_IP_TARGET  = "2"
    }
  })

  service_account_role_arn = aws_iam_role.vpc_cni.arn
}

resource "aws_eks_addon" "coredns" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "coredns"
  addon_version               = "v1.11.3-eksbuild.2"
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  configuration_values = jsonencode({
    replicaCount = 3
  })
}
```

## Quick Reference

```bash
# List add-ons:
aws eks list-addons --cluster-name <cluster>

# Describe (status, version, config):
aws eks describe-addon --cluster-name <cluster> --addon-name <addon>

# Install:
aws eks create-addon --cluster-name <cluster> --addon-name <addon> \
  --addon-version <version>

# Update:
aws eks update-addon --cluster-name <cluster> --addon-name <addon> \
  --addon-version <version> --resolve-conflicts PRESERVE

# Custom config:
aws eks update-addon --cluster-name <cluster> --addon-name <addon> \
  --configuration-values '<json>'

# See configurable schema:
aws eks describe-addon-configuration --addon-name <addon> --addon-version <version>

# Delete (keep resources):
aws eks delete-addon --cluster-name <cluster> --addon-name <addon> --preserve

# Delete (remove everything — dangerous):
aws eks delete-addon --cluster-name <cluster> --addon-name <addon>

# resolve-conflicts:
# NONE      = fail if conflicts exist
# OVERWRITE = EKS defaults win (your changes lost)
# PRESERVE  = your changes win (may be incompatible)

# Best practice:
# Use configurationValues for customization (survives updates)
# Use PRESERVE for updates (keeps your config)
# Use --preserve when deleting (doesn't remove running resources)
```
