# How Kubernetes Admission Webhooks Fire

The internal flow of mutating and validating admission webhooks — how the API server calls external services, handles failures, manages ordering, and integrates with policy engines.

Note: For a brief overview of admission controllers in the context of `kubectl apply`, see the kubectl apply internals guide. This article covers the webhook mechanics in depth.

## Where Webhooks Fit in the Request Pipeline

```
Request → Auth → AuthZ → Mutating Admission → Object Schema Validation → Validating Admission → etcd
                              │                                                │
                    ┌─────────┼─────────┐                          ┌───────────┼───────────┐
                    ▼         ▼         ▼                          ▼           ▼           ▼
              Built-in    Mutating     Mutating              Built-in     Validating   Validating
              controllers webhooks    webhooks              controllers   webhooks     webhooks
              (LimitRanger,(Istio,    (custom)              (ResourceQuota,(OPA,       (custom)
               SA default) Linkerd)                          PodSecurity)  Kyverno)
```

### Execution Order

1. **Mutating admission** (can modify the object):
   - Built-in mutating controllers run first
   - Then MutatingAdmissionWebhooks run (in alphabetical order by name)
   - Object is re-validated against OpenAPI schema after mutations

2. **Validating admission** (read-only, accept or reject):
   - Built-in validating controllers run
   - Then ValidatingAdmissionWebhooks run (in parallel, since they can't modify)

## The Webhook Call

When the API server invokes a webhook, it sends an `AdmissionReview` request:

```
┌───────────────┐                         ┌──────────────────┐
│   API Server  │──── POST /webhook ─────▶│  Webhook Server  │
│               │     (AdmissionReview)   │  (your service)  │
│               │                         │                  │
│               │◀─── AdmissionReview ────│  Allow/Deny +    │
│               │     (response)          │  optional patches│
└───────────────┘                         └──────────────────┘
```

### Request Payload (AdmissionReview)

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "abc-123",
    "kind": {"group": "", "version": "v1", "kind": "Pod"},
    "resource": {"group": "", "version": "v1", "resource": "pods"},
    "requestKind": {"group": "", "version": "v1", "kind": "Pod"},
    "requestResource": {"group": "", "version": "v1", "resource": "pods"},
    "name": "my-pod",
    "namespace": "default",
    "operation": "CREATE",
    "userInfo": {
      "username": "jane",
      "groups": ["developers", "system:authenticated"]
    },
    "object": { ... the pod spec being created ... },
    "oldObject": null,
    "dryRun": false
  }
}
```

| Field | Description |
|-------|-------------|
| `uid` | Unique ID for this request — must be echoed back in response |
| `operation` | CREATE, UPDATE, DELETE, or CONNECT |
| `object` | The new/desired object (null for DELETE) |
| `oldObject` | The existing object (null for CREATE) |
| `dryRun` | True if `--dry-run=server` — webhook should NOT have side effects |
| `userInfo` | Who made the request (for policy decisions) |

### Response Payload

**Allow:**

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "abc-123",
    "allowed": true
  }
}
```

**Deny:**

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "abc-123",
    "allowed": false,
    "status": {
      "code": 403,
      "message": "Pod must have resource limits defined"
    }
  }
}
```

**Allow with mutations (mutating webhooks only):**

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "abc-123",
    "allowed": true,
    "patchType": "JSONPatch",
    "patch": "W3sib3AiOiAiYWRkIiwgInBhdGgiOiAiL3NwZWMvY29udGFpbmVycy8wL3Jlc291cmNlcy9saW1pdHMvbWVtb3J5IiwgInZhbHVlIjogIjI1Nk1pIn1d"
  }
}
```

The `patch` field is a base64-encoded JSON Patch (RFC 6902):

```json
[{"op": "add", "path": "/spec/containers/0/resources/limits/memory", "value": "256Mi"}]
```

## Webhook Configuration Objects

### MutatingWebhookConfiguration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: istio-sidecar-injector
webhooks:
- name: sidecar-injector.istio.io
  admissionReviewVersions: ["v1"]
  
  # What to intercept
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
    scope: "Namespaced"
  
  # Where to send
  clientConfig:
    service:
      name: istiod
      namespace: istio-system
      path: /inject
      port: 443
    caBundle: <base64-CA-cert>
  
  # Matching rules
  namespaceSelector:
    matchLabels:
      istio-injection: enabled
  objectSelector:
    matchExpressions:
    - key: sidecar.istio.io/inject
      operator: NotIn
      values: ["false"]
  
  # Failure handling
  failurePolicy: Fail          # Fail or Ignore
  timeoutSeconds: 10           # Max 30s
  reinvocationPolicy: IfNeeded # Never or IfNeeded
  sideEffects: None            # None, NoneOnDryRun
  
  # Matching conditions (CEL expressions, K8s 1.28+)
  matchConditions:
  - name: exclude-leases
    expression: "!(request.resource.resource == 'leases')"
```

### ValidatingWebhookConfiguration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: gatekeeper-validating
webhooks:
- name: validation.gatekeeper.sh
  admissionReviewVersions: ["v1"]
  rules:
  - apiGroups: ["*"]
    apiVersions: ["*"]
    operations: ["CREATE", "UPDATE"]
    resources: ["*"]
  clientConfig:
    service:
      name: gatekeeper-webhook-service
      namespace: gatekeeper-system
      path: /v1/admit
  failurePolicy: Ignore
  timeoutSeconds: 5
  sideEffects: None
```

## Matching Rules — What Triggers a Webhook

### rules (Required)

```yaml
rules:
- apiGroups: ["apps"]
  apiVersions: ["v1"]
  operations: ["CREATE", "UPDATE"]
  resources: ["deployments"]
  scope: "Namespaced"    # Namespaced, Cluster, or * (both)
```

### namespaceSelector

Only calls the webhook for objects in namespaces matching this selector:

```yaml
namespaceSelector:
  matchLabels:
    environment: production
  matchExpressions:
  - key: kubernetes.io/metadata.name
    operator: NotIn
    values: ["kube-system", "kube-public"]
```

### objectSelector

Matches based on the object's own labels:

```yaml
objectSelector:
  matchLabels:
    webhook: enabled
```

### matchConditions (CEL — Kubernetes 1.28+)

Fine-grained filtering using CEL expressions:

```yaml
matchConditions:
- name: exclude-system-users
  expression: "!request.userInfo.username.startsWith('system:')"
- name: only-new-pods
  expression: "request.operation == 'CREATE'"
- name: has-no-skip-annotation
  expression: "!has(object.metadata.annotations) || !('skip-validation' in object.metadata.annotations)"
```

All conditions must evaluate to `true` for the webhook to be called.

## Failure Policy

What happens when the webhook is unreachable or returns an error:

| Policy | Behavior | Use Case |
|--------|----------|----------|
| `Fail` (default) | Request is rejected | Security-critical webhooks (don't allow unvalidated objects) |
| `Ignore` | Request proceeds without webhook | Non-critical webhooks (monitoring, optional enrichment) |

```yaml
failurePolicy: Fail    # Reject if webhook is down
failurePolicy: Ignore  # Allow through if webhook is down
```

**Warning**: `failurePolicy: Fail` can break your cluster if the webhook server goes down — all matching requests fail. Always exclude `kube-system` and critical namespaces.

## Timeout

```yaml
timeoutSeconds: 10  # Default: 10s, Max: 30s
```

If the webhook doesn't respond within this timeout:
- `failurePolicy: Fail` → request rejected with timeout error
- `failurePolicy: Ignore` → request allowed through

## Reinvocation Policy (Mutating Only)

When multiple mutating webhooks modify an object, earlier webhooks may need to see later mutations:

```yaml
reinvocationPolicy: IfNeeded  # Re-call if object was modified by another webhook
reinvocationPolicy: Never     # Call only once (default)
```

```
Webhook A modifies object → Webhook B modifies object → Webhook A called AGAIN (IfNeeded)
```

The API server caps reinvocations to prevent infinite loops. After all mutating webhooks stabilize (no more changes), validating webhooks run on the final object.

## sideEffects

Tells the API server whether the webhook has external side effects:

| Value | Meaning | dry-run Behavior |
|-------|---------|------------------|
| `None` | No side effects | Called during dry-run |
| `NoneOnDryRun` | Has side effects, but not during dry-run | Called during dry-run (won't do side effects) |

Webhooks with side effects that would fire during dry-run should use `NoneOnDryRun`.

## Execution Flow — Mutating Webhooks

```
┌───────────────────────────────────────────────────────────────────┐
│  API Server: Mutating Admission Phase                             │
│                                                                   │
│  1. Collect all MutatingWebhookConfigurations                     │
│  2. Sort webhooks alphabetically by name                          │
│  3. For each webhook (sequential):                                │
│     a. Check if rules/selectors match the request                 │
│     b. If match → POST AdmissionReview to webhook service         │
│     c. If response has patches → apply JSONPatch to object        │
│     d. If reinvocationPolicy=IfNeeded and object changed:         │
│        → Mark earlier webhooks for re-call                        │
│  4. After all webhooks complete:                                  │
│     → Validate object against OpenAPI schema                      │
│     → Proceed to validating admission                             │
│                                                                   │
│  If ANY webhook denies → entire request rejected                  │
│  If ANY webhook times out:                                        │
│    failurePolicy=Fail → rejected                                  │
│    failurePolicy=Ignore → skip this webhook, continue             │
└───────────────────────────────────────────────────────────────────┘
```

## Execution Flow — Validating Webhooks

```
┌───────────────────────────────────────────────────────────────────┐
│  API Server: Validating Admission Phase                           │
│                                                                   │
│  1. Collect all ValidatingWebhookConfigurations                   │
│  2. For each matching webhook (PARALLEL):                         │
│     a. POST AdmissionReview to webhook service                    │
│     b. Collect response (allow/deny)                              │
│  3. After all webhooks respond:                                   │
│     → If ALL allow → proceed to etcd                              │
│     → If ANY deny → request rejected (with denial reasons)        │
│                                                                   │
│  Validating webhooks run in PARALLEL because they can't mutate    │
│  (no ordering dependency)                                         │
└───────────────────────────────────────────────────────────────────┘
```

## Common Webhook Patterns

### Sidecar Injection (Istio)

```
Request: CREATE Pod
  │
  ▼
Mutating Webhook (istio-sidecar-injector):
  - Checks namespace label: istio-injection=enabled
  - Checks pod annotation: sidecar.istio.io/inject != "false"
  - Injects istio-proxy container into pod spec
  - Injects init container for iptables setup
  - Returns JSONPatch adding containers + volumes
```

### Policy Enforcement (OPA/Gatekeeper)

```
Request: CREATE Deployment
  │
  ▼
Validating Webhook (gatekeeper):
  - Loads all Constraint objects from its cache
  - Evaluates Rego policies against the object
  - Returns deny if any constraint violated
  - Message: "Container must not run as root"
```

### Default Values (Custom Webhook)

```
Request: CREATE Pod (no resource limits set)
  │
  ▼
Mutating Webhook (resource-defaults):
  - Checks if resources.limits is empty
  - Returns JSONPatch adding default limits:
    memory: 256Mi, cpu: 500m
```

## ValidatingAdmissionPolicy (CEL — Kubernetes 1.30+)

Starting with Kubernetes 1.30, you can define validation policies without a webhook server:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-labels
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: ["apps"]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["deployments"]
  validations:
  - expression: "has(object.metadata.labels) && 'app' in object.metadata.labels"
    message: "Deployment must have an 'app' label"
  - expression: "object.spec.replicas <= 10"
    message: "Cannot have more than 10 replicas"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-labels-binding
spec:
  policyName: require-labels
  validationActions: ["Deny"]
  matchResources:
    namespaceSelector:
      matchLabels:
        environment: production
```

Benefits over webhooks:
- No external server to deploy/maintain
- No network latency
- Evaluated in-process by the API server
- CEL expressions are sandboxed and safe

## Networking — How the API Server Reaches Webhooks

```
┌───────────────┐                    ┌──────────────────┐
│   API Server  │                    │  Webhook Pod     │
│(control plane)│                    │  (in cluster)    │
│               │                    │                  │
│  Resolves:    │                    │  Service:        │
│  service.name │───── ClusterIP ───▶│  webhook-svc     │
│  + namespace  │     (port 443)     │  port: 443       │
│               │                    │  targetPort: 8443│
└───────────────┘                    └──────────────────┘
```

The `clientConfig` can specify either:

```yaml
# In-cluster service (most common)
clientConfig:
  service:
    name: my-webhook
    namespace: webhook-system
    path: /validate
    port: 443
  caBundle: <base64-encoded-CA>

# External URL (for testing or external services)
clientConfig:
  url: "https://webhook.example.com/validate"
  caBundle: <base64-encoded-CA>
```

**TLS is required**. The API server validates the webhook's certificate against `caBundle`.

## Debugging Webhooks

```bash
# List all webhook configurations:
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations

# Describe a webhook (see rules, selectors, failure policy):
kubectl describe mutatingwebhookconfiguration istio-sidecar-injector

# Check if a webhook is blocking requests:
kubectl get events -A --field-selector reason=FailedCreate | grep -i webhook
kubectl get events -A --field-selector reason=Admission

# Test webhook connectivity from API server:
kubectl get --raw /readyz/poststarthook/generic-apiserver-start-informers

# Check webhook server logs:
kubectl logs -n <webhook-namespace> -l app=<webhook-app>

# Check webhook service has endpoints:
kubectl get endpoints -n <webhook-namespace> <webhook-service>

# Temporarily disable a webhook (emergency):
kubectl delete mutatingwebhookconfiguration <name>
# Or patch failurePolicy to Ignore:
kubectl patch mutatingwebhookconfiguration <name> \
  --type='json' -p='[{"op":"replace","path":"/webhooks/0/failurePolicy","value":"Ignore"}]'

# See which webhooks fired for a specific request (audit logs):
# Look for "admission.k8s.io/webhook" annotations in audit events
```

### Common Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| All pod creation fails | Webhook down + `failurePolicy: Fail` | Patch to Ignore or delete webhook config |
| Webhook timeout errors | Service not reachable or slow | Check endpoints, network policies, increase timeout |
| Certificate errors | caBundle doesn't match webhook cert | Update caBundle or rotate certs |
| Infinite loops | Webhook modifies object → triggers another webhook | Use `reinvocationPolicy: Never` or narrow rules |
| kube-system pods failing | Webhook matches system namespace | Add namespaceSelector to exclude kube-system |

## Best Practices

```yaml
# Always exclude system namespaces:
namespaceSelector:
  matchExpressions:
  - key: kubernetes.io/metadata.name
    operator: NotIn
    values: ["kube-system", "kube-public", "kube-node-lease"]

# Use objectSelector to limit scope:
objectSelector:
  matchLabels:
    validate: "true"

# Set reasonable timeout:
timeoutSeconds: 5  # Don't block API for too long

# Use Ignore for non-critical webhooks:
failurePolicy: Ignore

# Declare no side effects:
sideEffects: None

# Support dry-run:
# Check request.dryRun in your webhook logic
```

## Quick Reference

```bash
# List webhooks
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations

# Describe (see rules, selectors, failure policy)
kubectl describe mutatingwebhookconfiguration <name>

# Check webhook endpoints are healthy
kubectl get endpoints -n <ns> <service-name>

# Emergency: disable a blocking webhook
kubectl delete mutatingwebhookconfiguration <name>

# Emergency: switch to Ignore
kubectl patch mutatingwebhookconfiguration <name> \
  --type='json' -p='[{"op":"replace","path":"/webhooks/0/failurePolicy","value":"Ignore"}]'

# Check webhook server logs
kubectl logs -n <ns> -l app=<webhook-app> --tail=50

# Key facts:
# - Mutating webhooks run sequentially (alphabetical by name)
# - Validating webhooks run in parallel
# - failurePolicy: Fail can break your cluster
# - TLS is mandatory (caBundle in config)
# - Always exclude kube-system from webhook rules
# - ValidatingAdmissionPolicy (CEL) replaces simple webhooks (K8s 1.30+)
```
