# Terraform JSON, One-Liners, and Tips & Tricks

Practical patterns for working with Terraform output, state manipulation, JSON extraction with `jq`, debugging, and everyday workflow shortcuts.

## Extracting Data with terraform output and jq

### Basic Output Extraction

```bash
# Get a single value (raw, no quotes)
terraform output -raw instance_id

# Get all outputs as JSON
terraform output -json

# Get a specific output as JSON
terraform output -json vpc_id

# Get a list output and iterate
terraform output -json instance_ids | jq -r '.[]'

# Get first element of a list
terraform output -json private_ips | jq -r '.[0]'

# Get a map value by key
terraform output -json instance_map | jq -r '.["web-1"]'

# Count elements in a list output
terraform output -json instance_ids | jq 'length'
```

### Build Commands from Outputs

```bash
# SSH into the instance
ssh ubuntu@$(terraform output -raw public_ip)

# SCP a file
scp file.txt ubuntu@$(terraform output -raw public_ip):/tmp/

# AWS CLI with terraform output
aws ec2 describe-instances --instance-ids $(terraform output -raw instance_id)

# kubectl with EKS
aws eks update-kubeconfig --name $(terraform output -raw cluster_name) --region $(terraform output -raw region)

# Docker login to ECR
aws ecr get-login-password --region $(terraform output -raw region) | \
  docker login --username AWS --password-stdin $(terraform output -raw ecr_url)
```

### Export Outputs as Environment Variables

```bash
# Export a single output
export DB_HOST=$(terraform output -raw db_endpoint)

# Export multiple outputs
eval $(terraform output -json | jq -r 'to_entries[] | "export TF_\(.key | ascii_upcase)=\(.value.value)"')

# Export a map output as individual vars
terraform output -json azure_info | jq -r 'to_entries[] | "export \(.key)=\(.value)"'

# Source into current shell
terraform output -json azure_info | jq -r 'to_entries[] | "\(.key)=\(.value)"' > .env
source .env
```

## Working with terraform show

### Extract from State

```bash
# Full state as JSON
terraform show -json | jq .

# All outputs
terraform show -json | jq '.values.outputs'

# All resources
terraform show -json | jq '.values.root_module.resources'

# Find a specific resource
terraform show -json | jq '.values.root_module.resources[] | select(.address == "aws_instance.web")'

# Get all resource addresses
terraform show -json | jq -r '.values.root_module.resources[].address'

# Get all IPs from instances
terraform show -json | jq -r '.values.root_module.resources[] | select(.type == "aws_instance") | .values.public_ip'

# Resources in child modules
terraform show -json | jq '.values.root_module.child_modules[].resources[]'
```

### Extract from Plan

```bash
# Save plan to file
terraform plan -out=tfplan

# Convert plan to JSON
terraform show -json tfplan > tfplan.json

# One-liner without saving JSON separately
terraform plan -out=tfplan && terraform show -json tfplan

# Show what will be created
jq '.resource_changes[] | select(.change.actions[] == "create") | .address' tfplan.json

# Show what will be destroyed
jq '.resource_changes[] | select(.change.actions[] == "delete") | .address' tfplan.json

# Show what will be updated in-place
jq '.resource_changes[] | select(.change.actions[] == "update") | .address' tfplan.json

# Show only resources with actual changes (exclude no-op)
jq '.resource_changes[] | select(.change.actions != ["no-op"])' tfplan.json

# Summary: address and planned actions
jq '.resource_changes[] | {address: .address, actions: .change.actions}' tfplan.json

# Pretty-print: one line per resource with action
jq -r '.resource_changes[] | "\(.address): \(.change.actions | join(", "))"' tfplan.json

# Count changes grouped by action type
jq '.resource_changes | group_by(.change.actions) | map({action: .[0].change.actions, count: length})' tfplan.json

# Show before/after for a specific resource
jq '.resource_changes[] | select(.address == "aws_instance.web") | .change' tfplan.json

# Show detailed changes (address, actions, before, after) for all changed resources
jq '.resource_changes[] | select(.change.actions != ["no-op"]) | {address, actions: .change.actions, before: .change.before, after: .change.after}' tfplan.json
```

## State Manipulation One-Liners

### terraform state commands

```bash
# List all resources in state
terraform state list

# List resources matching a pattern
terraform state list | grep aws_instance
terraform state list 'module.vpc.*'

# Show details of a specific resource
terraform state show aws_instance.web
terraform state show 'module.vpc.aws_subnet.public[0]'

# Move a resource (rename without recreating)
terraform state mv aws_instance.old aws_instance.new
terraform state mv 'module.old_name' 'module.new_name'

# Move into a module
terraform state mv aws_instance.web 'module.compute.aws_instance.web'

# Remove from state (resource still exists in cloud, Terraform forgets it)
terraform state rm aws_instance.web
terraform state rm 'module.vpc'

# Pull remote state to local file
terraform state pull > state.json

# Push local state to remote backend
terraform state push state.json

# Replace provider in state (useful after provider renames)
terraform state replace-provider hashicorp/aws registry.terraform.io/hashicorp/aws
```

### Bulk State Operations

```bash
# Remove all instances of a resource type
terraform state list | grep 'aws_cloudwatch_log_group' | xargs -I{} terraform state rm '{}'

# Move all resources from one module to another
terraform state list 'module.old' | while read resource; do
  new_resource=$(echo "$resource" | sed 's/module.old/module.new/')
  terraform state mv "$resource" "$new_resource"
done

# Count resources by type
terraform state list | awk -F'.' '{print $1"."$2}' | sort | uniq -c | sort -rn
```

## terraform console

Interactive expression evaluation — test before you commit.

```bash
# Start console
terraform console

# Test expressions
> var.environment
"production"

> length(var.subnets)
3

> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"

> formatdate("YYYY-MM-DD", timestamp())
"2024-12-15"

> jsonencode({name = "test", ports = [80, 443]})
"{\"name\":\"test\",\"ports\":[80,443]}"

> yamlencode({replicas = 3, image = "nginx:latest"})

> regex("^ami-([a-z0-9]+)$", "ami-0c55b159cbfafe1f0")
["0c55b159cbfafe1f0"]

> try(file("exists.txt"), "fallback")
"fallback"

# Test against a var file
terraform console -var-file=prod.tfvars
```

## Useful Functions for JSON/Data Manipulation

### jsonencode / jsondecode

```hcl
# Convert HCL to JSON (for IAM policies, API bodies, etc.)
locals {
  policy_json = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject"]
      Resource = "arn:aws:s3:::${var.bucket_name}/*"
    }]
  })
}

# Decode JSON from a file
locals {
  config = jsondecode(file("${path.module}/config.json"))
}

# Use decoded values
resource "aws_instance" "app" {
  instance_type = local.config.instance_type
  ami           = local.config.ami_id
}
```

### yamlencode / yamldecode

```hcl
# Generate YAML for Kubernetes manifests
locals {
  values_yaml = yamlencode({
    replicaCount = var.replicas
    image = {
      repository = var.image_repo
      tag        = var.image_tag
    }
    service = {
      type = "ClusterIP"
      port = 80
    }
  })
}

# Write to file
resource "local_file" "helm_values" {
  content  = local.values_yaml
  filename = "${path.module}/values-generated.yaml"
}
```

### templatefile

```hcl
# Render a template with variables
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = var.instance_type
  user_data     = templatefile("${path.module}/scripts/init.sh.tpl", {
    hostname    = "web-${var.environment}"
    packages    = join(" ", var.packages)
    nfs_server  = var.nfs_server_ip
    domain      = var.domain
  })
}
```

### String Manipulation

```hcl
locals {
  # Split and join
  parts      = split("-", "us-east-1")        # ["us", "east", "1"]
  rejoined   = join(", ", var.availability_zones)

  # Replace
  sanitized  = replace(var.name, "/[^a-zA-Z0-9]/", "-")

  # Regex
  account_id = regex("arn:aws:iam::(\\d+):", var.role_arn)[0]

  # Format
  bucket     = format("%s-%s-%s", var.project, var.environment, random_id.suffix.hex)

  # Upper/Lower/Title
  env_upper  = upper(var.environment)
  env_lower  = lower(var.ENVIRONMENT)

  # Trim
  cleaned    = trimspace(var.user_input)
  no_prefix  = trimprefix(var.arn, "arn:aws:")
  no_suffix  = trimsuffix(var.hostname, ".example.com")
}
```

### Collection Functions

```hcl
locals {
  # Flatten nested lists
  all_subnets = flatten([var.public_subnets, var.private_subnets])

  # Distinct values
  unique_azs = distinct(var.availability_zones)

  # Merge maps
  all_tags = merge(var.default_tags, var.extra_tags, { ManagedBy = "terraform" })

  # Lookup with default
  ami = lookup(var.ami_map, var.region, "ami-default")

  # Zipmap (parallel lists → map)
  instance_map = zipmap(aws_instance.app[*].tags["Name"], aws_instance.app[*].public_ip)

  # Sort
  sorted_ips = sort(aws_instance.app[*].private_ip)

  # Chunklist (split into batches)
  batches = chunklist(var.instances, 3)

  # Coalesce list (first non-empty list)
  subnets = coalescelist(var.custom_subnets, data.aws_subnets.default.ids)
}
```

## Debugging Tips

### Enable Logging

```bash
# Debug provider communication
export TF_LOG=DEBUG
terraform plan

# Log levels: TRACE, DEBUG, INFO, WARN, ERROR
export TF_LOG=TRACE

# Log to file instead of stderr
export TF_LOG_PATH="terraform.log"
terraform apply

# Provider-specific logging
export TF_LOG_PROVIDER=DEBUG

# Disable logging
unset TF_LOG TF_LOG_PATH
```

### Inspect State Without Modifying

```bash
# Validate syntax only
terraform validate

# Format check (CI-friendly, exits non-zero if changes needed)
terraform fmt -check -recursive

# Show providers in use
terraform providers

# Show provider lock file
cat .terraform.lock.hcl

# Graph dependencies (output DOT format)
terraform graph | dot -Tpng > graph.png
terraform graph -type=plan | dot -Tsvg > plan-graph.svg
```

### Force Recreate a Resource

```bash
# Taint a resource (deprecated but still works)
terraform taint aws_instance.web

# Replace (modern approach, Terraform 0.15.2+)
terraform apply -replace="aws_instance.web"

# Replace a resource in a module
terraform apply -replace="module.vpc.aws_subnet.public[0]"
```

## Workflow One-Liners

### Planning and Applying

```bash
# Plan with specific var file
terraform plan -var-file=environments/prod.tfvars

# Apply auto-approve (CI/CD pipelines only)
terraform apply -auto-approve

# Destroy specific resource only
terraform destroy -target=aws_instance.web

# Apply only specific resources
terraform apply -target=module.vpc -target=aws_instance.web

# Plan for destroy
terraform plan -destroy

# Refresh state without making changes
terraform apply -refresh-only

# Import existing resource into state
terraform import aws_instance.web i-1234567890abcdef0
terraform import 'module.vpc.aws_subnet.public[0]' subnet-abc123
```

### Working with Workspaces

```bash
# List workspaces
terraform workspace list

# Create and switch
terraform workspace new staging
terraform workspace select production

# Show current workspace
terraform workspace show

# Delete workspace
terraform workspace delete staging

# Use in config
locals {
  environment = terraform.workspace
}
```

### Lock and Unlock State

```bash
# Force unlock (use with caution — only if lock is stale)
terraform force-unlock LOCK_ID

# Plan with lock timeout
terraform plan -lock-timeout=5m
```

## Practical jq Recipes

### Parse terraform.tfstate Directly

```bash
# Get all resource types
jq -r '.resources[].type' terraform.tfstate | sort -u

# Get all instance IDs
jq -r '.resources[] | select(.type == "aws_instance") | .instances[].attributes.id' terraform.tfstate

# Get all security group rules
jq '.resources[] | select(.type == "aws_security_group") | .instances[].attributes' terraform.tfstate

# Find resources by name pattern
jq -r '.resources[] | select(.name | test("web")) | .type + "." + .name' terraform.tfstate

# Export all outputs from state
jq '.outputs | to_entries[] | "\(.key) = \(.value.value)"' terraform.tfstate
```

### CI/CD Pipeline Patterns

```bash
# Check if plan has changes (exit code based)
terraform plan -detailed-exitcode
# Exit 0 = no changes, Exit 1 = error, Exit 2 = changes present

# Generate plan summary for PR comment
terraform plan -no-color 2>&1 | tail -5

# Count resources to be changed
terraform plan -out=tfplan && terraform show -json tfplan | jq '[.resource_changes[] | select(.change.actions != ["no-op"])] | length'

# Get list of changed resources for review
terraform plan -out=tfplan && terraform show -json tfplan | jq -r '.resource_changes[] | select(.change.actions != ["no-op"]) | "\(.change.actions | join(",")): \(.address)"'
```

## HCL Tips

### Dynamic Backend Configuration

```bash
# Initialize with backend config from CLI
terraform init \
  -backend-config="bucket=my-state-bucket" \
  -backend-config="key=prod/terraform.tfstate" \
  -backend-config="region=eu-west-1"

# Or from a file
terraform init -backend-config=backends/prod.hcl
```

```hcl
# backends/prod.hcl
bucket         = "my-state-bucket"
key            = "prod/terraform.tfstate"
region         = "eu-west-1"
dynamodb_table = "terraform-locks"
encrypt        = true
```

### Override Files

Terraform automatically loads `*_override.tf` files — useful for local dev overrides without modifying tracked files.

```hcl
# override.tf (not committed to git)
# Force a specific provider version locally
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "= 5.31.0"
    }
  }
}
```

### Prevent Accidental Destroys

```hcl
resource "aws_db_instance" "production" {
  # ...

  lifecycle {
    prevent_destroy = true
  }
}
```

### Ignore External Changes

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = var.instance_type

  lifecycle {
    ignore_changes = [
      tags["LastModifiedBy"],
      ami,  # Don't replace instance when AMI updates
    ]
  }
}
```

### Create Before Destroy

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = var.instance_type

  lifecycle {
    create_before_destroy = true
  }
}
```

## File Generation

### Generate Ansible Inventory

```hcl
resource "local_file" "ansible_inventory" {
  content = templatefile("${path.module}/templates/inventory.tpl", {
    web_servers = aws_instance.web[*].public_ip
    db_servers  = aws_instance.db[*].private_ip
    ssh_user    = var.ssh_user
  })
  filename = "${path.module}/inventory.ini"
}
```

### Generate .env File

```hcl
resource "local_file" "env" {
  content = <<-EOT
    DB_HOST=${aws_db_instance.main.endpoint}
    DB_NAME=${aws_db_instance.main.db_name}
    REDIS_HOST=${aws_elasticache_cluster.main.cache_nodes[0].address}
    S3_BUCKET=${aws_s3_bucket.assets.id}
    REGION=${var.region}
  EOT
  filename        = "${path.module}/.env"
  file_permission = "0600"
}
```

### Generate SSH Config

```hcl
resource "local_file" "ssh_config" {
  content = join("\n", [for name, ip in zipmap(
    aws_instance.app[*].tags["Name"],
    aws_instance.app[*].public_ip
  ) : <<-EOT
    Host ${name}
      HostName ${ip}
      User ${var.ssh_user}
      IdentityFile ~/.ssh/id_rsa
      StrictHostKeyChecking no
  EOT
  ])
  filename = "${path.module}/ssh_config"
}
```

## Quick Reference

```bash
# Common workflow
terraform init                              # Initialize
terraform fmt -recursive                    # Format all files
terraform validate                          # Check syntax
terraform plan -out=tfplan                  # Plan and save
terraform apply tfplan                      # Apply saved plan
terraform output -json | jq .              # View outputs

# Debugging
TF_LOG=DEBUG terraform plan                 # Verbose logging
terraform console                           # Interactive REPL
terraform show -json | jq .                 # Inspect state as JSON
terraform show -json tfplan | jq .resource_changes  # Plan changes as JSON

# State
terraform state list | wc -l               # Count resources
terraform state show aws_instance.web      # Inspect resource
terraform apply -replace="resource.name"   # Force recreate
terraform import type.name cloud-id        # Import existing
```
