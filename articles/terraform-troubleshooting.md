# Terraform Troubleshooting

Common errors, debugging techniques, state recovery, provider issues, and practical fixes for day-to-day Terraform problems.

## Common Errors and Fixes

### "Error acquiring the state lock"

The state is locked by another process (or a stale lock from a crashed run).

```bash
# See who holds the lock (error message includes lock ID)
# Error: Error acquiring the state lock
# Lock Info:
#   ID:        a1b2c3d4-5678-...
#   Operation: OperationTypePlan
#   Who:       user@hostname

# Force unlock (only if you're sure no other process is running)
terraform force-unlock a1b2c3d4-5678-...

# Check if DynamoDB lock table has stale entries (S3 backend)
aws dynamodb scan --table-name terraform-locks --output json | jq '.Items[]'

# Delete a specific stale lock from DynamoDB
aws dynamodb delete-item \
  --table-name terraform-locks \
  --key '{"LockID": {"S": "my-state-bucket/env/terraform.tfstate-md5"}}'
```

### "Error: Resource already exists"

Terraform tries to create a resource that already exists in the cloud.

```bash
# Solution 1: Import the existing resource
terraform import aws_instance.web i-0abc123def456789
terraform import 'aws_s3_bucket.main' my-existing-bucket
terraform import 'module.vpc.aws_subnet.public[0]' subnet-abc123

# Solution 2: Remove from state and let Terraform recreate
terraform state rm aws_instance.web
terraform apply

# Solution 3: Use import blocks (Terraform 1.5+)
# Add to your config:
```

```hcl
import {
  to = aws_instance.web
  id = "i-0abc123def456789"
}
```

### "Error: Cycle detected"

Circular dependency between resources.

```bash
# Visualize the dependency graph
terraform graph | dot -Tpng > graph.png

# Find the cycle
terraform graph 2>&1 | grep -i cycle

# Common fix: break the cycle with depends_on or separate resources
# Example: Security group referencing itself
```

```hcl
# BAD — cycle
resource "aws_security_group" "a" {
  ingress {
    security_groups = [aws_security_group.b.id]
  }
}
resource "aws_security_group" "b" {
  ingress {
    security_groups = [aws_security_group.a.id]
  }
}

# GOOD — use separate rules
resource "aws_security_group" "a" {}
resource "aws_security_group" "b" {}

resource "aws_security_group_rule" "a_from_b" {
  type                     = "ingress"
  security_group_id        = aws_security_group.a.id
  source_security_group_id = aws_security_group.b.id
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
}
```

### "Error: Provider configuration not present"

```bash
# Usually after removing a provider or moving resources between modules

# Solution: add an empty provider block or alias
# If migrating providers:
terraform state replace-provider hashicorp/aws registry.terraform.io/hashicorp/aws
```

### "Error: Unsupported attribute"

Resource doesn't have the attribute you're referencing.

```bash
# Check available attributes
terraform state show aws_instance.web

# Check provider docs for the resource
terraform providers schema -json | jq '.provider_schemas["registry.terraform.io/hashicorp/aws"].resource_schemas["aws_instance"].attributes'
```

### "Error: Invalid count argument" / "depends on resource attributes that cannot be determined until apply"

```bash
# count/for_each can't use values that aren't known until apply

# BAD
resource "aws_subnet" "app" {
  count = length(aws_availability_zones.available.names)  # Not known until apply
}

# GOOD — use a data source that resolves at plan time
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "app" {
  count = length(data.aws_availability_zones.available.names)
}
```

### "Error: Backend configuration changed"

```bash
# When switching backends or modifying backend config

# Migrate state to new backend
terraform init -migrate-state

# Skip migration and reinitialize (loses remote state reference)
terraform init -reconfigure

# When to use which:
# -migrate-state: Moving state from local → S3, or between S3 buckets
# -reconfigure: Fresh start, backend params changed but state is already there
```

### "Error: Inconsistent dependency lock file"

```bash
# Lock file doesn't match required providers

# Update lock file for current platform
terraform init -upgrade

# Generate lock file for multiple platforms (CI/CD)
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64

# Delete and regenerate
rm .terraform.lock.hcl
terraform init
```

### "Error: Failed to query available provider packages"

```bash
# Network issues or registry unavailable

# Use a provider mirror
terraform providers mirror /path/to/mirror

# Configure filesystem mirror in .terraformrc
cat > ~/.terraformrc << 'EOF'
provider_installation {
  filesystem_mirror {
    path = "/path/to/mirror"
  }
  direct {
    exclude = []
  }
}
EOF

# Use explicit provider source
terraform init -plugin-dir=/path/to/plugins
```

## Debugging

### Enable Debug Logging

```bash
# Log levels: TRACE, DEBUG, INFO, WARN, ERROR
export TF_LOG=DEBUG
terraform plan

# Log to file
export TF_LOG=TRACE
export TF_LOG_PATH="terraform-debug.log"
terraform apply
tail -f terraform-debug.log

# Provider-specific logging
export TF_LOG_PROVIDER=DEBUG
export TF_LOG_CORE=WARN

# Disable
unset TF_LOG TF_LOG_PATH TF_LOG_PROVIDER TF_LOG_CORE
```

### Verbose Plan Output

```bash
# Show full plan (no truncation)
terraform plan -no-color 2>&1 | tee plan-output.txt

# Plan as JSON for programmatic analysis
terraform plan -out=tfplan
terraform show -json tfplan | jq '.resource_changes[] | select(.change.actions != ["no-op"])'

# Plan with specific targets to narrow scope
terraform plan -target=aws_instance.web
terraform plan -target=module.vpc
```

### Inspect Current State

```bash
# List everything in state
terraform state list

# Show a specific resource
terraform state show aws_instance.web

# Full state as JSON
terraform show -json | jq .

# Find resource by attribute value
terraform show -json | jq '.values.root_module.resources[] | select(.values.tags.Name == "web-prod")'

# Check resource drift (compare state vs cloud)
terraform plan -refresh-only
```

### Terraform Console for Testing

```bash
terraform console

# Test references
> aws_instance.web.public_ip
> module.vpc.vpc_id
> var.environment

# Test functions
> cidrsubnet("10.0.0.0/16", 8, 1)
> formatdate("YYYY-MM-DD", timestamp())
> try(file("missing.txt"), "default")

# Test conditionals
> var.environment == "prod" ? "t3.large" : "t3.micro"

# Test for expressions
> [for s in var.subnets : cidrsubnet(var.vpc_cidr, 8, s)]
```

## State Recovery

### State is Corrupted or Lost

```bash
# Pull current state (if remote backend is accessible)
terraform state pull > backup-state.json

# Push a fixed state back
terraform state push fixed-state.json

# If state is completely lost — reimport everything
terraform import aws_vpc.main vpc-abc123
terraform import aws_subnet.public subnet-def456
# ... import each resource one by one

# Generate import blocks for existing infrastructure (Terraform 1.5+)
terraform plan -generate-config-out=generated.tf
```

### State Out of Sync (Drift)

```bash
# Detect drift without making changes
terraform plan -refresh-only

# Apply refresh only (update state to match reality)
terraform apply -refresh-only

# If a resource was deleted outside Terraform
terraform state rm aws_instance.deleted_manually

# If a resource was modified outside Terraform
# Option 1: Let Terraform fix it
terraform apply

# Option 2: Accept the drift
terraform apply -refresh-only
```

### Move Resources Between States

```bash
# Remove from source state
terraform state rm -state=source.tfstate aws_instance.web

# Or using state mv between state files
terraform state mv -state=source.tfstate -state-out=dest.tfstate \
  aws_instance.web aws_instance.web

# Move into a module
terraform state mv aws_instance.web 'module.compute.aws_instance.web'

# Move out of a module
terraform state mv 'module.compute.aws_instance.web' aws_instance.web
```

### Recover from Failed Apply

```bash
# Terraform partially applied — some resources created, some failed

# Option 1: Fix the config and re-apply
terraform apply

# Option 2: Destroy only what was created
terraform destroy -target=aws_instance.failed_resource

# Option 3: If resource is tainted (failed provisioner)
# Untaint to keep it
terraform untaint aws_instance.web

# Or let it be recreated on next apply (tainted = will be destroyed and recreated)
terraform apply
```

## Provider Issues

### Upgrade Provider Without Breaking State

```bash
# Check current provider versions
terraform providers

# Upgrade providers
terraform init -upgrade

# Pin to a specific version in case of issues
```

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"  # Allow 5.30.x but not 6.x
    }
  }
}
```

### Provider Authentication Issues

```bash
# AWS — check credentials
aws sts get-caller-identity
export AWS_PROFILE=my-profile

# Azure — re-authenticate
az login
az account set --subscription "My Subscription"

# GCP — re-authenticate
gcloud auth application-default login

# Generic — check environment variables
env | grep -E "(AWS_|AZURE_|ARM_|GOOGLE_|TF_VAR_)"
```

### Multiple Provider Configurations

```hcl
# When you need resources in different regions/accounts
provider "aws" {
  region = "eu-west-1"
}

provider "aws" {
  alias  = "us"
  region = "us-east-1"
}

# Use alias in resources
resource "aws_s3_bucket" "us_bucket" {
  provider = aws.us
  bucket   = "my-us-bucket"
}
```

## Performance Issues

### Slow Plans/Applies

```bash
# Reduce parallelism (helps with API rate limiting)
terraform apply -parallelism=5

# Target specific resources to plan faster
terraform plan -target=module.app

# Disable refresh for faster plan (unsafe — may miss drift)
terraform plan -refresh=false

# Split large configs into smaller states
# Use workspaces or separate root modules
```

### API Rate Limiting / Throttling

```bash
# AWS throttling — reduce parallelism
terraform apply -parallelism=2

# Add retry logic in provider config
```

```hcl
provider "aws" {
  region = "eu-west-1"

  default_tags {
    tags = {
      ManagedBy = "terraform"
    }
  }

  # Retry configuration
  retry_mode  = "adaptive"
  max_retries = 10
}
```

### Large State Files

```bash
# Check state size
ls -lh terraform.tfstate
terraform state list | wc -l

# Split into smaller states by concern:
# - networking/
# - compute/
# - database/
# - monitoring/

# Use data sources to reference across states
```

## Common Gotchas

### Whitespace/Formatting Changes Cause Diff

```bash
# Format all files to eliminate spurious diffs
terraform fmt -recursive

# Check formatting in CI (fails if unformatted)
terraform fmt -check -recursive -diff
```

### Sensitive Values in Plan Output

```hcl
# Mark variables as sensitive
variable "db_password" {
  type      = string
  sensitive = true
}

# Mark output as sensitive
output "connection_string" {
  value     = "postgres://admin:${var.db_password}@${aws_db_instance.main.endpoint}/mydb"
  sensitive = true
}
```

```bash
# Sensitive values still exist in state file — encrypt it
# S3 backend with encryption:
```

```hcl
backend "s3" {
  bucket         = "my-state"
  key            = "terraform.tfstate"
  region         = "eu-west-1"
  encrypt        = true
  dynamodb_table = "terraform-locks"
}
```

### Terraform Tries to Recreate Resources Unnecessarily

```hcl
# Use lifecycle to prevent unwanted changes
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = var.instance_type

  lifecycle {
    # Don't recreate when AMI changes (patch manually)
    ignore_changes = [ami]

    # Never destroy this (database, critical infra)
    prevent_destroy = true

    # Zero-downtime replacement
    create_before_destroy = true
  }
}
```

### for_each vs count — When Resources Get Recreated

```hcl
# count — index-based: removing item [1] shifts [2] → [1] (recreation!)
resource "aws_instance" "app" {
  count = 3
}
# Removing the second instance forces the third to be recreated as index [1]

# for_each — key-based: removing a key only affects that key
resource "aws_instance" "app" {
  for_each = toset(["web", "api", "worker"])
}
# Removing "api" only destroys that one instance
```

### Handling Timeouts

```hcl
resource "aws_db_instance" "main" {
  # ...

  timeouts {
    create = "60m"
    update = "30m"
    delete = "45m"
  }
}

resource "aws_eks_cluster" "main" {
  # ...

  timeouts {
    create = "30m"
    update = "60m"
    delete = "15m"
  }
}
```

## Cleanup and Maintenance

### Find and Remove Orphaned Resources

```bash
# List all resources Terraform manages
terraform state list > managed.txt

# Compare with what's in the cloud
aws ec2 describe-instances --query 'Reservations[].Instances[].InstanceId' --output text > cloud.txt

# Find orphans (in cloud but not in state)
comm -23 <(sort cloud.txt) <(grep aws_instance managed.txt | sort)
```

### Clean Up .terraform Directory

```bash
# Remove plugins and reinitialize (frees disk space)
rm -rf .terraform
terraform init

# Remove only plugin cache
rm -rf .terraform/providers

# Global plugin cache (avoid re-downloading)
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"
mkdir -p "$TF_PLUGIN_CACHE_DIR"
```

### Validate Before Commit

```bash
# Pre-commit checks
terraform fmt -check -recursive
terraform validate
terraform plan -detailed-exitcode  # Exit 2 = changes present

# Lint with tflint
tflint --init
tflint --recursive

# Security scan with tfsec/trivy
trivy config .
```

## Quick Troubleshooting Checklist

| Symptom | Check First | Fix |
|---------|-------------|-----|
| Lock error | Is another `terraform` process running? | `force-unlock` if stale |
| Plan shows unexpected destroy | Did you rename a resource? | `terraform state mv` |
| Resource already exists | Was it created manually? | `terraform import` |
| Provider auth failure | Are credentials/env vars set? | Re-authenticate, check `AWS_PROFILE` |
| Cycle detected | Circular resource references | Use separate rule resources or `depends_on` |
| Count/for_each unknown | Using a computed value? | Move to data source or variable |
| Backend changed | Modified backend block? | `terraform init -migrate-state` |
| State drift | Resource changed outside TF? | `terraform apply -refresh-only` |
| Slow performance | Large state or API throttling? | Reduce `-parallelism`, split state |
| Lock file mismatch | New platform or provider update? | `terraform init -upgrade` |
| Sensitive in output | Missing `sensitive = true`? | Add to variable and output |
| Timeout | Resource takes too long | Add `timeouts {}` block |

## Emergency Commands

```bash
# I need to see what Terraform thinks it manages
terraform state list

# I need to stop managing a resource without destroying it
terraform state rm aws_instance.problem

# I need to force a resource to be recreated
terraform apply -replace="aws_instance.web"

# I need to undo the last apply (no built-in rollback — revert code and re-apply)
git checkout HEAD~1 -- .
terraform apply

# I need to destroy everything
terraform destroy

# I need to destroy just one thing
terraform destroy -target=aws_instance.web

# State is locked and no one is using it
terraform force-unlock LOCK-ID

# I need to start fresh with existing infrastructure
rm -rf .terraform terraform.tfstate*
terraform init
# Then import each resource
```
