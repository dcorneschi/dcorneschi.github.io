# Terraform toset(), each.key vs each.value

How `toset()` works, when to use `each.key` vs `each.value` with `for_each`, deduplication patterns, and practical examples for creating multiple resources with different configurations.

## toset() Function

`toset()` converts a list or tuple to a set — an unordered collection of unique values that automatically removes duplicates.

```hcl
toset(list)
```

### Basic Examples

```hcl
# List with duplicates
variable "availability_zones" {
  default = ["us-west-2a", "us-west-2b", "us-west-2a", "us-west-2c", "us-west-2b"]
}

# Convert to set (removes duplicates)
locals {
  unique_azs = toset(var.availability_zones)
  # Result: ["us-west-2a", "us-west-2b", "us-west-2c"]
}
```

```hcl
# List of environment names with duplicates
variable "environments" {
  default = ["dev", "staging", "prod", "dev", "test", "staging"]
}

locals {
  unique_environments = toset(var.environments)
  # Result: ["dev", "prod", "staging", "test"]
}
```

### Key Properties of Sets

1. **No duplicates** — automatically removes duplicate values
2. **Unordered** — doesn't maintain original order
3. **for_each compatible** — works directly with `for_each`
4. **Type consistency** — all elements must be the same type

## each.key vs each.value

### With Sets (toset) — Only each.key

When using `for_each` with a set, `each.key` and `each.value` are the same — the element itself.

```hcl
resource "aws_instance" "web" {
  for_each = toset(["web", "app", "db"])

  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = each.key  # "web", "app", or "db"
  }
}
```

### With Maps — Both each.key and each.value

When using `for_each` with a map, `each.key` is the map key and `each.value` is the associated value.

```hcl
variable "instances" {
  type = map(string)
  default = {
    "web"   = "t3.medium"
    "app"   = "t3.large"
    "db"    = "t3.xlarge"
    "cache" = "t3.small"
  }
}

resource "aws_instance" "servers" {
  for_each = var.instances

  ami           = var.ami_id
  instance_type = each.value  # "t3.medium", "t3.large", etc.

  tags = {
    Name = each.key    # "web", "app", "db", "cache"
    Type = each.value  # "t3.medium", "t3.large", etc.
  }
}
```

### Summary Table

| Collection Type | `each.key` | `each.value` | Example |
|----------------|-----------|-------------|---------|
| Set | The element | Same as each.key | `toset(["a", "b", "c"])` |
| Map (simple) | Map key | Map value | `{"web" = "t3.medium"}` |
| Map (object) | Map key | Object with attributes | `{"web" = {type = "t3.medium", ...}}` |

## Practical Examples

### Security Group Rules (toset — deduplication)

```hcl
variable "allowed_ports" {
  description = "List of allowed ports (may have duplicates)"
  type        = list(number)
  default     = [80, 443, 22, 80, 443, 8080]
}

resource "aws_security_group_rule" "ingress" {
  for_each = toset([for port in var.allowed_ports : tostring(port)])

  type        = "ingress"
  from_port   = tonumber(each.key)
  to_port     = tonumber(each.key)
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]

  security_group_id = aws_security_group.main.id
}
# Result: creates rules for ports 22, 80, 443, 8080 (duplicates removed)
```

### IAM Policy Attachments (toset — deduplication)

```hcl
variable "iam_policies" {
  description = "List of IAM policy ARNs (may have duplicates)"
  type        = list(string)
  default = [
    "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy",
    "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy",
    "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy",
    "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly",
  ]
}

resource "aws_iam_role_policy_attachment" "node_policies" {
  for_each = toset(var.iam_policies)

  role       = aws_iam_role.node_group.name
  policy_arn = each.key
}
# Result: attaches 3 unique policies (duplicate removed)
```

### Subnets Across Unique AZs (toset)

```hcl
variable "availability_zones" {
  type    = list(string)
  default = ["us-west-2a", "us-west-2b", "us-west-2a", "us-west-2c"]
}

resource "aws_subnet" "private" {
  for_each = toset(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  availability_zone = each.key
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, index(tolist(toset(var.availability_zones)), each.key))

  tags = {
    Name = "private-subnet-${each.key}"
    Type = "private"
  }
}
# Result: creates subnets only for unique AZs (3, not 4)
```

### Security Groups with Different Configs (map of objects — each.value)

```hcl
variable "security_groups" {
  type = map(object({
    description = string
    port        = number
    protocol    = string
  }))
  default = {
    "web" = {
      description = "HTTP/HTTPS traffic"
      port        = 80
      protocol    = "tcp"
    }
    "ssh" = {
      description = "SSH access"
      port        = 22
      protocol    = "tcp"
    }
    "database" = {
      description = "MySQL database"
      port        = 3306
      protocol    = "tcp"
    }
  }
}

resource "aws_security_group" "application" {
  for_each = var.security_groups

  name        = "${each.key}-sg"
  description = each.value.description
  vpc_id      = aws_vpc.main.id
}

resource "aws_security_group_rule" "ingress" {
  for_each = var.security_groups

  type        = "ingress"
  from_port   = each.value.port
  to_port     = each.value.port
  protocol    = each.value.protocol
  cidr_blocks = ["0.0.0.0/0"]

  security_group_id = aws_security_group.application[each.key].id
}
```

### EKS Node Groups (map of objects — each.value)

```hcl
variable "node_groups" {
  type = map(object({
    instance_types = list(string)
    desired_size   = number
    min_size       = number
    max_size       = number
    disk_size      = number
  }))
  default = {
    "general" = {
      instance_types = ["t3.medium"]
      desired_size   = 2
      min_size       = 1
      max_size       = 4
      disk_size      = 20
    }
    "compute" = {
      instance_types = ["c5.large", "c5.xlarge"]
      desired_size   = 1
      min_size       = 0
      max_size       = 10
      disk_size      = 50
    }
    "memory" = {
      instance_types = ["r5.large"]
      desired_size   = 1
      min_size       = 0
      max_size       = 5
      disk_size      = 100
    }
  }
}

resource "aws_eks_node_group" "main" {
  for_each = var.node_groups

  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${each.key}-nodes"
  node_role_arn   = aws_iam_role.node_group.arn
  subnet_ids      = var.private_subnet_ids

  instance_types = each.value.instance_types
  disk_size      = each.value.disk_size

  scaling_config {
    desired_size = each.value.desired_size
    max_size     = each.value.max_size
    min_size     = each.value.min_size
  }

  tags = {
    Name        = "${each.key}-node-group"
    Type        = each.key
    Environment = var.environment
  }
}
```

### S3 Buckets with Different Settings (map of objects — each.value)

```hcl
variable "s3_buckets" {
  type = map(object({
    versioning_enabled = bool
    encryption_enabled = bool
    public_access      = bool
    lifecycle_days     = number
  }))
  default = {
    "logs" = {
      versioning_enabled = true
      encryption_enabled = true
      public_access      = false
      lifecycle_days     = 30
    }
    "assets" = {
      versioning_enabled = false
      encryption_enabled = true
      public_access      = true
      lifecycle_days     = 365
    }
    "backups" = {
      versioning_enabled = true
      encryption_enabled = true
      public_access      = false
      lifecycle_days     = 90
    }
  }
}

resource "aws_s3_bucket" "main" {
  for_each = var.s3_buckets

  bucket = "${var.environment}-${each.key}-bucket"

  tags = {
    Name        = each.key
    Environment = var.environment
    Public      = each.value.public_access
  }
}

resource "aws_s3_bucket_versioning" "main" {
  for_each = {
    for k, v in var.s3_buckets : k => v
    if v.versioning_enabled
  }

  bucket = aws_s3_bucket.main[each.key].id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "main" {
  for_each = var.s3_buckets

  bucket = aws_s3_bucket.main[each.key].id

  rule {
    id     = "lifecycle-${each.key}"
    status = "Enabled"

    expiration {
      days = each.value.lifecycle_days
    }
  }
}
```

### Database Instances (map of objects — each.value)

```hcl
variable "databases" {
  type = map(object({
    engine            = string
    engine_version    = string
    instance_class    = string
    allocated_storage = number
    multi_az          = bool
  }))
  default = {
    "primary" = {
      engine            = "mysql"
      engine_version    = "8.0"
      instance_class    = "db.t3.medium"
      allocated_storage = 100
      multi_az          = true
    }
    "replica" = {
      engine            = "mysql"
      engine_version    = "8.0"
      instance_class    = "db.t3.small"
      allocated_storage = 50
      multi_az          = false
    }
    "analytics" = {
      engine            = "postgres"
      engine_version    = "13.7"
      instance_class    = "db.r5.large"
      allocated_storage = 200
      multi_az          = false
    }
  }
}

resource "aws_db_instance" "main" {
  for_each = var.databases

  identifier        = "${var.environment}-${each.key}-db"
  engine            = each.value.engine
  engine_version    = each.value.engine_version
  instance_class    = each.value.instance_class
  allocated_storage = each.value.allocated_storage
  multi_az          = each.value.multi_az

  db_name             = "${each.key}db"
  username            = "admin"
  password            = var.db_password
  skip_final_snapshot = true

  tags = {
    Name        = "${each.key}-database"
    Engine      = each.value.engine
    Environment = var.environment
  }
}
```

### Converting Set to Map for each.value

When you start with a set but need different configs per item, convert to a map using `for`:

```hcl
variable "environments" {
  type    = list(string)
  default = ["dev", "staging", "prod", "dev"]
}

locals {
  env_configs = {
    for env in toset(var.environments) : env => {
      instance_type = env == "prod" ? "t3.large" : "t3.micro"
      min_size      = env == "prod" ? 2 : 1
      max_size      = env == "prod" ? 10 : 3
    }
  }
}

resource "aws_launch_template" "app" {
  for_each = local.env_configs

  name_prefix   = "${each.key}-template-"
  image_id      = var.ami_id
  instance_type = each.value.instance_type

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "${each.key}-instance"
      Environment = each.key
      Type        = each.value.instance_type
    }
  }
}
# Result: 3 templates (dev, staging, prod) — duplicate "dev" removed by toset()
```

### Dynamic Tags from List (toset — deduplication)

```hcl
variable "tag_environments" {
  type    = list(string)
  default = ["dev", "staging", "prod", "dev", "test"]
}

locals {
  environment_tags = {
    for env in toset(var.tag_environments) :
    "Environment-${env}" => "true"
  }
  # Result: {
  #   "Environment-dev"     = "true"
  #   "Environment-prod"    = "true"
  #   "Environment-staging" = "true"
  #   "Environment-test"    = "true"
  # }
}
```

### Combining and Deduplicating Multiple Sources

```hcl
variable "primary_azs" {
  default = ["us-west-2a", "us-west-2b"]
}

variable "secondary_azs" {
  default = ["us-west-2c", "us-west-2a"]
}

locals {
  all_unique_azs = toset(concat(var.primary_azs, var.secondary_azs))
  # Result: ["us-west-2a", "us-west-2b", "us-west-2c"]

  az_list = tolist(local.all_unique_azs)
  # Convert back to list when you need indexing
}

resource "aws_subnet" "main" {
  for_each = local.all_unique_azs

  vpc_id            = aws_vpc.main.id
  availability_zone = each.key
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, index(local.az_list, each.key) + 1)

  tags = {
    Name = "subnet-${each.key}"
  }
}
```

### Filtered for_each (conditional resource creation)

```hcl
# Only create resources for items matching a condition
resource "aws_s3_bucket_versioning" "enabled" {
  for_each = {
    for k, v in var.s3_buckets : k => v
    if v.versioning_enabled
  }

  bucket = aws_s3_bucket.main[each.key].id
  versioning_configuration {
    status = "Enabled"
  }
}
# Result: only creates versioning for buckets where versioning_enabled = true
```

## Converting Between Types

```hcl
locals {
  # List → Set (deduplicate)
  unique_azs = toset(var.availability_zones)

  # Set → List (restore ordering/indexing)
  az_list = tolist(local.unique_azs)

  # List → Map (for for_each with index-based keys)
  az_map = { for i, az in var.availability_zones : az => i }

  # Set → Map (add computed values)
  az_configs = {
    for az in local.unique_azs : az => {
      cidr = cidrsubnet("10.0.0.0/16", 8, index(local.az_list, az))
    }
  }
}
```

## Error: Mixed Types in Set

Sets require all elements to be the same type:

```hcl
# BAD — mixed types will fail
variable "mixed_values" {
  default = ["string", 123, true]
}

locals {
  bad_set = toset(var.mixed_values)  # ERROR: elements must be same type
}

# GOOD — convert to strings first
locals {
  good_set = toset([for v in var.mixed_values : tostring(v)])
}
```

## When to Use What

| Scenario | Use | Why |
|----------|-----|-----|
| Unique identifiers, same config | `toset()` + `each.key` | Simple, deduplicates automatically |
| Unique identifiers, different configs | Map of objects + `each.value` | Access per-resource settings |
| List with possible duplicates | `toset()` wrapper | Prevents duplicate resource errors |
| Need ordering or indexing | `tolist()` conversion | Sets are unordered |
| Conditional resource creation | `for` + `if` in `for_each` | Filter which resources to create |
| Multiple sources combined | `toset(concat(...))` | Merge and deduplicate |

## Best Practices

1. **Use `toset()` for deduplication** — when lists might contain duplicates from multiple sources
2. **Prefer maps for `for_each`** — when each resource needs different configuration
3. **Use `toset()` with `for_each`** — when all resources have the same config and you just need unique identifiers
4. **Convert types when needed** — `tolist()` for indexing, `tomap()` for key-value access
5. **Filter in `for_each`** — use `for k, v in map : k => v if condition` to conditionally create resources
6. **Keep sets homogeneous** — all elements must be the same type (convert with `tostring()` if needed)
