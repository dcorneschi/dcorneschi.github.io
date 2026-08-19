# Terraform Heredoc: EOT vs EOF

HCL heredoc syntax for multi-line strings in Terraform — what the delimiter names mean, `<<` vs `<<-`, indentation handling, interpolation, and when to use each pattern.

## Heredoc Syntax in Terraform

Terraform uses heredoc syntax to define multi-line strings. The delimiter name (EOT, EOF, etc.) is arbitrary — it just marks the start and end of the string.

```hcl
# Basic heredoc
variable = <<DELIMITER
content goes here
DELIMITER
```

The two common delimiters are `EOT` (End Of Text) and `EOF` (End Of File), but you can use any identifier.

## EOT vs EOF — Is There a Difference?

**No functional difference.** Both `EOT` and `EOF` are just delimiter names. Terraform doesn't treat them differently. The choice is purely a convention:

| Delimiter | Convention | Commonly Used For |
|-----------|-----------|-------------------|
| `EOT` | End Of Text | General multi-line strings, IAM policies, config files |
| `EOF` | End Of File | Shell scripts, user_data, file content |
| `YAML` | Custom name | Embedding YAML content |
| `JSON` | Custom name | Embedding JSON content |
| `SCRIPT` | Custom name | Embedding shell scripts |
| `POLICY` | Custom name | IAM or resource policies |

Using a descriptive delimiter name improves readability — `<<POLICY` is clearer than `<<EOF` when defining an IAM policy.

## Standard Heredoc (<<)

The opening `<<` preserves all whitespace exactly as written. The closing delimiter must be on its own line with no leading whitespace.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  user_data = <<EOF
#!/bin/bash
apt-get update
apt-get install -y nginx
systemctl enable --now nginx
echo "Hello from $(hostname)" > /var/www/html/index.html
EOF
}
```

The resulting string has no leading indentation — content starts at column 0.

## Indented Heredoc (<<-)

The `<<-` variant strips leading whitespace from all lines and the closing delimiter, letting you indent the heredoc with your code.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  user_data = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl enable --now nginx
    echo "Hello from $(hostname)" > /var/www/html/index.html
  EOF
}
```

Terraform strips the common leading whitespace, producing the same output as the non-indented version. This keeps your HCL readable without affecting the content.

### How <<- Stripping Works

Terraform finds the line with the least indentation (excluding the delimiter line) and removes that many spaces from every line:

```hcl
locals {
  script = <<-EOT
    line 1 (4 spaces removed)
      line 2 (4 spaces removed, 2 remain)
    line 3 (4 spaces removed)
  EOT
}
# Result:
# "line 1 (4 spaces removed)\n  line 2 (4 spaces removed, 2 remain)\nline 3 (4 spaces removed)\n"
```

## Interpolation in Heredocs

Terraform interpolation (`${...}`) works inside heredocs. This is different from shell heredocs where quoting the delimiter controls expansion.

```hcl
variable "environment" {
  type    = string
  default = "production"
}

variable "app_port" {
  type    = number
  default = 8080
}

locals {
  config = <<-EOT
    [app]
    environment = ${var.environment}
    port = ${var.app_port}
    workers = ${var.environment == "production" ? 4 : 1}
  EOT
}
# Result:
# [app]
# environment = production
# port = 8080
# workers = 4
```

### Escaping Dollar Signs

If you need a literal `${` in your heredoc (for shell variables, for example), escape it with `$${`:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  user_data = <<-EOF
    #!/bin/bash
    # Terraform variable (interpolated at plan time)
    ENVIRONMENT="${var.environment}"

    # Shell variable (literal, passed through to the script)
    HOSTNAME=$${HOSTNAME}
    echo "Running on $${HOSTNAME} in $ENVIRONMENT"

    # Shell command substitution (literal)
    CURRENT_DATE=$$(date +%Y-%m-%d)
  EOF
}
# $${HOSTNAME} becomes ${HOSTNAME} in the rendered script
# $$(date) becomes $(date) in the rendered script
```

### Escaping Percent Signs

For `%{` template directives, use `%%{` to produce a literal `%{`:

```hcl
locals {
  output = <<-EOT
    This is a template directive: %{ if var.enabled }yes%{ endif }
    This is a literal percent-brace: %%{ not a directive }
  EOT
}
```

## Common Patterns

### IAM Policy

```hcl
resource "aws_iam_role_policy" "app" {
  name = "app-policy"
  role = aws_iam_role.app.id

  policy = <<-POLICY
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "s3:GetObject",
            "s3:PutObject"
          ],
          "Resource": "arn:aws:s3:::${var.bucket_name}/*"
        }
      ]
    }
  POLICY
}
```

### User Data Script

```hcl
resource "aws_instance" "app" {
  ami           = var.ami
  instance_type = var.instance_type

  user_data = <<-SCRIPT
    #!/bin/bash
    set -euo pipefail

    # Terraform variables (interpolated)
    APP_VERSION="${var.app_version}"
    DB_HOST="${aws_db_instance.main.endpoint}"

    # Shell variables (escaped, not interpolated by Terraform)
    LOG_FILE="/var/log/setup-$${HOSTNAME}.log"

    exec > >(tee -a "$${LOG_FILE}") 2>&1

    echo "Installing app version $${APP_VERSION}..."
    yum install -y myapp-$${APP_VERSION}
    systemctl enable --now myapp
  SCRIPT
}
```

### Nginx Configuration

```hcl
resource "local_file" "nginx_conf" {
  filename = "/etc/nginx/conf.d/app.conf"

  content = <<-EOT
    server {
        listen 80;
        server_name ${var.domain};

        location / {
            proxy_pass http://127.0.0.1:${var.app_port};
            proxy_set_header Host $${host};
            proxy_set_header X-Real-IP $${remote_addr};
        }
    }
  EOT
}
# $${host} and $${remote_addr} are nginx variables, not Terraform
```

### Multi-Line Local Values

```hcl
locals {
  motd = <<-EOT
    =============================================
    Environment: ${var.environment}
    Region:      ${var.region}
    Managed by:  Terraform
    =============================================
  EOT

  ssh_config = <<-EOT
    Host bastion
      HostName ${aws_instance.bastion.public_ip}
      User ec2-user
      IdentityFile ~/.ssh/id_rsa
      StrictHostKeyChecking no
  EOT
}
```

### Heredoc in Provisioners

```hcl
resource "null_resource" "setup" {
  provisioner "local-exec" {
    command = <<-EOT
      echo "Setting up ${var.cluster_name}..."
      aws eks update-kubeconfig \
        --name ${var.cluster_name} \
        --region ${var.region}
      kubectl apply -f manifests/
    EOT
  }
}
```

### Template Directives in Heredocs

```hcl
locals {
  hosts_file = <<-EOT
    # Managed by Terraform
    127.0.0.1 localhost

    %{ for name, ip in var.hosts ~}
    ${ip} ${name}
    %{ endfor ~}
  EOT
}
# The ~ after %{ trims the newline from the directive line
```

## Heredoc vs templatefile()

| Approach | Best For |
|----------|----------|
| Heredoc | Short inline strings, simple interpolation, IAM policies |
| `templatefile()` | Long scripts, complex logic, reusable templates, separation of concerns |

```hcl
# Heredoc — inline, simple
user_data = <<-EOF
  #!/bin/bash
  echo "${var.message}" > /tmp/hello.txt
EOF

# templatefile — external file, complex
user_data = templatefile("${path.module}/scripts/init.sh.tpl", {
  message     = var.message
  packages    = var.packages
  environment = var.environment
})
```

## Common Mistakes

### Closing Delimiter Must Be Alone on Its Line

```hcl
# BAD — closing delimiter has trailing content
content = <<EOT
hello
EOT something  # Error!

# BAD — closing delimiter is indented (with <<, not <<-)
content = <<EOT
hello
  EOT  # Error! Use <<-EOT if you want to indent the closing delimiter

# GOOD
content = <<EOT
hello
EOT

# GOOD — with <<- the closing delimiter can be indented
content = <<-EOT
  hello
EOT
```

### Accidental Interpolation

```hcl
# BAD — Terraform tries to interpolate ${PATH}
user_data = <<-EOF
  export PATH=${PATH}:/opt/bin
EOF
# Error: variable "PATH" not found

# GOOD — escape with $$
user_data = <<-EOF
  export PATH=$${PATH}:/opt/bin
EOF
```

### Unexpected Whitespace

```hcl
# With << (not <<-), the content includes ALL indentation
resource "x" "y" {
  content = <<EOT
    this line has 4 spaces of indentation in the output
EOT
}

# With <<- the common indentation is stripped
resource "x" "y" {
  content = <<-EOT
    this line has NO leading spaces in the output
  EOT
}
```

## Quick Reference

| Syntax | Behavior |
|--------|----------|
| `<<EOT` | Preserves all whitespace, delimiter must be at column 0 |
| `<<-EOT` | Strips common leading whitespace, delimiter can be indented |
| `${var.x}` | Terraform interpolation (evaluated at plan time) |
| `$${VAR}` | Literal `${VAR}` in output (escaped, for shell/nginx/etc.) |
| `$$(cmd)` | Literal `$(cmd)` in output |
| `%%{` | Literal `%{` in output (escape template directive) |
| `%{ for ... }` | Template directive (loop) |
| `%{ if ... }` | Template directive (conditional) |
| `~}` | Trim trailing newline from directive |
| `{~ ` | Trim leading newline from directive |
