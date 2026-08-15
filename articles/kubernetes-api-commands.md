# Kubernetes Control Plane API Commands

Interacting with the Kubernetes API server directly — using `kubectl get --raw`, `curl` with bearer tokens, API discovery, health checks, metrics, and debugging.

## API Server Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/api` | Core API group (v1 resources: pods, services, configmaps) |
| `/apis` | All API groups (apps, batch, networking, etc.) |
| `/healthz` | Basic health check |
| `/livez` | Liveness check (individual components) |
| `/readyz` | Readiness check (can the server handle requests) |
| `/metrics` | Prometheus metrics |
| `/version` | Server version |
| `/openapi/v2` | Full OpenAPI spec |

## kubectl get --raw

Access the API directly without resource type abstraction:

```sh
# API server version
kubectl get --raw /version | jq

# Health checks
kubectl get --raw /healthz
kubectl get --raw /livez
kubectl get --raw /readyz

# Verbose health (shows each component)
kubectl get --raw /livez?verbose
kubectl get --raw /readyz?verbose

# List all API groups
kubectl get --raw /apis | jq '.groups[].name'

# List all resources in core API
kubectl get --raw /api/v1 | jq '.resources[].name'

# List all resources in apps/v1
kubectl get --raw /apis/apps/v1 | jq '.resources[].name'

# Get a specific resource directly
kubectl get --raw /api/v1/namespaces/default/pods | jq '.items[].metadata.name'

# Get a specific pod
kubectl get --raw /api/v1/namespaces/default/pods/my-pod | jq

# Get node metrics (from metrics-server)
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes | jq

# Get pod metrics
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/default/pods | jq
```

## API Server Metrics

```sh
# Full metrics dump (Prometheus format)
kubectl get --raw /metrics | head -100

# API request latency
kubectl get --raw /metrics | grep apiserver_request_duration_seconds

# API request count by verb and resource
kubectl get --raw /metrics | grep apiserver_request_total

# Current inflight requests
kubectl get --raw /metrics | grep apiserver_current_inflight_requests

# etcd request latency
kubectl get --raw /metrics | grep etcd_request_duration_seconds

# etcd database size
kubectl get --raw /metrics | grep etcd_db_total_size

# Watch cache size
kubectl get --raw /metrics | grep apiserver_watch_events_sizes

# Storage object count
kubectl get --raw /metrics | grep apiserver_storage_objects

# Admission webhook latency
kubectl get --raw /metrics | grep apiserver_admission_webhook_admission_duration_seconds

# Client certificate expiration
kubectl get --raw /metrics | grep apiserver_client_certificate_expiration_seconds
```

## Accessing the API with curl

### Get the API Server URL and Token

```sh
# Get API server endpoint
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')

# Get a service account token (for automation)
TOKEN=$(kubectl create token default --duration=1h)

# Or get an existing secret-based token (older clusters)
TOKEN=$(kubectl get secret $(kubectl get sa default -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' | base64 -d)
```

### Make API Calls with curl

```sh
# List namespaces
curl -sk -H "Authorization: Bearer $TOKEN" $APISERVER/api/v1/namespaces | jq '.items[].metadata.name'

# List pods in a namespace
curl -sk -H "Authorization: Bearer $TOKEN" $APISERVER/api/v1/namespaces/default/pods | jq '.items[].metadata.name'

# Get a specific pod
curl -sk -H "Authorization: Bearer $TOKEN" $APISERVER/api/v1/namespaces/default/pods/my-pod | jq

# Create a resource (POST)
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  $APISERVER/api/v1/namespaces/default/configmaps \
  -d '{"apiVersion":"v1","kind":"ConfigMap","metadata":{"name":"test-cm"},"data":{"key":"value"}}'

# Delete a resource
curl -sk -X DELETE -H "Authorization: Bearer $TOKEN" \
  $APISERVER/api/v1/namespaces/default/configmaps/test-cm

# Patch a resource (strategic merge)
curl -sk -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/strategic-merge-patch+json" \
  $APISERVER/api/v1/namespaces/default/configmaps/test-cm \
  -d '{"data":{"newkey":"newvalue"}}'

# Watch for changes (streaming)
curl -sk -H "Authorization: Bearer $TOKEN" "$APISERVER/api/v1/namespaces/default/pods?watch=true"
```

### Using kubectl proxy (Simplest for Local Testing)

```sh
# Start a local proxy (no auth needed for requests)
kubectl proxy --port=8001 &

# Then use curl without tokens
curl http://localhost:8001/api/v1/namespaces
curl http://localhost:8001/apis/apps/v1/deployments
curl http://localhost:8001/healthz
curl http://localhost:8001/metrics

# Stop the proxy
kill %1
```

## API Discovery

### List All API Groups

```sh
kubectl api-versions

# Output:
# admissionregistration.k8s.io/v1
# apps/v1
# autoscaling/v1
# autoscaling/v2
# batch/v1
# ...
```

### List All Resource Types

```sh
kubectl api-resources

# With details
kubectl api-resources -o wide

# Only namespaced resources
kubectl api-resources --namespaced=true

# Only cluster-scoped resources
kubectl api-resources --namespaced=false

# Filter by API group
kubectl api-resources --api-group=apps
kubectl api-resources --api-group=batch

# Filter by verb (what you can do)
kubectl api-resources --verbs=list,get
kubectl api-resources --verbs=create,delete
```

### Explain Resources

```sh
# Show fields for a resource
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.resources

# Show all fields recursively
kubectl explain pod --recursive

# Specific API version
kubectl explain deployment --api-version=apps/v1
```

## Health and Status Endpoints

### Basic Health

```sh
# Simple up/down
kubectl get --raw /healthz
# Returns: ok

# Verbose (shows each component)
kubectl get --raw /healthz?verbose

# Output:
# [+]ping ok
# [+]log ok
# [+]etcd ok
# [+]poststarthook/start-kube-apiserver-admission-initializer ok
# [+]poststarthook/generic-apiserver-start-informers ok
# [+]poststarthook/start-apiextensions-informers ok
# healthz check passed
```

### Liveness vs Readiness

```sh
# Liveness — is the server alive?
kubectl get --raw /livez?verbose

# Readiness — can the server accept requests?
kubectl get --raw /readyz?verbose

# Check specific components
kubectl get --raw /livez/etcd
kubectl get --raw /readyz/shutdown
kubectl get --raw /readyz/informer-sync
```

### Component Status (Deprecated but still works)

```sh
kubectl get componentstatuses
# OR
kubectl get cs

# Shows: scheduler, controller-manager, etcd health
```

## API Request Flow

```
kubectl get pods
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│  1. Authentication (who are you?)                            │
│     - Client certificate, bearer token, or OIDC              │
├──────────────────────────────────────────────────────────────┤
│  2. Authorization (can you do this?)                         │
│     - RBAC check: does this user have "get pods" permission? │
├──────────────────────────────────────────────────────────────┤
│  3. Admission Control (should we allow this?)                │
│     - Mutating webhooks (inject sidecars, defaults)          │
│     - Validating webhooks (policy enforcement)               │
├──────────────────────────────────────────────────────────────┤
│  4. etcd (persist or retrieve)                               │
│     - Read: fetch from etcd and return                       │
│     - Write: validate, persist to etcd, return               │
└──────────────────────────────────────────────────────────────┘
```

## Debugging API Server Issues

### Check API Server Response Time

```sh
# Measure latency of a simple request
time kubectl get pods > /dev/null

# Verbose mode shows the actual HTTP request/response
kubectl get pods -v=8 2>&1 | grep -E "GET|Response"

# Very verbose (shows headers, timing)
kubectl get pods -v=9 2>&1 | head -30
```

### Verbosity Levels

| Level | What It Shows |
|:-----:|---------------|
| `-v=0` | Only errors |
| `-v=4` | Debug (request URL) |
| `-v=6` | Request body |
| `-v=7` | Response headers |
| `-v=8` | Response body |
| `-v=9` | Full curl equivalent |

```sh
# See the full HTTP request as curl command
kubectl get pods -v=9 2>&1 | grep "curl"
```

### Check for Throttling

```sh
# Are requests being throttled?
kubectl get --raw /metrics | grep apiserver_flowcontrol_request_concurrency_limit

# Check priority levels
kubectl get --raw /metrics | grep apiserver_flowcontrol_current_inqueue_requests

# View APF (API Priority and Fairness) config
kubectl get flowschema
kubectl get prioritylevelconfiguration
```

### Check Admission Webhooks

```sh
# List all validating webhooks
kubectl get validatingwebhookconfigurations

# List all mutating webhooks
kubectl get mutatingwebhookconfigurations

# Describe a webhook (check timeoutSeconds, failurePolicy)
kubectl describe validatingwebhookconfiguration <name>

# Check if webhooks are slowing things down
kubectl get --raw /metrics | grep apiserver_admission_webhook_admission_duration_seconds
```

### Check API Server Logs (EKS)

```sh
# Enable API server logging
aws eks update-cluster-config --name <cluster> \
  --logging '{"clusterLogging":[{"types":["api"],"enabled":true}]}'

# Query logs in CloudWatch
aws logs filter-log-events \
  --log-group-name /aws/eks/<cluster>/cluster \
  --log-stream-name-prefix kube-apiserver \
  --filter-pattern "ERROR" \
  --start-time $(date -u -d '1 hour ago' '+%s000')
```

## Service Account Tokens

### Create a Token (Kubernetes 1.24+)

```sh
# Short-lived token (recommended)
kubectl create token my-service-account --duration=1h

# Token for a specific audience
kubectl create token my-service-account --audience=https://my-api.example.com

# Token bound to a specific resource
kubectl create token my-service-account --bound-object-ref=pod/my-pod
```

### Verify a Token

```sh
# Decode a JWT token (without verifying signature)
TOKEN=$(kubectl create token default)
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | jq

# TokenReview API (server-side verification)
kubectl create -f - <<EOF
apiVersion: authentication.k8s.io/v1
kind: TokenReview
spec:
  token: "$TOKEN"
EOF
```

## RBAC: Check Permissions

```sh
# Can I do X?
kubectl auth can-i create pods
kubectl auth can-i delete nodes
kubectl auth can-i '*' '*'  # Am I cluster-admin?

# Can a specific user do X?
kubectl auth can-i get pods --as=developer
kubectl auth can-i create deployments --as=system:serviceaccount:default:my-sa

# Can a group do X?
kubectl auth can-i list secrets --as-group=developers

# List all permissions for current user
kubectl auth can-i --list

# List permissions for a service account
kubectl auth can-i --list --as=system:serviceaccount:kube-system:aws-node

# Check who can do X (requires RBAC lookup)
kubectl who-can create pods  # requires kubectl-who-can plugin
```

## Common API Paths

| Resource | API Path |
|----------|----------|
| Pods | `/api/v1/namespaces/{ns}/pods` |
| Services | `/api/v1/namespaces/{ns}/services` |
| ConfigMaps | `/api/v1/namespaces/{ns}/configmaps` |
| Secrets | `/api/v1/namespaces/{ns}/secrets` |
| Nodes | `/api/v1/nodes` |
| Namespaces | `/api/v1/namespaces` |
| Deployments | `/apis/apps/v1/namespaces/{ns}/deployments` |
| StatefulSets | `/apis/apps/v1/namespaces/{ns}/statefulsets` |
| DaemonSets | `/apis/apps/v1/namespaces/{ns}/daemonsets` |
| Jobs | `/apis/batch/v1/namespaces/{ns}/jobs` |
| CronJobs | `/apis/batch/v1/namespaces/{ns}/cronjobs` |
| Ingress | `/apis/networking.k8s.io/v1/namespaces/{ns}/ingresses` |
| NetworkPolicy | `/apis/networking.k8s.io/v1/namespaces/{ns}/networkpolicies` |
| PV | `/api/v1/persistentvolumes` |
| PVC | `/api/v1/namespaces/{ns}/persistentvolumeclaims` |
| StorageClass | `/apis/storage.k8s.io/v1/storageclasses` |
| CRDs | `/apis/apiextensions.k8s.io/v1/customresourcedefinitions` |

## Gotchas

- **`/healthz` is deprecated**: Use `/livez` and `/readyz` instead. `/healthz` still works but may not reflect all components.
- **Metrics endpoint requires auth**: `kubectl get --raw /metrics` works because kubectl handles auth. Raw `curl` needs a token.
- **Watch connections consume resources**: Every `watch` is a long-lived connection. Too many watches (from controllers, operators) can overwhelm the API server.
- **Token expiration**: `kubectl create token` tokens expire (default 1h). Plan for rotation in automation.
- **Proxy doesn't forward websockets**: `kubectl proxy` doesn't support `exec` or `attach` (which use websockets). Use `kubectl exec` directly.
- **-v=9 exposes secrets**: Verbose output at level 8+ shows full response bodies including Secrets. Don't log this in production.
- **API Priority and Fairness (APF)**: Since 1.20, the API server throttles requests using APF. If your custom controller is in a low-priority flow schema, it may get 429 responses.
- **EKS doesn't expose all metrics**: Some API server metrics are unavailable on EKS because you don't control the API server process flags.


## Accessing the API from Inside a Pod

Every pod automatically has service account credentials mounted. Use them to call the API server without external auth setup:

```sh
APISERVER=https://kubernetes.default.svc
SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount
TOKEN=$(cat ${SERVICEACCOUNT}/token)
CACERT=${SERVICEACCOUNT}/ca.crt

# List pods in the pod's namespace
curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" \
  -X GET ${APISERVER}/api/v1/namespaces/default/pods/

# Get cluster info
curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" \
  -X GET ${APISERVER}/api/v1/

# Using wget (if curl isn't available)
wget -qO- --ca-certificate ${CACERT} \
  --header "Authorization: Bearer ${TOKEN}" \
  ${APISERVER}/api/v1/namespaces/default/pods/
```

Key paths inside a pod:
- Token: `/var/run/secrets/kubernetes.io/serviceaccount/token`
- CA cert: `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`
- Namespace: `/var/run/secrets/kubernetes.io/serviceaccount/namespace`
- API server: `https://kubernetes.default.svc` (or use env vars `$KUBERNETES_SERVICE_HOST:$KUBERNETES_PORT_443_TCP_PORT`)

## Finding the API Server URL

```sh
# List all clusters and their server URLs
kubectl config view -o jsonpath='{"Cluster\tServer\n"}{range .clusters[*]}{.name}{"\t"}{.cluster.server}{"\n"}{end}'

# Get the current cluster's API server
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'

# From EKS
aws eks describe-cluster --name <cluster> --query "cluster.endpoint" --output text
```

## Using --cacert for Self-Signed Certificates

When calling the API server directly with curl (not via proxy), you need the CA certificate:

```sh
# Extract CA cert from kubeconfig
kubectl config view --minify --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d > /tmp/ca.crt

# Use it with curl
curl -s --cacert /tmp/ca.crt -H "Authorization: Bearer $TOKEN" \
  https://<apiserver>:6443/api/v1/namespaces

# Or skip verification (not for production)
curl -sk -H "Authorization: Bearer $TOKEN" \
  https://<apiserver>:6443/api/v1/namespaces
```

## Kubelet API Endpoints

The kubelet exposes its own API on each node (port 10250 secure, 10248 healthz):

```sh
# Health check (unauthenticated, localhost only)
curl -s http://localhost:10248/healthz

# Pods running on this node (requires auth)
curl -sk https://localhost:10250/pods/

# Metrics
curl -sk https://localhost:10250/metrics

# cAdvisor metrics (container-level resource usage)
curl -sk https://localhost:10250/metrics/cadvisor

# Resource metrics (CPU/memory per pod)
curl -sk https://localhost:10250/metrics/resource

# Node stats summary
curl -sk https://localhost:10250/stats/summary
```

> On EKS, direct kubelet access requires a client certificate or token with `nodes/proxy` RBAC permission. The healthz port (10248) is accessible without auth from localhost only.

## Notes

- The API server typically runs on port **6443** (or 443 behind a load balancer)
- Authentication requires either service account tokens, client certificates, or OIDC tokens
- Use `kubectl proxy` to avoid manually handling authentication during development
- From within pods, always use `https://kubernetes.default.svc` as the API server address
- The `--cacert` flag is needed when using self-signed certificates (skip with `-k` for testing only)
- HTTP success codes: 2xx (200-299)
- Writes return 201 (Created), updates return 200 (OK), deletes return 200 or 202 (Accepted)


## Query Parameters for Raw API Calls

```sh
# Filter by label selector (URL-encode the = as %3D)
kubectl get --raw "/api/v1/pods?labelSelector=app%3Dnginx"

# Filter by field selector
kubectl get --raw "/api/v1/pods?fieldSelector=spec.nodeName%3Dworker-1"

# Limit results
kubectl get --raw "/api/v1/pods?limit=10"

# Watch for changes (streaming)
kubectl get --raw "/api/v1/pods?watch=true"

# Combine parameters
kubectl get --raw "/api/v1/namespaces/default/pods?labelSelector=app%3Dweb&limit=5"
```

### URL Encoding Reference

| Character | Encoded | Example |
|:---------:|:-------:|---------|
| `=` | `%3D` | `app%3Dnginx` |
| `:` | `%3A` | `port%3A8080` |
| `/` | `%2F` | — |
| `?` | `%3F` | — |
| `&` | `%26` | — |
| ` ` (space) | `%20` | `my%20label` |
| `,` | `%2C` | `app%3Dweb%2Cenv%3Dprod` |

## Additional Raw API Paths

```sh
# Pod logs (raw)
kubectl get --raw /api/v1/namespaces/<namespace>/pods/<pod-name>/log

# Services
kubectl get --raw /api/v1/namespaces/<namespace>/services

# Endpoints
kubectl get --raw /api/v1/namespaces/<namespace>/endpoints

# ConfigMaps
kubectl get --raw /api/v1/namespaces/<namespace>/configmaps

# Secrets
kubectl get --raw /api/v1/namespaces/<namespace>/secrets

# Custom Resource Definitions
kubectl get --raw /apis/apiextensions.k8s.io/v1/customresourcedefinitions

# Events in a namespace
kubectl get --raw /api/v1/namespaces/<namespace>/events

# RBAC: Roles
kubectl get --raw /apis/rbac.authorization.k8s.io/v1/namespaces/<namespace>/roles

# RBAC: ClusterRoles
kubectl get --raw /apis/rbac.authorization.k8s.io/v1/clusterroles

# Service Accounts
kubectl get --raw /api/v1/namespaces/<namespace>/serviceaccounts
```

## Advanced Raw API Recipes

### Get Pod Resource Usage (jq)

```sh
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/default/pods | \
  jq '.items[] | {name: .metadata.name, cpu: .containers[0].usage.cpu, memory: .containers[0].usage.memory}'
```

### Monitor Pod Status Changes (Watch + jq)

```sh
kubectl get --raw "/api/v1/namespaces/default/pods?watch=true" | \
  jq -r '.object.metadata.name + " " + .object.status.phase'
```

### Get All Resources in a Namespace

```sh
for resource in $(kubectl api-resources --verbs=list --namespaced -o name); do
  echo "--- $resource ---"
  kubectl get --raw "/api/v1/namespaces/default/$resource" 2>/dev/null | \
    jq -r '.items[]?.metadata.name // empty' 2>/dev/null
done
```

### Get Node Capacity

```sh
kubectl get --raw /api/v1/nodes | jq '.items[] | {name: .metadata.name, cpu: .status.capacity.cpu, memory: .status.capacity.memory}'
```

## API Resource Group Paths

| Path | Resources |
|------|-----------|
| `/api/v1` | Core: pods, services, configmaps, secrets, nodes, namespaces, PVs, PVCs |
| `/apis/apps/v1` | Deployments, ReplicaSets, DaemonSets, StatefulSets |
| `/apis/batch/v1` | Jobs, CronJobs |
| `/apis/networking.k8s.io/v1` | NetworkPolicies, Ingresses |
| `/apis/storage.k8s.io/v1` | StorageClasses, VolumeAttachments, CSIDrivers |
| `/apis/autoscaling/v1` | HorizontalPodAutoscalers |
| `/apis/rbac.authorization.k8s.io/v1` | Roles, ClusterRoles, RoleBindings, ClusterRoleBindings |
| `/apis/policy/v1` | PodDisruptionBudgets |
| `/apis/apiextensions.k8s.io/v1` | CustomResourceDefinitions |
| `/apis/metrics.k8s.io/v1beta1` | Node and pod metrics (requires metrics-server) |
