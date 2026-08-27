# How Kubernetes RBAC Evaluation Works

The internal authorization flow — how the API server evaluates RBAC rules, resolves bindings, handles deny-by-default, aggregated ClusterRoles, and multiple authorizer chains.

## High-Level Flow

```
┌──────────┐     ┌───────────────┐     ┌──────────────────┐     ┌───────────┐
│  Request │────▶│ Authentication│────▶│  Authorization   │────▶│ Admission │
│          │     │ (WHO are you) │     │  (CAN you do it) │     │           │
└──────────┘     └───────────────┘     └──────────────────┘     └───────────┘
                                              │
                                    ┌─────────┼─────────┐
                                    ▼         ▼         ▼
                               Node Authz   RBAC    Webhook
                               (kubelet     (roles   (external)
                                only)       & bindings)
```

## Request Attributes

Every API request is decomposed into attributes that RBAC evaluates:

```
┌──────────────────────────────────────────────────────┐
│  Request: kubectl get pods -n production             │
│                                                      │
│  Attributes extracted:                               │
│    user:       "jane"                                │
│    groups:     ["developers", "system:authenticated"]│
│    verb:       "get"                                 │
│    resource:   "pods"                                │
│    apiGroup:   "" (core)                             │
│    namespace:  "production"                          │
│    name:       "" (all pods, not a specific one)     │
│    subresource: ""                                   │
└──────────────────────────────────────────────────────┘
```

| Attribute | Source | Example |
|-----------|--------|---------|
| `user` | Authentication result (CN from cert, token identity) | `jane`, `system:serviceaccount:default:my-sa` |
| `groups` | Authentication result (O from cert, token groups) | `["developers", "system:authenticated"]` |
| `verb` | HTTP method mapped to API verb | GET→get, POST→create, PUT→update, DELETE→delete, PATCH→patch |
| `resource` | URL path segment | `pods`, `deployments`, `services` |
| `apiGroup` | URL path | `""` (core), `apps`, `batch`, `rbac.authorization.k8s.io` |
| `namespace` | URL path or body | `default`, `production` |
| `name` | URL path (for specific resources) | `my-pod` |
| `subresource` | URL path suffix | `log`, `exec`, `status`, `scale` |

### HTTP Method to API Verb Mapping

| HTTP Method | API Verb | Notes |
|-------------|----------|-------|
| GET (specific) | `get` | `/api/v1/namespaces/default/pods/nginx` |
| GET (list) | `list` | `/api/v1/namespaces/default/pods` |
| GET (watch) | `watch` | `/api/v1/pods?watch=true` |
| POST | `create` | |
| PUT | `update` | |
| PATCH | `patch` | |
| DELETE (specific) | `delete` | |
| DELETE (collection) | `deletecollection` | |

## The Authorization Chain

The API server evaluates multiple authorizers in order:

```
┌────────────────────────────────────────────────────────┐
│  Authorization Chain (evaluated in order)              │
│                                                        │
│  1. Node Authorizer                                    │
│     → Only for kubelet requests                        │
│     → Grants kubelet access to its own node's pods     │
│     → Returns: Allow, Deny, or NoOpinion               │
│                                                        │
│  2. RBAC Authorizer                                    │
│     → Evaluates Roles, ClusterRoles, Bindings          │
│     → Returns: Allow or NoOpinion (never Deny)         │
│                                                        │
│  3. Webhook Authorizer (if configured)                 │
│     → Calls external service                           │
│     → Returns: Allow, Deny, or NoOpinion               │
│                                                        │
│  Decision logic:                                       │
│    - First "Allow" wins → request proceeds             │
│    - If all return "NoOpinion" → request DENIED        │
│    - An explicit "Deny" → request DENIED immediately   │
│                                                        │
│      RBAC never explicitly denies — only Allow or      │
│     NoOpinion. There are no "deny rules" in RBAC.      │
└────────────────────────────────────────────────────────┘
```

**Key insight**: RBAC is additive only. You cannot write a rule that says "deny access to X." You can only grant permissions. If no rule grants access, the default is deny.

## RBAC Evaluation Algorithm

```
┌─────────────────────────────────────────────────────────────────┐
│ For request (user=jane, groups=[devs], verb=get, resource=pods, │
│              namespace=production):                             │
│                                                                 │
│  1. Find all ClusterRoleBindings where:                         │
│     - subjects match user "jane" OR group "devs"                │
│     - OR group "system:authenticated"                           │
│                                                                 │
│  2. Find all RoleBindings in namespace "production" where:      │
│     - subjects match user "jane" OR group "devs"                │
│     - OR group "system:authenticated"                           │
│                                                                 │
│  3. For each matching binding:                                  │
│     - Load the referenced Role/ClusterRole                      │
│     - Check if any rule grants verb="get" on resource="pods"    │
│       in apiGroup=""                                            │
│                                                                 │
│  4. If ANY rule matches → ALLOW                                 │
│     If NO rule matches → NoOpinion (effectively deny)           │
└─────────────────────────────────────────────────────────────────┘
```

### Rule Matching Logic

A rule matches if ALL of these are true:
- `apiGroups` contains the request's apiGroup (or `"*"`)
- `resources` contains the request's resource (or `"*"`)
- `verbs` contains the request's verb (or `"*"`)
- `resourceNames` (if specified) contains the request's resource name

```yaml
rules:
- apiGroups: [""]           # Must match request apiGroup
  resources: ["pods"]       # Must match request resource
  verbs: ["get", "list"]    # Must match request verb
  # resourceNames: []       # If empty/absent, matches all names
```

## The Four RBAC Objects

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Namespace-scoped:              Cluster-scoped:             │
│                                                             │
│  ┌──────────┐  binds to  ┌───────────────┐                  │
│  │   Role   │◀───────────│  RoleBinding  │                  │
│  │(ns: prod)│            │  (ns: prod)   │                  │
│  └──────────┘            └───────────────┘                  │
│                                  │                          │
│                           subjects: [user, group, SA]       │
│                                                             │
│  ┌──────────────┐  binds to  ┌────────────────────┐         │
│  │ ClusterRole  │◀───────────│ ClusterRoleBinding │         │
│  │(cluster-wide)│            │ (cluster-wide)     │         │
│  └──────────────┘            └────────────────────┘         │
│         ▲                                                   │
│         │                                                   │
│         │  can also be referenced by:                       │
│         │                                                   │
│  ┌──────────────┐                                           │
│  │ RoleBinding  │  (grants ClusterRole permissions          │
│  │ (ns: prod)   │   but ONLY in "prod" namespace)           │
│  └──────────────┘                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Key Distinction: ClusterRole + RoleBinding

A RoleBinding can reference a ClusterRole. This grants the ClusterRole's permissions but scoped to the RoleBinding's namespace:

```yaml
# ClusterRole (reusable template)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
# RoleBinding scopes it to "production" namespace only
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-production
  namespace: production
subjects:
- kind: User
  name: jane
roleRef:
  kind: ClusterRole    # References cluster-wide role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Jane can read pods in `production` only — not in other namespaces.

## Subject Types

```yaml
subjects:
# User (from authentication — no User object in K8s)
- kind: User
  name: "jane"
  apiGroup: rbac.authorization.k8s.io

# Group (from authentication — O field in x509, groups in token)
- kind: Group
  name: "developers"
  apiGroup: rbac.authorization.k8s.io

# ServiceAccount (actual K8s object)
- kind: ServiceAccount
  name: "my-app"
  namespace: "default"
```

### Built-in Groups

| Group | Members |
|-------|---------|
| `system:authenticated` | All authenticated users |
| `system:unauthenticated` | Anonymous requests (if allowed) |
| `system:masters` | Full admin — bypasses RBAC entirely |
| `system:serviceaccounts` | All service accounts in all namespaces |
| `system:serviceaccounts:<ns>` | All service accounts in namespace `<ns>` |

**Warning**: `system:masters` is checked BEFORE RBAC. Members get unrestricted access — RBAC rules don't apply to them at all.

## Aggregated ClusterRoles

ClusterRoles can be automatically composed via label selectors:

```yaml
# Base ClusterRole with aggregation rule
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-view
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: []  # Rules are auto-populated from matching ClusterRoles
---
# Contributing ClusterRole (auto-aggregated into monitoring-view)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-pods
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

The built-in `admin`, `edit`, and `view` ClusterRoles use aggregation. CRD authors can extend them by labeling their ClusterRoles:

```yaml
# Automatically adds CRD permissions to the "admin" ClusterRole
metadata:
  labels:
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
```

```bash
# See what's aggregated into "admin":
kubectl get clusterrole admin -o yaml | grep -A5 aggregationRule
kubectl get clusterroles -l rbac.authorization.k8s.io/aggregate-to-admin=true
```

## Non-Resource URLs

Some API server endpoints aren't resource-based (health checks, metrics, API discovery):

```yaml
rules:
- nonResourceURLs: ["/healthz", "/healthz/*"]
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
- nonResourceURLs: ["/api", "/api/*", "/apis", "/apis/*"]
  verbs: ["get"]  # API discovery
```

These can only be granted via ClusterRoles (they're not namespace-scoped).

## Subresources

Some resources have subresources with separate permissions:

```yaml
rules:
# Read pod logs (subresource)
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]

# Exec into pods (subresource)
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]

# Scale deployments (subresource)
- apiGroups: ["apps"]
  resources: ["deployments/scale"]
  verbs: ["get", "update", "patch"]

# Approve CSRs (subresource)
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/approval"]
  verbs: ["update"]
```

Common subresources:

| Resource | Subresource | Verb | What It Allows |
|----------|-------------|------|----------------|
| pods | log | get | `kubectl logs` |
| pods | exec | create | `kubectl exec` |
| pods | portforward | create | `kubectl port-forward` |
| pods | attach | create | `kubectl attach` |
| deployments | scale | update | `kubectl scale` |
| nodes | proxy | create | Proxy to kubelet |
| */status | status | update/patch | Update status subresource |

## Evaluation Order and Performance

The RBAC authorizer processes in this order:

```
1. Check ClusterRoleBindings (cluster-wide permissions)
   - Quick check for superuser/admin access
   
2. Check RoleBindings in the request's namespace
   - Namespace-scoped permissions

3. For each binding, resolve the Role/ClusterRole
   - Rules are loaded and matched against request attributes
```

The API server caches RBAC rules in memory (via informers watching RBAC objects). Rule evaluation is fast — no etcd calls per request.

## Debugging RBAC

```bash
# Check if you can perform an action:
kubectl auth can-i create pods
kubectl auth can-i create pods -n production
kubectl auth can-i create pods --as jane
kubectl auth can-i create pods --as system:serviceaccount:default:my-sa

# List all permissions for a user:
kubectl auth can-i --list --as jane
kubectl auth can-i --list --as system:serviceaccount:default:my-sa -n production

# Find bindings for a user/group/SA:
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.subjects[]? | .name == "jane") | .metadata.name'
kubectl get rolebindings -A -o json | jq -r '.items[] | select(.subjects[]? | .name == "my-sa") | "\(.metadata.namespace)/\(.metadata.name)"'

# See what a ClusterRole grants:
kubectl describe clusterrole admin
kubectl get clusterrole edit -o yaml

# Simulate authorization (requires API access):
kubectl create -f - <<EOF
apiVersion: authorization.k8s.io/v1
kind: SubjectAccessReview
spec:
  user: "jane"
  groups: ["developers"]
  resourceAttributes:
    verb: "create"
    resource: "deployments"
    group: "apps"
    namespace: "production"
EOF

# Check escalation prevention:
# You cannot grant permissions you don't have yourself
kubectl create clusterrolebinding --clusterrole=cluster-admin --user=jane
# Error: user cannot grant cluster-admin (unless they already have it)
```

## Escalation Prevention

RBAC prevents privilege escalation: you cannot create a RoleBinding/ClusterRoleBinding granting permissions you don't already have, UNLESS you have the `escalate` verb on the binding's role:

```yaml
# Required to grant roles you don't have yourself:
rules:
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles"]
  verbs: ["escalate"]  # Dangerous — allows granting any permission
```

Similarly, you need `bind` to create bindings that reference roles you don't have full permissions on:

```yaml
rules:
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterrolebindings"]
  verbs: ["create"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles"]
  verbs: ["bind"]
  resourceNames: ["admin", "edit"]  # Can only bind these specific roles
```

## Common Patterns

### Namespace Admin

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: namespace-admin
  namespace: team-alpha
subjects:
- kind: Group
  name: "team-alpha"
roleRef:
  kind: ClusterRole
  name: admin          # Built-in admin role (not cluster-admin)
```

### Read-Only Cluster Access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-viewer
subjects:
- kind: Group
  name: "all-developers"
roleRef:
  kind: ClusterRole
  name: view           # Built-in view role
```

### CI/CD ServiceAccount (Deploy Only)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "update", "patch"]
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer
  namespace: production
subjects:
- kind: ServiceAccount
  name: ci-pipeline
  namespace: ci-system
roleRef:
  kind: Role
  name: deployer
```

## Quick Reference

```bash
# Check access
kubectl auth can-i <verb> <resource> [-n namespace] [--as user]
kubectl auth can-i --list [--as user] [-n namespace]

# List RBAC objects
kubectl get roles,rolebindings -n <namespace>
kubectl get clusterroles,clusterrolebindings

# Describe roles (see rules)
kubectl describe role <name> -n <namespace>
kubectl describe clusterrole <name>

# Find bindings for a subject
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq -r '.items[] | select(.subjects[]?.name == "TARGET") | 
  "\(.kind) \(.metadata.name) → \(.roleRef.name)"'

# Create role imperatively
kubectl create role pod-reader --verb=get,list --resource=pods -n default
kubectl create rolebinding jane-reader --role=pod-reader --user=jane -n default

# Create cluster-level
kubectl create clusterrole node-reader --verb=get,list --resource=nodes
kubectl create clusterrolebinding jane-nodes --clusterrole=node-reader --user=jane

# Key facts:
# - RBAC is additive (no deny rules)
# - Default is deny (no matching rule = denied)
# - system:masters bypasses RBAC entirely
# - Cached in API server memory (fast evaluation)
# - Escalation prevention: can't grant what you don't have
```
