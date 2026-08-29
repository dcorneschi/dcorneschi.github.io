# hcloud CLI Cheatsheet

`hcloud` is the official command-line interface for [Hetzner Cloud](https://www.hetzner.com/cloud). It manages servers, volumes, networks, firewalls, load balancers, floating/primary IPs, certificates, SSH keys, and more via the Hetzner Cloud API. Contexts (like `kubectl` contexts) let you switch between projects, each backed by its own API token.

## Installation

```bash
# macOS / Linux (Homebrew)
brew install hcloud

# Linux — download the latest release binary
curl -fsSL https://github.com/hetznercloud/cli/releases/latest/download/hcloud-linux-amd64.tar.gz \
  | sudo tar -xz -C /usr/local/bin hcloud

# Go install
go install github.com/hetznercloud/cli/cmd/hcloud@latest

# Pre-built binaries are also published at:
# https://github.com/hetznercloud/cli/releases

# Verify
hcloud version
```

Enable shell completion (bash example; `zsh`, `fish`, `powershell` also supported):

```bash
hcloud completion bash | sudo tee /etc/bash_completion.d/hcloud >/dev/null
```

## Contexts (Projects / API Tokens)

A context stores one project's API token. Create a token in the Hetzner Cloud Console under **Security → API Tokens**.

```bash
# Create and activate a context (prompts for the API token, input hidden)
hcloud context create my-project

# List contexts (active one marked with *)
hcloud context list

# Switch the active context
hcloud context use my-project

# Show the active context
hcloud context active

# Delete a context
hcloud context delete my-project
```

Each context represents a Hetzner Cloud project; create as many as you need and switch between them. You can also authenticate non-interactively with environment variables, which override the active context — handy in CI:

```bash
export HCLOUD_TOKEN="your-api-token"   # use a token directly
export HCLOUD_CONTEXT="my-project"     # or select a stored context
```

## Servers

### List and describe

```bash
# List all servers in the context
hcloud server list

# Custom columns / no header
hcloud server list -o columns=id,name,status,ipv4,type
hcloud server list -o noheader

# Filter by label selector
hcloud server list --selector env=production

# Describe a server (by name or ID)
hcloud server describe my-server
hcloud server describe 12345

# Print the public IPv4
hcloud server ip my-server
```

### Create

```bash
# Basic
hcloud server create --name web-server --type cx22 --image ubuntu-24.04

# With one or more SSH keys
hcloud server create --name web-server --type cx22 --image ubuntu-24.04 \
  --ssh-key key1 --ssh-key key2

# In a specific location
hcloud server create --name web-server --type cx22 --image ubuntu-24.04 \
  --location nbg1

# With labels
hcloud server create --name web-server --type cx22 --image ubuntu-24.04 \
  --label env=production --label app=web

# With cloud-init user data from a file
hcloud server create --name web-server --type cx22 --image ubuntu-24.04 \
  --user-data-from-file cloud-init.yaml

# With a volume attached
hcloud server create --name web-server --type cx22 --image ubuntu-24.04 \
  --volume my-volume

# Attach to a private network at creation
hcloud server create --name web-server --type cx22 --image ubuntu-24.04 \
  --network my-network

# Create without starting (for preparation)
hcloud server create --name web-server --type cx22 --image ubuntu-24.04 \
  --start-after-create=false
```

> If you don't pass `--ssh-key`, Hetzner prints/emails a generated root password. Always prefer SSH keys for production — create them first with `hcloud ssh-key create`.

### Power and lifecycle

```bash
hcloud server poweron my-server
hcloud server poweroff my-server      # hard power off (like pulling the plug)
hcloud server shutdown my-server      # graceful ACPI shutdown
hcloud server reboot my-server
hcloud server reset my-server         # hard reset
```

### Access and recovery

```bash
# SSH into a server (uses its public IP)
hcloud server ssh my-server
hcloud server ssh my-server --user ubuntu

# Reset the root password (printed once)
hcloud server reset-password my-server
```

### Resize, rebuild, and images

```bash
# Change server type (power off first; --keep-disk for cross-type/downgrade)
hcloud server change-type my-server --server-type cx32 --keep-disk

# Rebuild from an image (destroys data on the server)
hcloud server rebuild my-server --image ubuntu-24.04

# Create a snapshot image from a server
hcloud server create-image my-server --type snapshot --description "My backup"

# Automatic backups
hcloud server enable-backup my-server
hcloud server disable-backup my-server
```

### Management, labels, and protection

```bash
# Rename
hcloud server update my-server --name new-name

# Labels
hcloud server add-label my-server env=staging
hcloud server add-label my-server team=backend
hcloud server remove-label my-server env

# Protection (guards against accidental delete/rebuild)
hcloud server enable-protection my-server delete
hcloud server enable-protection my-server rebuild
hcloud server disable-protection my-server delete

# Delete (one or many)
hcloud server delete my-server
hcloud server delete server1 server2 server3
```

## Server Types

```bash
hcloud server-type list
hcloud server-type describe cx22
```

Common types:

| Type | vCPU | RAM | Disk | Class |
|------|------|-----|------|-------|
| `cx22` | 2 | 4 GB | 40 GB | shared (Intel/AMD) |
| `cx32` | 4 | 8 GB | 80 GB | shared |
| `cpx21` | 3 | 4 GB | 80 GB | dedicated (AMD) |
| `cpx31` | 4 | 8 GB | 160 GB | dedicated |

## Images

```bash
# List all, or filter by type
hcloud image list
hcloud image list --type system
hcloud image list --type snapshot
hcloud image list --type backup

# Describe / update / delete (snapshots and backups only)
hcloud image describe ubuntu-24.04
hcloud image update 12345 --description "Production snapshot"
hcloud image delete my-snapshot
```

Popular images: `ubuntu-24.04`, `ubuntu-22.04`, `debian-12`, `debian-11`, `fedora-39`, `rocky-9`, `rocky-8`, `centos-stream-9`.

## SSH Keys

First, generate a key pair locally if you don't already have one (`-N ''` sets an empty passphrase):

```sh
# Ed25519 (recommended)
ssh-keygen -f ~/.ssh/my-key -t ed25519 -N ''

# RSA 4096 (if you need broader compatibility)
ssh-keygen -f ~/.ssh/my-key -t rsa -b 4096 -N ''
```

Then upload and manage it:

```bash
hcloud ssh-key list

# Create from a file or an inline string
hcloud ssh-key create --name my-key --public-key-from-file ~/.ssh/my-key.pub
hcloud ssh-key create --name my-key --public-key "ssh-ed25519 AAAAC3..."

hcloud ssh-key describe my-key
hcloud ssh-key update my-key --name new-name
hcloud ssh-key add-label my-key env=production
hcloud ssh-key delete my-key
```

## Volumes

```bash
hcloud volume list

# Create unattached (needs a location) or attached to a server
hcloud volume create --name my-volume --size 100 --location nbg1
hcloud volume create --name my-volume --size 100 --server my-server

hcloud volume attach my-volume --server my-server
hcloud volume detach my-volume

# Resize (grow only — extend the filesystem in-guest afterward)
hcloud volume resize my-volume --size 200

# Protection and delete (detach first)
hcloud volume enable-protection my-volume delete
hcloud volume disable-protection my-volume delete
hcloud volume delete my-volume
```

> Volumes must be detached before deletion. Use `hcloud volume detach` first.

## Networks (Private Networks)

```bash
hcloud network list

# Create a network with a private IP range
hcloud network create --name my-network --ip-range 10.0.0.0/16

# Add a subnet (cloud type, in a network zone)
hcloud network add-subnet my-network \
  --type cloud --network-zone eu-central --ip-range 10.0.1.0/24

# Add a route
hcloud network add-route my-network --destination 10.100.0.0/16 --gateway 10.0.1.1

# Attach / detach a server (optionally pin its private IP)
hcloud server attach-to-network my-server --network my-network --ip 10.0.1.5
hcloud server detach-from-network my-server --network my-network

hcloud network describe my-network
hcloud network delete my-network
```

## Floating IPs

```bash
hcloud floating-ip list

# Create (optionally assign to a server directly)
hcloud floating-ip create --type ipv4 --home-location nbg1 --description "Web server IP"
hcloud floating-ip create --type ipv4 --server my-server

# Assign / unassign
hcloud floating-ip assign 12345 my-server
hcloud floating-ip unassign 12345

# Reverse DNS
hcloud floating-ip set-rdns 12345 --ip 1.2.3.4 --hostname host.example.com

# Protection and delete
hcloud floating-ip enable-protection 12345 delete
hcloud floating-ip disable-protection 12345 delete
hcloud floating-ip delete 12345
```

> **Primary IPs** are similar but bound to a server at creation and kept when the server is deleted:
> ```bash
> hcloud primary-ip create --type ipv4 --datacenter nbg1-dc3 --name web-pip
> hcloud primary-ip list
> ```

## Load Balancers

```bash
hcloud load-balancer list

# Create
hcloud load-balancer create --name my-lb --type lb11 --location nbg1

# Add a service (listen 80 -> destination 80)
hcloud load-balancer add-service my-lb \
  --protocol http --listen-port 80 --destination-port 80

# Targets — by server or by label selector
hcloud load-balancer add-target my-lb --server my-server
hcloud load-balancer add-target my-lb --label-selector env=prod
hcloud load-balancer remove-target my-lb --server my-server

# Attach to a private network
hcloud load-balancer attach-to-network my-lb --network my-network

hcloud load-balancer update my-lb --name new-name
hcloud load-balancer describe my-lb
hcloud load-balancer delete my-lb
```

## Firewalls

```bash
hcloud firewall list

# Create
hcloud firewall create --name my-firewall

# Add an inbound rule (SSH from anywhere, IPv4 + IPv6)
hcloud firewall add-rule my-firewall \
  --direction in --protocol tcp --port 22 \
  --source-ips 0.0.0.0/0 --source-ips ::/0

# Multiple ports in one rule
hcloud firewall add-rule my-firewall \
  --direction in --protocol tcp --port 80 --port 443 --source-ips 0.0.0.0/0

# Apply to / remove from a server (or use --label-selector for dynamic assignment)
hcloud firewall apply-to-resource my-firewall --type server --server my-server
hcloud firewall remove-from-resource my-firewall --type server --server my-server

hcloud firewall describe my-firewall
hcloud firewall delete my-firewall
```

Common rule fragments:

```bash
# SSH
--direction in --protocol tcp --port 22 --source-ips 0.0.0.0/0

# HTTP / HTTPS
--direction in --protocol tcp --port 80 --source-ips 0.0.0.0/0
--direction in --protocol tcp --port 443 --source-ips 0.0.0.0/0

# Allow all outbound
--direction out --protocol tcp --port any --destination-ips 0.0.0.0/0
```

## Certificates (for Load Balancers)

```bash
hcloud certificate list

# Managed (Hetzner obtains and renews via Let's Encrypt)
hcloud certificate create-managed \
  --name my-cert --domain example.com --domain www.example.com

# Upload your own
hcloud certificate create \
  --name my-cert \
  --certificate-from-file cert.pem \
  --private-key-from-file key.pem

hcloud certificate delete my-cert
```

## ISO Images

```bash
hcloud iso list
hcloud server attach-iso my-server --iso 12345
hcloud server detach-iso my-server
```

## Rescue Mode

```bash
# Enable (optionally pick a type and inject an SSH key), then reboot into it
hcloud server enable-rescue my-server
hcloud server enable-rescue my-server --type linux64
hcloud server enable-rescue my-server --ssh-key my-key
hcloud server reboot my-server

# Disable
hcloud server disable-rescue my-server
```

## Locations and Datacenters

```bash
hcloud location list
hcloud datacenter list
hcloud location describe nbg1
hcloud datacenter describe nbg1-dc3
```

Available locations:

- `nbg1` — Nuremberg, Germany
- `fsn1` — Falkenstein, Germany
- `hel1` — Helsinki, Finland
- `ash` — Ashburn, USA
- `hil` — Hillsboro, USA

## Labels and Selectors

Labels organize and filter resources, and drive dynamic assignment for firewalls and load balancers.

```bash
# During creation
hcloud server create --name web-1 --type cx22 --image ubuntu-24.04 \
  --label env=prod --label app=web

# On existing resources
hcloud server add-label my-server env=production
hcloud server remove-label my-server env

# Filter with selectors
hcloud server list --selector env=production
hcloud server list --selector env=production,app=web
hcloud server list --selector 'env!=staging'
hcloud server list --selector 'env in (prod,staging)'
```

## Output Formatting

```bash
# JSON / YAML for parsing
hcloud server list -o json
hcloud server describe my-server -o yaml
hcloud server list -o json | jq -r '.[].name'

# Custom columns, no header, combined
hcloud server list -o columns=id,name,status,ipv4
hcloud server list -o noheader
hcloud server list -o noheader -o columns=name,ipv4
```

## Tips and Tricks

### Bulk operations with shell loops

```bash
# Delete all servers with a label (careful!)
for server in $(hcloud server list -o noheader -o columns=name --selector env=test); do
  hcloud server delete "$server"
done

# Power off all servers
for server in $(hcloud server list -o noheader -o columns=name); do
  hcloud server poweroff "$server"
done
```

### Get a server IP quickly

```bash
hcloud server ip my-server                                     # simplest
hcloud server describe my-server -o json | jq -r '.public_net.ipv4.ip'
```

### Waiting for operations

The CLI blocks until an operation finishes, so a `create` returns only once the server is running — no extra polling needed.

### Scripting best practices

```bash
# Only create if it doesn't already exist
if ! hcloud server describe my-server &>/dev/null; then
  hcloud server create --name my-server --type cx22 --image ubuntu-24.04
fi

# Parse JSON for reliable values
SERVER_IP=$(hcloud server describe my-server -o json | jq -r '.public_net.ipv4.ip')
```

### Quick provisioning with cloud-init

```bash
cat > cloud-init.yaml <<'EOF'
#cloud-config
packages:
  - docker.io
  - git
runcmd:
  - systemctl enable --now docker
EOF

hcloud server create \
  --name docker-host --type cx22 --image ubuntu-24.04 \
  --ssh-key my-key --user-data-from-file cloud-init.yaml
```

### Cost control

```bash
# See types at a glance
hcloud server list -o columns=name,type,status

# Power off vs delete: a powered-off server still bills for its disk;
# delete it to stop all charges.
hcloud server poweroff dev-server
hcloud server delete dev-server
```

### Backup strategy

```bash
# Snapshot before major changes
hcloud server create-image my-server \
  --type snapshot --description "Before upgrade $(date +%F)"

# Or enable automatic backups
hcloud server enable-backup my-server
```

### Monitoring status

```bash
if [ "$(hcloud server describe my-server -o json | jq -r '.status')" = "running" ]; then
  echo "Server is running"
fi

hcloud server describe my-server -o json | jq '.included_traffic'
```

## Common Workflows

### Deploy a web application

```bash
# 1. SSH key
hcloud ssh-key create --name deploy-key --public-key-from-file ~/.ssh/id_ed25519.pub

# 2. Firewall
hcloud firewall create --name web-firewall
hcloud firewall add-rule web-firewall --direction in --protocol tcp --port 22 --source-ips 0.0.0.0/0
hcloud firewall add-rule web-firewall --direction in --protocol tcp --port 80 --source-ips 0.0.0.0/0
hcloud firewall add-rule web-firewall --direction in --protocol tcp --port 443 --source-ips 0.0.0.0/0

# 3. Server
hcloud server create \
  --name web-server --type cx22 --image ubuntu-24.04 \
  --ssh-key deploy-key --label env=production --label app=web

# 4. Apply the firewall and connect
hcloud firewall apply-to-resource web-firewall --type server --server web-server
hcloud server ssh web-server
```

### Create a Kubernetes node

```bash
hcloud server create \
  --name k8s-node-1 --type cpx31 --image ubuntu-24.04 \
  --ssh-key my-key --label cluster=production --label role=worker

NODE_IP=$(hcloud server describe k8s-node-1 -o json | jq -r '.public_net.ipv4.ip')
hcloud server ssh k8s-node-1
```

### Database server with a volume

```bash
hcloud volume create --name db-volume --size 100 --location nbg1

hcloud server create \
  --name db-server --type cx32 --image ubuntu-24.04 \
  --ssh-key my-key --location nbg1

hcloud volume attach db-volume --server db-server

# In-guest: format and mount the volume (device path is stable by ID)
#   mkfs.ext4 /dev/disk/by-id/scsi-0HC_Volume_*
#   mount /dev/disk/by-id/scsi-0HC_Volume_* /mnt/data
```

### Network isolation for backend services

```bash
hcloud network create --name backend-net --ip-range 10.0.0.0/16
hcloud network add-subnet backend-net --type cloud --network-zone eu-central --ip-range 10.0.1.0/24

hcloud server attach-to-network db-server  --network backend-net --ip 10.0.1.10
hcloud server attach-to-network app-server --network backend-net --ip 10.0.1.20
```

## Troubleshooting

### Server won't start

```bash
hcloud server describe my-server
hcloud server describe my-server -o json | jq '.protection'

hcloud server poweroff my-server
hcloud server poweron my-server
```

### Can't delete a resource

```bash
# Delete protection is usually the cause
hcloud server describe my-server -o json | jq '.protection'
hcloud server disable-protection my-server delete
hcloud server delete my-server
```

### Volume won't attach

```bash
# Volume and server must be in the same location, and the volume detached
hcloud volume describe my-volume
hcloud server describe my-server
hcloud volume detach my-volume
hcloud volume attach my-volume --server my-server
```

## Using the Token with Terraform

The same Hetzner Cloud API token works with the [`hetznercloud/hcloud`](https://registry.terraform.io/providers/hetznercloud/hcloud/latest) Terraform provider. Declare a variable for the token and pass it at plan/apply time:

```hcl
variable "hcloud_token" {
  type      = string
  sensitive = true
}

provider "hcloud" {
  token = var.hcloud_token
}
```

```sh
terraform plan  -var='hcloud_token=<YOUR-API-TOKEN-HERE>'
terraform apply -var='hcloud_token=<YOUR-API-TOKEN-HERE>'
```

> Prefer not to put the token on the command line (it lands in shell history). Use the `TF_VAR_hcloud_token` environment variable instead, or the `HCLOUD_TOKEN` variable which the provider also reads:
> ```sh
> export HCLOUD_TOKEN="your-api-token"
> terraform plan
> ```

## Getting Help

```bash
hcloud version
hcloud --help
hcloud server --help
hcloud server create --help
```

Almost every command accepts `--help`; use it to discover the exact flags for a subcommand and its version.

## Additional Resources

- Cloud documentation: [docs.hetzner.com/cloud](https://docs.hetzner.com/cloud/)
- API reference: [docs.hetzner.cloud](https://docs.hetzner.cloud/)
- CLI source: [github.com/hetznercloud/cli](https://github.com/hetznercloud/cli)
- Community tutorials: [community.hetzner.com/tutorials](https://community.hetzner.com/tutorials)
