# Terraform Outputs Guide

Outputs expose values from your Terraform configuration — making them available to the CLI, to other modules, to remote state consumers, and to automation scripts. They are Terraform's way of returning data after an apply.

## Basic Output

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }
}

output "instance_id" {
  value       = aws_instance.web.id
  description = "The ID of the EC2 instance"
}

output "public_ip" {
  value       = aws_instance.web.public_ip
  description = "The public IP address of the instance"
}
```

After `terraform apply`, outputs are displayed:

```bash
Outputs:

instance_id = "i-0abc123def456789"
public_ip = "54.23.100.50"
```

## Output Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `value` | Yes | The expression to expose |
| `description` | No | Human-readable description (shown in docs and CLI) |
| `sensitive` | No | Mark as sensitive to hide from CLI output (`true`/`false`) |
| `depends_on` | No | Explicit dependency list (rarely needed) |
| `precondition` | No | Validation rule that must pass before output is evaluated (1.2+) |

## Output Types

Outputs can return any Terraform type — strings, numbers, booleans, lists, maps, and objects.

### String

```hcl
output "bucket_name" {
  value       = aws_s3_bucket.main.id
  description = "The name of the S3 bucket"
}
```

### Number

```hcl
output "instance_count" {
  value       = length(aws_instance.app)
  description = "Number of instances created"
}
```

### Boolean

```hcl
output "is_production" {
  value       = var.environment == "production"
  description = "Whether this is a production deployment"
}
```

### List

```hcl
output "instance_ids" {
  value       = aws_instance.app[*].id
  description = "List of all instance IDs"
}

output "private_ips" {
  value       = aws_instance.app[*].private_ip
  description = "List of all private IP addresses"
}
```

### Map

```hcl
output "instance_map" {
  value = {
    for instance in aws_instance.app :
    instance.tags["Name"] => instance.public_ip
  }
  description = "Map of instance names to their public IPs"
}
```

### Object (Complex)

```hcl
output "cluster_info" {
  value = {
    endpoint        = aws_eks_cluster.main.endpoint
    ca_certificate  = aws_eks_cluster.main.certificate_authority[0].data
    cluster_name    = aws_eks_cluster.main.name
    version         = aws_eks_cluster.main.version
    security_groups = aws_eks_cluster.main.vpc_config[0].security_group_ids
  }
  description = "EKS cluster connection details"
}
```

## Sensitive Outputs

Mark outputs as sensitive to prevent their values from being shown in CLI output and logs. The value is still stored in state.

```hcl
output "db_password" {
  value       = aws_db_instance.main.password
  description = "The database master password"
  sensitive   = true
}

output "api_key" {
  value       = aws_api_gateway_api_key.main.value
  description = "The API key for the gateway"
  sensitive   = true
}
```

```bash
# CLI shows:
# db_password = <sensitive>

# To reveal sensitive outputs:
terraform output db_password
terraform output -raw db_password    # No quotes, useful for piping
```

## Conditional Outputs

Output a value only when the resource exists.

```hcl
variable "create_bucket" {
  type    = bool
  default = true
}

resource "aws_s3_bucket" "main" {
  count  = var.create_bucket ? 1 : 0
  bucket = "my-app-bucket"
}

output "bucket_arn" {
  value       = var.create_bucket ? aws_s3_bucket.main[0].arn : null
  description = "The S3 bucket ARN, or null if not created"
}

# With one() for for_each resources
output "bucket_arn_v2" {
  value       = one(aws_s3_bucket.main[*].arn)
  description = "The bucket ARN using one() — returns null if empty"
}
```

## Output Preconditions (Terraform 1.2+)

Validate output values before they are exposed.

```hcl
output "api_endpoint" {
  value       = aws_api_gateway_rest_api.main.execution_arn
  description = "The API Gateway execution ARN"

  precondition {
    condition     = aws_api_gateway_rest_api.main.execution_arn != ""
    error_message = "API Gateway execution ARN is empty — deployment may have failed."
  }
}
```

## CLI Commands

### View All Outputs

```bash
# Show all outputs after apply
terraform output

# JSON format (useful for scripts)
terraform output -json

# Specific output
terraform output instance_id

# Raw value (no quotes, no newline) — useful for shell scripts
terraform output -raw public_ip
```

### Use Outputs in Scripts

```bash
# Store in a variable
INSTANCE_IP=$(terraform output -raw public_ip)
ssh ubuntu@$INSTANCE_IP

# Get a list as JSON and parse with jq
terraform output -json instance_ids | jq -r '.[]'

# Get a map value
terraform output -json instance_map | jq -r '.["web-1"]'

# Use in another command
aws ec2 describe-instances \
  --instance-ids $(terraform output -raw instance_id)
```

### Outputs from State File

```bash
# Show outputs from a specific state file
terraform output -state=terraform.tfstate

# Show outputs from remote state without full init
terraform output -state=.terraform/terraform.tfstate
```

## Module Outputs

Outputs are the primary way to pass data between modules. A child module must explicitly declare outputs for the parent to consume them.

### Child Module (modules/vpc/outputs.tf)

```hcl
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "The ID of the VPC"
}

output "public_subnet_ids" {
  value       = aws_subnet.public[*].id
  description = "List of public subnet IDs"
}

output "private_subnet_ids" {
  value       = aws_subnet.private[*].id
  description = "List of private subnet IDs"
}

output "nat_gateway_ip" {
  value       = aws_eip.nat.public_ip
  description = "The NAT Gateway public IP"
}
```

### Parent Module (main.tf)

```hcl
module "vpc" {
  source = "./modules/vpc"

  vpc_cidr    = "10.0.0.0/16"
  environment = var.environment
}

# Access child module outputs
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  subnet_id     = module.vpc.public_subnet_ids[0]

  tags = {
    Name = "app-server"
  }
}

# Re-export child module outputs
output "vpc_id" {
  value       = module.vpc.vpc_id
  description = "The VPC ID from the vpc module"
}
```

### Module Output with count

```hcl
module "database" {
  source = "./modules/rds"
  count  = var.create_database ? 1 : 0

  instance_class = "db.t3.micro"
  engine         = "postgres"
}

output "db_endpoint" {
  value       = var.create_database ? module.database[0].endpoint : null
  description = "The database endpoint, or null if not created"
}
```

## Remote State Outputs

Access outputs from another Terraform state using `terraform_remote_state`.

### Producer (outputs the data)

```hcl
# In the networking workspace
output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
```

### Consumer (reads the data)

```hcl
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "eu-west-1"
  }
}

# Use remote outputs
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  subnet_id     = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]

  tags = {
    VPC = data.terraform_remote_state.networking.outputs.vpc_id
  }
}
```

## Output Patterns

### All Attributes of a Resource

```hcl
output "instance_details" {
  value       = aws_instance.web
  description = "All attributes of the web instance"
  sensitive   = true  # May contain sensitive data
}
```

### Formatted Output String

```hcl
output "connection_string" {
  value       = "postgresql://${aws_db_instance.main.username}:${aws_db_instance.main.password}@${aws_db_instance.main.endpoint}/${aws_db_instance.main.db_name}"
  description = "PostgreSQL connection string"
  sensitive   = true
}

output "ssh_command" {
  value       = "ssh -i ~/.ssh/id_rsa ${var.ssh_user}@${aws_instance.web.public_ip}"
  description = "SSH command to connect to the instance"
}
```

### Multiline Heredoc Output (Summary Banner)

Use a heredoc to produce a formatted summary with all key values in one block — useful for EKS clusters, full stack deployments, or anything with many related outputs.

```hcl
output "eks" {
  value = <<EOF
###################################### KUBECONFIG ###########################################

        aws eks --region us-east-1 update-kubeconfig --name ${var.cluster_name}

############################# N E T W O R K I N G ###########################################
        VPC ID                                  ${aws_vpc.main.id}
        Public subnet 1                         ${aws_subnet.public-1.id}
        Public subnet 2                         ${aws_subnet.public-2.id}
        Public subnet 3                         ${aws_subnet.public-3.id}
        Private subnet 1                        ${aws_subnet.private-1.id}
        Private subnet 2                        ${aws_subnet.private-2.id}
        Private subnet 3                        ${aws_subnet.private-3.id}
###################################### ALB ASG ACM ##########################################
        EKS Cluster autoscaler role arn         ${aws_iam_role.eks_cluster_autoscaler.arn}
        AWS LoadBalancer controller arn         ${aws_iam_role.aws_load_balancer_controller.arn}
        ACM Certificate arn                     ${module.cert.arn}
###################################### EKS Cluster #########################################
        ${var.cluster_name} EKS Cluster Role    ${aws_iam_role.dev.arn}
        EKS Nodes Group Role                    ${aws_iam_role.nodes.arn}
        EKS OIDC                                ${aws_iam_role.dev_oidc.arn}
        OpenID Connect Provider                 ${aws_iam_openid_connect_provider.eks.url}
    EOF
}
# Result: after apply, prints a complete infrastructure summary
# Useful for copy-pasting into documentation or sharing with the team
```

### Map Output for Cloud Credentials (Azure Example)

Output a structured map of values needed for downstream configuration — CI/CD pipelines, backend setup, or environment variable injection.

```hcl
output "azure_info" {
  value = {
    RESOURCE_GROUP      = azurerm_storage_account.sa.resource_group_name
    CONTAINER_NAME      = azurerm_storage_container.ct.name
    ARM_CLIENT_ID       = azuread_service_principal.remote_state.application_id
    ARM_CLIENT_SECRET   = nonsensitive(azuread_service_principal_password.remote_state.value)
    ARM_SUBSCRIPTION_ID = data.azurerm_subscription.current.subscription_id
    ARM_TENANT_ID       = data.azuread_client_config.current.tenant_id
  }
}
# Result: outputs all values needed to configure an Azure remote backend
# Use: terraform output -json azure_info | jq -r 'to_entries[] | "\(.key)=\(.value)"'
# Note: nonsensitive() explicitly unwraps a sensitive value for display
```

### Splat Expressions for Lists

```hcl
resource "aws_instance" "app" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

# All IDs
output "all_ids" {
  value = aws_instance.app[*].id
}

# All public IPs
output "all_public_ips" {
  value = aws_instance.app[*].public_ip
}

# Zipped (name → IP)
output "instance_ips" {
  value = { for i, inst in aws_instance.app : "app-${i}" => inst.public_ip }
}
```

### for_each Resource Outputs

```hcl
variable "buckets" {
  type    = set(string)
  default = ["logs", "assets", "backups"]
}

resource "aws_s3_bucket" "multi" {
  for_each = var.buckets
  bucket   = "myapp-${each.key}"
}

# Map of bucket names to ARNs
output "bucket_arns" {
  value = { for k, v in aws_s3_bucket.multi : k => v.arn }
}

# Just the ARN list
output "bucket_arn_list" {
  value = values(aws_s3_bucket.multi)[*].arn
}
```

### Depends_on in Outputs

Rarely needed, but forces Terraform to complete a resource before evaluating the output.

```hcl
output "app_url" {
  value       = "https://${aws_route53_record.app.fqdn}"
  description = "The application URL (available after DNS propagation)"
  depends_on  = [aws_route53_record.app]
}
```

## File Organization

Standard Terraform convention is to place outputs in a dedicated file:

```
project/
├── main.tf            # Resources
├── variables.tf       # Input variables
├── outputs.tf         # Output declarations
├── providers.tf       # Provider configuration
└── terraform.tfvars   # Variable values
```

### outputs.tf Example

```hcl
# outputs.tf

output "vpc_id" {
  value       = aws_vpc.main.id
  description = "The VPC ID"
}

output "public_subnet_ids" {
  value       = aws_subnet.public[*].id
  description = "Public subnet IDs"
}

output "instance_public_ip" {
  value       = aws_instance.web.public_ip
  description = "Web server public IP"
}

output "db_endpoint" {
  value       = aws_db_instance.main.endpoint
  description = "RDS endpoint"
  sensitive   = true
}

output "load_balancer_dns" {
  value       = aws_lb.main.dns_name
  description = "ALB DNS name"
}
```

## Debugging Outputs

### terraform console

```bash
# Open interactive console to test expressions
terraform console

# Test output expressions
> aws_instance.web.public_ip
"54.23.100.50"

> aws_instance.app[*].id
["i-abc123", "i-def456", "i-ghi789"]

> { for i, inst in aws_instance.app : "app-${i}" => inst.private_ip }
{ "app-0" = "10.0.1.10", "app-1" = "10.0.1.11", "app-2" = "10.0.1.12" }
```

### terraform show

```bash
# Show all outputs from current state
terraform show -json | jq '.values.outputs'

# Show a specific output's full detail
terraform show -json | jq '.values.outputs.instance_id'
```

## Best Practices

1. **Always add `description`** — documents what the output is for and makes `terraform output` self-explanatory
2. **Mark secrets as `sensitive`** — passwords, tokens, private keys, connection strings
3. **Use `outputs.tf`** — keep all outputs in one file per module for discoverability
4. **Output only what consumers need** — don't expose internal implementation details
5. **Use meaningful names** — `vpc_id` not `id`, `public_subnet_ids` not `subnets`
6. **Prefer structured outputs** — maps and objects are easier to consume than multiple flat outputs
7. **Use `-raw` for scripts** — avoids quoting issues when piping to other commands
8. **Use `-json` for automation** — parse with `jq` for reliable extraction
9. **Don't output entire resources** unless necessary — mark as sensitive if you do
10. **Add `precondition`** to validate outputs that consumers depend on being non-empty

## Quick Reference

```bash
# View all outputs
terraform output

# View one output
terraform output vpc_id

# Raw value (no quotes)
terraform output -raw public_ip

# JSON format
terraform output -json

# Use in shell variable
IP=$(terraform output -raw public_ip)

# Parse JSON output with jq
terraform output -json instance_ids | jq -r '.[0]'

# Show outputs from state
terraform show -json | jq '.values.outputs'
```

| Pattern | Example |
|---------|---------|
| Simple value | `value = aws_instance.web.id` |
| List (splat) | `value = aws_instance.app[*].id` |
| Map comprehension | `value = { for k, v in resource : k => v.attr }` |
| Conditional | `value = var.create ? resource[0].id : null` |
| one() | `value = one(aws_eip.nat[*].public_ip)` |
| Sensitive | `sensitive = true` |
| Module output | `value = module.vpc.vpc_id` |
| Remote state | `data.terraform_remote_state.x.outputs.y` |
| Formatted string | `value = "ssh user@${resource.ip}"` |
| Precondition | `precondition { condition = ... }` |
