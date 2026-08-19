# Terraform Variables: Declaration, Validation, and Usage

Complete reference for declaring, validating, assigning, and using variables in Terraform configurations.

## Variable Declaration Syntax

### Basic Declaration

```hcl
variable "variable_name" {
  description = "Description of the variable"
  type        = string
  default     = "default_value"
  sensitive   = false
  nullable    = true
}
```

### With Validation

```hcl
variable "environment" {
  description = "Environment name"
  type        = string

  validation {
    condition     = contains(["dev", "test", "stage", "prod"], var.environment)
    error_message = "Environment must be one of: dev, test, stage, prod."
  }
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"

  validation {
    condition     = can(regex("^[a-z][0-9]+\\.", var.instance_type))
    error_message = "Instance type must be a valid EC2 instance type."
  }
}
```

### With Optional Attributes

```hcl
variable "complex_config" {
  description = <<-EOD
    Complex configuration object for application deployment.

    Required fields:
    - name: Application name (string)
    - version: Application version (string)
    - replicas: Number of replicas (number, 1-10)

    Optional fields:
    - resources: Resource requirements
    - health_check: Health check configuration
  EOD

  type = object({
    name     = string
    version  = string
    replicas = number

    resources = optional(object({
      cpu    = string
      memory = string
    }), {
      cpu    = "100m"
      memory = "128Mi"
    })

    health_check = optional(object({
      path          = string
      port          = number
      initial_delay = number
      timeout       = number
    }), null)
  })
}
```

## Variable Types

### String

```hcl
variable "instance_name" {
  description = "Name of the EC2 instance"
  type        = string
  default     = "my-instance"
}

# Usage
resource "aws_instance" "example" {
  tags = {
    Name = var.instance_name
  }
}
```

### Number

```hcl
variable "instance_count" {
  description = "Number of instances to create"
  type        = number
  default     = 3

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}

# Usage
resource "aws_instance" "example" {
  count = var.instance_count
}
```

### Bool

```hcl
variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = true
}

# Usage
resource "aws_instance" "example" {
  monitoring = var.enable_monitoring
}
```

### List

```hcl
# List of strings
variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["us-west-2a", "us-west-2b", "us-west-2c"]
}

# List of numbers
variable "allowed_ports" {
  description = "List of allowed ports"
  type        = list(number)
  default     = [80, 443, 22]
}

# Usage
resource "aws_security_group_rule" "ingress" {
  count     = length(var.allowed_ports)
  from_port = var.allowed_ports[count.index]
  to_port   = var.allowed_ports[count.index]
}
```

### Set

```hcl
variable "unique_tags" {
  description = "Set of unique tags"
  type        = set(string)
  default     = ["web", "production", "critical"]
}

# Usage
locals {
  tag_map = { for tag in var.unique_tags : tag => true }
}
```

### Map

```hcl
# Map of strings
variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default = {
    Environment = "production"
    Owner       = "devops-team"
    Project     = "web-app"
  }
}

# Map of numbers
variable "instance_sizes" {
  description = "Instance sizes by environment"
  type        = map(number)
  default = {
    dev  = 1
    test = 2
    prod = 5
  }
}

# Usage
resource "aws_instance" "example" {
  tags = var.tags
}
```

### Tuple

```hcl
variable "server_config" {
  description = "Server configuration tuple"
  type        = tuple([string, number, bool])
  default     = ["t3.micro", 20, true]
}

# Usage
resource "aws_instance" "example" {
  instance_type = var.server_config[0]

  root_block_device {
    volume_size = var.server_config[1]
    encrypted   = var.server_config[2]
  }
}
```

### Object

```hcl
variable "database_config" {
  description = "Database configuration"
  type = object({
    engine                  = string
    engine_version          = string
    instance_class          = string
    allocated_storage       = number
    encrypted               = bool
    backup_retention_period = number
  })

  default = {
    engine                  = "mysql"
    engine_version          = "8.0"
    instance_class          = "db.t3.micro"
    allocated_storage       = 20
    encrypted               = true
    backup_retention_period = 7
  }
}

# Usage
resource "aws_db_instance" "example" {
  engine                  = var.database_config.engine
  engine_version          = var.database_config.engine_version
  instance_class          = var.database_config.instance_class
  allocated_storage       = var.database_config.allocated_storage
  storage_encrypted       = var.database_config.encrypted
  backup_retention_period = var.database_config.backup_retention_period
}
```

### Any (Dynamic)

```hcl
variable "flexible_config" {
  description = "Flexible configuration that can accept any type"
  type        = any
  default = {
    name    = "example"
    count   = 3
    enabled = true
  }
}

# Usage with type checking
locals {
  config_name  = try(var.flexible_config.name, "default")
  config_count = try(var.flexible_config.count, 1)
}
```

## Variable Assignment Methods

### terraform.tfvars

```hcl
# terraform.tfvars
instance_name     = "production-web-server"
instance_count    = 5
enable_monitoring = true

tags = {
  Environment = "production"
  Team        = "backend"
}

availability_zones = ["us-west-2a", "us-west-2b"]
```

### terraform.tfvars.json

```json
{
  "instance_name": "production-web-server",
  "instance_count": 5,
  "enable_monitoring": true,
  "tags": {
    "Environment": "production",
    "Team": "backend"
  }
}
```

### Environment Variables

```bash
export TF_VAR_instance_name="staging-server"
export TF_VAR_instance_count=3
export TF_VAR_enable_monitoring=false
```

### Command Line

```bash
terraform apply -var="instance_name=dev-server" -var="instance_count=1"

# Using variable files
terraform apply -var-file="prod.tfvars"
terraform apply -var-file="secrets.tfvars.json"
```

### Auto-Loaded Files (in order)

```
terraform.tfvars
terraform.tfvars.json
*.auto.tfvars        (alphabetical)
*.auto.tfvars.json   (alphabetical)
```

## Variable Precedence (Highest to Lowest)

| Priority | Source |
|----------|--------|
| 1 (Highest) | Command-line `-var` and `-var-file` flags |
| 2 | `*.auto.tfvars` and `*.auto.tfvars.json` (alphabetical order) |
| 3 | `terraform.tfvars.json` |
| 4 | `terraform.tfvars` |
| 5 | Environment variables (`TF_VAR_*`) |
| 6 (Lowest) | Default values in variable declarations |

## Variable Usage Patterns

### Interpolation

```hcl
# String interpolation
resource "aws_instance" "example" {
  tags = {
    Name = "${var.project_name}-${var.environment}-instance"
  }
}

# Direct reference (no interpolation needed for standalone values)
resource "aws_instance" "example" {
  instance_type = var.instance_type
}
```

### Conditional Expressions

```hcl
resource "aws_instance" "example" {
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"

  tags = merge(
    var.common_tags,
    var.environment == "prod" ? var.production_tags : {}
  )
}
```

### For Expressions

```hcl
# Transform list
locals {
  uppercase_zones = [for zone in var.availability_zones : upper(zone)]
}

# Transform map
locals {
  environment_tags = {
    for key, value in var.tags : key => "${var.environment}-${value}"
  }
}

# Conditional for expression
locals {
  production_zones = [
    for zone in var.availability_zones : zone
    if var.environment == "prod"
  ]
}
```

### Dynamic Blocks

```hcl
variable "ingress_rules" {
  description = "List of ingress rules"
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))

  default = [
    {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]
}

resource "aws_security_group" "example" {
  name = var.sg_name

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

## Built-in Functions for Variables

### Type Functions

```hcl
locals {
  # Type conversion
  string_count = tostring(var.instance_count)
  number_port  = tonumber(var.port_string)
  bool_enabled = tobool(var.enabled_string)

  # Type checking
  is_string = can(tostring(var.flexible_input))
  is_number = can(tonumber(var.flexible_input))
  is_list   = can(tolist(var.flexible_input))
}
```

### Collection Functions

```hcl
locals {
  # Length
  zone_count = length(var.availability_zones)
  tag_count  = length(var.tags)

  # Contains
  has_prod = contains(var.environments, "prod")

  # Index and slice
  first_zone      = var.availability_zones[0]
  first_two_zones = slice(var.availability_zones, 0, 2)

  # Keys and values
  tag_keys   = keys(var.tags)
  tag_values = values(var.tags)

  # Merge
  all_tags = merge(var.common_tags, var.specific_tags)
}
```

### String Functions

```hcl
locals {
  # String manipulation
  upper_name = upper(var.project_name)
  lower_name = lower(var.project_name)
  title_name = title(var.project_name)

  # String operations
  trimmed_name  = trimspace(var.instance_name)
  replaced_name = replace(var.instance_name, "-", "_")

  # Formatting
  formatted_name = format("%s-%s-%03d", var.project, var.environment, var.instance_number)

  # Joining and splitting
  joined_zones = join(",", var.availability_zones)
  split_string = split(",", var.comma_separated_values)
}
```

### Error Handling with try()

```hcl
locals {
  # Safe access to potentially missing values
  db_port = try(var.database_config.port, 3306)

  # Multiple fallbacks
  instance_type = try(
    var.custom_instance_type,
    var.environment_defaults[var.environment].instance_type,
    "t3.micro"
  )
}
```

## Complex Examples

### Multi-Environment Configuration

```hcl
variable "environments" {
  description = "Environment configurations"
  type = map(object({
    instance_type = string
    min_size      = number
    max_size      = number
    desired_size  = number

    tags = map(string)

    database = object({
      instance_class = string
      storage_size   = number
      backup_days    = number
    })
  }))

  default = {
    dev = {
      instance_type = "t3.micro"
      min_size      = 1
      max_size      = 2
      desired_size  = 1

      tags = {
        Environment = "development"
        CostCenter  = "engineering"
      }

      database = {
        instance_class = "db.t3.micro"
        storage_size   = 20
        backup_days    = 3
      }
    }

    prod = {
      instance_type = "t3.large"
      min_size      = 3
      max_size      = 10
      desired_size  = 5

      tags = {
        Environment = "production"
        CostCenter  = "operations"
      }

      database = {
        instance_class = "db.t3.large"
        storage_size   = 100
        backup_days    = 30
      }
    }
  }
}

# Usage
variable "current_environment" {
  description = "Current environment"
  type        = string
  default     = "dev"
}

locals {
  env_config = var.environments[var.current_environment]
}

resource "aws_launch_template" "app" {
  instance_type = local.env_config.instance_type

  tag_specifications {
    resource_type = "instance"
    tags = merge(
      local.env_config.tags,
      { Name = "${var.project_name}-${var.current_environment}" }
    )
  }
}
```

### Feature Flags

```hcl
variable "feature_flags" {
  description = "Feature flags configuration"
  type = object({
    monitoring = object({
      enabled            = bool
      detailed_metrics   = bool
      log_retention_days = number
    })

    security = object({
      encryption_at_rest    = bool
      encryption_in_transit = bool
      vpc_flow_logs         = bool
      access_logging        = bool
    })

    backup = object({
      enabled        = bool
      retention_days = number
      cross_region   = bool
      point_in_time  = bool
    })

    scaling = object({
      auto_scaling        = bool
      target_cpu          = number
      scale_up_cooldown   = number
      scale_down_cooldown = number
    })
  })

  default = {
    monitoring = {
      enabled            = true
      detailed_metrics   = false
      log_retention_days = 30
    }

    security = {
      encryption_at_rest    = true
      encryption_in_transit = true
      vpc_flow_logs         = false
      access_logging        = true
    }

    backup = {
      enabled        = true
      retention_days = 7
      cross_region   = false
      point_in_time  = true
    }

    scaling = {
      auto_scaling        = true
      target_cpu          = 70
      scale_up_cooldown   = 300
      scale_down_cooldown = 300
    }
  }
}

# Usage with conditional resources
resource "aws_cloudwatch_log_group" "app_logs" {
  count             = var.feature_flags.monitoring.enabled ? 1 : 0
  retention_in_days = var.feature_flags.monitoring.log_retention_days
}

resource "aws_kms_key" "encryption" {
  count = var.feature_flags.security.encryption_at_rest ? 1 : 0
}
```

### Network Configuration with Validation

```hcl
variable "network_config" {
  description = "Network configuration"
  type = object({
    vpc_cidr = string

    public_subnets = list(object({
      cidr = string
      az   = string
      name = string
    }))

    private_subnets = list(object({
      cidr = string
      az   = string
      name = string
    }))

    enable_nat_gateway   = bool
    enable_vpn_gateway   = bool
    enable_dns_hostnames = bool
    enable_dns_support   = bool
  })

  validation {
    condition     = can(cidrhost(var.network_config.vpc_cidr, 0))
    error_message = "VPC CIDR must be a valid CIDR block."
  }

  validation {
    condition     = length(var.network_config.public_subnets) >= 1
    error_message = "At least one public subnet must be specified."
  }

  validation {
    condition = alltrue([
      for subnet in var.network_config.public_subnets :
      can(cidrhost(subnet.cidr, 0))
    ])
    error_message = "All subnet CIDRs must be valid CIDR blocks."
  }
}
```

## Using Locals for Computed Values

```hcl
locals {
  # Computed from variables
  full_name = "${var.project_name}-${var.environment}"

  # Conditional logic
  instance_count = var.environment == "prod" ? var.prod_instance_count : 1

  # Complex transformations
  subnet_cidrs = [
    for i, subnet in var.subnet_configs :
    cidrsubnet(var.vpc_cidr, 8, i + 1)
  ]
}
```

## Sensitive Variables

```hcl
variable "database_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}

variable "api_keys" {
  description = "API keys for external services"
  type        = map(string)
  sensitive   = true
  default     = {}
}
```

Sensitive variables:
- Are redacted from `terraform plan` and `terraform apply` output
- Are still stored in state (encrypt your state backend)
- Cannot be used in `for_each` keys or resource names

## Best Practices

1. **Always add `description`** — documents intent for teammates
2. **Use `type` constraints** — catches errors early
3. **Add `validation` blocks** — provide clear error messages for invalid input
4. **Use `sensitive = true`** — for passwords, tokens, and keys
5. **Prefer `object` over `map(any)`** — explicit structure is safer
6. **Use `optional()` for object attributes** — reduces required boilerplate in tfvars
7. **Use `locals` for computed values** — keep variable declarations simple
8. **Use `try()` for safe access** — handle missing or optional nested attributes

### Naming Conventions

```hcl
# Good — descriptive, consistent, uses underscores
variable "vpc_cidr_block" { type = string }
variable "enable_monitoring" { type = bool }
variable "allowed_ingress_ports" { type = list(number) }
variable "instance_count" { type = number }

# Bad — abbreviations, unclear, inconsistent
variable "vpc_cb" { type = string }       # Unclear abbreviation
variable "flag1" { type = bool }          # Not descriptive
variable "Ports" { type = list(number) }  # Uppercase
```

### Bool Negation Pattern

```hcl
variable "public_access" {
  type        = bool
  description = "Allow public access to resources"
  default     = false
}

resource "aws_s3_bucket_public_access_block" "example" {
  bucket = aws_s3_bucket.example.id

  block_public_acls       = !var.public_access
  block_public_policy     = !var.public_access
  ignore_public_acls      = !var.public_access
  restrict_public_buckets = !var.public_access
}
# Result: public_access=false → all blocks enabled (secure default)
#         public_access=true  → all blocks disabled
```

### Cross-Field Validation

```hcl
variable "scaling_config" {
  type = object({
    min_size         = number
    max_size         = number
    desired_capacity = number
  })

  validation {
    condition = (
      var.scaling_config.min_size <= var.scaling_config.desired_capacity &&
      var.scaling_config.desired_capacity <= var.scaling_config.max_size
    )
    error_message = "Must satisfy: min_size <= desired_capacity <= max_size."
  }
}
```

### Default Value Patterns

```hcl
# Use null for optional computed values
variable "custom_domain" {
  type    = string
  default = null
}

locals {
  domain_name = var.custom_domain != null ? var.custom_domain : "${random_id.suffix.hex}.example.com"
}

# Use empty collections for optional lists/maps
variable "additional_tags" {
  type    = map(string)
  default = {}
}

variable "extra_security_groups" {
  type    = list(string)
  default = []
}
```
