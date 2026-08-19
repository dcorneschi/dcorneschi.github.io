# Terraform Debugging Guide

Terraform is powerful, but when things go wrong, debugging can feel opaque. Unlike traditional programming where you get stack traces and breakpoints, Terraform errors can range from cryptic provider messages to silent state corruption.

This guide covers how to enable verbose logging for deeper investigation, the most common issues you'll encounter, and practical strategies to resolve them quickly.

## Enabling Verbose Logging

The first step in any troubleshooting session is getting more information. Terraform uses the `TF_LOG` environment variable to control log verbosity.

### Log Levels

| Level | Description |
|-------|-------------|
| `TRACE` | Most verbose — shows every API call, internal decisions, and plugin communication |
| `DEBUG` | Detailed diagnostic information without low-level protocol noise |
| `INFO` | General operational events |
| `WARN` | Potential issues that don't prevent execution |
| `ERROR` | Only critical failures |

### Basic Usage

```bash
# Enable debug logging for a single command
TF_LOG=DEBUG terraform plan

# Enable trace logging and save to file
TF_LOG=TRACE TF_LOG_PATH=./terraform.log terraform apply

# Separate core and provider logging
TF_LOG_CORE=TRACE TF_LOG_PROVIDER=DEBUG terraform plan
```

### Persistent Logging

For ongoing debugging sessions, export the variables:

```bash
export TF_LOG=DEBUG
export TF_LOG_PATH="./terraform-$(date +%Y%m%d_%H%M%S).log"

# Run your commands
terraform init
terraform plan
terraform apply

# Disable when done
unset TF_LOG TF_LOG_PATH
```

**Tip**: Use `TF_LOG_PROVIDER=TRACE` when you suspect the issue is with a specific provider (AWS, Azure, GCP). This isolates provider-level API calls without flooding you with core Terraform internals.

## Common Issues and Solutions

### State Lock Errors

**Symptom:**

```
Error: Error acquiring the state lock
Lock Info:
  ID:        xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  Path:      terraform.tfstate
  Operation: OperationTypePlan
```

**Cause:** A previous Terraform operation crashed or was interrupted, leaving a stale lock.

**Solution:**

```bash
# First, confirm no other operation is running
# Then force-unlock with the lock ID from the error
terraform force-unlock xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

**Prevention:**
- Never kill Terraform with `kill -9` — use `Ctrl+C` and let it clean up
- Use remote backends with proper lock timeout settings

### Provider Authentication Failures

**Symptom:**

```
Error: error configuring Terraform AWS Provider: no valid credential sources found
```

**Solution:**

```bash
# Check your credentials are set
aws sts get-caller-identity

# Or export explicitly
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
export AWS_REGION="us-east-1"

# For assumed roles, check token expiration
aws sts get-session-token
```

**Debug further:**

```bash
TF_LOG=DEBUG terraform plan 2>&1 | grep -i "auth\|credential\|token"
```

### State Drift and Out-of-Sync Resources

**Symptom:** `terraform plan` shows changes you didn't make, or resources appear to be modified externally.

**Diagnosis:**

```bash
# Detect drift without applying changes
terraform plan -refresh-only

# Show current state for a specific resource
terraform state show aws_instance.example
```

**Solution:**

```bash
# Accept the current real-world state
terraform apply -refresh-only

# Or if you want to force Terraform's desired state
terraform apply
```

**For resources modified outside Terraform:**

```bash
# Remove from state and re-import
terraform state rm aws_instance.example
terraform import aws_instance.example i-1234567890abcdef0
```

### Dependency Cycle Errors

**Symptom:**

```
Error: Cycle: aws_security_group.web, aws_security_group.db
```

**Diagnosis:**

```bash
# Visualize the dependency graph
terraform graph | dot -Tpng > graph.png

# Or look for cycles specifically
terraform graph -draw-cycles
```

**Solution:** Break circular dependencies by using separate rule resources instead of inline rules:

```hcl
# Instead of inline ingress/egress in security groups
resource "aws_security_group_rule" "web_to_db" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.web.id
  security_group_id        = aws_security_group.db.id
}
```

### Plugin/Provider Crashes

**Symptom:**

```
Error: Plugin did not respond
The plugin encountered an error, and failed to respond to the plugin.(*GRPCProvider).ReadResource call
```

**Solution:**

```bash
# Clear the plugin cache and re-download
rm -rf .terraform/providers
terraform init

# If using a plugin cache, clear it too
rm -rf ~/.terraform.d/plugin-cache
terraform init
```

**If the issue persists:**

```bash
# Check for version conflicts
terraform providers
```

```hcl
# Lock to a known-good version in your config
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "= 5.30.0"  # Pin exact version
    }
  }
}
```

### Timeout Errors

**Symptom:**

```
Error: timeout while waiting for state to become 'running' (timeout: 10m0s)
```

**Solution:** Add custom timeouts to resources that take longer than default:

```hcl
resource "aws_db_instance" "example" {
  # ... configuration ...

  timeouts {
    create = "60m"
    update = "30m"
    delete = "60m"
  }
}
```

**Debug with verbose logging:**

```bash
TF_LOG=DEBUG terraform apply 2>&1 | grep -i "waiting\|timeout\|state"
```

### "Resource Already Exists" Errors

**Symptom:**

```
Error: error creating S3 Bucket: BucketAlreadyOwnedByYou: Your previous request to create the bucket succeeded
```

**Cause:** The resource exists in the cloud but not in Terraform state (created manually, state was lost, or previous apply partially succeeded).

**Solution:**

```bash
# Import the existing resource into state
terraform import aws_s3_bucket.example my-bucket-name

# Then plan to see if any changes are needed
terraform plan
```

### Module Source Errors

**Symptom:**

```
Error: Failed to download module
Could not download module "vpc" source "git::https://example.com/modules/vpc.git"
```

**Solution:**

```bash
# Clear module cache
rm -rf .terraform/modules

# Re-initialize with verbose output
TF_LOG=DEBUG terraform init

# For Git-based modules, check SSH/HTTPS access
git ls-remote https://example.com/modules/vpc.git

# For private registries, ensure you're authenticated
terraform login
```

### Backend Configuration Errors

**Symptom:**

```
Error: Backend initialization required, please run "terraform init"
Error: Backend configuration changed
```

**Solution:**

```bash
# Reconfigure backend from scratch
terraform init -reconfigure

# Or migrate state to new backend
terraform init -migrate-state

# If state is corrupted, pull and inspect
terraform state pull > backup.tfstate
```

### "Invalid Count/For_Each" Errors

**Symptom:**

```
Error: Invalid count argument
The "count" value depends on resource attributes that cannot be determined until apply
```

**Cause:** You're using a value that isn't known until apply-time in `count` or `for_each`.

**Solution:**

```bash
# Use -target to create the dependency first
terraform apply -target=aws_instance.example

# Then run the full apply
terraform apply
```

**Better long-term fix:** Restructure your code so `count`/`for_each` depends only on variables or data sources, not other resource attributes.

## Advanced Debugging Techniques

### Inspecting State Manually

```bash
# Export state as JSON for analysis
terraform show -json > state.json

# Find specific resources
cat state.json | jq '.values.root_module.resources[] | select(.type == "aws_instance")'

# Compare with a backup
diff <(terraform state pull | jq .) <(cat terraform.tfstate.backup | jq .)
```

### Tracing API Calls

For AWS-specific issues, enable SDK logging alongside Terraform logging:

```bash
export TF_LOG=TRACE
export AWS_DEBUG=true
terraform plan 2>&1 | tee full-debug.log
```

### Generating a Crash Log

If Terraform crashes, it automatically writes a crash log:

```bash
# Find crash logs
ls crash.log

# The crash log contains the Go stack trace
# Include it when filing GitHub issues
```

### Running in Dry-Run Mode

```bash
# Plan without refreshing state (faster, useful for syntax issues)
terraform plan -refresh=false

# Validate without any cloud calls
terraform validate
```

## Troubleshooting Checklist

When you hit an issue, work through this checklist:

1. **Read the full error message** — Terraform errors often contain the solution in the details
2. **Enable verbose logging** — `TF_LOG=DEBUG terraform plan`
3. **Check provider credentials** — Expired tokens are the #1 silent failure
4. **Verify state consistency** — `terraform state list` and `terraform plan -refresh-only`
5. **Clear caches** — `rm -rf .terraform && terraform init`
6. **Check version compatibility** — `terraform version` and `terraform providers`
7. **Isolate the problem** — Use `-target` to narrow down which resource is failing
8. **Check rate limits** — Cloud providers throttle API calls; add `-parallelism=1` to slow down
9. **Review recent changes** — `git diff` your `.tf` files
10. **Search provider issues** — Check the provider's GitHub issues for known bugs

## Environment Variables Reference

| Variable | Description | Example |
|----------|-------------|---------|
| `TF_LOG` | Set log level | `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR` |
| `TF_LOG_PATH` | Write logs to file | `./terraform.log` |
| `TF_LOG_CORE` | Core Terraform logging | `TRACE` |
| `TF_LOG_PROVIDER` | Provider plugin logging | `DEBUG` |
| `TF_INPUT=false` | Disable interactive prompts | Useful in CI/CD |
| `TF_IN_AUTOMATION=true` | Adjust output for automation | Cleaner logs in pipelines |
| `TF_CLI_ARGS` | Default arguments for all commands | `-no-color` |
| `AWS_DEBUG=true` | Enable AWS SDK debug logging | Use with TF_LOG=TRACE |

## Key Takeaways

1. Start with verbose logging to understand what's happening
2. Isolate whether the issue is in Terraform core, a provider, or your configuration
3. Use the state management commands to inspect and fix state issues
4. When in doubt, `terraform init -reconfigure` and start fresh
5. Keep your Terraform and provider versions pinned
6. Back up your state before major operations
7. Always run `plan` before `apply`
