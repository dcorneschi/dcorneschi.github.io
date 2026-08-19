# Packer Cheatsheet

HashiCorp Packer automates the creation of machine images for multiple platforms from a single template. It integrates with cloud providers, hypervisors, and container runtimes — producing identical images for dev, staging, and production.

## Install

```bash
# RHEL / CentOS / Rocky / Alma
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo dnf install -y packer

# Ubuntu / Debian
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y packer

# macOS (Homebrew)
brew tap hashicorp/tap
brew install hashicorp/tap/packer

# Binary install (any platform)
PACKER_VERSION="1.11.2"
curl -fsSL "https://releases.hashicorp.com/packer/${PACKER_VERSION}/packer_${PACKER_VERSION}_linux_amd64.zip" -o packer.zip
unzip packer.zip && sudo mv packer /usr/local/bin/
packer version
```

## Core Commands

| Command | Description |
|---------|-------------|
| `packer init .` | Install required plugins from config |
| `packer init -upgrade .` | Upgrade plugins to latest matching version |
| `packer fmt .` | Format HCL2 templates |
| `packer fmt -recursive .` | Format HCL2 templates recursively |
| `packer fmt -check -diff .` | Check formatting without writing (CI gate) |
| `packer validate .` | Validate template syntax & config |
| `packer validate -syntax-only .` | Validate syntax only (no config checks) |
| `packer build .` | Build image from template |
| `packer inspect template.pkr.hcl` | Show variables, builders, provisioners |
| `packer console template.pkr.hcl` | Interactive HCL expression evaluation |
| `packer version` | Show Packer version |
| `packer plugins installed` | List installed plugins |
| `packer plugins install github.com/hashicorp/amazon` | Install a specific plugin |
| `packer plugins remove github.com/hashicorp/docker` | Remove a plugin |

## Build Options

```bash
# Pass variables
packer build -var 'ami_name=my-ami' .
packer build -var-file=prod.pkrvars.hcl .

# Only run specific builds (when multiple sources)
packer build -only='amazon-ebs.ubuntu' .

# Except specific builds
packer build -except='docker.test' .

# Parallel builds (default: unlimited)
packer build -parallel-builds=2 .

# Force build (ignore existing artifacts)
packer build -force .

# Enable debug mode (step-by-step, pauses between steps)
packer build -debug .

# Machine-readable output
packer build -machine-readable . 2>&1 | tee build.log

# Set color off (CI)
packer build -color=false .

# Timestamp output
packer build -timestamp-ui .
```

## Packer Console

Interactive evaluation of HCL expressions — useful for testing functions and variable references:

```bash
# Open interactive console
packer console template.pkr.hcl

# Examples inside the console:
# > formatdate("YYYYMMDD-hhmm", timestamp())
# > upper("hello")
# > regex_replace(timestamp(), "[- TZ:]", "")
# > legacy_isotime("2006-01-02-1504")
```

## One-Liners

```bash
# Validate all templates in current dir
find . -name '*.pkr.hcl' -exec packer validate {} \;

# Build with timestamp + log to file
PACKER_LOG=1 PACKER_LOG_PATH=build.log packer build -timestamp-ui .

# Quick format check (CI gate)
packer fmt -check -diff .

# Get latest Ubuntu AMI ID for use in templates
aws ec2 describe-images --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-*-amd64-server-*" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' --output text

# Build and extract AMI ID from output
packer build -machine-readable . | tee /dev/stderr | \
  grep 'artifact,0,id' | cut -d, -f6 | cut -d: -f2

# Clean up orphaned packer security groups (AWS)
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=packer_*" \
  --query 'SecurityGroups[].GroupId' --output text

# Kill a stuck build's temporary instance
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=Packer Builder" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].InstanceId' --output text
```

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `PACKER_LOG=1` | Enable detailed logging |
| `PACKER_LOG_PATH=file.log` | Log to file instead of stderr |
| `PACKER_CACHE_DIR=~/.packer.d/cache` | ISO/image cache directory |
| `PACKER_CONFIG_DIR=~/.config/packer` | Plugin & config directory |
| `PACKER_PLUGIN_PATH=/custom/path` | Override plugin lookup path |
| `PACKER_NO_COLOR=1` | Disable color output |
| `PKR_VAR_<name>=value` | Set variable via env (e.g., `PKR_VAR_region=us-east-1`) |
| `CHECKPOINT_DISABLE=1` | Disable update check |
| `AWS_MAX_ATTEMPTS=60` | Increase AWS API retry attempts |
| `AWS_POLL_DELAY_SECONDS=15` | AWS polling delay between retries |

## HCL2 Template Structure

```hcl
packer {
  required_plugins {
    amazon = {
      version = ">= 1.2.0"
      source  = "github.com/hashicorp/amazon"
    }
  }
}

variable "region" {
  type    = string
  default = "us-east-1"
}

locals {
  timestamp = formatdate("YYYYMMDD-hhmm", timestamp())
  ami_name  = "app-${local.timestamp}"
}

source "amazon-ebs" "base" {
  region        = var.region
  instance_type = "t3.micro"
  ami_name      = local.ami_name

  source_ami_filter {
    filters = {
      name                = "ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"
      root-device-type    = "ebs"
      virtualization-type = "hvm"
    }
    owners      = ["099720109477"]
    most_recent = true
  }

  ssh_username = "ubuntu"

  tags = {
    Name    = local.ami_name
    Builder = "packer"
  }
}

build {
  sources = ["source.amazon-ebs.base"]

  provisioner "shell" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
    ]
  }

  provisioner "file" {
    source      = "configs/"
    destination = "/tmp/"
  }

  provisioner "shell" {
    script = "scripts/setup.sh"
  }

  post-processor "manifest" {
    output = "manifest.json"
  }
}
```

## Tips & Tricks

### Debugging

- **SSH into a failed build**: Use `-debug` flag — Packer pauses after each step and dumps the SSH key to the current directory. SSH in manually to inspect state.
- **Keep instance on failure**: Add `ssh_keep_alive_interval` and set `skip_clean_instance = true` (plugin-specific) or use `-on-error=abort` to leave the instance running.

```bash
packer build -on-error=abort .    # Leave instance running on failure
packer build -on-error=ask .      # Prompt what to do on failure
packer build -on-error=cleanup .  # Default: terminate on failure
```

### Performance

- **Use `async_resourcedisk_format = true`** (Azure) for faster provisioning.
- **Snapshot instead of full AMI copy**: For AWS, use `ami_regions` only for regions you actually need.
- **Parallelize shell provisioners** where order doesn't matter:

```hcl
provisioner "shell" {
  inline = [
    "sudo apt-get update && sudo apt-get install -y pkg1 pkg2 pkg3 &",
    "curl -sL https://example.com/install.sh | sudo bash &",
    "wait"
  ]
}
```

### Security

- **Never bake secrets into AMIs** — use IAM roles, SSM Parameter Store, or Vault at boot.
- **Encrypt AMIs**: Set `encrypt_boot = true` and specify `kms_key_id`.
- **Restrict AMI sharing**: Use `ami_users` or `ami_org_arns` instead of making public.
- **Use temporary credentials**: Packer respects `AWS_PROFILE`, IAM roles, and STS assume-role.

```hcl
source "amazon-ebs" "secure" {
  encrypt_boot = true
  kms_key_id   = "alias/packer-key"
  
  # Share only with specific accounts
  ami_users = ["111111111111", "222222222222"]
}
```

### CI/CD Integration

```bash
# GitLab CI / GitHub Actions pattern
packer init .
packer fmt -check -diff .
packer validate -var-file=ci.pkrvars.hcl .
packer build -var-file=ci.pkrvars.hcl -color=false -timestamp-ui .
```

### Useful Patterns

**Conditional provisioner (skip on certain OS):**
```hcl
provisioner "shell" {
  only   = ["amazon-ebs.ubuntu"]
  inline = ["sudo apt-get install -y htop"]
}

provisioner "shell" {
  only   = ["amazon-ebs.amazon-linux"]
  inline = ["sudo yum install -y htop"]
}
```

**Wait for cloud-init before provisioning:**
```hcl
provisioner "shell" {
  inline = ["cloud-init status --wait"]
}
```

**Tag AMI with git commit:**
```hcl
locals {
  git_sha = trimspace(file("/tmp/git_sha.txt"))  # or use data source
}

# Or pass it as a variable:
# packer build -var "git_sha=$(git rev-parse --short HEAD)" .
```

**Multi-stage build (base + app):**
```bash
# Stage 1: build base AMI
packer build -var 'ami_name=base-ami' base.pkr.hcl

# Stage 2: use base AMI as source for app AMI
packer build -var "source_ami=$(cat manifest.json | jq -r '.builds[-1].artifact_id' | cut -d: -f2)" app.pkr.hcl
```

**Ansible provisioner:**
```hcl
provisioner "ansible" {
  playbook_file = "ansible/playbook.yml"
  extra_arguments = [
    "--extra-vars", "env=production",
    "--scp-extra-args", "'-O'",  # Fix for newer OpenSSH
  ]
  ansible_env_vars = ["ANSIBLE_HOST_KEY_CHECKING=False"]
}
```

## Common Gotchas

| Issue | Fix |
|-------|-----|
| `timeout waiting for SSH` | Check security group allows inbound SSH; verify `ssh_username` matches AMI |
| `UnauthorizedOperation` | Ensure IAM policy has `ec2:CreateImage`, `ec2:RunInstances`, etc. |
| `VPCIdNotSpecified` | Specify `vpc_id` and `subnet_id` (no default VPC in account) |
| Build works locally, fails in CI | Set `associate_public_ip_address = true` or use a NAT gateway |
| Ansible provisioner SSH errors | Add `"--scp-extra-args", "'-O'"` to `extra_arguments` (OpenSSH 9+) |
| `ResourceInUse: AMI is in use` | Deregister old AMIs before rebuilding with same name, or use timestamps |
| Plugin not found after `packer init` | Run `packer init -upgrade .` to refresh plugin cache |

## Minimum IAM Policy (AWS)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:CopyImage",
        "ec2:CreateImage",
        "ec2:CreateKeypair",
        "ec2:CreateSecurityGroup",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteKeyPair",
        "ec2:DeleteSecurityGroup",
        "ec2:DeleteSnapshot",
        "ec2:DeleteVolume",
        "ec2:DeregisterImage",
        "ec2:DescribeImageAttribute",
        "ec2:DescribeImages",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:DescribeRegions",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSnapshots",
        "ec2:DescribeSubnets",
        "ec2:DescribeVolumes",
        "ec2:DetachVolume",
        "ec2:GetPasswordData",
        "ec2:ModifyImageAttribute",
        "ec2:ModifyInstanceAttribute",
        "ec2:ModifySnapshotAttribute",
        "ec2:RegisterImage",
        "ec2:RunInstances",
        "ec2:StopInstances",
        "ec2:TerminateInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

## Data Sources

Data sources allow dynamic lookups at build time (HCL2 only):

```hcl
# Look up the latest Ubuntu AMI dynamically
data "amazon-ami" "ubuntu" {
  filters = {
    name                = "ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"
    root-device-type    = "ebs"
    virtualization-type = "hvm"
  }
  owners      = ["099720109477"]
  most_recent = true
}

source "amazon-ebs" "app" {
  source_ami    = data.amazon-ami.ubuntu.id
  instance_type = "t3.micro"
  region        = "us-east-1"
  ami_name      = "app-${local.timestamp}"
  ssh_username  = "ubuntu"
}
```

```hcl
# Read a local file as a data source
data "external" "git_sha" {
  program = ["git", "rev-parse", "--short", "HEAD"]
}

# Use an HTTP data source (requires http plugin)
data "http" "metadata" {
  url = "http://169.254.169.254/latest/meta-data/instance-id"
}

# Azure image data source
data "azure-image" "ubuntu" {
  location  = var.location
  publisher = "Canonical"
  offer     = "0001-com-ubuntu-server-focal"
  sku       = "20_04-lts-gen2"
}
```

## Variable Validation

```hcl
variable "region" {
  type        = string
  description = "AWS region for the build"
  default     = "us-east-1"

  validation {
    condition     = can(regex("^(us|eu|ap|sa|ca|me|af)-(north|south|east|west|central|southeast|northeast)-[1-3]$", var.region))
    error_message = "Must be a valid AWS region identifier."
  }
}

variable "instance_type" {
  type    = string
  default = "t3.micro"

  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium", "m5.large"], var.instance_type)
    error_message = "Instance type must be one of: t3.micro, t3.small, t3.medium, m5.large."
  }
}

variable "ssh_username" {
  type      = string
  default   = "ubuntu"
  sensitive = false
}

variable "api_key" {
  type      = string
  sensitive = true  # Won't appear in logs or output
}
```

### Variable Types

| Type | Example |
|------|---------|
| `string` | `"us-east-1"` |
| `number` | `8` |
| `bool` | `true` |
| `list(string)` | `["us-east-1", "eu-west-1"]` |
| `map(string)` | `{ Name = "app", Env = "prod" }` |
| `object({...})` | `object({ name = string, size = number })` |

### Variable Files (`.pkrvars.hcl`)

```hcl
# prod.pkrvars.hcl
region        = "us-east-1"
instance_type = "m5.large"
ami_users     = ["111111111111", "222222222222"]
tags = {
  Environment = "production"
  Team        = "platform"
}
```

```bash
# Use variable file
packer build -var-file=prod.pkrvars.hcl .

# Multiple var files (later overrides earlier)
packer build -var-file=base.pkrvars.hcl -var-file=prod.pkrvars.hcl .
```

## Communicators

Communicators define how Packer connects to the instance for provisioning.

### SSH (Linux/macOS)

```hcl
source "amazon-ebs" "linux" {
  ssh_username         = "ubuntu"
  ssh_timeout          = "10m"
  ssh_agent_auth       = false              # Use ssh-agent (for bastion setups)
  ssh_private_key_file = "~/.ssh/id_ed25519" # Optional: use specific key
  ssh_port             = 22

  # Bastion / jump host
  ssh_bastion_host        = "bastion.example.com"
  ssh_bastion_username    = "ec2-user"
  ssh_bastion_private_key_file = "~/.ssh/bastion.pem"

  # Keep connection alive
  ssh_keep_alive_interval = "5s"

  # Disable strict host key checking (common for ephemeral builders)
  ssh_handshake_attempts  = 20
}
```

### WinRM (Windows)

```hcl
source "amazon-ebs" "windows" {
  communicator   = "winrm"
  winrm_username = "Administrator"
  winrm_password = "SuperSecret123!"
  winrm_timeout  = "15m"
  winrm_use_ssl  = true
  winrm_insecure = true
  winrm_port     = 5986

  # EC2 Windows — get password from key pair
  user_data_file = "scripts/winrm-setup.ps1"
}
```

### None (no provisioning, just create image)

```hcl
source "amazon-ebs" "snapshot" {
  communicator = "none"
  # No SSH/WinRM — just launch, snapshot, terminate
}
```

## Provisioners

### Shell

```hcl
# Inline commands
provisioner "shell" {
  inline = [
    "sudo apt-get update",
    "sudo apt-get install -y nginx curl jq",
  ]
  environment_vars = [
    "DEBIAN_FRONTEND=noninteractive",
  ]
}

# External script
provisioner "shell" {
  scripts = [
    "scripts/01-base.sh",
    "scripts/02-app.sh",
    "scripts/03-cleanup.sh",
  ]
  execute_command = "chmod +x {{ .Path }}; {{ .Vars }} sudo -E sh '{{ .Path }}'"
}

# Elevated (root) inline
provisioner "shell" {
  inline           = ["yum install -y httpd"]
  execute_command  = "sudo sh -c '{{ .Vars }} {{ .Path }}'"
}
```

### Understanding execute_command

`execute_command` controls how Packer runs provisioner scripts on the target machine. It uses Go template syntax:

```hcl
execute_command = "sudo -S env {{ .Vars }} {{ .Path }}"
```

**Template variables:**
- `{{ .Vars }}` — replaced with environment variables at runtime (e.g., `HOME=/root USER=root DEBIAN_FRONTEND=noninteractive`)
- `{{ .Path }}` — replaced with the path to the uploaded script (e.g., `/tmp/packer-shell123456.sh`)

**After substitution, the command becomes:**
```bash
sudo -S env HOME=/root USER=root DEBIAN_FRONTEND=noninteractive /tmp/packer-shell123456.sh
```

**Flags explained:**
- `sudo` — run as root
- `-S` — read password from stdin (allows automated/non-interactive execution)
- `-E` — preserve environment variables (alternative to `env`)
- `env` — set environment variables before the script runs

**Common patterns:**

```hcl
# Default (no elevation)
execute_command = "chmod +x {{ .Path }}; {{ .Vars }} {{ .Path }}"

# Run as root with sudo (most common)
execute_command = "sudo -S sh -c '{{ .Vars }} {{ .Path }}'"

# Root with environment preserved
execute_command = "chmod +x {{ .Path }}; {{ .Vars }} sudo -E sh '{{ .Path }}'"

# Root with explicit env command
execute_command = "sudo -S env {{ .Vars }} {{ .Path }}"

# With bash instead of sh
execute_command = "sudo -S bash -c '{{ .Vars }} {{ .Path }}'"

# Run as a specific user
execute_command = "sudo -u deploy {{ .Vars }} {{ .Path }}"

# echo password for sudo -S (when SSH user has a password)
execute_command = "echo 'packer' | sudo -S sh -c '{{ .Vars }} {{ .Path }}'"
```

> **When to customize:** Override `execute_command` when the default (`chmod +x {{.Path}}; {{.Vars}} {{.Path}}`) doesn't work — typically because the script needs root privileges and the SSH user isn't root.

### Are {{ .Vars }} and {{ .Path }} Mandatory?

`{{ .Path }}` is essential when using `script` or `scripts` — without it, Packer doesn't know which uploaded script to execute. `{{ .Vars }}` is only needed if you pass `environment_vars` to the provisioner.

```hcl
# Minimal — just the script (no env vars needed)
provisioner "shell" {
  execute_command = "sudo bash {{ .Path }}"
  script          = "scripts/01-install-common.sh"
}

# With environment variables flowing through
provisioner "shell" {
  execute_command  = "sudo bash -c '{{ .Vars }} {{ .Path }}'"
  script           = "scripts/01-install-common.sh"
  environment_vars = [
    "DEBIAN_FRONTEND=noninteractive",
    "APP_VERSION=2.1.0",
  ]
}

# No execute_command at all — Packer uses its default
provisioner "shell" {
  inline = ["sudo apt-get update", "sudo apt-get install -y curl"]
}
```

If you omit `execute_command` entirely, Packer uses its built-in default which includes both `{{ .Vars }}` and `{{ .Path }}`.

### File Provisioner

```hcl
# Upload a file
provisioner "file" {
  source      = "configs/nginx.conf"
  destination = "/tmp/nginx.conf"
}

# Upload a directory (trailing slash matters!)
provisioner "file" {
  source      = "configs/"       # Uploads CONTENTS of configs/ into /tmp/
  destination = "/tmp/"
}

provisioner "file" {
  source      = "configs"        # Uploads the configs DIRECTORY itself into /tmp/configs
  destination = "/tmp/"
}

# Download from instance (direction = "download")
provisioner "file" {
  source      = "/var/log/cloud-init-output.log"
  destination = "logs/cloud-init.log"
  direction   = "download"
}
```

> **Gotcha:** The file provisioner runs as the SSH user. To place files in root-owned directories, upload to `/tmp/` first, then `mv` with a shell provisioner.

### PowerShell (Windows)

```hcl
provisioner "powershell" {
  inline = [
    "Install-WindowsFeature -Name Web-Server -IncludeManagementTools",
    "Set-Service -Name wuauserv -StartupType Disabled",
  ]
}

provisioner "powershell" {
  scripts = ["scripts/setup.ps1"]
  elevated_user     = "Administrator"
  elevated_password = "{{ .WinRMPassword }}"
}
```

### Ansible

```hcl
provisioner "ansible" {
  playbook_file = "ansible/playbook.yml"
  user          = "ubuntu"
  extra_arguments = [
    "--extra-vars", "env=production app_version=2.1.0",
    "--scp-extra-args", "'-O'",  # Required for OpenSSH 9+
  ]
  ansible_env_vars = [
    "ANSIBLE_HOST_KEY_CHECKING=False",
    "ANSIBLE_SSH_ARGS='-o ForwardAgent=yes'",
  ]
}

# Ansible Local (runs on the instance, no SSH needed from controller)
provisioner "ansible-local" {
  playbook_file = "ansible/playbook.yml"
  playbook_dir  = "ansible/"
  extra_arguments = ["--extra-vars", "env=production"]
}
```

### Breakpoint (debugging)

```hcl
# Pause build — SSH in manually to inspect state
provisioner "breakpoint" {
  disable = false
  note    = "Pausing before app install — SSH in to inspect"
}
```

### Windows Update Provisioner

```hcl
provisioner "windows-update" {
  search_criteria = "IsInstalled=0"
  filters = [
    "exclude:$_.Title -like '*Preview*'",
    "include:$true",
  ]
  restart_timeout = "5m"
}
```

### Advanced Shell Options

```hcl
# Named provisioner with pause and retries
provisioner "shell" {
  name         = "docker-installation"
  pause_before = "10s"
  max_retries  = 3
  script       = "./scripts/install-docker.sh"
}

# Error handling pattern
provisioner "shell" {
  inline = [
    "set -e",
    "set -x",
    "./scripts/setup.sh || (echo 'Setup failed' && exit 1)",
  ]
}

# Remote script
provisioner "shell" {
  inline = ["curl -sSL https://get.docker.com | sh"]
}
```

### File Provisioner — Generated Content

```hcl
# Upload generated content (no source file needed)
provisioner "file" {
  content     = templatefile("./templates/config.tpl", { env = var.environment })
  destination = "/tmp/config.json"
}

# Upload and fix permissions pattern
provisioner "file" {
  source      = "./scripts/"
  destination = "/tmp/scripts/"
}

provisioner "shell" {
  inline = [
    "sudo chmod +x /tmp/scripts/*.sh",
    "sudo mv /tmp/scripts/* /usr/local/bin/",
  ]
}
```

### Ansible Remote (ansible-pull)

```hcl
provisioner "ansible-remote" {
  playbook_url  = "https://github.com/user/ansible-playbooks.git"
  playbook_file = "site.yml"
  inventory_file = "inventory/production"
}
```

## Post-Processors

### Manifest (output artifact metadata)

```hcl
post-processor "manifest" {
  output     = "manifest.json"
  strip_path = true
  custom_data = {
    version   = var.app_version
    timestamp = local.timestamp
  }
}
```

### Shell Local (run command after build)

```hcl
post-processor "shell-local" {
  inline = [
    "echo 'Build complete! AMI: {{.ArtifactId}}'",
    "jq '.builds[-1].artifact_id' manifest.json",
    "aws s3 cp manifest.json s3://my-bucket/artifacts/",
  ]
}
```

### Compress (create archive)

```hcl
post-processor "compress" {
  output            = "output/{{.BuildName}}.tar.gz"
  format            = "tar.gz"
  compression_level = 6
}
```

### Docker (tag and push)

```hcl
# Tag and push after docker build
post-processor "docker-tag" {
  repository = "registry.example.com/myapp"
  tags       = ["latest", var.app_version]
}

post-processor "docker-push" {
  login          = true
  login_server   = "registry.example.com"
  login_username = var.registry_user
  login_password = var.registry_pass
}
```

### Vagrant (create .box file)

```hcl
post-processor "vagrant" {
  output = "builds/{{.Provider}}-{{.BuildName}}.box"
}
```

### Chaining Post-Processors

```hcl
build {
  sources = ["source.amazon-ebs.base"]

  # Sequential chain — output of one feeds the next
  post-processors {
    post-processor "manifest" {
      output = "manifest.json"
    }
    post-processor "shell-local" {
      inline = ["./scripts/notify-slack.sh"]
    }
  }
}
```

## Proxmox Builder

Build VM templates directly in Proxmox:

```hcl
packer {
  required_plugins {
    proxmox = {
      version = ">= 1.1.0"
      source  = "github.com/hashicorp/proxmox"
    }
  }
}

variable "proxmox_url" {
  type    = string
  default = "https://proxmox.example.com:8006/api2/json"
}

variable "proxmox_token" {
  type      = string
  sensitive = true
}

source "proxmox-iso" "ubuntu" {
  proxmox_url              = var.proxmox_url
  username                 = "root@pam!packer"
  token                    = var.proxmox_token
  insecure_skip_tls_verify = true
  node                     = "pve01"

  vm_id                = 9000
  vm_name              = "ubuntu-template"
  template_name        = "ubuntu-22.04-template"
  template_description = "Ubuntu 22.04 base template - built by Packer"

  iso_file    = "local:iso/ubuntu-22.04.4-live-server-amd64.iso"
  unmount_iso = true

  os       = "l26"
  cores    = 2
  memory   = 2048
  machine  = "q35"
  bios     = "ovmf"
  cpu_type = "host"

  scsi_controller = "virtio-scsi-single"

  disks {
    disk_size    = "20G"
    storage_pool = "local-lvm"
    type         = "scsi"
    format       = "raw"
    io_thread    = true
  }

  efi_config {
    efi_storage_pool  = "local-lvm"
    efi_type          = "4m"
    pre_enrolled_keys = true
  }

  network_adapters {
    bridge   = "vmbr0"
    model    = "virtio"
    firewall = false
  }

  cloud_init              = true
  cloud_init_storage_pool = "local-lvm"

  ssh_username = "ubuntu"
  ssh_password = "ubuntu"
  ssh_timeout  = "20m"

  # Boot command for autoinstall (subiquity)
  boot_command = [
    "<spacebar><wait><spacebar><wait><spacebar><wait><spacebar><wait><spacebar><wait>",
    "e<wait>",
    "<down><down><down><end>",
    " autoinstall ds=nocloud-net\\;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/",
    "<f10>",
  ]

  http_directory = "http"  # Serve autoinstall configs via built-in HTTP server
}

build {
  sources = ["source.proxmox-iso.ubuntu"]

  provisioner "shell" {
    inline = [
      "while [ ! -f /var/lib/cloud/instance/boot-finished ]; do sleep 1; done",
      "sudo apt-get update",
      "sudo apt-get install -y qemu-guest-agent cloud-init",
      "sudo systemctl enable qemu-guest-agent",
      "sudo cloud-init clean",
      "sudo truncate -s 0 /etc/machine-id",
    ]
  }
}
```

### Proxmox Clone Builder (from existing template)

```hcl
source "proxmox-clone" "app" {
  proxmox_url              = var.proxmox_url
  username                 = "root@pam!packer"
  token                    = var.proxmox_token
  insecure_skip_tls_verify = true
  node                     = "pve01"

  clone_vm    = "ubuntu-22.04-template"
  vm_name     = "app-template"
  vm_id       = 9001
  full_clone  = true

  cores  = 2
  memory = 4096

  ssh_username = "ubuntu"
  ssh_timeout  = "10m"
}

build {
  sources = ["source.proxmox-clone.app"]

  provisioner "shell" {
    scripts = ["scripts/install-app.sh"]
  }
}
```

## QEMU / KVM Builder

Build images locally with QEMU (no cloud account needed):

```hcl
packer {
  required_plugins {
    qemu = {
      version = ">= 1.0.0"
      source  = "github.com/hashicorp/qemu"
    }
  }
}

source "qemu" "ubuntu" {
  iso_url      = "https://releases.ubuntu.com/22.04/ubuntu-22.04.4-live-server-amd64.iso"
  iso_checksum = "sha256:45f873de9f8cb637345d6e66a583762730bbea30277ef7b32c9c3bd6700a32b2"

  output_directory = "output-qemu"
  vm_name          = "ubuntu-22.04.qcow2"

  format       = "qcow2"
  disk_size    = "20G"
  memory       = 2048
  cpus         = 2
  accelerator  = "kvm"   # Use KVM if available, fallback to tcg
  machine_type = "q35"

  headless     = true     # No GUI (for CI/headless servers)
  vnc_bind_address = "0.0.0.0"  # Allow remote VNC during debug

  net_device   = "virtio-net"
  disk_interface = "virtio"

  ssh_username = "ubuntu"
  ssh_password = "ubuntu"
  ssh_timeout  = "30m"

  http_directory = "http"

  boot_wait = "5s"
  boot_command = [
    "<spacebar><wait><spacebar><wait><spacebar><wait><spacebar><wait><spacebar><wait>",
    "e<wait>",
    "<down><down><down><end>",
    " autoinstall ds=nocloud-net\\;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/",
    "<f10>",
  ]

  shutdown_command = "echo 'ubuntu' | sudo -S shutdown -P now"
}

build {
  sources = ["source.qemu.ubuntu"]

  provisioner "shell" {
    inline = [
      "while [ ! -f /var/lib/cloud/instance/boot-finished ]; do sleep 1; done",
      "sudo apt-get update && sudo apt-get upgrade -y",
      "sudo apt-get install -y qemu-guest-agent",
    ]
  }
}
```

### QEMU Tips

```bash
# Build with QEMU (requires KVM or will be slow)
packer build -var 'headless=true' qemu-ubuntu.pkr.hcl

# Convert output format
qemu-img convert -f qcow2 -O raw output-qemu/ubuntu-22.04.qcow2 ubuntu.raw

# Import into KVM/libvirt
virt-install --import --name ubuntu-template \
  --disk output-qemu/ubuntu-22.04.qcow2,format=qcow2 \
  --memory 2048 --vcpus 2 --os-variant ubuntu22.04

# Import into Proxmox
qm importdisk 9000 output-qemu/ubuntu-22.04.qcow2 local-lvm
```

## Docker Builder

```hcl
packer {
  required_plugins {
    docker = {
      version = ">= 1.0.0"
      source  = "github.com/hashicorp/docker"
    }
  }
}

source "docker" "ubuntu" {
  image  = "ubuntu:22.04"
  commit = true
  changes = [
    "EXPOSE 80",
    "CMD [\"/usr/sbin/nginx\", \"-g\", \"daemon off;\"]",
  ]
}

build {
  sources = ["source.docker.ubuntu"]

  provisioner "shell" {
    inline = [
      "apt-get update",
      "apt-get install -y nginx",
    ]
  }

  post-processor "docker-tag" {
    repository = "myregistry.example.com/nginx"
    tags       = ["latest", "1.0"]
  }
}
```

## Builders Quick Reference

| Builder | Plugin | Use Case |
|---------|--------|----------|
| `amazon-ebs` | `hashicorp/amazon` | AWS AMIs from EBS-backed instances |
| `amazon-ebssurrogate` | `hashicorp/amazon` | Custom root device AMIs |
| `azure-arm` | `hashicorp/azure` | Azure managed images |
| `googlecompute` | `hashicorp/googlecompute` | GCP images |
| `proxmox-iso` | `hashicorp/proxmox` | Proxmox templates from ISO |
| `proxmox-clone` | `hashicorp/proxmox` | Proxmox templates from existing VM |
| `qemu` | `hashicorp/qemu` | Local QEMU/KVM qcow2/raw images |
| `vsphere-iso` | `hashicorp/vsphere` | VMware vSphere templates from ISO |
| `vsphere-clone` | `hashicorp/vsphere` | VMware vSphere templates from existing VM |
| `docker` | `hashicorp/docker` | Docker images (commit or export) |
| `vagrant` | `hashicorp/vagrant` | Vagrant boxes |
| `virtualbox-iso` | `hashicorp/virtualbox` | VirtualBox OVA/OVF from ISO |
| `null` | (built-in) | No builder — just run provisioners on existing machine |

## HCL2 Functions

Common functions used in Packer templates:

```hcl
locals {
  # Timestamps
  timestamp  = formatdate("YYYYMMDD-hhmm", timestamp())
  date_only  = formatdate("YYYY-MM-DD", timestamp())

  # String manipulation
  name_slug  = replace(lower(var.app_name), " ", "-")
  trimmed    = trimspace(var.raw_input)

  # Conditionals
  ami_prefix = var.env == "prod" ? "prod" : "dev"

  # File operations
  user_data  = file("scripts/user-data.sh")
  b64_data   = base64encode(file("scripts/user-data.sh"))

  # Collections
  all_tags   = merge(var.default_tags, { BuildDate = local.timestamp })
  regions    = compact(split(",", var.ami_regions))
}
```

| Function | Example | Result |
|----------|---------|--------|
| `formatdate` | `formatdate("YYYYMMDD", timestamp())` | `"20240115"` |
| `timestamp` | `timestamp()` | `"2024-01-15T10:30:00Z"` |
| `lower` / `upper` | `lower("Hello")` | `"hello"` |
| `replace` | `replace("a-b-c", "-", "_")` | `"a_b_c"` |
| `split` / `join` | `split(",", "a,b,c")` | `["a", "b", "c"]` |
| `trimspace` | `trimspace(" hi ")` | `"hi"` |
| `file` | `file("path/to/file")` | File contents as string |
| `base64encode` | `base64encode("hello")` | `"aGVsbG8="` |
| `merge` | `merge({a=1}, {b=2})` | `{a=1, b=2}` |
| `compact` | `compact(["a", "", "b"])` | `["a", "b"]` |
| `contains` | `contains(["a","b"], "a")` | `true` |
| `length` | `length(["a","b","c"])` | `3` |
| `lookup` | `lookup({a="1"}, "a", "default")` | `"1"` |
| `can` | `can(regex("^ami-", var.id))` | `true` / `false` |
| `regex` | `regex("^ami-(.+)$", "ami-123")` | `["123"]` |
| `regex_replace` | `regex_replace(timestamp(), "[- TZ:]", "")` | `"20240115103000"` |
| `legacy_isotime` | `legacy_isotime("2006-01-02-1504")` | `"2024-01-15-1030"` |
| `templatefile` | `templatefile("f.tpl", {k="v"})` | Rendered template |
| `jsondecode` | `jsondecode(file("c.json"))` | Parsed object |

## Important Files & Directories

| Path | Purpose |
|------|---------|
| `*.pkr.hcl` | HCL2 template files |
| `*.pkrvars.hcl` | Variable definition files |
| `*.auto.pkrvars.hcl` | Auto-loaded variable files |
| `~/.config/packer/plugins/` | Installed plugins (Linux/macOS) |
| `%APPDATA%/packer.d/plugins/` | Installed plugins (Windows) |
| `packer_cache/` | Downloaded ISOs/images cache |
| `manifest.json` | Build artifact manifest (from post-processor) |
| `.packer.d/` | Legacy plugin directory |

## Plugin Management

```bash
# Install plugins defined in required_plugins block
packer init .

# Upgrade plugins to latest matching version
packer init -upgrade .

# List installed plugins
packer plugins installed

# Install a specific plugin manually
packer plugins install github.com/hashicorp/proxmox

# Remove a plugin
packer plugins remove github.com/hashicorp/docker
```

## Advanced Source Configurations

### AWS EBS — Full Configuration

```hcl
source "amazon-ebs" "ubuntu" {
  ami_name                    = local.name
  instance_type               = var.instance_type
  region                      = var.region
  source_ami                  = data.amazon-ami.ubuntu.id
  ssh_username                = "ubuntu"
  vpc_id                      = var.vpc_id
  subnet_id                   = var.subnet_id
  security_group_ids          = [var.security_group_id]
  associate_public_ip_address = true

  # EBS volume configuration
  launch_block_device_mappings {
    device_name           = "/dev/sda1"
    volume_size           = 20
    volume_type           = "gp3"
    iops                  = 3000
    throughput            = 125
    encrypted             = true
    delete_on_termination = true
  }

  # Additional EBS volume
  launch_block_device_mappings {
    device_name = "/dev/sdf"
    volume_size = 100
    volume_type = "gp3"
    encrypted   = true
  }

  # Instance metadata options (IMDSv2)
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
  }

  # Tags for the resulting AMI
  tags = merge(var.tags, {
    Name = local.name
    Type = "golden-image"
  })

  # Tags for the AMI snapshots
  snapshot_tags = var.tags

  # Tags on the temporary build instance
  run_tags = {
    Name = "packer-build-${local.name}"
    Type = "temporary"
  }

  # Use Ed25519 temporary key pair
  temporary_key_pair_type = "ed25519"

  # User data (cloud-init)
  user_data_file = "./scripts/cloud-init.yaml"
}
```

### AWS EBS — Spot Instances

```hcl
source "amazon-ebs" "spot" {
  ami_name           = local.name
  region             = var.region
  ssh_username       = "ubuntu"
  spot_price         = "0.05"
  spot_instance_types = ["t3.medium", "t3a.medium"]

  source_ami_filter {
    filters = {
      name                = "ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"
      root-device-type    = "ebs"
      virtualization-type = "hvm"
    }
    owners      = ["099720109477"]
    most_recent = true
  }
}
```

### Azure ARM — Full Configuration

```hcl
source "azure-arm" "ubuntu" {
  # Authentication
  use_azure_cli_auth = true

  # Resource configuration
  subscription_id     = var.subscription_id
  resource_group_name = var.resource_group
  location            = var.location

  # VM configuration
  vm_size = var.vm_size

  # Source image
  image_publisher = "Canonical"
  image_offer     = "0001-com-ubuntu-server-focal"
  image_sku       = "20_04-lts-gen2"

  # Target image
  managed_image_name                = local.name
  managed_image_resource_group_name = var.image_resource_group

  # OS disk
  os_type         = "Linux"
  os_disk_size_gb = 30

  # Build resource group (temporary)
  build_resource_group_name = "packer-build-${local.timestamp}"

  # Shared Image Gallery distribution
  shared_image_gallery_destination {
    subscription_id     = var.subscription_id
    resource_group      = var.sig_resource_group
    gallery_name        = var.sig_name
    image_name          = var.sig_image_name
    image_version       = "1.0.${local.timestamp}"
    replication_regions = ["West US 2", "East US"]
  }

  ssh_username = "ubuntu"
}
```

### VMware vSphere ISO Builder

```hcl
source "vsphere-iso" "ubuntu" {
  # vSphere connection
  vcenter_server      = var.vcenter_server
  username            = var.vcenter_username
  password            = var.vcenter_password
  insecure_connection = false

  # Placement
  datacenter = var.datacenter
  cluster    = var.cluster
  datastore  = var.datastore
  folder     = var.folder

  vm_name       = local.name
  guest_os_type = "ubuntu64Guest"

  # Hardware
  CPUs      = 2
  RAM       = 4096
  disk_size = 20000

  # Network
  network_adapters {
    network      = var.network
    network_card = "vmxnet3"
  }

  # ISO configuration
  iso_url      = "https://releases.ubuntu.com/22.04/ubuntu-22.04.4-live-server-amd64.iso"
  iso_checksum = "sha256:45f873de9f8cb637345d6e66a583762730bbea30277ef7b32c9c3bd6700a32b2"

  # Boot configuration
  boot_command = [
    "<enter><wait10><wait10><wait10><wait10>",
    "/casper/vmlinuz root=/dev/sr0 initrd=/casper/initrd quiet autoinstall ds=nocloud-net;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/",
    "<enter>",
  ]

  # HTTP server for autoinstall
  http_directory = "./http"

  # SSH
  ssh_username = "ubuntu"
  ssh_password = var.ssh_password
  ssh_timeout  = "20m"

  # Shutdown
  shutdown_command = "echo 'ubuntu' | sudo -S shutdown -P now"
}
```

### VMware ISO (Workstation/Fusion) Builder

```hcl
source "vmware-iso" "ubuntu" {
  iso_url      = var.iso_url
  iso_checksum = var.iso_checksum

  # VM hardware
  cpus         = 2
  memory       = 4096
  disk_size    = 40960
  disk_type_id = 0

  # Network
  network = "nat"

  # VMware tools
  tools_upload_flavor = "linux"

  # Boot configuration
  boot_command = [
    "<enter><wait><f6><esc>",
    "<bs><bs><bs><bs><bs><bs><bs><bs><bs><bs>",
    "<bs><bs><bs><bs><bs><bs><bs><bs><bs><bs>",
    "<bs><bs><bs><bs><bs><bs><bs><bs><bs><bs>",
    "<bs><bs><bs><bs><bs><bs><bs><bs><bs><bs>",
    "<bs><bs><bs><bs><bs><bs><bs><bs><bs><bs>",
    "/install/vmlinuz noapic ",
    "preseed/url=http://{{ .HTTPIP }}:{{ .HTTPPort }}/preseed.cfg ",
    "debian-installer=en_US auto locale=en_US kbd-chooser/method=us ",
    "hostname={{ .Name }} ",
    "fb=false debconf/frontend=noninteractive ",
    "keyboard-configuration/modelcode=SKIP keyboard-configuration/layout=USA ",
    "keyboard-configuration/variant=USA console-setup/ask_detect=false ",
    "initrd=/install/initrd.gz -- <enter>",
  ]

  http_directory  = "./http"
  ssh_username    = "ubuntu"
  ssh_password    = var.ssh_password
  ssh_timeout     = "20m"
  shutdown_command = "echo 'ubuntu' | sudo -S shutdown -P now"
}
```

## Multi-Builder Templates

```hcl
packer {
  required_version = ">= 1.8.0"
  required_plugins {
    amazon = {
      version = ">= 1.2.0"
      source  = "github.com/hashicorp/amazon"
    }
    azure = {
      version = ">= 1.4.0"
      source  = "github.com/hashicorp/azure"
    }
  }
}

build {
  name = "multi-cloud"
  sources = [
    "source.amazon-ebs.ubuntu",
    "source.azure-arm.ubuntu",
  ]

  # Common provisioning (runs on all builders)
  provisioner "shell" {
    scripts = [
      "./scripts/base.sh",
      "./scripts/security.sh",
    ]
  }

  # Cloud-specific provisioning
  provisioner "shell" {
    only   = ["amazon-ebs.ubuntu"]
    script = "./scripts/aws-cloudwatch.sh"
  }

  provisioner "shell" {
    only   = ["azure-arm.ubuntu"]
    script = "./scripts/azure-monitoring.sh"
  }

  post-processor "manifest" {
    output = "build-manifest-${build.name}.json"
  }
}
```

## Dynamic Blocks

```hcl
variable "security_groups" {
  type    = list(string)
  default = ["web-sg", "ssh-sg"]
}

source "amazon-ebs" "web" {
  # Dynamic security group filter
  dynamic "security_group_filter" {
    for_each = var.security_groups
    content {
      filters = {
        "group-name" = security_group_filter.value
      }
    }
  }
}
```

## Template Functions (Advanced)

```hcl
locals {
  # Remove special characters from timestamp
  clean_timestamp = regex_replace(timestamp(), "[- TZ:]", "")

  # JSON manipulation
  config = jsondecode(file("./config.json"))

  # Template rendering
  setup_script = templatefile("./scripts/setup.tpl", {
    app_version = var.app_version
    environment = var.environment
  })

  # Base64 encoding for user data
  user_data = base64encode(file("./scripts/user-data.sh"))

  # Conditional logic
  ami_prefix = var.environment == "prod" ? "prod" : "dev"
  ami_name   = "${local.ami_prefix}-${var.app_name}-${local.clean_timestamp}"
}
```

## Build Caching

```bash
# Set cache directory
export PACKER_CACHE_DIR="./packer_cache"

# Pre-download ISOs for offline builds
mkdir -p "$PACKER_CACHE_DIR"
wget -P "$PACKER_CACHE_DIR" "https://releases.ubuntu.com/22.04/ubuntu-22.04.4-live-server-amd64.iso"
```

## Recipes

### Recipe: AMI Pipeline (Base + App Layers)

```bash
#!/bin/bash
# Two-stage AMI pipeline: base OS → application layer

set -euo pipefail

echo "=== Stage 1: Building base AMI ==="
packer build -var-file=base.pkrvars.hcl base.pkr.hcl

# Extract AMI ID from manifest
BASE_AMI=$(jq -r '.builds[-1].artifact_id' manifest-base.json | cut -d: -f2)
echo "Base AMI: $BASE_AMI"

echo "=== Stage 2: Building app AMI ==="
packer build -var "source_ami=$BASE_AMI" -var-file=app.pkrvars.hcl app.pkr.hcl

APP_AMI=$(jq -r '.builds[-1].artifact_id' manifest-app.json | cut -d: -f2)
echo "App AMI: $APP_AMI"
```

### Recipe: Build for Multiple Regions

```hcl
variable "ami_regions" {
  type    = list(string)
  default = ["us-east-1", "us-west-2", "eu-west-1"]
}

source "amazon-ebs" "multi_region" {
  region      = "us-east-1"
  ami_name    = "app-${local.timestamp}"
  ami_regions = var.ami_regions  # Copy AMI to these regions after build

  # Encrypt copies in destination regions
  region_kms_key_ids = {
    "us-west-2" = "alias/packer-usw2"
    "eu-west-1" = "alias/packer-euw1"
  }
}
```

### Recipe: Deregister Old AMIs Before Build

```bash
#!/bin/bash
# Clean up AMIs older than 30 days with prefix "app-"
CUTOFF=$(date -d '30 days ago' +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -v-30d +%Y-%m-%dT%H:%M:%S)

aws ec2 describe-images --owners self \
  --filters "Name=name,Values=app-*" \
  --query "Images[?CreationDate<'${CUTOFF}'].[ImageId,Name]" --output text | \
while read ami_id name; do
  echo "Deregistering: $ami_id ($name)"
  # Find and delete associated snapshots
  SNAPS=$(aws ec2 describe-images --image-ids "$ami_id" \
    --query 'Images[0].BlockDeviceMappings[].Ebs.SnapshotId' --output text)
  aws ec2 deregister-image --image-id "$ami_id"
  for snap in $SNAPS; do
    aws ec2 delete-snapshot --snapshot-id "$snap"
  done
done
```

### Recipe: Validate in CI, Build on Merge

```yaml
# .github/workflows/packer.yml
name: Packer
on:
  pull_request:
    paths: ['packer/**']
  push:
    branches: [main]
    paths: ['packer/**']

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-packer@main
      - run: |
          cd packer
          packer init .
          packer fmt -check -diff .
          packer validate -var-file=ci.pkrvars.hcl .

  build:
    needs: validate
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-packer@main
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/packer-build
          aws-region: us-east-1
      - run: |
          cd packer
          packer init .
          packer build -var-file=ci.pkrvars.hcl -color=false -timestamp-ui .
```

### Recipe: Automated Image Testing

```bash
#!/bin/bash
# Test a built image before promoting to production
set -euo pipefail

AMI_ID=$(jq -r '.builds[0].artifact_id' manifest.json | cut -d':' -f2)
echo "Testing AMI: $AMI_ID"

# Launch test instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t3.micro \
  --key-name my-key \
  --security-group-ids sg-12345 \
  --subnet-id subnet-67890 \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=packer-test}]" \
  --query 'Instances[0].InstanceId' --output text)

echo "Launched instance: $INSTANCE_ID"

# Wait for instance to be running
aws ec2 wait instance-status-ok --instance-ids "$INSTANCE_ID"

# Get public IP
IP=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

# Run tests
ssh -o StrictHostKeyChecking=no ubuntu@"$IP" 'systemctl is-active nginx'

# Cleanup
aws ec2 terminate-instances --instance-ids "$INSTANCE_ID"
echo "Test passed. AMI $AMI_ID is ready."
```

### Recipe: CI/CD Build Script

```bash
#!/bin/bash
# build-image.sh — full CI pipeline
set -euo pipefail

echo "=== Validating ==="
packer validate -var-file="environments/${ENVIRONMENT}.pkrvars.hcl" template.pkr.hcl

echo "=== Building ==="
packer build \
  -var "version=${BUILD_NUMBER}" \
  -var "commit_sha=${GIT_COMMIT}" \
  -var-file="environments/${ENVIRONMENT}.pkrvars.hcl" \
  -color=false \
  -timestamp-ui \
  template.pkr.hcl

# Parse manifest for image ID
AMI_ID=$(jq -r '.builds[0].artifact_id' manifest.json | cut -d':' -f2)
echo "Built AMI: ${AMI_ID}"

# Update Terraform variables
echo "ami_id = \"${AMI_ID}\"" > terraform.auto.tfvars
```

## Environment-Specific Variable Files

```hcl
# environments/dev.pkrvars.hcl
environment       = "dev"
instance_type     = "t2.micro"
vpc_id            = "vpc-dev123"
subnet_id         = "subnet-dev456"
enable_debugging  = true
install_dev_tools = true

# environments/prod.pkrvars.hcl
environment       = "prod"
instance_type     = "t3.large"
vpc_id            = "vpc-prod789"
subnet_id         = "subnet-prod012"
enable_debugging  = false
install_dev_tools = false
encrypt_boot      = true
```

```bash
# Build for specific environment
packer build -var-file=environments/dev.pkrvars.hcl .
packer build -var-file=environments/prod.pkrvars.hcl .
```

## Debugging

```bash
# Enable detailed logging
export PACKER_LOG=1
export PACKER_LOG_PATH="./packer.log"
packer build template.pkr.hcl

# Debug mode (pauses between steps, writes SSH key to disk)
packer build -debug template.pkr.hcl

# On-error behavior
packer build -on-error=abort .    # Leave instance running on failure
packer build -on-error=ask .      # Prompt what to do on failure
packer build -on-error=cleanup .  # Default: terminate on failure

# Validate syntax only (faster, no provider checks)
packer validate -syntax-only template.pkr.hcl
```
