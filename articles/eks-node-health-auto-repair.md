# Node Health Monitoring and Auto-Repair for Amazon EKS

How EKS detects unhealthy nodes and replaces them — covering EC2 health checks, Kubernetes node conditions, managed node group auto-repair, Node Termination Handler, Karpenter disruption, and Node Problem Detector.

## The Health Check Layers

EKS nodes are monitored at multiple layers, each with different detection and repair mechanisms:

```
┌──────────────────────────────────────────────────────────────────────┐
│  Layer 1: EC2 Status Checks (AWS infrastructure)                     │
│  → Detects: hardware failure, hypervisor issues, network loss        │
│  → Action: instance marked impaired, can trigger auto-recovery       │
├──────────────────────────────────────────────────────────────────────┤
│  Layer 2: Kubernetes Node Conditions (kubelet)                       │
│  → Detects: memory pressure, disk pressure, PID pressure, not ready  │
│  → Action: pod eviction, no new scheduling                           │
├──────────────────────────────────────────────────────────────────────┤
│  Layer 3: Node Problem Detector (optional DaemonSet)                 │
│  → Detects: kernel issues, filesystem corruption, runtime crashes    │
│  → Action: reports conditions, triggers custom remediation           │
├──────────────────────────────────────────────────────────────────────┤
│  Layer 4: EKS Managed Node Auto-Repair (built-in)                    │
│  → Detects: nodes NotReady for extended period                       │
│  → Action: terminates and replaces the instance                      │
├──────────────────────────────────────────────────────────────────────┤
│  Layer 5: Karpenter Disruption (if using Karpenter)                  │
│  → Detects: drift, underutilization, expiry, spot interruption       │
│  → Action: cordons, drains, and replaces nodes                       │
├──────────────────────────────────────────────────────────────────────┤
│  Layer 6: Node Termination Handler (if using self-managed)           │
│  → Detects: spot interruption, scheduled maintenance, rebalance      │
│  → Action: cordons and drains before termination                     │
└──────────────────────────────────────────────────────────────────────┘
```

## Layer 1: EC2 Status Checks

AWS performs two checks every minute on every instance:

| Check | Detects | Who Fixes |
|-------|---------|-----------|
| System Status Check | Hardware failure, hypervisor issues, power loss | AWS (or stop/start the instance) |
| Instance Status Check | OS issues, exhausted memory, corrupted filesystem, bad kernel | You (reboot or replace) |

### EC2 Auto-Recovery

For non-EKS instances, you can set up CloudWatch alarms to trigger auto-recovery. For EKS managed node groups, the managed auto-repair handles this instead.

```sh
# Check EC2 status for all cluster nodes
for id in $(kubectl get nodes -o jsonpath='{.items[*].spec.providerID}' | tr ' ' '\n' | awk -F/ '{print $NF}'); do
  status=$(aws ec2 describe-instance-status --instance-ids $id \
    --query "InstanceStatuses[0].{System:SystemStatus.Status, Instance:InstanceStatus.Status}" --output text 2>/dev/null)
  echo "$id: $status"
done
```

## Layer 2: Kubernetes Node Conditions

kubelet continuously evaluates and reports node conditions:

| Condition | Healthy | Unhealthy Meaning |
|-----------|:-------:|-------------------|
| `Ready` | True | kubelet is not healthy, cannot accept pods |
| `MemoryPressure` | False | Node running low on memory |
| `DiskPressure` | False | Node running low on disk |
| `PIDPressure` | False | Too many processes on the node |
| `NetworkUnavailable` | False | CNI not configured correctly |

When conditions change:

- `Ready=False` → no new pods scheduled, existing pods may be evicted after `pod-eviction-timeout` (default 5 minutes)
- `MemoryPressure=True` → kubelet evicts BestEffort pods, then Burstable
- `DiskPressure=True` → kubelet garbage collects images and dead containers

```sh
# Find nodes with issues
kubectl get nodes -o custom-columns=NAME:.metadata.name,READY:.status.conditions[-1].status,MEM:.status.conditions[0].status,DISK:.status.conditions[1].status

# Detailed conditions
kubectl describe node <node-name> | grep -A 20 "Conditions:"
```

### Node Lease and Heartbeat

kubelet sends a heartbeat (node lease) to the API server every 10 seconds (default). If the API server doesn't receive a heartbeat for 40 seconds, it marks the node as `Unknown`. After `pod-eviction-timeout`, pods are evicted.

```sh
# Check node leases
kubectl get lease -n kube-node-lease

# Check when a node was last seen
kubectl get node <node-name> -o jsonpath='{.status.conditions[?(@.type=="Ready")].lastHeartbeatTime}'
```

## Layer 3: Node Problem Detector

[Node Problem Detector](https://github.com/kubernetes/node-problem-detector) detects issues invisible to kubelet's built-in checks.

### What It Detects

- Kernel deadlocks (`KernelDeadlock`)
- Read-only filesystem (`ReadonlyFilesystem`)
- Corrupted container runtime (`FrequentContainerdRestart`)
- OOM kills (kernel-level)
- NTP clock drift
- Hardware errors (MCE)
- Unresponsive kubelet (`FrequentKubeletRestart`)
- Conntrack table full

### Installation

```sh
# Helm
helm repo add deliveryhero https://charts.deliveryhero.io
helm install node-problem-detector deliveryhero/node-problem-detector -n kube-system

# Or kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes/node-problem-detector/master/deployment/node-problem-detector.yaml
```

### Check Extended Conditions

```sh
# View all conditions (including NPD-added ones)
kubectl get node <node-name> -o jsonpath='{.status.conditions}' | jq '.[] | {type, status, message}'

# Watch for NPD events
kubectl get events --field-selector source=node-problem-detector --sort-by='.lastTimestamp'
```

### Remediation with NPD

NPD reports conditions but doesn't fix them. To auto-remediate, pair it with:

- **Draino** — automatically drains nodes with specific conditions
- **Kured** — reboots nodes when `/var/run/reboot-required` exists
- **Custom controller** — watches node conditions and triggers actions

```yaml
# Example: Draino config to drain nodes with ReadonlyFilesystem
apiVersion: apps/v1
kind: Deployment
metadata:
  name: draino
spec:
  template:
    spec:
      containers:
        - name: draino
          args:
            - --node-conditions=ReadonlyFilesystem,KernelDeadlock,FrequentContainerdRestart
            - --evict-unreplicated-pods
```

## Layer 4: EKS Managed Node Group Auto-Repair

EKS managed node groups have **built-in auto-repair** that replaces unhealthy nodes automatically.

### How It Works

1. EKS monitors the EC2 and Kubernetes health of nodes in managed node groups
2. If a node is `NotReady` for an extended period (typically 10-15 minutes), EKS marks it for replacement
3. EKS terminates the unhealthy instance
4. The ASG launches a replacement instance
5. The new node joins the cluster automatically

### What Triggers Auto-Repair

| Trigger | Detection Time | Action |
|---------|:-------------:|--------|
| EC2 Status Check failure | ~2-5 minutes | Replace instance |
| Node `NotReady` sustained | ~10-15 minutes | Terminate and replace |
| EC2 scheduled maintenance | At notice time | Replace before maintenance |

### Configuration

Auto-repair is enabled by default on managed node groups. To check:

```sh
# Check node group health
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng> \
  --query "nodegroup.health" --output json

# Check node repair config (if available)
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng> \
  --query "nodegroup.nodeRepairConfig" --output json
```

### Enable or Disable Node Repair

```sh
# Enable auto-repair (default)
aws eks update-nodegroup-config --cluster-name <cluster> --nodegroup-name <ng> \
  --node-repair-config enabled=true

# Disable (not recommended for production)
aws eks update-nodegroup-config --cluster-name <cluster> --nodegroup-name <ng> \
  --node-repair-config enabled=false
```

### Limitations

- Only works with managed node groups (not self-managed ASGs)
- Doesn't drain the node gracefully before termination (pods may be interrupted)
- Doesn't respect PDBs during health-based replacement
- Detection can take 10-15 minutes (not instant)

## Layer 5: Karpenter Disruption

If you're using Karpenter, it handles node health as part of its disruption engine:

### Disruption Types

| Type | Trigger | Behavior |
|------|---------|----------|
| **Spot interruption** | AWS 2-min notice | Immediate cordon + drain |
| **Instance health event** | EC2 status check failure | Cordon + drain + replace |
| **Scheduled maintenance** | EC2 scheduled event | Drain before maintenance window |
| **Drift** | Node doesn't match NodePool spec | Cordon + drain + replace |
| **Consolidation** | Node underutilized | Drain + terminate |
| **Expiry** | Node exceeds `expireAfter` | Drain + replace |

### Configuration

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
    expireAfter: 168h  # 7-day max node lifetime
    budgets:
      - nodes: "10%"   # Max 10% disrupted at once
```

### Karpenter + Spot: Automatic Handling

Karpenter automatically:
1. Watches for Spot interruption notices (via SQS queue)
2. Cordons the node immediately
3. Drains pods (respecting PDBs)
4. Launches a replacement node (possibly different instance type)
5. Pods reschedule to new node

```sh
# Check for pending disruptions
kubectl get nodeclaims -o custom-columns=NAME:.metadata.name,STATE:.status.conditions[-1].type,DISRUPTION:.metadata.annotations.karpenter\\.sh/disruption-reason
```

## Layer 6: Node Termination Handler (Self-Managed Nodes)

For self-managed node groups (ASGs without EKS managed repair), deploy the [AWS Node Termination Handler](https://github.com/aws/aws-node-termination-handler):

### What It Handles

- EC2 Spot interruption notices (2-min warning)
- EC2 scheduled maintenance events
- EC2 instance rebalance recommendations
- ASG lifecycle hooks (scale-in)

### Installation

```sh
helm repo add eks https://aws.github.io/eks-charts
helm install aws-node-termination-handler eks/aws-node-termination-handler \
  --namespace kube-system \
  --set enableSpotInterruptionDraining=true \
  --set enableScheduledEventDraining=true \
  --set enableRebalanceMonitoring=true
```

### How It Works

```
EC2 event detected (via IMDS polling or SQS)
    │
    ▼
NTH cordons the node
    │
    ▼
NTH drains pods (respects PDBs, terminationGracePeriod)
    │
    ▼
Instance terminates (Spot interruption or maintenance)
    │
    ▼
ASG launches replacement
    │
    ▼
New node joins cluster
```

### IMDS Mode vs Queue Mode

| Mode | How Events Are Detected | Best For |
|------|------------------------|----------|
| IMDS (default) | Polls instance metadata on each node | Simple setup, no extra infra |
| Queue (SQS) | Central SQS queue receives all events | Faster detection, supports ASG lifecycle hooks |

For production, use Queue mode — it detects events faster and handles ASG lifecycle hooks for graceful scale-in.

## Self-Managed ASG Auto-Repair (Without NTH)

For self-managed nodes without NTH or Karpenter, you can configure ASG health checks:

### EC2 Health Check (Default)

ASG replaces instances that fail EC2 status checks. No Kubernetes awareness:

```sh
# ASG health check type
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg> \
  --query "AutoScalingGroups[0].HealthCheckType" --output text
# Output: EC2
```

### ELB Health Check (With Target Group)

If the ASG is associated with a target group, unhealthy targets get replaced:

```sh
# Change to ELB health check
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name <asg> \
  --health-check-type ELB \
  --health-check-grace-period 300
```

> ELB health checks don't drain nodes before termination. Combine with NTH lifecycle hooks for graceful shutdown.

## Combining Strategies

### Recommended: Managed Node Groups

```
EKS auto-repair (built-in)
  + Node Problem Detector (extends detection)
  + PDBs (protects workloads during repair)
```

### Recommended: Karpenter

```
Karpenter disruption engine (handles everything)
  + SQS queue (for Spot and maintenance events)
  + Node Problem Detector (optional, adds kernel-level detection)
  + PDBs + Node Disruption Budgets
```

### Recommended: Self-Managed Nodes

```
Node Termination Handler (drain before terminate)
  + ASG ELB health checks (replace unresponsive nodes)
  + Node Problem Detector (detect OS-level issues)
  + CloudWatch alarm → SNS → Lambda (custom remediation)
```

## Monitoring Auto-Repair Activity

### CloudWatch Metrics

```sh
# Node group health issues
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng> \
  --query "nodegroup.health.issues" --output json
```

### Kubernetes Events

```sh
# Node-related events (repairs, replacements)
kubectl get events -A --field-selector involvedObject.kind=Node --sort-by='.lastTimestamp' | tail -20

# Spot interruption events (if NTH deployed)
kubectl get events -A --field-selector reason=SpotInterruption
```

### Prometheus Alerts

```yaml
groups:
  - name: node-health
    rules:
      - alert: NodeNotReadyExtended
        expr: kube_node_status_condition{condition="Ready", status="true"} == 0
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.node }} has been NotReady for 10+ minutes"
          description: "Auto-repair should kick in soon. If not, investigate manually."

      - alert: NodeAutoRepairTriggered
        expr: increase(kube_pod_container_status_restarts_total{namespace="kube-system", container="aws-node-termination-handler"}[5m]) > 0
        labels:
          severity: warning
        annotations:
          summary: "NTH restarted — possible node health event"

      - alert: NodeReplacedByASG
        expr: changes(kube_node_created[10m]) > 0 and changes(kube_node_info[10m]) > 0
        labels:
          severity: info
        annotations:
          summary: "New node appeared — possible auto-repair replacement"
```

### Datadog

```json
{
  "name": "EKS Node Auto-Repair Triggered",
  "type": "event alert",
  "query": "events('source:kubernetes priority:all status:warn tags:reason:NodeNotReady').rollup('count').last('15m') > 0",
  "message": "A node has been NotReady long enough to trigger auto-repair.\n\nCheck:\n- `kubectl get nodes`\n- ASG activity\n- Node group health\n\n@slack-infra-alerts"
}
```

## Gotchas

- **Auto-repair doesn't drain gracefully**: EKS managed auto-repair terminates the instance without cordoning or draining. Pods on that node are abruptly killed. Always use PDBs to protect critical workloads.
- **Detection latency**: It takes 10-15 minutes for EKS to detect and act on a NotReady node. For faster recovery, use Karpenter or custom monitoring.
- **Spot nodes have only 2 minutes**: When a Spot instance gets interrupted, you have 2 minutes to drain. Without NTH or Karpenter, pods are killed without warning.
- **NotReady ≠ always a node problem**: Kubelet can't reach the API server due to network issues, but the node itself is fine. Auto-repair may replace a perfectly healthy node with a network blip.
- **PDBs can block Karpenter**: If a PDB prevents draining, Karpenter (and NTH) can't complete the replacement. Set `consolidateAfter` high enough for PDB-heavy workloads.
- **Self-managed ASG health checks are basic**: EC2 health checks only detect hardware failures. They don't detect kubelet crashes, OOM, or application-level issues.
- **Node Problem Detector doesn't repair**: It only reports. You need an additional component (Draino, Kured, custom controller) to act on the conditions.
- **Multiple repair mechanisms can conflict**: If you have both EKS auto-repair AND a custom Lambda replacing nodes, they may race. Choose one primary mechanism.
