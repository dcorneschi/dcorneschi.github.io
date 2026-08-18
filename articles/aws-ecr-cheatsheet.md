# AWS ECR Cheatsheet

Amazon Elastic Container Registry (ECR) — private and public container image registries with scanning, lifecycle policies, replication, and pull-through cache.

## Authentication

```bash
# Login to private ECR (valid for 12 hours)
aws ecr get-login-password --region eu-west-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.eu-west-1.amazonaws.com

# Login to ECR Public
aws ecr-public get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin public.ecr.aws

# Verify login
docker info | grep Registry
```

## Repositories

### Private Repositories

```bash
# Create repository
aws ecr create-repository --repository-name my-app --image-tag-mutability IMMUTABLE --image-scanning-configuration scanOnPush=true

# Create with encryption (custom KMS key)
aws ecr create-repository \
  --repository-name my-app \
  --encryption-configuration encryptionType=KMS,kmsKey=arn:aws:kms:eu-west-1:123456789012:key/key-id

# List repositories
aws ecr describe-repositories
aws ecr describe-repositories --query 'repositories[].{Name:repositoryName,URI:repositoryUri}' --output table

# Get repository URI
aws ecr describe-repositories --repository-names my-app --query 'repositories[0].repositoryUri' --output text

# Delete repository (including all images)
aws ecr delete-repository --repository-name my-app --force
```

### Public Repositories

```bash
# Create public repository
aws ecr-public create-repository --repository-name my-public-app

# List public repositories
aws ecr-public describe-repositories

# Delete public repository
aws ecr-public delete-repository --repository-name my-public-app --force
```

## Push and Pull Images

```bash
# Tag image for ECR
docker tag my-app:latest 123456789012.dkr.ecr.eu-west-1.amazonaws.com/my-app:latest
docker tag my-app:latest 123456789012.dkr.ecr.eu-west-1.amazonaws.com/my-app:v1.2.3

# Push image
docker push 123456789012.dkr.ecr.eu-west-1.amazonaws.com/my-app:latest
docker push 123456789012.dkr.ecr.eu-west-1.amazonaws.com/my-app:v1.2.3

# Pull image
docker pull 123456789012.dkr.ecr.eu-west-1.amazonaws.com/my-app:latest

# Push to public ECR
docker tag my-app:latest public.ecr.aws/a1b2c3d4/my-public-app:latest
docker push public.ecr.aws/a1b2c3d4/my-public-app:latest
```

## Image Management

```bash
# List images in repository
aws ecr list-images --repository-name my-app
aws ecr list-images --repository-name my-app --query 'imageIds[].imageTag' --output table

# Describe images (size, push date, scan status)
aws ecr describe-images --repository-name my-app
aws ecr describe-images --repository-name my-app \
  --query 'imageDetails[].{Tag:imageTags[0],Size:imageSizeInBytes,Pushed:imagePushedAt,Scan:imageScanStatus.status}' --output table

# Delete specific image by tag
aws ecr batch-delete-image --repository-name my-app --image-ids imageTag=old-tag

# Delete image by digest
aws ecr batch-delete-image --repository-name my-app --image-ids imageDigest=sha256:abc123

# Delete untagged images
aws ecr list-images --repository-name my-app --filter tagStatus=UNTAGGED --query 'imageIds[*]' --output json | \
  jq -c '.' | xargs -I{} aws ecr batch-delete-image --repository-name my-app --image-ids '{}'
```

## Image Scanning

```bash
# Enable scan on push (at repository creation)
aws ecr create-repository --repository-name my-app --image-scanning-configuration scanOnPush=true

# Enable scan on push (existing repository)
aws ecr put-image-scanning-configuration --repository-name my-app --image-scanning-configuration scanOnPush=true

# Trigger manual scan
aws ecr start-image-scan --repository-name my-app --image-id imageTag=latest

# Get scan findings
aws ecr describe-image-scan-findings --repository-name my-app --image-id imageTag=latest

# Get finding counts by severity
aws ecr describe-image-scan-findings --repository-name my-app --image-id imageTag=latest \
  --query 'imageScanFindings.findingSeverityCounts'
```

### Enhanced Scanning (Inspector)

```bash
# Enable enhanced scanning (uses Amazon Inspector)
aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{"repositoryFilters":[{"filter":"*","filterType":"WILDCARD"}],"scanFrequency":"SCAN_ON_PUSH"}]'

# Check scanning configuration
aws ecr get-registry-scanning-configuration
```

## Lifecycle Policies

Automatically clean up old images to control storage costs:

```bash
# Set lifecycle policy
aws ecr put-lifecycle-policy --repository-name my-app --lifecycle-policy-text '{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "Delete untagged images older than 1 day",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 1
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 3,
      "description": "Delete all images older than 90 days",
      "selection": {
        "tagStatus": "any",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 90
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}'

# Get lifecycle policy
aws ecr get-lifecycle-policy --repository-name my-app

# Preview what lifecycle policy would delete (must start preview first)
aws ecr start-lifecycle-policy-preview --repository-name my-app
aws ecr get-lifecycle-policy-preview --repository-name my-app

# Delete lifecycle policy
aws ecr delete-lifecycle-policy --repository-name my-app
```

## Replication

```bash
# Enable cross-region replication
aws ecr put-replication-configuration --replication-configuration '{
  "rules": [
    {
      "destinations": [
        {"region": "us-west-2", "registryId": "123456789012"},
        {"region": "eu-central-1", "registryId": "123456789012"}
      ],
      "repositoryFilters": [
        {"filter": "prod-", "filterType": "PREFIX_MATCH"}
      ]
    }
  ]
}'

# Cross-account replication
aws ecr put-replication-configuration --replication-configuration '{
  "rules": [
    {
      "destinations": [
        {"region": "eu-west-1", "registryId": "999999999999"}
      ]
    }
  ]
}'

# Get replication config
aws ecr describe-registry
```

## Pull-Through Cache

Cache images from public registries (Docker Hub, GitHub, Quay) in your private ECR:

```bash
# Create pull-through cache rule for Docker Hub
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix docker-hub \
  --upstream-registry-url registry-1.docker.io \
  --credential-arn arn:aws:secretsmanager:eu-west-1:123456789012:secret:docker-hub-creds

# Create rule for GitHub Container Registry
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix ghcr \
  --upstream-registry-url ghcr.io

# Create rule for Quay
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix quay \
  --upstream-registry-url quay.io

# Pull through cache (image is pulled and cached automatically)
docker pull 123456789012.dkr.ecr.eu-west-1.amazonaws.com/docker-hub/library/nginx:latest

# List cache rules
aws ecr describe-pull-through-cache-rules

# Delete cache rule
aws ecr delete-pull-through-cache-rule --ecr-repository-prefix docker-hub
```

## Repository Policies

```bash
# Allow cross-account pull
aws ecr set-repository-policy --repository-name my-app --policy-text '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::999999999999:root"
      },
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability"
      ]
    }
  ]
}'

# Get repository policy
aws ecr get-repository-policy --repository-name my-app

# Delete repository policy
aws ecr delete-repository-policy --repository-name my-app
```

## Tag Immutability

```bash
# Enable immutable tags (prevent overwriting)
aws ecr put-image-tag-mutability --repository-name my-app --image-tag-mutability IMMUTABLE

# Disable (allow overwriting tags like 'latest')
aws ecr put-image-tag-mutability --repository-name my-app --image-tag-mutability MUTABLE
```

## Registry Settings

```bash
# Get registry ID
aws ecr describe-registry

# Set registry scanning configuration
aws ecr put-registry-scanning-configuration --scan-type BASIC --rules '[]'

# Set registry policy (for replication permissions)
aws ecr put-registry-policy --policy-text file://registry-policy.json
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `no basic auth credentials` | Run `aws ecr get-login-password` and pipe to `docker login` |
| `denied: requested access to the resource is denied` | Check IAM permissions (ecr:GetAuthorizationToken, ecr:BatchGetImage) |
| `image not found` | Verify repository exists and image tag is correct |
| `token has expired` | Re-authenticate (token lasts 12 hours) |
| Pull-through cache 403 | Add upstream registry credentials via Secrets Manager |
| `tag already exists` (immutable) | Use a different tag or set MUTABLE |
| Large images slow to push | Use `--max-concurrent-uploads` in Docker daemon config |

## Best Practices

- Enable **scan on push** for vulnerability detection
- Use **immutable tags** for production images (prevent tag overwriting)
- Set **lifecycle policies** to auto-delete old/untagged images
- Use **pull-through cache** to avoid Docker Hub rate limits
- Enable **cross-region replication** for multi-region deployments
- Use **repository policies** for cross-account access (instead of making repos public)
- Tag images with git SHA + semantic version (e.g., `v1.2.3` and `abc1234`)
- Never use only `latest` — it makes rollbacks impossible
