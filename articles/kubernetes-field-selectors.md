# Kubernetes Field Selectors

Field selectors let you filter Kubernetes resources by specific field values using `--field-selector`. Unlike labels (user-defined metadata), field selectors filter on actual resource spec/status fields.

## Syntax

```sh
# Basic usage
kubectl get pods --field-selector status.phase=Running

# Multiple selectors (AND logic)
kubectl get pods --field-selector status.phase=Running,spec.nodeName=worker-1

# Not-equal
kubectl get pods --field-selector status.phase!=Running

# Combined with other flags
kubectl get pods -A --field-selector status.phase=Pending -o wide
```

## Supported Operators

| Operator | Example |
|----------|---------|
| `=` | `status.phase=Running` |
| `==` | `status.phase==Running` (same as `=`) |
| `!=` | `status.phase!=Running` |

> Field selectors only support equality operators. For set-based filtering (In, NotIn, Exists), use label selectors instead.

## Universal Fields (All Resources)

Every resource type supports these fields:

| Field | Example |
|-------|---------|
| `metadata.name` | `--field-selector metadata.name=my-pod` |
| `metadata.namespace` | `--field-selector metadata.namespace=production` |

## Fields by Resource Type

### Pod

| Field | Description | Example |
|-------|-------------|---------|
| `spec.nodeName` | Node the pod is running on | `spec.nodeName=worker-1` |
| `spec.restartPolicy` | Restart policy | `spec.restartPolicy=Always` |
| `spec.schedulerName` | Scheduler used | `spec.schedulerName=default-scheduler` |
| `spec.serviceAccountName` | Service account | `spec.serviceAccountName=default` |
| `status.phase` | Pod phase | `status.phase=Running` |
| `status.podIP` | Pod IP address | `status.podIP=10.0.1.42` |
| `status.nominatedNodeName` | Nominated node (preemption) | `status.nominatedNodeName=worker-2` |

### Node

| Field | Description | Example |
|-------|-------------|---------|
| `spec.unschedulable` | Whether node is cordoned | `spec.unschedulable=true` |

### Event

| Field | Description | Example |
|-------|-------------|---------|
| `involvedObject.kind` | Resource type | `involvedObject.kind=Pod` |
| `involvedObject.namespace` | Resource namespace | `involvedObject.namespace=default` |
| `involvedObject.name` | Resource name | `involvedObject.name=my-pod` |
| `involvedObject.uid` | Resource UID | `involvedObject.uid=abc-123` |
| `involvedObject.apiVersion` | API version | `involvedObject.apiVersion=v1` |
| `involvedObject.resourceVersion` | Resource version | — |
| `involvedObject.fieldPath` | Field path | `involvedObject.fieldPath=spec.containers{app}` |
| `reason` | Event reason | `reason=Killing` |
| `source` | Component that generated it | `source=kubelet` |
| `type` | Normal or Warning | `type=Warning` |
| `reportingComponent` | Reporting component | — |

### Namespace

| Field | Description | Example |
|-------|-------------|---------|
| `status.phase` | Active or Terminating | `status.phase=Active` |

### Job

| Field | Description | Example |
|-------|-------------|---------|
| `status.successful` | Number of successful completions | `status.successful=1` |

### CronJob

| Field | Description | Example |
|-------|-------------|---------|
| `status.successful` | Number of successful completions | — |

### CertificateSigningRequest

| Field | Description | Example |
|-------|-------------|---------|
| `spec.signerName` | Signer name | `spec.signerName=kubernetes.io/kube-apiserver-client` |

### Secret

| Field | Description | Example |
|-------|-------------|---------|
| `type` | Secret type | `type=kubernetes.io/tls` |

### StatefulSet

| Field | Description | Example |
|-------|-------------|---------|
| `status.successful` | Number of successful completions | — |

### ReplicationController

| Field | Description | Example |
|-------|-------------|---------|
| `status.replicas` | Current replica count | `status.replicas=3` |

## Practical Examples

### Pods

```sh
# Pods on a specific node
kubectl get pods -A --field-selector spec.nodeName=ip-10-0-1-42.ec2.internal

# Pending pods (not scheduled yet)
kubectl get pods -A --field-selector status.phase=Pending

# Non-running pods (Failed, Pending, Unknown)
kubectl get pods -A --field-selector status.phase!=Running

# Running pods only
kubectl get pods -A --field-selector status.phase=Running

# Pods using a specific service account
kubectl get pods -A --field-selector spec.serviceAccountName=my-sa

# Pods with a specific IP
kubectl get pods -A --field-selector status.podIP=10.0.1.42

# Pods scheduled by a specific scheduler
kubectl get pods -A --field-selector spec.schedulerName=my-custom-scheduler

# Combine: running pods on a specific node
kubectl get pods -A --field-selector status.phase=Running,spec.nodeName=worker-1
```

### Nodes

```sh
# Cordoned nodes
kubectl get nodes --field-selector spec.unschedulable=true

# Schedulable nodes
kubectl get nodes --field-selector spec.unschedulable!=true
```

### Events

```sh
# Events for pods only
kubectl get events -A --field-selector involvedObject.kind=Pod

# Events for a specific pod
kubectl get events --field-selector involvedObject.name=my-pod

# Warning events only
kubectl get events -A --field-selector type=Warning

# Normal events only
kubectl get events -A --field-selector type=Normal

# Events NOT about pods
kubectl get events -A --field-selector involvedObject.kind!=Pod

# Events for nodes
kubectl get events --field-selector involvedObject.kind=Node

# Events for a specific node
kubectl get events --field-selector involvedObject.kind=Node,involvedObject.name=worker-1

# Events with a specific reason
kubectl get events -A --field-selector reason=Killing
kubectl get events -A --field-selector reason=FailedScheduling
kubectl get events -A --field-selector reason=Pulled

# Combine: warning events for pods
kubectl get events -A --field-selector involvedObject.kind=Pod,type=Warning
```

### Namespaces

```sh
# Active namespaces
kubectl get ns --field-selector status.phase=Active

# Terminating namespaces (stuck deletions)
kubectl get ns --field-selector status.phase=Terminating
```

### Secrets

```sh
# TLS secrets only
kubectl get secrets -A --field-selector type=kubernetes.io/tls

# Docker config secrets
kubectl get secrets -A --field-selector type=kubernetes.io/dockerconfigjson

# Opaque secrets
kubectl get secrets -A --field-selector type=Opaque

# Service account token secrets
kubectl get secrets -A --field-selector type=kubernetes.io/service-account-token
```

### Jobs

```sh
# Successful jobs
kubectl get jobs -A --field-selector status.successful=1
```

## Combining with Other Flags

```sh
# Field selector + label selector
kubectl get pods -l app=nginx --field-selector status.phase=Running

# Field selector + output format
kubectl get pods --field-selector status.phase=Pending -o wide

# Field selector + sort
kubectl get pods --field-selector spec.nodeName=worker-1 --sort-by=.metadata.creationTimestamp

# Field selector + watch
kubectl get pods --field-selector status.phase=Pending -w

# Field selector + count
kubectl get pods -A --field-selector status.phase=Pending --no-headers | wc -l
```

## What Doesn't Work

Field selectors are limited. Many fields you'd expect to work are NOT supported:

```sh
# THESE DON'T WORK:
kubectl get pods --field-selector spec.containers[0].image=nginx        # ✗ No array access
kubectl get pods --field-selector metadata.labels.app=nginx             # ✗ Use -l instead
kubectl get pods --field-selector spec.resources.requests.cpu=500m      # ✗ Not supported
kubectl get pods --field-selector status.containerStatuses[0].ready=true # ✗ No array access
kubectl get deployments --field-selector spec.replicas=3                # ✗ Not supported
```

For unsupported fields, use:
- **Label selectors** (`-l`) for metadata-based filtering
- **JSONPath** + **jq** for post-filtering
- **Custom columns** with grep

```sh
# Alternative: post-filter with jq for unsupported fields
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.containers[0].image | contains("nginx")) | .metadata.name'
```

## field-selector vs label-selector

| Feature | `--field-selector` | `-l` (label selector) |
|---------|-------------------|----------------------|
| Filters on | Resource spec/status fields | User-defined labels |
| Server-side | Yes | Yes |
| Operators | `=`, `!=` only | `=`, `!=`, `in`, `notin`, `exists` |
| Supported fields | Limited per resource type | Any label key |
| Performance | Efficient (indexed) | Efficient (indexed) |
| Use case | Status/phase filtering | Application/team grouping |

## Finding Available Fields

The supported fields are not well-documented. To discover what works for a resource type, you can:

```sh
# Trial and error — invalid fields give a clear error
kubectl get pods --field-selector spec.invalid=test
# Error: "spec.invalid" is not a known field selector

# Check source code: look for GetAttrs in **/strategy.go in the Kubernetes repo
# https://github.com/kubernetes/kubernetes/search?q=GetAttrs+strategy.go
```

## Gotchas

- **Very limited field set**: Most fields you'd expect are not supported. Only the fields listed above work.
- **No set-based operators**: Can't use `In`, `NotIn`, or `Exists` — only `=` and `!=`.
- **No array indexing**: Can't filter on `spec.containers[0].image` or similar nested arrays.
- **Server-side filtering**: Field selectors are evaluated by the API server, which is efficient but limits what's possible.
- **`status.phase` for pods only has 5 values**: Pending, Running, Succeeded, Failed, Unknown. There's no "CrashLoopBackOff" phase — that's a container state, not a pod phase.
- **`spec.nodeName` is empty for unscheduled pods**: You can't use `spec.nodeName=""` to find unscheduled pods. Use `status.phase=Pending` instead.
- **Events are short-lived**: Events are garbage-collected after ~1 hour by default. Field selectors on events only work for recent events.
