# Azure Resource Groups Cheatsheet

## What is a Resource Group?

A resource group is a logical container for Azure resources. All resources in a group share the same lifecycle — they are deployed, updated, and deleted together. A resource group is tied to a region (for metadata), but its resources can be in different regions.

## Create a Resource Group

```sh
az group create --name <name> --location <location>

# With tags
az group create --name <name> --location westeurope --tags env=prod team=infra project=myapp
```

## List Resource Groups

```sh
# Table format
az group list --output table

# Filter by tag
az group list --tag env=prod --output table

# Filter by location
az group list --query "[?location=='westeurope']" --output table

# Show only names
az group list --query "[].name" --output tsv
```

## Show Resource Group Details

```sh
az group show --name <name>

# Show provisioning state
az group show --name <name> --query properties.provisioningState --output tsv
```

## Check if a Resource Group Exists

```sh
az group exists --name <name>
```

Returns `true` or `false`.

## Update Tags

```sh
# Set tags (replaces all existing tags)
az group update --name <name> --tags env=staging team=dev

# Add a tag without removing existing ones (use az tag)
az tag update --resource-id /subscriptions/<sub-id>/resourceGroups/<name> --operation merge --tags newkey=newvalue

# Remove all tags
az group update --name <name> --tags ""
```

## List Resources in a Group

```sh
# All resources
az resource list --resource-group <name> --output table

# Filter by resource type
az resource list --resource-group <name> --resource-type Microsoft.Compute/virtualMachines --output table

# Count resources by type
az resource list --resource-group <name> --query "[].type" | sort | uniq -c
```

## Move Resources Between Groups

```sh
# Move resources to another resource group
az resource move \
  --destination-group <target-rg> \
  --ids <resource-id-1> <resource-id-2>

# Validate before moving
az resource move \
  --destination-group <target-rg> \
  --ids <resource-id> \
  --validate-only
```

Not all resource types support moving. Check the docs for supported types.

## Lock a Resource Group

Locks prevent accidental deletion or modification.

```sh
# Add a delete lock
az lock create \
  --name <lock-name> \
  --resource-group <name> \
  --lock-type CanNotDelete

# Add a read-only lock
az lock create \
  --name <lock-name> \
  --resource-group <name> \
  --lock-type ReadOnly

# List locks
az lock list --resource-group <name> --output table

# Delete a lock
az lock delete --name <lock-name> --resource-group <name>
```

## Export a Resource Group Template

```sh
# Export as ARM template
az group export --name <name> > template.json

# Export specific resources only
az group export --name <name> --resource-ids <resource-id-1> <resource-id-2>
```

## Delete a Resource Group

```sh
# Delete (prompts for confirmation)
az group delete --name <name>

# Delete without confirmation
az group delete --name <name> --yes

# Delete without waiting
az group delete --name <name> --yes --no-wait

# Delete multiple resource groups
for rg in rg1 rg2 rg3; do az group delete --name $rg --yes --no-wait; done
```

> Deleting a resource group deletes ALL resources inside it. This is irreversible.

## RBAC on Resource Groups

```sh
# Assign Contributor role to a user
az role assignment create \
  --assignee <user-email-or-object-id> \
  --role "Contributor" \
  --resource-group <name>

# Assign Reader role to a group
az role assignment create \
  --assignee <group-object-id> \
  --role "Reader" \
  --resource-group <name>

# List role assignments for a resource group
az role assignment list --resource-group <name> --output table

# Remove a role assignment
az role assignment delete \
  --assignee <user-email-or-object-id> \
  --role "Contributor" \
  --resource-group <name>
```

## Deployment History

```sh
# List deployments in a resource group
az deployment group list --resource-group <name> --output table

# Show a specific deployment
az deployment group show --resource-group <name> --name <deployment-name>

# Delete old deployments
az deployment group delete --resource-group <name> --name <deployment-name>
```

## Best Practices

- Group resources by lifecycle (resources that are deployed and deleted together)
- Use consistent naming: `<project>-<env>-<region>-rg` (e.g., `myapp-prod-westeurope-rg`)
- Apply tags for cost tracking, ownership, and environment identification
- Use locks on production resource groups to prevent accidental deletion
- Keep resource groups focused — avoid dumping unrelated resources together


## Gotchas

- **Deletion is async**: `az group delete` starts the deletion but resources may still be running for a few minutes. Use `--no-wait` to return immediately, or omit it to block until complete.
- **Locks block everything**: A `ReadOnly` lock prevents modifications to ANY resource in the group — including deployments. A `CanNotDelete` lock is usually more practical for production.
- **Moving resources**: Not all resource types support moving between groups. Virtual network gateways, App Service Environments, and some others cannot be moved. Always validate first with `--validate-only`.
- **Tags are not inherited**: Resources do NOT inherit tags from their resource group. You need Azure Policy to enforce tagging on child resources.
- **Location is for metadata only**: The resource group's location stores where its metadata lives. Resources inside can be in any region.

## Enforce Tagging with Azure Policy

```sh
# Assign built-in policy: "Require a tag on resource groups"
az policy assignment create \
  --name "require-env-tag" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/96670d01-0a4d-4649-9c89-2d3abc0a5025" \
  --params '{"tagName": {"value": "env"}}' \
  --scope "/subscriptions/<sub-id>"

# List policy assignments
az policy assignment list --output table
```

## Cost Analysis per Resource Group

```sh
# View costs for a resource group (requires Cost Management)
az consumption usage list \
  --query "[?contains(instanceName, '<rg-name>')].{Resource:instanceName, Cost:pretaxCost, Currency:currency}" \
  --output table

# Quick cost summary using the portal
# Go to: Resource Group > Cost Analysis
```

## One-Liners

```sh
# List all empty resource groups (no resources)
az group list --query "[?properties.provisioningState=='Succeeded']" -o tsv | while read -r rg; do
  count=$(az resource list -g "$rg" --query "length(@)" -o tsv 2>/dev/null)
  [ "$count" = "0" ] && echo "$rg"
done

# Delete all resource groups with a specific tag
az group list --tag env=temp --query "[].name" -o tsv | xargs -I {} az group delete --name {} --yes --no-wait

# Count resources per resource group
az group list --query "[].name" -o tsv | while read -r rg; do
  count=$(az resource list -g "$rg" --query "length(@)" -o tsv)
  echo "$rg: $count"
done
```


## Using jq with Resource Group Commands

### Resource Group Analysis

```sh
# Get resource groups with key details
az group list --output json | jq '.[] | {name, location, provisioningState, tags}'

# Count resource groups by location
az group list --output json | jq 'group_by(.location) | map({location: .[0].location, count: length})'

# Find resource groups with specific tags
az group list --output json | jq '.[] | select(.tags.environment == "production")'

# Count resources by type within a resource group
az resource list --resource-group <rg> --output json | jq 'group_by(.type) | map({type: .[0].type, count: length})'

# Quick resource count by type across subscription
az resource list --output json | jq 'group_by(.type) | map({type: .[0].type, count: length}) | sort_by(.count) | reverse'

# Get resource groups with most resources
az resource list --output json | jq 'group_by(.resourceGroup) | map({resourceGroup: .[0].resourceGroup, count: length}) | sort_by(.count) | reverse | .[0:10]'
```

### Tag Governance

```sh
# Find untagged resources
az resource list --output json | jq '.[] | select(.tags == null or (.tags | length == 0)) | {name, type, resourceGroup}'

# Tag coverage by resource type
az resource list --output json | jq 'group_by(.type) | map({
  type: .[0].type,
  count: length,
  tagged_pct: (([.[] | select(.tags != null and (.tags | length > 0))] | length) / length * 100 | round)
})'

# Most common tags
az resource list --output json | jq '[.[].tags // {} | to_entries[]] | group_by(.key) | map({tag: .[0].key, usage_count: length}) | sort_by(.usage_count) | reverse'
```

### Resource Distribution

```sh
# Find all resources in a specific location
az resource list --output json | jq --arg location "eastus" '.[] | select(.location == $location) | {name, type, resourceGroup}'

# Find resources created today
az resource list --output json | jq --arg today "$(date +%Y-%m-%d)" '.[] | select(.createdTime | startswith($today)) | {name, type, resourceGroup}'
```
