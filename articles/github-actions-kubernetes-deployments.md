# GitHub Actions for Kubernetes Deployments

CI/CD patterns for building container images and deploying to Kubernetes clusters with GitHub Actions — multi-stage workflows, ECR/GHCR push, EKS authentication, manifest deployment, Helm releases, and security best practices.

## Basic Build → Push → Deploy Pipeline

```yaml
name: Deploy to Kubernetes

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write      # For OIDC auth to AWS
      contents: read

    steps:
    - uses: actions/checkout@v4

    - name: Build Docker image
      run: docker build -t my-app:${{ github.sha }} .

    - name: Push to registry
      run: |
        docker tag my-app:${{ github.sha }} registry/my-app:${{ github.sha }}
        docker push registry/my-app:${{ github.sha }}

    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/my-app app=registry/my-app:${{ github.sha }}
        kubectl rollout status deployment/my-app --timeout=300s
```

## Authenticating to AWS (EKS)

### OIDC (Recommended — No Long-Lived Credentials)

```yaml
    - name: Configure AWS credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/GitHubActionsDeployRole
        aws-region: us-east-1
        # No access keys needed — uses GitHub's OIDC token

    - name: Update kubeconfig
      run: aws eks update-kubeconfig --name my-cluster --region us-east-1
```

**IAM Role Trust Policy for GitHub OIDC:**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
      }
    }
  }]
}
```

**Required IAM Permissions:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["eks:DescribeCluster"],
      "Resource": "arn:aws:eks:us-east-1:123456789:cluster/my-cluster"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    }
  ]
}
```

### Access Keys (Legacy — Use OIDC Instead)

```yaml
    - name: Configure AWS credentials (keys)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1
```

## Pushing to Container Registries

### Amazon ECR

```yaml
    - name: Login to ECR
      id: ecr-login
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build and push to ECR
      env:
        REGISTRY: ${{ steps.ecr-login.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $REGISTRY/my-app:$IMAGE_TAG .
        docker build -t $REGISTRY/my-app:latest .
        docker push $REGISTRY/my-app:$IMAGE_TAG
        docker push $REGISTRY/my-app:latest
```

### GitHub Container Registry (GHCR)

```yaml
    permissions:
      packages: write
      contents: read

    steps:
    - name: Login to GHCR
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Build and push to GHCR
      run: |
        docker build -t ghcr.io/${{ github.repository }}/my-app:${{ github.sha }} .
        docker push ghcr.io/${{ github.repository }}/my-app:${{ github.sha }}
```

### Docker Hub

```yaml
    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Build and push
      run: |
        docker build -t myuser/my-app:${{ github.sha }} .
        docker push myuser/my-app:${{ github.sha }}
```

### Multi-Platform Build (Docker Buildx)

```yaml
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build and push (multi-arch)
      uses: docker/build-push-action@v5
      with:
        push: true
        platforms: linux/amd64,linux/arm64
        tags: |
          ${{ env.REGISTRY }}/my-app:${{ github.sha }}
          ${{ env.REGISTRY }}/my-app:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

## Deploying to Kubernetes

### Pattern 1: kubectl apply (Manifests)

```yaml
    - name: Deploy manifests
      run: |
        # Replace image tag in manifests
        sed -i "s|IMAGE_TAG|${{ github.sha }}|g" k8s/deployment.yaml

        kubectl apply -f k8s/
        kubectl rollout status deployment/my-app -n production --timeout=300s
```

### Pattern 2: kubectl set image (In-Place Update)

```yaml
    - name: Update deployment image
      run: |
        kubectl set image deployment/my-app \
          app=${{ env.REGISTRY }}/my-app:${{ github.sha }} \
          -n production
        kubectl rollout status deployment/my-app -n production --timeout=300s
```

### Pattern 3: Kustomize

```yaml
    - name: Deploy with Kustomize
      run: |
        cd k8s/overlays/production
        kustomize edit set image my-app=${{ env.REGISTRY }}/my-app:${{ github.sha }}
        kustomize build . | kubectl apply -f -
        kubectl rollout status deployment/my-app -n production --timeout=300s
```

### Pattern 4: Helm

```yaml
    - name: Deploy with Helm
      run: |
        helm upgrade --install my-app ./charts/my-app \
          --namespace production \
          --set image.repository=${{ env.REGISTRY }}/my-app \
          --set image.tag=${{ github.sha }} \
          --wait --timeout 300s
```

### Pattern 5: ArgoCD (GitOps — Update Manifest, ArgoCD Syncs)

```yaml
    - name: Update image tag in GitOps repo
      run: |
        git clone https://x-access-token:${{ secrets.GITOPS_TOKEN }}@github.com/myorg/gitops-repo.git
        cd gitops-repo
        
        # Update kustomization or values file
        cd apps/my-app/overlays/production
        kustomize edit set image my-app=${{ env.REGISTRY }}/my-app:${{ github.sha }}
        
        git config user.name "github-actions"
        git config user.email "actions@github.com"
        git add .
        git commit -m "Deploy my-app ${{ github.sha }}"
        git push

    # ArgoCD detects the change and syncs automatically
```

## Complete Production Workflow

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  EKS_CLUSTER: production
  ECR_REPOSITORY: my-app

jobs:
  # ──────────────────────────────────────────────
  # Job 1: Build, Test, Scan
  # ──────────────────────────────────────────────
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      packages: write
    outputs:
      image: ${{ steps.build.outputs.image }}

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/GitHubActionsBuild
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to ECR
      id: ecr-login
      uses: aws-actions/amazon-ecr-login@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build and push
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        push: ${{ github.event_name == 'push' }}
        tags: |
          ${{ steps.ecr-login.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          ${{ steps.ecr-login.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Run Trivy vulnerability scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ steps.ecr-login.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'

  # ──────────────────────────────────────────────
  # Job 2: Deploy to Staging
  # ──────────────────────────────────────────────
  deploy-staging:
    needs: build
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: staging
    permissions:
      id-token: write
      contents: read

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/GitHubActionsDeploy
        aws-region: ${{ env.AWS_REGION }}

    - name: Update kubeconfig
      run: aws eks update-kubeconfig --name staging --region ${{ env.AWS_REGION }}

    - name: Deploy to staging
      run: |
        helm upgrade --install my-app ./charts/my-app \
          --namespace staging \
          --set image.repository=${{ needs.build.outputs.image }} \
          --set image.tag=${{ github.sha }} \
          --set environment=staging \
          --wait --timeout 300s

    - name: Run smoke tests
      run: |
        kubectl wait pods -l app=my-app -n staging --for=condition=Ready --timeout=120s
        STAGING_URL=$(kubectl get ingress my-app -n staging -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
        curl -sf "http://$STAGING_URL/health" || exit 1

  # ──────────────────────────────────────────────
  # Job 3: Deploy to Production (Manual Approval)
  # ──────────────────────────────────────────────
  deploy-production:
    needs: [build, deploy-staging]
    runs-on: ubuntu-latest
    environment: production          # Requires manual approval in GitHub settings
    permissions:
      id-token: write
      contents: read

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/GitHubActionsDeployProd
        aws-region: ${{ env.AWS_REGION }}

    - name: Update kubeconfig
      run: aws eks update-kubeconfig --name ${{ env.EKS_CLUSTER }} --region ${{ env.AWS_REGION }}

    - name: Deploy to production
      run: |
        helm upgrade --install my-app ./charts/my-app \
          --namespace production \
          --set image.repository=$(aws ecr describe-repositories --repository-names ${{ env.ECR_REPOSITORY }} --query 'repositories[0].repositoryUri' --output text) \
          --set image.tag=${{ github.sha }} \
          --set environment=production \
          --set replicaCount=3 \
          --wait --timeout 300s

    - name: Verify deployment
      run: |
        kubectl rollout status deployment/my-app -n production --timeout=300s
        kubectl get pods -l app=my-app -n production
```

## Environment Protection Rules

Configure in GitHub repository Settings → Environments:

| Environment | Protection | Use Case |
|-------------|-----------|----------|
| `staging` | None (auto-deploy) | Every push to main deploys here |
| `production` | Required reviewers + wait timer | Manual approval before production |

```yaml
# In the job:
environment: production    # Triggers the protection rules
```

## Caching Strategies

### Docker Layer Cache (GitHub Actions Cache)

```yaml
    - uses: docker/build-push-action@v5
      with:
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

### Helm Chart Cache

```yaml
    - name: Cache Helm charts
      uses: actions/cache@v4
      with:
        path: ~/.cache/helm
        key: helm-${{ hashFiles('charts/**/Chart.lock') }}
```

### kubectl/tools Cache

```yaml
    - name: Cache tools
      uses: actions/cache@v4
      with:
        path: |
          /usr/local/bin/kubectl
          /usr/local/bin/helm
          /usr/local/bin/kustomize
        key: tools-v1
```

## Rollback on Failure

```yaml
    - name: Deploy
      id: deploy
      run: |
        helm upgrade --install my-app ./charts/my-app \
          --namespace production \
          --set image.tag=${{ github.sha }} \
          --wait --timeout 300s

    - name: Rollback on failure
      if: failure() && steps.deploy.outcome == 'failure'
      run: |
        echo "Deployment failed, rolling back..."
        helm rollback my-app -n production
        # Or: kubectl rollout undo deployment/my-app -n production
```

## Security Best Practices

| Practice | Implementation |
|----------|---------------|
| No long-lived credentials | Use OIDC (aws-actions/configure-aws-credentials with role-to-assume) |
| Least-privilege IAM | Role can only push to specific ECR repo + deploy to specific cluster |
| Pin action versions | `uses: actions/checkout@v4` not `@main` (supply chain safety) |
| Image scanning | Trivy/Snyk scan before deploy |
| Signed images | Cosign sign + verify in deploy step |
| Environment protection | Require approval for production |
| Branch protection | Only deploy from `main` (enforce with `if: github.ref == 'refs/heads/main'`) |
| Restrict OIDC subject | Trust policy condition limits to specific repo + branch |
| Immutable tags | Use git SHA as tag (never `:latest` in production) |
| Secrets in GitHub Secrets | Never hardcode credentials in workflows |

### Pin Actions by SHA (Most Secure)

```yaml
    - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1
    - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502  # v4.0.2
```

## Debugging Failed Deployments

```yaml
    - name: Debug on failure
      if: failure()
      run: |
        echo "=== Deployment Status ==="
        kubectl get deployment my-app -n production -o yaml
        echo "=== Pod Status ==="
        kubectl get pods -l app=my-app -n production
        echo "=== Recent Events ==="
        kubectl get events -n production --sort-by=.lastTimestamp | tail -20
        echo "=== Pod Logs ==="
        kubectl logs -l app=my-app -n production --tail=50 --all-containers
```

## Quick Reference

```yaml
# Key actions:
# aws-actions/configure-aws-credentials@v4  — OIDC auth to AWS
# aws-actions/amazon-ecr-login@v2           — ECR docker login
# docker/build-push-action@v5              — Build + push (multi-arch, caching)
# docker/login-action@v3                   — Login to any registry

# Auth pattern (OIDC, no secrets):
# 1. Create IAM role with GitHub OIDC trust policy
# 2. permissions: id-token: write
# 3. aws-actions/configure-aws-credentials with role-to-assume
# 4. aws eks update-kubeconfig

# Deploy patterns:
# kubectl apply -f k8s/                      — Raw manifests
# kubectl set image deployment/x app=img:tag — In-place update
# kustomize build . | kubectl apply -f -     — Kustomize
# helm upgrade --install x ./chart --wait    — Helm
# Update GitOps repo → ArgoCD syncs          — GitOps

# Image tagging:
# Always use ${{ github.sha }} (immutable, traceable)
# Never use :latest in production

# Environments:
# staging: auto-deploy on push to main
# production: require manual approval (environment protection rules)

# Rollback:
# helm rollback <release> -n <namespace>
# kubectl rollout undo deployment/<name> -n <namespace>
```
