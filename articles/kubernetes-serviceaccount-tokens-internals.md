# How Kubernetes ServiceAccount Tokens Work

How pods authenticate to the API server — projected volumes, bound tokens, the TokenRequest API, audience scoping, automatic rotation, and the migration from legacy long-lived secrets.

## High-Level Flow

```
┌─────────────┐     ┌──────────┐     ┌───────────────┐     ┌───────────────┐
│  Pod starts │────▶│  Kubelet │────▶│  TokenRequest │────▶│   API Server  │
│             │     │  mounts  │     │  API          │     │  validates    │
│  Needs SA   │     │ projected│     │  (issues JWT) │     │  JWT on each  │
│  token      │     │  volume  │     │               │     │  request      │
└─────────────┘     └──────────┘     └───────────────┘     └───────────────┘
```

## ServiceAccount Basics

Every pod runs as a ServiceAccount. If you don't specify one, it uses the `default` SA in the namespace:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
automountServiceAccountToken: true   # Default: true
```

```bash
# Check which SA a pod uses:
kubectl get pod my-pod -o jsonpath='{.spec.serviceAccountName}'

# List SAs in a namespace:
kubectl get serviceaccounts -n production
```

## Modern Token System (Kubernetes 1.22+)

### Projected Volume Token (Default)

Every pod gets a token automatically mounted via a **projected volume**:

```yaml
# This is auto-injected by the ServiceAccount admission controller:
spec:
  volumes:
  - name: kube-api-access-xxxxx
    projected:
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          name: kube-root-ca.crt
          items:
          - key: ca.crt
            path: ca.crt
      - downwardAPI:
          items:
          - path: namespace
            fieldRef:
              fieldPath: metadata.namespace
  containers:
  - name: app
    volumeMounts:
    - name: kube-api-access-xxxxx
      mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      readOnly: true
```

The mounted directory contains:

```bash
/var/run/secrets/kubernetes.io/serviceaccount/
├── token       # JWT token (short-lived, auto-rotated)
├── ca.crt      # Cluster CA certificate (for TLS verification)
└── namespace   # Pod's namespace (plain text)
```

### Token Properties

| Property | Value |
|----------|-------|
| Format | JWT (JSON Web Token) |
| Default expiry | 1 hour (3607 seconds) |
| Audience | `https://kubernetes.default.svc` (API server) |
| Issuer | API server (`--service-account-issuer` flag) |
| Bound to | Specific pod (pod name + UID in claims) |
| Auto-rotation | Kubelet refreshes before expiry (~80% of lifetime) |

### Decoding the Token

```bash
# Inside a pod:
cat /var/run/secrets/kubernetes.io/serviceaccount/token | \
  cut -d'.' -f2 | base64 -d 2>/dev/null | jq .
```

```json
{
  "aud": ["https://kubernetes.default.svc"],
  "exp": 1710500000,
  "iat": 1710496393,
  "iss": "https://kubernetes.default.svc",
  "kubernetes.io": {
    "namespace": "production",
    "pod": {
      "name": "my-app-abc123",
      "uid": "12345-abcde-67890"
    },
    "serviceaccount": {
      "name": "my-app",
      "uid": "aaaaa-bbbbb-ccccc"
    }
  },
  "nbf": 1710496393,
  "sub": "system:serviceaccount:production:my-app"
}
```

### Key Claims

| Claim | Meaning |
|-------|---------|
| `sub` | Subject — `system:serviceaccount:<namespace>:<name>` |
| `aud` | Audience — who the token is intended for |
| `exp` | Expiration time (Unix timestamp) |
| `iat` | Issued at time |
| `iss` | Issuer — API server URL |
| `kubernetes.io.pod` | Bound to this specific pod (name + UID) |
| `kubernetes.io.serviceaccount` | The SA identity |

## TokenRequest API

The kubelet uses the TokenRequest API to get tokens for pods:

```
┌──────────────────────────────────────────────────────────────┐
│  Kubelet Token Management                                    │
│                                                              │
│  1. Pod needs a token (projected volume mounted)             │
│  2. Kubelet calls POST /api/v1/namespaces/{ns}/              │
│     serviceaccounts/{sa}/token                               │
│  3. API server issues a signed JWT                           │
│  4. Kubelet writes token to the projected volume path        │
│  5. At ~80% of expiry, kubelet requests a new token          │
│  6. Kubelet atomically replaces the file                     │
│  7. Application reads new token on next file access          │
│                                                              │
│  The pod itself never calls TokenRequest — kubelet does it   │
└──────────────────────────────────────────────────────────────┘
```

### Manual Token Request

```bash
# Request a token manually (for debugging or custom audiences):
kubectl create token my-app \
  --namespace production \
  --duration 1h \
  --audience "https://my-external-service.example.com"

# The output is a JWT you can use directly:
# eyJhbGciOiJSUzI1NiIsImtpZCI6...
```

### Custom Audience Tokens

Tokens can be scoped to a specific audience (not just the API server):

```bash
# Token for AWS IRSA (IAM Roles for Service Accounts):
kubectl create token my-app --audience sts.amazonaws.com

# Token for a Vault integration:
kubectl create token my-app --audience vault

# Token for a custom external service:
kubectl create token my-app --audience https://api.myservice.com
```

The receiving service validates the token against the cluster's OIDC issuer.

## Token Validation — API Server Side

When a pod makes a request to the API server with a token:

```
Pod sends request with header:
  Authorization: Bearer eyJhbGciOiJSUzI1NiI...
    │
    ▼
┌───────────────────────────────────────────────────────────────┐
│  API Server Token Validation                                  │
│                                                               │
│  1. Verify JWT signature (using SA signing key)               │
│  2. Check exp claim (not expired)                             │
│  3. Check aud claim (matches API server audience)             │
│  4. Check iss claim (matches configured issuer)               │
│  5. Verify bound pod still exists (not deleted)               │
│     → If pod is deleted, token is IMMEDIATELY invalid         │
│  6. Verify SA still exists                                    │
│  7. Extract identity: system:serviceaccount:{ns}:{name}       │
│  8. Proceed to authorization (RBAC)                           │
└───────────────────────────────────────────────────────────────┘
```

### Bound Token Invalidation

Because tokens are bound to a specific pod, they become invalid when:
- The pod is deleted (even if token hasn't expired)
- The ServiceAccount is deleted
- The token expires

This is a major security improvement over legacy long-lived tokens.

## Legacy Tokens (Pre-1.24)

Before 1.24, ServiceAccounts got a long-lived Secret with a non-expiring token:

```yaml
# Legacy (auto-created, deprecated since 1.24):
apiVersion: v1
kind: Secret
metadata:
  name: my-app-token-abc12
  annotations:
    kubernetes.io/service-account.name: my-app
type: kubernetes.io/service-account-token
data:
  token: <base64-encoded-JWT>     # Never expires!
  ca.crt: <base64-encoded-CA>
  namespace: <base64-encoded-ns>
```

### Legacy Token Problems

| Issue | Risk |
|-------|------|
| Never expires | Stolen token grants permanent access |
| Not bound to pods | Any process with the token can use it |
| Shared across all pods | One compromised pod exposes all pods using the SA |
| No audience scoping | Token works for any service (not just API server) |
| Stored in etcd as Secret | Visible to anyone with Secret read access |

### Migration Timeline

| Version | Behavior |
|---------|----------|
| ≤ 1.21 | Legacy Secret auto-created for every SA |
| 1.22 | Projected tokens become default for pods |
| 1.24 | Auto-creation of legacy Secrets stopped |
| 1.24+ | Existing legacy Secrets still work but emit warnings |
| Future | LegacyServiceAccountTokenNoAutoGeneration fully enforced |

```bash
# Find legacy token Secrets still in use:
kubectl get secrets -A --field-selector type=kubernetes.io/service-account-token

# Check if pods are still using legacy tokens:
kubectl get pods -A -o json | jq -r '
  .items[] | select(.spec.volumes[]?.secret.secretName? | test("token")) |
  "\(.metadata.namespace)/\(.metadata.name)"'
```

## Token Rotation

### Automatic (Projected Volume)

```
┌────────────────────────────────────────────────────────────┐
│  Kubelet Rotation Logic                                    │
│                                                            │
│  Token issued at: T+0, expires at: T+3607s (1h)            │
│                                                            │
│  At ~80% of lifetime (T+2886s):                            │
│    → Kubelet requests a new token via TokenRequest API     │
│    → Writes new token to projected volume (atomic replace) │
│    → Old token still valid until original expiry           │
│                                                            │
│  Application:                                              │
│    → Must re-read the token file periodically              │
│    → OR use a client library that handles this             │
│    → Do NOT cache the token in memory forever              │
└────────────────────────────────────────────────────────────┘
```

### Application Best Practices

```bash
# WRONG: Read token once at startup
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
# This token will expire in ~1 hour!

# RIGHT: Read token on each request (or periodically)
# Most K8s client libraries do this automatically
```

```python
# Python client-go equivalent — reads token on each call:
from kubernetes import client, config
config.load_incluster_config()  # Reads token file each time
```

## Disabling Token Mounting

```yaml
# Disable for a specific pod:
apiVersion: v1
kind: Pod
spec:
  automountServiceAccountToken: false
  containers:
  - name: app
    image: my-app

# Disable for all pods using a SA:
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
automountServiceAccountToken: false
```

Pod-level setting overrides SA-level setting.

## OIDC Discovery — External Token Validation

The API server publishes its OIDC discovery document so external services can validate tokens:

```bash
# OIDC issuer URL (from API server flag --service-account-issuer):
# e.g., https://oidc.eks.us-east-1.amazonaws.com/id/ABC123

# Discovery endpoint:
curl https://<issuer>/.well-known/openid-configuration

# JWKS (public keys for token signature verification):
curl https://<issuer>/openid/v1/jwks
```

This is how AWS IRSA, GCP Workload Identity, and Vault validate K8s tokens without calling the API server:

```
Pod → gets token with audience "sts.amazonaws.com"
Pod → sends token to AWS STS AssumeRoleWithWebIdentity
AWS STS → fetches JWKS from cluster's OIDC issuer
AWS STS → validates JWT signature + audience + expiry
AWS STS → returns temporary AWS credentials
```

## Token Signing Keys

The API server uses a private key to sign tokens and a public key to verify them:

```
--service-account-signing-key-file    # Private key (signs tokens)
--service-account-key-file            # Public key (verifies tokens)
--service-account-issuer              # Issuer URL in JWT "iss" claim
```

Key rotation:
- Add new public key to `--service-account-key-file` (accepts both old and new)
- Switch `--service-account-signing-key-file` to new private key
- Old tokens (signed with old key) still validate (old public key still trusted)
- After all old tokens expire, remove old public key

## Debugging Tokens

```bash
# Check what token a pod has:
kubectl exec my-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token

# Decode the token (without verification):
kubectl exec my-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token | \
  cut -d'.' -f2 | base64 -d 2>/dev/null | jq .

# Check token expiry:
kubectl exec my-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token | \
  cut -d'.' -f2 | base64 -d 2>/dev/null | jq '.exp | todate'

# Create a token and validate it:
TOKEN=$(kubectl create token my-app -n production)
kubectl auth can-i get pods --token="$TOKEN"

# Check if a SA token can do something:
kubectl auth can-i create deployments --as system:serviceaccount:production:my-app

# Find pods not mounting tokens:
kubectl get pods -A -o json | jq -r '
  .items[] | select(.spec.automountServiceAccountToken == false) |
  "\(.metadata.namespace)/\(.metadata.name)"'

# Check OIDC issuer:
kubectl get --raw /.well-known/openid-configuration | jq .
```

## Quick Reference

```bash
# Modern tokens (1.22+):
# - Short-lived JWT (1h default, auto-rotated by kubelet)
# - Bound to specific pod (invalid when pod deleted)
# - Audience-scoped (can target external services)
# - Mounted via projected volume

# Token path in pod:
# /var/run/secrets/kubernetes.io/serviceaccount/token
# /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
# /var/run/secrets/kubernetes.io/serviceaccount/namespace

# Request a token manually:
kubectl create token <sa-name> --duration 1h --audience <aud>

# Check SA for a pod:
kubectl get pod <name> -o jsonpath='{.spec.serviceAccountName}'

# Disable auto-mount:
# Pod: spec.automountServiceAccountToken: false
# SA:  automountServiceAccountToken: false

# Token identity format:
# system:serviceaccount:<namespace>:<sa-name>

# Rotation: kubelet refreshes at ~80% of lifetime
# Applications MUST re-read the token file (not cache forever)

# Legacy tokens (Secrets): never expire, not bound, security risk
# Find them: kubectl get secrets -A --field-selector type=kubernetes.io/service-account-token

# OIDC discovery (external validation):
# GET <issuer>/.well-known/openid-configuration
# GET <issuer>/openid/v1/jwks
```
