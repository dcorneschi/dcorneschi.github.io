# EKS Node NotReady with I/O and CPU Spikes

## Issue

An EKS node was in NotReady state for a minute with spikes in I/O and CPU usage on Datadog chart.

## What It Means

An EKS node going `NotReady` with simultaneous I/O and CPU spikes indicates the node became unresponsive due to resource exhaustion. The combination suggests:

- A workload or process caused high CPU usage
- High CPU led to I/O wait (iowait) or the process was performing intensive disk operations
- The kubelet couldn't maintain heartbeats to the control plane due to resource starvation

## Common Causes

- **Runaway process** — A pod or system process consumed excessive CPU and disk resources
- **Disk I/O exhaustion** — The node's disk was overwhelmed (high IOPS usage, full disk, or slow EBS volume) causing CPU to spike in iowait
- **kubelet resource starvation** — The kubelet couldn't get CPU time to send heartbeats to the control plane
- **Pod eviction storm** — Resource pressure triggered pod evictions, causing cascading CPU/I/O as containers were terminated
- **OOM killer activity** — Memory pressure led to processes being killed, causing I/O spikes and CPU thrashing

## Investigation Commands

### Check node events around that time

```bash
kubectl describe node <node-name>
```

### Check if disk pressure was the issue

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,DISK-PRESSURE:.status.conditions[?(@.type=='DiskPressure')].status
```

### Check resource pressure on nodes

```bash
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, conditions: .status.conditions[] | select(.type=="MemoryPressure" or .type=="DiskPressure" or .type=="PIDPressure")}'
```

### Check top pods by CPU/memory at that time

```bash
kubectl top pods --all-namespaces --sort-by=cpu
kubectl top pods --all-namespaces --sort-by=memory
```

### Review CloudWatch metrics for the EBS volume

```bash
aws cloudwatch get-metric-statistics --namespace AWS/EBS \
  --metric-name VolumeQueueLength \
  --dimensions Name=VolumeId,Value=<volume-id> \
  --start-time <time> --end-time <time> \
  --period 60 --statistics Average
```

## Analysis

The combined I/O and CPU spikes indicate:

- **Most likely** — A pod or process caused high CPU usage while performing intensive disk operations (e.g., log processing, data indexing, backup)
- **Also possible** — High iowait from disk saturation caused CPU to spike waiting for I/O operations to complete
- **Result** — The kubelet couldn't maintain heartbeats due to resource starvation, triggering the NotReady state

The spikes were likely the **cause** of the NotReady state, though pod eviction during the incident may have amplified them.
