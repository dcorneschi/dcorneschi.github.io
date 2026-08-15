# Cluster Autoscaler Restarts — "RequestError: send request failed, Service Unavailable"

When the cluster-autoscaler keeps restarting and logs show `RequestError: send request failed` with `Service Unavailable`, the issue is usually the Kubernetes API server being temporarily unreachable — not AWS API throttling.

## Understanding the Error

```
RequestError: send request failed
caused by: Get "https://<api-server-endpoint>/...": Service Unavailable
```

- **Service Unavailable (503)** = the kube-apiserver rejected the connection
- This is a Kubernetes API error, not an AWS API error
- If it were AWS throttling, you'd see `RequestLimitExceeded` or `Throttling` instead

## Common Causes

| Cause | Explanation |
|-------|-------------|
| API server overload | Too many requests from controllers, Terraform, monitoring, CI — apiserver returns 503 |
| API server rolling restart | During EKS control plane upgrades/patches, brief unavailability |
| Network connectivity issues | VPC CNI problems, ENI limits, DNS resolution failures between pod and API server |
| Node resource pressure | Pod gets OOMKilled or evicted, kubelet restarts it |
| Liveness probe failure | Autoscaler can't reach API server long enough, probe fails → restart |
| AWS API throttling (indirect) | Autoscaler's AWS calls fail → falls behind → queues retries → stresses API server further |

## Diagnosis

### Check Restart Reason

```sh
kubectl describe pod -n kube-system -l app=cluster-autoscaler
```

Look for:
- `OOMKilled` — needs more memory
- `Liveness probe failed` — API server was unreachable
- `Back-off restarting failed container` — repeated crashes

### Previous Container Logs

```sh
kubectl logs -n kube-system -l app=cluster-autoscaler --previous
```

### API Server Events Around Restart Time

```sh
kubectl get events -n kube-system --sort-by='.lastTimestamp' | grep -i "unhealthy\|restart\|backoff\|503"
```

### Check API Server Metrics

```sh
kubectl get --raw /metrics | grep apiserver_request_total | grep 503
```

### Check if AWS API Throttling Is Also Happening

```sh
aws cloudtrail lookup-events \
  --max-results 50 \
  --start-time $(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ') \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=autoscaling.amazonaws.com | \
  jq '.Events[].CloudTrailEvent | fromjson | select(.errorCode != null) | {eventTime, eventName, errorCode}'
```

## Fixes

### 1. Increase Liveness Probe Tolerances

The default probe is aggressive. Give it more room:

```yaml
livenessProbe:
  httpGet:
    path: /health-check
    port: 8085
  initialDelaySeconds: 30
  periodSeconds: 15
  failureThreshold: 5
  timeoutSeconds: 5
```

This allows ~75 seconds of API server unavailability before restart instead of the default ~30s.

### 2. Set PriorityClass to Prevent Eviction

```yaml
priorityClassName: system-cluster-critical
```

Ensures the autoscaler pod won't be evicted when nodes are under resource pressure.

### 3. Set Proper Resource Requests and Limits

```yaml
resources:
  requests:
    cpu: 100m
    memory: 600Mi
  limits:
    memory: 1Gi
```

For large clusters (100+ nodes), increase memory — the autoscaler caches all node/pod state in memory.

### 4. Reduce Scan Interval for Large Clusters

```sh
--scan-interval=30s   # default is 10s
```

Reduces API server LIST calls significantly. Each scan lists all pods and nodes.

### 5. Reduce API Server Load from Other Sources

- Lower Terraform parallelism: `terraform plan -parallelism=2`
- Stagger CI/CD pipelines (don't run all at :00)
- Check monitoring tools (Datadog, Prometheus) scrape intervals
- Reduce unnecessary watches/list calls from other controllers
- Review custom operators and their reconcile frequency

### 6. Use Expander Priority and Limit Node Groups

```sh
--expander=priority
--node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<cluster-name>
```

Fewer ASGs to poll = fewer AWS API calls = less pressure on everything.

### 7. Add the safe-to-evict Annotation

```sh
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler \
  cluster-autoscaler.kubernetes.io/safe-to-evict="false"
```

Prevents the autoscaler from evicting itself during scale-down.

## Relationship with AWS API Throttling

The cluster-autoscaler makes calls to **both**:

1. **Kubernetes API** — list pods, nodes, check pending pods
2. **AWS APIs** — describe ASGs, set desired capacity, describe launch templates

### AWS API Calls Made by Cluster-Autoscaler

| API Call | Purpose |
|----------|---------|
| `DescribeAutoScalingGroups` | Get current ASG state |
| `DescribeLaunchTemplates` | Determine instance types |
| `SetDesiredCapacity` | Scale up/down |
| `TerminateInstanceInAutoScalingGroup` | Scale down specific node |
| `DescribeInstances` | Verify instance status |

All of these share the EC2/AutoScaling token bucket with Terraform and other tooling.

If AWS APIs are throttled:
- Autoscaler's scaling decisions are delayed
- Retries pile up
- The pod may appear unresponsive to liveness probes
- Kubelet restarts it

### Identifying AWS Throttling vs API Server Issues

| Log Pattern | Source | Meaning |
|-------------|--------|---------|
| `Service Unavailable` | Kubernetes API | API server overloaded or unreachable |
| `RequestLimitExceeded` | AWS API | AWS throttling the autoscaler's calls |
| `Throttling: Rate exceeded` | AWS API | Same as above (different format) |
| `TLS handshake timeout` | Kubernetes API | Network issue to API server |
| `context deadline exceeded` | Either | Request took too long |

## Monitoring

### Datadog

```
# Autoscaler restarts
sum:kubernetes.containers.restarts{kube_deployment:cluster-autoscaler}.rollup(sum, 3600)

# API server 503s
sum:kubernetes_apiserver.request.count{code:503}.as_count()

# AWS throttling from autoscaler
sum:aws.apigateway.4xx_error{service:autoscaling}.as_count()
```

### Prometheus / Grafana

```promql
# Restart count
kube_pod_container_status_restarts_total{container="cluster-autoscaler"}

# API server request latency
histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket{verb="LIST", resource="pods"}[5m]))

# API server error rate
sum(rate(apiserver_request_total{code=~"5.."}[5m])) / sum(rate(apiserver_request_total[5m]))

# Cluster autoscaler unschedulable pods (its core metric)
cluster_autoscaler_unschedulable_pods_count
```

### CloudWatch (Cluster Autoscaler specific)

```sh
# Check if autoscaler pod has restarted recently
kubectl get pods -n kube-system -l app=cluster-autoscaler -o jsonpath='{.items[0].status.containerStatuses[0].restartCount}'
```

## Helm Values for Resilient Deployment

```yaml
# values.yaml for cluster-autoscaler Helm chart
replicaCount: 1

priorityClassName: system-cluster-critical

resources:
  requests:
    cpu: 100m
    memory: 600Mi
  limits:
    memory: 1Gi

extraArgs:
  scan-interval: 30s
  scale-down-delay-after-add: 10m
  scale-down-unneeded-time: 10m
  skip-nodes-with-local-storage: false
  balance-similar-node-groups: true
  expander: priority

podAnnotations:
  cluster-autoscaler.kubernetes.io/safe-to-evict: "false"

livenessProbe:
  httpGet:
    path: /health-check
    port: 8085
  initialDelaySeconds: 30
  periodSeconds: 15
  failureThreshold: 5
  timeoutSeconds: 5
```

## Summary

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `Service Unavailable` + restarts | API server overloaded or briefly down | Increase probe tolerances, reduce API load |
| `RequestLimitExceeded` in logs | AWS API throttling | Reduce scan interval, request quota increase |
| `OOMKilled` in pod events | Not enough memory | Increase memory limits |
| Restarts only during upgrades | EKS control plane patching | Expected — increase probe tolerance |
| Restarts correlate with Terraform runs | Shared API token bucket exhaustion | Stagger Terraform, reduce parallelism |
| Restarts after adding many nodes | Autoscaler caching large cluster state | Increase memory, increase scan interval |
