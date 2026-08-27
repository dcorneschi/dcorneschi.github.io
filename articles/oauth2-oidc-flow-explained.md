# OAuth2 and OIDC Flow Explained

How OAuth2 and OpenID Connect work — tokens, scopes, grant types, the authorization code flow, and how Kubernetes integrates with OIDC for authentication.

## OAuth2 vs OIDC — The Difference

```
┌────────────────────────────────────────────────────────────────────┐
│  OAuth2 = AUTHORIZATION ("what can you access?")                   │
│    → Grants a third-party app limited access to a resource         │
│    → Results in an Access Token (opaque or JWT)                    │
│    → Defines scopes (permissions)                                  │
│                                                                    │
│  OIDC = AUTHENTICATION ("who are you?") built ON TOP of OAuth2     │
│    → Adds an Identity Layer to OAuth2                              │
│    → Results in an ID Token (always a JWT)                         │
│    → Contains user identity claims (name, email, groups)           │
│    → Uses OAuth2 flows but adds identity endpoints                 │
│                                                                    │
│  OIDC = OAuth2 + ID Token + UserInfo endpoint + Discovery          │
└────────────────────────────────────────────────────────────────────┘
```

| Protocol | Purpose | Token | Use Case |
|----------|---------|-------|----------|
| OAuth2 | Authorization | Access Token | "Can this app read my photos?" |
| OIDC | Authentication + Authorization | ID Token + Access Token | "Who is this user? Also give them access." |

## Key Concepts

### Roles (OAuth2 Terminology)

| Role | Who | Example |
|------|-----|---------|
| Resource Owner | The user who owns the data | You (the human) |
| Client | The app requesting access | A web app, CLI tool, mobile app |
| Authorization Server | Issues tokens after authentication | Okta, Auth0, Cognito, Keycloak, Dex |
| Resource Server | Hosts the protected resources | Your API, Kubernetes API server |

### Tokens

| Token | Format | Purpose | Lifetime |
|-------|--------|---------|----------|
| Access Token | Opaque string or JWT | Authorize API calls | Short (5-60 min) |
| Refresh Token | Opaque string | Get new access tokens without re-login | Long (days-months) |
| ID Token (OIDC only) | Always JWT | Prove user identity | Short (5-60 min) |

### Scopes

Scopes define what permissions the token grants:

```
Common OAuth2 scopes:
  openid        → Required for OIDC (returns ID Token)
  profile       → User's name, picture, locale
  email         → User's email address
  groups        → User's group memberships (custom)
  offline_access → Get a refresh token

Kubernetes-relevant scopes:
  openid        → Required
  groups        → Maps to K8s groups for RBAC
  email         → Used as username in K8s
```

## Grant Types (Flows)

### Authorization Code Flow (Most Common, Most Secure)

Used by: web apps, CLI tools (with PKCE), Kubernetes OIDC auth.

```
┌──────────┐     ┌───────────────┐     ┌──────────────────┐     ┌──────────────┐
│  User    │     │  Client App   │     │  Auth Server     │     │  Resource    │
│ (browser)│     │  (your app)   │     │  (Okta/Cognito)  │     │  Server (API)│
└──────────┘     └───────────────┘     └──────────────────┘     └──────────────┘
     │                  │                       │                       │
     │  1. Click Login  │                       │                       │
     │─────────────────▶│                       │                       │
     │                  │                       │                       │
     │                  │  2. Redirect to       │                       │
     │◀─────────────────│     /authorize        │                       │
     │  (302 redirect)  │                       │                       │
     │                  │                       │                       │
     │  3. User enters credentials              │                       │
     │─────────────────────────────────────────▶│                       │
     │                  │                       │                       │
     │  4. Auth server validates credentials    │                       │
     │     + user consents to scopes            │                       │
     │                  │                       │                       │
     │◀─────────────────────────────────────────│                       │
     │  5. Redirect back with authorization code│                       │
     │     (code in URL query parameter)        │                       │
     │                  │                       │                       │
     │─────────────────▶│                       │                       │
     │  6. App receives │                       │                       │
     │     the code     │                       │                       │
     │                  │ 7. Exchange code for  │                       │
     │                  │    tokens (server-    │                       │
     │                  │    to-server, with    │                       │
     │                  │    client secret)     │                       │
     │                  │──────────────────────▶│                       │
     │                  │                       │                       │
     │                  │◀──────────────────────│                       │
     │                  │  8. Tokens returned:  │                       │
     │                  │     - access_token    │                       │
     │                  │     - id_token (OIDC) │                       │
     │                  │     - refresh_token   │                       │
     │                  │                       │                       │
     │                  │  9. Call API with access token                │
     │                  │──────────────────────────────────────────────▶│
     │                  │                       │                       │
     │                  │◀──────────────────────────────────────────────│
     │                  │  10. Protected resource                       │
```

### Authorization Code Flow with PKCE (Public Clients)

For clients that can't keep a secret (SPAs, mobile apps, CLI tools):

```
┌────────────────────────────────────────────────────────────────┐
│  PKCE (Proof Key for Code Exchange)                            │
│                                                                │
│  Problem: Public clients can't securely store a client_secret  │
│  Solution: One-time cryptographic challenge per request        │
│                                                                │
│  1. Client generates random code_verifier (high entropy)       │
│  2. Client computes code_challenge = SHA256(code_verifier)     │
│  3. Client sends code_challenge with /authorize request        │
│  4. Auth server stores code_challenge                          │
│  5. Client exchanges code with code_verifier at /token         │
│  6. Auth server verifies: SHA256(code_verifier) == stored      │
│                                                                │
│  Even if an attacker intercepts the authorization code,        │
│  they can't exchange it without the code_verifier              │
└────────────────────────────────────────────────────────────────┘
```

### Client Credentials Flow (Machine-to-Machine)

No user involved — for services/APIs authenticating to each other:

```
┌─────────────┐                    ┌──────────────────┐
│  Service A  │── client_id +      │  Auth Server     │
│  (client)   │   client_secret ─▶ │                  │
│             │                    │                  │
│             │◀── access_token ── │                  │
│             │                    └──────────────────┘
│             │
│             │── access_token ──▶ Service B (resource server)
└─────────────┘
```

### Device Code Flow

For devices without a browser (IoT, CLI, TV apps):

```
1. Device shows a code: "Go to https://example.com/activate, enter: ABCD-1234"
2. User opens browser on phone/laptop, enters code
3. User authenticates in the browser
4. Device polls /token endpoint until approved
5. Device receives tokens
```

## The JWT (JSON Web Token)

JWTs are the standard format for OIDC ID tokens and often for access tokens:

```
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4ifQ.signature
└─────── Header ───────┘└──────────── Payload ─────────────────────────────┘└─ Signature ─┘
```

### JWT Structure

```json
// Header (base64url-decoded):
{
  "alg": "RS256",       // Signing algorithm
  "kid": "abc123",      // Key ID (which key signed this)
  "typ": "JWT"
}

// Payload (claims):
{
  "iss": "https://accounts.google.com",    // Issuer
  "sub": "user-12345",                     // Subject (unique user ID)
  "aud": "my-app-client-id",              // Audience (who this token is for)
  "exp": 1710500000,                       // Expiration (Unix timestamp)
  "iat": 1710496400,                       // Issued at
  "nonce": "random-value",                 // Replay protection

  // OIDC standard claims:
  "name": "John Doe",
  "email": "john@example.com",
  "email_verified": true,
  "groups": ["developers", "admins"],      // Custom claim (for K8s RBAC)
  "preferred_username": "johndoe"
}

// Signature:
// RS256(base64url(header) + "." + base64url(payload), private_key)
```

### JWT Validation

The resource server validates a JWT by:

1. Decode header → get `kid` and `alg`
2. Fetch signing keys from issuer's JWKS endpoint (`/.well-known/jwks.json`)
3. Verify signature using the public key matching `kid`
4. Check `exp` (not expired)
5. Check `iss` (matches expected issuer)
6. Check `aud` (matches my client ID or API identifier)

```bash
# Decode a JWT (without verification):
echo "$TOKEN" | cut -d'.' -f2 | base64 -d 2>/dev/null | jq .

# Fetch JWKS (public keys for verification):
curl -s https://accounts.google.com/.well-known/openid-configuration | jq .jwks_uri
curl -s https://accounts.google.com/oauth2/v3/certs | jq .
```

## OIDC Discovery

Every OIDC provider publishes a discovery document at a well-known URL:

```bash
curl -s https://accounts.google.com/.well-known/openid-configuration | jq .
```

```json
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "scopes_supported": ["openid", "email", "profile"],
  "response_types_supported": ["code", "token", "id_token"],
  "id_token_signing_alg_values_supported": ["RS256"]
}
```

## Kubernetes OIDC Integration

Kubernetes uses OIDC to authenticate users (not service accounts — those use SA tokens).

### How It Works

```
┌──────────┐     ┌──────────────┐     ┌───────────────┐     ┌──────────────┐
│  User    │────▶│  OIDC        │────▶│  kubectl      │────▶│  API Server  │
│ (browser)│     │  Provider    │     │  (with token) │     │  (validates  │
│          │     │ (Dex, Okta)  │     │               │     │   JWT)       │
└──────────┘     └──────────────┘     └───────────────┘     └──────────────┘
```

```
1. User authenticates with OIDC provider (via browser)
2. Provider returns ID Token (JWT) with claims
3. kubectl sends ID Token as Bearer token to API server
4. API server validates the JWT:
   - Checks signature against provider's JWKS
   - Checks issuer matches --oidc-issuer-url
   - Checks audience matches --oidc-client-id
   - Extracts username from --oidc-username-claim
   - Extracts groups from --oidc-groups-claim
5. API server maps identity to RBAC (user + groups)
```

### API Server OIDC Flags

```bash
# kube-apiserver configuration (on the control plane):
--oidc-issuer-url=https://dex.example.com
--oidc-client-id=kubernetes
--oidc-username-claim=email          # Which JWT claim = K8s username
--oidc-groups-claim=groups           # Which JWT claim = K8s groups
--oidc-username-prefix=oidc:         # Prefix to avoid collisions
--oidc-groups-prefix=oidc:           # Prefix for groups
--oidc-ca-file=/etc/ssl/certs/oidc-ca.pem  # If self-signed issuer
```

On EKS, you configure this via the cluster's identity provider association:

```bash
# Associate an OIDC identity provider with EKS:
aws eks associate-identity-provider-config \
  --cluster-name my-cluster \
  --oidc '{
    "identityProviderConfigName": "corporate-sso",
    "issuerUrl": "https://login.corp.example.com",
    "clientId": "kubernetes",
    "usernameClaim": "email",
    "groupsClaim": "groups",
    "usernamePrefix": "oidc:",
    "groupsPrefix": "oidc:"
  }'
```

### kubeconfig with OIDC

```yaml
users:
- name: oidc-user
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: kubectl
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://dex.example.com
      - --oidc-client-id=kubernetes
      - --oidc-extra-scope=groups
      - --oidc-extra-scope=email
```

Or using `kubelogin` (kubectl-oidc_login plugin):

```bash
# Install:
kubectl krew install oidc-login

# Setup:
kubectl oidc-login setup \
  --oidc-issuer-url=https://dex.example.com \
  --oidc-client-id=kubernetes
```

### RBAC with OIDC Groups

Once OIDC authentication maps users to K8s groups, RBAC controls access:

```yaml
# Grant the "developers" OIDC group read access:
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-developers-view
subjects:
- kind: Group
  name: "oidc:developers"    # Prefix + group claim value
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
---
# Grant a specific OIDC user admin in a namespace:
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: oidc-jane-admin
  namespace: production
subjects:
- kind: User
  name: "oidc:jane@example.com"    # Prefix + username claim value
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

## Common OIDC Providers for Kubernetes

| Provider | Use Case | Notes |
|----------|----------|-------|
| Dex | Self-hosted, multi-connector (LDAP, GitHub, SAML) | Popular for on-prem K8s |
| Keycloak | Full IAM platform, self-hosted | Heavy but feature-rich |
| Okta | Enterprise SSO (SaaS) | Easy setup, corporate environments |
| Auth0 | Developer-friendly (SaaS) | Good docs, free tier |
| Azure AD / Entra ID | Microsoft ecosystem | Native AKS integration |
| Google Workspace | Google ecosystem | Works with GKE natively |
| AWS Cognito | AWS ecosystem | Can be used with EKS |
| Pinniped | Kubernetes-native OIDC aggregator | Works with any cluster |

## Token Refresh Flow

```
┌────────────────────────────────────────────────────────────────┐
│  Access Token expired (HTTP 401 from API)                      │
│                                                                │
│  Client has a refresh_token:                                   │
│    POST /token                                                 │
│    grant_type=refresh_token                                    │
│    refresh_token=<stored-refresh-token>                        │
│    client_id=<my-client-id>                                    │
│                                                                │
│  Response:                                                     │
│    new access_token (fresh expiry)                             │
│    new id_token (fresh claims)                                 │
│    optionally new refresh_token (rotation)                     │
│                                                                │
│  No user interaction needed — transparent to the user          │
└────────────────────────────────────────────────────────────────┘
```

## Security Best Practices

| Practice | Why |
|----------|-----|
| Always use PKCE (even for confidential clients) | Prevents authorization code interception |
| Short access token lifetime (5-15 min) | Limits damage from stolen tokens |
| Validate `aud` claim | Prevents token reuse across different services |
| Validate `iss` claim | Ensures token came from expected provider |
| Use `state` parameter | Prevents CSRF on the callback |
| Store tokens securely (httpOnly cookies, not localStorage) | Prevents XSS token theft |
| Rotate refresh tokens | Limits lifetime of compromised refresh tokens |
| Use specific scopes (not wildcard) | Least privilege principle |

## Quick Reference

```bash
# OAuth2 = Authorization (access tokens, scopes)
# OIDC = Authentication (ID tokens, user identity) built on OAuth2

# Authorization Code Flow (most common):
# 1. Redirect to /authorize
# 2. User authenticates + consents
# 3. Redirect back with ?code=xxx
# 4. Exchange code for tokens at /token (server-to-server)
# 5. Use access_token for API calls

# PKCE (for public clients): adds code_verifier/code_challenge
# Client Credentials (M2M): client_id + client_secret → token directly

# JWT = Header.Payload.Signature (base64url encoded, dot-separated)
# Validate: signature (JWKS), exp, iss, aud

# OIDC Discovery: GET <issuer>/.well-known/openid-configuration
# JWKS (public keys): GET <jwks_uri from discovery>

# Kubernetes OIDC:
# API server validates ID Token JWT directly (no callback to provider)
# Maps claims to K8s user/groups for RBAC
# Configure via --oidc-* flags or EKS identity provider association

# Decode a JWT:
echo "$TOKEN" | cut -d'.' -f2 | base64 -d | jq .

# Common scopes: openid, profile, email, groups, offline_access
```
