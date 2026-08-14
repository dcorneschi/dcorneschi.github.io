# Azure Key Vault, Monitoring, Cost Management, IAM, and App Services Cheatsheet

## Key Vault

### Key Vault Management

```sh
# List key vaults
az keyvault list --output table

# Create a key vault
az keyvault create --resource-group <rg> --name <vault-name> --location westeurope

# Get key vault properties
az keyvault list --output json | jq '.[] | {name, resourceGroup, location, sku: .properties.sku.name, enabledForDeployment: .properties.enabledForDeployment}'

# Show key vault details
az keyvault show --name <vault-name> --output json
```

### Secrets

```sh
# List secrets
az keyvault secret list --vault-name <vault-name> --output table

# List secrets with jq
az keyvault secret list --vault-name <vault-name> --output json | jq '.[] | {name: .name, enabled: .attributes.enabled, created: .attributes.created}'

# Set a secret
az keyvault secret set --vault-name <vault-name> --name <secret-name> --value "<secret-value>"

# Get a secret value
az keyvault secret show --vault-name <vault-name> --name <secret-name> --query value --output tsv

# Find recently created secrets (last 7 days)
az keyvault secret list --vault-name <vault-name> --output json | jq --arg date "$(date -d '7 days ago' '+%Y-%m-%d')" '.[] | select(.attributes.created >= $date) | {name: .name, created: .attributes.created}'

# Delete a secret
az keyvault secret delete --vault-name <vault-name> --name <secret-name>

# Purge a deleted secret (if soft-delete enabled)
az keyvault secret purge --vault-name <vault-name> --name <secret-name>
```

### Keys

```sh
# List keys
az keyvault key list --vault-name <vault-name> --output table

# List keys with jq
az keyvault key list --vault-name <vault-name> --output json | jq '.[] | {name: .name, keyType: .key.kty, enabled: .attributes.enabled}'

# Create a key
az keyvault key create --vault-name <vault-name> --name <key-name> --kty RSA --size 2048

# Delete a key
az keyvault key delete --vault-name <vault-name> --name <key-name>
```

### Access Policies

```sh
# Get access policies
az keyvault show --name <vault-name> --output json | jq '.properties.accessPolicies[] | {objectId, permissions: {keys: .permissions.keys, secrets: .permissions.secrets, certificates: .permissions.certificates}}'

# Set access policy for a user/SP
az keyvault set-policy --name <vault-name> --object-id <object-id> --secret-permissions get list --key-permissions get list

# Remove access policy
az keyvault delete-policy --name <vault-name> --object-id <object-id>
```

### Network Rules

```sh
# Get network rules
az keyvault show --name <vault-name> --output json | jq '.properties.networkAcls | {bypass, defaultAction, ipRules: [.ipRules[].value], virtualNetworkRules: [.virtualNetworkRules[].id]}'

# Add network rule (allow from IP)
az keyvault network-rule add --name <vault-name> --ip-address 203.0.113.0/24

# Set default action to deny
az keyvault update --name <vault-name> --set properties.networkAcls.defaultAction=Deny
```

### Security Status

```sh
# Check soft delete and purge protection
az keyvault list --output json | jq '.[] | {name, softDelete: .properties.enableSoftDelete, purgeProtection: .properties.enablePurgeProtection}'
```

---

## Monitoring and Logs

### Activity Logs

```sh
# Get recent activity log entries
az monitor activity-log list --max-events 50 --output table

# Filter by resource group
az monitor activity-log list --resource-group <rg> --max-events 20 --output json | jq '.[] | {eventTimestamp, level, operationName, resourceId}'

# Get failed operations
az monitor activity-log list --max-events 100 --output json | jq '.[] | select(.level == "Error" or .level == "Critical") | {eventTimestamp, level, operationName, status}'

# Count operations by caller
az monitor activity-log list --max-events 100 --output json | jq 'group_by(.caller) | map({caller: .[0].caller, count: length})'
```

### Metrics

```sh
# List metric definitions for a VM
az monitor metrics list-definitions \
  --resource /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/<vm> \
  --output json | jq '.[] | {name: .name.value, unit, primaryAggregationType}'

# Get CPU metrics for a VM
az monitor metrics list \
  --resource /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/<vm> \
  --metric "Percentage CPU" \
  --output json

# Get metrics for the last hour
az monitor metrics list \
  --resource <resource-id> \
  --metric "Percentage CPU" \
  --start-time $(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ') \
  --end-time $(date -u '+%Y-%m-%dT%H:%M:%SZ') \
  --interval PT1M \
  --output json
```

### Alerts

```sh
# List alert rules
az monitor metrics alert list --resource-group <rg> --output table

# Create a metric alert (CPU > 80%)
az monitor metrics alert create \
  --resource-group <rg> \
  --name "high-cpu-alert" \
  --scopes <vm-resource-id> \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m

# Delete an alert
az monitor metrics alert delete --resource-group <rg> --name "high-cpu-alert"
```

### Log Analytics

```sh
# List Log Analytics workspaces
az monitor log-analytics workspace list --output table

# List workspaces with jq
az monitor log-analytics workspace list --output json | jq '.[] | {name, resourceGroup, location, sku: .sku.name, retentionInDays}'

# Create a workspace
az monitor log-analytics workspace create \
  --resource-group <rg> \
  --workspace-name <name> \
  --location westeurope \
  --retention-time 30

# Run a query
az monitor log-analytics query \
  --workspace <workspace-id> \
  --analytics-query "Heartbeat | summarize count() by Computer" \
  --output json
```

---

## Cost Management

### Usage and Cost Analysis

```sh
# Get recent usage data
az consumption usage list --top 10 --output json | jq '.[] | {instanceName, meterName, quantity, unit, cost: .pretaxCost}'

# List budgets
az consumption budget list --output json | jq '.[] | {name, amount, timeGrain, category}'

# Create a budget
az consumption budget create \
  --budget-name "monthly-limit" \
  --amount 1000 \
  --time-grain monthly \
  --category cost \
  --start-date 2024-01-01 \
  --end-date 2025-12-31
```

### Resource Optimization

```sh
# Find unused public IPs (costing money)
az network public-ip list --output json | jq '.[] | select(.ipConfiguration == null) | {name, resourceGroup, ipAddress, location}'

# Find stopped VMs (still incurring disk costs)
az vm list --show-details --output json | jq '.[] | select(.powerState != "VM running") | {name, resourceGroup, size, powerState}'

# Find unattached managed disks
az disk list --output json | jq '.[] | select(.managedBy == null) | {name, resourceGroup, diskSizeGb, sku: .sku.name}'

# Find unattached NICs
az network nic list --output json | jq '.[] | select(.virtualMachine == null) | {name, resourceGroup}'

# Cost savings summary
{
  echo '{"public_ips":'; az network public-ip list --output json;
  echo ',"disks":'; az disk list --output json;
  echo ',"nics":'; az network nic list --output json;
} | jq '{
  unused_public_ips: [.public_ips[] | select(.ipConfiguration == null)] | length,
  unattached_disks: [.disks[] | select(.managedBy == null)] | length,
  orphaned_nics: [.nics[] | select(.virtualMachine == null)] | length
}'
```

---

## Identity and Access Management

### Role Assignments

```sh
# List role assignments
az role assignment list --output table

# List with jq
az role assignment list --output json | jq '.[] | {principalName, roleDefinitionName, scope}'

# Get role assignments for a resource group
az role assignment list --scope /subscriptions/<sub-id>/resourceGroups/<rg> --output json | jq '.[] | {principalName, roleDefinitionName, scope}'

# Find users with Owner or Contributor
az role assignment list --output json | jq '.[] | select(.roleDefinitionName == "Owner" or .roleDefinitionName == "Contributor") | {principalName, roleDefinitionName, scope}'

# Assign a role
az role assignment create \
  --assignee <user-or-sp> \
  --role "Contributor" \
  --resource-group <rg>

# Remove a role
az role assignment delete \
  --assignee <user-or-sp> \
  --role "Contributor" \
  --resource-group <rg>
```

### Custom Roles

```sh
# List custom roles
az role definition list --custom-role-only --output json | jq '.[] | {roleName, description, permissions: .permissions[].actions}'

# Create a custom role
az role definition create --role-definition @custom-role.json
```

### Service Principals

```sh
# List service principals
az ad sp list --all --output json | jq '.[] | {displayName, appId, objectId}'

# Create a service principal
az ad sp create-for-rbac --name <sp-name> --role Contributor --scopes /subscriptions/<sub-id>/resourceGroups/<rg>

# Reset service principal credentials
az ad sp credential reset --id <app-id>
```

### Managed Identities

```sh
# List managed identities
az identity list --output json | jq '.[] | {name, resourceGroup, location, principalId, clientId}'

# Create a user-assigned managed identity
az identity create --resource-group <rg> --name <identity-name>

# Find VMs with managed identity
az vm list --output json | jq '.[] | select(.identity != null) | {name, identityType: .identity.type, principalId: .identity.principalId}'
```

### Azure AD

```sh
# List Azure AD groups
az ad group list --output json | jq '.[] | {displayName, objectId, description}'

# List group members
az ad group member list --group <group-id> --output json | jq '.[] | {displayName, userPrincipalName, objectId}'

# List Azure AD applications
az ad app list --output json | jq '.[] | {displayName, appId}'
```

---

## App Services

### App Service Management

```sh
# List app services
az webapp list --output table

# Get details with jq
az webapp list --output json | jq '.[] | {name, resourceGroup, location, state, hostNames, httpsOnly}'

# Find app services by runtime
az webapp list --output json | jq '.[] | select(.siteConfig.linuxFxVersion | contains("NODE")) | {name, runtime: .siteConfig.linuxFxVersion}'

# Count app services by location
az webapp list --output json | jq 'group_by(.location) | map({location: .[0].location, count: length})'

# Find stopped app services
az webapp list --output json | jq '.[] | select(.state == "Stopped") | {name, state, resourceGroup}'
```

### App Service Plans

```sh
# List app service plans
az appservice plan list --output table

# Plans with details
az appservice plan list --output json | jq '.[] | {name, resourceGroup, location, sku: .sku.name, numberOfSites}'

# Create a plan
az appservice plan create --resource-group <rg> --name <plan-name> --sku B1 --is-linux

# Scale up
az appservice plan update --resource-group <rg> --name <plan-name> --sku S1
```

### App Configuration

```sh
# Get app settings
az webapp config appsettings list --resource-group <rg> --name <app> --output json | jq '.[] | {name, value}'

# Set app settings
az webapp config appsettings set --resource-group <rg> --name <app> --settings KEY1=value1 KEY2=value2

# Get connection strings
az webapp config connection-string list --resource-group <rg> --name <app> --output json

# Get SSL/TLS status
az webapp list --output json | jq '.[] | {name, httpsOnly, clientCertEnabled}'

# Find app services with custom domains
az webapp list --output json | jq '.[] | select(.hostNames | length > 1) | {name, customDomains: .hostNames}'
```

### Deployment Slots

```sh
# List deployment slots
az webapp deployment slot list --resource-group <rg> --name <app> --output json | jq '.[] | {name, state, hostNames}'

# Create a slot
az webapp deployment slot create --resource-group <rg> --name <app> --slot staging

# Swap slots
az webapp deployment slot swap --resource-group <rg> --name <app> --slot staging --target-slot production
```

### Logs and Diagnostics

```sh
# Configure logging
az webapp log config --resource-group <rg> --name <app> --application-logging filesystem --level information

# Get log config
az webapp log config show --resource-group <rg> --name <app> --output json | jq '{applicationLogs, httpLogs, failedRequestsTracing, detailedErrorMessages}'

# Stream live logs
az webapp log tail --resource-group <rg> --name <app>

# Download logs
az webapp log download --resource-group <rg> --name <app> --log-file app-logs.zip
```

### Deployment

```sh
# Deploy from local zip
az webapp deployment source config-zip --resource-group <rg> --name <app> --src app.zip

# Deploy from GitHub
az webapp deployment source config \
  --resource-group <rg> \
  --name <app> \
  --repo-url https://github.com/<user>/<repo> \
  --branch main

# List deployments
az webapp deployment list-publishing-profiles --resource-group <rg> --name <app> --output json

# Restart
az webapp restart --resource-group <rg> --name <app>
```
