# EKS VPC CNI Proxy Configuration

How proxy settings work for the aws-node DaemonSet (VPC CNI plugin) in EKS clusters — NO_PROXY rules, critical endpoints, private vs public EKS endpoints, and troubleshooting.

## How Proxy Configuration Works

When you configure proxy settings via ConfigMap for aws-node:

```yaml
HTTP_PROXY=http://proxy.example.com:3128
HTTPS_PROXY=http://proxy.example.com:3128
NO_PROXY=169.254.169.254,10.0.0.0/8,*.eks.amazonaws.com
```

### Traffic Routing Rules

**Simple rule: NO_PROXY = Direct, Everything Else = Proxy**

| Match | Routing | Example |
|-------|---------|---------|
| Matches NO_PROXY | Direct connection (bypass proxy) | `169.254.169.254`, `10.x.x.x` |
| Does NOT match NO_PROXY | Goes through proxy | `ec2.us-west-2.amazonaws.com` |

## Critical Endpoints for aws-node

### Must Always Be in NO_PROXY

| Endpoint | Address | Purpose |
|----------|---------|---------|
| EC2 Metadata (IMDS) | `169.254.169.254` | IAM credentials, instance identity, ENI info |
| VPC CIDR | `10.0.0.0/8`, `172.16.0.0/12` | Pod-to-pod, internal VPC traffic |
| K8s internal DNS | `.svc`, `.cluster.local` | Service discovery |
| Localhost | `127.0.0.1`, `localhost` | Local health checks |

> **Critical:** If `169.254.169.254` goes through the proxy, aws-node will fail to get IAM credentials and crash.

### EKS Endpoint Configuration

#### Public EKS Endpoint

Can be in NO_PROXY (recommended) or routed through proxy:

```bash
# Option A: Direct (recommended — lower latency, no proxy overhead)
NO_PROXY=169.254.169.254,...,<cluster-id>.eks.amazonaws.com

# Option B: Through proxy (works, adds latency, useful for auditing)
# Proxy must allow HTTPS to *.eks.amazonaws.com
```

#### Private EKS Endpoint

**MUST be in NO_PROXY — this is critical:**

```bash
NO_PROXY=169.254.169.254,...,<cluster-id>.eks.amazonaws.com
```

Why:
- Private endpoint resolves to a private IP inside your VPC
- Routing through proxy breaks the connection (proxy can't reach private IPs)
- Cluster becomes non-functional if not in NO_PROXY

## Recommended NO_PROXY Configurations

### Minimal

```bash
NO_PROXY=169.254.169.254,localhost,127.0.0.1,.svc,.cluster.local,10.0.0.0/8,172.16.0.0/12
```

### With EKS Endpoint (Public or Private)

```bash
NO_PROXY=169.254.169.254,localhost,127.0.0.1,.svc,.cluster.local,10.0.0.0/8,172.16.0.0/12,<eks-endpoint>.eks.amazonaws.com
```

### With VPC Endpoints

```bash
NO_PROXY=169.254.169.254,localhost,127.0.0.1,.svc,.cluster.local,10.0.0.0/8,172.16.0.0/12,*.eks.amazonaws.com,*.ec2.amazonaws.com,*.vpce.amazonaws.com
```

### Comprehensive

```bash
NO_PROXY=169.254.169.254,localhost,127.0.0.1,.svc,.cluster.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,*.eks.amazonaws.com,*.ec2.amazonaws.com
```

## NO_PROXY Pattern Matching

| Pattern | Matches | Example |
|---------|---------|---------|
| `*.eks.amazonaws.com` | Any subdomain | `abc123.gr7.us-west-2.eks.amazonaws.com` |
| `10.0.0.0/8` | Entire CIDR range | `10.1.2.3` |
| `169.254.169.254` | Exact IP | Only that IP |
| `.svc` | Domain suffix | `myapp.default.svc` |
| `localhost` | Exact match | Only `localhost` |

Rules:
- Comma-separated (no spaces)
- Case-insensitive
- No protocol prefixes (`https://` is wrong)

## Terraform: Extract EKS Endpoint for NO_PROXY

```hcl
# EKS endpoint comes as: https://ABC123.gr7.us-west-2.eks.amazonaws.com
# NO_PROXY needs hostname only: ABC123.gr7.us-west-2.eks.amazonaws.com

locals {
  eks_endpoint_hostname = replace(data.aws_eks_cluster.eks.endpoint, "https://", "")
  
  no_proxy = join(",", [
    "169.254.169.254",
    "localhost",
    "127.0.0.1",
    ".svc",
    ".cluster.local",
    "10.0.0.0/8",
    "172.16.0.0/12",
    local.eks_endpoint_hostname,
  ])
}
```

## ConfigMap Example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: proxy-config
  namespace: kube-system
data:
  HTTP_PROXY: "http://proxy.example.com:3128"
  HTTPS_PROXY: "http://proxy.example.com:3128"
  NO_PROXY: "169.254.169.254,localhost,127.0.0.1,.svc,.cluster.local,10.0.0.0/8,172.16.0.0/12,abc123.gr7.us-west-2.eks.amazonaws.com"
```

## DaemonSet Configuration

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: aws-node
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: aws-node
        envFrom:
        - configMapRef:
            name: proxy-config
```

## Verifying Proxy Configuration

### Check ConfigMap and DaemonSet

```bash
# Check if DaemonSet references a proxy ConfigMap
kubectl get daemonset aws-node -n kube-system -o yaml | grep -A 10 envFrom

# List ConfigMaps that might contain proxy config
kubectl get configmap -n kube-system | grep proxy

# View ConfigMap contents
kubectl get configmap proxy-config -n kube-system -o yaml
```

### Check Environment in Running Pods

```bash
# List all environment variables
kubectl exec -n kube-system <aws-node-pod> -- printenv

# Filter proxy variables
kubectl exec -n kube-system <aws-node-pod> -- printenv | grep -i proxy

# Get from pod spec (JSON)
kubectl get pod -n kube-system <aws-node-pod> -o json | \
    jq '.spec.containers[0].env[] | select(.name | contains("PROXY"))'
```

### Check aws-node Logs

```bash
# Recent logs
kubectl logs -n kube-system -l k8s-app=aws-node --tail=100

# Connection errors
kubectl logs -n kube-system -l k8s-app=aws-node | grep -i "error\|failed\|timeout"

# Metadata/ENI issues
kubectl logs -n kube-system -l k8s-app=aws-node | grep -i "metadata\|eni\|connection"
```

### Verify Cluster Functionality

```bash
# Check pods are getting IPs
kubectl get pods -A -o wide

# Check aws-node status
kubectl get pods -n kube-system -l k8s-app=aws-node

# Describe for events
kubectl describe daemonset aws-node -n kube-system
```

## Troubleshooting

### aws-node Pods Failing to Start

**Symptoms:** CrashLoopBackOff, logs show "failed to get instance metadata"

```bash
# Verify IMDS is in NO_PROXY
kubectl exec -n kube-system <aws-node-pod> -- printenv NO_PROXY | grep 169.254.169.254

# Test metadata access from the node
curl -s http://169.254.169.254/latest/meta-data/instance-id
```

**Fix:** Ensure `169.254.169.254` is in NO_PROXY.

### Pods Not Getting IP Addresses

**Symptoms:** New pods stuck in ContainerCreating, logs show "failed to attach ENI"

```bash
# Check aws-node logs
kubectl logs -n kube-system -l k8s-app=aws-node | grep -i "attach\|eni\|error"

# Verify EC2 API is reachable
kubectl exec -n kube-system <aws-node-pod> -- \
    curl -s -o /dev/null -w "%{http_code}" https://ec2.us-west-2.amazonaws.com/
```

**Fix:** Verify IAM permissions and that EC2 API is reachable (through proxy or directly).

### Private Endpoint Not Accessible

**Symptoms:** kubectl commands timeout, aws-node can't reach K8s API

```bash
# Check if EKS endpoint is in NO_PROXY
kubectl exec -n kube-system <aws-node-pod> -- printenv NO_PROXY

# Verify endpoint hostname
aws eks describe-cluster --name my-cluster --query 'cluster.endpoint' --output text
```

**Fix:** Add the EKS endpoint hostname (without `https://`) to NO_PROXY.

### Proxy Not Taking Effect

**Symptoms:** No HTTP_PROXY in pod environment

```bash
# Verify ConfigMap exists and has data
kubectl get configmap proxy-config -n kube-system -o yaml

# Verify DaemonSet references it
kubectl get daemonset aws-node -n kube-system -o yaml | grep -A 5 envFrom

# Restart aws-node pods after ConfigMap change
kubectl rollout restart daemonset aws-node -n kube-system
```

**Fix:** Ensure the ConfigMap is referenced via `envFrom` in the DaemonSet and pods are restarted.

## Key Takeaways

1. **NO_PROXY = Direct connection, everything else = proxy**
2. **Always include:** `169.254.169.254`, VPC CIDRs, `.svc`, `.cluster.local`
3. **Private EKS endpoints MUST be in NO_PROXY** (will break otherwise)
4. **Public EKS endpoints should be in NO_PROXY** for performance
5. **NO_PROXY uses hostnames without protocol** (no `https://`)
6. **ConfigMap must be referenced in the DaemonSet** for proxy to work
7. **Restart aws-node pods** after changing proxy ConfigMap
