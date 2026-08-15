# Fix DaemonSet Scheduling Issues on EKS Nodes

## Problem

Multiple pods have been scheduled on an EKS node before the DaemonSet, consuming available CPU resources and preventing the DaemonSet pod from being scheduled.

## Diagnosis Commands

### Check Node Resource Usage

```sh
# Check overall node resource usage
kubectl top nodes

# Get detailed node information
kubectl describe nodes

# Check resource allocation per node
kubectl describe node <node-name>
```

### Identify Pending DaemonSet Pods

```sh
# Find pending pods
kubectl get pods --all-namespaces --field-selector=status.phase=Pending

# Get detailed information about pending DaemonSet pod
kubectl describe pod <pending-daemonset-pod> -n <namespace>
```

### Check Current Pod Resource Allocation

```sh
# See which pods are consuming resources on the problematic node
kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=<node-name>

# Check resource requests for pods on the node
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node-name> \
  -o custom-columns=NAME:.metadata.name,NAMESPACE:.metadata.namespace,CPU_REQUEST:.spec.containers[*].resources.requests.cpu,MEMORY_REQUEST:.spec.containers[*].resources.requests.memory
```

## Solutions

### Solution 1: Use Priority Classes (Recommended)

DaemonSets should have high priority to ensure they can preempt lower-priority pods when needed.

Create a high-priority class:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: daemonset-priority
value: 1000000
globalDefault: false
description: "High priority for DaemonSets to ensure they always run"
```

Apply the priority class to your DaemonSet:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-daemonset
spec:
  selector:
    matchLabels:
      app: my-daemonset
  template:
    metadata:
      labels:
        app: my-daemonset
    spec:
      priorityClassName: daemonset-priority
      containers:
        - name: agent
          image: my-agent:latest
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 512Mi
```

With a higher priority, the scheduler will preempt lower-priority pods to make room for the DaemonSet.

### Solution 2: Evict Low-Priority Pods

```sh
# Evict specific pods to free up resources
kubectl delete pod <pod-name> -n <namespace>

# Or drain and re-cordon (keeps DaemonSets running)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --force --grace-period=30
kubectl uncordon <node-name>
```

### Solution 3: Adjust Resource Requests

Temporarily reduce resource requests for non-critical pods:

```sh
# Edit deployment to reduce CPU requests
kubectl edit deployment <deployment-name> -n <namespace>

# Or scale down deployments temporarily
kubectl scale deployment <deployment-name> --replicas=0 -n <namespace>
```

### Solution 4: Configure DaemonSet with Appropriate Resources

Ensure your DaemonSet has appropriate (and minimal) resource requests:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-daemonset
spec:
  selector:
    matchLabels:
      app: my-daemonset
  template:
    metadata:
      labels:
        app: my-daemonset
    spec:
      containers:
        - name: agent
          image: my-agent:latest
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

Keep DaemonSet resource requests small — they run on every node and compete with workload pods.

### Solution 5: Reserve Resources for System Pods (Node Allocatable)

Configure kubelet to reserve resources for system components, preventing workload pods from consuming everything:

```sh
# kubelet args (in bootstrap script or kubelet config)
--system-reserved=cpu=200m,memory=256Mi
--kube-reserved=cpu=200m,memory=512Mi
--eviction-hard=memory.available<100Mi
```

Or in kubelet config:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
systemReserved:
  cpu: 200m
  memory: 256Mi
kubeReserved:
  cpu: 200m
  memory: 512Mi
evictionHard:
  memory.available: "100Mi"
```

This ensures the scheduler doesn't allocate 100% of node capacity to workload pods.

### Solution 6: Enable Cluster Autoscaling

If this is a recurring issue, the cluster needs more capacity:

```sh
# Check if cluster autoscaler is running
kubectl get deployment cluster-autoscaler -n kube-system

# Scale up the node group manually
aws eks update-nodegroup-config --cluster-name <cluster> --nodegroup-name <ng> \
  --scaling-config minSize=3,maxSize=10,desiredSize=5
```

## Quick Fix Steps

```sh
# 1. Find pending DaemonSet pods
kubectl get pods --all-namespaces --field-selector=status.phase=Pending | grep -i daemonset

# 2. Check why it's pending
kubectl describe pod <pending-pod> -n <namespace> | grep -A 10 "Events"
# Look for: "Insufficient cpu" or "Insufficient memory"

# 3. Find which node it should be on
kubectl describe pod <pending-pod> -n <namespace> | grep "Node:"

# 4. See what's consuming resources on that node
kubectl describe node <node-name> | grep -A 20 "Allocated resources"

# 5. Free up resources — delete or scale down non-critical pods
kubectl delete pod <low-priority-pod> -n <namespace>
# OR
kubectl scale deployment <non-critical-deployment> --replicas=0 -n <namespace>

# 6. Verify DaemonSet pod gets scheduled
kubectl get pods -n <namespace> -l app=<daemonset-label> -o wide
```

## Why This Happens

DaemonSets are special in Kubernetes scheduling:

1. The DaemonSet controller creates a pod for each node
2. If the node's resources are already fully committed (sum of requests = allocatable), the DaemonSet pod can't be scheduled
3. Unlike regular pods, DaemonSet pods don't trigger the Cluster Autoscaler (they need to run on existing nodes, not new ones)
4. Without a PriorityClass, DaemonSet pods get default priority (0) — same as all other workloads

The root cause is usually:
- Workload pods with inflated resource requests consuming all allocatable capacity
- No PriorityClass on the DaemonSet (can't preempt)
- No `kube-reserved` / `system-reserved` configured on the node

## Prevention Strategies

1. **Always use PriorityClasses for DaemonSets** — set higher than workload pods so they can preempt
2. **Set appropriate resource requests on all pods** — don't over-request
3. **Configure node allocatable** — reserve CPU/memory for system components
4. **Monitor node resource usage** — alert when nodes approach full request capacity
5. **Use Pod Disruption Budgets** — for critical applications to survive preemption gracefully
6. **Enable cluster autoscaling** — for dynamic capacity growth
7. **Use `system-node-critical` or `system-cluster-critical`** for infrastructure DaemonSets (monitoring, logging, CNI)

```sh
# System-critical DaemonSets (highest priority, can't be preempted)
spec:
  priorityClassName: system-node-critical
```

## Monitoring

```sh
# Regular monitoring to prevent this issue
kubectl top nodes
kubectl get pods --all-namespaces --field-selector=status.phase=Pending
kubectl describe nodes | grep -A 5 "Allocated resources"

# One-liner: nodes with high request utilization
kubectl describe nodes | grep -B 5 "cpu.*[89][0-9]%"
```
