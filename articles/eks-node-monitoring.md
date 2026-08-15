# EKS Node Monitoring

How to monitor EKS node health, resource utilization, and capacity — using built-in Kubernetes mechanisms, CloudWatch, Prometheus, and Datadog.

## Node Conditions (Built-in Health Checks)

Every node reports health conditions that kubelet evaluates continuously:

```sh
# View all node conditions
kubectl describe node <node-name> | grep -A 20 "Conditions:"

# Get conditions as JSON
kubectl get node <node-name> -o jsonpath='{.status.conditions}' | jq
```

### Default Node Conditions

| Condition | Healthy Value | Meaning When Unhealthy |
|-----------|:------------:|------------------------|
| `Ready` | True | kubelet is healthy, can accept pods |
| `MemoryPressure` | False | Node running low on memory |
| `DiskPressure` | False | Node running low on disk space |
| `PIDPressure` | False | Too many processes on the node |
| `NetworkUnavailable` | False | Network not configured correctly (CNI issue) |

```sh
# Find nodes with problems
kubectl get nodes -o custom-columns=NAME:.metadata.name,READY:.status.conditions[-1].status

# Find NotReady nodes
kubectl get nodes --field-selector status.conditions.type=Ready,status.conditions.status!=True

# Quick health overview
kubectl get nodes -o wide
```

## kubectl top — Resource Usage

Requires Metrics Server to be installed:

```sh
# Install metrics server (if not present)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Node resource usage
kubectl top nodes

# Example output:
# NAME                           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# ip-10-0-1-42.ec2.internal     450m         11%    3200Mi          42%
# ip-10-0-2-18.ec2.internal     1200m        30%    5800Mi          76%

# Pod resource usage per node
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory

# Pods on a specific node
kubectl get pods -A --field-selector spec.nodeName=<node-name> -o wide
```

## Node Resource Allocation

Understanding the difference between capacity, allocatable, requests, and actual usage:

```sh
# Full node resource breakdown
kubectl describe node <node-name> | grep -A 20 "Allocated resources:"

# Example output:
# Allocated resources:
#   (Total limits may be over 100 percent, i.e., overcommitted.)
#   Resource           Requests    Limits
#   --------           --------    ------
#   cpu                1850m (46%) 4000m (100%)
#   memory             3200Mi (42%) 6144Mi (80%)
#   ephemeral-storage  0 (0%)      0 (0%)
```

| Metric | Meaning |
|--------|---------|
| **Capacity** | Total physical resources (CPU, memory) |
| **Allocatable** | Capacity minus kubelet/system reserved |
| **Requests** (sum of pods) | What the scheduler committed |
| **Limits** (sum of pods) | Maximum pods can consume |
| **Actual usage** | Real-time consumption (from `kubectl top`) |

```sh
# Compare allocatable vs requested across all nodes
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
CPU_ALLOC:.status.allocatable.cpu,\
MEM_ALLOC:.status.allocatable.memory,\
PODS_ALLOC:.status.allocatable.pods

# One-liner: find overcommitted nodes
kubectl describe nodes | grep -A 5 "Allocated resources" | grep -E "cpu|memory"
```

## CloudWatch Container Insights

AWS-native monitoring for EKS nodes and pods.

### Enable Container Insights

```sh
# Install CloudWatch agent + Fluent Bit
curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml \
  | sed "s/{{cluster_name}}/<cluster>/;s/{{region_name}}/<region>/" \
  | kubectl apply -f -

# Or use the EKS add-on
aws eks create-addon --cluster-name <cluster> --addon-name amazon-cloudwatch-observability
```

### Key CloudWatch Metrics for Nodes

| Metric | Namespace | Description |
|--------|-----------|-------------|
| `node_cpu_utilization` | ContainerInsights | CPU usage percentage |
| `node_memory_utilization` | ContainerInsights | Memory usage percentage |
| `node_filesystem_utilization` | ContainerInsights | Disk usage percentage |
| `node_network_total_bytes` | ContainerInsights | Network throughput |
| `node_number_of_running_pods` | ContainerInsights | Pod count per node |
| `node_cpu_reserved_capacity` | ContainerInsights | CPU requests as % of allocatable |
| `node_memory_reserved_capacity` | ContainerInsights | Memory requests as % of allocatable |

### CloudWatch Alarms for Node Health

```sh
# CPU alarm (>80% for 5 minutes)
aws cloudwatch put-metric-alarm \
  --alarm-name "eks-node-high-cpu" \
  --namespace ContainerInsights \
  --metric-name node_cpu_utilization \
  --dimensions Name=ClusterName,Value=<cluster> \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions <sns-topic-arn>

# Memory alarm (>85%)
aws cloudwatch put-metric-alarm \
  --alarm-name "eks-node-high-memory" \
  --namespace ContainerInsights \
  --metric-name node_memory_utilization \
  --dimensions Name=ClusterName,Value=<cluster> \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions <sns-topic-arn>

# Disk alarm (>90%)
aws cloudwatch put-metric-alarm \
  --alarm-name "eks-node-high-disk" \
  --namespace ContainerInsights \
  --metric-name node_filesystem_utilization \
  --dimensions Name=ClusterName,Value=<cluster> \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 90 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions <sns-topic-arn>
```

### CloudWatch Log Insights Queries

```
# Find nodes with high CPU (Container Insights logs)
fields @timestamp, NodeName, node_cpu_utilization
| filter node_cpu_utilization > 80
| sort @timestamp desc
| limit 20

# Find nodes with memory pressure
fields @timestamp, NodeName, node_memory_utilization
| filter node_memory_utilization > 85
| sort @timestamp desc

# Pod count per node
fields @timestamp, NodeName, node_number_of_running_pods
| stats max(node_number_of_running_pods) by NodeName
| sort max_node_number_of_running_pods desc
```

## EC2 Instance-Level Metrics

CloudWatch also tracks the underlying EC2 instance:

| Metric | Namespace | Description |
|--------|-----------|-------------|
| `CPUUtilization` | AWS/EC2 | EC2 CPU usage |
| `StatusCheckFailed` | AWS/EC2 | Instance or system check failed |
| `StatusCheckFailed_Instance` | AWS/EC2 | OS-level issue |
| `StatusCheckFailed_System` | AWS/EC2 | Hardware/hypervisor issue |
| `NetworkIn` / `NetworkOut` | AWS/EC2 | Network bytes |
| `EBSIOBalance%` | AWS/EC2 | Burst I/O balance remaining |

```sh
# Check EC2 status checks for all nodes
for id in $(kubectl get nodes -o jsonpath='{.items[*].spec.providerID}' | tr ' ' '\n' | awk -F/ '{print $NF}'); do
  status=$(aws ec2 describe-instance-status --instance-ids $id \
    --query "InstanceStatuses[0].{System:SystemStatus.Status, Instance:InstanceStatus.Status}" --output text)
  echo "$id: $status"
done
```

## Node Problem Detector

[node-problem-detector](https://github.com/kubernetes/node-problem-detector) detects issues that kubelet's built-in checks miss:

### What It Detects

- Kernel deadlocks
- Corrupted filesystems
- Unresponsive container runtime
- Hardware errors (MCE)
- NTP sync issues
- CNI health issues
- OOM kills (kernel-level)

### Install

```sh
# Helm install
helm repo add deliveryhero https://charts.deliveryhero.io
helm install node-problem-detector deliveryhero/node-problem-detector \
  -n kube-system

# Or apply directly
kubectl apply -f https://raw.githubusercontent.com/kubernetes/node-problem-detector/master/deployment/node-problem-detector.yaml
```

### Check Node Conditions (Extended)

After installing, nodes get additional conditions:

```sh
kubectl get node <node-name> -o jsonpath='{.status.conditions}' | jq '.[] | {type, status, message}'
```

Additional conditions reported:
- `KernelDeadlock`
- `ReadonlyFilesystem`
- `FrequentContainerdRestart`
- `FrequentKubeletRestart`
- `FrequentDockerRestart`

### View Node Problem Events

```sh
kubectl get events --field-selector source=node-problem-detector --sort-by='.lastTimestamp'
```

## Prometheus Monitoring

### Key Node Metrics (from node-exporter)

| Metric | Description |
|--------|-------------|
| `node_cpu_seconds_total` | CPU time by mode (user, system, idle, iowait) |
| `node_memory_MemAvailable_bytes` | Available memory |
| `node_filesystem_avail_bytes` | Available disk space |
| `node_network_receive_bytes_total` | Network bytes received |
| `node_load1` / `node_load5` / `node_load15` | Load average |
| `node_disk_io_time_seconds_total` | Disk I/O time |
| `node_vmstat_oom_kill` | OOM kill count |
| `node_nf_conntrack_entries` | Conntrack table entries |

### Key Kubelet Metrics

| Metric | Description |
|--------|-------------|
| `kubelet_running_pods` | Number of running pods |
| `kubelet_node_config_error` | Node config errors |
| `kubelet_runtime_operations_errors_total` | Container runtime errors |
| `kubelet_pod_start_duration_seconds` | Pod startup latency |
| `kubelet_evictions` | Node-initiated evictions |
| `kubelet_volume_stats_available_bytes` | PV available space |

### Prometheus Alert Rules for Nodes

```yaml
groups:
  - name: node-alerts
    rules:
      - alert: NodeNotReady
        expr: kube_node_status_condition{condition="Ready", status="true"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.node }} is NotReady"

      - alert: NodeHighCPU
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.instance }} CPU > 80%"

      - alert: NodeHighMemory
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.instance }} memory > 85%"

      - alert: NodeDiskFull
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.instance }} disk > 90%"

      - alert: NodeOOMKills
        expr: increase(node_vmstat_oom_kill[5m]) > 0
        labels:
          severity: warning
        annotations:
          summary: "OOM kills detected on {{ $labels.instance }}"

      - alert: NodeTooManyPods
        expr: kubelet_running_pods / on(node) kube_node_status_allocatable{resource="pods"} > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.node }} running at >90% pod capacity"
```

## Datadog Node Monitoring

### Key Metrics

| Metric | Description |
|--------|-------------|
| `kubernetes.cpu.usage.total` | CPU usage per node |
| `kubernetes.memory.usage` | Memory usage per node |
| `kubernetes.filesystem.usage_pct` | Disk usage percentage |
| `kubernetes.pods.running` | Running pods per node |
| `kubernetes_state.node.status` | Node ready status |
| `kubernetes_state.node.condition` | All node conditions |

### Datadog Monitor: Node NotReady

```json
{
  "name": "EKS Node NotReady",
  "type": "service check",
  "query": "\"kubernetes_state.node.ready\".over(\"cluster_name:<cluster>\").by(\"node\").last(3).count_by_status()",
  "message": "Node {{node.name}} is NotReady for the past 3 checks.\n\nCheck kubelet status and node conditions.\n\n@slack-infra-alerts",
  "options": {
    "thresholds": { "critical": 2, "ok": 1 }
  },
  "tags": ["team:platform", "env:production"]
}
```

### Datadog Monitor: Node High CPU

```json
{
  "name": "EKS Node High CPU",
  "type": "metric alert",
  "query": "avg(last_10m):avg:system.cpu.user{cluster_name:<cluster>} by {host} > 80",
  "message": "Node {{host.name}} CPU usage above 80% for 10 minutes.\n\n@slack-infra-alerts",
  "options": {
    "thresholds": { "critical": 80, "warning": 70 },
    "notify_no_data": false,
    "renotify_interval": 60
  }
}
```

### Datadog Monitor: Node Disk Pressure

```json
{
  "name": "EKS Node Disk Pressure",
  "type": "metric alert",
  "query": "avg(last_5m):avg:kubernetes.filesystem.usage_pct{cluster_name:<cluster>} by {host} > 85",
  "message": "Node {{host.name}} disk usage above 85%.\n\nConsider:\n- Cleaning up unused container images\n- Increasing node volume size\n- Checking for log accumulation\n\n@slack-infra-alerts",
  "options": {
    "thresholds": { "critical": 90, "warning": 85 }
  }
}
```

## Capacity Planning

### Current Capacity Usage

```sh
# Cluster-wide resource summary
kubectl top nodes

# Percentage used across the cluster
kubectl describe nodes | grep -E "cpu|memory" | grep -E "Requests|Limits|Allocatable"

# Node count by instance type
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.labels.node\.kubernetes\.io/instance-type}{"\n"}{end}' | sort | uniq -c

# Pods per node
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort | uniq -c | sort -rn
```

### Scaling Indicators

| Indicator | Signal | Action |
|-----------|--------|--------|
| CPU requests > 70% of allocatable | Cluster nearing CPU capacity | Scale out or use larger instances |
| Memory requests > 80% of allocatable | Cluster nearing memory capacity | Add nodes |
| Pods > 90% of allocatable pods | Hit pod density limit | Enable prefix delegation or add nodes |
| Pending pods > 0 for > 2 min | Insufficient capacity | Autoscaler should react, check why it's not |
| Node count at ASG max | Can't scale further | Increase ASG max or add node groups |

### Proactive Monitoring Script

```sh
#!/bin/bash
echo "=== EKS Node Health Summary ==="
echo ""
echo "--- Node Status ---"
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].status,VERSION:.status.nodeInfo.kubeletVersion,INSTANCE:.metadata.labels.node\\.kubernetes\\.io/instance-type

echo ""
echo "--- Resource Usage ---"
kubectl top nodes 2>/dev/null || echo "(metrics-server not available)"

echo ""
echo "--- Pods Per Node ---"
kubectl get pods -A --no-headers -o custom-columns=NODE:.spec.nodeName | sort | uniq -c | sort -rn

echo ""
echo "--- Pending Pods ---"
PENDING=$(kubectl get pods -A --field-selector status.phase=Pending --no-headers 2>/dev/null | wc -l)
echo "Pending pods: $PENDING"

echo ""
echo "--- NotReady Nodes ---"
kubectl get nodes --no-headers | grep -v " Ready " || echo "All nodes Ready"

echo ""
echo "--- Recent Node Events ---"
kubectl get events -A --field-selector involvedObject.kind=Node --sort-by='.lastTimestamp' 2>/dev/null | tail -10
```

## Gotchas

- **`kubectl top` shows actual usage; scheduler uses requests**: A node can show 20% CPU usage but be "full" because requests are committed at 100%. Watch both.
- **Metrics Server has a ~60s lag**: Don't use `kubectl top` for real-time debugging. It's delayed.
- **Container Insights costs money**: CloudWatch charges per metric and log ingestion. Estimate cost before enabling on large clusters.
- **Node conditions don't trigger alerts by default**: You need monitoring tools (Prometheus, Datadog, CloudWatch alarms) to alert on condition changes.
- **EC2 status checks ≠ Kubernetes node conditions**: A node can be healthy at the EC2 level but unhealthy in Kubernetes (e.g., kubelet crashing, CNI broken).
- **Spot interruption notices**: Not visible in node conditions. Use Node Termination Handler or Karpenter to detect and act on them.
- **Disk usage includes container images**: The root volume holds both OS files and cached container images. Large images can fill disk quickly.


## Network Traffic and Bandwidth

| Metric | Source | Why |
|--------|--------|-----|
| Bytes in/out per interface | CloudWatch `NetworkIn`/`NetworkOut`, node_exporter | Detect saturation against instance bandwidth limit |
| Throughput per pod/namespace | VPC Flow Logs, eBPF (Cilium/Hubble) | Identify noisy neighbors |
| Baseline bandwidth vs burst | CloudWatch `NetworkBandwidthInAllowanceExceeded` / `NetworkBandwidthOutAllowanceExceeded` | Nitro instances use credit-based burst; exceeding = throttling |

## Packets Per Second (PPS)

Each EC2 instance type has a hard PPS ceiling enforced at the Nitro card level. AWS publishes bandwidth figures (e.g., "Up to 10 Gbps"), but PPS is a separate, independent limit.

### PPS Limits by Instance Type

| Instance Size | Approximate PPS Limit |
|---------------|----------------------|
| t3.medium | ~250,000 |
| m5.large | ~750,000 |
| m5.xlarge | ~1,000,000 |
| m5.2xlarge | ~1,500,000 |
| m5.4xlarge | ~2,000,000 |
| c5n.18xlarge | ~15,000,000 |

These are aggregate (in + out combined). Small packets (DNS, health checks, microservice chatter) eat PPS budget fast without touching bandwidth limits.

### Why PPS Matters More Than Bandwidth on EKS

Kubernetes workloads generate lots of small packets:

- Service mesh sidecars (Envoy) → extra hops per request
- Health checks (liveness/readiness probes from kubelet, ALB target group health checks)
- DNS queries (every service resolution → UDP packets to CoreDNS)
- Conntrack state packets (SYN/FIN/RST)
- kube-proxy iptables/IPVS packet mangling

A node running 50+ pods with probes every 10s + inter-service gRPC can easily saturate PPS while using <20% of bandwidth.

### How to Monitor PPS

```sh
# Real-time PPS per interface
ethtool -S eth0 | grep -E "rx_packets|tx_packets"

# ENA allowance exceeded (tells you you're being throttled)
ethtool -S eth0 | grep pps_allowance_exceeded

# Kernel-level packet counters
cat /proc/net/dev
```

CloudWatch metrics:
- `NetworkPacketsIn` / `NetworkPacketsOut` — actual PPS
- `NetworkPacketsPerSecondAllowanceExceeded` — throttled packets (enhanced networking)

Prometheus (node_exporter):

```promql
# PPS rate
rate(node_network_receive_packets_total{device="eth0"}[1m])
rate(node_network_transmit_packets_total{device="eth0"}[1m])
```

### Symptoms of PPS Throttling

- Intermittent timeouts with no CPU/memory pressure
- DNS resolution failures (CoreDNS replies dropped)
- Increased latency on health checks → pods marked unhealthy
- TCP retransmits spike
- Application "connection reset" errors
- **No errors in system logs** — the drops happen at the Nitro card before reaching the OS

### PPS Mitigations

1. **Upsize the instance** — more PPS headroom
2. **Spread pods across more nodes** — reduce per-node PPS load
3. **Use NodeLocal DNSCache** — eliminates cross-node DNS packets
4. **Reduce probe frequency** — less health check traffic
5. **Batch/coalesce requests** — fewer, larger packets instead of many small ones
6. **Use c5n/m5n/r5n** — "n" variants have significantly higher PPS limits

## ENA Driver Counters

Pull with `ethtool -S <interface>` — these reveal throttling invisible to normal metrics:

| Counter | Meaning |
|---------|---------|
| `bw_in_allowance_exceeded` | Instance bandwidth cap hit (inbound) |
| `bw_out_allowance_exceeded` | Instance bandwidth cap hit (outbound) |
| `pps_allowance_exceeded` | PPS limit hit |
| `conntrack_allowance_exceeded` | Connection tracking table full (hypervisor-level) |
| `conntrack_allowance_available` | Remaining conntrack capacity (newer ENA) |
| `linklocal_allowance_exceeded` | Metadata/DNS rate limit hit (IMDS, Route 53 Resolver) |
| `rx_queue_N_drops` | Ring buffer overflow on queue N |
| `tx_queue_N_drops` | TX ring buffer overflow |

```sh
# Full ENA stats
ethtool -S eth0

# Just the allowance counters
ethtool -S eth0 | grep -E "allowance|drops"
```

> These are the most commonly missed issues on EKS nodes — they cause packet drops with zero visibility unless you're explicitly collecting them.

### Collecting ENA Counters at Scale

Use a DaemonSet with node_exporter's textfile collector or a custom exporter:

```sh
# Cron on each node (or in a DaemonSet) writing to textfile collector
#!/bin/bash
OUTPUT="/var/lib/prometheus/node-exporter/ena.prom"
for metric in bw_in_allowance_exceeded bw_out_allowance_exceeded pps_allowance_exceeded conntrack_allowance_exceeded linklocal_allowance_exceeded; do
  value=$(ethtool -S eth0 2>/dev/null | grep "$metric" | awk '{print $2}')
  echo "ena_${metric} ${value:-0}" >> "$OUTPUT.tmp"
done
mv "$OUTPUT.tmp" "$OUTPUT"
```

## Conntrack (Connection Tracking)

There are **two** conntrack limits on EKS nodes:

1. **Kernel-level** (`nf_conntrack_max`) — tunable via sysctl
2. **Hypervisor/ENI-level** — instance-type dependent, NOT tunable

| Metric | How to get it |
|--------|---------------|
| `nf_conntrack_count` | `/proc/sys/net/netfilter/nf_conntrack_count` |
| `nf_conntrack_max` | `/proc/sys/net/netfilter/nf_conntrack_max` |
| ENA `conntrack_allowance_exceeded` | `ethtool -S eth0` |
| Usage percentage | `count / max × 100` |

```sh
# Check current conntrack usage
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# Calculate usage percentage
echo "$(cat /proc/sys/net/netfilter/nf_conntrack_count) / $(cat /proc/sys/net/netfilter/nf_conntrack_max) * 100" | bc -l

# Increase kernel conntrack max (temporary)
sudo sysctl -w net.netfilter.nf_conntrack_max=524288

# Persistent (add to /etc/sysctl.d/)
echo "net.netfilter.nf_conntrack_max = 524288" | sudo tee /etc/sysctl.d/99-conntrack.conf
```

Conntrack exhaustion symptoms:
- New connections fail (`nf_conntrack: table full, dropping packet` in dmesg)
- Random pod-to-pod connection failures
- Service connections timing out intermittently

## CPU softirq Saturation

Network interrupt processing (`softirq`) can saturate a single CPU core even when overall CPU usage looks low:

```sh
# Check softirq time per core
cat /proc/softirqs | grep NET

# Watch in real-time
watch -n 1 "cat /proc/softirqs | head -3"

# Check if ksoftirqd is consuming an entire core
top -b -n 1 | grep ksoftirqd
```

Prometheus:

```promql
# Per-core softirq time (>30% = concern)
rate(node_softnet_processed_total[5m])
rate(node_softnet_dropped_total[5m])   # If non-zero = packets dropped
```

Single-core saturation is a hidden issue — `kubectl top nodes` shows 25% CPU on a 4-core node, but one core is at 100% handling all network interrupts.

Fix: use instances with more cores, enable RSS (Receive Side Scaling), or reduce packet rate.

## Recommended Alerting Thresholds

```yaml
# ENA throttling (any non-zero = problem)
- alert: ENABandwidthExceeded
  expr: rate(ena_bw_in_allowance_exceeded[5m]) > 0 or rate(ena_bw_out_allowance_exceeded[5m]) > 0
  labels:
    severity: warning

- alert: ENAPPSExceeded
  expr: rate(ena_pps_allowance_exceeded[5m]) > 0
  labels:
    severity: warning

- alert: ENAConntrackExceeded
  expr: rate(ena_conntrack_allowance_exceeded[5m]) > 0
  labels:
    severity: critical

- alert: ENALinklocalExceeded
  expr: rate(ena_linklocal_allowance_exceeded[5m]) > 0
  labels:
    severity: warning

# Conntrack saturation (kernel level)
- alert: ConntrackNearFull
  expr: node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.80
  for: 5m
  labels:
    severity: warning

# Network bandwidth (sustained)
- alert: NetworkBandwidthHigh
  expr: rate(node_network_receive_bytes_total{device="eth0"}[5m]) > (instance_bandwidth_limit * 0.80)
  for: 5m
  labels:
    severity: warning

# Softirq saturation (single core)
- alert: SoftirqSaturation
  expr: rate(node_softnet_dropped_total[5m]) > 0
  labels:
    severity: critical
```

## Kubernetes-Specific Network Metrics

| Metric | Source | Why |
|--------|--------|-----|
| Pod count vs `max-pods` limit | kubelet metrics | Approaching ENI/IP limits |
| ENI count / IPs assigned | VPC CNI (aws-node) metrics | IP exhaustion |
| `IPAMD` errors | VPC CNI logs | Pod IP allocation failures |
| DNS latency/failures | CoreDNS metrics | Often caused by `linklocal_allowance_exceeded` |
| `linklocal_allowance_exceeded` | ENA counters | IMDS + DNS + NTP share a 1024 PPS budget |

## Tooling Summary

| Tool | What It Gives You |
|------|-------------------|
| CloudWatch (enhanced monitoring) | Instance-level network allowance metrics |
| `ethtool -S` via DaemonSet / textfile collector | ENA counters (most critical) |
| Prometheus node_exporter | CPU, memory, disk, netdev, conntrack |
| kube-state-metrics | Pod counts, node conditions, resource requests |
| VPC Flow Logs | Per-flow traffic analysis |
| Cilium/Hubble or Calico flow logs | Pod-level L3/L4 visibility |
| Node Problem Detector | Kernel issues, OOM, filesystem corruption |
