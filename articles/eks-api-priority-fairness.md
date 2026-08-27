# EKS API Server Throttling and Priority & Fairness

How the Kubernetes API server on EKS throttles requests using API Priority and Fairness (APF) — FlowSchemas, PriorityLevelConfigurations, request queues, diagnosing 429 errors, and tuning for high-throughput clusters.

## Why API Throttling Exists

The API server has finite capacity. Without throttling, a misbehaving controller or aggressive CI/CD tool can monopolize the API server and starve critical system components (scheduler, kubelet heartbeats, controller-manager).

APF replaced the old `--max-requests-inflight` + `--max-mutating-requests-inflight` global counters with a fine-grained, per-flow queuing system.

## How APF Works — High-Level

```
┌──────────────────────────────────────────────────────────────────────┐
│  Incoming API Request                                                │
│                                                                      │
│  1. Match against FlowSchemas (ordered by matchingPrecedence)        │
│     → Determines which PriorityLevel the request belongs to          │
│                                                                      │
│  2. Assigned to a PriorityLevel queue                                │
│     → Each PriorityLevel has a share of total API server capacity    │
│     → Requests are fair-queued within the level                      │
│                                                                      │
│  3. If queue is full or capacity exceeded:                           │
│     → Request is rejected with 429 Too Many Requests                 │
│     → Retry-After header tells the client when to retry              │
│                                                                      │
│  4. If queue has capacity:                                           │
│     → Request is dispatched and processed normally                   │
└──────────────────────────────────────────────────────────────────────┘
```

```
                    API Request arrives
                          │
                          ▼
              ┌─────────────────────┐
              │   FlowSchema Match  │
              │   (by user, verb,   │
              │    resource, ns)    │
              └──────────┬──────────┘
                         │
           ┌─────────────┼─────────────────┐
           ▼             ▼                 ▼
    ┌──────────┐  ┌──────────────┐  ┌────────────┐
    │  exempt  │  │  system      │  │  workload  │
    │  (no     │  │  (high share)│  │  (limited  │
    │   limit) │  │              │  │   share)   │
    └──────────┘  └──────────────┘  └────────────┘
         │              │                  │
         ▼              ▼                  ▼
    Process         Queue + dispatch    Queue + dispatch
    immediately     (fair sharing)      (fair sharing)
                                             │
                                             ▼
                                    If full → 429
```

## Built-in Priority Levels (EKS Default)

EKS clusters come with these PriorityLevelConfigurations out of the box:

| Priority Level | Share | Queues | Description |
|---------------|:-----:|:------:|-------------|
| `exempt` | Unlimited | 0 | Never throttled (system:masters, healthz) |
| `system` | 30% | 64 | Core system components (scheduler, controller-manager) |
| `leader-election` | 10% | 16 | Leader election (Lease renewals) |
| `workload-high` | 40% | 128 | High-priority workload (service accounts in kube-system) |
| `workload-low` | 20% | 64 | Default for user requests |
| `global-default` | — | — | Catch-all (lowest priority) |
| `catch-all` | 5% | — | Anything that doesn't match above |

```bash
# See priority levels on your cluster:
kubectl get prioritylevelconfigurations

# Detailed view:
kubectl get prioritylevelconfigurations -o custom-columns=\
  NAME:.metadata.name,\
  TYPE:.spec.type,\
  SHARES:.spec.limited.nominalConcurrencyShares,\
  QUEUES:.spec.limited.limitResponse.queuing.queues
```

## FlowSchemas — Matching Requests to Priority Levels

FlowSchemas are evaluated in order of `matchingPrecedence` (lowest number = highest priority). The first match wins:

```bash
# List FlowSchemas (ordered by precedence):
kubectl get flowschemas --sort-by='.spec.matchingPrecedence'
```

### Key Built-in FlowSchemas

| FlowSchema | Precedence | Matches | Priority Level |
|-----------|:----------:|---------|---------------|
| `exempt` | 1 | system:masters group | exempt |
| `probes` | 2 | health check endpoints | exempt |
| `system-leader-election` | 100 | system SA Lease operations | leader-election |
| `system-node-high` | 400 | kubelet requests (node status, pods) | system |
| `system-nodes` | 500 | other node requests | system |
| `kube-controller-manager` | 800 | controller-manager SA | workload-high |
| `kube-scheduler` | 800 | scheduler SA | workload-high |
| `service-accounts` | 9000 | all authenticated SAs | workload-low |
| `global-default` | 9900 | everything else | global-default |
| `catch-all` | 10000 | catch-all | catch-all |

```bash
# See what FlowSchema a request would match:
kubectl get flowschemas -o custom-columns=\
  NAME:.metadata.name,\
  PRECEDENCE:.spec.matchingPrecedence,\
  PRIORITY:.spec.priorityLevelConfiguration.name,\
  MATCHING:.spec.rules[*].subjects[*].kind

# Describe a specific FlowSchema:
kubectl describe flowschema service-accounts
```

## How Concurrency Shares Work

The API server has a total concurrency limit (how many requests can be processed simultaneously). Shares divide this capacity:

```
Total API server concurrency: 600 (EKS typical)

Priority Level         Shares    Actual Concurrency
────────────────       ──────    ──────────────────
system                 30%       180 concurrent requests
leader-election        10%       60
workload-high          40%       240
workload-low           20%       120
────────────────       ──────    ──────────────────
Total                  100%      600
```

If a priority level's queue is empty, its unused capacity is borrowed by other levels (work-stealing). Shares are a guarantee, not a cap.

## Queuing and Fair Dispatch

Within each priority level, requests are distributed across queues using **shuffle sharding**:

```
┌──────────────────────────────────────────────────────────┐
│  PriorityLevel: workload-low (120 concurrency, 64 queues)│
│                                                          │
│  Queue assignment (shuffle sharding):                    │
│    - Each "flow" (user + namespace + resource) is        │
│      deterministically mapped to a subset of queues      │
│    - Prevents one flow from blocking all others          │
│                                                          │
│  Dispatch:                                               │
│    - Fair queuing: oldest request from least-served queue│
│    - Virtual time tracking per queue                     │
│    - No single flow can monopolize the level             │
│                                                          │
│  Queue full:                                             │
│    - Request rejected with 429                           │
│    - Retry-After: 1 (seconds)                            │
└──────────────────────────────────────────────────────────┘
```

## Diagnosing 429 Throttling

### Symptoms

```bash
# In application/controller logs:
# "the server was unable to return a response in the time allotted"
# "Too Many Requests" (HTTP 429)
# "request throttled" 

# kubectl may show:
# Error from server (TooManyRequests): ...
```

### Check APF Metrics

```bash
# Port-forward to API server metrics (if accessible):
kubectl get --raw /metrics | grep apiserver_flowcontrol

# Key metrics:
# apiserver_flowcontrol_rejected_requests_total          — 429s issued
# apiserver_flowcontrol_dispatched_requests_total        — requests processed
# apiserver_flowcontrol_current_inqueue_requests         — requests waiting
# apiserver_flowcontrol_request_concurrency_limit        — max concurrency per level
# apiserver_flowcontrol_current_executing_requests       — active requests per level

# Quick check for rejected requests:
kubectl get --raw /metrics 2>/dev/null | grep "apiserver_flowcontrol_rejected_requests_total" | grep -v "^#"
```

### Check via API Server Audit Logs (EKS CloudWatch)

```bash
# Enable API server audit logging:
aws eks update-cluster-config --name <cluster> \
  --logging '{"clusterLogging":[{"types":["api","audit"],"enabled":true}]}'

# Query CloudWatch for throttled requests:
aws logs filter-log-events \
  --log-group-name /aws/eks/<cluster>/cluster \
  --log-stream-name-prefix kube-apiserver-audit \
  --filter-pattern '"responseStatus.code":429'
```

### Identify What's Being Throttled

```bash
# Check which priority levels are rejecting:
kubectl get --raw /metrics | grep "apiserver_flowcontrol_rejected_requests_total{" | sort -t'=' -k2 -rn

# Check queue depth (requests waiting):
kubectl get --raw /metrics | grep "apiserver_flowcontrol_current_inqueue_requests{" | grep -v "^#"

# Check which flows are consuming the most:
kubectl get --raw /metrics | grep "apiserver_flowcontrol_dispatched_requests_total{" | sort -t'"' -k4 -rn | head -20
```

## Common Throttling Scenarios on EKS

| Scenario | What's Throttled | Why | Fix |
|----------|-----------------|-----|-----|
| ArgoCD with 200+ apps | `service-accounts` flow | High LIST/WATCH frequency | Create a dedicated FlowSchema with higher priority |
| Custom operator polling aggressively | `workload-low` level | Too many GET/LIST requests | Use informers (watch) instead of polling |
| Large Helm install (many objects) | `workload-low` level | Burst of CREATE requests | Batch or throttle client-side |
| CI/CD deploying to many namespaces | `service-accounts` flow | Concurrent kubectl apply | Stagger deployments, increase concurrency shares |
| Cluster Autoscaler + many pending pods | `system` level (if SA in kube-system) | Rapid node provisioning triggers many API calls | Usually self-resolves; increase shares if chronic |
| Datadog/monitoring agents | `workload-low` level | LIST all pods/nodes frequently | Use watch-based collection, reduce polling |

## Creating Custom FlowSchemas

Give a critical controller more API capacity:

```yaml
# Dedicated FlowSchema for ArgoCD (higher priority than default):
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: argocd-high-priority
spec:
  matchingPrecedence: 1000    # Lower than system, higher than service-accounts (9000)
  priorityLevelConfiguration:
    name: workload-high        # Use the high-priority level
  distinguisherMethod:
    type: ByNamespace          # Fair-queue per namespace
  rules:
  - subjects:
    - kind: ServiceAccount
      serviceAccount:
        name: argocd-application-controller
        namespace: argocd
    - kind: ServiceAccount
      serviceAccount:
        name: argocd-server
        namespace: argocd
    resourceRules:
    - verbs: ["*"]
      apiGroups: ["*"]
      resources: ["*"]
      namespaces: ["*"]
```

### Create a Custom PriorityLevel (If Needed)

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: custom-critical
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 50    # Higher share
    lendablePercent: 50              # Can lend 50% when idle
    limitResponse:
      type: Queue
      queuing:
        queues: 32
        handSize: 6
        queueLengthLimit: 50
```

## EKS-Specific Considerations

### You Cannot Modify the API Server Directly

On EKS, you can't change `--max-requests-inflight` or other API server flags. You control throttling via:
- FlowSchemas (which requests go where)
- PriorityLevelConfigurations (how much capacity each level gets)

### EKS Provisioned Mode

For high-throughput clusters, EKS Provisioned mode pre-allocates API server capacity, effectively increasing the total concurrency budget that APF divides:

```bash
# Check if you're on Standard or Provisioned:
aws eks describe-cluster --name <cluster> --query "cluster.computeConfig"
```

### System Components Are Protected

EKS pre-installs FlowSchemas that protect:
- Kubelet heartbeats (node status updates)
- kube-scheduler and kube-controller-manager
- Leader election (Lease renewals)
- Health checks (readiness/liveness probes via API)

Even under heavy user load, these system flows aren't starved.

## Monitoring APF on EKS

### Prometheus Queries

```promql
# 429 rejection rate by priority level:
rate(apiserver_flowcontrol_rejected_requests_total[5m])

# Requests waiting in queue:
apiserver_flowcontrol_current_inqueue_requests

# Queue utilization per priority level:
apiserver_flowcontrol_current_executing_requests / apiserver_flowcontrol_request_concurrency_limit

# Which flows generate the most requests:
topk(10, rate(apiserver_flowcontrol_dispatched_requests_total[5m]))
```

### CloudWatch Metrics (EKS)

EKS surfaces some API server metrics in CloudWatch (if control plane logging is enabled):
- Look for `429` status codes in audit logs
- Monitor `apiserver_request_total` with status code filtering

## Debugging Flow

```
Receiving 429 errors?
    │
    ├── 1. Which priority level is rejecting?
    │      kubectl get --raw /metrics | grep rejected
    │
    ├── 2. Which FlowSchema is the request hitting?
    │      kubectl describe flowschema <name>
    │      Or check audit log annotations
    │
    ├── 3. Is it a burst or sustained?
    │      Burst: increase queueLengthLimit
    │      Sustained: increase shares or add dedicated FlowSchema
    │
    ├── 4. Can the client reduce request rate?
    │      Use informers/watches instead of LIST polling
    │      Implement exponential backoff
    │      Batch operations
    │
    └── 5. Is the cluster overloaded overall?
           Consider EKS Provisioned mode
           Or reduce total API call volume
```

## Quick Reference

```bash
# APF objects:
kubectl get flowschemas --sort-by='.spec.matchingPrecedence'
kubectl get prioritylevelconfigurations

# Check for throttling:
kubectl get --raw /metrics | grep "apiserver_flowcontrol_rejected_requests_total"
kubectl get --raw /metrics | grep "apiserver_flowcontrol_current_inqueue_requests"

# Create custom FlowSchema to prioritize a controller:
# matchingPrecedence: 1000 (between system and user default)
# priorityLevelConfiguration: workload-high

# EKS specifics:
# - Can't modify API server flags directly
# - Control via FlowSchemas and PriorityLevelConfigurations
# - Provisioned mode increases total concurrency budget
# - System components are always protected (exempt/system levels)
# - Enable audit logging to see 429s in CloudWatch

# Key metrics:
# apiserver_flowcontrol_rejected_requests_total (429s)
# apiserver_flowcontrol_current_inqueue_requests (queued)
# apiserver_flowcontrol_current_executing_requests (active)
# apiserver_flowcontrol_request_concurrency_limit (capacity)
```
