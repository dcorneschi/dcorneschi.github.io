# How kubectl Discovers and Resolves API Resources

How kubectl loads your kubeconfig, discovers available API groups and resources, maps short names to full GVR paths, and negotiates versions — the mechanics behind every kubectl command.

## High-Level Flow

```
kubectl get pods
    │
    ├── 1. Load kubeconfig (~/.kube/config)
    ├── 2. Resolve context → cluster + user
    ├── 3. Discover API groups (GET /api, /apis)
    ├── 4. Map "pods" → Group: "", Version: "v1", Resource: "pods"
    ├── 5. Build URL: /api/v1/namespaces/default/pods
    └── 6. Send authenticated request
```

## Step 1: Kubeconfig Loading

kubectl loads configuration in this order (first found wins per field):

```
1. --kubeconfig flag             (explicit path)
2. $KUBECONFIG env variable      (colon-separated list, merged)
3. ~/.kube/config                (default location)
```

### Kubeconfig Structure

```yaml
apiVersion: v1
kind: Config
current-context: production

clusters:
- name: production
  cluster:
    server: https://api.prod.example.com:6443
    certificate-authority-data: <base64-CA>

users:
- name: jane
  user:
    exec:                          # Dynamic credential plugin
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws
      args: ["eks", "get-token", "--cluster-name", "prod"]

contexts:
- name: production
  context:
    cluster: production
    user: jane
    namespace: default             # Default namespace if -n not specified
```

### Resolution Order

```
┌──────────────────────────────────────────────────────┐
│  kubectl get pods -n kube-system                     │
│                                                      │
│  1. Context: "production" (from current-context)     │
│  2. Cluster: "production" → server URL + CA          │
│  3. User: "jane" → exec credential plugin            │
│  4. Namespace: "kube-system" (from -n flag)          │
│     (would be "default" from context if no -n)       │
└──────────────────────────────────────────────────────┘
```

```bash
# See resolved config:
kubectl config view --minify

# See current context:
kubectl config current-context

# See all contexts:
kubectl config get-contexts

# See what server kubectl would talk to:
kubectl cluster-info
```

## Step 2: API Discovery

Before kubectl can make any resource request, it needs to know what resources exist and which API group/version serves them. It does this via **API discovery**.

### Discovery Endpoints

```
GET /api                    → Core API group (v1)
GET /api/v1                 → Resources in core group
GET /apis                   → All named API groups
GET /apis/apps/v1           → Resources in apps/v1
GET /apis/batch/v1          → Resources in batch/v1
...
```

### Discovery Response Example

```bash
kubectl get --raw /api/v1 | jq '.resources[] | {name, namespaced, kind, verbs}'
```

```json
{
  "name": "pods",
  "namespaced": true,
  "kind": "Pod",
  "verbs": ["create", "delete", "get", "list", "patch", "update", "watch"]
}
{
  "name": "pods/log",
  "namespaced": true,
  "kind": "Pod",
  "verbs": ["get"]
}
{
  "name": "services",
  "namespaced": true,
  "kind": "Service",
  "verbs": ["create", "delete", "get", "list", "patch", "update", "watch"]
}
```

### Discovery Cache

kubectl caches discovery results locally to avoid hitting the API server every time:

```bash
# Cache location:
ls ~/.kube/cache/discovery/<cluster-hash>/

# Contents:
# servergroups.json          — API group list
# v1/serverresources.json    — core/v1 resources
# apps/v1/serverresources.json
# batch/v1/serverresources.json
# ...

# Cache TTL: 6 hours (then re-fetched)

# Force refresh:
kubectl api-resources  # Always fetches fresh
rm -rf ~/.kube/cache/discovery/  # Clear cache manually
```

### Aggregated Discovery (Kubernetes 1.27+)

Instead of many requests, kubectl can fetch all discovery info in two calls:

```
GET /api?timeout=32s              → Core group
GET /apis?timeout=32s             → All other groups (aggregated)
```

This reduces API server load significantly in clusters with many CRDs.

## Step 3: GVR — Group, Version, Resource

Every Kubernetes resource is identified by three components:

```
┌────────────────────────────────────────────────────────────┐
│  GVR = Group + Version + Resource                          │
│                                                            │
│  pods        → Group: ""      Version: "v1"    Resource: "pods"
│  deployments → Group: "apps"  Version: "v1"    Resource: "deployments"
│  jobs        → Group: "batch" Version: "v1"    Resource: "jobs"
│  ingresses   → Group: "networking.k8s.io" Version: "v1" Resource: "ingresses"
│                                                            │
│  GVK = Group + Version + Kind (used in YAML)               │
│                                                            │
│  apiVersion: v1           → Group: "" (core), Version: "v1"
│  kind: Pod                → Kind: "Pod"                    │
│                                                            │
│  apiVersion: apps/v1      → Group: "apps", Version: "v1"   │
│  kind: Deployment         → Kind: "Deployment"             │
└────────────────────────────────────────────────────────────┘
```

### How kubectl Maps User Input to GVR

```
User types: kubectl get deploy

kubectl:
  1. "deploy" → check short names table
  2. Match: "deploy" is short for "deployments"
  3. Look up "deployments" in discovery cache
  4. Found in group "apps", version "v1"
  5. GVR: apps/v1/deployments
  6. URL: /apis/apps/v1/namespaces/default/deployments
```

### Short Names and Aliases

```bash
# See all short names:
kubectl api-resources | grep -v "^NAME" | awk '{if($2 != "") print $2, "→", $1}'

# Common short names:
# po    → pods
# svc   → services
# deploy → deployments
# ds    → daemonsets
# sts   → statefulsets
# rs    → replicasets
# cm    → configmaps
# sa    → serviceaccounts
# ns    → namespaces
# no    → nodes
# pv    → persistentvolumes
# pvc   → persistentvolumeclaims
# ing   → ingresses
# hpa   → horizontalpodautoscalers
# ep    → endpoints
# netpol → networkpolicies
```

## Step 4: URL Construction

kubectl builds the API URL from the resolved GVR:

### Namespaced Resources

```
/apis/{group}/{version}/namespaces/{namespace}/{resource}
/apis/{group}/{version}/namespaces/{namespace}/{resource}/{name}

# Core group (no "apis" prefix, no group in path):
/api/{version}/namespaces/{namespace}/{resource}
/api/{version}/namespaces/{namespace}/{resource}/{name}
```

Examples:

```
kubectl get pods -n default
  → GET /api/v1/namespaces/default/pods

kubectl get pod nginx -n default
  → GET /api/v1/namespaces/default/pods/nginx

kubectl get deployments -n production
  → GET /apis/apps/v1/namespaces/production/deployments

kubectl logs nginx -n default
  → GET /api/v1/namespaces/default/pods/nginx/log
```

### Cluster-Scoped Resources

```
/apis/{group}/{version}/{resource}
/api/{version}/{resource}
```

Examples:

```
kubectl get nodes
  → GET /api/v1/nodes

kubectl get clusterroles
  → GET /apis/rbac.authorization.k8s.io/v1/clusterroles

kubectl get namespaces
  → GET /api/v1/namespaces
```

```bash
# See the actual URL kubectl constructs:
kubectl get pods -v=6
# I0315 10:00:00.000000  GET https://api.example.com:6443/api/v1/namespaces/default/pods 200 OK in 50ms
```

## Step 5: Version Negotiation

When a resource exists in multiple API versions, kubectl picks the preferred version:

```
# CronJobs exist in both:
# - batch/v1 (stable)
# - batch/v1beta1 (deprecated)

# kubectl prefers the storage version (what the API server marks as preferred)
# Usually the most stable version
```

The API server returns a preference order in its discovery response. kubectl follows it.

```bash
# See which versions serve a resource:
kubectl api-resources | grep cronjob
# cronjobs   cj   batch/v1   true   CronJob

# See all API versions for a group:
kubectl api-versions | grep batch
# batch/v1
```

### Version Conversion

The API server converts between versions transparently:

```
kubectl apply -f job.yaml (apiVersion: batch/v1)
  → API server stores as batch/v1 (storage version)
  → kubectl get job -o yaml → returns batch/v1

# If you request a deprecated version:
kubectl get cronjobs.v1beta1.batch  → API server converts from v1 to v1beta1
```

## API Groups

```
┌────────────────────────────────────────────────────────────────┐
│  Core Group (""): /api/v1                                      │
│    pods, services, configmaps, secrets, namespaces,            │
│    nodes, persistentvolumes, serviceaccounts, events           │
│                                                                │
│  Named Groups: /apis/{group}/{version}                         │
│    apps/v1: deployments, statefulsets, daemonsets, replicasets │
│    batch/v1: jobs, cronjobs                                    │
│    networking.k8s.io/v1: ingresses, networkpolicies            │
│    rbac.authorization.k8s.io/v1: roles, clusterroles, bindings │
│    autoscaling/v2: horizontalpodautoscalers                    │
│    storage.k8s.io/v1: storageclasses, csinodes, csidrivers     │
│    policy/v1: poddisruptionbudgets                             │
│    certificates.k8s.io/v1: certificatesigningrequests          │
│    coordination.k8s.io/v1: leases                              │
│    discovery.k8s.io/v1: endpointslices                         │
│    admissionregistration.k8s.io/v1: webhook configs            │
│    apiextensions.k8s.io/v1: customresourcedefinitions          │
│                                                                │
│  Custom Groups (from CRDs):                                    │
│    karpenter.sh/v1: nodepools, nodeclaims                      │
│    cert-manager.io/v1: certificates, issuers                   │
│    argoproj.io/v1alpha1: applications                          │
└────────────────────────────────────────────────────────────────┘
```

## CRD Discovery

When a CRD is installed, the API server dynamically registers new endpoints. kubectl discovers them on next API discovery refresh:

```bash
# Install a CRD:
kubectl apply -f my-crd.yaml

# kubectl discovers it (may need cache refresh):
kubectl api-resources | grep myresource

# If not found, force discovery refresh:
kubectl get myresources  # Triggers fresh discovery
# Or: rm -rf ~/.kube/cache/discovery/
```

## kubectl explain — Schema Introspection

`kubectl explain` uses the OpenAPI spec published by the API server:

```bash
# Top-level fields:
kubectl explain pod

# Nested fields:
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.resources.limits

# Recursive (show all fields):
kubectl explain pod.spec --recursive

# Specific API version:
kubectl explain deployment --api-version=apps/v1
```

The API server serves the OpenAPI spec at:
```
GET /openapi/v2    → Full OpenAPI v2 spec (large)
GET /openapi/v3    → OpenAPI v3 (Kubernetes 1.27+)
```

## Debugging API Discovery

```bash
# See all available resources:
kubectl api-resources

# See resources in a specific group:
kubectl api-resources --api-group=apps

# See only namespaced resources:
kubectl api-resources --namespaced=true

# See verbs available for a resource:
kubectl api-resources -o wide | grep deployments

# Raw discovery call:
kubectl get --raw /apis | jq '.groups[].name'
kubectl get --raw /api/v1 | jq '.resources[].name'

# Check if a specific resource exists:
kubectl api-resources | grep -i "myresource"

# See kubectl's HTTP requests (verbosity levels):
kubectl get pods -v=6    # URL + response code
kubectl get pods -v=7    # + request headers
kubectl get pods -v=8    # + request/response bodies
kubectl get pods -v=9    # + curl command equivalent

# Check cache:
ls ~/.kube/cache/discovery/
cat ~/.kube/cache/discovery/*/servergroups.json | jq .
```

## Quick Reference

```bash
# Kubeconfig loading order:
# 1. --kubeconfig flag
# 2. $KUBECONFIG env (colon-separated, merged)
# 3. ~/.kube/config

# Discovery endpoints:
# GET /api       → core group
# GET /apis      → named groups
# GET /api/v1    → core resources
# GET /apis/apps/v1 → apps group resources

# GVR (Group/Version/Resource) → URL mapping:
# core: /api/v1/namespaces/{ns}/{resource}
# named: /apis/{group}/{version}/namespaces/{ns}/{resource}
# cluster-scoped: /api/v1/{resource} or /apis/{group}/{version}/{resource}

# Discovery cache: ~/.kube/cache/discovery/ (6h TTL)

# Key commands:
kubectl api-resources                    # All resources
kubectl api-resources --api-group=apps   # Specific group
kubectl api-versions                     # All group/versions
kubectl explain <resource>.<path>        # Schema
kubectl get <resource> -v=6              # See URL constructed
```
