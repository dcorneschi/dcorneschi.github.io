# Hammer CLI Cheatsheet

Hammer is the command-line interface for Red Hat Satellite and Foreman. It interacts with the Satellite/Foreman API to manage hosts, content, provisioning, and configuration.

## Installation and Configuration

```bash
# Install on Satellite/Foreman server (usually pre-installed)
yum install rubygem-hammer_cli_foreman    # Foreman
yum install tfm-rubygem-hammer_cli_katello  # Satellite

# Check version
hammer --version

# Test connection
hammer ping
```

### Authentication

```bash
# Interactive (prompts for password)
hammer -u admin -p <password> host list

# Save credentials (avoids typing password every time)
mkdir -p ~/.hammer
cat > ~/.hammer/cli_modules.d/foreman.yml << EOF
:foreman:
  :host: 'https://satellite.example.com'
  :username: 'admin'
  :password: 'changeme'
EOF
chmod 600 ~/.hammer/cli_modules.d/foreman.yml

# Or use the global config
cat /etc/hammer/cli.modules.d/foreman.yml
```

### Output Formats

```bash
# Default table format
hammer host list

# CSV output
hammer --csv host list

# JSON output
hammer --output json host list

# YAML output
hammer --output yaml host list

# Specific fields only
hammer --csv host list --fields "Name,IP"
```

## Host Management

### List and Search Hosts

```bash
# List all hosts
hammer host list

# Search by name
hammer host list --search "name ~ web"

# Search by organization
hammer host list --organization "ACME"

# Search by hostgroup
hammer host list --search "hostgroup = Production"

# Search by OS
hammer host list --search "os = RedHat 8"

# Search by fact
hammer host list --search "facts.os.name = RedHat"

# Complex search
hammer host list --search "name ~ web AND os = RedHat 8 AND hostgroup = Production"
```

### Host Information

```bash
# Show detailed host info
hammer host info --name "server01.example.com"
hammer host info --id 42

# Show host facts
hammer host facts --name "server01.example.com"

# Show host reports
hammer host reports --id 42

# Show host subscriptions
hammer host subscription list --host "server01.example.com"
```

### Create and Delete Hosts

```bash
# Create a host
hammer host create \
  --name "newhost.example.com" \
  --organization "ACME" \
  --location "Default Location" \
  --hostgroup "RHEL8-Base" \
  --mac "aa:bb:cc:dd:ee:ff" \
  --interface "type=interface,mac=aa:bb:cc:dd:ee:ff,ip=192.168.1.100,primary=true,provision=true" \
  --build true

# Delete a host
hammer host delete --name "oldhost.example.com"

# Delete multiple hosts
hammer host list --search "name ~ temp" --csv | tail -n +2 | cut -d, -f2 | xargs -I {} hammer host delete --name {}
```

### Host Actions

```bash
# Set build mode (triggers reinstall on next boot)
hammer host update --name "server01.example.com" --build true

# Power management
hammer host start --name "server01.example.com"
hammer host stop --name "server01.example.com"
hammer host reboot --name "server01.example.com"
hammer host reset --name "server01.example.com"

# Run Puppet
hammer host puppetrun --name "server01.example.com"

# Change hostgroup
hammer host update --name "server01.example.com" --hostgroup "NewGroup"

# Change organization
hammer host update --name "server01.example.com" --new-organization "NewOrg"
```

## Content Management

### Organizations

```bash
# List organizations
hammer organization list

# Create organization
hammer organization create --name "ACME" --label "ACME"

# Show organization info
hammer organization info --name "ACME"

# Delete organization
hammer organization delete --name "ACME"
```

### Products and Repositories

```bash
# List products
hammer product list --organization "ACME"

# Create a product
hammer product create --name "Custom Software" --organization "ACME"

# Create a yum repository
hammer repository create \
  --name "My Repo" \
  --product "Custom Software" \
  --organization "ACME" \
  --content-type yum \
  --url "http://repo.example.com/rhel8/"

# Sync a repository
hammer repository synchronize \
  --name "My Repo" \
  --product "Custom Software" \
  --organization "ACME"

# Sync all repos in a product
hammer product synchronize --name "Custom Software" --organization "ACME"

# List repositories
hammer repository list --organization "ACME"

# Show repository info
hammer repository info --name "My Repo" --product "Custom Software" --organization "ACME"
```

### Content Views

```bash
# List content views
hammer content-view list --organization "ACME"

# Create a content view
hammer content-view create \
  --name "RHEL8-CV" \
  --organization "ACME"

# Add repository to content view
hammer content-view add-repository \
  --name "RHEL8-CV" \
  --organization "ACME" \
  --repository "Red Hat Enterprise Linux 8 for x86_64 - BaseOS RPMs 8" \
  --product "Red Hat Enterprise Linux for x86_64"

# Publish a content view (create new version)
hammer content-view publish \
  --name "RHEL8-CV" \
  --organization "ACME"

# Promote content view to lifecycle environment
hammer content-view version promote \
  --content-view "RHEL8-CV" \
  --organization "ACME" \
  --to-lifecycle-environment "Production" \
  --version "1.0"

# List content view versions
hammer content-view version list \
  --content-view "RHEL8-CV" \
  --organization "ACME"

# List all content view versions across all CVs
hammer content-view version list --organization "ACME"

# Remove old content view version
hammer content-view version delete \
  --content-view "RHEL8-CV" \
  --organization "ACME" \
  --version "1.0"
```

### Lifecycle Environments

```bash
# List lifecycle environments
hammer lifecycle-environment list --organization "ACME"

# Show lifecycle environment paths
hammer lifecycle-environment paths --organization "ACME"

# Create lifecycle environment
hammer lifecycle-environment create \
  --name "Development" \
  --prior "Library" \
  --organization "ACME"

hammer lifecycle-environment create \
  --name "Production" \
  --prior "Development" \
  --organization "ACME"

# Show paths
hammer lifecycle-environment paths --organization "ACME"
```

### Activation Keys

```bash
# List activation keys
hammer activation-key list --organization "ACME"

# Create activation key
hammer activation-key create \
  --name "rhel8-production" \
  --organization "ACME" \
  --lifecycle-environment "Production" \
  --content-view "RHEL8-CV" \
  --unlimited-hosts

# Add subscription to activation key
hammer activation-key add-subscription \
  --name "rhel8-production" \
  --organization "ACME" \
  --subscription-id 1

# Show activation key info
hammer activation-key info --name "rhel8-production" --organization "ACME"
```

### Subscriptions

```bash
# List subscriptions
hammer subscription list --organization "ACME"

# Upload subscription manifest
hammer subscription upload \
  --file /tmp/manifest.zip \
  --organization "ACME"

# Refresh manifest
hammer subscription refresh-manifest --organization "ACME"

# Delete manifest
hammer subscription delete-manifest --organization "ACME"
```

### Sync Plans

```bash
# List sync plans
hammer sync-plan list --organization "ACME"

# Create a sync plan
hammer sync-plan create \
  --name "Daily Sync" \
  --organization "ACME" \
  --interval daily \
  --sync-date "2024-01-01 00:00:00" \
  --enabled true

# Add product to sync plan
hammer product set-sync-plan \
  --name "Red Hat Enterprise Linux for x86_64" \
  --organization "ACME" \
  --sync-plan "Daily Sync"

# List products assigned to a sync plan
hammer product list --organization "ACME" --sync-plan "Daily Sync"
```

## Host Groups

```bash
# List hostgroups
hammer hostgroup list

# Create hostgroup
hammer hostgroup create \
  --name "RHEL8-Base" \
  --organization "ACME" \
  --location "Default Location" \
  --lifecycle-environment "Production" \
  --content-view "RHEL8-CV" \
  --content-source "satellite.example.com"

# Update hostgroup
hammer hostgroup update --name "RHEL8-Base" --new-name "RHEL8-Production"

# Set hostgroup parameters
hammer hostgroup set-parameter \
  --hostgroup "RHEL8-Base" \
  --name "ntp_server" \
  --value "ntp.example.com"

# Delete hostgroup
hammer hostgroup delete --name "OldGroup"
```

## Provisioning

### Compute Resources

```bash
# List compute resources
hammer compute-resource list

# Create VMware compute resource
hammer compute-resource create \
  --name "VMware" \
  --provider "Vmware" \
  --url "vcenter.example.com" \
  --user "admin@vsphere.local" \
  --password "secret" \
  --datacenter "DC1"

# List compute resource images
hammer compute-resource image list --compute-resource "VMware"
```

### Provisioning Templates

```bash
# List templates
hammer template list

# Show template content
hammer template dump --name "Kickstart default"

# Create template from file
hammer template create \
  --name "My Kickstart" \
  --type "provision" \
  --file /tmp/my_kickstart.erb

# Associate template with OS
hammer template add-operatingsystem \
  --name "My Kickstart" \
  --operatingsystem "RedHat 8.9"
```

### Operating Systems

```bash
# List operating systems
hammer os list

# Show OS details
hammer os info --id 1

# Create operating system
hammer os create \
  --name "RedHat" \
  --major 8 \
  --minor 9 \
  --family "Redhat"

# Set default templates for OS
hammer os set-default-template \
  --id 1 \
  --provisioning-template "Kickstart default"

# Associate partition table with OS
hammer os add-ptable \
  --id 1 \
  --partition-table "Kickstart default"
```

## Errata and Packages

```bash
# List errata for a host
hammer host errata list --host "server01.example.com"

# Apply specific erratum
hammer host errata apply \
  --host "server01.example.com" \
  --errata-ids "RHSA-2024:1234"

# List applicable errata for a host
hammer host errata list --host "server01.example.com" --search "type = security"

# List packages for a host
hammer host package list --host "server01.example.com"

# Install package on a host
hammer host package install --host "server01.example.com" --packages "vim-enhanced"

# Update packages
hammer host package update --host "server01.example.com" --packages "bash"

# Remove package
hammer host package remove --host "server01.example.com" --packages "vim-enhanced"
```

## Users and Permissions

```bash
# List users
hammer user list

# Create user
hammer user create \
  --login "jdoe" \
  --firstname "John" \
  --lastname "Doe" \
  --mail "jdoe@example.com" \
  --password "secret123" \
  --organization "ACME" \
  --location "Default Location"

# List roles
hammer role list

# Assign role to user
hammer user add-role --login "jdoe" --role "Manager"

# List permissions
hammer filter list --role "Manager"
```

## Smart Proxies (Capsules)

```bash
# List smart proxies
hammer capsule list

# Show capsule info
hammer capsule info --name "capsule01.example.com"

# Sync content to capsule
hammer capsule content synchronize --name "capsule01.example.com"

# List lifecycle environments on capsule
hammer capsule content lifecycle-environments --name "capsule01.example.com"

# Add lifecycle environment to capsule
hammer capsule content add-lifecycle-environment \
  --name "capsule01.example.com" \
  --lifecycle-environment "Production" \
  --organization "ACME"
```

## Remote Execution (REX)

```bash
# Run a command on a host
hammer job-invocation create \
  --job-template "Run Command - SSH Default" \
  --inputs "command=uptime" \
  --search-query "name = server01.example.com"

# Run command on multiple hosts
hammer job-invocation create \
  --job-template "Run Command - SSH Default" \
  --inputs "command=yum update -y" \
  --search-query "hostgroup = Production"

# List job invocations
hammer job-invocation list

# Show job status
hammer job-invocation info --id 42

# Show job output
hammer job-invocation output --id 42 --host "server01.example.com"
```

## Puppet

```bash
# List Puppet classes
hammer puppet-class list

# Import Puppet classes from proxy
hammer proxy import-classes --id 1

# Assign Puppet class to hostgroup
hammer hostgroup puppet-class add \
  --hostgroup "RHEL8-Base" \
  --puppet-class "ntp"

# List Puppet environments
hammer environment list

# Override a smart class parameter
hammer sc-param update \
  --puppet-class "ntp" \
  --parameter "servers" \
  --override true \
  --default-value '["ntp1.example.com", "ntp2.example.com"]'
```

## Discovery

```bash
# List discovered hosts
hammer discovery list

# Show discovered host info
hammer discovery info --name "mac001122334455"

# Provision a discovered host
hammer discovery provision \
  --name "mac001122334455" \
  --hostgroup "RHEL8-Base" \
  --organization "ACME" \
  --location "Default Location"

# Auto-provision by rule
hammer discovery-rule create \
  --name "web-servers" \
  --search "facts.memorysize_mb > 4096" \
  --hostgroup "Web-Servers" \
  --organization "ACME" \
  --location "Default Location" \
  --enabled true
```

## Domains, Subnets, and Networking

```bash
# List domains
hammer domain list

# Create domain
hammer domain create --name "example.com"

# List subnets
hammer subnet list

# Create subnet
hammer subnet create \
  --name "Production" \
  --network "192.168.1.0" \
  --mask "255.255.255.0" \
  --gateway "192.168.1.1" \
  --dns-primary "192.168.1.10" \
  --domain "example.com"
```

## Tasks and Status

```bash
# List running tasks
hammer task list --search "state = running"

# List recent tasks
hammer task list --order "started_at DESC" --per-page 20

# Show task details
hammer task info --id <task-id>

# Resume paused task
hammer task resume --id <task-id>

# Ping all services
hammer ping
```

## Useful Patterns

### Bulk Operations

```bash
# Register all hosts to a content view
for host in $(hammer --csv host list --search "os = RedHat 8" | tail -n +2 | cut -d, -f2); do
  hammer host update --name "$host" --content-view "RHEL8-CV" --lifecycle-environment "Production"
done

# Apply security errata to all production hosts
hammer job-invocation create \
  --job-template "Run Command - SSH Default" \
  --inputs "command=yum update --security -y" \
  --search-query "hostgroup = Production"

# Export host list to CSV
hammer --csv host list --per-page 1000 > hosts.csv
```

### Check Sync Status

```bash
# Check sync status for all repos
hammer repository list --organization "ACME" --fields "Name,Sync State,Last Sync Date"

# Find repos that failed to sync
hammer --csv repository list --organization "ACME" | grep -i "not_synced\|error"
```

### Content View Workflow

```bash
# Full workflow: publish and promote
hammer content-view publish --name "RHEL8-CV" --organization "ACME"

VERSION=$(hammer --csv content-view version list \
  --content-view "RHEL8-CV" \
  --organization "ACME" \
  --order "version DESC" \
  --per-page 1 | tail -1 | cut -d, -f3)

hammer content-view version promote \
  --content-view "RHEL8-CV" \
  --organization "ACME" \
  --to-lifecycle-environment "Development" \
  --version "$VERSION"
```

## Repository Set Management (Red Hat)

### Enable Red Hat Repository Sets

```bash
# RHEL 8 BaseOS
hammer repository-set enable \
  --organization "ACME" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --basearch "x86_64" \
  --releasever "8" \
  --name "Red Hat Enterprise Linux 8 for x86_64 - BaseOS (RPMs)"

# RHEL 8 AppStream
hammer repository-set enable \
  --organization "ACME" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --basearch "x86_64" \
  --releasever "8" \
  --name "Red Hat Enterprise Linux 8 for x86_64 - AppStream (RPMs)"

# RHEL 9 BaseOS
hammer repository-set enable \
  --organization "ACME" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --basearch "x86_64" \
  --releasever "9" \
  --name "Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)"

# List enabled repository sets
hammer repository-set list --organization "ACME" --product "Red Hat Enterprise Linux for x86_64"
```

## Ubuntu (DEB) Repositories

### Create Ubuntu Product and Repository

```bash
# Create product
hammer product create \
  --organization "ACME" \
  --name "Ubuntu 22.04 LTS" \
  --description "Ubuntu 22.04 LTS (Jammy Jellyfish) repositories"

# Create DEB repository
hammer repository create \
  --organization "ACME" \
  --product "Ubuntu 22.04 LTS" \
  --name "Ubuntu 22.04 Main" \
  --label "ubuntu_22_04_main" \
  --content-type "deb" \
  --url "http://archive.ubuntu.com/ubuntu/" \
  --deb-releases "jammy" \
  --deb-components "main" \
  --deb-architectures "amd64" \
  --download-policy "on_demand" \
  --mirror-on-sync "yes" \
  --gpg-key "Ubuntu Archive Key"

# Sync Ubuntu repository
hammer repository synchronize \
  --organization "ACME" \
  --product "Ubuntu 22.04 LTS" \
  --name "Ubuntu 22.04 Main" \
  --async
```

### List Repositories by Content Type

```bash
# RPM repositories only
hammer repository list --organization "ACME" --content-type yum

# DEB repositories only
hammer repository list --organization "ACME" --content-type deb
```

## GPG Key Management

```bash
# Create GPG key from URL (download first, then create)
curl -s https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x871920D1991BC93C > /tmp/ubuntu-key.gpg
hammer gpg create \
  --organization "ACME" \
  --name "Ubuntu Archive Key" \
  --key /tmp/ubuntu-key.gpg

# Create GPG key from file
hammer gpg create \
  --organization "ACME" \
  --name "Custom Repo Key" \
  --key /path/to/RPM-GPG-KEY-custom

# List GPG keys
hammer gpg list --organization "ACME"
```

## Download Policies

```bash
# Set download policy on repository creation
hammer repository create \
  --name "My Repo" \
  --product "My Product" \
  --organization "ACME" \
  --content-type yum \
  --url "http://repo.example.com/rhel8/" \
  --download-policy "on_demand"

# Update existing repository policy
hammer repository update \
  --organization "ACME" \
  --product "My Product" \
  --name "My Repo" \
  --download-policy "on_demand"

# Policy options:
#   immediate  — download all content during sync
#   on_demand  — download only when requested by clients (saves disk)
#   background — download after sync completes
```

## Async Operations and Monitoring

### Sync with Async Flag

```bash
# Async sync (returns immediately, runs in background)
hammer repository synchronize \
  --organization "ACME" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --name "Red Hat Enterprise Linux 8 for x86_64 - BaseOS RPMs 8" \
  --async

# Sync entire product asynchronously
hammer product synchronize \
  --organization "ACME" \
  --name "Red Hat Enterprise Linux for x86_64" \
  --async
```

### Monitor Sync Progress

```bash
# Watch sync status continuously
watch 'hammer repository list --organization "ACME" --fields "Name,Product,Sync State"'

# Search by sync status
hammer repository list --organization "ACME" --search "last_sync_result = success"
hammer repository list --organization "ACME" --search "last_sync_result = error"
hammer repository list --organization "ACME" --search "last_sync_result = not_synced"

# Task search by specific action type
hammer task list --search 'label = Actions::Katello::Repository::Sync'
hammer task list --search 'label = Actions::Katello::ContentView::Publish'
hammer task list --search 'result = error'
```

### Cancel and Retry Tasks

```bash
# Cancel a running task
hammer task cancel --id <task-id>

# Retry a failed sync
hammer repository synchronize \
  --organization "ACME" \
  --product "PRODUCT_NAME" \
  --name "REPOSITORY_NAME" \
  --async
```

## Content Export/Import

```bash
# Export a content view version (for disconnected environments)
hammer content-export complete \
  --organization "ACME" \
  --content-view "RHEL8-CV" \
  --version 1.0

# List content exports
hammer content-export list --organization "ACME"
```

## Performance Tips

```bash
# Check disk space (Pulp storage)
df -h /var/lib/pulp

# Use on_demand download policy for less frequently accessed content
# (only downloads packages when clients request them)
hammer repository update \
  --organization "ACME" \
  --product "PRODUCT_NAME" \
  --name "REPO_NAME" \
  --download-policy "on_demand"

# Always use --async for large sync operations to avoid timeout
hammer product synchronize --organization "ACME" --name "Large Product" --async

# Bulk sync all products
for product in $(hammer --csv product list --organization "ACME" | tail -n +2 | cut -d',' -f1); do
  hammer product synchronize --organization "ACME" --id "$product" --async
done
```

## Errata Management

```bash
# List all errata
hammer erratum list

# Show erratum details
hammer erratum info --id 3484

# Search critical errata (applicable to registered hosts)
hammer erratum list --search "severity = Critical" --errata-restrict-applicable true

# Find hosts affected by a specific erratum
hammer host list --search "applicable_errata = RHSA-2019:1267"
```

## Repository Set Information

```bash
# List repository sets for a product
hammer repository-set list --organization "ACME" --product "Red Hat Enterprise Linux Server"
hammer repository-set list --organization "ACME" --product "Red Hat Enterprise Linux for x86_64"

# Show repository set details
hammer repository-set info --organization "ACME" --name "Red Hat Enterprise Linux 8 for x86_64 - BaseOS (RPMs)"
```

## Bulk Sync All Repositories by ID

```bash
# Sync all repositories one by one
for i in $(hammer repository list | cut -d "|" -f1 | grep [0-9]); do
  hammer repository synchronize --id $i
done
```

## Task Progress

```bash
# Monitor a specific task's progress
hammer task progress --id <task-id>
```

## Foreman Maintain

```bash
# Run a health check
foreman-maintain health check

# Run specific health check (e.g., disk performance)
foreman-maintain health check --label disk-performance

# Run all health checks
foreman-maintain health check --assumeyes
```

## Monitoring via API

```bash
# Check repository sync status via API
curl -u admin:password -H "Accept: application/json" \
  https://satellite.example.com/katello/api/v2/repositories | \
  jq '.results[] | {name: .name, product: .product.name, last_sync: .last_sync, sync_state: .sync_state}'
```

## Important Files and Directories

> **Warning:** No file under `/var/lib/pulp` shall be manually deleted, moved, or modified without explicit confirmation from Red Hat support or Pulp engineering. Unwanted removal might cause harm and will almost surely hide evidence of what potential error caused orphaned files.

| Path | Purpose |
|------|---------|
| `~/.hammer/cli.modules.d/foreman.yml` | Hammer user credentials |
| `/etc/hammer/cli.modules.d/foreman.yml` | Global Hammer configuration |
| `/var/lib/pulp/` | Pulp content storage (DO NOT modify) |
| `/var/log/foreman-installer/foreman.log` | Installer log (foreman) |
| `/var/log/foreman-installer/katello.log` | Installer log (katello) |
| `/etc/foreman-installer/scenarios.d/foreman-answers.yaml` | Installation parameters (answer file) |
| `/etc/rhsm/rhsm.conf` | Subscription manager configuration |
| `/etc/apt/sources.list.d/rhsm.sources` | Ubuntu RHSM repository sources |

## Quick Reference

| Task | Command |
|------|---------|
| List hosts | `hammer host list` |
| Host info | `hammer host info --name <fqdn>` |
| List content views | `hammer content-view list --organization <org>` |
| Publish content view | `hammer content-view publish --name <cv> --organization <org>` |
| Sync repository | `hammer repository synchronize --name <repo> --product <product> --organization <org>` |
| List errata | `hammer host errata list --host <fqdn>` |
| Apply errata | `hammer host errata apply --host <fqdn> --errata-ids <id>` |
| Run remote command | `hammer job-invocation create --job-template "Run Command - SSH Default" --inputs "command=<cmd>" --search-query "name = <host>"` |
| List subscriptions | `hammer subscription list --organization <org>` |
| Ping services | `hammer ping` |
| List tasks | `hammer task list --search "state = running"` |
