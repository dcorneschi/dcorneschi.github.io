# ImagePullBackOff Troubleshooting Guide

Diagnosing and fixing `ImagePullBackOff` and `ErrImagePull` errors in Kubernetes — authentication, image names, registry connectivity, and pull secrets.

## What ImagePullBackOff Means

```
NAME       READY   STATUS             RESTARTS   AGE
myapp-abc  0/1     ImagePullBackOff   0          5m
```

The kubelet tried to pull the container image and failed. After the initial `ErrImagePull`, Kubernetes backs off exponentially (10s, 20s, 40s... up to 5 minutes) and keeps retrying — that's the "BackOff" phase.

## Quick Diagnosis

```bash
# 1. Check pod events for the exact error
kubectl describe pod <pod-name> | grep -A 10 Events

# 2. Look for the pull error message
kubectl get events --field-selector involvedObject.name=<pod-name> --sort-by='.lastTimestamp'
```

Common error messages:

| Error Message | Cause |
|--------------|-------|
| `manifest unknown` | Image tag doesn't exist |
| `unauthorized: authentication required` | Missing or wrong credentials |
| `dial tcp: lookup <registry>: no such host` | Registry hostname can't be resolved |
| `connection refused` or `i/o timeout` | Network can't reach registry |
| `403 Forbidden` | IAM/permissions deny access |
| `repository does not exist` | Wrong image name or repository |
| `toomanyrequests` | Docker Hub rate limit hit |

## Cause 1: Wrong Image Name or Tag

The most common cause — typo in the image name, tag, or registry URL.

```bash
# Check what image the pod is trying to pull
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].image}'
```

### Common Mistakes

```yaml
# Wrong: missing registry prefix
image: myapp:latest

# Wrong: typo in image name
image: docker.io/library/ngnix:latest

# Wrong: tag doesn't exist
image: nginx:1.99.0

# Wrong: wrong registry URL
image: 123456789012.dkr.ecr.us-east-2.amazonaws.com/myapp:v1
#                                    ^ wrong region
```

### Verify the Image Exists

```bash
# Docker Hub
docker manifest inspect nginx:1.25.0
crane manifest docker.io/library/nginx:1.25.0

# ECR
aws ecr describe-images --repository-name myapp --image-ids imageTag=v1

# List all tags in an ECR repository
aws ecr list-images --repository-name myapp --query 'imageIds[*].imageTag' --output table

# GCR / Artifact Registry
gcloud artifacts docker images list <region>-docker.pkg.dev/<project>/<repo>

# Generic registry with crane
crane ls <registry>/<repo>
```

## Cause 2: Authentication Required (Private Registry)

### Check if imagePullSecrets is Set

```bash
# Check pod spec
kubectl get pod <pod-name> -o jsonpath='{.spec.imagePullSecrets}'

# Check service account (may have imagePullSecrets attached)
kubectl get serviceaccount <sa-name> -o jsonpath='{.imagePullSecrets}'
```

### Create a Pull Secret

```bash
# For Docker Hub or generic registry
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<username> \
  --docker-password=<token> \
  --docker-email=<email>

# For a private registry
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=<username> \
  --docker-password=<password>
```

### Use the Secret in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: registry.example.com/myapp:v1
  imagePullSecrets:
  - name: regcred
```

### Attach to Service Account (Applies to All Pods)

```bash
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```

### Verify the Secret Works

```bash
# Decode and check the secret content
kubectl get secret regcred -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq

# Test pull manually (from a node or debug pod)
docker login <registry> -u <user> -p <password>
docker pull <image>
```

## Cause 3: ECR Authentication on EKS

ECR tokens expire every 12 hours. EKS nodes handle this automatically via the kubelet credential provider, but issues can arise.

### Verify ECR Access

```bash
# Check if the node role can access ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Check the node IAM role has the right policy
aws iam list-attached-role-policies --role-name <node-role-name>
# Must include: AmazonEC2ContainerRegistryReadOnly
```

### Common ECR Issues

| Issue | Fix |
|-------|-----|
| Missing `AmazonEC2ContainerRegistryReadOnly` on node role | Attach the policy |
| Cross-account ECR — no repository policy | Add cross-account policy to ECR repo |
| Wrong region in image URL | Fix the region in the image path |
| ECR endpoint not reachable (private subnet) | Add VPC endpoint for `com.amazonaws.<region>.ecr.dkr` and `com.amazonaws.<region>.ecr.api` |
| S3 endpoint missing (layers stored in S3) | Add VPC endpoint for `com.amazonaws.<region>.s3` |

### Cross-Account ECR Access

The source account's ECR repository needs a resource policy:

```bash
aws ecr set-repository-policy --repository-name myapp --policy-text '{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "CrossAccountPull",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::<target-account-id>:root"},
    "Action": [
      "ecr:GetDownloadUrlForLayer",
      "ecr:BatchGetImage",
      "ecr:BatchCheckLayerAvailability"
    ]
  }]
}'
```

### GCR (Google Container Registry)

```bash
# Create service account key
gcloud iam service-accounts keys create key.json \
  --iam-account <sa-name>@<project-id>.iam.gserviceaccount.com

# Create pull secret
kubectl create secret docker-registry gcr-secret \
  --docker-server=gcr.io \
  --docker-username=_json_key \
  --docker-password="$(cat key.json)"
```

### ACR (Azure Container Registry)

```bash
# Create pull secret with service principal
kubectl create secret docker-registry acr-secret \
  --docker-server=<registry-name>.azurecr.io \
  --docker-username=<service-principal-id> \
  --docker-password=<service-principal-password>

# Or attach ACR to AKS cluster (no secret needed)
az aks update --name <cluster> --resource-group <rg> --attach-acr <acr-name>
```

## Cause 4: Docker Hub Rate Limits

Docker Hub limits anonymous pulls to 100/6h and authenticated free pulls to 200/6h.

### Symptoms

```
Failed to pull image "docker.io/library/nginx:latest": toomanyrequests: You have reached your pull rate limit
```

### Check Your Rate Limit Status

```bash
TOKEN=$(curl -s "https://auth.docker.io/token?service=registry.docker.io&scope=repository:ratelimitpreview/test:pull" | jq -r .token)
curl -sI -H "Authorization: Bearer $TOKEN" https://registry-1.docker.io/v2/ratelimitpreview/test/manifests/latest | grep -i ratelimit
```

### Fixes

```bash
# Option 1: Authenticate to Docker Hub (increases limit)
kubectl create secret docker-registry dockerhub-cred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<username> \
  --docker-password=<access-token>

# Option 2: Mirror images to your own registry (ECR, GCR, etc.)
docker pull nginx:1.25
docker tag nginx:1.25 123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx:1.25
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx:1.25

# Option 3: Use ECR pull-through cache
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix docker-hub \
  --upstream-registry-url registry-1.docker.io
```

## Cause 5: Network Connectivity

The node can't reach the registry.

### Diagnose from the Node

```bash
# SSM into the node
aws ssm start-session --target <instance-id>

# Test DNS resolution
nslookup 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Test TCP connectivity
curl -I https://123456789012.dkr.ecr.us-east-1.amazonaws.com/v2/

# Check if NAT Gateway / Internet Gateway is working
curl -s https://checkip.amazonaws.com
```

### Common Network Issues

| Issue | Fix |
|-------|-----|
| Private subnet, no NAT Gateway | Add NAT Gateway or VPC endpoints |
| Security group blocks outbound HTTPS | Allow outbound TCP 443 |
| NACL blocks return traffic | Allow ephemeral ports inbound |
| VPC endpoint missing for ECR | Create `ecr.dkr`, `ecr.api`, and `s3` endpoints |
| DNS can't resolve registry | Check VPC DNS settings |
| Proxy not configured | Set `HTTP_PROXY`/`HTTPS_PROXY` in containerd config |

### Test from a Debug Pod

```bash
kubectl run net-test --image=busybox:1.36 --restart=Never --rm -it -- sh

# Inside the pod
wget -qO- https://registry-1.docker.io/v2/
nslookup registry-1.docker.io
```

## Cause 6: Image Platform Mismatch

Pulling an image built for a different architecture.

```bash
# Check image platforms
crane manifest <image> | jq '.manifests[].platform'

# Check node architecture
kubectl get node <node-name> -o jsonpath='{.status.nodeInfo.architecture}'
```

Common scenario: pulling an `amd64` image on an ARM (`arm64`) node or vice versa.

Fix: use multi-arch images or specify the platform:

```yaml
# Build multi-arch
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:v1 --push .
```

## Cause 7: Image Too Large / Disk Full

```bash
# Check node disk usage
kubectl describe node <node-name> | grep -A3 "DiskPressure"

# On the node
df -h /var/lib/containerd
```

If the disk is full, kubelet can't store the pulled image layers.

Fixes:
- Clean unused images: `sudo crictl rmi --prune`
- Expand root volume
- Use larger root volume in launch template

## Cause 8: Registry Certificate Errors

Symptom: `x509: certificate signed by unknown authority` or `certificate verify failed`

```bash
# Test from the node
curl -v https://<private-registry>/v2/

# Check certificate
openssl s_client -connect <registry>:443 -showcerts
```

Fixes:
- Add the CA certificate to the node's trust store
- For containerd, add to `/etc/containerd/certs.d/<registry>/ca.crt`
- For self-signed registries, configure containerd to skip TLS verification (not recommended for production)

```bash
# Add CA cert on the node (Amazon Linux / RHEL)
sudo cp ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

# Ubuntu
sudo cp ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# Restart containerd
sudo systemctl restart containerd
```

## Emergency Recovery

When you need to get a deployment working immediately:

```bash
# Rollback to previous working version
kubectl rollout undo deployment/<name>

# Force retry by deleting the pod
kubectl delete pod <pod-name>

# Scale down and up to retry all pods
kubectl scale deployment <name> --replicas=0
kubectl scale deployment <name> --replicas=3

# Patch deployment with a known-good image
kubectl set image deployment/<name> <container>=<working-image>

# Switch imagePullPolicy to IfNotPresent (if image is cached on node)
kubectl patch deployment <name> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container>","imagePullPolicy":"IfNotPresent"}]}}}}'
```

## Debug with crictl (On Node)

```bash
# SSM into the node
aws ssm start-session --target <instance-id>

# Check if image is already on the node
sudo crictl images | grep <image-name>

# Try pulling manually (shows detailed errors)
sudo crictl pull <full-image-name>

# Check containerd config for registry mirrors/auth
sudo cat /etc/containerd/config.toml | grep -A 10 "registry"
```

## BackOff Timing

After the initial `ErrImagePull`, Kubernetes retries with exponential backoff:

```
Attempt 1: immediate
Attempt 2: 10s delay
Attempt 3: 20s delay
Attempt 4: 40s delay
Attempt 5: 80s delay
Attempt 6: 160s delay
Attempt 7+: 300s (5 min cap)
```

The pod stays in `ImagePullBackOff` until the image pull succeeds or the pod is deleted.

## Troubleshooting Commands

```bash
# Full event history for the pod
kubectl describe pod <pod-name>

# Check image name
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].image}'

# Check imagePullSecrets
kubectl get pod <pod-name> -o jsonpath='{.spec.imagePullSecrets}'

# Check service account pull secrets
SA=$(kubectl get pod <pod-name> -o jsonpath='{.spec.serviceAccountName}')
kubectl get sa $SA -o jsonpath='{.imagePullSecrets}'

# Decode pull secret
kubectl get secret <secret-name> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq

# Check if image exists in ECR
aws ecr describe-images --repository-name <repo> --image-ids imageTag=<tag>

# Check node can reach registry (via SSM)
aws ssm start-session --target <instance-id>
curl -I https://<registry>/v2/

# Force re-pull (delete and recreate pod)
kubectl delete pod <pod-name>

# Check imagePullPolicy
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].imagePullPolicy}'
```

## imagePullPolicy Reference

| Policy | Behavior |
|--------|----------|
| `Always` | Pull every time the pod starts (default for `:latest` tag) |
| `IfNotPresent` | Pull only if image isn't already on the node (default for specific tags) |
| `Never` | Never pull — image must already exist on the node |

```yaml
containers:
- name: app
  image: myapp:v1.2.3
  imagePullPolicy: Always    # Force fresh pull every time
```

## Fix Checklist

```
□ Check exact error: kubectl describe pod <name>
□ Verify image name and tag are correct
□ Verify image exists in the registry
□ Check imagePullSecrets (pod spec or service account)
□ Verify registry credentials are valid (decode and test)
□ Check node IAM role has registry access (ECR)
□ Check network: node can reach registry (DNS + HTTPS)
□ Check VPC endpoints if private subnet (ECR: ecr.dkr, ecr.api, s3)
□ Check Docker Hub rate limits
□ Check architecture matches (amd64 vs arm64)
□ Check disk space on node
□ Delete pod to force immediate retry
```
