# Azure Networking Cheatsheet

## Virtual Networks (VNets)

### Create a VNet

```sh
az network vnet create \
  --resource-group <rg> \
  --name <vnet-name> \
  --address-prefix 10.0.0.0/16 \
  --location westeurope

# With a subnet in one command
az network vnet create \
  --resource-group <rg> \
  --name <vnet-name> \
  --address-prefix 10.0.0.0/16 \
  --subnet-name default \
  --subnet-prefixes 10.0.1.0/24
```

### List VNets

```sh
az network vnet list --resource-group <rg> --output table

# All VNets in subscription
az network vnet list --output table
```

### Show VNet Details

```sh
az network vnet show --resource-group <rg> --name <vnet-name>

# Show address space
az network vnet show --resource-group <rg> --name <vnet-name> \
  --query "addressSpace.addressPrefixes" --output tsv
```

### Delete a VNet

```sh
az network vnet delete --resource-group <rg> --name <vnet-name>
```

## Subnets

### Create a Subnet

```sh
az network vnet subnet create \
  --resource-group <rg> \
  --vnet-name <vnet-name> \
  --name <subnet-name> \
  --address-prefixes 10.0.2.0/24

# With an NSG attached
az network vnet subnet create \
  --resource-group <rg> \
  --vnet-name <vnet-name> \
  --name <subnet-name> \
  --address-prefixes 10.0.3.0/24 \
  --network-security-group <nsg-name>

# With service endpoints
az network vnet subnet create \
  --resource-group <rg> \
  --vnet-name <vnet-name> \
  --name <subnet-name> \
  --address-prefixes 10.0.4.0/24 \
  --service-endpoints Microsoft.Storage Microsoft.Sql
```

### List Subnets

```sh
az network vnet subnet list --resource-group <rg> --vnet-name <vnet-name> --output table
```

### Show Subnet Details

```sh
az network vnet subnet show --resource-group <rg> --vnet-name <vnet-name> --name <subnet-name>
```

### Update a Subnet

```sh
# Associate an NSG
az network vnet subnet update \
  --resource-group <rg> \
  --vnet-name <vnet-name> \
  --name <subnet-name> \
  --network-security-group <nsg-name>

# Remove NSG association
az network vnet subnet update \
  --resource-group <rg> \
  --vnet-name <vnet-name> \
  --name <subnet-name> \
  --network-security-group ""
```

### Delete a Subnet

```sh
az network vnet subnet delete --resource-group <rg> --vnet-name <vnet-name> --name <subnet-name>
```

## Network Security Groups (NSGs)

### Create an NSG

```sh
az network nsg create --resource-group <rg> --name <nsg-name> --location westeurope
```

### List NSGs

```sh
az network nsg list --resource-group <rg> --output table
```

### Show NSG Rules

```sh
az network nsg rule list --resource-group <rg> --nsg-name <nsg-name> --output table

# Include default rules
az network nsg rule list --resource-group <rg> --nsg-name <nsg-name> --include-default --output table
```

### Create NSG Rules

```sh
# Allow SSH from a specific IP
az network nsg rule create \
  --resource-group <rg> \
  --nsg-name <nsg-name> \
  --name AllowSSH \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes 203.0.113.0/24 \
  --destination-port-ranges 22

# Allow HTTPS from anywhere
az network nsg rule create \
  --resource-group <rg> \
  --nsg-name <nsg-name> \
  --name AllowHTTPS \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes "*" \
  --destination-port-ranges 443

# Allow multiple ports
az network nsg rule create \
  --resource-group <rg> \
  --nsg-name <nsg-name> \
  --name AllowWebTraffic \
  --priority 120 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes "*" \
  --destination-port-ranges 80 443 8080

# Deny all inbound (explicit deny at low priority)
az network nsg rule create \
  --resource-group <rg> \
  --nsg-name <nsg-name> \
  --name DenyAllInbound \
  --priority 4000 \
  --direction Inbound \
  --access Deny \
  --protocol "*" \
  --source-address-prefixes "*" \
  --destination-port-ranges "*"

# Allow outbound to a service tag
az network nsg rule create \
  --resource-group <rg> \
  --nsg-name <nsg-name> \
  --name AllowStorageOutbound \
  --priority 100 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --destination-address-prefixes Storage \
  --destination-port-ranges 443
```

### Delete an NSG Rule

```sh
az network nsg rule delete --resource-group <rg> --nsg-name <nsg-name> --name <rule-name>
```

### Delete an NSG

```sh
az network nsg delete --resource-group <rg> --name <nsg-name>
```

## Public IP Addresses

```sh
# Create a public IP
az network public-ip create \
  --resource-group <rg> \
  --name <pip-name> \
  --sku Standard \
  --allocation-method Static

# List public IPs
az network public-ip list --resource-group <rg> --output table

# Show a public IP address value
az network public-ip show --resource-group <rg> --name <pip-name> --query ipAddress --output tsv

# Delete a public IP
az network public-ip delete --resource-group <rg> --name <pip-name>
```

## Network Interfaces (NICs)

```sh
# Create a NIC
az network nic create \
  --resource-group <rg> \
  --name <nic-name> \
  --vnet-name <vnet-name> \
  --subnet <subnet-name>

# List NICs
az network nic list --resource-group <rg> --output table

# Show NIC IP configuration
az network nic ip-config list --resource-group <rg> --nic-name <nic-name> --output table

# Associate a public IP with a NIC
az network nic ip-config update \
  --resource-group <rg> \
  --nic-name <nic-name> \
  --name ipconfig1 \
  --public-ip-address <pip-name>

# Remove public IP from a NIC
az network nic ip-config update \
  --resource-group <rg> \
  --nic-name <nic-name> \
  --name ipconfig1 \
  --public-ip-address ""
```

## VNet Peering

```sh
# Create peering from vnet1 to vnet2
az network vnet peering create \
  --resource-group <rg1> \
  --name vnet1-to-vnet2 \
  --vnet-name <vnet1> \
  --remote-vnet <vnet2-resource-id> \
  --allow-vnet-access

# Create peering from vnet2 to vnet1 (both directions required)
az network vnet peering create \
  --resource-group <rg2> \
  --name vnet2-to-vnet1 \
  --vnet-name <vnet2> \
  --remote-vnet <vnet1-resource-id> \
  --allow-vnet-access

# List peerings
az network vnet peering list --resource-group <rg> --vnet-name <vnet-name> --output table

# Show peering state
az network vnet peering show \
  --resource-group <rg> \
  --vnet-name <vnet-name> \
  --name <peering-name> \
  --query peeringState --output tsv

# Delete a peering
az network vnet peering delete --resource-group <rg> --vnet-name <vnet-name> --name <peering-name>
```

## DNS

```sh
# Create a private DNS zone
az network private-dns zone create --resource-group <rg> --name <zone-name>

# Link a private DNS zone to a VNet
az network private-dns link vnet create \
  --resource-group <rg> \
  --zone-name <zone-name> \
  --name <link-name> \
  --virtual-network <vnet-name> \
  --registration-enabled true

# Add an A record
az network private-dns record-set a add-record \
  --resource-group <rg> \
  --zone-name <zone-name> \
  --record-set-name <hostname> \
  --ipv4-address 10.0.1.10

# List records
az network private-dns record-set list --resource-group <rg> --zone-name <zone-name> --output table
```

## Load Balancers

```sh
# Create a public load balancer
az network lb create \
  --resource-group <rg> \
  --name <lb-name> \
  --sku Standard \
  --frontend-ip-name <frontend-name> \
  --public-ip-address <pip-name> \
  --backend-pool-name <backend-pool>

# Create a health probe
az network lb probe create \
  --resource-group <rg> \
  --lb-name <lb-name> \
  --name <probe-name> \
  --protocol Tcp \
  --port 80

# Create a load balancing rule
az network lb rule create \
  --resource-group <rg> \
  --lb-name <lb-name> \
  --name <rule-name> \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name <frontend-name> \
  --backend-pool-name <backend-pool> \
  --probe-name <probe-name>

# List load balancers
az network lb list --resource-group <rg> --output table
```

## Troubleshooting

```sh
# Check effective NSG rules on a NIC
az network nic list-effective-nsg --resource-group <rg> --name <nic-name>

# Check effective routes on a NIC
az network nic show-effective-route-table --resource-group <rg> --name <nic-name> --output table

# Test IP flow (verify traffic between two IPs)
az network watcher test-ip-flow \
  --resource-group <rg> \
  --vm <vm-name> \
  --direction Inbound \
  --protocol Tcp \
  --local 10.0.1.4:22 \
  --remote 203.0.113.5:*

# Show topology
az network watcher show-topology --resource-group <rg>
```

## Service Tags

Common service tags for use in NSG rules:

| Tag | Description |
|-----|-------------|
| `Internet` | All public internet addresses |
| `VirtualNetwork` | VNet address space + peered VNets |
| `AzureLoadBalancer` | Azure infrastructure load balancer |
| `Storage` | Azure Storage service IPs |
| `Sql` | Azure SQL Database IPs |
| `AzureActiveDirectory` | Azure AD IPs |
| `AzureMonitor` | Azure Monitor, Log Analytics, App Insights |


## Gotchas

- **NSG rules are stateful**: If you allow inbound traffic on port 443, the return traffic is automatically allowed. You don't need a matching outbound rule.
- **Subnet ranges cannot overlap**: Two subnets in the same VNet cannot share any part of their address space. Plan your CIDR blocks carefully upfront.
- **5 reserved addresses per subnet**: Azure reserves the first 4 and last IP in every subnet. A /24 gives you 251 usable IPs, not 254.
  - `.0` — Network address
  - `.1` — Default gateway
  - `.2`, `.3` — Azure DNS mapping
  - `.255` — Broadcast
- **NSGs can be applied to subnets OR NICs**: If both are configured, BOTH must allow the traffic. A common debugging mistake is checking only one.
- **VNet peering is non-transitive**: If VNet A is peered with B, and B is peered with C, A cannot reach C unless explicitly peered.
- **Default outbound access is deprecated**: New VMs without an explicit outbound method (NAT Gateway, Load Balancer, or public IP) may not have internet access.

## User Defined Routes (UDRs)

```sh
# Create a route table
az network route-table create --resource-group <rg> --name <rt-name>

# Add a route (force traffic through an NVA)
az network route-table route create \
  --resource-group <rg> \
  --route-table-name <rt-name> \
  --name to-internet \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.2.4

# Associate route table with a subnet
az network vnet subnet update \
  --resource-group <rg> \
  --vnet-name <vnet-name> \
  --name <subnet-name> \
  --route-table <rt-name>

# List routes
az network route-table route list --resource-group <rg> --route-table-name <rt-name> --output table
```

## Network Watcher

```sh
# Enable Network Watcher in a region
az network watcher configure --resource-group NetworkWatcherRG --locations westeurope --enabled true

# Connection troubleshoot (test connectivity between two resources)
az network watcher test-connectivity \
  --resource-group <rg> \
  --source-resource <vm-id> \
  --dest-address <ip-or-fqdn> \
  --dest-port 443

# IP flow verify
az network watcher test-ip-flow \
  --resource-group <rg> \
  --vm <vm-name> \
  --direction Inbound \
  --protocol Tcp \
  --local 10.0.1.4:80 \
  --remote 203.0.113.5:*

# Next hop
az network watcher show-next-hop \
  --resource-group <rg> \
  --vm <vm-name> \
  --source-ip 10.0.1.4 \
  --dest-ip 10.0.2.4
```

## One-Liners

```sh
# List all NSG rules that allow SSH from anywhere
az network nsg list --query "[].{NSG:name, Rules:securityRules[?destinationPortRange=='22' && sourceAddressPrefix=='*' && access=='Allow'].name}" --output json

# Find all public IPs in use
az network public-ip list --query "[?ipAddress!=null].{Name:name, IP:ipAddress, RG:resourceGroup}" --output table

# List all subnets and their address ranges across all VNets
az network vnet list --query "[].{VNet:name, Subnets:subnets[].{Name:name, Prefix:addressPrefix}}" --output json

# Check if a specific port is reachable from a VM
az network watcher test-ip-flow --vm <vm-name> -g <rg> --direction Inbound --protocol Tcp --local "10.0.1.4:3306" --remote "203.0.113.5:*"
```
