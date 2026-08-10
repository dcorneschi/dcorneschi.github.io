# Importing Existing Infrastructure Into Terraform

## Overview

`terraform import` maps real-world infrastructure to your Terraform state so you can manage pre-existing resources with IaC. This is essential when adopting Terraform for manually created infrastructure, migrating between states, or recovering from a lost state file.

The overall workflow:

1. Write a resource block (or use `terraform plan -generate-config-out=...` to auto-generate one)
2. Import the real object into state (`terraform import` CLI or `import {}` block)
3. Run `terraform plan` and iterate until the plan shows no changes
4. Commit and continue with normal plan/apply workflow

## Import Command Syntax

```bash
terraform import <RESOURCE_ADDRESS> <RESOURCE_ID>
```

- **RESOURCE_ADDRESS** — the Terraform resource address (e.g., `aws_instance.myvm`)
- **RESOURCE_ID** — the provider-specific resource identifier (e.g., EC2 instance ID, S3 bucket name, IAM role name)

Examples:

```bash
terraform import aws_instance.myvm i-0b9be609418aa0609
terraform import aws_s3_bucket.data my-data-bucket
terraform import aws_iam_role.app_role my-app-role
terraform import aws_vpc.main vpc-0127895db175d45ff
```

## Step-by-Step: Importing a Single Resource

### 1. Write a Minimal Resource Block

The import command requires a matching resource block in your configuration. It doesn't need to be complete — just enough to satisfy required arguments:

```hcl
resource "aws_instance" "myvm" {
  ami           = "unknown"    # Will fix after import
  instance_type = "unknown"    # Will fix after import
}
```

### 2. Run the Import

```bash
terraform import aws_instance.myvm i-0b9be609418aa0609
```

Output:

```
aws_instance.myvm: Importing from ID "i-0b9be609418aa0609"...
aws_instance.myvm: Import prepared!
  Prepared aws_instance for import
aws_instance.myvm: Refreshing state... [id=i-0b9be609418aa0609]

Import successful!
```

The state file now contains all attributes of the imported resource.

### 3. Run Plan and Fix Replacements First

```bash
terraform plan
```

The plan will likely show `1 to add, 0 to change, 1 to destroy` — meaning Terraform wants to replace the resource because critical attributes (like `ami`) don't match.

Fix the attributes that cause replacement first (these are marked with `forces replacement` in the plan output):

```hcl
resource "aws_instance" "myvm" {
  ami           = "ami-00f22f6155d6d92c5"  # From plan output
  instance_type = "unknown"                 # Still wrong, but won't cause replacement
}
```

Run plan again — it should now show `0 to add, 1 to change, 0 to destroy`.

### 4. Fix Remaining Differences

Update remaining attributes until the plan is clean:

```hcl
resource "aws_instance" "myvm" {
  ami           = "ami-00f22f6155d6d92c5"
  instance_type = "t2.micro"

  tags = {
    Name = "MyVM"
  }
}
```

```bash
terraform plan
# No changes. Your infrastructure matches the configuration.
```

## Importing Into Modules

Prefix the resource address with `module.<module_name>`:

```bash
terraform import module.vpc.aws_vpc.this vpc-0127895db175d45ff
terraform import module.vpc.aws_subnet.public[0] subnet-abc123
terraform import module.vpc.aws_subnet.public[1] subnet-def456
```

After importing one resource from a module, run `terraform plan` to see how many remain:

```
Plan: 28 to add, 0 to change, 0 to destroy.
```

This tells you 28 more resources need importing for a fully consistent state.

## Importing With count and for_each

### count (indexed)

```bash
terraform import 'aws_instance.workers[0]' i-1234567890abcdef0
terraform import 'aws_instance.workers[1]' i-0987654321fedcba0
```

### for_each (keyed)

Escape the quotes when using string keys:

```bash
terraform import 'aws_iam_role.roles["admin"]' admin-role
terraform import 'aws_iam_role.roles["readonly"]' readonly-role
```

Example configuration with `for_each`:

```hcl
locals {
  roles = ["import_role1", "import_role2"]
}

resource "aws_iam_role" "import_roles" {
  for_each = toset(local.roles)
  name     = each.value

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}
```

Import:

```bash
terraform import 'aws_iam_role.import_roles["import_role1"]' import_role1
terraform import 'aws_iam_role.import_roles["import_role2"]' import_role2
```

> **Tip:** Prefer `for_each` with explicit map keys over `count`, since list reordering silently changes indexes and can cause drift.

## Import Blocks (Terraform 1.5+)

Starting with Terraform 1.5, you can declare imports in HCL rather than using the CLI command. This makes imports plannable and reviewable like any other change.

### Define Import Blocks

```hcl
import {
  id = "import-bucket-tf15"
  to = aws_s3_bucket.this
}

import {
  id = "import-bucket-tf15-2"
  to = aws_s3_bucket.this2
}
```

### Generate Configuration Automatically

```bash
terraform plan -generate-config-out=generated_resources.tf
```

Output:

```
Plan: 2 to import, 0 to add, 0 to change, 0 to destroy.

Terraform has generated configuration and written it to generated_resources.tf.
Please review the configuration and edit it as necessary before adding it to version control.
```

The generated file:

```hcl
# __generated__ by Terraform from "import-bucket-tf15"
resource "aws_s3_bucket" "this" {
  bucket              = "import-bucket-tf15"
  bucket_prefix       = null
  force_destroy       = null
  object_lock_enabled = false
  tags                = {}
  tags_all            = {}
}

# __generated__ by Terraform from "import-bucket-tf15-2"
resource "aws_s3_bucket" "this2" {
  bucket              = "import-bucket-tf15-2"
  bucket_prefix       = null
  force_destroy       = null
  object_lock_enabled = false
  tags                = {}
  tags_all            = {}
}
```

### Apply the Import

```bash
terraform apply
# Apply complete! Resources: 2 imported, 0 added, 0 changed, 0 destroyed.
```

After a successful apply, you can remove the `import {}` blocks from your configuration — they're no longer needed.

## Bulk Import Script

For complex deployments with many resources:

```bash
#!/bin/bash
# import_resources.sh
# Format: resource_address,resource_id

while IFS=, read -r address id; do
  echo "Importing $address with ID $id"
  terraform import "$address" "$id"
done < import_list.csv
```

Example `import_list.csv`:

```csv
aws_instance.web,i-1234567890abcdef0
aws_instance.db,i-0987654321fedcba0
aws_security_group.web,sg-abc123
aws_s3_bucket.logs,my-logs-bucket
```

## Common Resource Import IDs

| Resource | ID Format | Example |
|----------|-----------|---------|
| `aws_instance` | Instance ID | `i-1234567890abcdef0` |
| `aws_s3_bucket` | Bucket name | `my-bucket` |
| `aws_iam_role` | Role name | `my-role` |
| `aws_vpc` | VPC ID | `vpc-abc123` |
| `aws_subnet` | Subnet ID | `subnet-abc123` |
| `aws_security_group` | Security Group ID | `sg-abc123` |
| `aws_db_instance` | DB identifier | `my-database` |
| `aws_route53_zone` | Zone ID | `Z1234567890ABC` |
| `aws_lambda_function` | Function name | `my-function` |
| `aws_ecs_cluster` | Cluster ARN | `arn:aws:ecs:...` |

> **Tip:** Check the provider documentation for the specific import ID format — it varies per resource type.

## terraform import vs terraform state mv

| Command | Purpose |
|---------|---------|
| `terraform import` | Bring an existing resource (not in state) under Terraform management |
| `terraform state mv` | Move/rename a resource already managed by Terraform |

Use `import` when the resource exists in the cloud but not in state. Use `state mv` when you're refactoring code (renaming resources, moving into modules).

## terraform import vs Data Sources

| | `terraform import` | `data` source |
|---|---|---|
| Manages lifecycle | Yes (create, update, destroy) | No (read-only) |
| Modifies state | Yes | No |
| Can change the resource | Yes | No |
| Use case | Take ownership of existing resources | Reference existing resources without managing them |

## Use Cases

- **Adopting Terraform** — bring manually created ("ClickOps") infrastructure under IaC management
- **Splitting state files** — move resources between state files when refactoring
- **Disaster recovery** — rebuild a lost or corrupted state file
- **Phased adoption** — import resources incrementally as your team learns Terraform

## Best Practices

1. **Write configuration before importing** — have a resource block ready, even with placeholder values
2. **Use version control** — commit your state (or use a remote backend with versioning) before importing
3. **Fix replacements first** — after import, prioritize attributes that cause `forces replacement`
4. **Iterate with `terraform plan`** — keep adjusting config until the plan shows no changes
5. **Import as-is first** — match the existing resource exactly, then refactor in later commits
6. **Use import blocks for new projects** — the declarative `import {}` syntax (Terraform 1.5+) is more reviewable than CLI commands
7. **Prefer `for_each` over `count`** — stable keys prevent index drift when importing collections
8. **Check provider docs** — not all resources support import, and ID formats vary

## Post-Import Checklist

```bash
# 1. Import the resource
terraform import aws_instance.myvm i-0b9be609418aa0609

# 2. Pull latest state attributes
terraform plan -refresh-only

# 3. Iterate until clean
terraform plan
# Fix config → repeat until "No changes"

# 4. Verify
terraform state show aws_instance.myvm

# 5. Commit
git add -A && git commit -m "Import aws_instance.myvm into Terraform"
```
