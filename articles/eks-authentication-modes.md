# EKS Authentication Modes: ConfigMap vs Access Entries

## Overview

Amazon EKS provides two methods for granting IAM principals access to the Kubernetes API. Understanding the difference between them is essential for avoiding lockout scenarios and following AWS security best practices.

---

## Method 1: aws-auth ConfigMap (Legacy)

### How It Works

The `aws-auth` ConfigMap is a Kubernetes resource stored inside the cluster itself, in the `kube-system` namespace. It maps IAM roles and users to Kubernetes RBAC permissions.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/my-node-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/alice
      username: alice
      groups:
        - system:masters
```

### Granting Access

To grant a new user access, you must:

1. Already have `kubectl` access to the cluster
2. Edit the ConfigMap directly:

```bash
kubectl edit configmap aws-auth -n kube-system
```

### Managing the ConfigMap

**View current contents:**

```bash
kubectl get configmap aws-auth -n kube-system -o yaml
```

**Export to a file for editing:**

```bash
kubectl get cm aws-auth -n kube-system -o yaml > aws-auth.yaml
```

**Backup before editing (always do this first):**

```bash
kubectl get cm aws-auth -n kube-system -o yaml > aws-auth-backup-$(date +%F).yaml
```

**Apply the new configuration:**

```bash
kubectl apply -f aws-auth.yaml
```

**Add a new IAM role (using kubectl patch):**

```bash
kubectl patch configmap aws-auth -n kube-system --patch '
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/my-new-role
      username: my-new-role
      groups:
        - system:masters
'
```

**Or edit directly in vim:**

```bash
kubectl edit configmap aws-auth -n kube-system
```

This opens the ConfigMap in your default editor (usually vim). Make your changes, save and quit (`:wq`) to apply immediately.

**Validate the configuration:**

```bash
yamllint aws-auth.yaml

kubectl diff -f aws-auth.yaml         # this will also show multiple \n if there are any extra spaces
```

### Limitations

- **Chicken-and-egg problem:** The ConfigMap lives inside Kubernetes. To modify it, you need Kubernetes API access. If you lose access, you cannot get back in without the original cluster creator role.
- **No AWS-native tooling:** Cannot be managed via AWS Console, AWS CLI, CloudFormation, or Terraform without already having cluster access.
- **Error-prone:** Manual YAML editing with no validation — a single indentation mistake can break cluster access entirely.
- **No audit trail:** Changes to the ConfigMap are not recorded in AWS CloudTrail.
- **Deprecated:** AWS has officially deprecated this method.

### Authentication mode value

```
CONFIG_MAP
```

---

## Method 2: EKS Access Entries (Recommended)

### How It Works

Access Entries are an AWS-native API for managing Kubernetes access. They live entirely outside Kubernetes, managed through the EKS API. Each access entry associates an IAM principal (user or role) with a set of Kubernetes permissions via an Access Policy.

### Granting Access

Access can be granted using any standard AWS tool — no existing cluster access required.

**AWS CLI:**

```bash
# Create the access entry
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:user/alice \
  --type STANDARD \
  --region us-east-1

# Attach a policy
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:user/alice \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster \
  --region us-east-1
```

**List access entries:**

```bash
aws eks list-access-entries \
  --cluster-name my-cluster \
  --region us-east-1
```

**Describe an access entry:**

```bash
aws eks describe-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:user/alice \
  --region us-east-1
```

**Delete an access entry:**

```bash
aws eks delete-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:user/alice \
  --region us-east-1
```

**Terraform:**

```hcl
resource "aws_eks_access_entry" "example" {
  cluster_name  = "my-cluster"
  principal_arn = "arn:aws:iam::123456789012:user/alice"
  type          = "STANDARD"
}

resource "aws_eks_access_policy_association" "example" {
  cluster_name  = "my-cluster"
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
  principal_arn = "arn:aws:iam::123456789012:user/alice"

  access_scope {
    type = "cluster"
  }
}
```

### Available Access Policies

AWS provides managed Access Policies with predefined permission levels:

| Policy | Description |
|--------|-------------|
| `AmazonEKSClusterAdminPolicy` | Full admin access to all cluster resources |
| `AmazonEKSAdminPolicy` | Admin access excluding cluster-level settings |
| `AmazonEKSEditPolicy` | Read/write access to most resources |
| `AmazonEKSViewPolicy` | Read-only access to all resources |

### Access Scope

Permissions can be scoped at two levels:

```hcl
# Cluster-wide access
access_scope {
  type = "cluster"
}

# Namespace-scoped access
access_scope {
  type       = "namespace"
  namespaces = ["production", "staging"]
}
```

### Advantages

- **No cluster access required to grant access** — managed entirely via AWS IAM API
- **Full AWS tooling support** — Console, CLI, SDK, CloudFormation, Terraform
- **CloudTrail audit trail** — all access changes are logged
- **No lockout risk** — any IAM user with `eks:CreateAccessEntry` permission can grant access
- **Fine-grained scoping** — limit access to specific namespaces
- **Validation** — AWS validates the IAM principal ARN before creating the entry

### Authentication mode value

```
API
```

---

## Comparison

| | ConfigMap | Access Entries |
|---|---|---|
| Where access is stored | Inside Kubernetes | AWS IAM API (outside Kubernetes) |
| Requires existing cluster access to grant | ✅ Yes | ❌ No |
| AWS Console support | ❌ No | ✅ Yes |
| Terraform support | ❌ Indirect only | ✅ Native resources |
| CloudTrail audit logging | ❌ No | ✅ Yes |
| Namespace-scoped permissions | ⚠️ Manual RBAC | ✅ Built-in |
| Risk of admin lockout | ✅ High | ❌ Low |
| AWS status | ⚠️ Deprecated | ✅ Recommended |

---

## Authentication Modes

EKS clusters have three authentication mode settings that control which method is active:

| Mode | ConfigMap | Access Entries API | Use Case |
|------|-----------|-------------------|----------|
| `CONFIG_MAP` | ✅ Active | ❌ Disabled | Legacy clusters only — deprecated |
| `API_AND_CONFIG_MAP` | ✅ Active | ✅ Active | **Migration path** — supports both methods simultaneously |
| `API` | ❌ Disabled | ✅ Active | **New clusters** — Access Entries only |

### Changing Authentication Mode

The mode can only be upgraded, never downgraded:

```
CONFIG_MAP → API_AND_CONFIG_MAP → API
```

To upgrade:

```bash
aws eks update-cluster-config \
  --name my-cluster \
  --access-config authenticationMode=API_AND_CONFIG_MAP \
  --region us-east-1
```

Wait for the cluster to return to `ACTIVE` status before creating access entries:

```bash
aws eks describe-cluster \
  --name my-cluster \
  --region us-east-1 \
  --query "cluster.status"
```

---

## Migration from ConfigMap to Access Entries

If you have an existing cluster using the ConfigMap method:

1. **Upgrade the authentication mode** to `API_AND_CONFIG_MAP` (keeps existing ConfigMap entries working)
2. **Create equivalent access entries** for all users/roles currently in the ConfigMap
3. **Verify access** works via Access Entries before removing ConfigMap entries
4. **Remove ConfigMap entries** (optional — they can coexist)
5. **Upgrade to `API` mode** (optional — permanently disables ConfigMap method)

> Note: EKS-managed ConfigMap entries (for managed node groups and Fargate profiles) cannot be migrated — they are handled automatically by EKS when in `API_AND_CONFIG_MAP` mode.

---

## Platform Version Requirements

Access Entries require a minimum platform version per Kubernetes version:

| Kubernetes version | Minimum platform version |
|-------------------|--------------------------|
| 1.31 and above | All versions supported |
| 1.30 | eks.2 |
| 1.29 | eks.1 |
| 1.28 | eks.6 |

---

## Common Kubernetes Groups

| Group | Description |
|-------|-------------|
| `system:masters` | Full cluster admin — bypasses all RBAC. Use sparingly. |
| `system:nodes` | Required for worker nodes to join the cluster |
| `system:bootstrappers` | Required for node bootstrap (used with `system:nodes`) |
| `system:authenticated` | Any authenticated user (implicit) |
| `system:unauthenticated` | Anonymous access (disabled by default) |

Custom groups can be mapped to ClusterRoleBindings or RoleBindings for fine-grained access:

```yaml
# Example: bind a custom group to a ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dev-team-view
subjects:
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

---

## Troubleshooting & Lockout Recovery

### Symptoms of lockout

```
error: You must be logged in to the server (Unauthorized)
```

### Recovery options

**1. Use the cluster creator identity**

The IAM principal that created the cluster always has `system:masters` access (even if not in the ConfigMap). Authenticate as that identity:

```bash
aws eks update-kubeconfig --name my-cluster --region us-east-1 --role-arn arn:aws:iam::123456789012:role/cluster-creator-role
```

**2. If using API_AND_CONFIG_MAP or API mode — create an access entry from outside**

No cluster access needed:

```bash
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:user/your-user \
  --type STANDARD \
  --region us-east-1

aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:user/your-user \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster \
  --region us-east-1
```

**3. Restore from backup**

If you have a backup of the ConfigMap:

```bash
kubectl apply -f aws-auth-backup-2025-01-15.yaml
```

(Requires access via the cluster creator identity first.)

**4. Contact AWS Support**

As a last resort, AWS Support can help restore access to the cluster.

### Prevention

- Always keep a backup of `aws-auth` before editing
- Never remove the last `system:masters` entry
- Prefer Access Entries over ConfigMap — eliminates lockout risk entirely
- Keep the cluster creator IAM role available and documented

---

## References

- [Grant IAM users access to Kubernetes with EKS access entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)
- [Grant IAM users and roles access to Kubernetes APIs](https://docs.aws.amazon.com/eks/latest/userguide/grant-k8s-access.html)
- [aws-auth ConfigMap (deprecated)](https://docs.aws.amazon.com/en_en/eks/latest/userguide/auth-configmap.html)
- [Change authentication mode to use access entries](https://docs.aws.amazon.com/eks/latest/userguide/setting-up-access-entries.html)
- [Migrating existing aws-auth ConfigMap entries](https://docs.aws.amazon.com/eks/latest/userguide/migrating-access-entries.html)
- [Cluster access management best practices](https://docs.aws.amazon.com/eks/latest/best-practices/cluster-access-management.html)
