# Azure Storage Cheatsheet

## Storage Accounts

### Create a Storage Account

```sh
az storage account create \
  --resource-group <rg> \
  --name <account-name> \
  --location westeurope \
  --sku Standard_LRS \
  --kind StorageV2

# With geo-redundant storage
az storage account create \
  --resource-group <rg> \
  --name <account-name> \
  --location westeurope \
  --sku Standard_GRS \
  --kind StorageV2

# With network rules (deny by default)
az storage account create \
  --resource-group <rg> \
  --name <account-name> \
  --location westeurope \
  --sku Standard_LRS \
  --default-action Deny
```

### List Storage Accounts

```sh
az storage account list --resource-group <rg> --output table

# All in subscription
az storage account list --output table
```

### Show Account Details

```sh
az storage account show --resource-group <rg> --name <account-name>
```

### Get Connection String

```sh
az storage account show-connection-string --resource-group <rg> --name <account-name> --output tsv
```

### Get Access Keys

```sh
az storage account keys list --resource-group <rg> --account-name <account-name> --output table

# Regenerate a key
az storage account keys renew --resource-group <rg> --account-name <account-name> --key primary
```

### Delete a Storage Account

```sh
az storage account delete --resource-group <rg> --name <account-name> --yes
```

## SKU Options

| SKU | Description |
|-----|-------------|
| `Standard_LRS` | Locally redundant (3 copies in one datacenter) |
| `Standard_ZRS` | Zone-redundant (3 copies across availability zones) |
| `Standard_GRS` | Geo-redundant (6 copies, 2 regions) |
| `Standard_RAGRS` | Read-access geo-redundant |
| `Premium_LRS` | Premium SSD, locally redundant |

## Blob Storage

### Create a Container

```sh
az storage container create \
  --account-name <account-name> \
  --name <container-name>

# With public access
az storage container create \
  --account-name <account-name> \
  --name <container-name> \
  --public-access blob
```

### List Containers

```sh
az storage container list --account-name <account-name> --output table
```

### Upload Files

```sh
# Upload a single file
az storage blob upload \
  --account-name <account-name> \
  --container-name <container-name> \
  --name <blob-name> \
  --file <local-file-path>

# Upload all files in a directory
az storage blob upload-batch \
  --account-name <account-name> \
  --destination <container-name> \
  --source <local-directory>

# Upload with overwrite
az storage blob upload \
  --account-name <account-name> \
  --container-name <container-name> \
  --name <blob-name> \
  --file <local-file-path> \
  --overwrite
```

### Download Files

```sh
# Download a single blob
az storage blob download \
  --account-name <account-name> \
  --container-name <container-name> \
  --name <blob-name> \
  --file <local-file-path>

# Download all blobs in a container
az storage blob download-batch \
  --account-name <account-name> \
  --source <container-name> \
  --destination <local-directory>
```

### List Blobs

```sh
az storage blob list \
  --account-name <account-name> \
  --container-name <container-name> \
  --output table

# With prefix filter
az storage blob list \
  --account-name <account-name> \
  --container-name <container-name> \
  --prefix "logs/2024/" \
  --output table
```

### Delete Blobs

```sh
# Delete a single blob
az storage blob delete \
  --account-name <account-name> \
  --container-name <container-name> \
  --name <blob-name>

# Delete all blobs in a container
az storage blob delete-batch \
  --account-name <account-name> \
  --source <container-name>

# Delete blobs matching a pattern
az storage blob delete-batch \
  --account-name <account-name> \
  --source <container-name> \
  --pattern "*.log"
```

### Copy Blobs

```sh
# Copy within same account
az storage blob copy start \
  --account-name <account-name> \
  --destination-container <dest-container> \
  --destination-blob <dest-name> \
  --source-container <src-container> \
  --source-blob <src-name>

# Copy from URL
az storage blob copy start \
  --account-name <account-name> \
  --destination-container <dest-container> \
  --destination-blob <dest-name> \
  --source-uri <source-url>
```

### Generate SAS Token

```sh
# Account-level SAS
az storage account generate-sas \
  --account-name <account-name> \
  --permissions rwdlacup \
  --resource-types sco \
  --services b \
  --expiry 2025-12-31T00:00:00Z \
  --output tsv

# Container-level SAS
az storage container generate-sas \
  --account-name <account-name> \
  --name <container-name> \
  --permissions rwdl \
  --expiry 2025-12-31T00:00:00Z \
  --output tsv

# Blob-level SAS
az storage blob generate-sas \
  --account-name <account-name> \
  --container-name <container-name> \
  --name <blob-name> \
  --permissions r \
  --expiry 2025-12-31T00:00:00Z \
  --full-uri
```

### Blob Access Tiers

```sh
# Set blob tier
az storage blob set-tier \
  --account-name <account-name> \
  --container-name <container-name> \
  --name <blob-name> \
  --tier Cool

# Tiers: Hot, Cool, Cold, Archive
```

| Tier | Use case |
|------|----------|
| `Hot` | Frequently accessed data |
| `Cool` | Infrequently accessed, stored 30+ days |
| `Cold` | Rarely accessed, stored 90+ days |
| `Archive` | Long-term backup, stored 180+ days (offline) |

## File Shares

```sh
# Create a file share
az storage share-rm create \
  --resource-group <rg> \
  --storage-account <account-name> \
  --name <share-name> \
  --quota 100

# List file shares
az storage share-rm list --resource-group <rg> --storage-account <account-name> --output table

# Upload a file
az storage file upload \
  --account-name <account-name> \
  --share-name <share-name> \
  --source <local-file-path>

# List files
az storage file list --account-name <account-name> --share-name <share-name> --output table

# Delete a share
az storage share-rm delete --resource-group <rg> --storage-account <account-name> --name <share-name> --yes
```

### Mount on Linux

```sh
sudo mount -t cifs //<account-name>.file.core.windows.net/<share-name> /mnt/<mount-point> \
  -o vers=3.0,username=<account-name>,password=<account-key>,dir_mode=0777,file_mode=0777
```

## Managed Disks

```sh
# Create a managed disk
az disk create \
  --resource-group <rg> \
  --name <disk-name> \
  --size-gb 128 \
  --sku Premium_LRS \
  --location westeurope

# List disks
az disk list --resource-group <rg> --output table

# Show disk details
az disk show --resource-group <rg> --name <disk-name>

# Resize a disk (VM must be deallocated)
az disk update --resource-group <rg> --name <disk-name> --size-gb 256

# Change disk SKU
az disk update --resource-group <rg> --name <disk-name> --sku StandardSSD_LRS

# Delete a disk
az disk delete --resource-group <rg> --name <disk-name> --yes
```

### Disk SKUs

| SKU | Description |
|-----|-------------|
| `Standard_LRS` | Standard HDD, locally redundant |
| `StandardSSD_LRS` | Standard SSD, locally redundant |
| `Premium_LRS` | Premium SSD, locally redundant |
| `UltraSSD_LRS` | Ultra disk, sub-ms latency |

## Network Rules

```sh
# Set default action to deny
az storage account update --resource-group <rg> --name <account-name> --default-action Deny

# Allow access from a subnet
az storage account network-rule add \
  --resource-group <rg> \
  --account-name <account-name> \
  --subnet <subnet-resource-id>

# Allow access from an IP
az storage account network-rule add \
  --resource-group <rg> \
  --account-name <account-name> \
  --ip-address 203.0.113.0/24

# List network rules
az storage account network-rule list --resource-group <rg> --account-name <account-name> --output table

# Remove an IP rule
az storage account network-rule remove \
  --resource-group <rg> \
  --account-name <account-name> \
  --ip-address 203.0.113.0/24
```

## Lifecycle Management

```sh
# Create a lifecycle policy (move to cool after 30 days, archive after 90, delete after 365)
az storage account management-policy create \
  --resource-group <rg> \
  --account-name <account-name> \
  --policy @policy.json
```

Example `policy.json`:

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "lifecycle-rule",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
            "delete": { "daysAfterModificationGreaterThan": 365 }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["logs/"]
        }
      }
    }
  ]
}
```

## AzCopy

For large data transfers, use `azcopy` instead of `az storage blob`:

```sh
# Login
azcopy login

# Upload a directory
azcopy copy "/local/path/*" "https://<account>.blob.core.windows.net/<container>/<path>?<sas>" --recursive

# Download a directory
azcopy copy "https://<account>.blob.core.windows.net/<container>/<path>?<sas>" "/local/path/" --recursive

# Sync (only copy changed files)
azcopy sync "/local/path" "https://<account>.blob.core.windows.net/<container>?<sas>" --recursive

# Copy between storage accounts
azcopy copy "https://<src-account>.blob.core.windows.net/<container>?<sas>" \
  "https://<dest-account>.blob.core.windows.net/<container>?<sas>" --recursive
```


## Gotchas

- **Storage account names are globally unique**: Must be 3-24 characters, lowercase letters and numbers only. No hyphens, no uppercase.
- **`az vm stop` vs `deallocate`**: When a VM is stopped (not deallocated), its managed disks still incur charges. Always deallocate to stop all billing.
- **Archive tier is offline**: You cannot read archived blobs directly. You must rehydrate them first (takes up to 15 hours for standard priority).
- **Soft delete is not enabled by default**: Enable it proactively to protect against accidental deletion.
- **SAS tokens**: Once generated, they cannot be revoked individually. To invalidate, regenerate the storage account key used to sign them.
- **Firewall rules**: If you set default action to Deny, you must also allow Azure services or your own portal access will break. Add `--bypass AzureServices`.

## Soft Delete and Versioning

```sh
# Enable soft delete for blobs (retain for 14 days)
az storage account blob-service-properties update \
  --resource-group <rg> \
  --account-name <account-name> \
  --enable-delete-retention true \
  --delete-retention-days 14

# Enable soft delete for containers
az storage account blob-service-properties update \
  --resource-group <rg> \
  --account-name <account-name> \
  --enable-container-delete-retention true \
  --container-delete-retention-days 7

# Enable versioning
az storage account blob-service-properties update \
  --resource-group <rg> \
  --account-name <account-name> \
  --enable-versioning true

# List soft-deleted blobs
az storage blob list \
  --account-name <account-name> \
  --container-name <container-name> \
  --include d \
  --output table

# Undelete a blob
az storage blob undelete \
  --account-name <account-name> \
  --container-name <container-name> \
  --name <blob-name>
```

## Immutable Blobs (WORM)

```sh
# Set a time-based retention policy (compliance)
az storage container immutability-policy create \
  --account-name <account-name> \
  --container-name <container-name> \
  --period 365

# Set a legal hold
az storage container legal-hold set \
  --account-name <account-name> \
  --container-name <container-name> \
  --tags "case-001"

# Remove a legal hold
az storage container legal-hold clear \
  --account-name <account-name> \
  --container-name <container-name> \
  --tags "case-001"
```

## Diagnostic Logging

```sh
# Enable storage analytics logging for blobs
az storage logging update \
  --account-name <account-name> \
  --log rwd \
  --retention 30 \
  --services b

# Enable metrics
az storage metrics update \
  --account-name <account-name> \
  --hour true \
  --minute true \
  --retention 30 \
  --services b
```

## One-Liners

```sh
# Check blob properties and last modified time
az storage blob show --account-name <account-name> --container-name <container-name> --name <blob-name> --query "{Size:properties.contentLength, Modified:properties.lastModified, Tier:properties.blobTier}" --output table

# Get total size of all blobs in a container (in bytes)
az storage blob list --account-name <account-name> --container-name <container-name> --query "[].properties.contentLength" -o tsv | paste -sd+ | bc

# Find all blobs larger than 1GB
az storage blob list --account-name <account-name> --container-name <container-name> --query "[?properties.contentLength > \`1073741824\`].{Name:name, SizeGB:properties.contentLength}" --output table

# List all storage accounts and their SKUs
az storage account list --query "[].{Name:name, SKU:sku.name, Kind:kind, Location:location}" --output table

# Quickly check if a storage account allows public blob access
az storage account show --name <account-name> --query allowBlobPublicAccess --output tsv
```


## Using jq with Storage Commands

### Storage Account Analysis

```sh
# Get storage accounts with access tiers and SKU
az storage account list --output json | jq '.[] | {name, resourceGroup, location, sku: .sku.name, accessTier, kind}'

# Find Premium storage accounts
az storage account list --output json | jq '.[] | select(.sku.tier == "Premium") | {name, sku: .sku.name, location}'

# Check security settings
az storage account list --output json | jq '.[] | {name, allowBlobPublicAccess, minimumTlsVersion, publicNetworkAccess}'

# Get storage account keys
az storage account keys list --resource-group <rg> --account-name <account> --output json | jq '.[] | {keyName, permissions}'
```

### Blob Analysis with jq

```sh
# List blobs with metadata
az storage blob list --container-name <container> --account-name <account> --output json | jq '.[] | {name, lastModified, size: .properties.contentLength, contentType: .properties.contentType}'

# Find large blobs (over 1GB)
az storage blob list --container-name <container> --account-name <account> --output json | jq '.[] | select(.properties.contentLength > 1073741824) | {name, sizeGB: (.properties.contentLength / 1073741824 * 100 | floor / 100)}'

# Group blobs by content type with total size
az storage blob list --container-name <container> --account-name <account> --output json | jq 'group_by(.properties.contentType) | map({contentType: .[0].properties.contentType, count: length, totalSizeMB: ([.[].properties.contentLength] | add / 1048576 | floor)})'

# List containers with public access status
az storage container list --account-name <account> --output json | jq '.[] | {name, lastModified, publicAccess}'
```

### Storage Account Usage

```sh
# Get storage account usage in a region
az storage account show-usage --location eastus --output json | jq '.[] | {name: .name.value, currentValue, limit, unit}'
```
