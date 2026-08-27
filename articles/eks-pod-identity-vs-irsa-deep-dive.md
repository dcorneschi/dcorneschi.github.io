# EKS Pod Identity vs IRSA — Deep Dive

How EKS Pod Identity and IRSA deliver AWS credentials to pods — internal architecture, credential flow, trust policies, the Pod Identity Agent, migration path, and edge cases.

Note: For a brief comparison table, see the EKS architecture deep-dive. This article is the full standalone reference covering internals, troubleshooting, and migration.

## How IRSA Works (Since 2019)

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  IRSA — IAM Roles for Service Accounts                              │
│                                                                     │
│  ┌──────────┐     ┌───────────────┐     ┌────────────────┐          │
│  │   Pod    │────▶│   AWS SDK     │────▶│ AWS STS        │          │
│  │          │     │  (in your app)│     │ AssumeRole     │          │
│  │ Projected│     │               │     │ WithWebIdentity│          │
│  │ JWT token│     │ Reads:        │     │                │          │
│  │ mounted  │     │ AWS_ROLE_ARN  │     │ Validates JWT  │          │
│  │          │     │ AWS_WEB_ID_   │     │ against OIDC   │          │
│  │          │     │ TOKEN_FILE    │     │ issuer         │          │
│  └──────────┘     └───────────────┘     └────────┬───────┘          │
│                                                  │                  │
│                                                  ▼                  │
│                                         ┌────────────────┐          │
│                                         │  OIDC Provider │          │
│                                         │  (cluster's    │          │
│                                         │   public JWKS) │          │
│                                         └────────────────┘          │
└─────────────────────────────────────────────────────────────────────┘
```

### Credential Flow (Step by Step)

```
1. Pod starts
2. Kubelet mounts a projected ServiceAccount token (JWT) at:
   /var/run/secrets/eks.amazonaws.com/serviceaccount/token
3. Mutating webhook (pod-identity-webhook) injects environment variables:
   AWS_ROLE_ARN=arn:aws:iam::123456789:role/my-app-role
   AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
4. AWS SDK detects these env vars (credential chain)
5. SDK calls STS AssumeRoleWithWebIdentity:
   - Passes the JWT token
   - Passes the role ARN
6. STS validates the JWT:
   - Fetches JWKS from the cluster's OIDC issuer URL
   - Verifies JWT signature
   - Checks audience claim
   - Checks subject matches trust policy condition
7. STS returns temporary credentials (access key, secret key, session token)
8. SDK uses credentials for AWS API calls
9. Token is rotated by kubelet (~12h), SDK re-assumes on expiry
```

### Requirements for IRSA

```bash
# 1. OIDC provider must be registered in IAM (per cluster):
aws iam list-open-id-connect-providers | grep $(aws eks describe-cluster --name <cluster> \
  --query "cluster.identity.oidc.issuer" --output text | sed 's|https://||')

# If not registered:
eksctl utils associate-iam-oidc-provider --cluster <cluster> --approve

# 2. IAM role trust policy must reference the SPECIFIC cluster OIDC issuer:
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::123456789:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/ABC123"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.us-east-1.amazonaws.com/id/ABC123:sub": "system:serviceaccount:default:my-app",
        "oidc.eks.us-east-1.amazonaws.com/id/ABC123:aud": "sts.amazonaws.com"
      }
    }
  }]
}

# 3. ServiceAccount must be annotated:
kubectl annotate sa my-app eks.amazonaws.com/role-arn=arn:aws:iam::123456789:role/my-app-role
```

### IRSA Limitations

| Limitation | Impact |
|-----------|--------|
| One OIDC provider per cluster | Trust policies are cluster-specific |
| Trust policy references cluster OIDC ID | Must update trust policies when recreating clusters |
| No automatic session tags | Can't use ABAC without manual tag setup |
| Cross-account is complex | Requires chained roles or shared OIDC provider |
| Webhook must be running | If pod-identity-webhook is down, new pods won't get env vars |
| Max 100 OIDC providers per account | Limits number of clusters with IRSA |

## How EKS Pod Identity Works (Since 2023)

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  EKS Pod Identity                                                   │
│                                                                     │
│  ┌──────────┐     ┌───────────────┐     ┌───────────────────┐       │
│  │   Pod    │────▶│   AWS SDK     │────▶│  Pod Identity     │       │
│  │          │     │  (in your app)│     │  Agent            │       │
│  │ Token    │     │               │     │  (on every node)  │       │
│  │ injected │     │ Calls local   │     │                   │       │
│  │ via env  │     │ endpoint:     │     │  Calls EKS API    │       │
│  │          │     │ 169.254.170.23│     │  for credentials  │       │
│  └──────────┘     └───────────────┘     └─────────┬─────────┘       │
│                                                   │                 │
│                                                   ▼                 │
│                                          ┌────────────────┐         │
│                                          │  EKS Service   │         │
│                                          │  (AssumeRole   │         │
│                                          │   for Pod      │         │
│                                          │   Identity)    │         │
│                                          └────────────────┘         │
└─────────────────────────────────────────────────────────────────────┘
```

### Credential Flow (Step by Step)

```
1. Admin creates a Pod Identity Association:
   (maps ServiceAccount → IAM Role in EKS API)

2. Pod starts

3. EKS mutating webhook (eks-pod-identity-webhook) injects:
   - Environment variable: AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE
   - Environment variable: AWS_CONTAINER_CREDENTIALS_FULL_URI=http://169.254.170.23/v1/credentials
   - Projected token volume (audience: pods.eks.amazonaws.com)

4. AWS SDK detects these env vars (container credential provider in chain)

5. SDK calls the local Pod Identity Agent at 169.254.170.23:
   - Passes the pod's projected token
   - Agent verifies pod identity (via kubelet API)

6. Pod Identity Agent calls EKS service API:
   - "This pod runs as ServiceAccount X in namespace Y"
   - EKS service looks up the Pod Identity Association
   - EKS service calls STS AssumeRole on behalf of the pod

7. Agent returns temporary AWS credentials to the SDK

8. SDK caches credentials and refreshes before expiry
```

### The Pod Identity Agent

The agent runs as a DaemonSet (`eks-pod-identity-agent`) on every node:

```bash
# Check agent is running:
kubectl get daemonset eks-pod-identity-agent -n kube-system

# Check agent pods:
kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent
```

```
┌────────────────────────────────────────────────────────────────┐
│  Pod Identity Agent (DaemonSet on every node)                  │
│                                                                │
│  - Listens on 169.254.170.23:80 and 169.254.170.23:2703        │
│  - Port 80: credential endpoint (HTTP)                         │
│  - Port 2703: credential endpoint (HTTP, alternate)            │
│  - iptables rules redirect pod traffic to the agent            │
│  - Validates pod tokens against the kubelet API                │
│  - Caches credentials (reduces STS call volume)                │
│  - Runs with hostNetwork: true                                 │
│                                                                │
│  iptables rule (injected by agent):                            │
│    -d 169.254.170.23 -p tcp --dport 80 -j DNAT                 │
│    --to-destination <agent-pod-ip>:80                          │
└────────────────────────────────────────────────────────────────┘
```

### Pod Identity Association

The mapping between ServiceAccount and IAM Role is managed in the EKS API (not in Kubernetes objects):

```bash
# Create an association:
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace default \
  --service-account my-app \
  --role-arn arn:aws:iam::123456789:role/my-app-role

# List associations:
aws eks list-pod-identity-associations --cluster-name my-cluster

# Describe:
aws eks describe-pod-identity-association \
  --cluster-name my-cluster \
  --association-id a-1234567890

# Delete:
aws eks delete-pod-identity-association \
  --cluster-name my-cluster \
  --association-id a-1234567890
```

### IAM Role Trust Policy for Pod Identity

Much simpler than IRSA — no cluster-specific OIDC reference:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "pods.eks.amazonaws.com"
    },
    "Action": [
      "sts:AssumeRole",
      "sts:TagSession"
    ]
  }]
}
```

Key differences from IRSA trust policy:
- Principal is `pods.eks.amazonaws.com` service (not a specific OIDC provider)
- No `Condition` referencing cluster OIDC ID
- Same trust policy works across ALL clusters
- `sts:TagSession` enables automatic session tags

### Automatic Session Tags

Pod Identity automatically sets these session tags on assumed credentials:

| Tag Key | Value | Use Case |
|---------|-------|----------|
| `eks-cluster-arn` | `arn:aws:eks:us-east-1:123456789:cluster/my-cluster` | ABAC per cluster |
| `eks-cluster-name` | `my-cluster` | Simpler ABAC |
| `kubernetes-namespace` | `default` | Restrict S3 prefix by namespace |
| `kubernetes-service-account` | `my-app` | Fine-grained ABAC |
| `kubernetes-pod-name` | `my-app-abc123` | Per-pod audit trail |

These enable Attribute-Based Access Control (ABAC) without extra configuration:

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/${aws:PrincipalTag/kubernetes-namespace}/*",
  "Condition": {
    "StringEquals": {
      "aws:PrincipalTag/eks-cluster-name": "production"
    }
  }
}
```

## Head-to-Head Comparison

| Feature | IRSA | Pod Identity |
|---------|:----:|:------------:|
| Year introduced | 2019 | 2023 |
| OIDC provider per cluster | Required | Not needed |
| Trust policy per cluster | Yes (cluster-specific OIDC ID) | No (generic service principal) |
| SA annotation required | Yes (`eks.amazonaws.com/role-arn`) | No |
| Where association is stored | SA annotation + trust policy | EKS API (Pod Identity Association) |
| Session tags | Manual setup | Automatic (cluster, namespace, SA, pod) |
| Cross-account | Complex (share OIDC or chain roles) | Simple (same trust policy everywhere) |
| Credential delivery | STS AssumeRoleWithWebIdentity (pod calls STS directly) | Local agent (pod calls 169.254.170.23) |
| Agent required | No (just webhook) | Yes (DaemonSet) |
| Latency | Higher (network call to STS) | Lower (local agent with cache) |
| OIDC provider limit | 100 per account | N/A |
| EKS add-on | Not needed | Required (`eks-pod-identity-agent`) |
| Works on Fargate | Yes | Yes (agent runs on Fargate nodes) |
| Kubernetes version | 1.14+ | 1.24+ |
| AWS SDK version | Any recent | 2023+ SDKs (container credential provider) |

## Migration: IRSA → Pod Identity

### Can They Coexist?

Yes. Both can run simultaneously on the same cluster. Pods can use either method. During migration, you can have some pods on IRSA and others on Pod Identity.

### Migration Steps

```bash
# 1. Install the Pod Identity Agent add-on:
aws eks create-addon --cluster-name my-cluster --addon-name eks-pod-identity-agent

# 2. Update IAM role trust policy to accept BOTH:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Federated": "arn:aws:iam::123456789:oidc-provider/oidc.eks..."},
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks....:sub": "system:serviceaccount:default:my-app"
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": {"Service": "pods.eks.amazonaws.com"},
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }
  ]
}

# 3. Create Pod Identity Association:
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace default \
  --service-account my-app \
  --role-arn arn:aws:iam::123456789:role/my-app-role

# 4. Restart pods (they'll pick up Pod Identity if agent is running):
kubectl rollout restart deployment my-app

# 5. Verify pods use Pod Identity:
kubectl exec my-app-xxx -- env | grep AWS_CONTAINER_CREDENTIALS_FULL_URI
# Should show: http://169.254.170.23/v1/credentials

# 6. After verifying, remove IRSA annotation (optional):
kubectl annotate sa my-app eks.amazonaws.com/role-arn-

# 7. After all pods migrated, remove OIDC trust policy statement
```

### Priority When Both Are Configured

If a pod has BOTH IRSA annotation and Pod Identity association:
- **Pod Identity wins** (takes precedence)
- The webhook injects Pod Identity env vars, not IRSA env vars
- This makes migration safe — just add the association, restart pods

## Troubleshooting

### IRSA Issues

```bash
# Check SA annotation:
kubectl get sa my-app -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}'

# Check pod has the env vars:
kubectl exec my-app-xxx -- env | grep AWS_
# Should see: AWS_ROLE_ARN, AWS_WEB_IDENTITY_TOKEN_FILE

# Check token is mounted:
kubectl exec my-app-xxx -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | cut -d'.' -f2 | base64 -d | jq .

# Check OIDC provider exists:
aws iam list-open-id-connect-providers

# Verify trust policy:
aws iam get-role --role-name my-app-role --query "Role.AssumeRolePolicyDocument"

# Manual STS test:
TOKEN=$(kubectl exec my-app-xxx -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token)
aws sts assume-role-with-web-identity \
  --role-arn arn:aws:iam::123456789:role/my-app-role \
  --role-session-name test \
  --web-identity-token "$TOKEN"
```

### Pod Identity Issues

```bash
# Check agent is running on the node:
kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent -o wide

# Check association exists:
aws eks list-pod-identity-associations --cluster-name my-cluster \
  --query "associations[?serviceAccount=='my-app']"

# Check pod has Pod Identity env vars:
kubectl exec my-app-xxx -- env | grep AWS_CONTAINER
# Should see: AWS_CONTAINER_CREDENTIALS_FULL_URI=http://169.254.170.23/v1/credentials

# Test credential endpoint from the pod:
kubectl exec my-app-xxx -- wget -qO- http://169.254.170.23/v1/credentials

# Check agent logs:
kubectl logs -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent --tail=20

# Check iptables rules on node (agent must redirect traffic):
kubectl debug node/<node> -it --image=busybox -- iptables-save | grep 169.254.170.23

# Common errors:
# "No pod identity association found" → Association not created or wrong SA/namespace
# "AccessDenied" → Trust policy doesn't include pods.eks.amazonaws.com
# Connection refused to 169.254.170.23 → Agent not running or iptables not configured
```

## Quick Reference

```bash
# IRSA setup:
# 1. Register OIDC provider
# 2. Create IAM role with OIDC trust policy (cluster-specific)
# 3. Annotate ServiceAccount
# 4. Pod gets JWT + env vars → calls STS directly

# Pod Identity setup:
# 1. Install eks-pod-identity-agent add-on
# 2. Create IAM role with service principal trust (generic)
# 3. Create Pod Identity Association (EKS API)
# 4. Pod gets token + env vars → calls local agent → agent calls EKS

# Key differences:
# IRSA: cluster-specific trust policy, no agent, calls STS directly
# Pod Identity: generic trust policy, requires agent, automatic session tags

# Migration:
# Both can coexist. Pod Identity takes precedence.
# Add association → restart pods → verify → remove IRSA annotation

# Pod Identity commands:
aws eks create-pod-identity-association --cluster-name X --namespace Y --service-account Z --role-arn ARN
aws eks list-pod-identity-associations --cluster-name X
aws eks delete-pod-identity-association --cluster-name X --association-id ID

# Verify which method a pod uses:
kubectl exec <pod> -- env | grep AWS_
# IRSA: AWS_ROLE_ARN + AWS_WEB_IDENTITY_TOKEN_FILE
# Pod Identity: AWS_CONTAINER_CREDENTIALS_FULL_URI (169.254.170.23)
```
