# Cleaning Up Kubernetes Clusters from .kube/config

How to identify and remove stale, unreachable, or unused cluster entries from your kubeconfig file — keeping it lean and avoiding accidental connections to dead clusters.

## Why Clean Up kubeconfig?

Over time, `~/.kube/config` accumulates entries from:
- Deleted EKS/GKE/AKS clusters
- Decommissioned on-prem clusters
- Temporary test clusters (kind, minikube, k3d)
- Expired or rotated credentials
- Clusters you no longer have access to

Problems caused by a bloated kubeconfig:
- Slow tab completion and context switching
- Accidental commands against wrong clusters
- Confusing output from `kubectl config get-contexts`
- Expired tokens causing auth errors

## View Current State

```bash
# List all contexts
kubectl config get-contexts

# List just context names
kubectl config get-contexts -o name

# Show current context
kubectl config current-context

# List all clusters
kubectl config get-clusters

# List all users
kubectl config get-users

# View full config (sanitized)
kubectl config view
```

## Identify Stale Clusters

### Test Connectivity to All Clusters

```bash
# Quick check — try to reach each cluster's API server
for ctx in $(kubectl config get-contexts -o name); do
  echo -n "$ctx: "
  kubectl --context="$ctx" cluster-info --request-timeout=5s 2>&1 | head -1
done
```

### Find Unreachable Clusters

```bash
# Script to identify dead clusters
for ctx in $(kubectl config get-contexts -o name); do
  if ! kubectl --context="$ctx" get nodes --request-timeout=5s &>/dev/null; then
    echo "UNREACHABLE: $ctx"
  else
    echo "OK: $ctx"
  fi
done
```

### Check Cluster Age and Last Use

```bash
# See when contexts were last used (if shell history is available)
grep -h "kubectl.*--context" ~/.bash_history ~/.zsh_history 2>/dev/null | sort | uniq -c | sort -rn

# Check certificate expiry dates in kubeconfig
kubectl config view --raw -o jsonpath='{range .users[*]}{.name}{"\t"}{.user.client-certificate-data}{"\n"}{end}' | while read name cert; do
  if [[ -n "$cert" ]]; then
    expiry=$(echo "$cert" | base64 -d | openssl x509 -noout -enddate 2>/dev/null)
    echo "$name: $expiry"
  fi
done
```

## Remove a Single Cluster

Removing a cluster requires deleting three things: the context, the cluster entry, and the user entry.

### Step-by-Step

```bash
# 1. Identify what to remove
kubectl config get-contexts | grep <cluster-name>

# 2. Delete the context
kubectl config delete-context <context-name>

# 3. Delete the cluster entry
kubectl config delete-cluster <cluster-name>

# 4. Delete the user entry
kubectl config delete-user <user-name>
```

### Example: Remove an Old EKS Cluster

```bash
# View the context details first
kubectl config get-contexts | grep my-old-cluster

# Output:
#           my-old-cluster   arn:aws:eks:us-east-1:123456:cluster/my-old-cluster   arn:aws:eks:us-east-1:123456:cluster/my-old-cluster

# Remove all three components
kubectl config delete-context my-old-cluster
kubectl config delete-cluster arn:aws:eks:us-east-1:123456789012:cluster/my-old-cluster
kubectl config delete-user arn:aws:eks:us-east-1:123456789012:cluster/my-old-cluster
```

### Example: Remove a kind Cluster

```bash
kubectl config delete-context kind-my-cluster
kubectl config delete-cluster kind-my-cluster
kubectl config delete-user kind-my-cluster
```

## Remove All Unreachable Clusters

```bash
#!/bin/bash
# Remove all clusters that can't be reached

echo "Testing cluster connectivity..."
STALE_CONTEXTS=()

for ctx in $(kubectl config get-contexts -o name); do
  if ! kubectl --context="$ctx" get nodes --request-timeout=5s &>/dev/null; then
    STALE_CONTEXTS+=("$ctx")
    echo "  STALE: $ctx"
  else
    echo "  OK:    $ctx"
  fi
done

echo ""
echo "Found ${#STALE_CONTEXTS[@]} stale context(s)."

if [[ ${#STALE_CONTEXTS[@]} -eq 0 ]]; then
  echo "Nothing to clean up."
  exit 0
fi

echo ""
read -p "Remove all stale contexts? (y/N) " confirm
if [[ "$confirm" != "y" ]]; then
  echo "Aborted."
  exit 0
fi

for ctx in "${STALE_CONTEXTS[@]}"; do
  # Get cluster and user names from the context
  cluster=$(kubectl config view -o jsonpath="{.contexts[?(@.name==\"$ctx\")].context.cluster}")
  user=$(kubectl config view -o jsonpath="{.contexts[?(@.name==\"$ctx\")].context.user}")

  echo "Removing context: $ctx"
  kubectl config delete-context "$ctx" 2>/dev/null

  # Only delete cluster/user if no other context uses them
  remaining=$(kubectl config view -o jsonpath="{.contexts[?(@.context.cluster==\"$cluster\")].name}")
  if [[ -z "$remaining" ]]; then
    echo "  Removing cluster: $cluster"
    kubectl config delete-cluster "$cluster" 2>/dev/null
  fi

  remaining=$(kubectl config view -o jsonpath="{.contexts[?(@.context.user==\"$user\")].name}")
  if [[ -z "$remaining" ]]; then
    echo "  Removing user: $user"
    kubectl config delete-user "$user" 2>/dev/null
  fi
done

echo ""
echo "Cleanup complete."
```

## Remove Clusters by Pattern

### Remove All kind Clusters

```bash
for ctx in $(kubectl config get-contexts -o name | grep "^kind-"); do
  cluster=$(kubectl config view -o jsonpath="{.contexts[?(@.name==\"$ctx\")].context.cluster}")
  user=$(kubectl config view -o jsonpath="{.contexts[?(@.name==\"$ctx\")].context.user}")
  kubectl config delete-context "$ctx"
  kubectl config delete-cluster "$cluster"
  kubectl config delete-user "$user"
done
```

### Remove All minikube Clusters

```bash
# minikube has its own cleanup command
minikube delete --all --purge

# Or manually from kubeconfig
for ctx in $(kubectl config get-contexts -o name | grep "minikube"); do
  kubectl config delete-context "$ctx"
  kubectl config delete-cluster "$ctx"
  kubectl config delete-user "$ctx"
done
```

### Remove All EKS Clusters from a Specific Account

```bash
ACCOUNT_ID="123456789012"
for ctx in $(kubectl config get-contexts -o name | grep "$ACCOUNT_ID"); do
  cluster=$(kubectl config view -o jsonpath="{.contexts[?(@.name==\"$ctx\")].context.cluster}")
  user=$(kubectl config view -o jsonpath="{.contexts[?(@.name==\"$ctx\")].context.user}")
  kubectl config delete-context "$ctx"
  kubectl config delete-cluster "$cluster"
  kubectl config delete-user "$user"
done
```

### Remove All Clusters from a Region

```bash
REGION="us-west-2"
for ctx in $(kubectl config get-contexts -o name | grep "$REGION"); do
  cluster=$(kubectl config view -o jsonpath="{.contexts[?(@.name==\"$ctx\")].context.cluster}")
  user=$(kubectl config view -o jsonpath="{.contexts[?(@.name==\"$ctx\")].context.user}")
  kubectl config delete-context "$ctx"
  kubectl config delete-cluster "$cluster"
  kubectl config delete-user "$user"
done
```

## Flatten and Rebuild kubeconfig

### Start Fresh with Only Active Clusters

```bash
# Backup current config
cp ~/.kube/config ~/.kube/config.backup.$(date +%Y%m%d)

# Re-add only the clusters you need
# EKS
aws eks update-kubeconfig --region us-east-1 --name production-cluster
aws eks update-kubeconfig --region us-east-1 --name staging-cluster

# GKE
gcloud container clusters get-credentials my-cluster --region us-central1

# kind
kind export kubeconfig --name local-dev
```

### Merge Multiple kubeconfig Files

```bash
# Use KUBECONFIG env var to merge
export KUBECONFIG=~/.kube/config:~/.kube/cluster-a.yaml:~/.kube/cluster-b.yaml

# Flatten into a single file
kubectl config view --flatten > ~/.kube/config.merged

# Replace the original
mv ~/.kube/config.merged ~/.kube/config
chmod 600 ~/.kube/config
```

## Orphan Detection

After deleting contexts, check for orphaned clusters and users with no associated context:

```bash
# Find clusters with no context pointing to them
for cluster in $(kubectl config get-clusters | tail -n +2); do
  match=$(kubectl config view -o jsonpath="{.contexts[?(@.context.cluster==\"$cluster\")].name}")
  if [[ -z "$match" ]]; then
    echo "ORPHAN cluster: $cluster"
  fi
done

# Find users with no context pointing to them
for user in $(kubectl config get-users | tail -n +2); do
  match=$(kubectl config view -o jsonpath="{.contexts[?(@.context.user==\"$user\")].name}")
  if [[ -z "$match" ]]; then
    echo "ORPHAN user: $user"
  fi
done
```

### Remove All Orphans

```bash
# Remove orphaned clusters
for cluster in $(kubectl config get-clusters | tail -n +2); do
  match=$(kubectl config view -o jsonpath="{.contexts[?(@.context.cluster==\"$cluster\")].name}")
  if [[ -z "$match" ]]; then
    echo "Removing orphan cluster: $cluster"
    kubectl config delete-cluster "$cluster"
  fi
done

# Remove orphaned users
for user in $(kubectl config get-users | tail -n +2); do
  match=$(kubectl config view -o jsonpath="{.contexts[?(@.context.user==\"$user\")].name}")
  if [[ -z "$match" ]]; then
    echo "Removing orphan user: $user"
    kubectl config delete-user "$user"
  fi
done
```

## Rename Contexts for Clarity

EKS contexts often have long ARN-based names. Rename them for easier use:

```bash
# Rename a context
kubectl config rename-context \
  arn:aws:eks:us-east-1:123456789012:cluster/production \
  prod-us-east-1

kubectl config rename-context \
  arn:aws:eks:eu-west-1:123456789012:cluster/staging \
  staging-eu-west-1
```

Suggested naming convention: `<environment>-<region>` or `<cluster-name>-<region>`

## Using Separate kubeconfig Files

Instead of one large file, keep configs separate per cluster or environment:

```bash
# Set KUBECONFIG to load multiple files
export KUBECONFIG=~/.kube/configs/prod.yaml:~/.kube/configs/staging.yaml:~/.kube/configs/dev.yaml

# Add to shell profile
echo 'export KUBECONFIG=$(find ~/.kube/configs -name "*.yaml" | tr "\n" ":")' >> ~/.zshrc
```

Benefits:
- Easier to add/remove clusters (just add/delete a file)
- No risk of corrupting a monolithic config
- Can version-control individual configs
- Simpler sharing with teammates

## Safety Tips

### Always Backup Before Cleanup

```bash
cp ~/.kube/config ~/.kube/config.backup.$(date +%Y%m%d)
```

### Verify Current Context After Cleanup

```bash
# Make sure your current context still exists
kubectl config current-context

# If it was deleted, set a new one
kubectl config use-context <valid-context>
```

### Check File Permissions

```bash
# kubeconfig should be readable only by you
chmod 600 ~/.kube/config
ls -la ~/.kube/config
```

## Batch Removal by Name List

When you know exactly which clusters to remove:

```bash
#!/bin/bash
# Remove a predefined list of clusters

CLUSTERS_TO_REMOVE=(
    "dev-cluster-old"
    "staging-deprecated"
    "test-cluster-2023"
)

for cluster in "${CLUSTERS_TO_REMOVE[@]}"; do
    echo "Removing cluster: $cluster"

    # Find and remove associated contexts
    for context in $(kubectl config get-contexts -o name | grep "$cluster"); do
        echo "  Deleting context: $context"
        kubectl config delete-context "$context"
    done

    # Remove cluster
    echo "  Deleting cluster: $cluster"
    kubectl config delete-cluster "$cluster"

    # Find and remove associated users
    for user in $(kubectl config view -o jsonpath='{.users[*].name}' | tr ' ' '\n' | grep "$cluster"); do
        echo "  Deleting user: $user"
        kubectl config delete-user "$user"
    done
done

echo "Cleanup complete."
```

## Interactive Cleanup Tool

A menu-driven script for ad-hoc cleanup:

```bash
#!/bin/bash
# interactive-kubeconfig-cleanup.sh

backup_config() {
    backup_file="$HOME/.kube/config.backup.$(date +%Y%m%d_%H%M%S)"
    cp "$HOME/.kube/config" "$backup_file"
    echo "Config backed up to: $backup_file"
}

while true; do
    echo ""
    echo "=== Kubeconfig Cleanup Tool ==="
    echo "1. List all contexts"
    echo "2. List all clusters"
    echo "3. List all users"
    echo "4. Remove specific context"
    echo "5. Remove specific cluster"
    echo "6. Remove specific user"
    echo "7. Test cluster connectivity"
    echo "8. Backup current config"
    echo "9. Exit"
    echo ""
    read -p "Choose (1-9): " choice

    case $choice in
        1) kubectl config get-contexts ;;
        2) kubectl config get-clusters ;;
        3) kubectl config get-users ;;
        4)
            read -p "Context name: " name
            kubectl config delete-context "$name"
            ;;
        5)
            read -p "Cluster name: " name
            kubectl config delete-cluster "$name"
            ;;
        6)
            read -p "User name: " name
            kubectl config delete-user "$name"
            ;;
        7)
            for ctx in $(kubectl config get-contexts -o name); do
                echo -n "$ctx: "
                if kubectl --context="$ctx" cluster-info --request-timeout=5s &>/dev/null; then
                    echo "reachable"
                else
                    echo "UNREACHABLE"
                fi
            done
            ;;
        8) backup_config ;;
        9) exit 0 ;;
        *) echo "Invalid option." ;;
    esac
done
```

## Editing with yq

Use `yq` for precise YAML manipulation without kubectl subcommands:

```bash
# Remove a specific cluster
yq eval 'del(.clusters[] | select(.name == "cluster-to-remove"))' -i ~/.kube/config

# Remove a specific context
yq eval 'del(.contexts[] | select(.name == "context-to-remove"))' -i ~/.kube/config

# Remove a specific user
yq eval 'del(.users[] | select(.name == "user-to-remove"))' -i ~/.kube/config

# Remove all clusters matching a pattern
yq eval 'del(.clusters[] | select(.name == "*old*"))' -i ~/.kube/config
```

Install yq: `brew install yq` (macOS) or `sudo apt install yq` (Ubuntu).

## Using kubectx

If you manage many clusters, `kubectx` provides a faster workflow:

```bash
# Install
brew install kubectx

# List all contexts
kubectx

# Switch context
kubectx <context-name>

# Delete a context
kubectx -d <context-name>

# Delete multiple contexts
kubectx -d old-dev old-staging test-2023
```

`kubectx -d` only removes the context — you still need to clean up orphaned cluster/user entries separately.

## Scheduled Maintenance

### Cron Job to Flag Unreachable Clusters

```bash
# Add to crontab (crontab -e)
# Run connectivity check weekly, log stale clusters
0 9 * * 1 /path/to/check-clusters.sh >> ~/.kube/stale-clusters.log 2>&1
```

Check script:

```bash
#!/bin/bash
# check-clusters.sh — log unreachable clusters
echo "=== $(date) ==="
for ctx in $(kubectl config get-contexts -o name); do
  if ! kubectl --context="$ctx" cluster-info --request-timeout=5s &>/dev/null; then
    echo "STALE: $ctx"
  fi
done
```

### Validate Config Integrity

```bash
# Quick validation — exits non-zero if malformed
kubectl config view >/dev/null 2>&1 && echo "Config OK" || echo "Config INVALID"

# If corrupted, restore from backup
cp ~/.kube/config.backup ~/.kube/config
```

## Quick Reference

```bash
# List contexts
kubectl config get-contexts -o name

# Delete a full cluster entry (context + cluster + user)
kubectl config delete-context <name>
kubectl config delete-cluster <name>
kubectl config delete-user <name>

# Rename context
kubectl config rename-context <old> <new>

# Switch context
kubectl config use-context <name>

# Set default namespace for current context
kubectl config set-context --current --namespace=<ns>

# Backup config
cp ~/.kube/config ~/.kube/config.bak

# View raw config (with secrets)
kubectl config view --raw
```
