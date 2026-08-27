# What Happens When You Run kubectl apply

A step-by-step breakdown of every component involved when you run `kubectl apply -f manifest.yaml` — from your terminal to etcd and back.

## High-Level Flow

```
┌──────────┐     ┌───────────────┐     ┌──────────────────┐     ┌───────────┐
│  kubectl │────▶│   API Server  │────▶│  Admission       │────▶│   etcd    │
│  (client)│     │  (kube-api)   │     │  Controllers     │     │  (store)  │
└──────────┘     └───────────────┘     └──────────────────┘     └───────────┘
                                                                       │
                                                                       ▼
                                              ┌──────────────────────────────┐
                                              │  Controllers (watch loop)    │
                                              │  - Deployment controller     │
                                              │  - ReplicaSet controller     │
                                              │  - Scheduler                 │
                                              │  - Kubelet                   │
                                              └──────────────────────────────┘
```

## Detailed Step-by-Step

### Step 1: kubectl Client-Side Processing

```
┌─────────────────────────────────────────────────┐
│  kubectl apply -f deployment.yaml               │
│                                                 │
│  1. Read and parse YAML file                    │
│  2. Validate basic structure (kind, apiVersion) │
│  3. Look up API endpoint for the resource       │
│  4. Load kubeconfig (~/.kube/config)            │
│  5. Resolve cluster, user, context              │
│  6. Convert YAML to JSON                        │
│  7. Apply client-side defaulting                │
│  8. Compute 3-way merge patch (if update)       │
│  9. Send HTTP request to API server             │
└─────────────────────────────────────────────────┘
```

**What kubectl does locally:**

```bash
# kubectl reads your kubeconfig to find:
# - API server URL (cluster.server)
# - Auth credentials (user.token, user.exec, user.client-certificate)
# - CA certificate for TLS verification

# You can see what kubectl would send without actually sending it:
kubectl apply -f deployment.yaml --dry-run=client -o json

# See the HTTP request details:
kubectl apply -f deployment.yaml -v=8
```

**The 3-way merge (key to how `apply` works):**

kubectl apply uses three data sources to compute what to send:
1. **Last applied configuration** — stored in `kubectl.kubernetes.io/last-applied-configuration` annotation
2. **Current live state** — what's currently in the cluster
3. **New desired state** — your local YAML file

```
Local file (desired)  ──┐
                        ├──▶  3-way strategic merge patch
Last-applied (stored) ──┘         │
                                  ▼
                        Only changed fields are sent
```

This is why `apply` can coexist with manual edits — it only patches fields you manage, leaving fields set by other tools or controllers untouched.

### Step 2: Authentication

```
┌──────────┐                    ┌───────────────┐
│  kubectl │───── HTTPS ───────▶│   API Server  │
└──────────┘                    └───────────────┘
                                       │
                          ┌────────────┼────────────┐
                          ▼            ▼            ▼
                    x509 Certs    Bearer Token   OIDC/Webhook
                    (client cert)  (ServiceAccount  (AWS IAM,
                                    token, static)   Azure AD)
```

**The API server verifies identity using one of:**

| Method | How It Works | Common Use |
|--------|-------------|------------|
| x509 client certificate | Certificate CN = username, O = group | kubeadm clusters |
| Bearer token | Token in Authorization header | ServiceAccounts, static tokens |
| OIDC | JWT from identity provider | EKS (aws-iam-authenticator), GKE, AKS |
| Webhook token review | External service validates token | Custom auth integrations |

```bash
# EKS example — kubectl calls aws-iam-authenticator to get a token:
# 1. kubectl reads exec config from kubeconfig
# 2. Runs: aws eks get-token --cluster-name my-cluster
# 3. Gets a presigned STS URL as a bearer token
# 4. API server calls the aws-iam-authenticator webhook to validate
# 5. Webhook calls STS GetCallerIdentity to verify IAM identity
```

### Step 3: Authorization (RBAC)

```
┌───────────────┐
│   API Server  │
│               │
│  "Can user X  │──▶ Check RBAC rules:
│   create/patch│    - ClusterRoleBindings
│   deployments │    - RoleBindings
│   in ns Y?"   │    - ClusterRoles
│               │    - Roles
└───────────────┘
        │
        ▼
   Allow / Deny (403)
```

The API server evaluates:

```yaml
# Example: Does the user have permission to create Deployments?
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch"]  # "patch" needed for apply
```

```bash
# Check if you have permission:
kubectl auth can-i create deployments --namespace default
kubectl auth can-i patch deployments --namespace default

# Check as a specific user/service account:
kubectl auth can-i create deployments --as system:serviceaccount:default:my-sa
```

### Step 4: Admission Controllers

```
                     Request
                        │
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
   ┌─────────────┐ ┌──────────┐ ┌──────────────┐
   │  Mutating   │ │Validating│ │  Webhooks    │
   │  Admission  │ │Admission │ │  (external)  │
   └─────────────┘ └──────────┘ └──────────────┘
          │             │              │
          ▼             ▼              ▼
   Modify request  Accept/Reject  Accept/Reject
   (inject sidecars,  (enforce      (OPA/Gatekeeper,
    set defaults)      policies)     Kyverno)
```

Admission controllers run in order:

**Phase 1 — Mutating Admission (can modify the object):**

| Controller | What It Does |
|-----------|-------------|
| `NamespaceLifecycle` | Rejects requests in terminating namespaces |
| `LimitRanger` | Applies default resource requests/limits |
| `ServiceAccount` | Mounts default SA token if none specified |
| `DefaultStorageClass` | Assigns default StorageClass to PVCs |
| `MutatingAdmissionWebhook` | Calls external webhooks (e.g., Istio sidecar injection) |

**Phase 2 — Validating Admission (read-only check, accept or reject):**

| Controller | What It Does |
|-----------|-------------|
| `ResourceQuota` | Rejects if namespace quota would be exceeded |
| `PodSecurity` | Enforces pod security standards |
| `ValidatingAdmissionWebhook` | Calls external validators (e.g., OPA/Gatekeeper) |

```bash
# See which admission controllers are enabled:
kubectl -n kube-system get pod kube-apiserver-* -o yaml | grep enable-admission-plugins

# Common external webhooks:
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations
```

**Example: What gets mutated during a Deployment apply:**

```
Your YAML                          After Admission
─────────────                      ───────────────
spec:                              spec:
  containers:                        containers:
  - name: app                        - name: app
    image: nginx                       image: nginx
                                       resources:        ← LimitRanger added defaults
                                         requests:
                                           cpu: 100m
                                           memory: 128Mi
                                     - name: istio-proxy  ← Istio webhook injected sidecar
                                       image: istio/proxyv2
  serviceAccountName: (empty)        serviceAccountName: default  ← ServiceAccount controller
```

### Step 5: Persist to etcd

```
┌───────────────┐                    ┌───────────┐
│   API Server  │──── gRPC ─────────▶│   etcd    │
│               │                    │           │
│  Serialize to │                    │  Key:     │
│  protobuf     │                    │  /registry/deployments/default/my-app
│               │                    │           │
│  Store under  │                    │  Value:   │
│  /registry/   │                    │  (protobuf-encoded object)
└───────────────┘                    └───────────┘
```

The object is stored with:
- `metadata.uid` — unique identifier generated
- `metadata.resourceVersion` — etcd revision number (used for optimistic concurrency)
- `metadata.creationTimestamp` — set on first create
- `metadata.generation` — incremented when spec changes
- `status` — initialized empty (controllers fill this in)

```bash
# See the stored metadata:
kubectl get deployment my-app -o jsonpath='{.metadata.uid}'
kubectl get deployment my-app -o jsonpath='{.metadata.resourceVersion}'

# The last-applied-configuration annotation (used by next apply):
kubectl get deployment my-app -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}' | jq .
```

### Step 6: Controllers React (Watch Loop)

```
┌───────────┐    Watch stream (HTTP/2 long-poll)
│   etcd    │◀──────────────────────────────────────┐
└───────────┘                                       │
      │                                             │
      │  Event: Deployment "my-app" created         │
      ▼                                             │
┌───────────────┐     ┌──────────────────────┐      │
│   API Server  │────▶│Deployment Controller │──────┘
└───────────────┘     │                      │
                      │ Creates ReplicaSet   │
                      └──────────────────────┘
                               │
                               ▼
                      ┌──────────────────────┐
                      │ReplicaSet Controller │
                      │                      │
                      │Creates Pod objects   │
                      └──────────────────────┘
                               │
                               ▼
                      ┌──────────────────────┐
                      │     Scheduler        │
                      │                      │
                      │ Assigns pod to node  │
                      │ (sets spec.nodeName) │
                      └──────────────────────┘
                               │
                               ▼
                      ┌──────────────────────┐
                      │      Kubelet         │
                      │  (on assigned node)  │
                      │                      │
                      │ Pulls image          │
                      │ Creates container    │
                      │ Starts container     │
                      │ Reports status back  │
                      └──────────────────────┘
```

**Controller cascade for a Deployment:**

| Step | Controller | Action | Object Created/Modified |
|------|-----------|--------|------------------------|
| 1 | Deployment controller | Sees new Deployment, creates ReplicaSet | ReplicaSet |
| 2 | ReplicaSet controller | Sees new RS with replicas=3, creates 3 Pods | Pod (x3) |
| 3 | Scheduler | Sees unscheduled Pods, assigns nodes | Pod.spec.nodeName set |
| 4 | Kubelet (per node) | Sees Pod assigned to its node, runs containers | Pod.status updated |
| 5 | Kubelet | Runs readiness/liveness probes | Pod conditions updated |
| 6 | Endpoint controller | Sees ready Pod matching Service selector | Endpoints/EndpointSlice updated |

Each controller follows the same pattern:

```
while true:
    desired_state = read object spec from API server
    current_state = observe actual state
    if current_state != desired_state:
        take action to reconcile
        update status
```

### Step 7: Kubelet Runs the Pod

```
┌──────────────────────────────────────────────────────┐
│  Kubelet (on the assigned node)                      │
│                                                      │
│  1. Pull container image (via CRI → containerd)      │
│  2. Create pod sandbox (pause container + network)   │
│  3. Call CNI plugin to set up pod networking         │
│  4. Create and start containers                      │
│  5. Mount volumes (CSI driver if needed)             │
│  6. Run postStart lifecycle hooks                    │
│  7. Start liveness/readiness/startup probes          │
│  8. Report pod status back to API server             │
└──────────────────────────────────────────────────────┘
```

```
Kubelet
   │
   ├──▶ CRI (Container Runtime Interface)
   │       └──▶ containerd → runc → container process
   │
   ├──▶ CNI (Container Network Interface)
   │       └──▶ aws-vpc-cni / calico / flannel → pod IP assigned
   │
   └──▶ CSI (Container Storage Interface)
           └──▶ ebs-csi-driver / efs-csi-driver → volume mounted
```

### Step 8: Response Returned to kubectl

```
┌───────────────┐                    ┌──────────┐
│   API Server  │───── HTTP 200 ────▶│  kubectl │
│               │    (or 201 Created)│          │
│  Returns the  │                    │  Prints: │
│  full object  │                    │  deployment.apps/my-app configured
│  as stored    │                    │  (or: created)
└───────────────┘                    └──────────┘
```

By the time kubectl prints "configured", only steps 1–5 are complete. The controllers (step 6) and kubelet (step 7) work asynchronously — your pod isn't running yet.

```bash
# kubectl returns immediately, but the pod takes time:
kubectl apply -f deployment.yaml
# deployment.apps/my-app configured    ← API server accepted it

# Wait for the rollout to actually complete:
kubectl rollout status deployment/my-app
# Waiting for deployment "my-app" rollout to finish: 0 of 3 updated replicas are available...
# deployment "my-app" successfully rolled out
```

## Complete Timeline

```
Time ──────────────────────────────────────────────────────────────────▶

│ kubectl        │ API Server       │ etcd    │ Controllers    │ Kubelet
│                │                  │         │                │
│ Parse YAML     │                  │         │                │
│ Load kubeconfig│                  │         │                │
│ Compute patch  │                  │         │                │
│ Send request ──┼──▶               │         │                │
│                │ Authenticate     │         │                │
│                │ Authorize (RBAC) │         │                │
│                │ Mutate (admit)   │         │                │
│                │ Validate (admit) │         │                │
│                │ Persist ─────────┼──▶ Write│                │
│                │                  │         │                │
│ ◀──────────────┼── 200 OK         │         │                │
│ Print result   │                  │         │                │
│ (done)         │                  │         │                │
│                │                  │  Watch ─┼──▶ Deployment  │
│                │                  │  event  │    controller  │
│                │                  │         │    creates RS  │
│                │                  │         │                │
│                │                  │  Watch ─┼──▶ RS ctrl     │
│                │                  │  event  │    creates Pods│
│                │                  │         │                │
│                │                  │  Watch ─┼──▶ Scheduler   │
│                │                  │  event  │    binds Pod   │
│                │                  │         │    to node     │
│                │                  │         │                │
│                │                  │         │         Watch ─┼──▶ Pull image
│                │                  │         │         event  │    Start container
│                │                  │         │                │    Run probes
│                │                  │         │                │    Report Ready
```

## apply vs create vs replace

| Command | HTTP Verb | Behavior | Use Case |
|---------|-----------|----------|----------|
| `kubectl apply` | PATCH (strategic merge) | Create if missing, patch if exists. Tracks last-applied config. | Declarative management (GitOps, CI/CD) |
| `kubectl create` | POST | Create only. Fails if object exists. | One-time imperative creation |
| `kubectl replace` | PUT | Replace entire object. Fails if object doesn't exist. | Full object replacement (all fields) |

```bash
# apply — idempotent, safe to run repeatedly
kubectl apply -f deployment.yaml

# create — fails on second run
kubectl create -f deployment.yaml
# Error: deployments.apps "my-app" already exists

# replace — overwrites everything (dangerous: drops fields you didn't include)
kubectl replace -f deployment.yaml
```

### Why apply Uses PATCH, Not PUT

`PUT` (replace) requires you to send the complete object — if you omit a field, it gets deleted. This breaks when controllers add fields you didn't know about (status, finalizers, managed fields).

`PATCH` (apply) only sends your changes. Fields managed by other actors remain untouched.

## Server-Side Apply (SSA)

Starting with Kubernetes 1.22+, you can use server-side apply, which moves the merge logic from kubectl to the API server:

```bash
kubectl apply -f deployment.yaml --server-side
```

```
┌──────────┐                         ┌───────────────┐
│  kubectl │──── Full object ───────▶│   API Server  │
│          │     (no patch needed)   │               │
│          │                         │  Computes the │
│          │                         │  merge server-│
│          │                         │  side using   │
│          │                         │ field managers│
└──────────┘                         └───────────────┘
```

| Feature | Client-Side Apply | Server-Side Apply |
|---------|:-----------------:|:-----------------:|
| Merge logic runs in | kubectl | API server |
| Conflict detection | Last-applied annotation | Field ownership (managedFields) |
| Multi-actor safety | Weak (annotation-based) | Strong (per-field ownership) |
| Works with CRDs | Yes | Yes (better conflict handling) |
| Annotation used | `last-applied-configuration` | `managedFields` (in metadata) |

```bash
# See field managers:
kubectl get deployment my-app -o jsonpath='{.metadata.managedFields}' | jq .

# Force conflicts (take ownership of fields managed by others):
kubectl apply -f deployment.yaml --server-side --force-conflicts
```

## Dry Run and Diff

```bash
# Client-side dry run — validates YAML locally, no API server contact
kubectl apply -f deployment.yaml --dry-run=client -o yaml

# Server-side dry run — sends to API server, runs admission controllers, but doesn't persist
kubectl apply -f deployment.yaml --dry-run=server -o yaml

# Diff — shows what would change without applying
kubectl diff -f deployment.yaml
```

Server-side dry run is more accurate because it runs through admission controllers (mutations, validations) but doesn't write to etcd.

## Debugging the Pipeline

```bash
# See the full HTTP request/response (verbosity 6-9):
kubectl apply -f deployment.yaml -v=6    # URL + timing
kubectl apply -f deployment.yaml -v=7    # + request headers
kubectl apply -f deployment.yaml -v=8    # + request/response body
kubectl apply -f deployment.yaml -v=9    # + curl equivalent

# Check API server audit logs for the request:
kubectl logs -n kube-system kube-apiserver-<node> | grep my-app

# Watch the controller cascade in real-time:
kubectl get events --watch --field-selector involvedObject.name=my-app

# Trace the full object lifecycle:
kubectl get deployment my-app -o yaml | grep -A5 "managedFields"
```

## Common Failure Points

| Stage | Error | Cause |
|-------|-------|-------|
| Client | `error: no matches for kind "Deployment" in version "apps/v1beta1"` | Wrong apiVersion for your cluster version |
| Auth | `error: You must be logged in to the server (Unauthorized)` | Expired token, wrong kubeconfig context |
| RBAC | `Error from server (Forbidden): deployments.apps is forbidden` | Missing Role/ClusterRole binding |
| Admission | `Error from server: admission webhook "validate.example.com" denied the request` | Policy violation (OPA, Kyverno, PodSecurity) |
| Validation | `spec.containers[0].image: Required value` | Missing required field in spec |
| Quota | `Error from server (Forbidden): exceeded quota` | Namespace ResourceQuota exhausted |
| Conflict | `Operation cannot be fulfilled: the object has been modified` | Another actor changed the object simultaneously |
