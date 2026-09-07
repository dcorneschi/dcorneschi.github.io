# CPU Starvation Diagnostic Guide

## Overview

When a process like Defender or any other high-CPU consumer is affecting pod performance, use this guide to diagnose CPU starvation and its impact on other pods.

## Symptoms of CPU Starvation

- Pods experiencing CrashLoopBackOff
- Liveness/readiness probe timeouts
- Slow application response times
- Increased pod restart counts
- Health check failures with "context deadline exceeded"

## Node-Level Diagnostics

### 1. Check Overall Node CPU Usage

```bash
# View node CPU consumption
kubectl top nodes

# View all pods sorted by CPU usage
kubectl top pods -A --sort-by=cpu

# View pods in specific namespace
kubectl top pods -n <namespace> --sort-by=cpu
```

### 2. Check CPU Throttling (On Node)

**For cgroup v2 (EKS Ubuntu AMI, newer systems):**

```bash
# Check if using cgroup v2
mount | grep cgroup2

# Find kubepods cgroups
find /sys/fs/cgroup -name "*.slice" -path "*kubepods*" 2>/dev/null | head -5

# Check throttling for all kubepods (show only non-zero)
grep -r "throttled" /sys/fs/cgroup/kubepods.slice/*/cpu.stat 2>/dev/null | grep -v ":0$"

# More detailed view with pod identification
for pod_dir in /sys/fs/cgroup/kubepods.slice/kubepods-*.slice/*/; do
  if [ -f "$pod_dir/cpu.stat" ]; then
    THROTTLED=$(grep "throttled_usec" "$pod_dir/cpu.stat" 2>/dev/null | awk '{print $2}')
    if [ "$THROTTLED" -gt 0 ] 2>/dev/null; then
      echo "=== $(basename $pod_dir) ==="
      grep "throttled" "$pod_dir/cpu.stat" 2>/dev/null
    fi
  fi
done
```

**For cgroup v1 (older systems):**

```bash
# Check cgroup v1 throttling
cat /sys/fs/cgroup/cpu/kubepods/*/cpu.stat | grep throttled

# More detailed view
for d in /sys/fs/cgroup/cpu/kubepods/*/; do
  echo "=== $d ==="
  grep . "$d/cpu.stat" 2>/dev/null | grep throttled
done
```

**Key metrics:**

- `nr_throttled`: Number of times the cgroup was throttled
- `throttled_usec` (v2) or `throttled_time` (v1): Total time the cgroup was throttled
- `nr_periods`: Number of enforcement intervals
- `nr_bursts`: Number of times burst capacity was used (v2 only)

### 3. Check Node Resource Pressure

```bash
# Check for CPU pressure on nodes
kubectl describe nodes | grep -A 5 "Conditions:"

# Look for:
# - MemoryPressure: False
# - DiskPressure: False
# - PIDPressure: False
```

## Pod-Level Diagnostics

### 4. Identify Pods with Probe Failures

```bash
# Check for unhealthy events across all namespaces
kubectl get events --all-namespaces --field-selector reason=Unhealthy

# Check for failed liveness probes
kubectl get events --all-namespaces --field-selector reason=Unhealthy | grep -i liveness

# Check specific pod events
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Events:"
```

**Look for error messages:**

- `Liveness probe failed: Get http://...: context deadline exceeded`
- `Liveness probe failed: Client.Timeout exceeded while awaiting headers`
- `Readiness probe failed: context deadline exceeded`

### 5. Check Pod Restart Counts

```bash
# List pods with restarts
kubectl get pods -A | grep -v "0/0" | awk '$5 > 0 {print $0}'

# Sort pods by restart count
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount'

# Check for CrashLoopBackOff
kubectl get pods -A | grep -E "CrashLoopBackOff|Error"
```

### 6. Check Pod Resource Limits and Requests

```bash
# View resource configuration for a pod
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].resources}' | jq

# Check all pods for CPU limits
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.containers[].resources.limits.cpu != null) | "\(.metadata.namespace)/\(.metadata.name): \(.spec.containers[].resources.limits.cpu)"'

# Check pods without CPU limits (recommended configuration)
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.containers[].resources.limits.cpu == null) | "\(.metadata.namespace)/\(.metadata.name): No CPU limit"'
```

## Container-Level Diagnostics

### 7. Check Specific Container CPU Usage

```bash
# Get detailed container stats (requires metrics-server)
kubectl top pod <pod-name> -n <namespace> --containers

# Check container status
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[*]}' | jq
```

### 8. Check High-CPU Process (e.g., Defender)

```bash
# Check Defender/Twistlock resource usage
kubectl top pod -n twistlock

# Check any DaemonSet resource usage
kubectl top pod -n <namespace> -l app=<daemonset-label>

# On the node, find high CPU processes
top -b -n 1 | head -20

# Find container processes
ps aux --sort=-%cpu | head -20
```

## Prometheus Metrics (If Available)

### 9. Query CPU Throttling Metrics

```promql
# Total throttled time per container
rate(container_cpu_cfs_throttled_seconds_total[5m])

# Throttling periods per container
rate(container_cpu_cfs_throttled_periods_total[5m])

# CPU usage vs limits
container_cpu_usage_seconds_total / container_spec_cpu_quota

# Identify containers being throttled
rate(container_cpu_cfs_throttled_periods_total[5m]) > 0
```

## Analysis Steps

### Step 1: Identify the High-CPU Consumer

1. Run `kubectl top pods -A --sort-by=cpu`
2. Identify which pod/process is consuming the most CPU
3. Check if it's a DaemonSet (runs on every node) like Defender

### Step 2: Check for CPU Limits

1. Run resource limit checks on affected pods
2. Pods with CPU limits are more likely to be throttled
3. Compare CPU requests vs limits vs actual usage

### Step 3: Correlate with Pod Restarts

1. Check if high CPU usage correlates with pod restarts
2. Look at the timing of events
3. Check if restarts happen during CPU spikes

### Step 4: Check Throttling Evidence

1. Look for throttling in cgroup stats
2. Check Prometheus metrics for throttling
3. Correlate throttling with probe failures

## Common Patterns

### Pattern 1: CPU Limit Throttling

- Pod has CPU limit set (e.g., 500m)
- Pod hits limit during load spikes
- Gets throttled even with spare node capacity
- Liveness probes timeout
- Pod restarts

**Solution:** Remove CPU limits, keep only requests

### Pattern 2: Node CPU Saturation

- Node CPU at 90%+ utilization
- Multiple pods competing for CPU
- No throttling but slow response times
- Probes timeout due to actual CPU shortage

**Solution:** Add more nodes or reduce pod density

### Pattern 3: DaemonSet CPU Competition

- DaemonSet (like Defender) uses significant CPU
- Application pods get less CPU than expected
- CPU requests honored but performance degraded
- Probes timeout under load

**Solution:** Adjust probe timeouts, optimize DaemonSet config

## Quick Diagnostic Commands Summary

```bash
# 1. Node CPU overview
kubectl top nodes

# 2. Pod CPU usage
kubectl top pods -A --sort-by=cpu

# 3. Check for restarts
kubectl get pods -A | awk '$5 > 0'

# 4. Check for probe failures
kubectl get events --all-namespaces --field-selector reason=Unhealthy

# 5. Check CPU limits
kubectl get pods -A -o json | jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name): \(.spec.containers[].resources.limits.cpu // "no-limit")"'

# 6. Describe problematic pod
kubectl describe pod <pod-name> -n <namespace>

# 7. Check node conditions
kubectl describe nodes | grep -A 5 "Conditions:"

# 8. Check cgroup version (on node)
mount | grep cgroup

# 9. Check throttling (cgroup v2 - EKS Ubuntu)
grep -r "throttled" /sys/fs/cgroup/kubepods.slice/*/cpu.stat 2>/dev/null | grep -v ":0$"

# 10. Check throttling (cgroup v1 - older systems)
cat /sys/fs/cgroup/cpu/kubepods/*/cpu.stat 2>/dev/null | grep throttled
```

## Mitigation Strategies

### Immediate Actions

1. **Increase probe timeouts:**
   - Change `timeoutSeconds` from 1s to 5-10s
   - Increase `failureThreshold` from 3 to 5

2. **Remove CPU limits:**
   - Keep CPU requests for scheduling
   - Remove CPU limits to prevent throttling

3. **Optimize high-CPU process:**
   - Review Defender/security agent configuration
   - Disable unnecessary features
   - Adjust scanning frequency

### Long-Term Solutions

1. **Right-size resource requests**
2. **Monitor CPU throttling metrics**
3. **Use startup probes for slow-starting apps**
4. **Implement proper resource quotas**
5. **Add node capacity if needed**

## References

- [Stop Using CPU Limits](https://home.robusta.dev/blog/stop-using-cpu-limits)
- [Managing Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [cgroup v2 Documentation](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)

## Skills Practiced

- Diagnosing CPU starvation and CFS throttling at node, pod, and container levels
- Inspecting cgroup v1/v2 `cpu.stat` throttling metrics on the node
- Correlating throttling and node saturation with probe failures and restarts
- Applying mitigations: probe tuning, removing CPU limits, and right-sizing requests
