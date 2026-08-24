# Kubernetes Resource Scheduling & Node Capacity Guide

## Finding a DaemonSet Pod Stuck in Pending

A DaemonSet pod in `Pending` means there's a node it should be running on but can't schedule to.

### Identify the cause

```bash
kubectl describe pod <pending-pod-name> -n <namespace>
```

Look in the `Events` section for `FailedScheduling`. It will list the node names and the reason, e.g.:

```
0/3 nodes are available: 1 node(s) had insufficient cpu, 2 node(s) didn't match pod affinity rules
```

### Find all pending pods

```bash
kubectl get pods -n <namespace> --field-selector=status.phase=Pending -o wide
```

> Note: `-o wide` shows the `NODE` column, but for Pending pods it will show `<none>` since no node has been assigned yet. The node information comes from the `describe` events and the diff method below.

### Check resource pressure across nodes

```bash
kubectl top nodes
```

This shows actual CPU/memory usage vs capacity, helping you spot which node is maxed out.

### Find which node is missing a DaemonSet pod

Since DaemonSets target every node (or a subset via node selectors/affinity), a Pending pod means there's a specific node it *should* be running on but can't. The node information comes from the `describe` events and by comparing nodes against running pods.

Compare all nodes against nodes already running the DaemonSet:

```bash
# All nodes
kubectl get nodes -o name

# Nodes running the DaemonSet
kubectl get pods -n <namespace> -l <daemonset-label-selector> -o wide --field-selector=status.phase=Running
```

Or as a one-liner diff:

```bash
diff <(kubectl get nodes -o name | sort) \
     <(kubectl get pods -n <namespace> -l app=<daemonset-app> -o jsonpath='{range .items[?(@.status.phase=="Running")]}{.spec.nodeName}{"\n"}{end}' | sort)
```

Lines prefixed with `<` are nodes without a running pod from that DaemonSet.

### Typical fixes for a Pending DaemonSet pod

- Reduce the `resources.requests` on the DaemonSet
- Evict or scale down other workloads on the target node
- Add more capacity to the node (resize the VM/instance)
- Remove taints or adjust tolerations if the DaemonSet is being excluded

## How "Free" Resources Are Calculated on a Node

Kubernetes doesn't track real-time "free" resources for scheduling. It uses:

```
free = allocatable - sum(all pod requests)
```

- **Allocatable**: total node capacity minus system reserved (kubelet, OS, etc.)
- **Requests**: what each pod declares it needs

### Default behavior when no requests are set

If a pod doesn't specify `resources.requests`, Kubernetes treats it as requesting **0 CPU and 0 memory**:

- The scheduler thinks it needs zero resources, so it schedules it anywhere
- It won't show as consuming any allocatable capacity
- But it will use real resources at runtime, potentially starving other pods

This is why a DaemonSet with no resource requests will almost never be `Pending` due to resources — the scheduler sees it as "free." If your DaemonSet pod is Pending, it likely has explicit requests set that exceed what's available on the target node.

### Node capacity breakdown

A node's resources are split into layers:

```
Total Node Capacity
  - kube-reserved      (resources reserved for kubelet, container runtime)
  - system-reserved    (resources reserved for OS processes)
  - eviction-threshold (memory buffer before eviction kicks in)
  = Allocatable        (what's available for pod scheduling)
```

### View allocated resources on a node

```bash
kubectl describe node <node-name>
```

Look for the `Allocated resources` section:

```
Allocated resources:
  Resource           Requests     Limits
  --------           --------     ------
  cpu                1200m (60%)  2000m (100%)
  memory             512Mi (25%)  1Gi (50%)
```

Across all nodes:

```bash
kubectl describe nodes | grep -A 6 "Allocated resources"
```

### Check actual usage vs capacity

```bash
kubectl top nodes
```

### View DaemonSet resource requests

```bash
kubectl get daemonset <name> -n <namespace> -o jsonpath='{.spec.template.spec.containers[*].resources}'
```

### View node allocatable resources

```bash
kubectl get nodes -o custom-columns="NAME:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory"
```

### Set default requests/limits with LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-resources
  namespace: <namespace>
spec:
  limits:
    - default:
        cpu: 500m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      type: Container
```

Any container without explicit requests/limits gets these defaults applied automatically.

## CPU vs Memory: Compressible vs Incompressible Resources

Understanding the difference is key to how Kubernetes handles resource pressure:

| Resource | Type            | Under pressure                          | Consequence          |
|----------|-----------------|-----------------------------------------|----------------------|
| CPU      | Compressible    | Pods get throttled (CFS quota)          | Slower, not killed   |
| Memory   | Incompressible  | Pods get OOM-killed by the kernel       | Pod restarts/evicted |

This is why memory limits are more critical than CPU limits — exceeding memory can crash your pod, while exceeding CPU just slows it down.

## Will a Node Fill Up to 99-100%?

Yes, the scheduler will allow requests to fill up to 100% of allocatable, and actual usage can exceed that.

### Scheduling (requests)

The scheduler places pods on a node until the sum of **requests** reaches 100% of allocatable. It will not over-commit on requests.

Example: if a node has 2 CPU allocatable and existing pods request 1.8 CPU, a new pod requesting 300m won't be scheduled there.

### Runtime usage (limits)

The sum of **limits** across all pods on a node **can exceed** 100% of capacity — Kubernetes allows over-commit on limits.

- **CPU** (compressible): pods can burst up to their limit if spare CPU exists. Under contention, pods get throttled proportionally to their requests. Nothing gets killed, just slowed down.
- **Memory** (incompressible): pods can use up to their limit. If the node runs out of actual memory, the kubelet evicts pods (OOM kills), starting with pods that exceed their requests the most, based on QoS class priority:
  1. `BestEffort` (no requests/limits) — evicted first
  2. `Burstable` (requests < limits) — evicted second
  3. `Guaranteed` (requests == limits) — evicted last

### Kubelet eviction thresholds

The kubelet has built-in soft/hard eviction thresholds (defaults):

| Resource           | Default Threshold |
|--------------------|-------------------|
| memory.available   | < 100Mi           |
| nodefs.available   | < 10%             |
| imagefs.available  | < 15%             |

When actual usage hits these, the kubelet starts evicting pods before the node fully crashes. So it won't truly let things run to 100% of physical resources — the eviction thresholds act as a safety buffer.

There are two types of eviction:
- **Soft eviction**: kubelet waits a grace period before evicting (configurable)
- **Hard eviction**: kubelet evicts immediately with no grace period (the defaults above are hard thresholds)

### Over-commit scenario example

Consider a node with 4 CPU allocatable:

```
Pod A: requests 1 CPU, limits 2 CPU
Pod B: requests 1 CPU, limits 2 CPU
Pod C: requests 1 CPU, limits 2 CPU
Pod D: requests 1 CPU, limits 2 CPU
```

- Total requests = 4 CPU (100% of allocatable) → scheduler won't place more pods
- Total limits = 8 CPU (200% of allocatable) → all pods CAN try to use 2 CPU each
- If all pods burst at once, they get throttled back proportionally to their requests (1 CPU each)

### Best practice

Always set both **requests** and **limits**:

- **Requests** → accurate scheduling, prevents over-packing nodes
- **Limits** → runtime protection, prevents a single pod from consuming all node resources

### In practice

- If you don't set **limits**, pods can consume all available node resources with no cap
- If you don't set **requests**, the scheduler thinks everything is free and will over-pack nodes
- That's why setting both is important — requests for scheduling accuracy, limits for runtime protection

## DaemonSet Restart: What If It Doesn't Fit Anymore?

A common scenario: a DaemonSet pod gets evicted or restarted, and the node has filled up with other workloads so it can't schedule back.

### Priority Classes (the main solution)

Give your DaemonSet a high `PriorityClass` so the scheduler will preempt (evict) lower-priority pods to make room:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: daemonset-critical
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "High priority for DaemonSet workloads"
```

Then in your DaemonSet spec:

```yaml
spec:
  template:
    spec:
      priorityClassName: daemonset-critical
```

When the DaemonSet pod needs to reschedule, Kubernetes will evict lower-priority pods to free up space. The evicted pods then get rescheduled elsewhere (or wait in Pending).

Kubernetes also has two built-in priority classes you can use directly:

- `system-node-critical` (value 2000001000) — for things like kube-proxy, CNI
- `system-cluster-critical` (value 2000000000) — for cluster-essential workloads

If your DaemonSet is truly essential (monitoring, logging, networking), using one of these is reasonable.

### Built-in Priority Classes in detail

These are created automatically in every cluster — you don't need to define them.

**system-node-critical** (value 2000001000):
- For pods that are absolutely essential to the node functioning
- Used by things like kube-proxy, CNI plugins (Calico, Flannel), CSI drivers
- If these don't run, the node basically can't work (no networking, no storage)
- These pods will preempt almost anything else on the node

**system-cluster-critical** (value 2000000000):
- For pods essential to the cluster but not to a specific node
- Used by things like CoreDNS, kube-dns, metrics-server
- Slightly lower priority than system-node-critical
- If these don't run, cluster-wide services break but individual nodes still function

The priority values determine eviction order — higher value = higher priority = less likely to be evicted:

```
system-node-critical    (2000001000)  ← highest, almost never evicted
system-cluster-critical (2000000000)  ← very high
your custom class       (1000000)     ← high
default                 (0)           ← normal pods with no class set
```

You don't need to create these — just reference them directly:

```yaml
spec:
  template:
    spec:
      priorityClassName: system-node-critical
```

Use `system-node-critical` for DaemonSets that are truly node-essential (monitoring agents, log collectors, network plugins). Use `system-cluster-critical` for cluster-wide services. For your own app DaemonSets, creating a custom PriorityClass with a lower value (like 1000000) is more appropriate — you don't want your app preempting kube-proxy.

### Right-size your requests

Don't over-request. If your DaemonSet only needs 50m CPU and 64Mi memory, don't request 500m/256Mi "just in case." Smaller requests = easier to fit back in.

### Resource budgets — leave headroom on nodes

Don't schedule nodes to 100%. You can do this with:

- A placeholder pod with low priority that "reserves" space and gets evicted when real workloads need it
- Or by setting kubelet `--system-reserved` / `--kube-reserved` higher than strictly needed (see section below)

### Pod Disruption Budgets (PDBs)

Won't directly help the DaemonSet reschedule, but they protect your other workloads from being mass-evicted when preemption kicks in:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: my-app
```

## Kubelet Reserved Resources: --kube-reserved and --system-reserved

These are kubelet flags that carve out resources from the node's total capacity before calculating `allocatable`. They reduce what the scheduler can use, effectively reserving headroom.

### --kube-reserved

Reserves resources for Kubernetes system daemons (kubelet, container runtime):

```
--kube-reserved=cpu=200m,memory=256Mi,ephemeral-storage=1Gi
```

### --system-reserved

Reserves resources for OS-level processes (sshd, systemd, journald, etc.):

```
--system-reserved=cpu=200m,memory=256Mi,ephemeral-storage=1Gi
```

### The math

```
Allocatable = Capacity - kube-reserved - system-reserved - eviction-threshold
```

Example for a node with 4 CPU / 8Gi memory:

```
Capacity:           4 CPU,    8Gi memory
kube-reserved:      200m,     256Mi
system-reserved:    200m,     256Mi
eviction-threshold:   0,      100Mi
                    ----      -----
Allocatable:        3600m,    7430Mi
```

The scheduler will only pack pods up to 3600m CPU / 7430Mi memory, leaving the rest as buffer.

### Configuration by distribution

**kubeadm** — in the `KubeletConfiguration`:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
kubeReserved:
  cpu: "200m"
  memory: "256Mi"
  ephemeral-storage: "1Gi"
systemReserved:
  cpu: "200m"
  memory: "256Mi"
  ephemeral-storage: "1Gi"
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"
enforceNodeAllocatable:
  - pods
  - kube-reserved
  - system-reserved
```

**k3s** — pass as kubelet args:

```yaml
# In k3s config (e.g., /etc/rancher/k3s/config.yaml)
kubelet-arg:
  - "kube-reserved=cpu=200m,memory=256Mi"
  - "system-reserved=cpu=200m,memory=256Mi"
```

**microk8s**:

```bash
# Edit /var/snap/microk8s/current/args/kubelet
# Add:
--kube-reserved=cpu=200m,memory=256Mi
--system-reserved=cpu=200m,memory=256Mi
```

### enforceNodeAllocatable

Without this, the reservations only affect scheduling math but aren't enforced via cgroups. With it enabled, processes that exceed their reservation actually get throttled/killed.

Options:

- `pods` — enforce limits on pod cgroups (default, always recommended)
- `kube-reserved` — enforce on kube system daemons
- `system-reserved` — enforce on OS processes

### The headroom trick for DaemonSets

If you set these values slightly higher than what the system actually needs, the extra space acts as a buffer. When a DaemonSet pod restarts, there's room on the node because the scheduler never allocated that reserved space to regular pods.

For example, if your system daemons only need 100m CPU but you reserve 300m, that extra 200m is always available for a DaemonSet to squeeze back in. It's not the cleanest approach (PriorityClass is better), but it works as an extra safety net.
