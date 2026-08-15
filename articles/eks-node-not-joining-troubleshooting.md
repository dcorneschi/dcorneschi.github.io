# EKS Node Not Joining Cluster — Troubleshooting

When an EC2 instance launches but never appears as a node in `kubectl get nodes`, the problem is between the instance and the API server. This guide covers systematic diagnosis of the most common causes.

## How a Node Joins an EKS Cluster

Understanding the flow helps pinpoint where it breaks:

```
1. EC2 instance launches (ASG / managed node group / Karpenter)
2. Userdata runs the bootstrap script (/etc/eks/bootstrap.sh)
3. Bootstrap configures kubelet with:
   - API server endpoint
   - Certificate authority (CA)
   - Cluster name
4. Kubelet starts and calls the API server
5. API server authenticates the node via aws-iam-authenticator
6. aws-auth ConfigMap (or Access Entries) maps the node's IAM role → system:nodes group
7. Node is authorized → transitions to Ready
```

If any step fails, the node stays `NotReady` or never appears at all.

## Quick Diagnosis Flowchart

```
Node not in "kubectl get nodes"?
    │
    ├── Can you SSH/SSM into the instance?
    │   ├── No → Security group, SSM agent, or instance unreachable
    │   └── Yes ↓
    │
    ├── Is kubelet running?
    │   ├── No → Bootstrap script failed or kubelet crash
    │   └── Yes ↓
    │
    ├── Can kubelet reach the API server?
    │   ├── No → Networking (SG, NACL, route table, DNS, endpoint)
    │   └── Yes ↓
    │
    ├── Is the node authenticated?
    │   ├── No → IAM role not mapped in aws-auth / Access Entries
    │   └── Yes ↓
    │
    └── Node should be visible → check for NotReady condition
```

## Step 1: Check if the Instance Is Running

```sh
# Find instances in the node group
aws ec2 describe-instances \
  --filters "Name=tag:eks:cluster-name,Values=<cluster>" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].{ID:InstanceId, State:State.Name, AZ:Placement.AvailabilityZone, LaunchTime:LaunchTime}" \
  --output table

# Check node group status
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng> \
  --query "nodegroup.{Status:status, Health:health, Desired:scalingConfig.desiredSize}" --output json
```

If the instance isn't running, check the ASG activity:

```sh
# Get the ASG name
ASG=$(aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng> \
  --query "nodegroup.resources.autoScalingGroups[0].name" --output text)

# Check ASG activity
aws autoscaling describe-scaling-activities --auto-scaling-group-name $ASG \
  --query "Activities[0:5].{Status:StatusCode, Cause:Cause}" --output table
```

## Step 2: Check Instance Logs (Without SSH)

For managed node groups, use SSM or EC2 console:

```sh
# Get console output (shows boot logs)
aws ec2 get-console-output --instance-id <instance-id> --output text

# If SSM agent is running, connect via Session Manager
aws ssm start-session --target <instance-id>
```

If SSM isn't available:

```sh
# Check if SSM agent can reach the instance
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=<instance-id>" --output table
```

## Step 3: Check the Bootstrap Script

SSH or SSM into the node and check:

```sh
# Check if bootstrap ran
cat /var/log/cloud-init-output.log | grep -i "bootstrap\|error\|fail"

# Check bootstrap script logs
journalctl -u kubelet --no-pager | tail -50

# Check if kubelet is running
systemctl status kubelet

# Check kubelet config
cat /var/lib/kubelet/kubeconfig
cat /etc/kubernetes/kubelet/kubelet-config.json
```

### Common Bootstrap Failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| bootstrap.sh not found | Wrong AMI (not EKS-optimized) | Use EKS-optimized AMI |
| `--apiserver-endpoint` empty | Userdata missing cluster info | Check launch template userdata |
| Certificate error | Wrong CA data | Verify CA from `aws eks describe-cluster` |
| Cluster name mismatch | Typo in bootstrap args | Check `--b64-cluster-ca` and `--apiserver-endpoint` |

### Check Userdata

```sh
# View the userdata that ran
curl -s http://169.254.169.254/latest/user-data

# For managed node groups, it should contain:
# /etc/eks/bootstrap.sh <cluster-name>

# For self-managed, check the full bootstrap command
cat /var/lib/cloud/instance/user-data.txt
```

## Step 4: Check Kubelet

```sh
# Is kubelet running?
systemctl status kubelet

# Kubelet logs (most useful)
journalctl -u kubelet --no-pager -n 100

# Common kubelet error patterns:
journalctl -u kubelet | grep -i "error\|failed\|unable\|unauthorized\|forbidden"
```

### Common Kubelet Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Unable to register node` | API server unreachable | Check networking (step 5) |
| `Unauthorized` | IAM role not in aws-auth | Add role to aws-auth (step 6) |
| `x509: certificate signed by unknown authority` | Wrong CA certificate | Update CA in kubelet config |
| `dial tcp: lookup ... no such host` | DNS resolution failing | Check VPC DNS settings |
| `dial tcp <ip>:443: i/o timeout` | Can't reach API server endpoint | Check SG, NACLs, route tables |
| `node not found` | Node registered then removed | Check aws-auth permissions |
| `TLS handshake timeout` | Network timeout to API server | Check endpoint access config |

## Step 5: Check Networking

The node must be able to reach the EKS API server. The path depends on endpoint configuration:

### Public Endpoint (Default)

```
Node → Internet → NLB (public) → API server
```

Requirements:
- Node subnet has a route to the internet (NAT Gateway or public IP)
- Security group allows outbound HTTPS (443)
- No NACL blocking outbound

### Private Endpoint

```
Node → ENI in VPC → API server
```

Requirements:
- Node can resolve the API server DNS name to private IPs
- Security group allows traffic to the cluster security group on port 443
- VPC DNS must be enabled

### Verification

```sh
# From inside the node:

# 1. Can you resolve the API server DNS?
ENDPOINT=$(cat /var/lib/kubelet/kubeconfig | grep server | awk '{print $2}' | sed 's|https://||')
nslookup $ENDPOINT

# 2. Can you reach port 443?
curl -k https://$ENDPOINT/healthz
# Should return "ok" or a certificate error (both mean network works)

# 3. Test with timeout
timeout 5 bash -c "echo > /dev/tcp/$ENDPOINT/443" && echo "Port open" || echo "Port blocked"
```

### Security Group Checks

```sh
# Get the cluster security group
CLUSTER_SG=$(aws eks describe-cluster --name <cluster> --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)

# Get the node security group
NODE_SG=$(aws ec2 describe-instances --instance-ids <instance-id> \
  --query "Reservations[0].Instances[0].SecurityGroups[].GroupId" --output text)

# Check cluster SG allows inbound from node SG
aws ec2 describe-security-groups --group-ids $CLUSTER_SG \
  --query "SecurityGroups[0].IpPermissions" --output json

# Check node SG allows outbound to cluster SG on 443
aws ec2 describe-security-groups --group-ids $NODE_SG \
  --query "SecurityGroups[0].IpPermissionsEgress" --output json
```

Required security group rules:

| Direction | From | To | Port | Purpose |
|-----------|------|----|----|---------|
| Inbound on cluster SG | Node SG | Cluster SG | 443 | Node → API server |
| Inbound on cluster SG | Node SG | Cluster SG | 10250 | API server → kubelet |
| Outbound on node SG | Node SG | 0.0.0.0/0 | 443 | HTTPS to API server, ECR, STS |
| Outbound on node SG | Node SG | Cluster SG | 1025-65535 | Node → pods on other nodes |

### Check NACLs

```sh
# Get subnet NACLs
aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=<subnet-id>" \
  --query "NetworkAcls[0].Entries" --output table
```

NACLs are stateless — you need both inbound and outbound rules for the return traffic.

### Check Route Tables

```sh
# Get routes for the node's subnet
aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<subnet-id>" \
  --query "RouteTables[0].Routes" --output table
```

For private subnets reaching the public API endpoint, you need:
- `0.0.0.0/0 → NAT Gateway`

For private endpoint:
- No internet route needed (traffic stays in VPC)

### DNS Resolution

```sh
# From the node, check DNS config
cat /etc/resolv.conf

# The nameserver should be the VPC DNS (base of VPC CIDR + 2)
# For VPC 10.0.0.0/16, DNS is 10.0.0.2

# Test resolution
nslookup <cluster-endpoint-hostname>

# If private endpoint: should resolve to private IPs in your VPC
# If public endpoint: should resolve to public IPs
```

If DNS fails for private endpoint:
- Ensure `enableDnsSupport` and `enableDnsHostnames` are true on the VPC
- Check if the Route 53 private hosted zone is associated with the VPC

## Step 6: Check IAM Authentication

The node's instance profile IAM role must be mapped in the cluster:

### Check the Node's IAM Role

```sh
# Get the instance profile role
aws ec2 describe-instances --instance-ids <instance-id> \
  --query "Reservations[0].Instances[0].IamInstanceProfile.Arn" --output text

# Resolve to the role ARN
PROFILE_NAME=$(aws ec2 describe-instances --instance-ids <instance-id> \
  --query "Reservations[0].Instances[0].IamInstanceProfile.Arn" --output text | awk -F/ '{print $NF}')

ROLE_ARN=$(aws iam get-instance-profile --instance-profile-name $PROFILE_NAME \
  --query "InstanceProfile.Roles[0].Arn" --output text)

echo "Node role: $ROLE_ARN"
```

### Check aws-auth ConfigMap

```sh
kubectl get configmap aws-auth -n kube-system -o yaml
```

The node's role must be listed in `mapRoles`:

```yaml
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/NodeInstanceRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
```

### Add Missing Role

```sh
# Using eksctl
eksctl create iamidentitymapping --cluster <cluster> --region <region> \
  --arn $ROLE_ARN \
  --username "system:node:{{EC2PrivateDNSName}}" \
  --group system:bootstrappers --group system:nodes

# Or edit manually
kubectl edit configmap aws-auth -n kube-system
```

### Check Access Entries (Newer Method)

```sh
aws eks list-access-entries --cluster-name <cluster> --region <region>

# The node role should have an access entry of type EC2_LINUX
aws eks describe-access-entry --cluster-name <cluster> --region <region> \
  --principal-arn $ROLE_ARN
```

### Required IAM Policies on the Node Role

The node role must have these AWS managed policies:

| Policy | Purpose |
|--------|---------|
| `AmazonEKSWorkerNodePolicy` | Core EKS node permissions |
| `AmazonEKS_CNI_Policy` | VPC CNI pod networking |
| `AmazonEC2ContainerRegistryReadOnly` | Pull images from ECR |
| `AmazonSSMManagedInstanceCore` | (Optional) SSM access for debugging |

```sh
# Verify attached policies
aws iam list-attached-role-policies --role-name <node-role-name> --output table
```

## Step 7: Check API Server Authenticator Logs

If the node is reaching the API server but being rejected:

```sh
# Enable authenticator logging (if not already)
aws eks update-cluster-config --name <cluster> --region <region> \
  --logging '{"clusterLogging":[{"types":["authenticator"],"enabled":true}]}'

# Check logs in CloudWatch
aws logs filter-log-events \
  --log-group-name /aws/eks/<cluster>/cluster \
  --log-stream-name-prefix authenticator \
  --filter-pattern "ERROR" \
  --start-time $(date -u -d '1 hour ago' '+%s000')
```

Common authenticator errors:
- `access denied` — role not in aws-auth
- `invalid bearer token` — STS endpoint unreachable from node
- `could not get caller identity` — instance profile missing or incorrect

## Step 8: Check Node Conditions

If the node appears but shows `NotReady`:

```sh
kubectl describe node <node-name> | grep -A 10 "Conditions:"

# Common NotReady causes:
# KubeletNotReady: container runtime not ready
# NetworkNotReady: CNI plugin not initialized
```

```sh
# On the node, check container runtime
systemctl status containerd

# Check CNI
ls /etc/cni/net.d/
ls /opt/cni/bin/

# Check aws-node (VPC CNI) pod
kubectl get pods -n kube-system -l k8s-app=aws-node -o wide
```

## Common Scenarios

### Scenario 1: Self-Managed Nodes After Cluster Creation

Problem: You created the cluster with `--without-nodegroup` and manually launched instances that don't join.

Fix: Ensure the bootstrap script passes the correct cluster name, endpoint, and CA:

```sh
#!/bin/bash
/etc/eks/bootstrap.sh <cluster-name> \
  --apiserver-endpoint <endpoint-url> \
  --b64-cluster-ca <base64-ca-data>
```

Get the values:
```sh
aws eks describe-cluster --name <cluster> --query "cluster.{Endpoint:endpoint, CA:certificateAuthority.data}" --output json
```

### Scenario 2: Managed Node Group Stuck in CREATE_FAILED

```sh
# Check node group health issues
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng> \
  --query "nodegroup.health.issues" --output json
```

Common issues:
- `AccessDenied` — node role missing required policies
- `InsufficientFreeAddresses` — subnet is full
- `ClusterUnreachable` — API server can't reach nodes (SG issue)

### Scenario 3: Nodes Join Then Become NotReady

```sh
# Check kubelet logs for recurring errors
journalctl -u kubelet --since "5 minutes ago" | grep -i "error"

# Check if the node is being drained/cordoned
kubectl get node <node-name> -o jsonpath='{.spec.unschedulable}'
```

Common causes:
- VPC CNI running out of IPs (subnet exhaustion)
- kubelet resource pressure (disk, memory, PID)
- containerd crashing

### Scenario 4: Node Joins But Wrong Hostname

EKS expects the node name to be the EC2 private DNS name. If `--hostname-override` is set incorrectly:

```sh
# Check what name the node registered with
kubectl get nodes

# Compare with the instance's private DNS
aws ec2 describe-instances --instance-ids <id> --query "Reservations[0].Instances[0].PrivateDnsName" --output text
```

Don't set `--hostname-override` unless you know what you're doing.

## Diagnostic One-Liner

Run this from a node that won't join to quickly identify the issue:

```sh
echo "=== Instance Identity ===" && \
curl -s http://169.254.169.254/latest/meta-data/instance-id && echo && \
echo "=== IAM Role ===" && \
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/ && echo && \
echo "=== Kubelet Status ===" && \
systemctl is-active kubelet && \
echo "=== Kubelet Errors (last 10) ===" && \
journalctl -u kubelet --no-pager -n 10 --priority=err && \
echo "=== API Server Reachability ===" && \
ENDPOINT=$(grep server /var/lib/kubelet/kubeconfig 2>/dev/null | awk '{print $2}' | sed 's|https://||') && \
timeout 5 bash -c "echo > /dev/tcp/$ENDPOINT/443" 2>/dev/null && echo "API server reachable" || echo "API server UNREACHABLE" && \
echo "=== DNS ===" && \
nslookup $ENDPOINT 2>&1 | head -5
```

## Summary Checklist

- [ ] Instance is running (check ASG activity)
- [ ] Correct AMI (EKS-optimized for the K8s version)
- [ ] Bootstrap script ran successfully (`/var/log/cloud-init-output.log`)
- [ ] Kubelet is running (`systemctl status kubelet`)
- [ ] API server is reachable (port 443 open, DNS resolves)
- [ ] Security groups allow node ↔ control plane traffic
- [ ] No NACL blocking traffic
- [ ] Route table has path to API server (NAT GW for public endpoint)
- [ ] Node IAM role is in aws-auth ConfigMap or Access Entries
- [ ] Node IAM role has required policies (EKSWorkerNodePolicy, CNI, ECR)
- [ ] VPC DNS is enabled (`enableDnsSupport`, `enableDnsHostnames`)
- [ ] Subnet has available IP addresses
- [ ] Container runtime (containerd) is running
- [ ] VPC CNI (aws-node) pod is healthy


## Cloud-Init Status Check

Before digging into kubelet, verify that cloud-init (which runs the bootstrap) completed successfully:

```sh
# Quick status check
cloud-init status --long

# If status is "error", check:
cat /run/cloud-init/result.json
cat /var/log/cloud-init-output.log
cat /var/log/cloud-init.log
```

If `cloud-init status` shows `done` but kubelet isn't running, the issue is post-bootstrap. If it shows `error`, the bootstrap script itself failed.

## Clock Skew

If kubelet logs show `certificate has expired` or `x509: certificate has expired or is not yet valid`, the instance clock is wrong:

```sh
# Check time
timedatectl

# Check NTP sync
chronyc tracking
# or
ntpstat

# Force sync
sudo chronyc makestep
```

Clock skew causes TLS certificate validation to fail. Common on instances that were stopped for a long time or have broken NTP.

## Interpreting API Server Connectivity

```sh
# From the node:
curl -sk https://<EKS_API_ENDPOINT>
```

| Response | Meaning |
|----------|---------|
| `401 Unauthorized` (JSON) | Network works, auth is the problem (aws-auth/IAM) |
| `Connection timed out` | Firewall/SG/NACL blocking, or wrong route |
| `Could not resolve host` | DNS failure |
| `Connection refused` | Endpoint exists but not listening (rare) |
| No response (hangs) | SG or NACL blocking, no route to host |

Getting a `401` is actually good — it means networking is fine and the problem is purely authentication.

## Remote Diagnosis via SSM (No Interactive Session)

If you can't SSH or start an interactive SSM session, use `send-command`:

```sh
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["cloud-init status --long && echo --- && systemctl status kubelet && echo --- && journalctl -u kubelet --no-pager -n 30"]' \
  --output text

# Get the command output
aws ssm get-command-invocation \
  --command-id <command-id> \
  --instance-id <instance-id> \
  --query "StandardOutputContent" --output text
```

## Additional Failure Scenarios

### Node Appears Then Disappears

The node joins briefly, then vanishes from `kubectl get nodes`:

```sh
# Check if the instance was terminated
aws ec2 describe-instances --instance-ids <id> --query "Reservations[0].Instances[0].State.Name" --output text

# Check ASG activity (scale-in, health check failure)
aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg> \
  --query "Activities[0:3].{Status:StatusCode, Cause:Cause, Time:StartTime}" --output table

# Check EC2 status events
aws ec2 describe-instance-status --instance-ids <id> --query "InstanceStatuses[0].Events" --output json
```

Common causes:
- ASG health check failing (instance health or ELB health)
- Spot interruption
- EC2 scheduled maintenance
- Node failing kubelet health checks → ASG replaces it

### Can't Pull Container Images

Node joins but pods fail with `ImagePullBackOff`:

```sh
# From the node, test ECR access
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com

# Check if node can reach ECR
curl -s https://<account>.dkr.ecr.<region>.amazonaws.com/v2/
```

Requires:
- `AmazonEC2ContainerRegistryReadOnly` on the node role
- Network path to ECR (internet via NAT, or VPC endpoint `com.amazonaws.<region>.ecr.dkr`)

## Troubleshooting Priority Order

In most cases, the issue falls into one of three categories. Check in this order:

1. **aws-auth / Access Entries** (most common) — Node IAM role not mapped or has a typo
2. **Network connectivity** — Security groups, routing, DNS preventing the node from reaching the API server
3. **Bootstrap failure** — Userdata error, wrong AMI, proxy misconfiguration, or missing permissions

If all three check out, look at:
4. Clock skew
5. Subnet IP exhaustion
6. Instance profile missing or incorrect
7. VPC CNI failures (post-join NotReady)


## VPC Endpoints for Private Clusters

If the cluster has no public endpoint and nodes are in private subnets without NAT, you need VPC endpoints for the node to bootstrap:

| Service | Endpoint Type | Required For |
|---------|:-------------:|--------------|
| `com.amazonaws.<region>.eks` | Interface | API server communication |
| `com.amazonaws.<region>.ecr.api` | Interface | ECR image metadata |
| `com.amazonaws.<region>.ecr.dkr` | Interface | ECR image pull |
| `com.amazonaws.<region>.s3` | Gateway | ECR image layers (stored in S3) |
| `com.amazonaws.<region>.sts` | Interface | IAM authentication (token exchange) |
| `com.amazonaws.<region>.ec2` | Interface | VPC CNI ENI management |
| `com.amazonaws.<region>.logs` | Interface | CloudWatch logging (optional) |
| `com.amazonaws.<region>.ssm` | Interface | SSM access for debugging (optional) |
| `com.amazonaws.<region>.ssmmessages` | Interface | SSM Session Manager (optional) |

```sh
# Check existing VPC endpoints
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=<vpc-id>" \
  --query "VpcEndpoints[].{Service:ServiceName, State:State}" --output table

# Create a missing endpoint (example: STS)
aws ec2 create-vpc-endpoint \
  --vpc-id <vpc-id> \
  --service-name com.amazonaws.<region>.sts \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-aaa subnet-bbb \
  --security-group-ids <sg-allowing-443> \
  --private-dns-enabled
```

Without the STS endpoint, the node can't exchange its instance profile credentials for a token → kubelet gets `Unauthorized`.

## Max Pods Exceeded

If the node joined but new pods stay `Pending` with `Too many pods`:

```sh
# Check node pod capacity
kubectl describe node <node-name> | grep -A 3 "Capacity:"
kubectl describe node <node-name> | grep -A 3 "Allocatable:"

# Check how many pods are running
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node-name> --no-headers | wc -l
```

Max pods is determined by the instance type's ENI and IP limits:

```sh
# Check ENI/IP capacity for instance type
aws ec2 describe-instance-types --instance-types <type> \
  --query "InstanceTypes[0].NetworkInfo.{MaxENI:MaximumNetworkInterfaces, IPv4PerENI:Ipv4AddressesPerInterface}" --output table
```

Fix:
- Enable prefix delegation (`ENABLE_PREFIX_DELEGATION=true`) to increase max pods
- Use a larger instance type
- Set `--max-pods` in the bootstrap script to match the actual capacity

## IMDSv2 Hop Limit

If the IMDS hop limit is set to 1 (default), containers running inside pods **cannot** access instance metadata. This breaks:
- VPC CNI (aws-node) — needs IMDS to manage ENIs
- Pod Identity Agent
- Any pod using instance profile credentials (legacy)

```sh
# Check current hop limit
aws ec2 describe-instances --instance-ids <id> \
  --query "Reservations[0].Instances[0].MetadataOptions.HttpPutResponseHopLimit" --output text

# Fix: set hop limit to 2 (required for containers)
aws ec2 modify-instance-metadata-options --instance-id <id> --http-put-response-hop-limit 2
```

For managed node groups, set the hop limit in the launch template:

```json
{
  "MetadataOptions": {
    "HttpTokens": "required",
    "HttpPutResponseHopLimit": 2,
    "HttpEndpoint": "enabled"
  }
}
```

If the VPC CNI pod is crash-looping with `unable to get instance metadata`, this is almost certainly the cause.

## Token Expiry / Stale Launch Template

The EKS bootstrap token (used by kubelet to authenticate during initial registration) is derived from instance credentials. If an instance sits in a launch template queue or starts from a very old AMI:

- Certificates or API endpoints in hardcoded userdata may be stale
- If using a custom AMI with a baked kubeconfig, the token may have expired

Fix:
- Always use `aws eks describe-cluster` in the bootstrap script (dynamic, not hardcoded)
- Don't bake cluster-specific config into AMIs
- Use `latest` in the launch template version reference

```sh
# Verify the bootstrap is using current cluster info
grep apiserver-endpoint /var/lib/kubelet/kubeconfig
# Should match: aws eks describe-cluster --name <cluster> --query "cluster.endpoint"
```

## kube-proxy Not Running

Node appears in `kubectl get nodes` but pod networking is broken (pods can't reach Services):

```sh
# Check kube-proxy DaemonSet
kubectl get ds kube-proxy -n kube-system
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide

# Check if kube-proxy is running on the specific node
kubectl get pods -n kube-system -l k8s-app=kube-proxy --field-selector spec.nodeName=<node-name>

# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy --field-selector spec.nodeName=<node-name>
```

If kube-proxy isn't scheduled on the node:
- Check for taints on the node that kube-proxy doesn't tolerate
- Check if the kube-proxy DaemonSet has a nodeSelector that doesn't match
- Verify the kube-proxy add-on is installed: `aws eks list-addons --cluster-name <cluster>`

## Bottlerocket-Specific Debugging

Bottlerocket doesn't have a traditional userdata/cloud-init flow. Debugging is different:

```sh
# Bottlerocket uses a control container for admin access
# Connect via SSM to the control container
aws ssm start-session --target <instance-id>

# From the control container, enter the admin container
enter-admin-container

# Check kubelet logs (different path)
journalctl -u kubelet

# Check bootstrap settings
cat /etc/kubernetes/kubelet/config

# Bottlerocket settings (TOML-based)
apiclient get settings.kubernetes

# Check node configuration
apiclient get settings.kubernetes.cluster-name
apiclient get settings.kubernetes.api-server
apiclient get settings.kubernetes.cluster-certificate
```

Bottlerocket bootstrap settings are in `/etc/bottlerocket/config.toml` or set via the API. Common issues:
- `settings.kubernetes.cluster-name` is wrong or empty
- `settings.kubernetes.api-server` doesn't match the actual endpoint
- `settings.kubernetes.cluster-certificate` has wrong CA data

For managed node groups with Bottlerocket, these are set automatically. For self-managed, pass them via userdata in TOML format:

```toml
[settings.kubernetes]
cluster-name = "my-cluster"
api-server = "https://XXXX.gr7.us-east-1.eks.amazonaws.com"
cluster-certificate = "LS0tLS1CR..."
```


## Node Pressure Conditions (Quick Checks)

```sh
# Memory pressure across all nodes
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="MemoryPressure")].status}{"\n"}{end}'

# Disk pressure
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="DiskPressure")].status}{"\n"}{end}'

# PID pressure
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="PIDPressure")].status}{"\n"}{end}'

# All conditions at once
kubectl get nodes -o custom-columns=NAME:.metadata.name,READY:.status.conditions[3].status,MEM:.status.conditions[0].status,DISK:.status.conditions[1].status,PID:.status.conditions[2].status
```

## Check OOM Kills on a Node

```sh
# Using kubectl debug (no SSH needed)
kubectl debug node/<node-name> -it --image=alpine -- sh -c "dmesg | grep -i 'killed process\|out of memory'"

# Check disk usage
kubectl debug node/<node-name> -it --image=alpine -- sh -c "df -h"

# Check container log disk usage
kubectl debug node/<node-name> -it --image=alpine -- sh -c "du -sh /var/log/containers/"
```

## Check Kubelet Certificate Expiration

```sh
# From inside the node (SSH or kubectl debug)
kubectl debug node/<node-name> -it --image=alpine -- sh

# Inside:
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -text -noout | grep -A2 "Validity"

# If expired: remove and restart kubelet
rm /var/lib/kubelet/pki/kubelet-client*
systemctl restart kubelet
```

## Container Runtime Deep Dive

```sh
# Access node
kubectl debug node/<node-name> -it --image=alpine -- sh

# Inside:
# Check containerd config
cat /etc/containerd/config.toml

# Container resource usage stats
crictl stats

# Inspect a specific container
crictl inspect <container-id>

# Check for failed/exited containers
crictl ps -a --state exited
```

## Network Stack Analysis

```sh
# From inside a node (kubectl debug or SSH):

# Check network interfaces
ip addr show
ip route show

# Check iptables rules (kube-proxy)
iptables -L -n -v | head -50
iptables -t nat -L -n -v | head -50

# Monitor traffic to API server
tcpdump -i eth0 host <api-server-ip> -c 20
```

## QoS Class Distribution

```sh
# Check which QoS class each pod has (affects eviction order)
kubectl get pods -A -o custom-columns=NAME:.metadata.name,NS:.metadata.namespace,QOS:.status.qosClass

# Count by QoS class
kubectl get pods -A -o jsonpath='{range .items[*]}{.status.qosClass}{"\n"}{end}' | sort | uniq -c
```

BestEffort pods are evicted first under memory pressure. If most pods are BestEffort (no requests/limits), they'll be killed randomly.

## Emergency: Cluster Recovery

```sh
# 1. Quick assessment
kubectl cluster-info
kubectl get nodes
kubectl get pods -A --field-selector status.phase!=Running | grep -v Completed

# 2. Cordon failing nodes (stop scheduling)
kubectl cordon <node-name>

# 3. Drain safely
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 4. If ASG-managed, terminate and let ASG replace
aws ec2 terminate-instances --instance-ids <id>

# 5. Or increase capacity first
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name <asg-name> \
  --desired-capacity <higher-count>
```

## Emergency: Backup Critical Data

```sh
# Before replacing nodes, backup cluster state
kubectl get all -A -o yaml > cluster-backup.yaml
kubectl get pv,pvc -A -o yaml > storage-backup.yaml
kubectl get secrets,configmaps -A -o yaml > config-backup.yaml
```

## When to Escalate

Contact AWS Support when:

- Multiple nodes failing simultaneously across AZs
- API server becomes unresponsive
- Persistent storage issues affecting data integrity
- Network connectivity issues affecting entire cluster
- IAM/Security issues preventing all node registration
- EKS control plane showing degraded status in AWS console


## VPC CNI (aws-node) Troubleshooting

### VPC CNI Log Location

```sh
# On the node (via SSH or kubectl debug):
ls /var/log/aws-routed-eni/
# ipamd.log.*    — IP address management daemon logs
# plugin.log.*   — CNI plugin invocation logs
```

### ipamD Debugging Endpoints

From inside the node, query the local ipamD API:

```sh
# Get ENI and IP allocation info
curl http://localhost:61679/v1/enis | python3 -m json.tool
# Shows: AssignedIPs, ENIIPPools, TotalIPs, each IP's assignment status

# Get pod-to-IP mapping
curl http://localhost:61679/v1/pods | python3 -m json.tool
# Shows: which pod has which IP on which ENI

# Get ipamD metrics (Prometheus format)
curl http://localhost:61678/metrics
# Key metrics:
# awscni_assigned_ip_addresses — IPs currently assigned to pods
# awscni_eni_allocated — ENIs currently attached
# awscni_eni_max — maximum ENIs for this instance type
```

### Collect Support Bundle

```sh
# Run the built-in support script (on the node)
/opt/cni/bin/aws-cni-support.sh
# Generates: /var/log/eks_<instance-id>_<date>.tar.gz
```

### Pods Stuck in ContainerCreating (IP Exhaustion)

If pods stay in `ContainerCreating`, the subnet may be out of IPs:

```sh
# Check subnet available IPs
aws ec2 describe-subnets --subnet-ids <subnet-id> \
  --query "Subnets[0].AvailableIpAddressCount" --output text

# Check how many IPs the node has allocated
curl -s http://localhost:61679/v1/enis | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Assigned: {d[\"AssignedIPs\"]}, Total: {d[\"TotalIPs\"]}')"
```

Fix: Add more subnets, use larger CIDRs, or enable prefix delegation (`ENABLE_PREFIX_DELEGATION=true`).

### Large Cluster: EC2 API Throttling During Burst Scaling

When scaling from 0 to many pods simultaneously, all nodes' ipamD tries to allocate ENIs at once. EC2 API throttling causes some nodes to back off exponentially, leaving pods in `ContainerCreating`.

Mitigation: Set a higher `WARM_ENI_TARGET` (default 1) to pre-allocate ENIs before pods arrive:

```sh
kubectl set env daemonset aws-node -n kube-system WARM_ENI_TARGET=2
```

### FORWARD Policy Issue (Non-EKS AMIs)

If pods can't ping each other on custom AMIs, check if iptables FORWARD is set to DROP:

```sh
iptables -L FORWARD -n
# If Policy is DROP:
iptables -P FORWARD ACCEPT
```

EKS-optimized AMIs set this automatically. Custom AMIs need it added to the bootstrap script.

### Known Issues

| Issue | Symptom | Fix |
|-------|---------|-----|
| `systemd-udev` MAC change (Ubuntu 22.04+) | Pod connectivity breaks after ENI setup | Set `MACAddressPolicy=none` in `/usr/lib/systemd/network/99-default.link` |
| `NetworkManager-cloud-setup` (RHEL 8+) | Pod networking breaks (routing table 30200/30400 present) | Disable `nm-cloud-setup.service` |
| nftables incompatibility | IPAMD errors on RHEL 8+/Ubuntu 21+ | Set `ENABLE_NFTABLES=true` (v1.12.1+) or upgrade to v1.13.1+ (auto-detected) |
| Missing `ENABLE_IPv4` env var (v1.10+) | aws-node crashes with nil pointer dereference | Apply the full manifest from the release, not just the image update |


## Full Diagnostic Script

Save and run on a node that won't join:

```sh
#!/bin/bash
# eks-node-diagnostics.sh — run as root on failing node

echo "=== EKS Node Join Diagnostics ==="
echo "Time: $(date)"
echo ""

echo "--- Instance Info ---"
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)
PRIVATE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
echo "Instance: $INSTANCE_ID | Region: $REGION | IP: $PRIVATE_IP"

echo ""
echo "--- IAM Role ---"
ROLE=$(curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/)
[ -z "$ROLE" ] && echo "ERROR: No IAM role attached" || echo "Role: $ROLE"

echo ""
echo "--- Kubelet ---"
systemctl is-active kubelet && echo "Kubelet: Active" || echo "Kubelet: INACTIVE"
kubelet --version 2>/dev/null

echo ""
echo "--- Kubelet Errors (last 10) ---"
journalctl -u kubelet --no-pager -n 10 --priority=err 2>/dev/null

echo ""
echo "--- API Server Connectivity ---"
if [ -f /var/lib/kubelet/kubeconfig ]; then
  ENDPOINT=$(grep server /var/lib/kubelet/kubeconfig | awk '{print $2}' | sed 's|https://||')
  echo "Endpoint: $ENDPOINT"
  timeout 5 bash -c "echo > /dev/tcp/$ENDPOINT/443" 2>/dev/null && echo "Port 443: OPEN" || echo "Port 443: BLOCKED"
  HTTP_CODE=$(curl -sk -o /dev/null -w "%{http_code}" "https://$ENDPOINT/version" 2>/dev/null)
  echo "HTTP response: $HTTP_CODE"
else
  echo "kubeconfig not found — bootstrap may not have run"
fi

echo ""
echo "--- Container Runtime ---"
systemctl is-active containerd 2>/dev/null && echo "containerd: Active" || echo "containerd: INACTIVE"
crictl version 2>/dev/null

echo ""
echo "--- Network ---"
echo "Default route: $(ip route | grep default)"
echo "DNS: $(grep nameserver /etc/resolv.conf | head -1)"

echo ""
echo "--- Disk ---"
df -h / | tail -1

echo ""
echo "--- Cloud-init ---"
cloud-init status 2>/dev/null

echo ""
echo "=== Done ==="
```

Usage:

```sh
chmod +x eks-node-diagnostics.sh
sudo ./eks-node-diagnostics.sh
```

## Attach Missing IAM Instance Profile

If a node has no IAM role (metadata returns empty):

```sh
# Attach an instance profile to a running instance
aws ec2 associate-iam-instance-profile \
  --instance-id <instance-id> \
  --iam-instance-profile Name=<instance-profile-name>

# Then restart kubelet on the node
sudo systemctl restart kubelet
```

## When to Replace vs Fix

| Replace the node | Fix the node |
|:----------------:|:------------:|
| Production environment | Dev/test environment |
| Node is in an ASG (auto-replaced) | Standalone node (no ASG) |
| Issue persists >15 minutes | Need to understand root cause |
| Faster than debugging | Custom configuration to preserve |

In most cases, **replace is faster and safer** — terminate the instance and let the ASG launch a new one with the correct configuration.

## Root Cause Statistics

From production EKS clusters, most node join failures are:

| Cause | Frequency |
|:-----:|:---------:|
| IAM role not in aws-auth | ~50% |
| Security group misconfiguration | ~30% |
| Network/subnet issues | ~10% |
| Incorrect bootstrap/AMI | ~10% |
