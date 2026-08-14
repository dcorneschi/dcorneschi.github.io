# EKS Access Entries - Authentication Mode

## Overview

Starting with Kubernetes 1.29+, AWS EKS supports **Access Entries** as the recommended method for granting IAM principals access to Kubernetes resources. This replaces the deprecated `aws-auth` ConfigMap approach.

## Official AWS Documentation

### Key Points from AWS Docs

1. **aws-auth ConfigMap is deprecated**
   - [Official deprecation notice](https://docs.aws.amazon.com/en_en/eks/latest/userguide/auth-configmap.html)
   - AWS explicitly states: "The aws-auth ConfigMap is deprecated. For the recommended method to manage access to Kubernetes APIs, see Grant IAM users access to Kubernetes with EKS access entries."

2. **Access Entries are recommended for all new clusters**
   - [Grant IAM users access documentation](https://docs.aws.amazon.com/eks/latest/userguide/grant-k8s-access.html)
   - AWS recommends using Access Entries for any cluster at or above certain platform versions

3. **Platform version requirements**
   - For Kubernetes **1.31 and above** (including 1.33): All platform versions support Access Entries
   - No minimum platform version requirement needed
   - [Platform version details](https://docs.aws.amazon.com/eks/latest/userguide/setting-up-access-entries.html)

---

## Authentication Modes

EKS clusters support three authentication modes:

| Mode | Description | Use Case |
|------|-------------|----------|
| `CONFIG_MAP` | Legacy mode using aws-auth ConfigMap only | Deprecated - not recommended |
| `API_AND_CONFIG_MAP` | Both Access Entries API and ConfigMap supported | **Recommended** for migration scenarios |
| `API` | Access Entries API only | For new clusters with no legacy dependencies |

### Important Notes

- Authentication mode must be explicitly set to `API` or `API_AND_CONFIG_MAP` for Access Entries to work
- If not specified, Terraform defaults may not enable Access Entries
- You cannot downgrade from `API` back to `CONFIG_MAP` mode
- Managed node groups and Fargate profiles automatically get access entries in `API_AND_CONFIG_MAP` mode

---

## Terraform Configuration

### Enabling Access Entries on the Cluster

```hcl
access_config {
  authentication_mode = "API_AND_CONFIG_MAP"
}
```

This ensures:
- ✅ Access Entries API is enabled
- ✅ Backward compatibility with any existing aws-auth ConfigMap entries
- ✅ Future-proof for AWS's recommended authentication method

### Access Entry Resources

Create access entries to grant console or CLI access:

```hcl
resource "aws_eks_access_entry" "console_admin" {
  cluster_name  = aws_eks_cluster.eks.name
  principal_arn = each.value
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "console_admin" {
  cluster_name  = aws_eks_cluster.eks.name
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
  principal_arn = each.value

  access_scope {
    type = "cluster"
  }
}
```

### Configuration Variables

The `console_access_principals` variable allows flexibility:

```hcl
# Default: Grants access to whoever runs terraform apply
console_access_principals = []

# Explicit: Grant access to specific IAM principals
console_access_principals = [
  "arn:aws:iam::123456789012:user/alice",
  "arn:aws:iam::123456789012:role/DevOpsRole"
]
```

---

## Why This Matters

### The "Unauthorized" Console Error

Without proper access entries:
- ❌ Cluster exists and is functional
- ❌ Nodes are running and healthy
- ❌ But AWS Console shows "Unauthorized"
- ❌ Cannot view nodes, pods, or other Kubernetes resources in console

With access entries configured:
- ✅ IAM principals are registered with the cluster
- ✅ Kubernetes permissions are granted via Access Policies
- ✅ AWS Console can query and display cluster resources
- ✅ Users can view nodes, pods, deployments, etc.

### Why ConfigMap Mode Fails for Console Access

This is a **chicken-and-egg problem** that is fundamental to how ConfigMap mode works.

**The aws-auth ConfigMap lives inside Kubernetes.** To read or write it, you already need Kubernetes API access. So if your IAM user was never added to the ConfigMap, you can't get in to add yourself — and nobody else can add you without already being inside the cluster.

**How it typically breaks:**

1. The cluster is created by a CI/CD role (e.g. a Gitea/Forgejo pipeline role)
2. EKS auto-adds that CI role to the aws-auth ConfigMap as cluster admin
3. Your personal IAM user (the one you use in the AWS console) was **never added**
4. When the AWS console tries to call the Kubernetes API to list nodes and pods, it gets rejected — "Unauthorized"

**The trap:** You can't fix it by logging into the console because you don't have access. You'd need to either use the same CI role locally, or `kubectl` with a kubeconfig that authenticates as that CI role.

**Why API_AND_CONFIG_MAP mode solves this:**

Access is managed **outside Kubernetes** via the AWS IAM API. You grant access using standard AWS tools (console, CLI, Terraform) — no need to already be inside the cluster. The `aws_eks_access_entry` resource registers your IAM identity at the AWS level, completely bypassing the Kubernetes API for the access grant itself.

| | ConfigMap mode | API_AND_CONFIG_MAP mode |
|---|---|---|
| Where access is stored | Inside Kubernetes (ConfigMap) | AWS IAM API (outside Kubernetes) |
| Who can grant access | Only existing cluster admins | Anyone with IAM `eks:CreateAccessEntry` permission |
| Self-service fix possible | ❌ No | ✅ Yes |
| Tool to manage | `kubectl` | AWS Console / CLI / Terraform |
| Risk of lockout | ✅ High | ❌ Low |

---

## Upgrading an Existing Cluster from ConfigMap Mode

If an existing cluster is already running in `ConfigMap` mode (visible in the AWS console under **Access configuration → Authentication mode: ConfigMap**), you cannot apply the Terraform access entry resources until you first upgrade the authentication mode.

**Step 1 — Upgrade the authentication mode via AWS CLI:**

```bash
aws eks update-cluster-config \
  --name demo-cluster \
  --access-config authenticationMode=API_AND_CONFIG_MAP \
  --region us-east-1
```

**Step 2 — Wait for the cluster to finish updating:**

```bash
aws eks describe-cluster \
  --name demo-cluster \
  --region us-east-1 \
  --query "cluster.status"
# Wait until it returns "ACTIVE"
```

**Step 3 — Apply Terraform to create the access entries:**

```bash
terraform apply
```

### Important Constraints

- This change is **one-way only** — you cannot revert from `API_AND_CONFIG_MAP` back to `CONFIG_MAP`
- `API_AND_CONFIG_MAP` keeps all existing ConfigMap entries intact — managed node groups and Fargate profiles won't be affected
- To fully disable the ConfigMap (optional, future step), upgrade to `API` mode

---

## Available Access Policies

EKS provides built-in access policies that map to standard Kubernetes RBAC:

| Access Policy | K8s RBAC Equivalent | Scope | Use Case |
|---------------|--------------------:|-------|----------|
| `AmazonEKSClusterAdminPolicy` | `cluster-admin` ClusterRole | Cluster | Full admin — unrestricted access to all resources |
| `AmazonEKSAdminPolicy` | `admin` ClusterRole | Cluster or Namespace | Manage most resources, but can't modify RBAC or cluster-level settings |
| `AmazonEKSEditPolicy` | `edit` ClusterRole | Cluster or Namespace | Create/update/delete workloads, but no RBAC or namespace management |
| `AmazonEKSViewPolicy` | `view` ClusterRole | Cluster or Namespace | Read-only access to most resources (no secrets) |

Policy ARN format:

```
arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy
arn:aws:eks::aws:cluster-access-policy/AmazonEKSAdminPolicy
arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy
arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy
```

### Choosing the Right Policy

| Principal | Recommended Policy | Why |
|-----------|-------------------|-----|
| Platform/DevOps team | `ClusterAdminPolicy` | Full cluster management |
| Dev team leads | `AdminPolicy` (namespace-scoped) | Manage team resources without cluster-wide impact |
| Developers | `EditPolicy` (namespace-scoped) | Deploy and debug workloads |
| Read-only dashboards, auditors | `ViewPolicy` | Observe without modification |
| CI/CD pipelines | `ClusterAdminPolicy` or `EditPolicy` | Depends on deployment scope |

---

## Namespace-Scoped Access

The `access_scope` block supports restricting access to specific namespaces:

```hcl
resource "aws_eks_access_policy_association" "dev_team_edit" {
  cluster_name  = aws_eks_cluster.eks.name
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy"
  principal_arn = "arn:aws:iam::123456789012:role/DevTeamRole"

  access_scope {
    type       = "namespace"
    namespaces = ["app-frontend", "app-backend"]
  }
}
```

Scope types:

| Type | Behavior |
|------|----------|
| `cluster` | Policy applies to all namespaces and cluster-scoped resources |
| `namespace` | Policy applies only to the listed namespaces |

Notes:
- `AmazonEKSClusterAdminPolicy` only supports `type = "cluster"` — it cannot be namespace-scoped
- You can associate **multiple policies** with the same access entry (e.g., `ViewPolicy` cluster-wide + `EditPolicy` for one namespace)
- Namespace-scoped access does NOT grant access to cluster-scoped resources (nodes, PVs, namespaces themselves)

### Example: Multi-Team Access Pattern

```hcl
# Platform team — full cluster admin
resource "aws_eks_access_policy_association" "platform_admin" {
  cluster_name  = aws_eks_cluster.eks.name
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
  principal_arn = "arn:aws:iam::123456789012:role/PlatformTeamRole"

  access_scope {
    type = "cluster"
  }
}

# Backend team — edit access to their namespaces only
resource "aws_eks_access_policy_association" "backend_edit" {
  cluster_name  = aws_eks_cluster.eks.name
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy"
  principal_arn = "arn:aws:iam::123456789012:role/BackendTeamRole"

  access_scope {
    type       = "namespace"
    namespaces = ["backend-api", "backend-workers"]
  }
}

# Backend team — cluster-wide view for debugging
resource "aws_eks_access_policy_association" "backend_view" {
  cluster_name  = aws_eks_cluster.eks.name
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy"
  principal_arn = "arn:aws:iam::123456789012:role/BackendTeamRole"

  access_scope {
    type = "cluster"
  }
}
```

---

## Entry Types

The `type` field in an access entry determines how the principal interacts with the cluster:

| Type | Purpose | When to Use |
|------|---------|-------------|
| `STANDARD` | Human users, CI/CD roles, any IAM principal | Default for console/CLI access |
| `EC2_LINUX` | EC2 Linux node groups | Node IAM role for Linux worker nodes |
| `EC2_WINDOWS` | EC2 Windows node groups | Node IAM role for Windows worker nodes |
| `FARGATE_LINUX` | Fargate pod execution role | Fargate profile role |

### Key Differences

**STANDARD entries:**
- Require an explicit policy association to grant any permissions
- Without a policy, the entry exists but the principal has zero Kubernetes access
- Used for humans, CI roles, and service accounts

**EC2_LINUX / EC2_WINDOWS entries:**
- Automatically grant the `AmazonEKSNodePolicy` equivalent permissions
- The node can join the cluster and register itself
- You typically don't need to create these manually — managed node groups handle it
- Only needed when using self-managed node groups with `API` mode

**FARGATE_LINUX entries:**
- Automatically grant Fargate pod execution permissions
- Created automatically when using Fargate profiles with `API_AND_CONFIG_MAP` mode

```hcl
# Self-managed node group — must create EC2_LINUX entry manually
resource "aws_eks_access_entry" "self_managed_nodes" {
  cluster_name  = aws_eks_cluster.eks.name
  principal_arn = aws_iam_role.node_role.arn
  type          = "EC2_LINUX"
}
```

---

## Verifying Access Entries

### List All Access Entries

```bash
aws eks list-access-entries \
  --cluster-name demo-cluster \
  --region us-east-1
```

### Describe a Specific Entry

```bash
aws eks describe-access-entry \
  --cluster-name demo-cluster \
  --principal-arn "arn:aws:iam::123456789012:role/DevOpsRole" \
  --region us-east-1
```

### List Associated Policies for an Entry

```bash
aws eks list-associated-access-policies \
  --cluster-name demo-cluster \
  --principal-arn "arn:aws:iam::123456789012:role/DevOpsRole" \
  --region us-east-1
```

### Check Who Has Access (All Entries + Policies)

```bash
# List all entries and their policies
for arn in $(aws eks list-access-entries --cluster-name demo-cluster --query 'accessEntries[*]' --output text); do
  echo "=== $arn ==="
  aws eks list-associated-access-policies \
    --cluster-name demo-cluster \
    --principal-arn "$arn" \
    --query 'associatedAccessPolicies[*].{Policy:policyArn,Scope:accessScope.type}' \
    --output table
done
```

### Verify Your Own Access

```bash
# Check which IAM identity you're using
aws sts get-caller-identity

# Verify you can authenticate to the cluster
aws eks get-token --cluster-name demo-cluster --region us-east-1
```

---

## Common Mistakes

### 1. Entry Without Policy Association

Creating an access entry alone does **not** grant any Kubernetes permissions:

```hcl
# ❌ This alone does nothing — principal can authenticate but has zero permissions
resource "aws_eks_access_entry" "user" {
  cluster_name  = aws_eks_cluster.eks.name
  principal_arn = "arn:aws:iam::123456789012:user/alice"
  type          = "STANDARD"
}

# ✅ Must also associate a policy
resource "aws_eks_access_policy_association" "user" {
  cluster_name  = aws_eks_cluster.eks.name
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy"
  principal_arn = "arn:aws:iam::123456789012:user/alice"

  access_scope {
    type = "cluster"
  }
}
```

### 2. Using Instance Profile ARN Instead of Role ARN

```hcl
# ❌ Wrong — this is an instance profile ARN
principal_arn = "arn:aws:iam::123456789012:instance-profile/MyNodeRole"

# ✅ Correct — use the role ARN
principal_arn = "arn:aws:iam::123456789012:role/MyNodeRole"
```

### 3. Service-Linked Roles

Service-linked roles (path `/aws-service-role/`) cannot be used as access entry principals:

```hcl
# ❌ Will fail — service-linked roles are not supported
principal_arn = "arn:aws:iam::123456789012:role/aws-service-role/eks.amazonaws.com/AWSServiceRoleForAmazonEKS"
```

### 4. Forgetting to Set Authentication Mode

If the cluster is in `CONFIG_MAP` mode, creating access entry resources will fail:

```
Error: creating EKS Access Entry: InvalidParameterException: 
Cluster authentication mode does not support access entries
```

Fix: Upgrade the authentication mode first (see "Upgrading an Existing Cluster" section above).

### 5. ARN Path Issues with Roles Created in Non-Root Path

If a role is created under a custom IAM path, use the full ARN including the path:

```hcl
# Role created with path /teams/backend/
# ✅ Use full path in the ARN
principal_arn = "arn:aws:iam::123456789012:role/teams/backend/DevRole"

# ❌ Missing the path — will not match
principal_arn = "arn:aws:iam::123456789012:role/DevRole"
```

---

## Summary

Access Entries are not strictly "the default" for EKS 1.33, but they are:
- ✅ The **recommended** approach by AWS
- ✅ Required for console visibility with proper IAM authentication
- ✅ The **replacement** for deprecated aws-auth ConfigMap
- ✅ **Fully supported** on all Kubernetes 1.31+ clusters
- ✅ Must be explicitly enabled via `authentication_mode` setting

---

## References

- [Grant IAM users access to Kubernetes with EKS access entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)
- [Change authentication mode to use access entries](https://docs.aws.amazon.com/eks/latest/userguide/setting-up-access-entries.html)
- [Migrating existing aws-auth ConfigMap entries](https://docs.aws.amazon.com/eks/latest/userguide/migrating-access-entries.html)
- [Cluster access management best practices](https://docs.aws.amazon.com/eks/latest/best-practices/cluster-access-management.html)
- [aws-auth ConfigMap deprecation notice](https://docs.aws.amazon.com/en_en/eks/latest/userguide/auth-configmap.html)
