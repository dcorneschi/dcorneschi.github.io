# What Happens When a Kubernetes Node Goes NotReady

The internal mechanics of node health monitoring — how the kubelet reports status, how the node lifecycle controller detects failures, applies taints, and triggers pod eviction.

Note: For EKS-specific NotReady troubleshooting (I/O spikes, CPU starvation, investigation commands), see the EKS node NotReady guide. This article covers the generic Kubernetes internals.

## High-Level Flow

```
┌──────────┐     ┌───────────────┐     ┌──────────────────────┐     ┌─────────────┐
│  Kubelet │────▶│   API Server  │────▶│  Node Lifecycle      │────▶│  Pod        │
│  (heart- │     │  (Lease +     │     │  Controller          │     │  Eviction   │
│   beat)  │     │   NodeStatus) │     │  (taint manager)     │     │             │
└──────────┘     └───────────────┘     └──────────────────────┘     └─────────────┘
```

## Node Health Reporting — Two Mechanisms

The kubelet reports node health via two separate mechanisms:

### 1. Node Lease (Lightweight Heartbeat)

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: node-1              # Same as node name
  namespace: kube-node-lease
spec:
  holderIdentity: node-1
  leaseDurationSeconds: 40  # Default
  renewTime: "2024-03-15T10:00:30Z"  # Updated every 10s
```

- Kubelet renews the Lease every **10 seconds** (`nodeStatusUpdateFrequency`)
- The Lease object is tiny (no conditions, no resource usage) — cheap to update
- Only contains `renewTime` — a timestamp of last successful renewal
- This is the **primary** heartbeat mechanism since Kubernetes 1.17+

```bash
# Check a node's lease:
kubectl get lease -n kube-node-lease <node-name> -o yaml

# See renewal time:
kubectl get lease -n kube-node-lease <node-name> -o jsonpath='{.spec.renewTime}'
```

### 2. NodeStatus (Full Status Report)

```yaml
status:
  conditions:
  - type: Ready
    status: "True"
    lastHeartbeatTime: "2024-03-15T10:00:30Z"
    lastTransitionTime: "2024-03-10T08:00:00Z"
    reason: KubeletReady
    message: "kubelet is posting ready status"
  - type: MemoryPressure
    status: "False"
  - type: DiskPressure
    status: "False"
  - type: PIDPressure
    status: "False"
  - type: NetworkUnavailable
    status: "False"
```

- Kubelet updates NodeStatus every **5 minutes** (or on condition change)
- Contains full conditions, allocatable resources, node info, images
- Heavier update — avoided for frequent heartbeats

```bash
# Check node conditions:
kubectl get node <name> -o jsonpath='{.status.conditions}' | jq .

# Quick ready check:
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].status,REASON:.status.conditions[-1].reason
```

## The Node Lifecycle Controller

The `node-lifecycle-controller` (inside kube-controller-manager) monitors nodes and reacts to failures:

```
┌────────────────────────────────────────────────────────────────┐
│  Node Lifecycle Controller                                     │
│                                                                │
│  Every 5s (--node-monitor-period):                             │
│    for each node:                                              │
│   1. Check Lease renewTime                                     │
│   2. If renewTime older than 40s (--node-monitor-grace-period):│
│         → Set node condition Ready=Unknown                     │
│         → Apply taint: node.kubernetes.io/unreachable:NoExecute│
│                                                                │
│   3. If NodeStatus shows Ready=False:                          │
│         → Apply taint: node.kubernetes.io/not-ready:NoExecute  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--node-monitor-period` | 5s | How often the controller checks node health |
| `--node-monitor-grace-period` | 40s | How long to wait before marking Unknown |
| `--pod-eviction-timeout` | 5m | How long to wait before evicting pods (deprecated, replaced by taints) |

## Timeline: Node Becomes Unreachable

```
Time ──────────────────────────────────────────────────────────────────▶

Kubelet              API Server           Node Lifecycle Ctrl    Pods
   │                    │                       │                 │
   │ Lease renew (OK) ─▶│                       │                 │
   │ .................. │                       │                 │
   │                    │                       │                 │
   │ ╳ Node crashes or  │                       │                 │
   │network partitions  │                       │                 │
   │(no more heartbeats)│                       │                 │
   │                    │                       │                 │
   │         +10s       │  (Lease not renewed)  │                 │
   │         +20s       │  (Lease not renewed)  │                 │
   │         +30s       │  (Lease not renewed)  │                 │
   │         +40s       │                       │                 │
   │                    │  ◀── check ───────────│                 │
   │                    │     Lease expired!    │                 │
   │                    │                       │                 │
   │                    │  Set Ready=Unknown ◀──│                 │
   │                    │  Apply taint:         │                 │
   │                    │  unreachable:NoExecute│                 │
   │                    │                       │                 │
   │         +5m40s     │                       │                 │
   │                    │                       │ toleration      │
   │                    │                       │ timeout (300s)  │
   │                    │                       │ expires         │
   │                    │                       │ ───────────────▶│
   │                    │                       │                 │ Pods evicted
   │                    │                       │                 │ (rescheduled)
```

## Node Conditions and Taints

| Node Condition | Taint Applied | Effect |
|---------------|---------------|--------|
| `Ready=Unknown` (heartbeat timeout) | `node.kubernetes.io/unreachable:NoExecute` | Pods evicted after tolerationSeconds |
| `Ready=False` (kubelet reports unhealthy) | `node.kubernetes.io/not-ready:NoExecute` | Pods evicted after tolerationSeconds |
| `MemoryPressure=True` | `node.kubernetes.io/memory-pressure:NoSchedule` | No new pods scheduled |
| `DiskPressure=True` | `node.kubernetes.io/disk-pressure:NoSchedule` | No new pods scheduled |
| `PIDPressure=True` | `node.kubernetes.io/pid-pressure:NoSchedule` | No new pods scheduled |
| `NetworkUnavailable=True` | `node.kubernetes.io/network-unavailable:NoSchedule` | No new pods scheduled |

The difference between `unreachable` and `not-ready`:
- **unreachable**: Lease expired — we don't know the node's actual state (network partition, node crash)
- **not-ready**: Kubelet actively reported it's unhealthy (e.g., container runtime down)

## Pod Eviction via Taint-Based Mechanism

When a NoExecute taint is applied, the taint manager evicts pods that don't tolerate it:

### Default Tolerations (Added Automatically)

The default admission controller adds these tolerations to every pod:

```yaml
tolerations:
- key: node.kubernetes.io/not-ready
  operator: Exists
  effect: NoExecute
  tolerationSeconds: 300    # 5 minutes
- key: node.kubernetes.io/unreachable
  operator: Exists
  effect: NoExecute
  tolerationSeconds: 300    # 5 minutes
```

This means: after a node goes NotReady/Unreachable, pods stay for 5 minutes before eviction. This allows time for transient issues to recover.

### What "Eviction" Means Here

For `unreachable` nodes, the API server can't communicate with the kubelet, so:

1. API server sets `deletionTimestamp` on the pod
2. Pod status becomes `Terminating`
3. If the node is truly gone — kubelet never confirms deletion
4. Pod stays in `Terminating` until:
   - Node comes back and kubelet cleans up
   - Admin force-deletes the pod
   - Node object is deleted from the cluster

```bash
# Find pods stuck Terminating on unreachable nodes:
kubectl get pods -A --field-selector status.phase=Running -o json | \
  jq -r '.items[] | select(.metadata.deletionTimestamp != null) | 
  "\(.metadata.namespace)/\(.metadata.name) on \(.spec.nodeName)"'
```

### StatefulSet Pods — Special Behavior

StatefulSet pods have stable identity. When a node becomes unreachable:
- The StatefulSet controller does NOT create a replacement pod immediately
- It waits until the old pod is confirmed gone (force deleted or node removed)
- This prevents split-brain (two pods with same identity + same PVC)

```bash
# Force delete a stuck StatefulSet pod (after confirming node is gone):
kubectl delete pod <name> --grace-period=0 --force
```

## Node Recovery

When the node comes back online:

```
Node recovers → Kubelet starts → Lease renewed → Controller removes taints
     │
     ├── Pods that were only marked Terminating: kubelet kills them
     ├── Taints removed: new pods can be scheduled
     └── Previously evicted pods: already rescheduled elsewhere (won't come back)
```

```bash
# Check if taints were removed after recovery:
kubectl describe node <name> | grep Taints
```

## Large Cluster Considerations

### Rate-Limited Eviction

In large clusters, the node lifecycle controller rate-limits evictions to prevent cascading failures:

```
┌────────────────────────────────────────────────────────┐
│  Eviction Rate Limiting                                │
│                                                        │
│  --unhealthy-zone-threshold: 0.55 (default)            │
│  --large-cluster-size-threshold: 50 (default)          │
│                                                        │
│  If > 55% of nodes in a zone are unhealthy:            │
│    → Eviction rate reduced to secondary rate           │
│    → Prevents mass eviction during zone failures       │
│                                                        │
│  If cluster has < 50 nodes AND > 55% unhealthy:        │
│    → Evictions STOP entirely                           │
│    → Assumes cluster-wide issue, not node issue        │
└────────────────────────────────────────────────────────┘
```

| Scenario | Eviction Behavior |
|----------|-------------------|
| Single node failure (large cluster) | Normal rate — pods evicted after tolerationSeconds |
| Multiple nodes in one zone (>55% unhealthy) | Reduced rate — slower eviction |
| Small cluster with majority nodes failing | Evictions paused — assumes systemic issue |

### Zone-Aware Eviction

If the cluster spans multiple zones:
- Each zone's health is evaluated independently
- A healthy zone evicts at normal rate
- An unhealthy zone (>55% NotReady) evicts at reduced rate
- This prevents a network partition from emptying an entire zone

## Kubelet Reasons for NotReady

Common reasons the kubelet reports `Ready=False`:

| Reason | Meaning |
|--------|---------|
| `KubeletNotReady` | Container runtime not responding |
| `KubeletReady` → `KubeletNotReady` | Runtime crashed after being healthy |
| `NetworkPluginNotReady` | CNI plugin failed or not configured |
| `PLEG is not healthy` | Pod Lifecycle Event Generator stuck |
| `ContainerRuntimeNotReady` | containerd/CRI-O down |

```bash
# Check kubelet logs for the cause:
journalctl -u kubelet --since "10 minutes ago" | grep -i "not ready\|error\|failed"

# Check container runtime:
systemctl status containerd
crictl info
```

## Monitoring Node Health

```bash
# List node conditions:
kubectl get nodes -o custom-columns=\
  NAME:.metadata.name,\
  READY:.status.conditions[?(@.type==\"Ready\")].status,\
  SINCE:.status.conditions[?(@.type==\"Ready\")].lastTransitionTime

# Find NotReady nodes:
kubectl get nodes --field-selector status.conditions.type=Ready,status.conditions.status!=True

# Check lease freshness:
kubectl get leases -n kube-node-lease -o custom-columns=\
  NODE:.metadata.name,\
  RENEW:.spec.renewTime

# Watch node status changes:
kubectl get nodes -w

# Events related to node issues:
kubectl get events --field-selector reason=NodeNotReady
kubectl get events --field-selector reason=NodeReady
```

## Quick Reference

```bash
# Node heartbeat mechanism:
# - Lease in kube-node-lease namespace (every 10s)
# - NodeStatus conditions (every 5m or on change)

# Grace period before marking Unknown:
# - 40s (--node-monitor-grace-period)

# Default pod tolerance before eviction:
# - 300s (tolerationSeconds on not-ready/unreachable taints)

# Total time from node failure to pod rescheduling:
# - ~5m 40s (40s detection + 300s toleration)

# Check node lease
kubectl get lease -n kube-node-lease <node>

# Check node conditions
kubectl get node <node> -o jsonpath='{.status.conditions[?(@.type=="Ready")]}'

# Check taints applied
kubectl describe node <node> | grep -A5 Taints

# Find pods affected by NotReady node
kubectl get pods -A --field-selector spec.nodeName=<node>

# Force delete stuck Terminating pods (node is gone permanently)
kubectl delete pod <name> --grace-period=0 --force
```
