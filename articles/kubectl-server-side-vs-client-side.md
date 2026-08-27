# kubectl Server-Side vs Client-Side Operations

When operations run locally in kubectl vs when they run on the API server — dry-run, apply, diff, field managers, and when each approach matters.

## Overview

```
┌──────────────────────────────────────────────────────────────────┐
│  Client-Side (runs in kubectl)                                   │
│  - Validation against local schema                               │
│  - YAML parsing and defaulting                                   │
│  - 3-way merge patch computation (apply)                         │
│  - Dry-run (no API call)                                         │
│  - Diff rendering                                                │
│                                                                  │
│  Server-Side (runs in API server)                                │
│  - Schema validation against actual cluster version              │
│  - Admission controllers (mutating + validating)                 │
│  - Defaulting (fill in omitted fields)                           │
│  - Server-side apply (merge with field ownership)                │
│  - Server-side dry-run (full pipeline without persisting)        │
└──────────────────────────────────────────────────────────────────┘
```

## Dry-Run: Client vs Server

### Client-Side Dry-Run

```bash
kubectl apply -f deployment.yaml --dry-run=client
```

What happens:
1. kubectl parses YAML
2. Validates against locally-cached OpenAPI schema
3. Prints what it WOULD send
4. **No API server contact**

```
┌──────────┐
│  kubectl │─── parse + validate locally ──→ print result
│          │
│ (no network call)
└──────────┘
```

**Limitations:**
- Doesn't know about admission controllers (mutations, defaults)
- Doesn't know about CRDs that aren't in the local cache
- Doesn't validate against current cluster state
- May produce output different from what the API server would actually accept

```bash
# Client dry-run output:
kubectl apply -f deployment.yaml --dry-run=client -o yaml
# Shows: what kubectl would SEND (before server-side mutation)
```

### Server-Side Dry-Run

```bash
kubectl apply -f deployment.yaml --dry-run=server
```

What happens:
1. kubectl sends the request to the API server
2. API server runs the FULL admission pipeline:
   - Authentication ✓
   - Authorization ✓
   - Mutating admission controllers ✓ (defaults, sidecar injection)
   - Schema validation ✓
   - Validating admission controllers ✓ (OPA, Kyverno)
3. API server does NOT persist to etcd
4. Returns the fully processed object

```
┌──────────┐     ┌───────────────┐
│  kubectl │────▶│   API Server  │──→ auth → authz → mutate → validate
│          │◀────│               │
│          │     │  (does NOT    │
│  prints  │     │   write etcd) │
│  result  │     └───────────────┘
└──────────┘
```

**Advantages:**
- Shows the ACTUAL object as it would be stored (with mutations applied)
- Catches admission webhook denials before real apply
- Validates CRDs correctly
- Shows defaults that would be injected (resource limits, sidecars)

```bash
# Server dry-run output:
kubectl apply -f deployment.yaml --dry-run=server -o yaml
# Shows: what would be STORED in etcd (after all mutations)

# Compare the two:
diff <(kubectl apply -f d.yaml --dry-run=client -o yaml) \
     <(kubectl apply -f d.yaml --dry-run=server -o yaml)
# Differences = what admission controllers add/modify
```

### When to Use Each

| Scenario | Use | Why |
|----------|-----|-----|
| Quick YAML syntax check | `--dry-run=client` | Fast, no cluster needed |
| CI/CD pre-flight check | `--dry-run=server` | Catches policy violations |
| See what admission controllers would add | `--dry-run=server` | Shows mutations |
| Offline validation (no cluster access) | `--dry-run=client` | No network required |
| Generate YAML templates | `--dry-run=client -o yaml` | Quick scaffolding |
| Verify RBAC allows the action | `--dry-run=server` | Runs real authorization |

## Apply: Client-Side vs Server-Side

### Client-Side Apply (Default)

```bash
kubectl apply -f deployment.yaml
```

The merge logic runs in kubectl:

```
┌──────────────────────────────────────────────────────────────┐
│  Client-Side Apply (kubectl computes the patch)              │
│                                                              │
│  Inputs:                                                     │
│    1. Local file (what you want)                             │
│    2. last-applied-configuration annotation (what you sent   │
│       last time)                                             │
│    3. Live object from API server (current state)            │
│                                                              │
│  kubectl computes:                                           │
│    - Strategic Merge Patch (what changed between inputs)     │
│    - Sends PATCH request with only the differences           │
│                                                              │
│  Tracking:                                                   │
│    - Stores entire local file in annotation:                 │
│      kubectl.kubernetes.io/last-applied-configuration        │
└──────────────────────────────────────────────────────────────┘
```

```bash
# See the last-applied annotation:
kubectl get deployment my-app -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}' | jq .
```

**Problems with client-side apply:**
- Annotation grows large (entire spec stored as JSON)
- Conflict detection is weak (based on annotation, not field ownership)
- Multiple tools modifying the same object can conflict silently
- Removing a field requires it to be in the last-applied annotation

### Server-Side Apply (SSA)

```bash
kubectl apply -f deployment.yaml --server-side
```

The merge logic runs in the API server:

```
┌──────────────────────────────────────────────────────────────┐
│  Server-Side Apply (API server computes the merge)           │
│                                                              │
│  Inputs:                                                     │
│    1. The full object from the client                        │
│    2. The live object in etcd                                │
│    3. managedFields (who owns which fields)                  │
│                                                              │
│  API server:                                                 │
│    - Tracks which "field manager" owns each field            │
│    - Detects conflicts (two managers setting same field)     │
│    - Applies the merge with proper ownership transfer        │
│                                                              │
│  Tracking:                                                   │
│    - Uses metadata.managedFields (not an annotation)         │
│    - No last-applied-configuration annotation needed         │
└──────────────────────────────────────────────────────────────┘
```

```bash
# See managed fields:
kubectl get deployment my-app -o yaml | grep -A 30 managedFields

# See who manages which fields:
kubectl get deployment my-app -o jsonpath='{.metadata.managedFields}' | jq '.[].manager'
```

### Comparison

| Feature | Client-Side Apply | Server-Side Apply |
|---------|:-----------------:|:-----------------:|
| Merge runs in | kubectl | API server |
| Conflict tracking | `last-applied-configuration` annotation | `managedFields` metadata |
| Multiple managers | Silently overwrites | Detects conflicts |
| Field ownership | None (annotation-based) | Per-field, per-manager |
| Annotation size | Large (full spec stored) | None (no annotation) |
| Works with CRDs | Yes | Yes (better for complex CRDs) |
| Available since | Always | Kubernetes 1.22+ (stable) |
| Default in kubectl | Yes (without --server-side) | Must opt-in |

### Field Managers

With server-side apply, every actor that modifies an object has a "field manager" name:

```bash
# kubectl's default manager name:
kubectl apply -f deployment.yaml --server-side --field-manager=my-ci-tool

# See all managers:
kubectl get deployment my-app -o jsonpath='{.metadata.managedFields[*].manager}'
# kubectl  kube-controller-manager  my-ci-tool
```

Each manager "owns" the fields it sets:
- `kubectl` owns fields from your YAML file
- `kube-controller-manager` owns status fields
- `kube-scheduler` owns `spec.nodeName`
- HPA owns `spec.replicas` (if HPA is active)

### Conflict Detection

When two managers try to set the same field:

```bash
# Manager A sets replicas=3:
kubectl apply -f deploy.yaml --server-side --field-manager=team-a
# spec.replicas is now owned by "team-a"

# Manager B tries to set replicas=5:
kubectl apply -f deploy-b.yaml --server-side --field-manager=team-b
# Error: conflict on field spec.replicas (owned by team-a)

# Force take ownership:
kubectl apply -f deploy-b.yaml --server-side --field-manager=team-b --force-conflicts
# spec.replicas now owned by "team-b", value=5
```

## kubectl diff

```bash
kubectl diff -f deployment.yaml
```

Shows what would change if you applied the file:

```
┌──────────┐     ┌───────────────┐
│  kubectl │────▶│   API Server  │
│  diff    │     │               │
│          │     │ dry-run=server│
│          │◀────│ (returns what │
│          │     │   would exist)│
│  compares│     └───────────────┘
│  live vs │
│  dry-run │
│  result  │
└──────────┘
```

`kubectl diff` actually:
1. Fetches the live object
2. Sends a server-side dry-run with your file
3. Diffs the two results
4. Shows unified diff output

```bash
# Example output:
kubectl diff -f deployment.yaml
# -  replicas: 3
# +  replicas: 5
# -  image: my-app:v1
# +  image: my-app:v2
```

Exit codes:
- 0: no differences
- 1: differences found
- \>1: error

```bash
# Use in CI/CD to check if apply would change anything:
if kubectl diff -f deployment.yaml > /dev/null 2>&1; then
  echo "No changes needed"
else
  echo "Changes detected, applying..."
  kubectl apply -f deployment.yaml
fi
```

### KUBECTL_EXTERNAL_DIFF

You can use a custom diff tool:

```bash
# Use colordiff:
export KUBECTL_EXTERNAL_DIFF="colordiff -u"
kubectl diff -f deployment.yaml

# Use dyff (structured YAML diff):
export KUBECTL_EXTERNAL_DIFF="dyff between"
kubectl diff -f deployment.yaml
```

## Validation: Client vs Server

### Client-Side Validation

```bash
# Validate locally (default in kubectl 1.27+):
kubectl apply -f deployment.yaml --validate=true

# Explicitly client-only:
kubectl apply -f deployment.yaml --validate=client
```

Uses the locally-cached OpenAPI schema. Fast but may be outdated.

### Server-Side Validation

```bash
kubectl apply -f deployment.yaml --validate=server
```

Sends to API server for validation. Catches:
- Unknown fields (strict validation)
- CRD schema violations
- Incorrect types for known fields

```bash
# Strict server validation (reject unknown fields):
kubectl apply -f deployment.yaml --validate=strict

# This catches typos like:
# spec:
#   replcias: 3    ← typo caught by strict validation
```

## create vs apply vs replace

```
┌──────────────────────────────────────────────────────────────────┐
│  create                                                          │
│  - HTTP POST                                                     │
│  - Fails if object exists                                        │
│  - Imperative: "create this exact thing"                         │
│  - No merge, no field tracking                                   │
│                                                                  │
│  apply (client-side)                                             │
│  - HTTP PATCH (strategic merge patch)                            │
│  - Creates if missing, patches if exists                         │
│  - Declarative: "make it look like this"                         │
│  - Tracks via last-applied-configuration annotation              │
│                                                                  │
│  apply --server-side                                             │
│  - HTTP PATCH (apply patch type)                                 │
│  - Creates if missing, patches if exists                         │
│  - Declarative with field ownership                              │
│  - Tracks via managedFields, conflict detection                  │
│                                                                  │
│  replace                                                         │
│  - HTTP PUT                                                      │
│  - Fails if object doesn't exist                                 │
│  - Replaces ENTIRE object (drops fields you don't include)       │
│  - No merge — destructive                                        │
│                                                                  │
│  edit                                                            │
│  - GET → open in editor → PUT                                    │
│  - Interactive replacement                                       │
│  - Uses strategic merge internally                               │
│                                                                  │
│  patch                                                           │
│  - HTTP PATCH (various strategies)                               │
│  - Targeted field updates                                        │
│  - Types: strategic, merge, json                                 │
└──────────────────────────────────────────────────────────────────┘
```

### Patch Types

```bash
# Strategic merge patch (default — K8s-aware array handling):
kubectl patch deployment my-app -p '{"spec":{"replicas":5}}'
kubectl patch deployment my-app --type=strategic -p '...'

# JSON merge patch (RFC 7386 — arrays are replaced entirely):
kubectl patch deployment my-app --type=merge -p '{"spec":{"replicas":5}}'

# JSON patch (RFC 6902 — explicit operations):
kubectl patch deployment my-app --type=json \
  -p='[{"op":"replace","path":"/spec/replicas","value":5}]'
```

| Patch Type | Array Behavior | Use Case |
|-----------|---------------|----------|
| Strategic merge | Merge by key (e.g., container name) | Default for kubectl, preserves unlisted containers |
| JSON merge | Replace entire array | When you want to overwrite all containers |
| JSON patch | Explicit add/remove/replace operations | Precise surgery on specific paths |

## Migrating from Client-Side to Server-Side Apply

```bash
# Step 1: Check current state (last-applied annotation exists):
kubectl get deployment my-app -o jsonpath='{.metadata.annotations}' | jq 'keys'

# Step 2: Start using --server-side:
kubectl apply -f deployment.yaml --server-side --field-manager=kubectl-client

# Step 3: On first SSA call, it migrates:
# - Reads last-applied-configuration annotation
# - Converts to managedFields entries
# - Removes the annotation (eventually)

# If conflicts arise during migration:
kubectl apply -f deployment.yaml --server-side --force-conflicts
```

## Quick Reference

```bash
# Dry-run:
kubectl apply -f x.yaml --dry-run=client   # Local only (fast, no cluster)
kubectl apply -f x.yaml --dry-run=server   # Full pipeline (accurate, needs cluster)

# Apply:
kubectl apply -f x.yaml                    # Client-side apply (default)
kubectl apply -f x.yaml --server-side      # Server-side apply (field ownership)
kubectl apply -f x.yaml --server-side --force-conflicts  # Take ownership

# Diff:
kubectl diff -f x.yaml                     # Show what would change

# Validate:
kubectl apply -f x.yaml --validate=client  # Local schema
kubectl apply -f x.yaml --validate=server  # API server schema
kubectl apply -f x.yaml --validate=strict  # Reject unknown fields

# Patch types:
kubectl patch deploy x --type=strategic -p '...'  # K8s-aware merge (default)
kubectl patch deploy x --type=merge -p '...'      # JSON merge (replace arrays)
kubectl patch deploy x --type=json -p '[...]'     # Explicit operations

# Field managers (SSA):
kubectl apply -f x.yaml --server-side --field-manager=my-tool
kubectl get deploy x -o jsonpath='{.metadata.managedFields[*].manager}'

# Key differences:
# Client-side: merge in kubectl, last-applied annotation, no conflict detection
# Server-side: merge in API server, managedFields, conflict detection
# dry-run=client: no network, no mutations shown, fast
# dry-run=server: full pipeline, shows mutations, needs cluster access
```
