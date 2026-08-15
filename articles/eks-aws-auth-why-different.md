# Why aws-auth ConfigMap Looks Different on Different Clusters

The `aws-auth` ConfigMap in `kube-system` controls which IAM identities can access your EKS cluster. But if you look at it across clusters, the fields vary — some have `mapRoles` only, some have `mapUsers`, some have both, and some clusters don't have the ConfigMap at all. Here's why.

## What aws-auth Contains

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/NodeInstanceRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/admin
      username: admin
      groups:
        - system:masters
  mapAccounts: |
    - "123456789012"
```

Three possible fields:

| Field | Purpose | When It Appears |
|-------|---------|-----------------|
| `mapRoles` | Maps IAM roles to Kubernetes users/groups | Always (node roles must be here) |
| `mapUsers` | Maps IAM users to Kubernetes users/groups | Only if someone added IAM users |
| `mapAccounts` | Allows all IAM identities in an account | Rare, broad access |

## Why It Varies Between Clusters

### Scenario 1: Managed Node Groups (eksctl or Console)

When you create a cluster with `eksctl` or the console with managed node groups, EKS automatically creates the `aws-auth` ConfigMap with the node role:

```yaml
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/eksctl-my-cluster-nodegroup-NodeInstanceRole-XXXXX
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
```

Only `mapRoles` exists. No `mapUsers`, no `mapAccounts`.

### Scenario 2: Self-Managed Nodes

If you created the cluster without node groups (`--without-nodegroup`) and then launched self-managed EC2 instances, you must manually create or edit `aws-auth` to add the node role. The ConfigMap might not exist at all until you create it.

### Scenario 3: eksctl with IAM Identity Mappings

If you used `eksctl create iamidentitymapping` to add users or roles:

```yaml
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/NodeInstanceRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::123456789012:role/AdminRole
      username: admin
      groups:
        - system:masters
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/john
      username: john
      groups:
        - system:masters
```

Both `mapRoles` and `mapUsers` exist because both were explicitly added.

### Scenario 4: Multiple Node Groups

Each node group gets its own role entry. A cluster with 3 node groups has 3 entries in `mapRoles`:

```yaml
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/NodeRole-workers-az-a
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::123456789012:role/NodeRole-workers-az-b
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::123456789012:role/NodeRole-gpu
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
```

If all node groups share the same IAM role, there's only one entry.

### Scenario 5: Fargate Profiles

Fargate profiles add their pod execution role to `mapRoles` automatically:

```yaml
- rolearn: arn:aws:iam::123456789012:role/eks-fargate-pod-execution-role
  username: system:node:{{SessionName}}
  groups:
    - system:bootstrappers
    - system:nodes
    - system:node-proxier
```

### Scenario 6: Access Entries (Newer Clusters)

Clusters created with EKS platform version `eks.6` or later (K8s 1.23+) can use **Access Entries** instead of `aws-auth`. In this case:

- The `aws-auth` ConfigMap might only have node roles (or might not exist)
- User/role access is managed via the AWS API (`aws eks create-access-entry`)
- The ConfigMap becomes less important over time

```sh
# Check cluster authentication mode
aws eks describe-cluster --name <cluster> \
  --query "cluster.accessConfig.authenticationMode" --output text

# Possible values:
# CONFIG_MAP         — only aws-auth (legacy)
# API                — only Access Entries (no aws-auth needed)
# API_AND_CONFIG_MAP — both work (migration mode)
```

### Scenario 7: Cluster Creator Isn't in aws-auth

The IAM identity that created the cluster gets implicit `system:masters` access — it's NOT listed in `aws-auth`. This is why the creator can access the cluster but nobody else can (until they're added).

If the creator is an IAM role that gets deleted or a temporary session, you can lose access to the cluster.

## How the ConfigMap Gets Created

| Creation Method | Who Creates aws-auth | Initial Contents |
|----------------|---------------------|------------------|
| `eksctl create cluster` (with nodes) | eksctl | Node role in `mapRoles` |
| `eksctl create cluster --without-nodegroup` | Nobody (doesn't exist yet) | Empty / missing |
| AWS Console (with managed nodes) | EKS service | Node role in `mapRoles` |
| AWS CLI `create-cluster` + `create-nodegroup` | EKS service | Node role in `mapRoles` |
| Terraform `aws_eks_node_group` | EKS service | Node role in `mapRoles` |
| Self-managed ASG | You (manually) | Whatever you put in |

## The Ordering Matters

Fields in `aws-auth` are evaluated in order. If a role matches multiple entries, the **first match wins**. This is rarely an issue but can cause confusion if you have duplicate role ARNs with different groups.

## Common Mistakes That Cause Differences

### Different Node Role Per Cluster

```yaml
# Cluster A (eksctl-created)
- rolearn: arn:aws:iam::123456789012:role/eksctl-clusterA-nodegroup-NodeInstanceRole-ABC123

# Cluster B (Terraform-created)
- rolearn: arn:aws:iam::123456789012:role/EKSNodeRole-clusterB
```

Different tools name the IAM role differently. The role ARN must match exactly.

### Missing mapUsers Field

```yaml
# Cluster with only node access (no user mappings):
data:
  mapRoles: |
    - rolearn: ...

# Cluster where users were added:
data:
  mapRoles: |
    - rolearn: ...
  mapUsers: |
    - userarn: ...
```

`mapUsers` only appears if someone explicitly adds it. It's not created by default.

### Trailing Whitespace or YAML Indentation

The `data` fields are strings (pipe `|`), not structured YAML. A YAML indentation error inside the string won't cause a parsing error but will cause a silent auth failure:

```yaml
# Correct
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/MyRole
      username: admin
      groups:
        - system:masters

# WRONG (extra space before rolearn)
data:
  mapRoles: |
     - rolearn: arn:aws:iam::123456789012:role/MyRole
       username: admin
       groups:
         - system:masters
```

## Inspecting aws-auth

```sh
# View current state
kubectl get configmap aws-auth -n kube-system -o yaml

# Pretty-print with eksctl
eksctl get iamidentitymapping --cluster <cluster> --region <region>

# Check if it exists at all
kubectl get configmap aws-auth -n kube-system 2>/dev/null && echo "EXISTS" || echo "MISSING"
```

## Migrating to Access Entries

Access Entries are the newer, recommended replacement for `aws-auth`. They're managed via the AWS API (not a ConfigMap that can be accidentally deleted):

```sh
# 1. Enable API_AND_CONFIG_MAP mode (supports both during migration)
aws eks update-cluster-config --name <cluster> --region <region> \
  --access-config authenticationMode=API_AND_CONFIG_MAP

# 2. Create access entries for all users/roles currently in aws-auth
aws eks create-access-entry --cluster-name <cluster> --region <region> \
  --principal-arn arn:aws:iam::123456789012:role/AdminRole \
  --type STANDARD

aws eks associate-access-policy --cluster-name <cluster> --region <region> \
  --principal-arn arn:aws:iam::123456789012:role/AdminRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster

# 3. Verify access works via Access Entries
# 4. Remove entries from aws-auth (optional)
# 5. Switch to API-only mode (optional, irreversible)
aws eks update-cluster-config --name <cluster> --region <region> \
  --access-config authenticationMode=API
```

> Once you switch to `API` mode, you cannot go back to `CONFIG_MAP`. Use `API_AND_CONFIG_MAP` during migration.

## Quick Reference

```sh
# View aws-auth
kubectl get cm aws-auth -n kube-system -o yaml

# Add a role (eksctl)
eksctl create iamidentitymapping --cluster <cluster> --region <region> \
  --arn <role-arn> --group system:masters --username admin

# Add a role (kubectl)
kubectl edit cm aws-auth -n kube-system

# Check authentication mode
aws eks describe-cluster --name <cluster> --query "cluster.accessConfig.authenticationMode" --output text

# List access entries
aws eks list-access-entries --cluster-name <cluster> --region <region>

# Backup aws-auth before editing
kubectl get cm aws-auth -n kube-system -o yaml > aws-auth-backup.yaml
```

## Gotchas

- **Deleting aws-auth locks you out**: If you accidentally delete it and you're not the cluster creator, you lose access. Always backup before editing.
- **Cluster creator has implicit access**: The identity that created the cluster is NOT in aws-auth — it has hardcoded `system:masters` access. This can't be revoked via aws-auth.
- **Role ARN must be exact**: `arn:aws:iam::123456789012:role/my-role` ≠ `arn:aws:iam::123456789012:role/path/my-role`. Paths matter.
- **Assume-role sessions use the role ARN**: If a user assumes a role, the role ARN goes in `mapRoles`, not the user ARN.
- **mapUsers requires the user ARN, not the role**: If a user calls the API directly (not via assume-role), their user ARN goes in `mapUsers`.
- **EKS doesn't validate entries**: You can add invalid ARNs without errors — authentication just silently fails.
- **Managed node groups auto-add their role**: If you delete a node role entry and then update the node group, EKS adds it back.


## Why Field Order Differs (rolearn vs groups first)

You might notice that some clusters have `-rolearn` appearing before `-groups` in each entry, while others have `-groups` first. This is purely cosmetic:

### Why It Happens

1. **YAML is order-preserving** — whatever tool creates the ConfigMap writes entries in a specific order, and that order is maintained
2. **Different tools have different conventions**:
   - `eksctl` writes `-rolearn` first
   - Terraform providers may write `-groups` first depending on how the resource is structured
   - Manual edits follow whatever order the admin chose
3. **AWS doesn't enforce an ordering** — there's no standard for field order within `mapRoles`/`mapUsers` entries

### It Doesn't Matter Functionally

The aws-iam-authenticator webhook reads all fields in each entry regardless of their order. These two are equivalent:

```yaml
# Style A (eksctl default)
- rolearn: arn:aws:iam::123456789012:role/MyRole
  username: admin
  groups:
    - system:masters

# Style B (Terraform / manual)
- groups:
    - system:masters
  rolearn: arn:aws:iam::123456789012:role/MyRole
  username: admin
```

Both authenticate identically. The difference reflects the history of how each cluster's ConfigMap was created and managed — not a functional distinction.
