# EKS Node Troubleshooting Guide

Diagnosing unhealthy, NotReady, or misbehaving EKS worker nodes — from initial triage through kubelet, networking, storage, and instance-level issues.

## Quick Triage Flow

```
Node Problem
  │
  ├─ Node visible in cluster?
  │    ├─ No  → Instance not joining (bootstrap, IAM, networking)
  │    └─ Yes → Check node status
  │
  ├─ Node status?
  │    ├─ NotReady    → kubelet, container runtime, or system issue
  │    ├─ Ready       → Resource pressure or pod-level issues
  │    └─ Unknown     → Lost contact with API server
  │
  ├─ Node conditions?
  │    ├─ MemoryPressure  → OOM risk, evictions starting
  │    ├─ DiskPressure    → Ephemeral storage full
  │    ├─ PIDPressure     → Too many processes
  │    └─ NetworkUnavailable → CNI plugin issue
  │
  └─ Recent events?
       └─ kubectl describe node <name>
```

## Initial Investigation Commands

```bash
# Node status overview
kubectl get nodes -o wide

# Detailed node info — conditions, capacity, allocatable, events
kubectl describe node <node-name>

# Node conditions only
kubectl get node <node-name> -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}'

# Pods on the node
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node-name> -o wide

# Events for the node
kubectl get events --field-selector involvedObject.name=<node-name> --sort-by='.lastTimestamp'

# Node resource usage
kubectl top node <node-name>

# All pods resource usage on the node
kubectl top pods --all-namespaces --field-selector spec.nodeName=<node-name>
```

## Node NotReady

### Common Causes

| Cause | Symptom | Fix |
|-------|---------|-----|
| kubelet stopped | Node goes NotReady after ~40s | Restart kubelet |
| Container runtime crashed | kubelet reports runtime errors | Restart containerd |
| Network partition | Node can't reach API server | Check SG, NACLs, route tables |
| Disk full | kubelet can't write | Clear space, expand volume |
| Kernel panic/hang | Instance unresponsive | Terminate and let ASG replace |
| Instance status check failed | EC2 hardware issue | Terminate and let ASG replace |

### Investigate via SSM (No SSH Required)

```bash
# Start session via SSM
aws ssm start-session --target <instance-id>

# Check kubelet status
sudo systemctl status kubelet
sudo journalctl -u kubelet --since "10 minutes ago" --no-pager

# Check containerd
sudo systemctl status containerd
sudo journalctl -u containerd --since "10 minutes ago" --no-pager

# Check system health
uptime
free -h
df -h
dmesg | tail -50
```

### kubelet Not Running

```bash
# Check kubelet status
sudo systemctl status kubelet

# Common states:
# - Active (running) → kubelet is fine, check other causes
# - Inactive (dead) → needs restart
# - Activating (auto-restart) → crashing in a loop

# View kubelet logs
sudo journalctl -u kubelet -e --no-pager | tail -100

# Restart kubelet
sudo systemctl restart kubelet

# Watch it come back
sudo journalctl -u kubelet -f
```

Common kubelet crash reasons:
- Certificate expired or not found
- Can't reach API server
- Bad kubelet config (`/etc/kubernetes/kubelet/kubelet-config.json`)
- Disk pressure preventing operation

### Container Runtime Issues

```bash
# Check containerd
sudo systemctl status containerd
sudo journalctl -u containerd --since "5 minutes ago" --no-pager

# Check runtime endpoint
sudo crictl info
sudo crictl ps

# Restart containerd (last resort — will restart all pods)
sudo systemctl restart containerd
```

## Node Not Joining the Cluster

### Bootstrap Process

EKS nodes follow this sequence:
1. EC2 instance launches
2. User data script runs (bootstrap.sh)
3. kubelet starts with cluster CA and API endpoint
4. kubelet registers with the API server
5. aws-auth ConfigMap maps instance role to Kubernetes identity

### Check Bootstrap Logs

```bash
# Via SSM on the instance
sudo cat /var/log/cloud-init-output.log
sudo cat /var/log/messages | grep -i kubelet
sudo journalctl -u kubelet --no-pager | head -100
```

### Common Join Failures

#### IAM Role Not in aws-auth

```bash
# Check aws-auth ConfigMap
kubectl get configmap aws-auth -n kube-system -o yaml
```

The node's instance role ARN must be mapped:

```yaml
mapRoles:
- rolearn: arn:aws:iam::123456789012:role/eks-node-role
  username: system:node:{{EC2PrivateDNSName}}
  groups:
  - system:bootstrappers
  - system:nodes
```

#### Security Group Blocking Communication

Required connectivity:
- Node → API server (TCP 443)
- API server → Node (TCP 10250, for kubelet)
- Node → Node (all ports, for pod-to-pod)

```bash
# Check node security groups
aws ec2 describe-instances --instance-ids <instance-id> \
  --query 'Reservations[*].Instances[*].SecurityGroups' --output table

# Verify connectivity from the node
curl -k https://<api-server-endpoint>
```

#### Wrong Cluster Name or Endpoint

```bash
# Check kubelet args on the node
ps aux | grep kubelet | grep -o '\-\-kubeconfig=[^ ]*'
sudo cat /var/lib/kubelet/kubeconfig

# Check bootstrap script
sudo cat /etc/eks/bootstrap.sh
```

#### Subnet Has No Route to API Server

```bash
# Check route table for the subnet
aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<subnet-id>" \
  --query 'RouteTables[*].Routes'
```

For private subnets, ensure:
- NAT Gateway route for internet access, OR
- VPC endpoints for EKS, EC2, ECR, S3, STS

## Resource Pressure

### Memory Pressure

```bash
# Check from kubectl
kubectl describe node <node-name> | grep -A3 "MemoryPressure"

# On the node
free -h
cat /proc/meminfo | grep -E "MemTotal|MemAvailable|MemFree"

# Top memory consumers
ps aux --sort=-%mem | head -20

# Check kubelet eviction thresholds
cat /etc/kubernetes/kubelet/kubelet-config.json | grep -A5 eviction
```

Default eviction thresholds:
- `memory.available < 100Mi` → eviction starts
- `memory.available < 0` → hard eviction

Fix options:
- Scale up the node group (larger instance type)
- Reduce pod memory requests to schedule fewer pods
- Set proper memory limits on pods to prevent leaks
- Add cluster autoscaler / Karpenter for scaling

### Disk Pressure

```bash
# Check from kubectl
kubectl describe node <node-name> | grep -A3 "DiskPressure"

# On the node
df -h
du -sh /var/lib/containerd/*
du -sh /var/log/*

# Find large files
find / -size +100M -type f 2>/dev/null | head -20

# Check image storage
sudo crictl images | sort -k3 -h
```

Default eviction thresholds:
- `nodefs.available < 10%` → eviction starts
- `imagefs.available < 15%` → image garbage collection

Fix options:
- Clean unused images: `sudo crictl rmi --prune`
- Clear old logs: `sudo journalctl --vacuum-size=500M`
- Expand the EBS root volume
- Use larger root volume in launch template

### PID Pressure

```bash
# Check from kubectl
kubectl describe node <node-name> | grep -A3 "PIDPressure"

# On the node
cat /proc/sys/kernel/pid_max
ls /proc | grep -c "^[0-9]"

# Top PID consumers
ps aux --sort=-nlwp | head -20

# Per-cgroup PID count
find /sys/fs/cgroup -name "pids.current" -exec sh -c 'echo "$1: $(cat $1)"' _ {} \; | sort -t: -k2 -rn | head -20
```

## Networking Issues

### Node NetworkUnavailable

Usually means the CNI plugin (VPC CNI) failed to configure.

```bash
# Check aws-node DaemonSet (VPC CNI)
kubectl get pods -n kube-system -l k8s-app=aws-node -o wide | grep <node-name>
kubectl logs -n kube-system <aws-node-pod> -c aws-node

# On the node
sudo journalctl -u kubelet | grep -i network
ip addr show
ip route show
```

### VPC CNI — IP Address Exhaustion

```bash
# Check ENI and IP allocation on the node
kubectl get node <node-name> -o jsonpath='{.status.allocatable.pods}'

# aws-node logs for IP allocation failures
kubectl logs -n kube-system <aws-node-pod> -c aws-node | grep -i "failed\|error\|exhaust"

# Check subnet available IPs
aws ec2 describe-subnets --subnet-ids <subnet-id> \
  --query 'Subnets[*].[SubnetId, AvailableIpAddressCount, CidrBlock]' --output table
```

Fix options:
- Use prefix delegation (`ENABLE_PREFIX_DELEGATION=true`)
- Add secondary CIDR to VPC
- Use smaller pod density per node
- Switch to custom networking with separate pod subnets

### DNS Resolution Failures

```bash
# Test DNS from a debug pod on the node
kubectl run dns-test --image=busybox --restart=Never --overrides='{"spec":{"nodeName":"<node-name>"}}' -- nslookup kubernetes.default

# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide

# Check resolv.conf in pods
kubectl exec <pod-on-node> -- cat /etc/resolv.conf
```

## Instance-Level Issues

### EC2 Status Checks

```bash
# Check instance status
aws ec2 describe-instance-status --instance-ids <instance-id> \
  --query 'InstanceStatuses[*].[InstanceId, InstanceStatus.Status, SystemStatus.Status]' --output table
```

| Status Check | Meaning | Action |
|-------------|---------|--------|
| System: impaired | Hardware/hypervisor issue | Stop and start (migrates to new hardware) |
| Instance: impaired | OS-level issue | Reboot or terminate |
| Both: ok | EC2 infrastructure fine | Issue is in Kubernetes layer |

### Instance Metadata Service (IMDS)

EKS nodes need IMDS for IAM roles:

```bash
# Test IMDS from the node
curl -s http://169.254.169.254/latest/meta-data/instance-id
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/

# If using IMDSv2
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
```

If IMDS is unreachable:
- Check hop limit for IMDSv2 (must be 2 for containers): `aws ec2 modify-instance-metadata-options --instance-id <id> --http-put-response-hop-limit 2`
- Check if NetworkPolicy or iptables rules block `169.254.169.254`

### NTP / Clock Skew

Certificate validation and token expiry depend on accurate time:

```bash
# Check time sync
timedatectl status
chronyc tracking
chronyc sources

# If clock is off, kubelet certificate validation fails
# Fix:
sudo systemctl restart chronyd
```

## kubelet Configuration Issues

### View Effective Configuration

```bash
# On the node
sudo cat /etc/kubernetes/kubelet/kubelet-config.json

# Or via API (if node is Ready)
kubectl get --raw /api/v1/nodes/<node-name>/proxy/configz | jq
```

### Common Misconfigurations

| Issue | Symptom | Where to Fix |
|-------|---------|-------------|
| Wrong `--max-pods` | Pods stuck Pending even with resources | Launch template user data or kubelet-config |
| `evictionHard` too aggressive | Frequent evictions | kubelet-config |
| `kubeReserved` too high | Less allocatable resources | kubelet-config |
| `systemReserved` too low | System processes starved | kubelet-config |

### Kubelet Reserved Resources

```bash
# View allocatable vs capacity
kubectl describe node <node-name> | grep -A10 "Allocatable" | head -12
kubectl describe node <node-name> | grep -A10 "Capacity" | head -12
```

```
Capacity:
  cpu:                4
  memory:             16384Mi
  pods:               58
Allocatable:
  cpu:                3920m       # 4000m - 80m (kube-reserved)
  memory:             15764Mi     # Minus kube-reserved and system-reserved
  pods:               58
```

## EKS-Specific Issues

### Managed Node Group Update Stuck

```bash
# Check node group status
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <nodegroup> \
  --query 'nodegroup.[status, health.issues]'

# Check for PDB blocking drain
kubectl get pdb --all-namespaces

# Check pods that won't evict
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node-name> \
  -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'
```

### Node Stuck in Draining

```bash
# Find pods blocking drain
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node-name> | grep -v "Running\|Succeeded"

# Check for pods with no ownerReferences (standalone pods)
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node-name> \
  -o jsonpath='{range .items[?(@.metadata.ownerReferences==nil)]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'

# Force delete stuck pods (if safe)
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force
```

### AMI / Launch Template Issues

```bash
# Check what AMI the node is running
aws ec2 describe-instances --instance-ids <instance-id> \
  --query 'Reservations[*].Instances[*].[ImageId, LaunchTemplate]' --output table

# Compare with latest EKS AMI
aws ssm get-parameter --name /aws/service/eks/optimized-ami/<k8s-version>/amazon-linux-2/recommended/image_id \
  --query 'Parameter.Value' --output text
```

### EKS Add-on Issues Affecting Nodes

```bash
# Check add-on status
aws eks describe-addon --cluster-name <cluster> --addon-name vpc-cni \
  --query 'addon.[addonVersion, status, health]'

# Key add-ons that affect node health:
# - vpc-cni (aws-node): networking
# - kube-proxy: service routing
# - coredns: DNS resolution
```

## Immediate Actions

When a node is at risk of termination, act fast:

```bash
# 1. Cordon to prevent new pods from scheduling
kubectl cordon <node-name>

# 2. Get instance ID
INSTANCE_ID=$(kubectl get node <node-name> -o json | jq -r '.spec.providerID' | cut -d'/' -f5)

# 3. Preserve evidence before the node is gone
kubectl describe node <node-name> > node-describe-backup.txt
kubectl get events --field-selector involvedObject.name=<node-name> > node-events-backup.txt
kubectl get pods -A --field-selector spec.nodeName=<node-name> -o yaml > pods-on-node-backup.yaml
aws ec2 get-console-output --instance-id $INSTANCE_ID --latest --query 'Output' --output text > console-output.txt
```

## Time-Critical Phased Investigation

### Phase 1: Immediate Assessment (30 seconds)

```bash
# Node status
kubectl get nodes | grep <node-name>

# Ready condition with reason
kubectl get node <node-name> -o json | jq -r '.status.conditions[] | select(.type=="Ready") | "\(.status) - \(.reason) - \(.message)"'

# Emergency resource pressure check
kubectl get node <node-name> -o json | jq '.status.conditions[] | select(.type == "Ready" or .type == "DiskPressure" or .type == "MemoryPressure" or .type == "PIDPressure") | {type: .type, status: .status, reason: .reason, message: .message}'
```

### Phase 2: Detailed Investigation (2-3 minutes)

```bash
# Full node description
kubectl describe node <node-name>

# Recent events
kubectl get events --field-selector involvedObject.name=<node-name> --sort-by='.lastTimestamp' | tail -10

# Pods on node
kubectl get pods -A --field-selector spec.nodeName=<node-name>

# Evicted or failing pods
kubectl get pods -A --field-selector spec.nodeName=<node-name> | grep -E "(Evicted|Failed|Pending|Error)"

# Resource allocation percentage
kubectl describe node <node-name> | awk '/Allocated resources:/,/Events:/' | grep -E "(cpu|memory|ephemeral-storage)" | grep -E "\([0-9]+%\)"
```

### Phase 3: AWS Investigation (2-3 minutes)

```bash
INSTANCE_ID=$(kubectl get node <node-name> -o json | jq -r '.spec.providerID' | cut -d'/' -f5)

# EC2 instance status
aws ec2 describe-instance-status --instance-ids $INSTANCE_ID

# Console output (system logs — no SSH needed)
aws ec2 get-console-output --instance-id $INSTANCE_ID --latest --query 'Output' --output text | tail -50

# Auto Scaling Group activities
ASG_NAME=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID --query 'Reservations[0].Instances[0].Tags[?Key==`aws:autoscaling:groupName`].Value' --output text)
aws autoscaling describe-scaling-activities --auto-scaling-group-name $ASG_NAME --max-items 5
```

## SSM Remote Commands (No SSH Required)

When SSH isn't available, use SSM to run commands and retrieve output:

```bash
# Check if SSM agent is responsive
aws ssm describe-instance-information --filters "Key=InstanceIds,Values=<instance-id>" \
  --query 'InstanceInformationList[0].PingStatus'
```

### Disk Space Check

```bash
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["df -h", "du -sh /var/lib/kubelet", "du -sh /var/lib/containerd"]' \
  --output text --query 'Command.CommandId'
```

### Memory and OOM Check

```bash
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["free -h", "cat /proc/meminfo | head -20", "sudo dmesg | grep -i \"killed process\" | tail -5"]' \
  --output text --query 'Command.CommandId'
```

### kubelet Health Check

```bash
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["sudo systemctl is-active kubelet", "sudo journalctl -u kubelet --since \"15 minutes ago\" --no-pager | tail -30"]' \
  --output text --query 'Command.CommandId'
```

### Container Runtime Check

```bash
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["sudo systemctl is-active containerd", "sudo crictl ps | wc -l", "sudo crictl images | wc -l"]' \
  --output text --query 'Command.CommandId'
```

### Get Command Results

```bash
# Replace COMMAND_ID with the output from above
aws ssm get-command-invocation --command-id <COMMAND_ID> --instance-id <instance-id> \
  --query '[StandardOutputContent, StandardErrorContent]' --output text
```

## Deep System Analysis

### OOM Kills and Resource Pressure

```bash
# Check for out-of-memory kills
sudo dmesg | grep -i "killed process\|out of memory\|oom"

# Pressure Stall Information (PSI)
cat /proc/pressure/memory
cat /proc/pressure/cpu
cat /proc/pressure/io

# Check for deleted files still held open (hidden disk usage)
sudo lsof +L1

# Inode usage (can exhaust even with free disk space)
df -i
```

### Network Deep Dive

```bash
# Check network interfaces and routes
ip addr show
ip route show

# Check DNS resolution
nslookup kubernetes.default.svc.cluster.local

# Check connectivity to API server
curl -k https://<eks-endpoint>/healthz

# Check CNI plugin binaries and config
ls -la /opt/cni/bin/
cat /etc/cni/net.d/*
```

### Pod Resource Analysis

```bash
# Find resource-heavy pods on the node
kubectl get pods -A --field-selector spec.nodeName=<node-name> -o json | \
  jq '.items[] | {name: .metadata.name, namespace: .metadata.namespace, requests: .spec.containers[].resources.requests}'

# Check for pods without resource limits
kubectl get pods -A --field-selector spec.nodeName=<node-name> -o json | \
  jq '.items[] | select(.spec.containers[].resources.limits == null) | {name: .metadata.name, namespace: .metadata.namespace}'
```

## CloudWatch Investigation (No Instance Access)

### EC2 Instance Metrics

```bash
# CPU utilization over last hour
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=<instance-id> \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum
```

### EBS Volume Metrics

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeReadOps \
  --dimensions Name=VolumeId,Value=<volume-id> \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum
```

### Container Insights (If Enabled)

```bash
aws logs filter-log-events \
  --log-group-name "/aws/containerinsights/<cluster-name>/performance" \
  --start-time $(date -v-1H +%s)000 \
  --filter-pattern "{ $.kubernetes.host = \"<node-name>\" }"
```

## System Components Check

```bash
# Check all system pods
kubectl get pods -n kube-system -o wide | grep <node-name>

# Check VPC CNI DaemonSet
kubectl get daemonset -n kube-system aws-node
kubectl get pods -n kube-system -l k8s-app=aws-node -o wide

# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide | grep <node-name>

# VPC CNI configuration
kubectl get configmap -n kube-system amazon-vpc-cni -o yaml
```

### VPC CNI Troubleshooting

```bash
# Restart VPC CNI pods on the node
kubectl delete pod -n kube-system -l k8s-app=aws-node --field-selector spec.nodeName=<node-name>

# Check ENI limits for instance type
aws ec2 describe-instance-types --instance-types <instance-type> \
  --query 'InstanceTypes[0].NetworkInfo.[MaximumNetworkInterfaces, Ipv4AddressesPerInterface]' --output table

# Check subnet IP availability
aws ec2 describe-subnets --subnet-ids <subnet-id> \
  --query 'Subnets[0].AvailableIpAddressCount'
```

## Quick Resolution Steps

### Terminate and Replace (ASG-Managed)

```bash
# Cordon, drain, then terminate — ASG launches replacement
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --timeout=300s
aws ec2 terminate-instances --instance-ids <instance-id>
```

### Force Drain and Delete

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --force --grace-period=300
kubectl delete node <node-name>
```

### Update Node Group

```bash
aws eks update-nodegroup-version --cluster-name <cluster> --nodegroup-name <nodegroup>
```

### Scale Node Group (Force Recreation)

```bash
# Scale down
aws eks update-nodegroup-config --cluster-name <cluster> --nodegroup-name <nodegroup> \
  --scaling-config minSize=0,maxSize=3,desiredSize=0

# Wait for termination, then scale back up
aws eks update-nodegroup-config --cluster-name <cluster> --nodegroup-name <nodegroup> \
  --scaling-config minSize=1,maxSize=3,desiredSize=2
```

### Node Problem Detector

If installed, provides detailed node health:

```bash
kubectl get events --field-selector source=node-problem-detector,involvedObject.name=<node-name>

# Common conditions reported:
# - KernelDeadlock
# - ReadonlyFilesystem
# - FrequentContainerdRestart
# - FrequentKubeletRestart
```

## Data Collection Script

Automated script to gather all troubleshooting data in one shot:

```bash
#!/bin/bash
# EKS Node Troubleshooting Data Collection
# Usage: ./collect-node-data.sh <node-name>

NODE_NAME="$1"
if [[ -z "$NODE_NAME" ]]; then
    echo "Usage: $0 <node-name>"
    exit 1
fi

OUTPUT_DIR="node-debug-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$OUTPUT_DIR"

echo "Collecting troubleshooting data for node: $NODE_NAME"
echo "Output directory: $OUTPUT_DIR"

# Get instance ID
INSTANCE_ID=$(kubectl get node "$NODE_NAME" -o json | jq -r '.spec.providerID' | cut -d'/' -f5)
echo "Instance ID: $INSTANCE_ID"

# Kubernetes data
kubectl describe node "$NODE_NAME" > "$OUTPUT_DIR/node-describe.txt"
kubectl get node "$NODE_NAME" -o yaml > "$OUTPUT_DIR/node-yaml.yaml"
kubectl get node "$NODE_NAME" -o json | jq '.status.conditions' > "$OUTPUT_DIR/node-conditions.json"
kubectl get events --field-selector involvedObject.name="$NODE_NAME" --sort-by='.lastTimestamp' > "$OUTPUT_DIR/node-events.txt"
kubectl get pods -A --field-selector spec.nodeName="$NODE_NAME" -o wide > "$OUTPUT_DIR/pods-on-node.txt"
kubectl describe node "$NODE_NAME" | grep -A 15 "Allocated resources" > "$OUTPUT_DIR/resource-allocation.txt"
kubectl top node "$NODE_NAME" > "$OUTPUT_DIR/node-resource-usage.txt" 2>/dev/null

# AWS data
aws ec2 describe-instances --instance-ids "$INSTANCE_ID" > "$OUTPUT_DIR/ec2-instance.json"
aws ec2 describe-instance-status --instance-ids "$INSTANCE_ID" > "$OUTPUT_DIR/ec2-status.json"
aws ec2 get-console-output --instance-id "$INSTANCE_ID" --latest --query 'Output' --output text > "$OUTPUT_DIR/console-output.txt"

# ASG data
ASG_NAME=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].Tags[?Key==`aws:autoscaling:groupName`].Value' --output text)
if [[ "$ASG_NAME" != "None" && -n "$ASG_NAME" ]]; then
    aws autoscaling describe-scaling-activities --auto-scaling-group-name "$ASG_NAME" --max-items 10 > "$OUTPUT_DIR/asg-activities.json"
fi

# SSM data (if available)
SSM_STATUS=$(aws ssm describe-instance-information --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
  --query 'InstanceInformationList[0].PingStatus' --output text 2>/dev/null)
if [[ "$SSM_STATUS" == "Online" ]]; then
    echo "SSM available — collecting system info..."
    COMMAND_ID=$(aws ssm send-command \
        --instance-ids "$INSTANCE_ID" \
        --document-name "AWS-RunShellScript" \
        --parameters 'commands=["df -h", "free -h", "uptime", "sudo systemctl status kubelet", "sudo systemctl status containerd", "sudo dmesg | tail -30"]' \
        --query 'Command.CommandId' --output text)
    sleep 10
    aws ssm get-command-invocation --command-id "$COMMAND_ID" --instance-id "$INSTANCE_ID" > "$OUTPUT_DIR/ssm-system-info.json"
fi

echo "Data collection complete: $OUTPUT_DIR/"
echo "Start with: $OUTPUT_DIR/node-describe.txt"
```

## Quick One-Liners

```bash
# Complete node status in one command
kubectl get node <node-name> -o json | jq '{name: .metadata.name, ready: .status.conditions[] | select(.type=="Ready"), capacity: .status.capacity, allocatable: .status.allocatable}'

# Get instance ID from node name
kubectl get node <node-name> -o json | jq -r '.spec.providerID' | cut -d'/' -f5

# Describe all NotReady nodes
kubectl get nodes --no-headers | grep NotReady | awk '{print $1}' | xargs -I {} kubectl describe node {}

# Check recent termination/eviction events
kubectl get events -A --sort-by='.lastTimestamp' | grep -E "(terminate|evict|kill)" | tail -10

# Check node allocation
kubectl describe node <node-name> | grep -A 20 "Allocated resources:"
```

## Useful Aliases

```bash
alias eks-nodes='kubectl get nodes -o wide'
alias eks-events='kubectl get events --all-namespaces --sort-by=.lastTimestamp | tail -20'
alias eks-system='kubectl get pods -n kube-system -o wide'

# Describe all NotReady nodes
notready() {
  kubectl get nodes --no-headers | grep NotReady | awk '{print $1}' | xargs -I {} kubectl describe node {}
}

# Get instance ID from node name
node-id() {
  kubectl get node "$1" -o json | jq -r '.spec.providerID' | cut -d'/' -f5
}
```

## Troubleshooting Checklist

```
□ kubectl get nodes -o wide (is the node visible? what status?)
□ kubectl describe node <name> (conditions, events, capacity)
□ kubectl get events for the node
□ Check resource pressure (memory, disk, PID conditions)
□ Get instance ID and check EC2 status checks
□ Get console output (aws ec2 get-console-output)
□ SSM into the node (or use send-command):
  □ systemctl status kubelet
  □ systemctl status containerd
  □ journalctl -u kubelet (recent errors)
  □ free -h / df -h / uptime
  □ dmesg | tail -50
□ Check security groups (node ↔ API server, node ↔ node)
□ Check aws-auth ConfigMap (role mapping)
□ Check VPC CNI (aws-node pod logs, subnet IPs)
□ Check ASG scaling activities
□ Check if drain is blocked by PDBs or standalone pods
□ Check CloudWatch metrics if no instance access
□ Preserve evidence if node may terminate
```
