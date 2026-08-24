# Deep Dive: system:masters Group in Kubernetes on EKS

How `system:masters` works, why it bypasses RBAC, who gets it on EKS, and why you should avoid assigning it.

## What is system:masters?

`system:masters` is a built-in Kubernetes group that grants **unrestricted, irrevocable cluster-admin access**. It's hardcoded into the API server — no ClusterRoleBinding needed.

```
system:masters → cluster-admin (built-in binding) → ALL verbs on ALL resources
```

Any identity (user, service account, IAM role) mapped to `system:masters` can do anything in the cluster with no way to restrict it via RBAC.

## Why It Bypasses RBAC

The Kubernetes API server has a special authorizer for `system:masters`:

```go
// In the API server authorization chain:
// 1. Check if user is in system:masters group
//    → YES: Allow immediately (skip all other authorizers)
//    → NO: Continue to RBAC, webhook, etc.
```

Key properties:
- **Cannot be restricted** — no RoleBinding, NetworkPolicy, or admission webhook can limit what `system:masters` identities do
- **Cannot be audited differently** — same audit logging applies, but you can't block their actions
- **Cannot be overridden** — even if you delete the `cluster-admin` ClusterRoleBinding, the API server still grants full access
- **Survives RBAC misconfigurations** — it's a safety net for cluster recovery

## system:masters on EKS

### Who Gets system:masters by Default?

On EKS, only the **cluster creator** (the IAM principal that called `eks:CreateCluster`) is automatically granted `system:masters`:

```bash
# The IAM user/role that created the cluster gets:
# - system:masters group membership
# - Invisible to kubectl (not in aws-auth ConfigMap)
# - Cannot be removed without cluster recreation
```

This identity doesn't appear in `aws-auth` or Access Entries — it's implicitly mapped by the EKS control plane.

### How to Check Who Has system:masters

```bash
# Check aws-auth ConfigMap for system:masters mappings
kubectl get configmap aws-auth -n kube-system -o yaml | grep -B2 "system:masters"

# Check EKS Access Entries for cluster-admin access
aws eks list-access-entries --cluster-name <cluster>
aws eks list-associated-access-policies --cluster-name <cluster> --principal-arn <arn>

# Check who created the cluster (implicit system:masters)
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateCluster \
  --query 'Events[?contains(Resources[].ResourceName, `<cluster-name>`)].Username'
```

### aws-auth ConfigMap Mapping

When you map an IAM role to `system:masters` in aws-auth:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/AdminRole
      username: admin
      groups:
        - system:masters    # ← Full unrestricted access
```

This gives `AdminRole` **god mode** in the cluster — equivalent to root.

### EKS Access Entries (Newer Method)

With EKS Access Entries, you can grant cluster-admin via the `AmazonEKSClusterAdminPolicy`:

```bash
# Create access entry with cluster-admin (equivalent to system:masters)
aws eks create-access-entry --cluster-name <cluster> \
  --principal-arn arn:aws:iam::123456789012:role/AdminRole

aws eks associate-access-policy --cluster-name <cluster> \
  --principal-arn arn:aws:iam::123456789012:role/AdminRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

`AmazonEKSClusterAdminPolicy` maps to `system:masters` group internally.

## Why You Should Avoid system:masters

### No Guardrails

```bash
# A system:masters user can:
kubectl delete namespace production              # Delete production
kubectl delete clusterrolebinding <anything>     # Remove all RBAC
kubectl delete validatingwebhookconfig <policy>  # Disable policy engines
kubectl exec -it <pod> -- rm -rf /               # Destroy running containers
kubectl delete node <all-nodes>                  # Remove all compute
```

No admission webhook (OPA/Gatekeeper, Kyverno), no ValidatingAdmissionPolicy, and no RBAC rule can prevent this.

### Blast Radius

If a `system:masters` credential is compromised:
- Attacker has full cluster control
- Can exfiltrate all secrets (including other credentials)
- Can create persistent backdoor access
- Can disable all monitoring and auditing
- Cannot be contained without revoking the IAM credential itself

### Audit Complexity

All `system:masters` actions succeed — you can only detect misuse after the fact through CloudTrail and Kubernetes audit logs.

## What to Use Instead

### For Human Administrators

Use scoped RBAC with specific permissions:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-admins
subjects:
- kind: Group
  name: platform-admins
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin      # Still powerful, but CAN be restricted by webhooks
  apiGroup: rbac.authorization.k8s.io
```

The key difference: `cluster-admin` ClusterRole via RBAC binding **can** be overridden by admission webhooks. `system:masters` group **cannot**.

### For EKS Access Entries

Use `AmazonEKSAdminPolicy` instead of `AmazonEKSClusterAdminPolicy`:

```bash
# Admin (not cluster-admin) — still very powerful but can be webhook-limited
aws eks associate-access-policy --cluster-name <cluster> \
  --principal-arn arn:aws:iam::123456789012:role/AdminRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSAdminPolicy \
  --access-scope type=cluster
```

### For CI/CD and Automation

Use namespace-scoped roles:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deploy-bot
  namespace: production
subjects:
- kind: Group
  name: deploy-bots
roleRef:
  kind: ClusterRole
  name: edit
```

### For Break-Glass / Emergency Access

Keep one `system:masters` identity as a break-glass credential, stored securely:

```bash
# Create a dedicated IAM role for emergencies
aws iam create-role --role-name EKS-BreakGlass \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:root"},
      "Action": "sts:AssumeRole",
      "Condition": {"Bool": {"aws:MultiFactorAuthPresent": "true"}}
    }]
  }'
```

Map it in aws-auth with `system:masters` but protect the role with:
- MFA requirement
- CloudTrail alerts on AssumeRole
- SCPs limiting who can assume it
- No long-lived credentials

## system:masters vs cluster-admin ClusterRole

| Aspect | system:masters Group | cluster-admin ClusterRole (via binding) |
|--------|---------------------|----------------------------------------|
| How it works | Hardcoded in API server | RBAC ClusterRoleBinding |
| Can admission webhooks block it? | **No** | **Yes** |
| Can you delete it? | No (hardcoded) | Yes (delete the binding) |
| Visible in RBAC? | Only via built-in binding | Yes, explicit binding |
| Affected by namespace restrictions? | No | No (cluster-scoped) |
| Can OPA/Gatekeeper/Kyverno limit it? | **No** | **Yes** |
| Recovery tool? | Yes — always works | No — can be broken by bad RBAC |
| Recommended for humans? | **No** (break-glass only) | Yes (with webhook guardrails) |

## The Built-In ClusterRoleBinding

Kubernetes automatically creates this binding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:masters
```

Even if you delete this binding, `system:masters` still works — the API server's built-in authorizer handles it before RBAC is consulted.

## Identifying system:masters Usage

### Audit Who Has It

```bash
# From aws-auth
kubectl get configmap aws-auth -n kube-system -o json | \
  jq -r '.data.mapRoles' | grep -B5 "system:masters"

kubectl get configmap aws-auth -n kube-system -o json | \
  jq -r '.data.mapUsers' | grep -B5 "system:masters"

# From Access Entries
for arn in $(aws eks list-access-entries --cluster-name <cluster> --query 'accessEntries' --output text); do
  policies=$(aws eks list-associated-access-policies --cluster-name <cluster> --principal-arn "$arn" \
    --query 'associatedAccessPolicies[].policyArn' --output text)
  if echo "$policies" | grep -q "ClusterAdminPolicy"; then
    echo "CLUSTER-ADMIN: $arn"
  fi
done
```

### Monitor system:masters Activity

CloudTrail logs all API server calls. Filter for `system:masters` activity:

```bash
# Kubernetes audit logs (if sent to CloudWatch)
aws logs filter-log-events \
  --log-group-name /aws/eks/<cluster>/cluster \
  --filter-pattern '{ $.user.groups[0] = "system:masters" }'
```

### Alert on Unexpected system:masters Usage

Set up CloudWatch alarms for any `system:masters` API calls outside break-glass scenarios.

## Migration: Remove system:masters from Regular Users

### Step 1: Create Proper RBAC

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-team
subjects:
- kind: Group
  name: admin-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### Step 2: Update aws-auth (Replace system:masters with custom group)

```yaml
mapRoles: |
  - rolearn: arn:aws:iam::123456789012:role/AdminRole
    username: admin:{{SessionName}}
    groups:
      - admin-team           # Custom group with RBAC binding
      # - system:masters     # REMOVED
```

### Step 3: Verify Access Still Works

```bash
# Assume the role and test
aws sts assume-role --role-arn arn:aws:iam::123456789012:role/AdminRole --role-session-name test
kubectl auth can-i '*' '*' --all-namespaces
# Should return "yes" (via cluster-admin RBAC binding)
```

### Step 4: Keep Break-Glass Role

Ensure one dedicated emergency role retains `system:masters` with proper protection (MFA, alerts, limited principals).

## Best Practices

- **Break-glass only** — only one or two emergency IAM roles should have `system:masters`
- **Protect with MFA** — require MFA to assume the break-glass role
- **Alert on use** — CloudTrail alarm for any AssumeRole call to the break-glass role
- **Use cluster-admin RBAC** for daily admin work — it's still powerful but can be guarded by webhooks
- **Audit regularly** — scan aws-auth and Access Entries for `system:masters` / ClusterAdminPolicy
- **Rotate cluster creator** — if the cluster creator leaves the org, you can't remove their implicit access without recreating the cluster (or migrating to Access Entries which override it)
- **Document who has it** — maintain a runbook of all `system:masters` identities and why they need it

### MFA Policy for Break-Glass Role

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::123456789012:role/EKS-BreakGlass",
    "Condition": {
      "Bool": {"aws:MultiFactorAuthPresent": "true"},
      "NumericLessThan": {"aws:MultiFactorAuthAge": "3600"}
    }
  }]
}
```

This requires MFA and limits the session to 1 hour after MFA authentication.

### Custom Admin ClusterRole (Scoped Alternative)

Instead of `cluster-admin`, create a role with specific exclusions:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: platform-admin
rules:
- apiGroups: ["", "apps", "batch", "extensions", "networking.k8s.io", "autoscaling"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles", "rolebindings"]
  verbs: ["*"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles", "clusterrolebindings"]
  verbs: ["get", "list", "watch"]   # Can view but not modify cluster RBAC
```

This gives broad admin access but prevents modifying cluster-level RBAC — reducing the blast radius of a compromised credential.

## When to Use system:masters

### Appropriate

- Cluster bootstrapping (initial setup)
- Emergency break-glass access
- Cluster migration and disaster recovery
- Platform team leads (1-2 trusted individuals max)
- Recovering from broken RBAC configuration

### Avoid

- Regular development work
- Application deployments
- CI/CD pipelines
- Monitoring and observability access
- Team-level access
- Any automation that runs frequently

## Quick Reference

```bash
# Find system:masters in aws-auth
kubectl get cm aws-auth -n kube-system -o yaml | grep -B5 "system:masters"

# Find cluster-admin Access Entries
aws eks list-access-entries --cluster-name <cluster>

# Check your own groups
kubectl auth whoami    # (Kubernetes 1.28+)

# Test if you have unrestricted access
kubectl auth can-i '*' '*' --all-namespaces

# View the built-in cluster-admin binding
kubectl get clusterrolebinding cluster-admin -o yaml

# Audit Kubernetes API calls by system:masters
aws logs filter-log-events \
  --log-group-name /aws/eks/<cluster>/cluster \
  --filter-pattern '{ $.user.groups[0] = "system:masters" }'
```
