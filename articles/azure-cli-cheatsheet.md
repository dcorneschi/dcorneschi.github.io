# Azure CLI Cheatsheet

## Installation

```sh
# macOS
brew install azure-cli

# Ubuntu / Debian
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# RHEL / CentOS
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo dnf install azure-cli

# Docker
docker run -it mcr.microsoft.com/azure-cli
```

## Authentication

```sh
# Interactive login (opens browser)
az login

# Login with a service principal
az login --service-principal -u <app-id> -p <password> --tenant <tenant-id>

# Login with a managed identity (from an Azure VM)
az login --identity

# Show current account
az account show

# List all subscriptions
az account list --output table

# Set active subscription
az account set --subscription <subscription-id>

# Logout
az logout
```

## Configuration

```sh
# Show current config
az configure --list-defaults

# Set default resource group
az configure --defaults group=<resource-group>

# Set default location
az configure --defaults location=westeurope

# Set default output format (table, json, yaml, tsv)
az configure --defaults output=table

# Clear defaults
az configure --defaults group="" location=""
```

## Output Formats

```sh
# Table (human-readable)
az vm list --output table

# JSON (default)
az vm list --output json

# TSV (tab-separated, good for scripting)
az vm list --query "[].{Name:name, RG:resourceGroup}" --output tsv

# YAML
az vm list --output yaml
```

## Query with JMESPath

```sh
# Get specific fields
az vm list --query "[].{Name:name, Size:hardwareProfile.vmSize}" --output table

# Filter results
az vm list --query "[?location=='westeurope']" --output table

# Get a single value
az vm show -g <rg> -n <vm> --query "hardwareProfile.vmSize" --output tsv

# Count items
az vm list --query "length(@)"
```

## Resource Groups

```sh
# List resource groups
az group list --output table

# Create a resource group
az group create --name <name> --location <location>

# Delete a resource group (and all resources in it)
az group delete --name <name> --yes --no-wait

# Check if a resource group exists
az group exists --name <name>

# List resources in a group
az resource list --resource-group <name> --output table
```

## Common Patterns

```sh
# Wait for an operation to complete
az vm create ... --no-wait
az vm wait --name <vm> --resource-group <rg> --created

# Find available VM sizes in a region
az vm list-sizes --location westeurope --output table

# Find available VM images
az vm image list --output table
az vm image list --publisher Canonical --offer 0001-com-ubuntu-server-jammy --all --output table

# List available regions
az account list-locations --output table

# Tag a resource
az resource tag --tags env=prod team=infra --resource-group <rg> --name <name> --resource-type <type>

# Find resources by tag
az resource list --tag env=prod --output table
```

## Extensions

```sh
# List installed extensions
az extension list --output table

# Add an extension
az extension add --name <extension-name>

# Update an extension
az extension update --name <extension-name>

# Remove an extension
az extension remove --name <extension-name>

# List available extensions
az extension list-available --output table
```

## Useful Tips

```sh
# Get help for any command
az vm --help
az vm create --help

# Find commands
az find "create virtual machine"

# Interactive mode (auto-completion)
az interactive

# Upgrade Azure CLI
az upgrade

# Show CLI version
az version
```


## Environment Variables for Scripting

Skip `az configure` in CI/CD by using environment variables:

```sh
export AZURE_DEFAULTS_GROUP=myapp-prod-rg
export AZURE_DEFAULTS_LOCATION=westeurope

# Now these commands don't need --resource-group or --location
az vm list --output table
az vm create --name myvm --image Ubuntu2204 --generate-ssh-keys
```

Other useful variables:

| Variable | Description |
|----------|-------------|
| `AZURE_DEFAULTS_GROUP` | Default resource group |
| `AZURE_DEFAULTS_LOCATION` | Default region |
| `AZURE_CONFIG_DIR` | Config directory (default `~/.azure`) |
| `AZURE_CLI_DISABLE_CONNECTION_VERIFICATION` | Skip SSL verification (testing only) |

## Call Any Azure API with az rest

```sh
# GET request
az rest --method get --url "https://management.azure.com/subscriptions/{sub-id}/providers/Microsoft.Compute/virtualMachines?api-version=2023-09-01"

# POST request with body
az rest --method post --url "<resource-id>/restart?api-version=2023-09-01"

# Useful for APIs not yet covered by az commands
```

## Async Operations with --no-wait

```sh
# Start an operation without blocking
az vm create --resource-group <rg> --name <vm> --image Ubuntu2204 --generate-ssh-keys --no-wait

# Check status later
az vm wait --resource-group <rg> --name <vm> --created

# Works with most long-running commands
az group delete --name <rg> --yes --no-wait
az aks create ... --no-wait
```

## Gotchas

- **Subscription context**: If commands fail unexpectedly, check you're in the right subscription with `az account show`. Use `az account set --subscription <id>` to switch.
- **Token expiry**: Tokens expire after ~1 hour. If you get auth errors after being idle, run `az login` again or use `az account get-access-token` to check.
- **Resource providers**: Some resources require a provider to be registered. If you get "resource provider not registered" errors:
  ```sh
  az provider register --namespace Microsoft.ContainerService
  az provider show --namespace Microsoft.ContainerService --query "registrationState"
  ```
- **API versions**: When using `az rest`, you must specify the correct `api-version`. Check docs for the resource type.
- **Quoting in PowerShell vs bash**: JMESPath queries containing `[]` need different escaping. In bash use single quotes, in PowerShell use double quotes with backticks.

## One-Liners

```sh
# Get current subscription ID
az account show --query id --output tsv

# List all resources with their types across subscription
az resource list --query "[].{Name:name, Type:type, RG:resourceGroup}" --output table

# Find the most expensive resource group (requires cost management)
az consumption usage list --query "sort_by(@, &pretaxCost)[-5:].{RG:instanceName, Cost:pretaxCost}" --output table

# Export all resource IDs in a resource group
az resource list --resource-group <rg> --query "[].id" --output tsv

# Check which regions support a VM size
az vm list-skus --size Standard_D2s_v3 --query "[].{Location:locationInfo[0].location, Zones:locationInfo[0].zones}" --output table
```
