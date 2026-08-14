# Adding Users to EKS via aws-auth ConfigMap

## Prerequisites

You must already have `kubectl` access to the cluster. If you don't have access, you'll need to use the same IAM role that created the cluster.

---

## Step 1: Configure kubectl

Get the kubeconfig for your cluster:

```bash
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```

Verify access:

```bash
kubectl get nodes
```

If this fails with "Unauthorized" or "access denied", you don't have access and need to authenticate as the cluster creator role.

---

## Step 2: Get Your IAM ARN

Find the IAM user or role ARN you want to add:

```bash
# For your current identity
aws sts get-caller-identity

# Example output:
{
  "UserId": "AIDACKCEVSQ6C2EXAMPLE",
  "Account": "717748260869",
  "Arn": "arn:aws:iam::717748260869:user/alice"
}
```

**Important:** If the ARN shows `arn:aws:sts::...assumed-role/...`, you need to convert it to the IAM role ARN:

- STS ARN: `arn:aws:sts::717748260869:assumed-role/AdminRole/session-name`
- IAM ARN: `arn:aws:iam::717748260869:role/AdminRole` ← use this

---

## Step 3: Edit the aws-auth ConfigMap

```bash
kubectl edit configmap aws-auth -n kube-system
```

This opens the ConfigMap in your default editor (usually `vi` or `nano`).

---

## Step 4: Add Your User/Role

### For an IAM User

Add to the `mapUsers` section:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    # Existing roles (managed node groups, etc.) - DO NOT REMOVE
    - rolearn: arn:aws:iam::717748260869:role/eks-node-group-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    # ADD YOUR USER HERE
    - userarn: arn:aws:iam::717748260869:user/alice
      username: alice
      groups:
        - system:masters
```

### For an IAM Role

Add to the `mapRoles` section:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    # Existing roles (managed node groups, etc.) - DO NOT REMOVE
    - rolearn: arn:aws:iam::717748260869:role/eks-node-group-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    # ADD YOUR ROLE HERE
    - rolearn: arn:aws:iam::717748260869:role/AdminRole
      username: admin-user
      groups:
        - system:masters
```

---

## Step 5: Save and Exit

- In `vi`: Press `Esc`, then type `:wq` and press Enter
- In `nano`: Press `Ctrl+O`, then `Ctrl+X`

---

## Step 6: Verify

Wait 10-15 seconds for the changes to propagate, then test with the AWS Console or with `kubectl` using the new identity:

```bash
# Verify the ConfigMap was updated
kubectl get configmap aws-auth -n kube-system -o yaml

# Test access with the new identity
aws eks update-kubeconfig --name demo-cluster --region us-east-1
kubectl get nodes
```

---

## Common Permission Groups

| Group | Permissions |
|-------|-------------|
| `system:masters` | Full cluster admin access (equivalent to Kubernetes cluster-admin) |
| `system:bootstrappers` | Required for worker nodes to join the cluster |
| `system:nodes` | Required for kubelet on worker nodes |
| Custom RBAC | Create your own RoleBinding/ClusterRoleBinding and reference it here |

For fine-grained permissions, create a Kubernetes `ClusterRole` or `Role` and bind it to the user:

```yaml
- userarn: arn:aws:iam::717748260869:user/developer
  username: developer
  groups:
    - developers  # Reference a ClusterRoleBinding or RoleBinding
```

Then create the RBAC resources:

```bash
kubectl create clusterrolebinding developers \
  --clusterrole=view \
  --group=developers
```

---

## Troubleshooting

### "Unauthorized" after editing

- Wait 10-15 seconds for changes to propagate
- Check for YAML syntax errors: `kubectl get configmap aws-auth -n kube-system -o yaml`
- Verify the ARN format (IAM ARN, not STS assumed-role ARN)

### "error: You must be logged in to the server (Unauthorized)"

- You don't have access. Use the cluster creator's IAM role
- Check your `~/.kube/config` points to the correct cluster

### ConfigMap doesn't exist

If the ConfigMap is missing entirely, create it:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::717748260869:role/eks-node-group-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::717748260869:user/YOUR_USER
      username: YOUR_USER
      groups:
        - system:masters
EOF
```

**Replace the role ARN and user ARN with your actual values.**

---

## Important Notes

- **Do not remove existing entries** for managed node groups or Fargate profiles — this will break your cluster
- YAML indentation matters — use 2 spaces, not tabs
- Changes propagate within ~10-15 seconds
- The aws-auth ConfigMap is **deprecated** by AWS — consider migrating to [Access Entries](articles/eks-access-entries.md) when possible

---

## References

- [AWS Documentation: aws-auth ConfigMap](https://docs.aws.amazon.com/en_en/eks/latest/userguide/auth-configmap.html)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
