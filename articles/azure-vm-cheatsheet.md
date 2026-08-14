# Azure VM Management Cheatsheet

## Create a VM

```sh
# Basic Linux VM
az vm create \
  --resource-group <rg> \
  --name <vm-name> \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys

# Specify an existing SSH key
az vm create \
  --resource-group <rg> \
  --name <vm-name> \
  --image RedHat:RHEL:9-lvm:latest \
  --size Standard_D2s_v3 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_ed25519.pub

# Windows VM
az vm create \
  --resource-group <rg> \
  --name <vm-name> \
  --image Win2022Datacenter \
  --size Standard_D2s_v3 \
  --admin-username azureuser \
  --admin-password '<password>'

# With custom data (cloud-init)
az vm create \
  --resource-group <rg> \
  --name <vm-name> \
  --image Ubuntu2204 \
  --custom-data cloud-init.yaml \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys
```

## List and Show VMs

```sh
# List all VMs
az vm list --output table

# List VMs in a resource group
az vm list --resource-group <rg> --output table

# Show VM details
az vm show --resource-group <rg> --name <vm-name>

# Show VM with instance view (power state)
az vm show --resource-group <rg> --name <vm-name> --show-details --output table

# List VM sizes available in a region
az vm list-sizes --location westeurope --output table

# List running VMs with their IPs
az vm list --resource-group <rg> --show-details --query "[].{Name:name, State:powerState, PublicIP:publicIps, PrivateIP:privateIps}" --output table
```

## VM Power Management

```sh
# Start a VM
az vm start --resource-group <rg> --name <vm-name>

# Stop a VM (still billed for compute)
az vm stop --resource-group <rg> --name <vm-name>

# Deallocate a VM (stops billing)
az vm deallocate --resource-group <rg> --name <vm-name>

# Restart a VM
az vm restart --resource-group <rg> --name <vm-name>

# Redeploy a VM (moves to a new host)
az vm redeploy --resource-group <rg> --name <vm-name>

# Start all VMs in a resource group
az vm start --ids $(az vm list -g <rg> --query "[].id" -o tsv)

# Deallocate all VMs in a resource group
az vm deallocate --ids $(az vm list -g <rg> --query "[].id" -o tsv)
```

## VM Disks

```sh
# List disks
az disk list --resource-group <rg> --output table

# Create a managed disk
az disk create \
  --resource-group <rg> \
  --name <disk-name> \
  --size-gb 128 \
  --sku Premium_LRS

# Attach a disk to a VM
az vm disk attach \
  --resource-group <rg> \
  --vm-name <vm-name> \
  --name <disk-name>

# Detach a disk
az vm disk detach \
  --resource-group <rg> \
  --vm-name <vm-name> \
  --name <disk-name>

# Resize a disk (VM must be deallocated)
az disk update \
  --resource-group <rg> \
  --name <disk-name> \
  --size-gb 256

# Resize OS disk
az vm deallocate --resource-group <rg> --name <vm-name>
az disk update --resource-group <rg> --name <os-disk-name> --size-gb 128
az vm start --resource-group <rg> --name <vm-name>
```

## VM Networking

```sh
# List NICs for a VM
az vm nic list --resource-group <rg> --vm-name <vm-name> --output table

# Show public IP of a VM
az vm show --resource-group <rg> --name <vm-name> --show-details --query publicIps --output tsv

# Open a port
az vm open-port --resource-group <rg> --name <vm-name> --port 443 --priority 1010

# List NSG rules
az network nsg rule list --resource-group <rg> --nsg-name <nsg-name> --output table

# Associate an existing public IP
az network nic ip-config update \
  --resource-group <rg> \
  --nic-name <nic-name> \
  --name ipconfig1 \
  --public-ip-address <pip-name>
```

## VM Resize

```sh
# List available sizes for an existing VM
az vm list-vm-resize-options --resource-group <rg> --name <vm-name> --output table

# Resize a VM (may require deallocation)
az vm resize --resource-group <rg> --name <vm-name> --size Standard_D4s_v3

# If resize requires deallocation
az vm deallocate --resource-group <rg> --name <vm-name>
az vm resize --resource-group <rg> --name <vm-name> --size Standard_D4s_v3
az vm start --resource-group <rg> --name <vm-name>
```

## VM Images

```sh
# List popular images
az vm image list --output table

# List all Ubuntu images from Canonical
az vm image list --publisher Canonical --all --output table

# List RHEL images
az vm image list --publisher RedHat --offer RHEL --all --output table

# Show image details
az vm image show --urn Canonical:0001-com-ubuntu-server-jammy:22_04-lts:latest

# List image publishers in a region
az vm image list-publishers --location westeurope --output table

# Accept marketplace terms
az vm image terms accept --urn <publisher>:<offer>:<sku>:<version>
```

## Run Commands on a VM

```sh
# Run a shell command
az vm run-command invoke \
  --resource-group <rg> \
  --name <vm-name> \
  --command-id RunShellScript \
  --scripts "hostname && uptime"

# Run a script file
az vm run-command invoke \
  --resource-group <rg> \
  --name <vm-name> \
  --command-id RunShellScript \
  --scripts @script.sh

# Run a PowerShell command (Windows)
az vm run-command invoke \
  --resource-group <rg> \
  --name <vm-name> \
  --command-id RunPowerShellScript \
  --scripts "Get-Service | Where-Object {$_.Status -eq 'Running'}"
```

## Serial Console and Boot Diagnostics

```sh
# Enable boot diagnostics
az vm boot-diagnostics enable --resource-group <rg> --name <vm-name>

# Get boot diagnostics log
az vm boot-diagnostics get-boot-log --resource-group <rg> --name <vm-name>

# Get screenshot (serial console output)
az vm boot-diagnostics get-boot-log-uris --resource-group <rg> --name <vm-name>
```

## VM Extensions

```sh
# List extensions on a VM
az vm extension list --resource-group <rg> --vm-name <vm-name> --output table

# Install custom script extension
az vm extension set \
  --resource-group <rg> \
  --vm-name <vm-name> \
  --name CustomScript \
  --publisher Microsoft.Azure.Extensions \
  --settings '{"commandToExecute": "apt-get update && apt-get install -y nginx"}'

# Remove an extension
az vm extension delete --resource-group <rg> --vm-name <vm-name> --name <extension-name>
```

## Snapshots and Backup

```sh
# Create a snapshot from a disk
az snapshot create \
  --resource-group <rg> \
  --name <snapshot-name> \
  --source <disk-name-or-id>

# List snapshots
az snapshot list --resource-group <rg> --output table

# Create a disk from a snapshot
az disk create \
  --resource-group <rg> \
  --name <new-disk-name> \
  --source <snapshot-name>

# Delete a snapshot
az snapshot delete --resource-group <rg> --name <snapshot-name>
```

## Delete a VM

```sh
# Delete VM only (keeps disks, NICs, etc.)
az vm delete --resource-group <rg> --name <vm-name> --yes

# Delete VM and associated resources
az vm delete --resource-group <rg> --name <vm-name> --yes
az disk delete --resource-group <rg> --name <vm-name>_OsDisk --yes
az network nic delete --resource-group <rg> --name <vm-name>VMNic
az network public-ip delete --resource-group <rg> --name <vm-name>PublicIP
az network nsg delete --resource-group <rg> --name <vm-name>NSG
```

## Tags

```sh
# Add tags to a VM
az vm update --resource-group <rg> --name <vm-name> --set tags.env=prod tags.team=infra

# Remove a tag
az vm update --resource-group <rg> --name <vm-name> --remove tags.team

# Find VMs by tag
az vm list --query "[?tags.env=='prod']" --output table
```


## Gotchas

- **`az vm stop` vs `az vm deallocate`**: `stop` halts the OS but keeps the VM allocated — you're still billed for compute. Always use `deallocate` to stop billing.
- **Ephemeral OS disks**: VMs with ephemeral OS disks lose all data on deallocation. They're fast but not for persistent workloads.
- **Public IP changes on deallocation**: Dynamic public IPs are released when a VM is deallocated and you get a new one on start. Use Static allocation if you need a stable IP.
- **NIC cannot be detached while VM is running**: You must stop/deallocate before adding or removing NICs.
- **Extension ordering**: Multiple extensions don't guarantee execution order. Use `dependsOn` in ARM templates or run them sequentially.
- **Disk caching**: OS disks default to ReadWrite cache. Data disks default to None. For databases, use ReadOnly on data disks and None on log disks.

## Proximity Placement Groups

Minimize latency between VMs by placing them physically close:

```sh
# Create a proximity placement group
az ppg create --resource-group <rg> --name <ppg-name> --location westeurope

# Create a VM in the proximity placement group
az vm create \
  --resource-group <rg> \
  --name <vm-name> \
  --image Ubuntu2204 \
  --size Standard_D2s_v3 \
  --ppg <ppg-name> \
  --generate-ssh-keys

# List VMs in a proximity placement group
az ppg show --resource-group <rg> --name <ppg-name> --query "virtualMachines[].id" --output tsv
```

## Accelerated Networking

```sh
# Enable on a new VM (supported sizes only)
az vm create \
  --resource-group <rg> \
  --name <vm-name> \
  --image Ubuntu2204 \
  --size Standard_D2s_v3 \
  --accelerated-networking true \
  --generate-ssh-keys

# Enable on an existing NIC (VM must be deallocated)
az network nic update --resource-group <rg> --name <nic-name> --accelerated-networking true

# Check if enabled
az network nic show --resource-group <rg> --name <nic-name> --query enableAcceleratedNetworking --output tsv
```

## Availability Sets vs Availability Zones

| Feature | Availability Set | Availability Zone |
|---------|-----------------|-------------------|
| Scope | Single datacenter (fault/update domains) | Separate datacenters in a region |
| SLA | 99.95% | 99.99% |
| Best for | Legacy apps, older regions | Modern HA, zone-redundant deployments |

```sh
# Create an availability set
az vm availability-set create --resource-group <rg> --name <as-name> --platform-fault-domain-count 2

# Create a VM in an availability zone
az vm create --resource-group <rg> --name <vm-name> --image Ubuntu2204 --zone 1 --generate-ssh-keys
```

## One-Liners

```sh
# Get power state of all VMs in subscription
az vm list -d --query "[].{Name:name, RG:resourceGroup, State:powerState}" --output table

# Find all running VMs
az vm list -d --query "[?powerState=='VM running'].{Name:name, RG:resourceGroup, Size:hardwareProfile.vmSize}" --output table

# Deallocate all VMs in a resource group
az vm deallocate --ids $(az vm list -g <rg> --query "[].id" -o tsv)

# Get total disk usage across all VMs in a resource group
az disk list -g <rg> --query "[].diskSizeGb" -o tsv | paste -sd+ | bc

# List VMs without tags
az vm list --query "[?tags==null || length(keys(tags))==\`0\`].{Name:name, RG:resourceGroup}" --output table

# Restart all VMs matching a name pattern
az vm list --query "[?contains(name, 'web')].id" -o tsv | xargs az vm restart --ids
```
