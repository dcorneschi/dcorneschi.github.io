# aws-auth ConfigMap Technical Details

How EKS authentication actually works at the protocol level — from `kubectl` to the API server, through STS token exchange and the aws-iam-authenticator webhook.

## The Authentication Flow

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         EKS Authentication Flow                              │
│                                                                              │
│  1. kubectl reads ~/.kube/config                                             │
│  2. Finds exec credential plugin: "aws eks get-token"                        │
│  3. aws eks get-token creates a pre-signed STS GetCallerIdentity URL         │
│  4. Base64-encodes the URL → returns as a bearer token                       │
│  5. kubectl sends the token in the Authorization header to the API server    │
│  6. API server passes the token to the aws-iam-authenticator webhook         │
│  7. Webhook decodes the token → calls STS GetCallerIdentity using the URL    │
│  8. STS returns the caller's IAM identity (ARN, account, user ID)            │
│  9. Webhook looks up the IAM ARN in aws-auth ConfigMap (or Access Entries)   │
│  10. Maps IAM identity → Kubernetes username + groups                        │
│  11. Returns the authenticated identity to the API server                    │
│  12. API server evaluates RBAC (is this user authorized for this request?)   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

Key insight: **STS GetCallerIdentity is never actually executed by the client**. The client only creates a pre-signed URL. The webhook is the one that calls STS to validate the identity.

## The Token Format

EKS tokens look like:

```
k8s-aws-v1.aHR0cHM6Ly9zdHMuYW1hem9uYXdzLmNvbS8_QWN0aW9u...
```

Structure:
- Prefix: `k8s-aws-v1.`
- Payload: Base64-encoded pre-signed STS GetCallerIdentity URL

```sh
# Generate a token
aws eks get-token --cluster-name my-cluster

# Output:
{
  "kind": "ExecCredential",
  "apiVersion": "client.authentication.k8s.io/v1beta1",
  "spec": {},
  "status": {
    "expirationTimestamp": "2024-01-15T12:15:00Z",
    "token": "k8s-aws-v1.aHR0cHM6Ly9zdHMuYW1hem9uYXdzLmNvbS8_QWN0aW9uPUdldENhbGxlcklkZW50aXR5..."
  }
}
```

### Decoding the Token

```sh
# Strip prefix and decode
TOKEN="k8s-aws-v1.aHR0cHM6Ly9zdHMuYW1hem9uYXdzLmNvbS8..."
echo "$TOKEN" | sed 's/k8s-aws-v1\.//' | base64 -d

# Output (a pre-signed STS URL):
# https://sts.amazonaws.com/?Action=GetCallerIdentity
# &Version=2011-06-15
# &X-Amz-Algorithm=AWS4-HMAC-SHA256
# &X-Amz-Credential=AKIAIOSFODNN7EXAMPLE/20240115/us-east-1/sts/aws4_request
# &X-Amz-Date=20240115T120000Z
# &X-Amz-Expires=60
# &X-Amz-SignedHeaders=host;x-k8s-aws-id
# &X-Amz-Signature=...
```

### Token Properties

| Property | Value |
|----------|-------|
| Token lifetime | 15 minutes (hardcoded) |
| Signed with | Caller's IAM credentials (access key + secret) |
| Contains | Pre-signed STS GetCallerIdentity URL |
| Custom header | `x-k8s-aws-id: <cluster-name>` (prevents cross-cluster token reuse) |
| STS region | Regional or global (depends on configuration) |

The `x-k8s-aws-id` header is critical — it's included in the signature so that a token generated for cluster A cannot be reused on cluster B.

## aws-iam-authenticator Webhook

The webhook runs as part of the EKS control plane (you can't see or modify it). It implements the Kubernetes [TokenReview API](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication).

### What the Webhook Does

1. Receives the token from the API server
2. Strips the `k8s-aws-v1.` prefix
3. Base64-decodes the payload to get the pre-signed URL
4. Validates the URL signature (checks it's properly signed, not expired)
5. Verifies the `x-k8s-aws-id` header matches the cluster name
6. Calls STS GetCallerIdentity using the pre-signed URL
7. Gets back the IAM ARN, account ID, and user ID
8. Looks up the ARN in aws-auth ConfigMap:
   - Checks `mapRoles` for role ARNs
   - Checks `mapUsers` for user ARNs
   - Checks `mapAccounts` for account-level access
9. Returns the mapped Kubernetes username and groups to the API server

### Token Validation Rules

The webhook rejects tokens if:
- Token doesn't start with `k8s-aws-v1.`
- Base64 decode fails
- URL is malformed
- Signature is expired (>15 minutes old)
- `x-k8s-aws-id` header doesn't match the cluster name
- STS returns an error (invalid credentials)
- IAM ARN is not found in aws-auth (or Access Entries)

## IAM Identity Resolution

### How ARNs Are Matched

The webhook receives an ARN from STS and must match it against aws-auth entries:

| STS Returns | Maps To | Notes |
|-------------|---------|-------|
| `arn:aws:iam::123:user/alice` | `mapUsers` entry with matching `userarn` | Direct IAM user |
| `arn:aws:sts::123:assumed-role/MyRole/session` | `mapRoles` entry with matching `rolearn` (without session) | Role assumption strips session name |
| `arn:aws:iam::123:root` | `mapAccounts` if account `123` is listed | Root account access |

### Session Names and Roles

When an IAM user assumes a role, STS returns:
```
arn:aws:sts::123456789012:assumed-role/MyRole/session-name
```

The authenticator strips the session name and matches against:
```yaml
- rolearn: arn:aws:iam::123456789012:role/MyRole
```

Not:
```yaml
# WRONG — don't use STS ARN format
- rolearn: arn:aws:sts::123456789012:assumed-role/MyRole/session-name
```

### The `{{EC2PrivateDNSName}}` Template

For node roles, the username uses a template:

```yaml
username: system:node:{{EC2PrivateDNSName}}
```

The authenticator replaces `{{EC2PrivateDNSName}}` with the actual private DNS name from the STS session name. For managed nodes, the session name is the instance's private DNS hostname (e.g., `ip-10-0-1-42.ec2.internal`).

This is how Kubernetes identifies which physical node is making the API call.

### The `{{SessionName}}` Template

For Fargate and general role assumptions:

```yaml
username: system:node:{{SessionName}}
```

Uses the session name from the STS assumed-role ARN directly.

## Cluster Creator Privilege

The IAM identity (user or role) that calls `CreateCluster` gets permanent, implicit `system:masters` access that:

- Is NOT stored in aws-auth
- Cannot be removed via aws-auth
- Cannot be removed via Access Entries (in API_AND_CONFIG_MAP mode)
- Persists even if the IAM identity is deleted (the cluster remembers the principal ID)

```sh
# See who created the cluster (check CloudTrail)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateCluster \
  --query "Events[?contains(Resources[].ResourceName, '<cluster-name>')].{User:Username, Time:EventTime}" \
  --output table
```

If the creator was a temporary role session (e.g., SSO), and that role is deleted, no one can revoke or modify the creator's access.

## ConfigMap vs Access Entries

### How Access Entries Work (Newer)

Access Entries bypass the aws-auth ConfigMap entirely. The API server queries the EKS API directly:

```
1. Token arrives at API server
2. Webhook validates with STS (same as before)
3. Instead of (or in addition to) checking aws-auth ConfigMap,
   the authenticator queries the EKS Access Entry API
4. Access Entry returns the user/groups mapping
5. Associated Access Policies define RBAC permissions
```

### Authentication Modes

```sh
aws eks describe-cluster --name <cluster> --query "cluster.accessConfig.authenticationMode" --output text
```

| Mode | Behavior |
|------|----------|
| `CONFIG_MAP` | Only aws-auth ConfigMap (legacy) |
| `API_AND_CONFIG_MAP` | Both methods checked (migration) |
| `API` | Only Access Entries (ConfigMap ignored) |

### Priority When Both Exist

In `API_AND_CONFIG_MAP` mode:
- Access Entries are checked first
- If no Access Entry matches, aws-auth ConfigMap is checked
- If both match, Access Entry wins

## STS Endpoint Behavior

### Regional vs Global STS

By default, `aws eks get-token` uses the global STS endpoint (`sts.amazonaws.com`). This can cause issues in some regions or with VPC endpoints.

```sh
# Force regional STS endpoint
aws eks get-token --cluster-name <cluster> --region us-east-1

# Or set in AWS config
# ~/.aws/config
[default]
sts_regional_endpoints = regional
```

### STS VPC Endpoint

In private clusters, the node needs to reach STS to exchange its credentials for a token. Without a NAT Gateway, you need a VPC endpoint:

```sh
aws ec2 create-vpc-endpoint \
  --vpc-id <vpc-id> \
  --service-name com.amazonaws.<region>.sts \
  --vpc-endpoint-type Interface \
  --subnet-ids <subnet-ids> \
  --security-group-ids <sg-id> \
  --private-dns-enabled
```

## Token Caching

### Client-Side

The exec credential plugin (`aws eks get-token`) is called on every kubectl request by default. To avoid repeated STS calls:

```yaml
# In ~/.kube/config, the exec plugin has caching built in:
users:
  - name: my-cluster
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        command: aws
        args:
          - eks
          - get-token
          - --cluster-name
          - my-cluster
          - --region
          - us-east-1
        interactiveMode: Never
```

kubectl caches the token until `expirationTimestamp` (15 minutes). The exec plugin is only called again when the cached token expires.

### Server-Side

The API server caches authenticated identities for a short period. Removing someone from aws-auth doesn't revoke their access immediately — they can continue until their cached session expires (~1-2 minutes).

## Debugging Authentication

### Check Your Current Identity

```sh
# What IAM identity am I using?
aws sts get-caller-identity

# What does the token look like?
aws eks get-token --cluster-name <cluster> | jq '.status.token' | cut -c1-50
```

### Test Authentication Without kubectl

```sh
# Get a token
TOKEN=$(aws eks get-token --cluster-name <cluster> --query "status.token" --output text)
ENDPOINT=$(aws eks describe-cluster --name <cluster> --query "cluster.endpoint" --output text)

# Call the API server directly
curl -sk -H "Authorization: Bearer $TOKEN" "$ENDPOINT/api/v1/namespaces"

# If 401: auth failed (check aws-auth)
# If 403: auth succeeded but RBAC denied (check ClusterRoleBindings)
```

### Check the Authenticator Logs

```sh
# Enable authenticator logging
aws eks update-cluster-config --name <cluster> \
  --logging '{"clusterLogging":[{"types":["authenticator"],"enabled":true}]}'

# Query logs
aws logs filter-log-events \
  --log-group-name /aws/eks/<cluster>/cluster \
  --log-stream-name-prefix authenticator \
  --filter-pattern "access denied" \
  --start-time $(date -u -d '1 hour ago' '+%s000')
```

### Common Auth Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Unauthorized` | IAM identity not in aws-auth | Add role/user to ConfigMap or Access Entry |
| `Forbidden` | Authenticated but RBAC denies the action | Add ClusterRoleBinding or RoleBinding |
| `error: exec plugin: invalid apiVersion` | Wrong kubeconfig format | Regenerate with `aws eks update-kubeconfig` |
| `could not get token: NoCredentialProviders` | No AWS credentials available | Check `aws sts get-caller-identity` |
| `token is expired` | Clock skew or stale cache | Check system time, clear credential cache |
| `the server has asked for the client to provide credentials` | Token not sent or invalid | Regenerate kubeconfig |

## Security Considerations

- **Tokens are bearer tokens**: Anyone with the token string can authenticate as that identity for 15 minutes. Treat them like passwords.
- **ConfigMap is not encrypted**: aws-auth is a plain ConfigMap. Any user with `get configmap` permission in `kube-system` can read it (but it only contains ARN mappings, not secrets).
- **mapAccounts is dangerous**: It grants access to ALL IAM identities in an account. Avoid using it.
- **system:masters is irrevocable via RBAC**: Once mapped to `system:masters`, RBAC cannot restrict that user. The only way to remove access is to remove them from aws-auth.
- **No audit of who edited aws-auth**: Kubernetes audit logs record the edit, but don't show what changed. Use GitOps (ArgoCD/Flux) to track changes.
- **Cross-cluster token reuse is prevented**: The `x-k8s-aws-id` header in the signature ties the token to a specific cluster.


## Variable Substitution

The `username` field supports template variables that are replaced at authentication time:

| Variable | Replaced With | Used For |
|----------|--------------|----------|
| `{{EC2PrivateDNSName}}` | Instance private DNS (e.g., `ip-10-0-1-42.ec2.internal`) | Node roles |
| `{{SessionName}}` | STS session name from assumed-role ARN | Fargate, general role assumptions |
| `{{AccountID}}` | AWS account ID of the caller | Multi-account setups |

```yaml
# Node role — each node gets a unique Kubernetes identity
- rolearn: arn:aws:iam::123456789012:role/NodeRole
  username: system:node:{{EC2PrivateDNSName}}
  groups:
    - system:bootstrappers
    - system:nodes

# Developer role — session name becomes the username
- rolearn: arn:aws:iam::123456789012:role/DeveloperRole
  username: developer:{{SessionName}}
  groups:
    - developers

# Cross-account role — include account ID for clarity
- rolearn: arn:aws:iam::987654321098:role/CrossAccountRole
  username: cross-account:{{AccountID}}:{{SessionName}}
  groups:
    - viewers
```

## Common Kubernetes Groups

| Group | Purpose | Notes |
|-------|---------|-------|
| `system:masters` | Full cluster admin, bypasses RBAC | Use sparingly, cannot be restricted |
| `system:bootstrappers` | Node bootstrap process | Required for node roles |
| `system:nodes` | kubelet operations | Required for node roles |
| `system:node-proxier` | kube-proxy operations | Required for Fargate pod execution roles |
| Custom (e.g., `developers`) | Mapped via RoleBindings | You define permissions via RBAC |

## Managing aws-auth with Terraform

```hcl
resource "kubernetes_config_map_v1_data" "aws_auth" {
  metadata {
    name      = "aws-auth"
    namespace = "kube-system"
  }

  data = {
    mapRoles = yamlencode([
      {
        rolearn  = aws_iam_role.nodes.arn
        username = "system:node:{{EC2PrivateDNSName}}"
        groups   = ["system:bootstrappers", "system:nodes"]
      },
      {
        rolearn  = aws_iam_role.admin.arn
        username = "admin:{{SessionName}}"
        groups   = ["system:masters"]
      },
      {
        rolearn  = aws_iam_role.developer.arn
        username = "dev:{{SessionName}}"
        groups   = ["developers"]
      }
    ])

    mapUsers = yamlencode([
      {
        userarn  = "arn:aws:iam::123456789012:user/breakglass"
        username = "breakglass"
        groups   = ["system:masters"]
      }
    ])
  }

  force = true
}
```

> Use `kubernetes_config_map_v1_data` instead of `kubernetes_config_map` to avoid overwriting the entire ConfigMap (EKS may add entries automatically for managed node groups).

## Testing Authentication and Authorization

```sh
# Test token generation with a specific role
aws eks get-token --cluster-name <cluster> --role-arn arn:aws:iam::123456789012:role/DeveloperRole

# Check what identity kubectl is using
aws sts get-caller-identity

# Test RBAC permissions (impersonate a user)
kubectl auth can-i get pods --as=developer
kubectl auth can-i create deployments --as=developer --namespace=production
kubectl auth can-i '*' '*' --as=developer   # Can this user do everything?

# Test as a group
kubectl auth can-i get pods --as-group=developers

# List all permissions for a user
kubectl auth can-i --list --as=developer
```

## Validating ConfigMap Syntax

A YAML error inside the `mapRoles` or `mapUsers` string silently breaks authentication:

```sh
# Validate with yq
kubectl get cm aws-auth -n kube-system -o yaml | yq eval '.data.mapRoles' - | yq eval '.' -

# Validate with Python
kubectl get cm aws-auth -n kube-system -o jsonpath='{.data.mapRoles}' | python3 -c "import sys, yaml; yaml.safe_load(sys.stdin.read()); print('Valid YAML')"

# Quick check: does it parse?
kubectl get cm aws-auth -n kube-system -o jsonpath='{.data.mapRoles}' | python3 -c "
import sys, yaml
try:
    data = yaml.safe_load(sys.stdin.read())
    print(f'Valid: {len(data)} entries')
    for entry in data:
        print(f'  - {entry.get(\"rolearn\", entry.get(\"userarn\", \"unknown\"))}')
except yaml.YAMLError as e:
    print(f'INVALID: {e}')
    sys.exit(1)
"
```

## Recovery: Locked Out of Cluster

If aws-auth is corrupted and you can't authenticate:

```sh
# Option 1: Use the cluster creator identity
# The IAM identity that called CreateCluster always has implicit access

# Option 2: Use AWS CLI to patch the ConfigMap via EKS API (if access entries are enabled)
aws eks create-access-entry --cluster-name <cluster> \
  --principal-arn arn:aws:iam::123456789012:role/EmergencyRole \
  --type STANDARD

aws eks associate-access-policy --cluster-name <cluster> \
  --principal-arn arn:aws:iam::123456789012:role/EmergencyRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster

# Option 3: If in API_AND_CONFIG_MAP mode, access via Access Entries even if ConfigMap is broken
```

> Always keep a backup: `kubectl get cm aws-auth -n kube-system -o yaml > aws-auth-backup.yaml`
