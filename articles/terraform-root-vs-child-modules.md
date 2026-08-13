# Terraform Root Module vs Child Modules

Every Terraform configuration has exactly one root module. When that root module calls other modules, those are child modules. Understanding this hierarchy is key to writing maintainable, reusable infrastructure code.

## What Is a Module?

A module is simply a directory containing `.tf` files. That's it. There's no special declaration or file that makes something a module — any directory with Terraform files IS a module.

```
# This is a module:
modules/vpc/
├── main.tf
├── variables.tf
├── outputs.tf

# This is also a module:
./ (your working directory)
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
```

## Root Module vs Child Module

| Aspect | Root Module | Child Module |
|--------|-------------|--------------|
| What is it | The directory where you run `terraform apply` | A module called by another module |
| How many per config | Exactly one | Zero or more |
| Contains state | Yes (owns the state file) | No (state lives in root) |
| Has providers | Yes (providers are configured here) | Inherits from root (usually) |
| Has backend | Yes (`terraform { backend {} }`) | No |
| Runs `terraform init` | Yes | No (initialized by root) |
| Can have `.tfvars` | Yes | No (receives values via `variable` inputs) |
| Typical purpose | Composition — wires modules together | Abstraction — encapsulates a reusable component |

### Visual

```
root module (where you run terraform apply)
├── main.tf          ← calls child modules
├── variables.tf     ← inputs for the root
├── outputs.tf       ← outputs from the root
├── terraform.tfvars ← variable values
├── providers.tf     ← provider configuration
├── backend.tf       ← state storage config
│
├── modules/vpc/     ← child module
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
│
└── modules/ec2/     ← child module
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

## The Root Module

The root module is your entry point. It:
- Configures the backend (where state is stored)
- Configures providers (AWS, Azure, GCP credentials)
- Calls child modules and passes inputs
- Wires module outputs together
- Defines the top-level `terraform apply` behavior

```hcl
# root module: main.tf

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.region
}

# Call child modules
module "vpc" {
  source = "./modules/vpc"

  cidr_block  = var.vpc_cidr
  environment = var.environment
}

module "ec2" {
  source = "./modules/ec2"

  subnet_id       = module.vpc.private_subnet_ids[0]
  security_groups = [module.vpc.default_sg_id]
  instance_type   = var.instance_type
}

# Root-level outputs (exposed from terraform output)
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "instance_ip" {
  value = module.ec2.private_ip
}
```

## Child Modules

A child module is called by the root (or by another child module). It:
- Defines `variable` blocks for its inputs
- Defines `output` blocks for values to return to the caller
- Contains the actual resource definitions
- Does NOT configure providers or backends
- Does NOT read `.tfvars` files

```hcl
# modules/vpc/variables.tf
variable "cidr_block" {
  type        = string
  description = "CIDR block for the VPC"
}

variable "environment" {
  type        = string
  description = "Environment name (dev, staging, prod)"
}
```

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.environment}-private-${count.index + 1}"
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  value       = aws_vpc.this.id
  description = "The ID of the VPC"
}

output "private_subnet_ids" {
  value       = aws_subnet.private[*].id
  description = "List of private subnet IDs"
}

output "default_sg_id" {
  value       = aws_vpc.this.default_security_group_id
  description = "Default security group ID"
}
```

### EC2 Child Module (Complete Example)

```hcl
# modules/ec2/main.tf
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_security_group" "web" {
  name_prefix = "${var.environment}-web-"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.environment}-web-sg"
    Environment = var.environment
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = [aws_security_group.web.id]

  tags = {
    Name        = "${var.environment}-web-server"
    Environment = var.environment
  }
}
```

```hcl
# modules/ec2/variables.tf
variable "instance_type" {
  description = "EC2 instance type"
  type        = string

  validation {
    condition     = can(regex("^t3\\.", var.instance_type))
    error_message = "Instance type must be from t3 family."
  }
}

variable "subnet_id" {
  description = "Subnet ID to launch the instance in"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID for the security group"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}
```

```hcl
# modules/ec2/outputs.tf
output "instance_id" {
  value       = aws_instance.web.id
  description = "The ID of the EC2 instance"
}

output "private_ip" {
  value       = aws_instance.web.private_ip
  description = "Private IP address of the instance"
}

output "security_group_id" {
  value       = aws_security_group.web.id
  description = "ID of the web security group"
}
```

## Calling a Module

```hcl
module "<local_name>" {
  source = "<source_path>"

  # Input variables
  variable_name = value
}
```

The `source` argument tells Terraform where to find the module code:

### Source Types

| Source | Example | Use Case |
|--------|---------|----------|
| Local path | `source = "./modules/vpc"` | Modules within the same repo |
| Terraform Registry | `source = "hashicorp/consul/aws"` | Public reusable modules |
| GitHub | `source = "github.com/org/repo//modules/vpc"` | Private/public GitHub repos |
| Bitbucket | `source = "bitbucket.org/org/repo//modules/vpc"` | Bitbucket repos |
| Generic Git | `source = "git::https://example.com/repo.git//modules/vpc"` | Any Git repo |
| Git with SSH | `source = "git::ssh://git@github.com/org/repo.git//modules/vpc"` | Private repos via SSH |
| S3 bucket | `source = "s3::https://s3-eu-west-1.amazonaws.com/bucket/vpc.zip"` | Module packages in S3 |
| GCS bucket | `source = "gcs::https://www.googleapis.com/storage/v1/bucket/vpc.zip"` | Module packages in GCS |
| HTTP URL | `source = "https://example.com/terraform-modules/vpc.zip"` | Module archive via HTTP |

### Versioning

```hcl
# Registry modules — use version constraint
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"    # >=5.0.0, <6.0.0
}

# Git — use ref (tag, branch, commit)
module "vpc" {
  source = "git::https://github.com/org/modules.git//vpc?ref=v2.1.0"
}

# GitHub shorthand with tag
module "vpc" {
  source = "github.com/org/modules//vpc?ref=v2.1.0"
}
```

## How Data Flows

```
┌─────────────────────────────────────┐
│         ROOT MODULE                 │
│                                     │
│  var.region ──→ provider config     │
│  var.vpc_cidr ──→ module "vpc" {    │
│                     cidr_block = .. │
│                   }                 │
│                                     │
│  module.vpc.vpc_id ──→ output       │
│  module.vpc.subnet_ids ──→ module   │
│                             "ec2"   │
└─────────────────────────────────────┘
         │                    ▲
    inputs (variables)   outputs (return values)
         ▼                    │
┌─────────────────────────────────────┐
│       CHILD MODULE (vpc)            │
│                                     │
│  variable "cidr_block" {}           │
│  resource "aws_vpc" "this" {}       │
│  output "vpc_id" { value = ... }    │
└─────────────────────────────────────┘
```

**Key rules:**
- Parent passes data DOWN via module arguments (mapped to child's `variable` blocks)
- Child passes data UP via `output` blocks (accessed as `module.<name>.<output>`)
- Siblings communicate THROUGH the parent (module A's output → root → module B's input)
- There's no direct reference between sibling modules

## Variable Scoping

Each Terraform module has its own **variable scope**. This is the most important concept to understand:

- Variables defined in the root module are NOT automatically available in child modules
- Child modules must explicitly declare every variable they use
- Values are passed from parent to child through the `module` block arguments

### How Variable Passing Works

```hcl
# Root module — variables.tf
variable "environment" {
  type    = string
  default = "dev"
}

# Root module — main.tf (passes the value to the child)
module "web_server" {
  source      = "./modules/ec2"
  environment = var.environment      # Root var → Child var
}

# Child module — modules/ec2/variables.tf (MUST declare it)
variable "environment" {
  description = "Environment name"
  type        = string
  # No default — receives value from caller
}

# Child module — modules/ec2/main.tf (uses its own variable)
resource "aws_instance" "web" {
  tags = {
    Environment = var.environment    # This is the CHILD's variable
  }
}
```

### Error: Missing Variable Declaration

If you use `var.environment` in a child module without declaring it in that module's `variables.tf`:

```
Error: Reference to undeclared input variable

  on modules/ec2/main.tf line 5, in resource "aws_instance" "web":
   5:     Name = "${var.environment}-web"

An input variable with the name "environment" has not been declared.
This variable must be declared in the child module's variables.tf.
```

### Variable Grouping with Objects

Instead of passing many individual variables, group related values:

```hcl
# Root module — variables.tf
variable "common_tags" {
  description = "Common tags for all resources"
  type = object({
    environment = string
    project     = string
    owner       = string
  })
  default = {
    environment = "dev"
    project     = "my-project"
    owner       = "team-a"
  }
}

# Root module — main.tf
module "vpc" {
  source      = "./modules/vpc"
  common_tags = var.common_tags
}

# Child module — modules/vpc/variables.tf
variable "common_tags" {
  description = "Common tags for all resources"
  type = object({
    environment = string
    project     = string
    owner       = string
  })
}

# Child module — modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags       = var.common_tags
}
```

### Using locals for Derived Values

```hcl
# modules/ec2/main.tf
locals {
  instance_name = "${var.environment}-${var.component}-server"

  default_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    Component   = var.component
  }
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = merge(local.default_tags, var.additional_tags)
}
```

## Providers in Modules

### Default: Inherited from Root

```hcl
# Root module configures the provider
provider "aws" {
  region = "us-east-1"
}

# Child module inherits it automatically
module "vpc" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
}
```

### Explicit Provider Passing

```hcl
# Root module with multiple provider configurations
provider "aws" {
  region = "us-east-1"
  alias  = "east"
}

provider "aws" {
  region = "eu-west-1"
  alias  = "europe"
}

# Pass specific provider to child module
module "vpc_east" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"

  providers = {
    aws = aws.east
  }
}

module "vpc_europe" {
  source     = "./modules/vpc"
  cidr_block = "10.1.0.0/16"

  providers = {
    aws = aws.europe
  }
}
```

### Provider Configuration in Child Module

```hcl
# modules/vpc/main.tf — declare required providers (but DON'T configure them)
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

# This module uses "aws" provider — it inherits the configuration from root
resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
}
```

> **Rule:** Child modules should declare `required_providers` but NOT include `provider` blocks with configuration. Provider configuration belongs in the root module only.

## Module Composition Patterns

### Flat Modules (Simple)

```
.
├── main.tf              # Root — calls modules directly
├── modules/
│   ├── vpc/
│   ├── ec2/
│   └── rds/
```

### Nested Modules (Layered)

```
.
├── main.tf              # Root — calls environment module
├── modules/
│   └── environment/     # Composes lower-level modules
│       ├── main.tf      # Calls vpc, ec2, rds internally
│       ├── modules/
│       │   ├── vpc/
│       │   ├── ec2/
│       │   └── rds/
```

### Registry + Local Modules

```hcl
# Use a public registry module for standard infrastructure
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
  azs  = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# Use a local module for company-specific logic
module "app" {
  source = "./modules/app"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
}
```

## for_each and count with Modules

### Multiple Instances of a Module

```hcl
# Create 3 environments from the same module
module "environment" {
  source   = "./modules/environment"
  for_each = toset(["dev", "staging", "prod"])

  environment = each.key
  vpc_cidr    = var.vpc_cidrs[each.key]
}

# Access outputs
output "vpc_ids" {
  value = { for k, v in module.environment : k => v.vpc_id }
}
```

```hcl
# Using count
module "worker" {
  source = "./modules/worker"
  count  = 3

  name = "worker-${count.index + 1}"
}
```

## Module Dependencies

### Implicit (Automatic)

```hcl
# Terraform auto-detects: ec2 depends on vpc because it uses vpc's output
module "vpc" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
}

module "ec2" {
  source    = "./modules/ec2"
  subnet_id = module.vpc.private_subnet_ids[0]    # ← implicit dependency
}
```

### Explicit (depends_on)

```hcl
# Force ordering when there's no data reference
module "iam" {
  source = "./modules/iam"
}

module "app" {
  source = "./modules/app"

  depends_on = [module.iam]    # Wait for IAM even though we don't reference its outputs
}
```

## Module Best Practices

### Structure

```
modules/
└── vpc/
    ├── main.tf          # Resources
    ├── variables.tf     # Inputs (all variable blocks)
    ├── outputs.tf       # Outputs (all output blocks)
    ├── versions.tf      # required_providers (optional, for clarity)
    ├── locals.tf        # Local values (optional)
    ├── data.tf          # Data sources (optional)
    └── README.md        # Module documentation
```

### Naming Conventions

```hcl
# Module call names — descriptive, lowercase, underscores
module "primary_vpc" { ... }
module "worker_instances" { ... }
module "monitoring_alerts" { ... }

# NOT:
module "vpc1" { ... }
module "myModule" { ... }
```

### Input Validation in Child Modules

```hcl
# modules/vpc/variables.tf
variable "cidr_block" {
  type        = string
  description = "CIDR block for the VPC"

  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
}

variable "environment" {
  type        = string
  description = "Environment name"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

### Output Descriptions

```hcl
# Always include descriptions — they appear in terraform output and docs
output "vpc_id" {
  value       = aws_vpc.this.id
  description = "The ID of the created VPC"
}

output "private_subnet_ids" {
  value       = aws_subnet.private[*].id
  description = "List of private subnet IDs across all availability zones"
}
```

## Commands Related to Modules

```bash
# Download/update modules
terraform init                # Downloads modules on first run
terraform init -upgrade       # Re-download modules (pick up new versions)
terraform get                 # Download modules only (no providers)
terraform get -update         # Force re-download of modules

# Validate and format
terraform validate            # Check configuration validity (including modules)
terraform fmt -recursive      # Format all .tf files including in module subdirectories

# Inspect module state
terraform state list          # Shows module.vpc.aws_vpc.this, etc.
terraform state show module.vpc.aws_vpc.this

# Target a specific module for apply/destroy
terraform apply -target=module.vpc
terraform destroy -target=module.ec2

# Move state between modules (refactoring)
terraform state mv module.old_name module.new_name
terraform state mv aws_vpc.main module.vpc.aws_vpc.this

# Import into a module resource
terraform import module.vpc.aws_vpc.this vpc-12345678

# Plan for a specific module
terraform plan -target=module.vpc
```

## Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Provider config in child module | Child modules should inherit providers | Move `provider` blocks to root only |
| Backend in child module | Only root has state/backend | Remove `backend` from child |
| Hard-coded values in child module | Reduces reusability | Use `variable` blocks for everything configurable |
| No outputs in child module | Caller can't access created resources | Add `output` blocks for IDs, ARNs, etc. |
| No descriptions on variables/outputs | Hard to use without reading code | Always add `description` |
| Relative path going up (`../`) | Fragile, breaks on reorganization | Use absolute module sources or restructure |
| Skipping `required_providers` in child | Version conflicts possible | Declare with version constraint |
| Giant monolithic root module | Unmaintainable | Split into focused child modules |

## Benefits of Using Child Modules

1. **Reusability** — use the same module across different environments and projects
2. **Organization** — keep related resources together (VPC + subnets + route tables)
3. **Abstraction** — hide complexity behind simple input/output interfaces
4. **Testing** — test modules independently with different variable sets
5. **Collaboration** — teams can work on different modules without conflicts
6. **Maintainability** — changes in one module don't affect others
7. **Version control** — pin module versions, upgrade on your schedule
8. **Consistency** — enforce standards (tagging, naming, security) across all environments

## When to Create a Module

**Create a child module when:**
- You have a logical group of resources that belong together (VPC + subnets + route tables)
- The same pattern is used more than once (with different inputs)
- You want to enforce standards (tags, naming, security settings)
- A team owns a component and wants to version it independently

**Keep it in the root when:**
- It's a one-off resource with no reuse
- The configuration is simple (< 5 resources)
- Extracting it would add complexity without benefit

## Quick Reference

```hcl
# Call a local module
module "name" {
  source = "./modules/path"
  input  = value
}

# Call a registry module with version
module "name" {
  source  = "org/module/provider"
  version = "~> 2.0"
  input   = value
}

# Access module output
module.name.output_name

# Module with for_each
module "name" {
  source   = "./modules/path"
  for_each = var.map
  input    = each.value
}

# Pass providers explicitly
module "name" {
  source = "./modules/path"
  providers = {
    aws = aws.alias_name
  }
}

# Depend on another module
module "name" {
  source     = "./modules/path"
  depends_on = [module.other]
}
```
