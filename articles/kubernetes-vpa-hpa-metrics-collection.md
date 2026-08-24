# VPA and HPA Metrics Collection

How HPA and VPA collect metrics for scaling decisions — metrics-server, Prometheus Adapter, custom metrics, and the metrics pipeline architecture.

## Metrics Pipeline Architecture

```
┌─────────────┐     ┌─────────────────┐     ┌──────────────────────────┐
│   kubelet   │────▶│ metrics-server  │────▶│  Metrics API             │
│  (cAdvisor) │     │                 │     │  metrics.k8s.io          │
└─────────────┘     └─────────────────┘     └───────────┬──────────────┘
                                                        │
                                              ┌─────────┼─────────┐
                                              │         │         │
                                              ▼         ▼         ▼
                                           HPA       VPA       kubectl top
```

Both HPA and VPA ultimately get their data from the kubelet's cAdvisor. But they use different APIs and have different collection patterns.

## What Each Autoscaler Needs

| Autoscaler | Metrics Source | API Used | What It Measures |
|-----------|---------------|----------|-----------------|
| **HPA** (resource metrics) | metrics-server | `metrics.k8s.io/v1beta1` | Current CPU/memory usage |
| **HPA** (custom metrics) | Prometheus Adapter | `custom.metrics.k8s.io/v1beta1` | App-level metrics (RPS, queue depth) |
| **HPA** (external metrics) | External adapter | `external.metrics.k8s.io/v1beta1` | Cloud metrics (SQS depth, Pub/Sub) |
| **VPA** | metrics-server | `metrics.k8s.io/v1beta1` | Historical CPU/memory usage over time |

## metrics-server (Foundation for Both)

metrics-server is the core metrics pipeline. Without it, neither HPA nor VPA can function for CPU/memory scaling.

### What It Does

- Scrapes kubelet's `/metrics/resource` endpoint every 15 seconds
- Aggregates CPU and memory usage per pod and node
- Exposes data via the Kubernetes Metrics API (`metrics.k8s.io`)
- Stores only the latest data point (no history)

### Install

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
kubectl top pods -A
```

### How Metrics Flow

```
Container (process)
  │
  ▼
cAdvisor (built into kubelet) — collects container resource usage from cgroups
  │
  ▼
kubelet /metrics/resource endpoint — exposes per-container CPU/memory
  │
  ▼
metrics-server — scrapes all kubelets every 15s, aggregates per-pod
  │
  ▼
Metrics API (metrics.k8s.io) — queryable by HPA, VPA, kubectl top
```

### What metrics-server Reports

```bash
# Pod-level metrics (what HPA and VPA see)
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/default/pods" | jq '.items[] | {name: .metadata.name, cpu: .containers[].usage.cpu, memory: .containers[].usage.memory}'

# Node-level metrics
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq '.items[] | {name: .metadata.name, cpu: .usage.cpu, memory: .usage.memory}'
```

## How HPA Uses Metrics

### Resource Metrics (CPU/Memory)

HPA queries the Metrics API every 15 seconds (configurable via `--horizontal-pod-autoscaler-sync-period`):

```
HPA Controller Loop (every 15s):
  1. Query metrics.k8s.io for current pod CPU/memory usage
  2. Calculate: currentMetric / targetMetric
  3. Compute desired replicas: ceil(currentReplicas × ratio)
  4. Apply tolerance (default 10%) — don't scale for small fluctuations
  5. Scale if outside tolerance
```

```yaml
# HPA using resource metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # Target 70% of requests
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

> **Important:** HPA calculates utilization as a percentage of the pod's **resource requests**, not limits. Without requests, HPA shows `<unknown>`.

### Custom Metrics (Application-Level)

For metrics like requests per second, queue depth, or latency — requires a custom metrics adapter:

```
Application → exports metrics → Prometheus scrapes → Prometheus Adapter → custom.metrics.k8s.io → HPA
```

```bash
# Install Prometheus Adapter
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  -n monitoring \
  --set prometheus.url=http://prometheus-server.monitoring.svc \
  --set prometheus.port=9090
```

```yaml
# HPA using custom metric
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

### External Metrics (Cloud Services)

For scaling based on metrics outside the cluster (SQS queue depth, CloudWatch, etc.):

```yaml
# HPA using external metric (e.g., SQS queue depth via KEDA or custom adapter)
metrics:
- type: External
  external:
    metric:
      name: sqs_messages_visible
      selector:
        matchLabels:
          queue: my-processing-queue
    target:
      type: AverageValue
      averageValue: "5"
```

## How VPA Uses Metrics

VPA's Recommender has a fundamentally different approach — it collects **historical data** over time:

```
VPA Recommender Loop:
  1. Query metrics.k8s.io for current pod CPU/memory usage (every ~1 min)
  2. Store data points in an in-memory histogram (default 8 days lookback)
  3. Compute percentiles: p50 (target), p90 (upper bound), p5 (lower bound)
  4. Factor in OOM events (increases memory recommendation)
  5. Output: lowerBound, target, upperBound, uncappedTarget
  6. Constrain by minAllowed/maxAllowed
```

### VPA vs HPA — Different Metric Interpretation

| Aspect | HPA | VPA |
|--------|-----|-----|
| **Data window** | Current snapshot (last 15s–5min) | Historical (up to 8 days) |
| **Stored where** | Not stored (real-time query) | In-memory histograms in Recommender |
| **Smoothing** | Stabilization window (5min scale-down) | Percentile-based (gradual adjustment) |
| **OOM events** | Not considered | Increases memory recommendation |
| **Result** | Replica count (how many pods) | Resource values (how big each pod) |
| **Action** | Scale pods horizontally | Evict and recreate with new resources |

### What VPA Sees vs What HPA Sees

Same metric (CPU usage = 800m), different interpretation:

```
Pod with requests: cpu=1000m

HPA sees: 800m / 1000m = 80% utilization → scale up (if target is 70%)
VPA sees: 800m sustained over 8 days → recommend requests: 900m (with buffer)
```

## Metrics APIs Reference

### Resource Metrics API (metrics.k8s.io)

```bash
# Check if the API is available
kubectl get apiservices | grep metrics

# Get pod metrics directly
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/<ns>/pods/<pod>" | jq

# Get node metrics
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes/<node>" | jq
```

### Custom Metrics API (custom.metrics.k8s.io)

```bash
# Check if custom metrics API is available
kubectl get apiservices | grep custom

# List available custom metrics
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq '.resources[].name'

# Query a specific custom metric
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second" | jq
```

### External Metrics API (external.metrics.k8s.io)

```bash
# Check if external metrics API is available
kubectl get apiservices | grep external

# List external metrics
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1" | jq
```

## Common Metrics Adapters

| Adapter | Provides | Metrics Source |
|---------|----------|---------------|
| **metrics-server** | Resource metrics (CPU/memory) | kubelet cAdvisor |
| **Prometheus Adapter** | Custom + external metrics | Prometheus |
| **KEDA** | External metrics (60+ scalers) | SQS, Kafka, Redis, Datadog, etc. |
| **Stackdriver Adapter** | External metrics | Google Cloud Monitoring |
| **Azure Metrics Adapter** | External metrics | Azure Monitor |
| **Datadog Cluster Agent** | External metrics | Datadog |

## Troubleshooting Metrics

### HPA Shows "unknown" for Metrics

```bash
# Check if metrics-server is running
kubectl get pods -n kube-system -l k8s-app=metrics-server

# Check if Metrics API is registered
kubectl get apiservices v1beta1.metrics.k8s.io

# Check if pods have resource requests set (required for utilization%)
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].resources.requests}'

# Check HPA status and events
kubectl describe hpa <name>

# Check raw metrics
kubectl top pods -n <namespace>
```

Common causes:
- metrics-server not installed
- Pods have no `resources.requests` set (can't calculate percentage)
- metrics-server can't reach kubelets (network/TLS issues)
- Custom metrics adapter not installed (for custom metrics)

### VPA Recommendations Are Empty

```bash
# Check VPA status
kubectl describe vpa <name>

# Check if Recommender is running
kubectl get pods -n kube-system -l app=vpa-recommender

# Check Recommender logs
kubectl logs -n kube-system -l app=vpa-recommender --tail=50

# Check if metrics-server returns data for the target pods
kubectl top pods -n <namespace> -l <pod-selector>
```

Common causes:
- VPA Recommender not running
- metrics-server not installed
- Not enough data yet (needs time to build history)
- Target pods have no resource usage (idle)

### Verify the Full Metrics Chain

```bash
# 1. kubelet is exposing metrics
kubectl get --raw "/api/v1/nodes/<node>/proxy/metrics/resource" | head -20

# 2. metrics-server can aggregate
kubectl top nodes
kubectl top pods -A

# 3. Metrics API is serving
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/pods" | jq '.items | length'

# 4. HPA can read metrics
kubectl describe hpa <name> | grep -A5 "Metrics:"

# 5. VPA can read metrics
kubectl describe vpa <name> | grep -A10 "Recommendation:"
```

## Metrics Collection Frequency

| Component | Frequency | Configurable |
|-----------|-----------|-------------|
| cAdvisor (in kubelet) | Every 10s | `--housekeeping-interval` |
| metrics-server scrape | Every 15s | `--metric-resolution` |
| HPA evaluation | Every 15s | `--horizontal-pod-autoscaler-sync-period` |
| VPA Recommender | Every ~1 min | Recommender flags |
| Prometheus scrape | Every 15–60s | `scrape_interval` in Prometheus config |

## Running HPA and VPA Together

They can coexist if they don't compete on the same metric:

```yaml
# HPA scales on custom metric (requests per second)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
---
# VPA right-sizes CPU/memory requests
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
spec:
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      controlledResources: ["cpu", "memory"]
```

> **Rule:** HPA on custom/external metrics + VPA on CPU/memory = safe. HPA on CPU + VPA on CPU = conflict (VPA changes requests, which changes HPA's utilization percentage).
