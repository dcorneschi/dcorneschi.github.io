# Pulling Images from ECR with ctr and Docker

How to authenticate and pull container images from Amazon ECR using `ctr` (containerd CLI) and Docker.

## The Problem

ECR requires authentication before pulling images. Unlike Docker which handles login natively, `ctr` (the containerd CLI) needs the credentials passed directly on the fetch command.

## Solution 1: Using ctr (containerd)

```bash
# Get ECR authentication token
ECR_TOKEN=$(aws ecr get-login-password --region us-east-1)

# Pull image using ctr with ECR credentials
ctr image pull --user "AWS:${ECR_TOKEN}" 602401143452.dkr.ecr.us-east-1.amazonaws.com/your-image:tag

# Fetch (download without unpacking)
ctr fetch --user "AWS:${ECR_TOKEN}" 602401143452.dkr.ecr.us-east-1.amazonaws.com/your-image:tag

# Pull into a specific namespace (e.g., k8s.io for Kubernetes)
ctr -n k8s.io image pull --user "AWS:${ECR_TOKEN}" 602401143452.dkr.ecr.us-east-1.amazonaws.com/your-image:tag
```

### Notes on ctr

- The token is valid for 12 hours
- Use `--user AWS:<token>` format (AWS is always the username for ECR)
- The `-n k8s.io` namespace is needed if you want the image visible to kubelet

## Solution 2: Using Docker

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin 602401143452.dkr.ecr.us-east-1.amazonaws.com

# Pull the image
docker pull 602401143452.dkr.ecr.us-east-1.amazonaws.com/eks/pause:3.5

# Pull from a private repository
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

## Solution 3: Using crictl (CRI-compatible)

```bash
# crictl uses the kubelet's credential provider (if configured)
# Or configure credentials in /etc/containerd/config.toml

# Pull directly with crictl
crictl pull 602401143452.dkr.ecr.us-east-1.amazonaws.com/eks/pause:3.5

# If auth is needed, configure containerd's ECR credential helper
```

## Solution 4: Using nerdctl

```bash
# Login (similar to Docker)
aws ecr get-login-password --region us-east-1 | \
    nerdctl login --username AWS --password-stdin 602401143452.dkr.ecr.us-east-1.amazonaws.com

# Pull
nerdctl pull 602401143452.dkr.ecr.us-east-1.amazonaws.com/your-image:tag
```

## ECR Registry URL Format

```bash
# Format: <account-id>.dkr.ecr.<region>.amazonaws.com/<repository>:<tag>

# AWS-managed EKS images (public)
602401143452.dkr.ecr.us-east-1.amazonaws.com/eks/pause:3.5
602401143452.dkr.ecr.us-east-1.amazonaws.com/amazon-k8s-cni:v1.15.0

# Your private repositories
123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

## Token Refresh Script

The ECR token expires after 12 hours. Use a script or cron job to refresh:

```bash
#!/bin/bash
# refresh-ecr-token.sh
REGION="${1:-us-east-1}"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGISTRY="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"

aws ecr get-login-password --region "$REGION" | \
    docker login --username AWS --password-stdin "$REGISTRY"

echo "ECR login refreshed for $REGISTRY"
```

```bash
# Add to crontab (refresh every 10 hours)
0 */10 * * * /path/to/refresh-ecr-token.sh us-east-1
```

## Troubleshooting

### "no basic credentials"

```bash
# Token expired — regenerate
aws ecr get-login-password --region us-east-1

# Check if credentials are configured
cat ~/.docker/config.json | grep ecr
```

### "denied: Your authorization token has expired"

```bash
# Token lasts 12 hours — re-authenticate
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin <registry-url>
```

### "repository does not exist"

```bash
# Verify the repository exists
aws ecr describe-repositories --region us-east-1

# Check the exact image name
aws ecr list-images --repository-name my-app --region us-east-1
```

### ctr pull hangs or times out

```bash
# Check containerd is running
systemctl status containerd

# Verify network connectivity to ECR
curl -s https://602401143452.dkr.ecr.us-east-1.amazonaws.com/v2/ -o /dev/null -w "%{http_code}"

# Check if ECR VPC endpoint is needed (private subnets)
aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.us-east-1.ecr.dkr
```
