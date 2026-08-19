# Terraform Conditional Expressions

Conditional expressions in Terraform let you choose between values, create or skip resources, toggle dynamic blocks, and validate inputs — all using HCL's ternary syntax and built-in functions.

## Syntax

```hcl
condition ? true_val : false_val
```

## Basic Conditional Expression

Pick a value based on a simple equality check. The most common pattern — one condition, two possible values.

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment (dev, staging, production)"
  default     = "dev"
}

variable "ami_id" {
  type        = string
  description = "AMI ID to use for the instance"
  default     = "ami-0c55b159cbfafe1f0"
}

resource "aws_instance" "example" {
  ami           = var.ami_id
  instance_type = var.environment == "production" ? "t3.large" : "t3.micro"

  tags = {
    Name = "web-${var.environment}"
  }
}
# Result: production gets t3.large, everything else gets t3.micro
```

## Conditional Resource Creation (count)

Use `count` to create a resource only when a condition is true. When `count = 0`, the resource is not created at all. When `count = 1`, exactly one instance is created.

```hcl
variable "enable_logging" {
  type    = bool
  default = false
}

variable "logging_bucket_name" {
  type        = string
  default     = "my-logs-bucket"
  description = "Name of the S3 bucket for logs"
}

variable "logging_retention_days" {
  type        = number
  default     = 90
  description = "Number of days to retain logs before expiration"
}

resource "aws_s3_bucket" "logs" {
  count  = var.enable_logging ? 1 : 0
  bucket = var.logging_bucket_name

  tags = {
    Purpose   = "logging"
    Retention = "${var.logging_retention_days} days"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "logs" {
  count  = var.enable_logging ? 1 : 0
  bucket = aws_s3_bucket.logs[0].id

  rule {
    id     = "expire-old-logs"
    status = "Enabled"

    expiration {
      days = var.logging_retention_days
    }
  }
}
```

Referencing a `count`-based resource requires index notation since it becomes a list:

```hcl
# Reference the resource (only safe when count = 1)
output "logs_bucket_arn" {
  value = var.enable_logging ? aws_s3_bucket.logs[0].arn : null
}

# In other resources
resource "aws_s3_bucket_policy" "logs" {
  count  = var.enable_logging ? 1 : 0
  bucket = aws_s3_bucket.logs[0].id
  policy = data.aws_iam_policy_document.logs.json
}
```

The downside of `count`: if you later change the condition, Terraform treats it as index-based — adding or removing element `[0]` forces a destroy/recreate. For resources where this matters, prefer `for_each` (see below).

## Conditional Values in Lists

Return different lists depending on a condition. Useful for security groups, subnets, or any list-type argument that varies by environment.

```hcl
variable "environment_type" {
  type        = string
  description = "Environment type: production or development"
  default     = "development"
}

resource "aws_security_group" "prod" {
  name = "prod-sg"
}

resource "aws_security_group" "dev" {
  name = "dev-sg"
}

resource "aws_security_group" "common" {
  name = "common-sg"
}

locals {
  security_groups = var.environment_type == "production" ? [
    aws_security_group.prod.id,
    aws_security_group.common.id
  ] : [
    aws_security_group.dev.id
  ]
}

# Use in a resource
resource "aws_instance" "app" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.micro"
  vpc_security_group_ids = local.security_groups
}
# Result: production gets prod+common SGs, dev gets only dev SG
```

## Multiple Conditions (Nested Ternary)

Chain ternaries for more than two outcomes. Wrap inner expressions in parentheses for readability.

```hcl
variable "environment" {
  type        = string
  description = "One of: production, staging, dev"
  default     = "dev"
}

locals {
  instance_type = var.environment == "production" ? "t3.large" : (
    var.environment == "staging" ? "t3.medium" : "t3.micro"
  )
}

# Result:
#   production → t3.large
#   staging    → t3.medium
#   anything else → t3.micro
```

## Null/Empty Handling

Set an attribute to `null` to omit it, or use an empty map for tags. When Terraform sees `null`, it treats the argument as if it wasn't set at all.

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

variable "key_name" {
  type        = string
  default     = ""
  description = "SSH key pair name. Leave empty to skip key assignment."
}

resource "aws_instance" "conditional_example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  # null means "don't set this argument at all"
  key_name = var.key_name != "" ? var.key_name : null

  # Empty map means no tags; non-empty map adds them
  tags = var.environment == "production" ? {
    Environment = "prod"
    Backup      = "true"
  } : {}
}
# Result: if key_name is empty, the instance launches without a key pair
```

## Boolean Conditions

Use a `bool` variable directly as the condition. No need for `== true` — the variable itself is the condition.

```hcl
variable "enable_monitoring" {
  type    = bool
  default = false
}

resource "aws_cloudwatch_metric_alarm" "cpu" {
  count               = var.enable_monitoring ? 1 : 0
  alarm_name          = "cpu-utilization"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = "120"
  statistic           = "Average"
  threshold           = "80"
  alarm_description   = "This metric monitors ec2 cpu utilization"
}
```

## Complex Conditions with Functions

Use `length()`, `contains()`, and other functions in conditions.

```hcl
variable "custom_subnets" {
  type    = list(string)
  default = []
}

data "aws_subnets" "default" {
  filter {
    name   = "default-for-az"
    values = ["true"]
  }
}

locals {
  subnet_ids = length(var.custom_subnets) > 0 ? var.custom_subnets : data.aws_subnets.default.ids
}
```

## Conditional Dynamic Blocks

Use `dynamic` with a conditional `for_each` to include or exclude blocks.

```hcl
variable "enable_ingress_rules" {
  type    = bool
  default = true
}

variable "ingress_ports" {
  type    = list(number)
  default = [80, 443]
}

resource "aws_security_group" "conditional_sg" {
  name_prefix = "conditional-sg"

  dynamic "ingress" {
    for_each = var.enable_ingress_rules ? var.ingress_ports : []
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

## Conditional Locals (Environment Config Maps)

Return an entire object of settings based on environment. Keeps resource blocks clean by referencing `local.config.*`.

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

locals {
  config = var.environment == "production" ? {
    min_size         = 3
    max_size         = 10
    desired_capacity = 5
    instance_type    = "t3.large"
  } : {
    min_size         = 1
    max_size         = 3
    desired_capacity = 1
    instance_type    = "t3.micro"
  }
}

# Use in resources — no ternaries needed in the resource block
resource "aws_autoscaling_group" "app" {
  min_size         = local.config.min_size
  max_size         = local.config.max_size
  desired_capacity = local.config.desired_capacity
  # ...
}
# Result: one local controls all environment-specific values
```

## Conditional String Interpolation

Build different string values per environment using interpolation inside a ternary.

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

resource "random_id" "bucket_suffix" {
  byte_length = 4
}

locals {
  bucket_name = (
    var.environment == "production"
    ? "prod-${random_id.bucket_suffix.hex}"
    : "dev-${random_id.bucket_suffix.hex}"
  )
}

# Result: "prod-a1b2c3d4" or "dev-a1b2c3d4"
```

## Count with Multiple Conditions

Combine `&&`, `||`, and `contains()` for complex logic.

```hcl
variable "create_database" {
  type    = bool
  default = false
}

variable "database_required_envs" {
  type    = list(string)
  default = ["production", "staging"]
}

resource "aws_db_instance" "conditional_db" {
  count = var.create_database && contains(var.database_required_envs, var.environment) ? 1 : 0

  engine         = "mysql"
  engine_version = "8.0"
  instance_class = var.environment == "production" ? "db.t3.medium" : "db.t3.micro"

  allocated_storage = var.environment == "production" ? 100 : 20
  storage_encrypted = var.environment == "production" ? true : false
}
```

## Conditional Resource Creation with for_each

Avoids `[0]` indexing that `count` requires. Resources are keyed by name, so adding/removing doesn't force recreation of unrelated items.

```hcl
variable "enabled" {
  type        = bool
  default     = true
  description = "Whether to create the resource"
}

resource "null_resource" "maybe" {
  for_each = var.enabled ? toset(["enabled"]) : toset([])
}

# Referencing — use the key name, not an index:
# null_resource.maybe["enabled"].id
```

### Create N Instances Only if N > 0

```hcl
variable "replicas" {
  type        = number
  default     = 0
  description = "Number of replicas to create. Set 0 to skip."
}

resource "null_resource" "replica" {
  for_each = var.replicas > 0 ? toset([for i in range(var.replicas) : tostring(i)]) : toset([])
  # each.key is "0".."N-1"
}
# Result: replicas=3 creates null_resource.replica["0"], ["1"], ["2"]
#         replicas=0 creates nothing
```

## Module Conditionally Created

Entire modules can be toggled with `count`. Reference with `module.thing[0]` when created.

```hcl
variable "create_thing" {
  type        = bool
  default     = false
  description = "Whether to deploy the thing module"
}

module "thing" {
  source = "./modules/thing"
  count  = var.create_thing ? 1 : 0
}

# Reference (only safe when count = 1)
output "thing_id" {
  value = var.create_thing ? module.thing[0].id : null
}
# Result: module is either fully deployed or completely absent
```

## Data Source Conditionally Read

Only query a data source when you actually need the result. Avoids unnecessary API calls.

```hcl
variable "lookup_ami" {
  type        = bool
  default     = true
  description = "Whether to look up the latest Ubuntu AMI"
}

variable "custom_ami" {
  type        = string
  default     = ""
  description = "Provide a specific AMI ID to skip the lookup"
}

data "aws_ami" "ubuntu" {
  count       = var.lookup_ami ? 1 : 0
  most_recent = true
  owners      = ["099720109477"] # Canonical
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

locals {
  ami_id = var.lookup_ami ? data.aws_ami.ubuntu[0].id : var.custom_ami
}
# Result: either dynamically finds the latest AMI or uses the one you provided
```

## Prefer Non-Null Using coalesce

`coalesce()` returns the first non-empty, non-null argument. Simpler than a ternary for fallback values.

```hcl
variable "custom_name" {
  type        = string
  default     = ""
  description = "Custom display name. Falls back to 'default-name' if empty."
}

locals {
  display_name = coalesce(var.custom_name, "default-name")
}
# Result: "my-app" if custom_name="my-app", "default-name" if custom_name=""
```

## Conditional Dynamic Nested Blocks

Include or exclude an entire nested block (like `ebs_block_device`) by toggling `for_each` between a single-element list and an empty list.

```hcl
variable "ami" {
  type    = string
  default = "ami-0c55b159cbfafe1f0"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "enable_ebs" {
  type        = bool
  default     = false
  description = "Attach an extra EBS volume to the instance"
}

variable "ebs_device_name" {
  type    = string
  default = "/dev/xvdf"
}

variable "ebs_volume_size" {
  type    = number
  default = 50
}

resource "aws_instance" "app_ebs" {
  ami           = var.ami
  instance_type = var.instance_type

  dynamic "ebs_block_device" {
    for_each = var.enable_ebs ? [1] : []
    content {
      device_name = var.ebs_device_name
      volume_size = var.ebs_volume_size
    }
  }
}
# Result: enable_ebs=true → instance gets an extra 50GB volume
#         enable_ebs=false → no extra volume attached
```

## Conditionally Merge Maps

Use `merge()` with a conditional empty map to optionally add extra key-value pairs (common for tags).

```hcl
variable "bucket_name" {
  type    = string
  default = "my-app-data"
}

variable "extra_tags" {
  type        = map(string)
  default     = null
  description = "Additional tags to apply. Set null to skip."
}

resource "aws_s3_bucket" "b" {
  bucket = var.bucket_name

  tags = merge(
    { Name = var.bucket_name },
    var.extra_tags != null ? var.extra_tags : {}
  )
}
# Result: extra_tags = { Team = "backend" } → tags get Name + Team
#         extra_tags = null                 → tags only get Name
```

## Optional File Content (guard with can/try)

Safely handle files that may not exist. `can()` tests if an expression succeeds; `try()` returns a fallback on failure.

```hcl
variable "ami" {
  type    = string
  default = "ami-0c55b159cbfafe1f0"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "user_data_path" {
  type        = string
  default     = "scripts/init.sh"
  description = "Path to user data script. If file doesn't exist, no user data is set."
}

resource "aws_instance" "app_userdata_file" {
  ami           = var.ami
  instance_type = var.instance_type

  # Pattern 1: can() — test then use
  user_data = can(file(var.user_data_path)) ? file(var.user_data_path) : null

  # Pattern 2: try() — attempt with fallback (same result, shorter)
  # user_data = try(file(var.user_data_path), null)
}
# Result: if scripts/init.sh exists → instance gets user_data
#         if file is missing       → user_data is null (skipped)
```

## Conditional Outputs

Output a value only when the resource was actually created. Use `null` to suppress the output otherwise.

```hcl
variable "create_bucket" {
  type        = bool
  default     = true
  description = "Whether to create the S3 bucket"
}

resource "aws_s3_bucket" "main" {
  count  = var.create_bucket ? 1 : 0
  bucket = "my-app-bucket"
}

output "bucket_id" {
  value       = var.create_bucket ? aws_s3_bucket.main[0].id : null
  description = "The S3 bucket ID, or null if bucket was not created"
}

output "bucket_arn" {
  value       = var.create_bucket ? aws_s3_bucket.main[0].arn : null
  description = "The S3 bucket ARN, or null if bucket was not created"
}
# Result: terraform output shows bucket_id = "my-app-bucket" or bucket_id = null
```

## Validations and Preconditions

### Variable Validation

```hcl
variable "env" {
  type = string
  validation {
    condition     = contains(["dev", "stg", "prod"], var.env)
    error_message = "env must be one of dev, stg, prod."
  }
}
```

### Resource Precondition

```hcl
resource "aws_instance" "guarded" {
  ami           = var.ami
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = var.env != "prod" || var.instance_type != "t3.micro"
      error_message = "Do not use t3.micro in prod."
    }
  }
}
```

## one() — Extract Single Element from Conditional Set

`one()` returns the single element from a set or list, or `null` if empty. Useful with `for_each`-based conditional resources to avoid map key lookups.

```hcl
resource "aws_eip" "nat" {
  for_each = var.create_nat ? toset(["this"]) : toset([])
  domain   = "vpc"
}

# one() returns the resource or null — no ["this"] key needed
output "nat_ip" {
  value = one(aws_eip.nat[*].public_ip)
}
```

## optional() — Optional Object Attributes (Terraform 1.3+)

Mark object attributes as optional with default values. Callers can omit fields without triggering errors.

```hcl
variable "database_config" {
  type = object({
    engine         = string
    engine_version = optional(string, "8.0")
    instance_class = optional(string, "db.t3.micro")
    multi_az       = optional(bool, false)
    storage_gb     = optional(number, 20)
  })
}

# Caller can provide only required fields
# database_config = { engine = "mysql" }
# All optional fields get their defaults automatically
```

## for Expression with if Filter

`for` comprehensions support an `if` clause to conditionally include elements. Builds filtered lists or maps without external logic.

```hcl
variable "instances" {
  type = map(object({
    type        = string
    environment = string
  }))
}

# Only include production instances
locals {
  prod_instances = { for k, v in var.instances : k => v if v.environment == "production" }
}

# Filter a list
locals {
  large_instances = [for name, cfg in var.instances : name if cfg.type == "t3.large"]
}

# Transform and filter in one expression
locals {
  prod_names = [for k, v in var.instances : upper(k) if v.environment == "production"]
}
```

## Conditional depends_on (Workaround)

`depends_on` cannot be made conditional directly. The workaround is to use `count` or `for_each` on the dependent resource itself so that when it doesn't exist, the dependency is irrelevant.

```hcl
# You CANNOT do this (invalid HCL):
# depends_on = var.need_dependency ? [aws_iam_policy.dep] : []

# Workaround: make the dependent resource conditional
resource "aws_iam_policy" "dep" {
  count  = var.create_policy ? 1 : 0
  name   = "my-policy"
  policy = data.aws_iam_policy_document.doc.json
}

resource "aws_lambda_function" "fn" {
  count         = var.create_policy ? 1 : 0
  function_name = "my-function"
  role          = aws_iam_role.role.arn
  runtime       = "python3.12"
  handler       = "main.handler"
  filename      = "lambda.zip"

  depends_on = [aws_iam_policy.dep]
  # When count=0 on both, neither is created — dependency is moot
}
```

## Conditional Provider Configuration

You cannot conditionally create providers, and the `provider` meta-argument does not accept expressions. Use separate resource blocks with `count` to route resources to different providers.

```hcl
provider "aws" {
  region = "eu-west-1"
}

provider "aws" {
  alias  = "us"
  region = "us-east-1"
}

variable "region" {
  type        = string
  default     = "eu"
  description = "Which region to deploy to: eu or us"
}

# Pattern: separate resources per provider, toggled with count
resource "aws_s3_bucket" "eu" {
  count  = var.region == "eu" ? 1 : 0
  bucket = "my-bucket-eu"
}

resource "aws_s3_bucket" "us" {
  count    = var.region == "us" ? 1 : 0
  provider = aws.us
  bucket   = "my-bucket-us"
}

# Combine the output
output "bucket_id" {
  value = var.region == "eu" ? aws_s3_bucket.eu[0].id : aws_s3_bucket.us[0].id
}
# Note: provider = var.x ? aws.us : aws is NOT valid HCL
# The provider argument must be a static reference
```

## Tips

- Use `count` for simple on/off; use `for_each` to avoid index-based references
- Use `null` to "unset" optional arguments
- Dynamic blocks include nested blocks conditionally
- `can()`/`try()` help when inputs (like files) may be absent
- Prefer merging maps/lists with a conditional empty value to keep plans clean

## Quick Reference

| Pattern | Example |
|---------|---------|
| Simple ternary | `var.env == "prod" ? "large" : "small"` |
| Create or skip resource | `count = var.enabled ? 1 : 0` |
| Null to omit attribute | `key_name = var.key != "" ? var.key : null` |
| Nested ternary | `var.a ? "x" : (var.b ? "y" : "z")` |
| List conditional | `length(var.list) > 0 ? var.list : default_list` |
| Boolean AND | `var.a && var.b ? 1 : 0` |
| Contains check | `contains(var.envs, var.env) ? 1 : 0` |
| Dynamic block toggle | `for_each = var.enabled ? var.items : []` |
| Conditional map | `var.env == "prod" ? { k = "v" } : {}` |
| for_each on/off | `for_each = var.enabled ? toset(["enabled"]) : toset([])` |
| Coalesce fallback | `coalesce(var.custom_name, "default")` |
| Merge maps | `merge({ Name = "x" }, var.extra != null ? var.extra : {})` |
| Guard with try | `try(file(var.path), null)` |
| Guard with can | `can(file(var.path)) ? file(var.path) : null` |
| Conditional output | `value = var.create ? resource[0].id : null` |
| Precondition | `condition = var.env != "prod" \|\| var.type != "t3.micro"` |
| Variable validation | `condition = contains(["dev","stg","prod"], var.env)` |
| one() extract | `one(aws_eip.nat[*].public_ip)` |
| optional() defaults | `optional(string, "default-value")` |
| for with if filter | `[for k, v in var.map : k if v.env == "prod"]` |
| Conditional provider | Use separate resources with `count` per provider alias |
