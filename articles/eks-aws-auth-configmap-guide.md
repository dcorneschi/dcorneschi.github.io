# EKS aws-auth ConfigMap Guide

Managing IAM role and user access to EKS clusters via the aws-auth ConfigMap — editing, recovery, validation, and best practices.

## Overview

The `aws-auth` ConfigMap in the `kube-system` namespace maps AWS IAM roles and users to Kubernetes RBAC groups. It controls who can authenticate to your EKS cluster.

## View Current ConfigMap

```bash
kubectl get configmap aws-auth -n kube-system -o yaml
```

## Backup First (Always)

```bash
kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth-backup.yaml
```

## Adding an IAM Role

### Method 1: Edit Directly

```bash
kubectl edit configmap aws-auth -n kube-system
```

Add your role to the `mapRoles` section:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    # Existing node instance role (don't remove this)
    - rolearn: arn:aws:iam::123456789012:role/NodeInstanceRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes

    # Your new IAM role (add this)
    - rolearn: arn:aws:iam::123456789012:role/YOUR-NEW-ROLE-NAME
      username: your-username
      groups:
        - system:masters
```

### Method 2: Patch ConfigMap

```bash
kubectl patch configmap aws-auth -n kube-system --patch '{"data":{"mapRoles":"- rolearn: arn:aws:iam::123456789012:role/ROLE_NAME\n  username: your-username\n  groups:\n  - system:masters\n"}}'
```

### Method 3: Using eksctl

```bash
eksctl create iamidentitymapping \
  --cluster CLUSTER_NAME \
  --region REGION \
  --arn arn:aws:iam::123456789012:role/ROLE_NAME \
  --username your-username \
  --group system:masters
```

### Verify Changes

```bash
kubectl describe configmap aws-auth -n kube-system
```

## Key Fields

| Field | Description |
|-------|-------------|
| `rolearn` | Full ARN of the IAM role |
| `userarn` | Full ARN of the IAM user (in mapUsers section) |
| `username` | Kubernetes username for this identity |
| `groups` | Kubernetes RBAC groups to assign |

## Common Groups

| Group | Description |
|-------|-------------|
| `system:masters` | Full cluster admin access |
| `system:authenticated` | Basic authenticated user |
| `system:nodes` | Node access (for worker nodes) |
| `system:bootstrappers` | Bootstrap access (for worker nodes) |
| Custom RBAC groups | Groups you've created with specific permissions |

## Complete Example Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    # EKS Worker Node Role
    - rolearn: arn:aws:iam::123456789012:role/eksctl-my-cluster-nodegroup-NodeInstanceRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes

    # Admin Role
    - rolearn: arn:aws:iam::123456789012:role/EKS-Admin-Role
      username: admin-user
      groups:
        - system:masters

    # Developer Role
    - rolearn: arn:aws:iam::123456789012:role/EKS-Developer-Role
      username: developer
      groups:
        - developers

  mapUsers: |
    # Individual IAM users (if needed)
    - userarn: arn:aws:iam::123456789012:user/admin
      username: admin
      groups:
        - system:masters
```

## Can aws-auth Be Edited in AWS Console?

No. The `aws-auth` ConfigMap is a Kubernetes resource, not an AWS resource.

**However, you can:**
- **View it** in EKS Console → Resources tab → ConfigMaps in `kube-system` (read-only)
- **Use EKS Access Entries** (recommended for new clusters) via EKS Console → Access tab → IAM access entries

**To edit, you must use:** `kubectl`, `eksctl`, Terraform, or other Kubernetes API clients.

## What Happens If aws-auth Breaks

A broken `aws-auth` ConfigMap can lock out **ALL IAM roles** except one:

### What Remains Accessible

The **IAM principal (user/role) that created the cluster** has permanent, built-in access that bypasses `aws-auth`.

### What Gets Locked Out

- All other IAM roles/users defined in `aws-auth`
- Worker nodes (if their role mapping is broken)
- Any service accounts or applications using IAM roles

### Common Ways It Breaks

1. YAML syntax errors (invalid formatting)
2. Deleting the ConfigMap entirely
3. Removing all role mappings (empty ConfigMap)
4. Typos in role ARNs (roles won't match)

### Finding the Cluster Creator

```bash
# Check CloudTrail for the CreateCluster API call
# That IAM principal is your emergency access
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateCluster \
  --query 'Events[].{User:Username,Time:EventTime}' --output table
```

## Recovery: If aws-auth Is Broken

### If You Have a Backup

```bash
kubectl apply -f aws-auth-backup.yaml
```

### Option 1: Use Cluster Creator Credentials

```bash
# Switch to the IAM identity that created the cluster
aws eks update-kubeconfig --name CLUSTER_NAME --region REGION

# Then fix the ConfigMap
kubectl apply -f aws-auth-backup.yaml
# or manually edit it
kubectl edit configmap aws-auth -n kube-system
```

### Option 2: Delete and Recreate

```bash
kubectl delete configmap aws-auth -n kube-system
# Then recreate with correct structure
kubectl apply -f aws-auth-correct.yaml
```

### Option 3: Using eksctl

```bash
eksctl create iamidentitymapping \
  --cluster CLUSTER_NAME \
  --region REGION \
  --arn arn:aws:iam::123456789012:role/ROLE_NAME \
  --username your-username \
  --group system:masters
```

### Recovery for Terraform-Created Clusters

The "cluster creator" is the IAM role/user that executed `terraform apply`.

```bash
# 1. Assume the IAM role/user that ran terraform apply
# 2. Update your kubeconfig
aws eks update-kubeconfig --name CLUSTER_NAME --region REGION

# 3. Fix the aws-auth ConfigMap
kubectl apply -f aws-auth-backup.yaml
```

If `aws-auth` is managed by Terraform, re-run:

```bash
terraform apply
```

## Validating aws-auth YAML Locally

### YAML Syntax Check

```bash
# Using Python
python3 -c "import yaml; yaml.safe_load(open('aws-auth.yaml'))" && echo "Valid YAML" || echo "Invalid YAML"

# Using yq
yq eval '.' aws-auth.yaml > /dev/null && echo "Valid YAML" || echo "Invalid YAML"
```

### Kubernetes Dry-Run Validation

```bash
# Validate against Kubernetes API without applying
kubectl apply --dry-run=client -f aws-auth.yaml

# Server-side validation (more thorough)
kubectl apply --dry-run=server -f aws-auth.yaml
```

### Check Required Fields

```bash
yq eval '.metadata.name' aws-auth.yaml        # Should be "aws-auth"
yq eval '.metadata.namespace' aws-auth.yaml   # Should be "kube-system"
yq eval '.kind' aws-auth.yaml                 # Should be "ConfigMap"
yq eval '.data.mapRoles' aws-auth.yaml        # Should contain role mappings
```

### Validate ARN Format

```bash
# Check if ARNs follow correct format (12-digit account ID, role or user)
grep -E "arn:aws:iam::[0-9]{12}:(role|user)/" aws-auth.yaml
```

### Validation Script

```bash
#!/bin/bash
# validate-aws-auth.sh

YAML_FILE="${1:-aws-auth.yaml}"

if [ ! -f "$YAML_FILE" ]; then
    echo "File not found: $YAML_FILE"
    exit 1
fi

echo "Validating $YAML_FILE..."

# YAML syntax check
if python3 -c "import yaml; yaml.safe_load(open('$YAML_FILE'))" 2>/dev/null; then
    echo "  YAML syntax is valid"
else
    echo "  Invalid YAML syntax"
    exit 1
fi

# Check required fields
NAME=$(yq eval '.metadata.name' "$YAML_FILE" 2>/dev/null)
NAMESPACE=$(yq eval '.metadata.namespace' "$YAML_FILE" 2>/dev/null)
KIND=$(yq eval '.kind' "$YAML_FILE" 2>/dev/null)

[ "$NAME" = "aws-auth" ] && echo "  metadata.name: aws-auth" || echo "  ERROR: metadata.name should be 'aws-auth', got: $NAME"
[ "$NAMESPACE" = "kube-system" ] && echo "  metadata.namespace: kube-system" || echo "  ERROR: namespace should be 'kube-system', got: $NAMESPACE"
[ "$KIND" = "ConfigMap" ] && echo "  kind: ConfigMap" || echo "  ERROR: kind should be 'ConfigMap', got: $KIND"

# Check ARN format
if grep -qE "arn:aws:iam::[0-9]{12}:(role|user)/" "$YAML_FILE"; then
    echo "  Found valid ARN format"
else
    echo "  WARNING: No valid ARN format found"
fi

echo "Validation complete!"
```

## Common Validation Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Invalid YAML syntax | Mixed tabs and spaces | Use spaces only (2-space indent) |
| Missing pipe character | Multiline string not marked | Add `|` after `mapRoles:` |
| ARN format error | Wrong account ID or missing path | Must be `arn:aws:iam::<12-digits>:role/<name>` |
| ConfigMap not applied | Wrong namespace | Must be `kube-system` |
| Nodes disconnecting | Node role mapping removed | Always keep the NodeInstanceRole entry |

## Important Notes

- **Never remove the node instance role** — this breaks worker node communication with the API server
- **Always backup before editing** — one syntax error can lock everyone out
- **Test with non-critical roles first** — verify access before adding production roles
- **Consider EKS Access Entries** — the newer, AWS-managed alternative to aws-auth (doesn't require ConfigMap editing)
- **system:masters bypasses RBAC** — use custom groups with proper Roles/ClusterRoles for least-privilege access
