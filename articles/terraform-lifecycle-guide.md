# Terraform Lifecycle Guide

The `lifecycle` block controls how Terraform creates, updates, and destroys resources. It lets you prevent accidental deletion, avoid downtime during replacements, ignore external changes, and validate configurations before applying.

## Lifecycle Arguments

| Argument | Type | Purpose |
|----------|------|---------|
| `create_before_destroy` | bool | Create replacement before destroying the original |
| `prevent_destroy` | bool | Block any plan that would destroy the resource |
| `ignore_changes` | list | Ignore specific attribute changes from external sources |
| `replace_triggered_by` | list | Force replacement when referenced values change |
| `precondition` | block | Validate assumptions before applying |
| `postcondition` | block | Validate results after applying |

## create_before_destroy

Creates the new resource before destroying the old one. Essential for zero-downtime replacements where the old resource must remain available until the new one is ready.

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = var.instance_type

  lifecycle {
    create_before_destroy = true
  }
}
# Default behavior: destroy old → create new (causes downtime)
# With create_before_destroy: create new → destroy old (zero downtime)
```

### When to Use

```hcl
# Launch configurations (required for ASG updates)
resource "aws_launch_configuration" "app" {
  image_id      = var.ami
  instance_type = var.instance_type

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "app" {
  launch_configuration = aws_launch_configuration.app.name
  min_size             = 2
  max_size             = 10

  lifecycle {
    create_before_destroy = true
  }
}

# Security groups referenced by other resources
resource "aws_security_group" "web" {
  name_prefix = "web-sg-"
  vpc_id      = var.vpc_id

  lifecycle {
    create_before_destroy = true
  }
}

# SSL certificates (old cert must stay valid until new one is in use)
resource "aws_acm_certificate" "main" {
  domain_name       = var.domain
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}
```

### Gotchas

```hcl
# Resources with unique constraints (like name) can't use create_before_destroy
# unless you use name_prefix instead of name

# BAD — can't create new with same name before deleting old
resource "aws_security_group" "web" {
  name   = "web-sg"  # Conflict!
  vpc_id = var.vpc_id

  lifecycle {
    create_before_destroy = true
  }
}

# GOOD — name_prefix generates unique names
resource "aws_security_group" "web" {
  name_prefix = "web-sg-"
  vpc_id      = var.vpc_id

  lifecycle {
    create_before_destroy = true
  }
}
```

## prevent_destroy

Blocks any plan that would destroy the resource. Terraform will exit with an error if destruction is attempted — including `terraform destroy`.

```hcl
resource "aws_db_instance" "production" {
  identifier     = "prod-database"
  engine         = "postgres"
  instance_class = "db.r5.large"

  lifecycle {
    prevent_destroy = true
  }
}
# Result: terraform destroy or any change that forces recreation will fail
# Error: Instance cannot be destroyed
```

### Common Use Cases

```hcl
# Production databases
resource "aws_db_instance" "prod" {
  identifier = "prod-db"
  # ...
  lifecycle { prevent_destroy = true }
}

# S3 buckets with important data
resource "aws_s3_bucket" "data" {
  bucket = "company-critical-data"
  lifecycle { prevent_destroy = true }
}

# KMS encryption keys
resource "aws_kms_key" "main" {
  description = "Main encryption key"
  lifecycle { prevent_destroy = true }
}

# Route53 hosted zones
resource "aws_route53_zone" "primary" {
  name = "example.com"
  lifecycle { prevent_destroy = true }
}

# VPCs (destroying would cascade to all resources inside)
resource "aws_vpc" "production" {
  cidr_block = "10.0.0.0/16"
  lifecycle { prevent_destroy = true }
}
```

### Removing prevent_destroy

To actually destroy a resource with `prevent_destroy`:

```hcl
# Step 1: Remove prevent_destroy from the lifecycle block
resource "aws_db_instance" "prod" {
  # ...
  lifecycle {
    # prevent_destroy = true  ← remove or set to false
  }
}

# Step 2: Apply the config change
# Step 3: Now you can destroy
terraform destroy -target=aws_db_instance.prod
```

Or remove from state without destroying:

```bash
# Remove from Terraform state (resource stays in cloud)
terraform state rm aws_db_instance.prod
```

## ignore_changes

Tells Terraform to ignore changes to specific attributes made outside of Terraform. The resource won't be updated even if the real-world value differs from the config.

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = var.instance_type

  tags = {
    Name = "web-server"
  }

  lifecycle {
    ignore_changes = [
      tags["LastModifiedBy"],  # Ignore tags set by automation
      ami,                     # Don't replace when AMI updates
    ]
  }
}
```

### Common Patterns

```hcl
# Ignore auto-scaling changes to desired_capacity
resource "aws_autoscaling_group" "app" {
  desired_capacity = 3
  min_size         = 2
  max_size         = 10

  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
# Result: ASG autoscaling changes desired_capacity, Terraform won't reset it

# Ignore ECS task definition updates (deployed by CI/CD)
resource "aws_ecs_service" "app" {
  name            = "app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 3

  lifecycle {
    ignore_changes = [task_definition, desired_count]
  }
}
# Result: CI/CD updates the task definition, Terraform won't revert it

# Ignore Lambda code updates (deployed separately)
resource "aws_lambda_function" "app" {
  function_name = "my-function"
  handler       = "index.handler"
  runtime       = "nodejs18.x"
  filename      = "lambda.zip"

  lifecycle {
    ignore_changes = [filename, source_code_hash]
  }
}

# Ignore tags added by AWS services (Config, SSM, etc.)
resource "aws_instance" "managed" {
  ami           = var.ami
  instance_type = "t3.micro"

  lifecycle {
    ignore_changes = [tags]
  }
}
```

### ignore_changes = all

Ignore ALL attribute changes — Terraform manages creation and destruction only:

```hcl
resource "aws_instance" "externally_managed" {
  ami           = var.ami
  instance_type = "t3.micro"

  lifecycle {
    ignore_changes = all
  }
}
# Result: Terraform creates the resource but never updates it
# Useful for resources managed by external tools after creation
```

### What ignore_changes Does NOT Do

```hcl
# ignore_changes does NOT prevent:
# - Resource creation
# - Resource destruction (use prevent_destroy for that)
# - Changes to arguments that force recreation

# If the ignored attribute causes a force-replacement, Terraform still replaces
resource "aws_instance" "web" {
  ami = var.ami  # Changing AMI forces new instance

  lifecycle {
    ignore_changes = [ami]  # Terraform won't detect the change at all
  }
}
```

## replace_triggered_by (Terraform 1.2+)

Forces a resource to be replaced when a referenced resource or attribute changes, even if the resource itself hasn't changed.

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"

  lifecycle {
    replace_triggered_by = [
      aws_security_group.web.id,  # Replace instance if SG changes
    ]
  }
}

# Replace when a null_resource trigger fires
resource "null_resource" "deployment_trigger" {
  triggers = {
    version = var.app_version
  }
}

resource "aws_instance" "app" {
  ami           = var.ami
  instance_type = "t3.micro"

  lifecycle {
    replace_triggered_by = [
      null_resource.deployment_trigger,
    ]
  }
}
# Result: instance is recreated every time app_version changes
```

### Practical Examples

```hcl
# Replace ECS tasks when secrets rotate
resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = var.db_password
}

resource "aws_ecs_service" "app" {
  name            = "app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn

  lifecycle {
    replace_triggered_by = [
      aws_secretsmanager_secret_version.db_password,
    ]
  }
}
# Result: ECS service redeployed when DB password rotates

# Replace instance when user_data template changes
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"
  user_data     = templatefile("init.sh.tpl", { version = var.app_version })

  lifecycle {
    replace_triggered_by = [
      terraform_data.userdata_hash,
    ]
  }
}

resource "terraform_data" "userdata_hash" {
  input = filemd5("init.sh.tpl")
}
```

## precondition and postcondition (Terraform 1.2+)

### precondition

Validates assumptions BEFORE Terraform applies changes. If the condition fails, Terraform stops with an error.

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = var.environment != "prod" || var.instance_type != "t3.micro"
      error_message = "Production instances must not use t3.micro."
    }
  }
}
# Result: prevents deploying t3.micro to production

resource "aws_db_instance" "main" {
  identifier     = "${var.environment}-db"
  engine         = "postgres"
  instance_class = var.db_instance_class
  multi_az       = var.environment == "prod"

  lifecycle {
    precondition {
      condition     = var.environment != "prod" || var.multi_az == true
      error_message = "Production databases must be multi-AZ."
    }

    precondition {
      condition     = var.allocated_storage >= 20
      error_message = "Minimum storage is 20 GB."
    }
  }
}
```

### postcondition

Validates results AFTER the resource is created/updated. Catches issues that can only be verified at apply time.

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = var.instance_type

  lifecycle {
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP assigned."
    }
  }
}

resource "aws_db_instance" "main" {
  identifier     = "prod-db"
  engine         = "postgres"
  instance_class = "db.r5.large"

  lifecycle {
    postcondition {
      condition     = self.status == "available"
      error_message = "Database did not reach 'available' state."
    }

    postcondition {
      condition     = self.endpoint != ""
      error_message = "Database endpoint is empty — creation may have failed."
    }
  }
}
```

### precondition in Data Sources

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-22.04-amd64-server-*"]
  }

  lifecycle {
    postcondition {
      condition     = self.image_id != ""
      error_message = "No matching AMI found."
    }
  }
}
```

## Combining Lifecycle Arguments

Multiple arguments can be used together:

```hcl
resource "aws_instance" "critical_app" {
  ami           = var.ami
  instance_type = var.instance_type

  tags = {
    Name        = "critical-app"
    Environment = var.environment
  }

  lifecycle {
    # Zero-downtime replacement
    create_before_destroy = true

    # Never accidentally destroy
    prevent_destroy = true

    # Ignore tags managed by AWS Systems Manager
    ignore_changes = [
      tags["aws:ssm:managed"],
      tags["LastPatchTime"],
    ]

    # Validate before applying
    precondition {
      condition     = var.environment == "prod"
      error_message = "This resource is only for production."
    }
  }
}
```

### Database with Full Lifecycle Protection

```hcl
resource "aws_db_instance" "production" {
  identifier     = "prod-main-db"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.r5.large"

  allocated_storage     = 100
  max_allocated_storage = 500
  multi_az              = true
  storage_encrypted     = true

  backup_retention_period = 30
  deletion_protection     = true
  skip_final_snapshot     = false
  final_snapshot_identifier = "prod-main-db-final-${formatdate("YYYY-MM-DD", timestamp())}"

  lifecycle {
    prevent_destroy = true

    ignore_changes = [
      max_allocated_storage,  # Auto-scaling manages this
    ]

    precondition {
      condition     = var.multi_az == true
      error_message = "Production database must be multi-AZ."
    }

    postcondition {
      condition     = self.status == "available"
      error_message = "Database is not in 'available' state."
    }
  }
}
```

## Lifecycle in Modules

Lifecycle blocks can't be passed as module variables. Each resource within a module must define its own lifecycle:

```hcl
# modules/ec2/main.tf
variable "prevent_destroy" {
  type    = bool
  default = false
}

# This does NOT work:
# lifecycle {
#   prevent_destroy = var.prevent_destroy  # ERROR: variables not allowed
# }

# Workaround: use separate resource blocks
resource "aws_instance" "protected" {
  count         = var.prevent_destroy ? 1 : 0
  ami           = var.ami
  instance_type = var.instance_type

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_instance" "unprotected" {
  count         = var.prevent_destroy ? 0 : 1
  ami           = var.ami
  instance_type = var.instance_type
}
```

**Note:** As of Terraform 1.3+, lifecycle values must be literal — they cannot reference variables, locals, or expressions.

## Common Patterns

### Blue-Green Deployments

```hcl
resource "aws_lb_target_group" "app" {
  name_prefix = "app-"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = var.vpc_id

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_lb_listener_rule" "app" {
  listener_arn = var.listener_arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }

  condition {
    path_pattern { values = ["/app/*"] }
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

### Immutable Infrastructure

```hcl
# Instance is replaced entirely on any config change
resource "aws_instance" "immutable" {
  ami           = var.ami
  instance_type = var.instance_type
  user_data     = templatefile("init.sh.tpl", { version = var.app_version })

  lifecycle {
    create_before_destroy = true
    # No ignore_changes — any drift triggers replacement
  }
}
```

### Externally Managed Resources

```hcl
# Terraform creates the resource, external tools manage it afterward
resource "aws_autoscaling_group" "managed_externally" {
  name             = "externally-managed-asg"
  min_size         = 1
  max_size         = 20
  desired_capacity = 3

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  lifecycle {
    ignore_changes = [
      desired_capacity,  # Managed by auto-scaling policies
      target_group_arns, # Managed by service mesh
      tag,               # Managed by cluster-autoscaler
    ]
  }
}
```

### Ignore Password (Managed Outside Terraform)

```hcl
resource "aws_db_instance" "app_db" {
  identifier     = "app-database"
  engine         = "postgres"
  instance_class = "db.t3.micro"
  username       = "dbadmin"
  password       = var.initial_db_password

  lifecycle {
    ignore_changes = [password]
  }
}
# Result: Terraform sets the initial password on creation
#         DBA rotates it later — Terraform won't reset it
```

### Ignore user_data After Initial Bootstrap

```hcl
resource "aws_instance" "bootstrap" {
  ami           = var.ami
  instance_type = "t3.micro"

  user_data = base64encode(templatefile("${path.module}/bootstrap.sh", {
    environment = var.environment
    app_version = var.app_version
  }))

  lifecycle {
    ignore_changes = [user_data]
  }
}
# Result: instance is bootstrapped once on creation
#         changing app_version won't force instance replacement
```

### Ignore All Tags (Managed by External Tagging Policies)

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "app-logs-bucket"

  lifecycle {
    ignore_changes = [tags, tags_all]
  }
}
# Result: AWS Config, Organizations tag policies, or other tools
#         can modify tags without triggering Terraform drift
```

### Force Replacement via Variable

```hcl
variable "force_instance_replacement" {
  type        = string
  default     = "v1"
  description = "Change this value to force instance recreation"
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.latest.id
  instance_type = "t3.micro"

  lifecycle {
    ignore_changes = [ami]
    replace_triggered_by = [
      var.force_instance_replacement,
    ]
  }
}
# Result: changing force_instance_replacement = "v2" forces rebuild
#         AMI updates are ignored unless you bump the trigger variable
```

### Combining All Meta-Arguments

```hcl
variable "web_servers" {
  type = map(object({
    instance_type = string
    subnet_id     = string
    enable        = bool
  }))
}

resource "aws_instance" "web_servers" {
  for_each = {
    for k, v in var.web_servers : k => v
    if v.enable
  }

  ami           = var.ami
  instance_type = each.value.instance_type
  subnet_id     = each.value.subnet_id

  depends_on = [
    aws_security_group.web,
    aws_route_table.main,
  ]

  lifecycle {
    create_before_destroy = true
    ignore_changes        = [ami]
  }

  provisioner "local-exec" {
    command = "echo 'Server ${each.key} created at ${self.public_ip}'"
  }

  tags = {
    Name        = "web-server-${each.key}"
    Environment = var.environment
  }
}
# Combines: for_each, depends_on, lifecycle, and provisioner in one resource
```

## Quick Reference

| Argument | Effect | Use When |
|----------|--------|----------|
| `create_before_destroy = true` | New resource created before old is destroyed | Zero-downtime, unique name constraints |
| `prevent_destroy = true` | Blocks plans that destroy the resource | Databases, encryption keys, critical infra |
| `ignore_changes = [attr]` | Ignores external modifications to listed attributes | Auto-scaling, CI/CD deployments, external tools |
| `ignore_changes = all` | Ignores ALL external modifications | Fully externally managed resources |
| `replace_triggered_by = [ref]` | Forces recreation when referenced value changes | Secret rotation, config file changes |
| `precondition {}` | Validates before apply | Guard rails, environment restrictions |
| `postcondition {}` | Validates after apply | Verify endpoints, status, connectivity |
