# Kubernetes Cluster Autoscaler Tuning Guide

A reference for Cluster Autoscaler parameters — scale-up/down timing, resource limits, performance tuning, and recommended configurations for different use cases.

## Core Scaling Parameters

### Scale-Up Options

| Parameter | Default | What It Does |
|-----------|---------|-------------|
| `--scale-up-delay-after-add` | 10m | Time to wait before scaling up after adding a node |
| `--scale-up-delay-after-delete` | 10s | Time to wait before scaling up after deleting a node |
| `--scale-up-delay-after-failure` | 3m | Time to wait before scaling up after a failed scale-up |

**Optimization notes:**
- Reduce `--scale-up-delay-after-add` to 3–5m for faster response to load spikes
- Reduce `--scale-up-delay-after-failure` to 1–2m for faster recovery
- Risk of too-low values: unnecessary scaling (thrashing)

### Scale-Down Options

| Parameter | Default | What It Does |
|-----------|---------|-------------|
| `--scale-down-delay-after-add` | 10m | Time to wait before scaling down after adding a node |
| `--scale-down-delay-after-delete` | 10s | Time to wait before scaling down after deleting a node |
| `--scale-down-delay-after-failure` | 3m | Time to wait before scaling down after a failed scale-down |
| `--scale-down-unneeded-time` | 10m | How long a node must be unneeded before eligible for removal |
| `--scale-down-utilization-threshold` | 0.5 | Node utilization below which it's considered for removal |

**Optimization notes:**
- Reduce `--scale-down-unneeded-time` to 5–8m for cost savings (watch for thrashing)
- Lower `--scale-down-utilization-threshold` to 0.2–0.3 for aggressive cost optimization
- Keep `--scale-down-delay-after-add` at 10m to avoid immediately removing just-added nodes

## Resource Management

### Node Group Limits

| Parameter | Default | What It Does |
|-----------|---------|-------------|
| `--nodes=<min>:<max>:<nodegroup>` | — | Sets min/max nodes per node group |
| `--max-nodes-total` | 0 (unlimited) | Maximum total nodes across all node groups |
| `--cores-total=<min>:<max>` | — | Limits total CPU cores in cluster |
| `--memory-total=<min>:<max>` | — | Limits total memory in cluster |

**Optimization notes:**
- Set realistic mins (avoid 0 for critical workloads)
- Set `--max-nodes-total` based on budget/quota limits
- Align `--cores-total` and `--memory-total` with actual workload requirements

### Node Protection

| Parameter | Default | What It Does |
|-----------|---------|-------------|
| `--skip-nodes-with-local-storage` | true | Skip nodes with local storage when scaling down |
| `--skip-nodes-with-system-pods` | true | Skip nodes with system pods when scaling down |

Keep both at `true` for data safety and cluster stability.

## Performance Tuning

### Scan Intervals

| Parameter | Default | What It Does |
|-----------|---------|-------------|
| `--scan-interval` | 10s | How often to evaluate cluster state |
| `--max-node-provision-time` | 15m | Maximum time to wait for node provisioning |

**Optimization notes:**
- Increase `--scan-interval` to 30–60s for large clusters to reduce API server load
- Adjust `--max-node-provision-time` based on cloud provider (AWS: ~10m, GCP: ~5m)

### Batch Operations

| Parameter | Default | What It Does |
|-----------|---------|-------------|
| `--max-empty-bulk-delete` | 10 | Maximum empty nodes to delete in one batch |
| `--max-graceful-termination-sec` | 600 | Maximum time to wait for graceful pod termination |

**Optimization notes:**
- Increase `--max-empty-bulk-delete` to 20–50 for faster scale-down in large clusters
- Reduce `--max-graceful-termination-sec` to 300s if pods terminate quickly

## Advanced Options

### Node Selection (Expander)

| Expander | Behavior |
|----------|----------|
| `random` | Pick a random node group (default) |
| `most-pods` | Choose the node group that schedules the most pending pods |
| `least-waste` | Choose the node group with the least idle resources after scaling |
| `price` | Choose the cheapest node group |
| `priority` | Use a user-defined priority list |

Set via `--expander=<value>`. Use `least-waste` for cost efficiency, `priority` for mixed instance types.

### Availability

| Parameter | Default | What It Does |
|-----------|---------|-------------|
| `--balance-similar-node-groups` | false | Balance pods across similar node groups |

Enable for better availability across AZs.

### Logging

| Parameter | What It Does |
|-----------|-------------|
| `--v=2` | Production logging level |
| `--v=4` | Debug logging level |
| `--logtostderr` | Log to stderr (enable for container environments) |

## Cloud Provider Specific

### AWS

```bash
# Auto-discovery via ASG tags
--node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<cluster-name>

# Faster instance type discovery
--aws-use-static-instance-list=true
```

### GCP

```bash
# Enable for regional persistent disks
--regional
```

## Recommended Configurations

### Cost-Optimized

```
--scale-down-utilization-threshold=0.2
--scale-down-unneeded-time=5m
--scale-up-delay-after-add=5m
--expander=least-waste
--max-empty-bulk-delete=20
```

Aggressive scale-down, minimizes idle resources. Best for non-critical workloads or dev/staging.

### Performance-Optimized

```
--scale-down-utilization-threshold=0.3
--scale-down-unneeded-time=8m
--scale-up-delay-after-add=3m
--scan-interval=30s
--expander=most-pods
```

Fast scale-up, moderate scale-down. Best for latency-sensitive workloads.

### Stability-Focused

```
--scale-down-utilization-threshold=0.4
--scale-down-unneeded-time=15m
--scale-up-delay-after-add=10m
--balance-similar-node-groups=true
--expander=priority
```

Conservative — avoids thrashing, balances across AZs. Best for production workloads.

## Key Optimization Strategies

1. **Monitor and adjust** — Start conservative, monitor metrics, then optimize
2. **Workload-specific** — Tune based on your application traffic patterns
3. **Cost vs performance** — Balance aggressive scaling with stability needs
4. **Test thoroughly** — Use staging environments to validate changes
5. **Set alerts** — Monitor scaling events and node utilization

## Common Pitfalls

- Setting thresholds too low → node thrashing (rapid add/remove cycles)
- Not accounting for system pod resource requirements → nodes appear "empty" but can't be removed
- Ignoring Pod Disruption Budgets → scale-down blocked indefinitely
- Over-aggressive scale-down in production → capacity gaps during traffic spikes
- Not setting appropriate resource requests on pods → autoscaler can't predict capacity needs
- Forgetting `--balance-similar-node-groups` → unbalanced AZ distribution after scale events
