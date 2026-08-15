# EKS Node Lifecycle During Template Updates

How nodes transition from Ready to terminated during a managed node group rolling update — the exact state progression, eviction mechanics, and ASG integration.

## Node State Progression

```
┌─────────────────────────────────────────────────────────────────────────┐
│  1. Ready                      Normal operation, accepting workloads    │
├─────────────────────────────────────────────────────────────────────────┤
│  2. Ready,SchedulingDisabled   Cordoned — no new pods, existing stay    │
├─────────────────────────────────────────────────────────────────────────┤
│  3. Ready,SchedulingDisabled   Draining — pods being evicted            │
├─────────────────────────────────────────────────────────────────────────┤
│  4. Ready,SchedulingDisabled   Nearly empty — only DaemonSets remain    │
├─────────────────────────────────────────────────────────────────────────┤
│  5. NotReady,SchedulingDisabled  Shutting down — kubelet stops          │
├─────────────────────────────────────────────────────────────────────────┤
│  6. (removed)                  Instance terminated, node object delete  │
├─────────────────────────────────────────────────────────────────────────┤
│  7. Ready (new node)           Replacement with updated template join   │
└─────────────────────────────────────────────────────────────────────────┘
```

## Detailed State Breakdown

### 1. Initial State: Ready

```
NAME                           STATUS   AGE   VERSION
ip-10-0-1-42.ec2.internal      Ready    5d    v1.30.0-eks-xxx
```

- Node is healthy and accepting workloads
- All pods running normally
- Part of the current launch template version

### 2. Cordon Phase: Ready,SchedulingDisabled

```
NAME                           STATUS                     AGE   VERSION
ip-10-0-1-42.ec2.internal      Ready,SchedulingDisabled   5d    v1.30.0-eks-xxx
```

- EKS marks the node as unschedulable
- Adds taint: `node.kubernetes.io/unschedulable:NoSchedule`
- Sets `spec.unschedulable: true`
- Existing pods continue running
- No new pods can be scheduled here

### 3. Drain Phase: Evicting Pods

While still showing `Ready,SchedulingDisabled`:

- EKS begins evicting pods via the Eviction API
- Pods receive SIGTERM (graceful termination signal)
- Respects `terminationGracePeriodSeconds` (default 30s)
- PodDisruptionBudgets (PDBs) are honored
- DaemonSet pods remain (they're ignored during drain)
- Pods reschedule to other nodes (or the new replacement node)

### 4. Nearly Empty

- Most application pods evicted
- Only DaemonSets and system pods remain
- Node is ready for termination

### 5. Termination: NotReady

```
NAME                           STATUS                        AGE   VERSION
ip-10-0-1-42.ec2.internal      NotReady,SchedulingDisabled   5d    v1.30.0-eks-xxx
```

- EC2 instance begins termination
- kubelet stops responding to the API server
- Node transitions to `NotReady`
- Eventually the node object is removed from the cluster

### 6. Replacement: New Node Ready

```
NAME                           STATUS   AGE   VERSION
ip-10-0-2-87.ec2.internal      Ready    2m    v1.31.0-eks-xxx
```

- New instance launched with updated launch template
- Bootstrap script configures kubelet
- Node joins cluster and becomes Ready
- Evicted pods get scheduled on this node

## Pod Eviction Mechanics

### Eviction API (Not Direct Deletion)

EKS uses the Kubernetes Eviction API, not `kubectl delete`:

```
POST /api/v1/namespaces/{namespace}/pods/{name}/eviction
```

This ensures:
- PodDisruptionBudgets are respected
- `terminationGracePeriodSeconds` is honored
- SIGTERM is sent before SIGKILL
- Pods get graceful shutdown time

### PDB Enforcement

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

If evicting a pod would violate the PDB, EKS waits until it's safe. This can cause the update to stall if PDBs are too restrictive.

### Drain Timeouts

EKS applies the equivalent of:

```sh
kubectl drain <node> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=300s \
  --grace-period=30
```

If drain can't complete within the timeout (PDB blocking, pods ignoring SIGTERM), EKS may force the update if `--force` was specified.

## Node Conditions During Transition

```yaml
# During cordon phase
status:
  conditions:
    - type: Ready
      status: "True"
      reason: KubeletReady
    - type: MemoryPressure
      status: "False"
    - type: DiskPressure
      status: "False"
    - type: PIDPressure
      status: "False"
    - type: NetworkUnavailable
      status: "False"
spec:
  unschedulable: true
  taints:
    - key: node.kubernetes.io/unschedulable
      effect: NoSchedule
```

## ASG Integration

### Managed Node Group Update Config

```yaml
updateConfig:
  maxUnavailable: 1              # At most 1 node updating at a time
  # OR
  maxUnavailablePercentage: 25   # At most 25% of nodes updating
```

### How EKS Coordinates with ASG

1. EKS increases ASG desired capacity (launches replacement first)
2. Waits for replacement node to become Ready
3. Cordons the old node
4. Drains the old node (evicts pods)
5. Terminates the old instance (decreases desired back)
6. Repeats for next node

This "surge" approach ensures capacity is always maintained.

### Instance Refresh (Self-Managed Nodes)

For self-managed ASGs (not EKS managed node groups), use instance refresh:

```sh
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name <asg-name> \
  --preferences '{
    "InstanceWarmup": 300,
    "MinHealthyPercentage": 50,
    "MaxHealthyPercentage": 200
  }'
```

> Instance refresh does NOT drain Kubernetes nodes. Combine with Node Termination Handler or lifecycle hooks for graceful pod eviction.

## Network and Storage During Updates

### ENI Management

- EKS detaches ENIs from the terminating instance
- VPC CNI handles IP address cleanup
- New nodes get fresh ENI allocation from the subnet
- Pod IPs on the old node become unreachable after termination

### EBS Volume Handling

PersistentVolumes backed by EBS:
- Are detached from the terminating node
- Reattach to the new node when the replacement pod is scheduled
- Must be in the same AZ as the new node

```sh
# Check PV zone affinity
kubectl get pv -o custom-columns=NAME:.metadata.name,ZONE:.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]
```

## Timing

### Node Bootstrap Sequence

```
1. EC2 instance launch            30-60s
2. kubelet starts and registers   10-30s
3. CNI plugin initializes         5-15s
4. Node becomes Ready             ─────────
                           Total: 45-105s
```

### Full Node Replacement Timeline

```
1. New node launched              ~1 min
2. New node Ready                 ~1-2 min
3. Old node cordoned              Immediate
4. Old node drained               30s - 5 min (depends on pods, PDBs)
5. Old instance terminated        ~30s
                           Total: ~3-8 minutes per node
```

With `maxUnavailable: 1` and 5 nodes, a full cluster update takes 15-40 minutes.

## Monitoring the Update

```sh
# Watch node states during update
kubectl get nodes -w

# Watch for scheduling and eviction events
kubectl get events -A --field-selector reason=Evicted --sort-by='.lastTimestamp'
kubectl get events --field-selector involvedObject.kind=Node -w

# Check node group update status
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng> \
  --query "nodegroup.{Status:status, Health:health.issues}" --output json

# Check which nodes are being drained
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].status,SCHED:.spec.unschedulable

# Watch pods rescheduling
kubectl get pods -A -o wide -w
```

## What Can Go Wrong

| Issue | Cause | Result |
|-------|-------|--------|
| Update stalls | PDB blocks eviction | Node stays in SchedulingDisabled indefinitely |
| Pod stays Pending | Not enough capacity on other nodes | Application degraded during update |
| Data loss | Pod using emptyDir without proper handling | Data lost when pod is evicted |
| Spot interruption during update | Replacement node is Spot and gets interrupted | Double replacement needed |
| New node can't join | Bootstrap failure on new template | Cluster loses capacity |

### Fixing a Stalled Update

```sh
# Check what's blocking
kubectl get pdb -A
kubectl describe pdb <pdb-name>

# If PDB is too restrictive, temporarily relax it
kubectl patch pdb <name> -p '{"spec":{"minAvailable":1}}'

# Or force the update (ignores PDBs — use with caution)
aws eks update-nodegroup-version --cluster-name <cluster> --nodegroup-name <ng> \
  --launch-template name=<lt>,version=<new-version> --force
```

## Best Practices

- **Always set PDBs** — but don't make them too restrictive or updates stall
- **Use `maxUnavailable: 1`** for production — safer but slower
- **Use `maxUnavailablePercentage: 33`** for large node groups — faster updates
- **Set `terminationGracePeriodSeconds`** appropriately — long enough for graceful shutdown, short enough not to block updates
- **Monitor during updates** — watch `kubectl get nodes -w` and events
- **Test new launch templates** in dev first — a bad AMI or userdata breaks all replacement nodes
- **Keep StatefulSet pods in mind** — they need time to detach volumes and reschedule in the correct AZ
