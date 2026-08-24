# doctl Cheatsheet

`doctl` is the official CLI for DigitalOcean — managing Droplets, Kubernetes, databases, networking, storage, and more from the terminal.

## Installation

```bash
# macOS
brew install doctl

# Linux (snap)
sudo snap install doctl

# Linux (manual)
curl -sL https://github.com/digitalocean/doctl/releases/download/v1.104.0/doctl-1.104.0-linux-amd64.tar.gz | tar -xz
sudo mv doctl /usr/local/bin/

# Windows
scoop install doctl
```

## Authentication

```bash
# Authenticate with API token
doctl auth init

# Authenticate with a specific token
doctl auth init --access-token <token>

# Switch between accounts/contexts
doctl auth list
doctl auth switch --context <name>

# Validate authentication
doctl account get
```

## Droplets (VMs)

### List and Inspect

```bash
# List all Droplets
doctl compute droplet list

# List with specific format
doctl compute droplet list --format ID,Name,PublicIPv4,Status,Region

# Get a specific Droplet
doctl compute droplet get <droplet-id>

# List Droplet actions/events
doctl compute droplet actions <droplet-id>
```

### Create Droplets

```bash
# Create a Droplet
doctl compute droplet create my-server \
  --region nyc1 \
  --size s-1vcpu-1gb \
  --image ubuntu-24-04-x64 \
  --ssh-keys <key-fingerprint> \
  --wait

# Create with user data
doctl compute droplet create my-server \
  --region nyc1 \
  --size s-2vcpu-4gb \
  --image ubuntu-24-04-x64 \
  --user-data-file ./cloud-init.yaml \
  --ssh-keys <key-fingerprint> \
  --tag-names web,production \
  --wait

# Create multiple Droplets
doctl compute droplet create web-1 web-2 web-3 \
  --region nyc1 \
  --size s-1vcpu-2gb \
  --image ubuntu-24-04-x64 \
  --ssh-keys <key-fingerprint> \
  --tag-names web \
  --wait
```

### Manage Droplets

```bash
# Delete a Droplet
doctl compute droplet delete <droplet-id> --force

# Delete by tag
doctl compute droplet delete-by-tag web --force

# Power off/on
doctl compute droplet-action power-off <droplet-id> --wait
doctl compute droplet-action power-on <droplet-id> --wait

# Reboot
doctl compute droplet-action reboot <droplet-id> --wait

# Resize
doctl compute droplet-action resize <droplet-id> --size s-2vcpu-4gb --wait

# Rename
doctl compute droplet-action rename <droplet-id> --droplet-name new-name

# Rebuild (reinstall OS)
doctl compute droplet-action rebuild <droplet-id> --image ubuntu-24-04-x64

# Create snapshot
doctl compute droplet-action snapshot <droplet-id> --snapshot-name my-snapshot --wait
```

### Available Resources

```bash
# List available regions
doctl compute region list

# List available sizes
doctl compute size list

# List available images (OS)
doctl compute image list --public

# List custom images/snapshots
doctl compute image list --private

# List SSH keys
doctl compute ssh-key list
```

## Kubernetes (DOKS)

### Clusters

```bash
# List clusters
doctl kubernetes cluster list

# Get cluster details
doctl kubernetes cluster get <cluster-id>

# Create cluster
doctl kubernetes cluster create my-cluster \
  --region nyc1 \
  --version 1.30.1-do.0 \
  --node-pool "name=default;size=s-2vcpu-4gb;count=3;auto-scale=true;min-nodes=2;max-nodes=10" \
  --wait

# Create with multiple node pools
doctl kubernetes cluster create my-cluster \
  --region nyc1 \
  --version 1.30.1-do.0 \
  --node-pool "name=general;size=s-2vcpu-4gb;count=3;label=role:general" \
  --node-pool "name=workers;size=s-4vcpu-8gb;count=2;label=role:worker;taint=dedicated=worker:NoSchedule" \
  --wait

# Delete cluster
doctl kubernetes cluster delete <cluster-id> --force

# List available Kubernetes versions
doctl kubernetes options versions

# List available node sizes
doctl kubernetes options sizes
```

### Kubeconfig

```bash
# Save kubeconfig (adds context)
doctl kubernetes cluster kubeconfig save <cluster-id>

# Remove kubeconfig
doctl kubernetes cluster kubeconfig remove <cluster-id>

# Show kubeconfig
doctl kubernetes cluster kubeconfig show <cluster-id>
```

### Node Pools

```bash
# List node pools
doctl kubernetes cluster node-pool list <cluster-id>

# Get node pool details
doctl kubernetes cluster node-pool get <cluster-id> <pool-id>

# Create node pool
doctl kubernetes cluster node-pool create <cluster-id> \
  --name gpu-pool \
  --size g-2vcpu-8gb \
  --count 1 \
  --label accelerator=gpu \
  --taint "nvidia.com/gpu=true:NoSchedule"

# Update node pool (resize)
doctl kubernetes cluster node-pool update <cluster-id> <pool-id> \
  --count 5 \
  --auto-scale \
  --min-nodes 2 \
  --max-nodes 10

# Delete node pool
doctl kubernetes cluster node-pool delete <cluster-id> <pool-id> --force

# Delete a specific node
doctl kubernetes cluster node-pool delete-node <cluster-id> <pool-id> <node-id> --force
```

### Cluster Upgrades

```bash
# List available upgrades
doctl kubernetes cluster get-upgrades <cluster-id>

# Upgrade cluster
doctl kubernetes cluster upgrade <cluster-id> --version 1.30.1-do.0
```

### Update Cluster Settings

```bash
# Update cluster name
doctl kubernetes cluster update <cluster-id> --name new-name

# Enable auto-upgrade
doctl kubernetes cluster update <cluster-id> --auto-upgrade=true

# Update tags
doctl kubernetes cluster update <cluster-id> --tag production
```

### Registry Integration

```bash
# Add container registry credentials to cluster
doctl kubernetes cluster registry add <cluster-name>

# Remove registry from cluster
doctl kubernetes cluster registry remove <cluster-name>
```

### Replace a Node

```bash
# Replace a specific node (delete and recreate)
doctl kubernetes cluster node-pool replace-node <cluster-id> <pool-id> <node-id>
```

### 1-Click Apps

```bash
# List available 1-Click apps
doctl kubernetes 1-click list

# Install a 1-Click app on cluster
doctl kubernetes 1-click install <cluster-id> --1-clicks <app-slug>
```

### Common DOKS Workflows

```bash
# Create cluster and configure kubectl in one go
doctl kubernetes cluster create my-cluster \
  --region nyc1 \
  --node-pool "name=workers;size=s-2vcpu-4gb;count=3" \
  --wait && \
doctl kubernetes cluster kubeconfig save my-cluster

# Scale a node pool
POOL_ID=$(doctl kubernetes cluster node-pool list my-cluster --format ID --no-header)
doctl kubernetes cluster node-pool update my-cluster $POOL_ID --count 5

# Quick cluster health check
doctl kubernetes cluster get my-cluster --format Status
doctl kubernetes cluster node-pool list my-cluster --format Name,Count,Status
```

## Networking

### Domains and DNS

```bash
# List domains
doctl compute domain list

# Create domain
doctl compute domain create example.com

# Delete domain
doctl compute domain delete example.com --force

# List DNS records
doctl compute domain records list example.com

# Create DNS record
doctl compute domain records create example.com \
  --record-type A \
  --record-name www \
  --record-data 1.2.3.4 \
  --record-ttl 300

# Create CNAME
doctl compute domain records create example.com \
  --record-type CNAME \
  --record-name blog \
  --record-data www.example.com. \
  --record-ttl 300

# Update DNS record
doctl compute domain records update example.com \
  --record-id <record-id> \
  --record-data 5.6.7.8

# Delete DNS record
doctl compute domain records delete example.com <record-id> --force
```

### Firewalls

```bash
# List firewalls
doctl compute firewall list

# Create firewall
doctl compute firewall create \
  --name web-firewall \
  --inbound-rules "protocol:tcp,ports:80,address:0.0.0.0/0 protocol:tcp,ports:443,address:0.0.0.0/0 protocol:tcp,ports:22,address:10.0.0.0/8" \
  --outbound-rules "protocol:tcp,ports:all,address:0.0.0.0/0 protocol:udp,ports:all,address:0.0.0.0/0" \
  --droplet-ids <id1>,<id2>

# Add Droplets to firewall
doctl compute firewall add-droplets <firewall-id> --droplet-ids <id>

# Remove Droplets from firewall
doctl compute firewall remove-droplets <firewall-id> --droplet-ids <id>

# Delete firewall
doctl compute firewall delete <firewall-id> --force
```

### Load Balancers

```bash
# List load balancers
doctl compute load-balancer list

# Create load balancer
doctl compute load-balancer create \
  --name web-lb \
  --region nyc1 \
  --forwarding-rules "entry_protocol:http,entry_port:80,target_protocol:http,target_port:8080" \
  --droplet-ids <id1>,<id2>

# Add Droplets
doctl compute load-balancer add-droplets <lb-id> --droplet-ids <id>

# Delete load balancer
doctl compute load-balancer delete <lb-id> --force
```

### Floating IPs (Reserved IPs)

```bash
# List reserved IPs
doctl compute reserved-ip list

# Create reserved IP
doctl compute reserved-ip create --region nyc1

# Assign to Droplet
doctl compute reserved-ip-action assign <ip> <droplet-id>

# Unassign
doctl compute reserved-ip-action unassign <ip>

# Delete reserved IP
doctl compute reserved-ip delete <ip> --force
```

### VPC

```bash
# List VPCs
doctl vpcs list

# Create VPC
doctl vpcs create --name my-vpc --region nyc1 --ip-range 10.10.0.0/16

# Get VPC details
doctl vpcs get <vpc-id>

# List members (resources in VPC)
doctl vpcs members list <vpc-id>

# Delete VPC
doctl vpcs delete <vpc-id> --force
```

## Databases

```bash
# List database clusters
doctl databases list

# Create database cluster
doctl databases create my-db \
  --engine pg \
  --version 16 \
  --region nyc1 \
  --size db-s-1vcpu-1gb \
  --num-nodes 1 \
  --wait

# Supported engines: pg, mysql, redis, mongodb, kafka

# Get connection info
doctl databases connection <db-id>

# List databases within a cluster
doctl databases db list <db-id>

# Create a database
doctl databases db create <db-id> myapp

# List users
doctl databases user list <db-id>

# Create user
doctl databases user create <db-id> myuser

# List connection pools (PostgreSQL)
doctl databases pool list <db-id>

# Create connection pool
doctl databases pool create <db-id> \
  --db myapp \
  --user myuser \
  --size 10 \
  --mode transaction \
  --name mypool

# Resize cluster
doctl databases resize <db-id> --size db-s-2vcpu-4gb --num-nodes 2

# Delete database cluster
doctl databases delete <db-id> --force
```

### Database Firewalls

```bash
# List trusted sources
doctl databases firewalls list <db-id>

# Allow a Droplet
doctl databases firewalls append <db-id> --rule droplet:<droplet-id>

# Allow a Kubernetes cluster
doctl databases firewalls append <db-id> --rule k8s:<cluster-id>

# Allow an IP
doctl databases firewalls append <db-id> --rule ip_addr:1.2.3.4

# Remove a rule
doctl databases firewalls remove <db-id> --uuid <rule-uuid>
```

## Spaces (Object Storage)

```bash
# List Spaces
doctl compute cdn list

# Spaces are managed via s3cmd or AWS CLI with DO endpoint:
# Endpoint: https://<region>.digitaloceanspaces.com
# Use AWS CLI with --endpoint-url flag

# Example with AWS CLI
aws s3 ls --endpoint-url https://nyc3.digitaloceanspaces.com
aws s3 mb s3://my-bucket --endpoint-url https://nyc3.digitaloceanspaces.com
aws s3 cp file.txt s3://my-bucket/ --endpoint-url https://nyc3.digitaloceanspaces.com
```

## Block Storage (Volumes)

```bash
# List volumes
doctl compute volume list

# Create volume
doctl compute volume create my-volume \
  --region nyc1 \
  --size 100GiB \
  --desc "Data volume"

# Attach to Droplet
doctl compute volume-action attach <volume-id> <droplet-id>

# Detach from Droplet
doctl compute volume-action detach <volume-id> <droplet-id>

# Resize volume
doctl compute volume-action resize <volume-id> --size 200GiB --region nyc1

# Create snapshot
doctl compute volume snapshot <volume-id> --snapshot-name vol-backup

# Delete volume
doctl compute volume delete <volume-id> --force
```

## Container Registry

```bash
# Create registry (one per account)
doctl registry create my-registry --region nyc3

# Get registry info
doctl registry get

# Login to registry (configures Docker)
doctl registry login

# Logout
doctl registry logout

# List repositories
doctl registry repository list-v2

# List tags for a repository
doctl registry repository list-tags <repo-name>

# Delete tag
doctl registry repository delete-tag <repo-name> <tag>

# Delete manifest (and all associated tags)
doctl registry repository delete-manifest <repo-name> <digest>

# Run garbage collection
doctl registry garbage-collection start --include-untagged-manifests

# Check GC status
doctl registry garbage-collection get-active

# Integrate with Kubernetes (add registry credentials)
doctl registry kubernetes-manifest | kubectl apply -f -
```

## Apps Platform

```bash
# List apps
doctl apps list

# Create app from spec
doctl apps create --spec app-spec.yaml

# Get app details
doctl apps get <app-id>

# Update app
doctl apps update <app-id> --spec app-spec.yaml

# List deployments
doctl apps list-deployments <app-id>

# Create deployment (trigger rebuild)
doctl apps create-deployment <app-id>

# Get logs
doctl apps logs <app-id> --type run

# Delete app
doctl apps delete <app-id> --force
```

## Projects

```bash
# List projects
doctl projects list

# Create project
doctl projects create --name "My Project" --purpose "Production services"

# Get project resources
doctl projects resources list <project-id>

# Move resources to project
doctl projects resources assign <project-id> --resource "do:droplet:<id>" --resource "do:database:<id>"
```

## Monitoring and Alerts

```bash
# List alert policies
doctl monitoring alert list

# Create alert
doctl monitoring alert create \
  --type v1/insights/droplet/cpu \
  --compare GreaterThan \
  --value 80 \
  --window 5m \
  --entities <droplet-id> \
  --emails admin@example.com
```

## SSH

```bash
# SSH into a Droplet by name
doctl compute ssh <droplet-name>

# SSH by ID
doctl compute ssh <droplet-id>

# SSH with specific user
doctl compute ssh <droplet-name> --ssh-user ubuntu

# SSH with specific key
doctl compute ssh <droplet-name> --ssh-key-path ~/.ssh/id_ed25519

# Manage SSH keys
doctl compute ssh-key list
doctl compute ssh-key create my-key --public-key "$(cat ~/.ssh/id_ed25519.pub)"
doctl compute ssh-key delete <key-id> --force
```

## Output Formatting

```bash
# JSON output
doctl compute droplet list --output json

# Specific columns
doctl compute droplet list --format ID,Name,PublicIPv4,Status

# No headers
doctl compute droplet list --no-header

# Combine for scripting
doctl compute droplet list --format ID --no-header
```

## Quick Reference

```bash
# Auth
doctl auth init
doctl account get

# Droplets
doctl compute droplet list
doctl compute droplet create <name> --region <r> --size <s> --image <i> --wait
doctl compute droplet delete <id> --force
doctl compute ssh <name>

# Kubernetes
doctl kubernetes cluster list
doctl kubernetes cluster create <name> --region <r> --version <v> --node-pool "name=x;size=y;count=z"
doctl kubernetes cluster kubeconfig save <id>
doctl kubernetes cluster delete <id> --force

# Databases
doctl databases list
doctl databases create <name> --engine pg --region <r> --size <s> --wait
doctl databases connection <id>
doctl databases delete <id> --force

# Networking
doctl compute domain list
doctl compute domain records list <domain>
doctl compute firewall list
doctl compute load-balancer list
doctl vpcs list

# Volumes
doctl compute volume list
doctl compute volume create <name> --region <r> --size <n>GiB
doctl compute volume-action attach <vol-id> <droplet-id>

# Registry
doctl registry login
doctl registry repository list-v2
```
