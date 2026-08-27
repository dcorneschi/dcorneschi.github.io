# Separate Kubeconfig Files for EKS Clusters

How to keep each EKS cluster in its own kubeconfig file instead of appending everything to `~/.kube/config` — cleaner management, easier sharing, and safer multi-cluster workflows.

## The Problem with the Default Behavior

```bash
# Default: appends to ~/.kube/config
aws eks update-kubeconfig --name my-cluster --region us-east-1
# Added new context arn:aws:eks:us-east-1:123456789:cluster/my-cluster to ~/.kube/config
```

Over time `~/.kube/config` becomes a monolith with dozens of clusters, stale entries, and merge conflicts. One accidental edit can break access to all clusters.

## Write to a Separate File

Use `--kubeconfig` to write to a dedicated file per cluster:

```bash
# Write to a separate file instead of ~/.kube/config:
aws eks update-kubeconfig \
  --name my-cluster \
  --region us-east-1 \
  --kubeconfig ~/.kube/eks-my-cluster.yaml
```

This creates `~/.kube/eks-my-cluster.yaml` containing only that cluster's config.

## Using the Separate File

### Option 1: KUBECONFIG Environment Variable

```bash
# Use for the current shell session:
export KUBECONFIG=~/.kube/eks-my-cluster.yaml
kubectl get nodes

# Use for a single command:
KUBECONFIG=~/.kube/eks-my-cluster.yaml kubectl get pods
```

### Option 2: --kubeconfig Flag

```bash
kubectl --kubeconfig ~/.kube/eks-my-cluster.yaml get nodes
```

### Option 3: Merge Multiple Files with KUBECONFIG

The `KUBECONFIG` variable accepts colon-separated paths. kubectl merges them at runtime without modifying any file:

```bash
# Merge two cluster configs at runtime:
export KUBECONFIG=~/.kube/eks-production.yaml:~/.kube/eks-staging.yaml

# Now both contexts are available:
kubectl config get-contexts
# CURRENT   NAME          CLUSTER                    NAMESPACE
# *         production    arn:aws:eks:...:production
#           staging       arn:aws:eks:...:staging

# Switch between them:
kubectl config use-context production
kubectl config use-context staging
```

### Option 4: Shell Function for Quick Switching

```bash
# Add to ~/.bashrc or ~/.zshrc:
kube() {
  export KUBECONFIG=~/.kube/eks-${1}.yaml
  echo "Switched to: $1"
  kubectl config current-context
}

# Usage:
kube production    # → export KUBECONFIG=~/.kube/eks-production.yaml
kube staging       # → export KUBECONFIG=~/.kube/eks-staging.yaml
```

## Setting Up Multiple Clusters

```bash
# Create separate configs for each cluster:
aws eks update-kubeconfig --name production --region us-east-1 \
  --kubeconfig ~/.kube/eks-production.yaml

aws eks update-kubeconfig --name staging --region us-east-1 \
  --kubeconfig ~/.kube/eks-staging.yaml

aws eks update-kubeconfig --name dev --region us-west-2 \
  --kubeconfig ~/.kube/eks-dev.yaml
```

### With a Custom Context Name (Alias)

By default, the context name is the full ARN. Use `--alias` for shorter names:

```bash
aws eks update-kubeconfig --name production --region us-east-1 \
  --kubeconfig ~/.kube/eks-production.yaml \
  --alias production

aws eks update-kubeconfig --name staging --region us-east-1 \
  --kubeconfig ~/.kube/eks-staging.yaml \
  --alias staging
```

Now `kubectl config current-context` shows `production` instead of `arn:aws:eks:us-east-1:123456789:cluster/production`.

### With a Specific IAM Role

```bash
aws eks update-kubeconfig --name production --region us-east-1 \
  --kubeconfig ~/.kube/eks-production.yaml \
  --alias production \
  --role-arn arn:aws:iam::123456789:role/EKSAdminRole
```

## Directory Structure

```
~/.kube/
├── config                    # Keep empty or minimal (optional default)
├── eks-production.yaml       # Production cluster
├── eks-staging.yaml          # Staging cluster
├── eks-dev.yaml              # Dev cluster
├── eks-tools.yaml            # Tools cluster
└── other-cluster.yaml        # Non-EKS clusters
```

## Automate with a Script

```bash
#!/bin/bash
# update-all-kubeconfigs.sh
# Refresh kubeconfig for all EKS clusters

CLUSTERS=(
  "production:us-east-1:EKSAdminRole"
  "staging:us-east-1:EKSAdminRole"
  "dev:us-west-2:"
)

for entry in "${CLUSTERS[@]}"; do
  IFS=':' read -r name region role <<< "$entry"
  file=~/.kube/eks-${name}.yaml

  echo "Updating: $name ($region) → $file"

  args=(
    --name "$name"
    --region "$region"
    --kubeconfig "$file"
    --alias "$name"
  )

  if [ -n "$role" ]; then
    args+=(--role-arn "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/$role")
  fi

  aws eks update-kubeconfig "${args[@]}"
done

echo "Done. Available configs:"
ls ~/.kube/eks-*.yaml
```

## KUBECONFIG in Shell Profile

Load all configs automatically on shell start:

```bash
# Add to ~/.bashrc or ~/.zshrc:

# Auto-discover all kubeconfig files in ~/.kube/
export KUBECONFIG=$(find ~/.kube -name "eks-*.yaml" -o -name "config" 2>/dev/null | tr '\n' ':')

# Or explicitly list them:
export KUBECONFIG=~/.kube/eks-production.yaml:~/.kube/eks-staging.yaml:~/.kube/eks-dev.yaml
```

Now all contexts are always available without manually setting KUBECONFIG.

## Advantages of Separate Files

| Concern | Single ~/.kube/config | Separate Files |
|---------|:--------------------:|:--------------:|
| Accidental deletion/corruption | Lose access to ALL clusters | Lose only one |
| Sharing a cluster config | Extract manually | Copy one file |
| Stale entries | Manual cleanup | Delete one file |
| Git-tracking configs | Awkward (secrets in one file) | Ignore specific files |
| CI/CD pipelines | Must filter contexts | Set KUBECONFIG to one file |
| Permission separation | All clusters same file permissions | Per-file ACLs possible |
| Merging on update | Can corrupt other entries | Isolated updates |

## CI/CD Usage

In pipelines, write the kubeconfig to a temporary location:

```bash
# GitHub Actions / GitLab CI:
aws eks update-kubeconfig --name production --region us-east-1 \
  --kubeconfig /tmp/kubeconfig

export KUBECONFIG=/tmp/kubeconfig
kubectl apply -f manifests/
```

No risk of contaminating a shared config file.

## Troubleshooting

```bash
# Check which kubeconfig is active:
echo $KUBECONFIG

# If KUBECONFIG is empty, kubectl uses ~/.kube/config

# Check merged contexts from multiple files:
kubectl config get-contexts

# Check which file a context came from:
kubectl config view --minify --flatten
# Shows the effective config for current context

# Verify a specific file works:
kubectl --kubeconfig ~/.kube/eks-production.yaml get nodes

# Fix: "error: no configuration has been provided"
# → KUBECONFIG points to a non-existent file
ls -la $(echo $KUBECONFIG | tr ':' '\n')

# Fix: "The connection to the server was refused"
# → Token expired, re-run aws eks update-kubeconfig to refresh
aws eks update-kubeconfig --name production --region us-east-1 \
  --kubeconfig ~/.kube/eks-production.yaml
```

## Quick Reference

```bash
# Write EKS config to separate file:
aws eks update-kubeconfig --name <cluster> --region <region> \
  --kubeconfig ~/.kube/eks-<cluster>.yaml --alias <cluster>

# Use a specific config:
export KUBECONFIG=~/.kube/eks-<cluster>.yaml
# Or:
kubectl --kubeconfig ~/.kube/eks-<cluster>.yaml get nodes

# Merge multiple at runtime:
export KUBECONFIG=~/.kube/eks-prod.yaml:~/.kube/eks-staging.yaml

# Auto-load all in shell profile:
export KUBECONFIG=$(find ~/.kube -name "eks-*.yaml" | tr '\n' ':')

# Quick switch function:
kube() { export KUBECONFIG=~/.kube/eks-${1}.yaml; kubectl config current-context; }

# List all available contexts (from merged files):
kubectl config get-contexts

# Switch context:
kubectl config use-context <name>
```
