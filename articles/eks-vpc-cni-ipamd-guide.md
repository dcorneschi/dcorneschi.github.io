# EKS VPC CNI: IPAMD and Port 50051

How the AWS VPC CNI plugin manages IP addresses on EKS nodes — IPAMD architecture, gRPC health checks, probe configuration, metrics, and troubleshooting.

## Architecture Overview

The VPC CNI plugin has three components:

| Component | Role | Runs As |
|-----------|------|---------|
| CNI Binary | Configures pod networking (called by kubelet) | Binary on host |
| IPAMD (aws-node) | Manages ENIs and IP pool, serves gRPC on port 50051 | DaemonSet (hostNetwork) |
| aws-vpc-cni-init | Installs CNI binaries, sets up prerequisites | Init container (privileged) |

## What IPAMD Does

The IPAMD (IP Address Management Daemon) runs as the `aws-node` DaemonSet and is responsible for:

1. **ENI Management** — attaching/detaching Elastic Network Interfaces to the node
2. **IP Pool Management** — maintaining a warm pool of pre-allocated IP addresses
3. **gRPC Server** — responding to CNI binary requests for IPs via gRPC on port 50051
4. **AWS API Calls** — calling EC2 APIs to attach ENIs, assign secondary IPs, describe interfaces
5. **IP Allocation** — assigning IPs from the warm pool when pods are created
6. **IP Recycling** — returning IPs to the warm pool (30-second cooldown) when pods are deleted

## Port 50051 — IPAMD gRPC

Port 50051 is the gRPC interface where the CNI binary communicates with IPAMD:

```
kubelet creates pod
  → invokes CNI binary
    → CNI binary sends gRPC request to localhost:50051
      → IPAMD allocates IP from warm pool
        → CNI binary configures pod networking
```

## Health Probes

Both liveness and readiness probes verify IPAMD is responsive:

```bash
/app/grpc-health-probe -addr=:50051 -connect-timeout=5s -rpc-timeout=5s
```

The probe:
- Connects to IPAMD's gRPC port
- Sends a standard gRPC health check request
- Returns success if status is `SERVING`
- Returns failure on timeout or connection error

### Liveness Probe Configuration

```yaml
livenessProbe:
  exec:
    command:
      - /app/grpc-health-probe
      - -addr=:50051
      - -connect-timeout=5s
      - -rpc-timeout=5s
  initialDelaySeconds: 60
  periodSeconds: 10
  timeoutSeconds: 60
  failureThreshold: 3
  successThreshold: 1
```

### Readiness Probe Configuration

```yaml
readinessProbe:
  exec:
    command:
      - /app/grpc-health-probe
      - -addr=:50051
      - -connect-timeout=5s
      - -rpc-timeout=5s
  initialDelaySeconds: 1
  periodSeconds: 10
  timeoutSeconds: 10
  failureThreshold: 3
  successThreshold: 1
```

### Older Versions (Caution)

Older VPC CNI versions used `timeoutSeconds: 1` for the liveness probe, which caused false positives under load. AWS now recommends `timeoutSeconds: 60` for liveness.

## Ports Reference

| Port | Service | Purpose |
|------|---------|---------|
| 50051 | IPAMD gRPC | CNI binary ↔ IPAMD communication + health probes |
| 61678 | Prometheus metrics | VPC CNI metrics endpoint |
| 8162 | Network policy agent | Metrics (if network policy is enabled) |

## Metrics (Port 61678)

```bash
# Access metrics from inside the pod
kubectl exec -n kube-system <aws-node-pod> -- curl -s http://localhost:61678/metrics
```

Key metrics:

| Metric | Description |
|--------|-------------|
| `awscni_assigned_ip_addresses` | IPs currently assigned to pods |
| `awscni_eni_allocated` | Number of ENIs attached to the node |
| `awscni_total_ip_addresses` | Total IPs managed by IPAMD |
| `awscni_eni_max` | Maximum ENIs the instance type supports |
| `awscni_ip_max` | Maximum IPs the instance type supports |
| `awscni_aws_api_latency_ms` | Latency of AWS API calls |
| `awscni_aws_api_error_count` | Count of failed AWS API calls |

## Why Port 50051 Health Checks Fail

### 1. IPAMD Overloaded

- High CPU/memory usage on the node
- AWS API throttling (too many EC2 API calls)
- Too many concurrent pod launches
- Resource contention on the node

### 2. IAM Permission Issues

```bash
# Required policy: AmazonEKS_CNI_Policy
# Key permissions needed:
# - ec2:DescribeNetworkInterfaces
# - ec2:AttachNetworkInterface
# - ec2:AssignPrivateIpAddresses
# - ec2:CreateNetworkInterface
# - ec2:DeleteNetworkInterface
# - ec2:DetachNetworkInterface
# - ec2:UnassignPrivateIpAddresses

# Check if CNI has the right IAM role
kubectl describe daemonset aws-node -n kube-system | grep -i iam
kubectl describe sa aws-node -n kube-system
```

### 3. IP Exhaustion

- Subnet out of available IPs
- No contiguous /28 blocks (if using prefix delegation)
- IPAMD unable to allocate new IPs

```bash
# Check subnet available IPs
aws ec2 describe-subnets --subnet-ids subnet-xxx \
    --query 'Subnets[].{CIDR:CidrBlock,Available:AvailableIpAddressCount}'
```

### 4. Node Resource Pressure

- PID limit exhausted
- Memory pressure causing OOM
- Disk space exhausted
- High CPU making gRPC responses slow

### 5. Network/Configuration Issues

- Custom networking misconfiguration
- Security group blocking internal communication
- ENI attachment failures (instance ENI limit reached)

## Troubleshooting

### Check Probe Configuration

```bash
# View current probe settings
kubectl get daemonset aws-node -n kube-system -o yaml | \
    grep -A 15 "livenessProbe\|readinessProbe"
```

### Check IPAMD Logs

```bash
# From kubectl
kubectl logs -n kube-system -l k8s-app=aws-node --tail=100

# From the node directly
cat /var/log/aws-routed-eni/ipamd.log

# Look for throttling or errors
kubectl logs -n kube-system -l k8s-app=aws-node | grep -i "error\|throttl\|timeout"
```

### Test gRPC Health Manually

```bash
# From inside the aws-node pod
kubectl exec -n kube-system <aws-node-pod> -- \
    /app/grpc-health-probe -addr=:50051

# Expected success output:
# status: SERVING

# Expected failure output:
# timeout: failed to connect service ":50051" within 5s
```

### Check Port is Listening

```bash
# From the node (via SSH or SSM)
ss -tlnp | grep 50051

# Check IPAMD process is running
ps aux | grep ipamd
```

### Check Metrics for Issues

```bash
# Get current metrics
kubectl exec -n kube-system <aws-node-pod> -- \
    curl -s http://localhost:61678/metrics | grep awscni

# Check for API errors
kubectl exec -n kube-system <aws-node-pod> -- \
    curl -s http://localhost:61678/metrics | grep aws_api_error
```

### Run AWS CNI Support Script

```bash
# On the node (collects diagnostic info)
sudo bash /opt/cni/bin/aws-cni-support.sh
```

### Check IP Capacity

```bash
# Check node allocatable
kubectl describe node <NODE_NAME> | grep -A 10 "Allocated resources"

# Check ENI and IP limits for instance type
aws ec2 describe-instance-types --instance-types m5.large \
    --query 'InstanceTypes[].NetworkInfo.{MaxENIs:MaximumNetworkInterfaces,IPv4PerENI:Ipv4AddressesPerInterface}'
```

## Environment Variables (Tuning)

Key environment variables on the `aws-node` DaemonSet:

| Variable | Default | Description |
|----------|---------|-------------|
| `WARM_IP_TARGET` | None | Number of free IPs to keep in the warm pool |
| `WARM_ENI_TARGET` | 1 | Number of free ENIs to keep attached |
| `MINIMUM_IP_TARGET` | None | Minimum IPs to allocate regardless of usage |
| `WARM_PREFIX_TARGET` | None | Prefixes to keep warm (prefix delegation) |
| `ENABLE_PREFIX_DELEGATION` | false | Use /28 prefix delegation for more IPs |
| `AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG` | false | Use custom networking (ENIConfig) |
| `AWS_VPC_ENI_MTU` | 9001 | MTU for ENI interfaces |
| `DISABLE_METRICS` | false | Disable Prometheus metrics |

```bash
# View current settings
kubectl get daemonset aws-node -n kube-system -o yaml | grep -A 2 "name: WARM_"

# Modify warm pool settings
kubectl set env daemonset aws-node -n kube-system \
    WARM_IP_TARGET=5 \
    MINIMUM_IP_TARGET=2
```

## Best Practices

- Use `timeoutSeconds: 60` for the liveness probe (not 10 or 1)
- Monitor VPC CNI metrics on port 61678 with Prometheus/Grafana
- Ensure the CNI IAM role has `AmazonEKS_CNI_Policy` attached
- Configure warm pool settings based on pod churn rate
- Monitor AWS API throttling (check `awscni_aws_api_error_count`)
- Keep VPC CNI plugin updated to the latest version
- Check IPAMD logs at `/var/log/aws-routed-eni/ipamd.log` on nodes
- Size subnets appropriately (use /19 for heavy workloads)
- Consider prefix delegation for high pod density
