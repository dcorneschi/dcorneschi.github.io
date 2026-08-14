# AKS (Azure Kubernetes Service) Cheatsheet

## Install the AKS CLI Extension

```sh
# Install kubectl via Azure CLI
az aks install-cli

# Install the aks-preview extension (optional, for preview features)
az extension add --name aks-preview
az extension update --name aks-preview
```

## Cluster Management

### Create a Cluster

```sh
# Basic cluster
az aks create \
  --resource-group <rg> \
  --name <cluster-name> \
  --node-count 3 \
  --node-vm-size Standard_D2s_v3 \
  --generate-ssh-keys

# With a specific Kubernetes version
az aks create \
  --resource-group <rg> \
  --name <cluster-name> \
  --node-count 3 \
  --kubernetes-version 1.29.2 \
  --generate-ssh-keys

# With managed identity (recommended)
az aks create \
  --resource-group <rg> \
  --name <cluster-name> \
  --node-count 3 \
  --enable-managed-identity \
  --generate-ssh-keys

# With Azure CNI networking
az aks create \
  --resource-group <rg> \
  --name <cluster-name> \
  --node-count 3 \
  --network-plugin azure \
  --vnet-subnet-id <subnet-id> \
  --generate-ssh-keys

# With availability zones
az aks create \
  --resource-group <rg> \
  --name <cluster-name> \
  --node-count 3 \
  --zones 1 2 3 \
  --generate-ssh-keys
```

### Get Credentials

```sh
# Merge kubeconfig for the cluster
az aks get-credentials --resource-group <rg> --name <cluster-name>

# Overwrite existing kubeconfig entry
az aks get-credentials --resource-group <rg> --name <cluster-name> --overwrite-existing

# Get admin credentials (bypasses RBAC)
az aks get-credentials --resource-group <rg> --name <cluster-name> --admin
```

### List and Show Clusters

```sh
# List all AKS clusters
az aks list --output table

# List clusters in a resource group
az aks list --resource-group <rg> --output table

# Show cluster details
az aks show --resource-group <rg> --name <cluster-name>

# Show cluster with specific fields
az aks show --resource-group <rg> --name <cluster-name> \
  --query "{Name:name, Version:kubernetesVersion, State:provisioningState, Nodes:agentPoolProfiles[0].count}" \
  --output table
```

### Delete a Cluster

```sh
az aks delete --resource-group <rg> --name <cluster-name> --yes --no-wait
```

## Kubernetes Versions

```sh
# List available Kubernetes versions in a region
az aks get-versions --location westeurope --output table

# Show current cluster version
az aks show --resource-group <rg> --name <cluster-name> --query kubernetesVersion --output tsv

# Upgrade cluster to a specific version
az aks upgrade --resource-group <rg> --name <cluster-name> --kubernetes-version 1.30.0

# Upgrade only the control plane
az aks upgrade --resource-group <rg> --name <cluster-name> --kubernetes-version 1.30.0 --control-plane-only
```

## Node Pools

### List Node Pools

```sh
az aks nodepool list --resource-group <rg> --cluster-name <cluster-name> --output table
```

### Add a Node Pool

```sh
# Standard node pool
az aks nodepool add \
  --resource-group <rg> \
  --cluster-name <cluster-name> \
  --name <pool-name> \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3

# Spot instance node pool
az aks nodepool add \
  --resource-group <rg> \
  --cluster-name <cluster-name> \
  --name spotnodes \
  --node-count 3 \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1

# Node pool with taints
az aks nodepool add \
  --resource-group <rg> \
  --cluster-name <cluster-name> \
  --name gpunodes \
  --node-count 1 \
  --node-vm-size Standard_NC6s_v3 \
  --node-taints "sku=gpu:NoSchedule"

# Node pool with labels
az aks nodepool add \
  --resource-group <rg> \
  --cluster-name <cluster-name> \
  --name workerpool \
  --node-count 3 \
  --labels env=prod tier=worker

# Node pool with availability zones
az aks nodepool add \
  --resource-group <rg> \
  --cluster-name <cluster-name> \
  --name zonalpool \
  --node-count 3 \
  --zones 1 2 3
```

### Scale a Node Pool

```sh
# Manual scale
az aks nodepool scale \
  --resource-group <rg> \
  --cluster-name <cluster-name> \
  --name <pool-name> \
  --node-count 5

# Enable cluster autoscaler
az aks nodepool update \
  --resource-group <rg> \
  --cluster-name <cluster-name> \
  --name <pool-name> \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 10

# Update autoscaler bounds
az aks nodepool update \
  --resource-group <rg> \
  --cluster-name <cluster-name> \
  --name <pool-name> \
  --update-cluster-autoscaler \
  --min-count 3 \
  --max-count 15

# Disable autoscaler
az aks nodepool update \
  --resource-group <rg> \
  --cluster-name <cluster-name> \
  --name <pool-name> \
  --disable-cluster-autoscaler
```

### Upgrade a Node Pool

```sh
# Upgrade node pool to match control plane
az aks nodepool upgrade \
  --resource-group <rg> \
  --cluster-name <cluster-name> \
  --name <pool-name> \
  --kubernetes-version 1.30.0

# Upgrade node image only (no K8s version change)
az aks nodepool upgrade \
  --resource-group <rg> \
  --cluster-name <cluster-name> \
  --name <pool-name> \
  --node-image-only
```

### Delete a Node Pool

```sh
az aks nodepool delete \
  --resource-group <rg> \
  --cluster-name <cluster-name> \
  --name <pool-name>
```

## Networking

```sh
# Show cluster networking config
az aks show --resource-group <rg> --name <cluster-name> \
  --query "networkProfile" --output json

# Enable HTTP application routing (dev only)
az aks enable-addons --resource-group <rg> --name <cluster-name> --addons http_application_routing

# Get the cluster's outbound IP
az aks show --resource-group <rg> --name <cluster-name> \
  --query "networkProfile.loadBalancerProfile.effectiveOutboundIPs[].id" --output tsv
```

## Azure Container Registry (ACR) Integration

```sh
# Attach ACR to AKS cluster
az aks update --resource-group <rg> --name <cluster-name> --attach-acr <acr-name>

# Detach ACR
az aks update --resource-group <rg> --name <cluster-name> --detach-acr <acr-name>

# Check ACR access
az aks check-acr --resource-group <rg> --name <cluster-name> --acr <acr-name>.azurecr.io
```

## Addons

```sh
# List available addons
az aks addon list-available --output table

# Enable monitoring (Container Insights)
az aks enable-addons --resource-group <rg> --name <cluster-name> --addons monitoring

# Enable Azure Policy
az aks enable-addons --resource-group <rg> --name <cluster-name> --addons azure-policy

# Enable Key Vault secrets provider
az aks enable-addons --resource-group <rg> --name <cluster-name> --addons azure-keyvault-secrets-provider

# Disable an addon
az aks disable-addons --resource-group <rg> --name <cluster-name> --addons monitoring

# List enabled addons
az aks show --resource-group <rg> --name <cluster-name> --query "addonProfiles" --output json
```

## Identity and RBAC

```sh
# Enable Azure RBAC for Kubernetes authorization
az aks update --resource-group <rg> --name <cluster-name> --enable-azure-rbac

# Enable AAD integration
az aks update --resource-group <rg> --name <cluster-name> --enable-aad

# Assign cluster admin role
az role assignment create \
  --assignee <user-or-sp-id> \
  --role "Azure Kubernetes Service Cluster Admin Role" \
  --scope <cluster-resource-id>

# Assign cluster user role
az role assignment create \
  --assignee <user-or-sp-id> \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope <cluster-resource-id>
```

## Maintenance and Operations

```sh
# Start a stopped cluster
az aks start --resource-group <rg> --name <cluster-name>

# Stop a cluster (saves compute costs)
az aks stop --resource-group <rg> --name <cluster-name>

# Rotate cluster certificates
az aks rotate-certs --resource-group <rg> --name <cluster-name>

# Get diagnostic logs
az aks kollect --resource-group <rg> --name <cluster-name> --storage-account <sa-name>

# Run kubectl commands via Azure CLI
az aks command invoke --resource-group <rg> --name <cluster-name> --command "kubectl get nodes"
az aks command invoke --resource-group <rg> --name <cluster-name> --command "kubectl get pods -A"
```

## Troubleshooting

```sh
# Check cluster provisioning state
az aks show --resource-group <rg> --name <cluster-name> --query provisioningState --output tsv

# Get node resource group (MC_ group)
az aks show --resource-group <rg> --name <cluster-name> --query nodeResourceGroup --output tsv

# List nodes and their status (via kubectl)
kubectl get nodes -o wide

# Check node pool provisioning state
az aks nodepool show --resource-group <rg> --cluster-name <cluster-name> --name <pool-name> --query provisioningState

# SSH into a node (using kubectl debug)
kubectl debug node/<node-name> -it --image=mcr.microsoft.com/cbl-mariner/busybox:2.0

# View cluster upgrade history
az aks show --resource-group <rg> --name <cluster-name> --query "currentKubernetesVersion"
```


## Gotchas

- **System node pool cannot be deleted**: Every AKS cluster must have at least one system node pool. You can add a new system pool and then delete the old one, but you can't have zero.
- **`az aks get-credentials` merges kubeconfig**: It merges the new cluster context into `~/.kube/config`. If a context with the same name exists, it gets overwritten silently. Use `--overwrite-existing` explicitly or `--file` to write to a separate file.
- **Node pool VM size is immutable**: You cannot resize an existing node pool. Create a new pool with the desired size, migrate workloads, then delete the old pool.
- **Cluster autoscaler and manual scaling conflict**: If autoscaler is enabled, don't use `az aks nodepool scale` — the autoscaler will override your manual count.
- **Kubernetes version skew**: Node pools can be at most 2 minor versions behind the control plane. Upgrade the control plane first, then node pools.
- **MC_ resource group**: AKS creates a managed resource group (prefixed `MC_`) for infrastructure resources. Don't modify resources in it manually — AKS manages them.

## Workload Identity

```sh
# Enable workload identity on the cluster
az aks update --resource-group <rg> --name <cluster-name> --enable-oidc-issuer --enable-workload-identity

# Get the OIDC issuer URL
az aks show --resource-group <rg> --name <cluster-name> --query "oidcIssuerProfile.issuerUrl" --output tsv

# Create a managed identity
az identity create --resource-group <rg> --name <identity-name>

# Create federated credential
az identity federated-credential create \
  --resource-group <rg> \
  --identity-name <identity-name> \
  --name <federated-cred-name> \
  --issuer <oidc-issuer-url> \
  --subject system:serviceaccount:<namespace>:<service-account-name> \
  --audience api://AzureADTokenExchange
```

Then annotate the Kubernetes ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <service-account-name>
  namespace: <namespace>
  annotations:
    azure.workload.identity/client-id: <managed-identity-client-id>
```

## Network Policies

```sh
# Create cluster with Azure network policy
az aks create \
  --resource-group <rg> \
  --name <cluster-name> \
  --network-plugin azure \
  --network-policy azure \
  --generate-ssh-keys

# Create cluster with Calico network policy
az aks create \
  --resource-group <rg> \
  --name <cluster-name> \
  --network-plugin azure \
  --network-policy calico \
  --generate-ssh-keys
```

| Engine | Best for |
|--------|----------|
| Azure NPM | Simple L3/L4 policies, Azure-native |
| Calico | Advanced policies, DNS rules, global network sets |

## Upgrade Best Practices

```sh
# Check available upgrades
az aks get-upgrades --resource-group <rg> --name <cluster-name> --output table

# Upgrade control plane first
az aks upgrade --resource-group <rg> --name <cluster-name> --kubernetes-version <version> --control-plane-only

# Upgrade node pools one at a time
az aks nodepool upgrade --resource-group <rg> --cluster-name <cluster-name> --name <pool> --kubernetes-version <version>

# Set max surge for faster upgrades (extra nodes during upgrade)
az aks nodepool update \
  --resource-group <rg> \
  --cluster-name <cluster-name> \
  --name <pool> \
  --max-surge 33%
```

For production: cordon and drain critical workloads before upgrading, or use PodDisruptionBudgets to control disruption.

## One-Liners

```sh
# Get all AKS clusters and their versions across subscription
az aks list --query "[].{Name:name, RG:resourceGroup, Version:kubernetesVersion, State:provisioningState}" --output table

# Check if any clusters have available upgrades
az aks list --query "[].{Name:name, Version:kubernetesVersion}" -o tsv | while read name version; do echo "$name: $version"; done

# Get node counts across all pools
az aks nodepool list --resource-group <rg> --cluster-name <cluster-name> --query "[].{Pool:name, Count:count, Size:vmSize, Version:orchestratorVersion}" --output table

# Quick cluster health check
az aks show --resource-group <rg> --name <cluster-name> --query "{State:provisioningState, Power:powerState.code, Version:kubernetesVersion, Nodes:agentPoolProfiles[].{Name:name,Count:count}}" --output json

# List all stopped clusters (save costs overnight)
az aks list --query "[?powerState.code=='Stopped'].{Name:name, RG:resourceGroup}" --output table

# Run a quick diagnostic without kubectl
az aks command invoke --resource-group <rg> --name <cluster-name> --command "kubectl top nodes && kubectl get pods -A | grep -v Running"
```
