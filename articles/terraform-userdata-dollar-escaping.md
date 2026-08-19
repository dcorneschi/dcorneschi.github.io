# Escaping $ in Terraform Userdata and Cloud-Init

When using `$` in Terraform `user_data` blocks and cloud-init, you need to escape it because Terraform uses `${}` for interpolation. Terraform processes the string first, then passes it to the cloud provider — so anything that looks like `${...}` or `$(...)` gets interpreted by Terraform unless escaped.

## The Problem

Terraform's HCL parser treats `${...}` as variable interpolation. Shell scripts also use `$` for variables and command substitution. When you embed a shell script in Terraform's `user_data`, these two clash:

```hcl
# BAD — Terraform tries to interpolate ${HOME} and $(whoami)
user_data = <<-EOF
  #!/bin/bash
  echo "Home: ${HOME}"         # Error: variable "HOME" not found
  echo "User: $(whoami)"       # Error: unexpected token
EOF
```

## Escaping with $$

In Terraform heredocs, escape `$` with `$$` to produce a literal `$` in the output:

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"

  user_data = <<-EOF
    #!/bin/bash
    set -euo pipefail

    # Shell variables — escape with $$
    echo "Current user: $$(whoami)"
    echo "Hostname: $${HOSTNAME}"
    echo "Home directory: $${HOME}"

    # Shell variable assignment — escape with $$
    MY_VAR="hello"
    echo "Variable: $${MY_VAR}"

    # Environment variables — escape with $$
    export PATH="$${PATH}:/usr/local/bin"

    # Command substitution — escape with $$
    CURRENT_DATE=$$(date +%Y-%m-%d)
    IP_ADDRESS=$$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)

    # Terraform variables — do NOT escape (interpolated at plan time)
    echo "Environment: ${var.environment}"
    echo "Instance name: ${var.instance_name}"

    # Mixing both
    echo "Server $${HOSTNAME} deployed in ${var.environment} at $${CURRENT_DATE}"
  EOF
}
```

After Terraform processes this, the rendered script becomes:

```bash
#!/bin/bash
set -euo pipefail

echo "Current user: $(whoami)"
echo "Hostname: ${HOSTNAME}"
echo "Home directory: ${HOME}"

MY_VAR="hello"
echo "Variable: ${MY_VAR}"

export PATH="${PATH}:/usr/local/bin"

CURRENT_DATE=$(date +%Y-%m-%d)
IP_ADDRESS=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)

echo "Environment: production"
echo "Instance name: web-server"

echo "Server ${HOSTNAME} deployed in production at ${CURRENT_DATE}"
```

## Using templatefile() (Recommended for Complex Scripts)

For complex scripts, use `templatefile()` with an external file. Inside the template file, Terraform variables use `${var}` syntax and shell variables are written normally — no escaping needed for shell `$`.

### main.tf

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"

  user_data = templatefile("${path.module}/scripts/userdata.sh.tpl", {
    environment   = var.environment
    app_name      = var.app_name
    app_version   = var.app_version
    nfs_server    = var.nfs_server_ip
    packages      = join(" ", var.packages)
  })
}
```

### scripts/userdata.sh.tpl

```bash
#!/bin/bash
set -euo pipefail

# Terraform variables (interpolated by templatefile)
APP_NAME="${app_name}"
APP_VERSION="${app_version}"
ENVIRONMENT="${environment}"
NFS_SERVER="${nfs_server}"

# Shell variables — no escaping needed in template files
HOSTNAME=$(hostname)
CURRENT_USER=$(whoami)
IP_ADDRESS=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)

echo "Deploying $APP_NAME v$APP_VERSION on $HOSTNAME ($IP_ADDRESS)"

# Install packages
yum install -y ${packages}

# Mount NFS
mkdir -p /mnt/data
mount -t nfs $NFS_SERVER:/data /mnt/data

# Start application
systemctl enable --now $APP_NAME
```

**Key difference:** In template files, `${var_name}` is Terraform interpolation, and bare `$VAR` or `$(cmd)` is shell syntax — they don't conflict.

### Escaping in Template Files

If you need a literal `${` in the template output (rare), use `$${`:

```bash
# In .tpl file:
# This is a Terraform variable:
echo "${app_name}"

# This produces a literal ${...} in the output:
echo "$${SOME_SHELL_VAR_USING_BRACES}"
```

## Using user_data_base64

Encode the rendered template as base64 — some providers or AMIs expect base64-encoded userdata:

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"

  user_data_base64 = base64encode(templatefile("${path.module}/scripts/userdata.sh.tpl", {
    app_name    = var.app_name
    environment = var.environment
  }))
}
```

## Cloud-Init YAML in Terraform

Cloud-init uses YAML format. When embedding cloud-init YAML directly in Terraform, you still need to escape `$`:

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"

  user_data = <<-EOF
    #cloud-config
    package_update: true
    packages:
      - nginx
      - curl

    runcmd:
      - echo "User: $$(whoami)" > /tmp/user.txt
      - echo "Hostname: $${HOSTNAME}" >> /tmp/user.txt
      - export MY_VAR="test-${var.environment}"
      - systemctl enable --now nginx

    write_files:
      - path: /etc/app/config.env
        content: |
          APP_ENV=${var.environment}
          APP_PORT=${var.app_port}
          HOSTNAME=$${HOSTNAME}
  EOF
}
```

### Cloud-Init YAML with templatefile()

Better approach — keep cloud-init in a separate template:

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"

  user_data = templatefile("${path.module}/cloud-init.yaml.tpl", {
    environment = var.environment
    app_port    = var.app_port
    packages    = ["nginx", "curl", "jq"]
  })
}
```

```yaml
#cloud-config
package_update: true
packages:
%{ for pkg in packages ~}
  - ${pkg}
%{ endfor ~}

runcmd:
  - echo "Deployed in ${environment}" > /tmp/deploy-info.txt
  - echo "User: $(whoami)" >> /tmp/deploy-info.txt
  - systemctl enable --now nginx

write_files:
  - path: /etc/app/config.env
    content: |
      APP_ENV=${environment}
      APP_PORT=${app_port}
```

## Multipart Cloud-Init

For complex setups combining cloud-config YAML and shell scripts:

```hcl
data "cloudinit_config" "web" {
  gzip          = true
  base64_encode = true

  part {
    content_type = "text/cloud-config"
    content = templatefile("${path.module}/cloud-config.yaml.tpl", {
      packages = var.packages
    })
  }

  part {
    content_type = "text/x-shellscript"
    content = templatefile("${path.module}/scripts/setup.sh.tpl", {
      environment = var.environment
      app_name    = var.app_name
    })
  }
}

resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"
  user_data     = data.cloudinit_config.web.rendered
}
```

## Quick Reference

| What You Write | What Terraform Produces | When to Use |
|---------------|------------------------|-------------|
| `${var.name}` | Value of Terraform variable | Inject Terraform values |
| `$${VAR}` | `${VAR}` (shell variable) | Shell variable with braces |
| `$$(cmd)` | `$(cmd)` (command substitution) | Shell command substitution |
| `$$VAR` | `$VAR` (shell variable) | Shell variable without braces |
| `%%{if ...}` | `%{if ...}` (literal) | Escape template directive |
| `%%` | `%` (literal percent) | Literal percent sign in output |
| `\"` | `"` (literal quote) | Quote inside double-quoted HCL string |
| `\\` | `\` (literal backslash) | Backslash in output |
| `$$((expr))` | `$((expr))` | Shell arithmetic |

## Escaping Percent Signs

Percent signs start template directives (`%{ if ... }`). To produce a literal `%`, double it:

```hcl
user_data = <<-EOF
  #!/bin/bash
  echo "CPU usage: 50%%"
  echo "Disk usage: $$(df -h / | tail -1 | awk '{print $$5}')"
EOF
# Result: "CPU usage: 50%" and the actual disk percentage
```

## Quotes and Backslashes

```hcl
user_data = <<-EOF
  #!/bin/bash
  # Single quotes — safe, no escaping needed
  echo 'Hello World'

  # Backslashes — double them for literal backslash
  echo "Windows path: C:\\Users\\Admin"
  sed -i 's/\\t/    /g' /etc/config.txt
EOF
```

## Whitespace Control with %{~ ~}

The `~` modifier trims whitespace/newlines from template directives — prevents blank lines in output:

```hcl
locals {
  packages = ["nginx", "docker", "git"]
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  user_data = <<-EOF
    #!/bin/bash
    %{~ for package in local.packages ~}
    apt-get install -y ${package}
    %{~ endfor ~}
    echo "All packages installed"
  EOF
}
# Without ~, you'd get blank lines between each install command
```

## Common Patterns

### Package Installation

```hcl
user_data = <<-EOF
  #!/bin/bash
  yum update -y
  yum install -y ${join(" ", var.packages)}
  
  for pkg in $$(rpm -qa --last | head -5 | awk '{print $$1}'); do
    echo "Recently installed: $${pkg}"
  done
EOF
```

### JSON Configuration in Userdata

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  user_data = <<-EOF
    #!/bin/bash

    # Write JSON config using a quoted inner heredoc (no shell expansion inside)
    cat > /etc/app/config.json << 'JSONEOF'
    {
      "server": {
        "port": 8080,
        "host": "0.0.0.0"
      },
      "database": {
        "connection": "postgresql://DB_USER:DB_PASS@localhost:5432/mydb"
      }
    }
    JSONEOF

    # Replace placeholders with actual values
    DB_USER="appuser"
    DB_PASS="$$(aws secretsmanager get-secret-value --secret-id db-password --query SecretString --output text)"

    sed -i "s/DB_USER/$${DB_USER}/g" /etc/app/config.json
    sed -i "s/DB_PASS/$${DB_PASS}/g" /etc/app/config.json
  EOF
}
```

### Inner Heredoc (Nested)

Use a quoted inner heredoc (`<< 'DELIM'`) to prevent both Terraform and shell from expanding variables inside the block:

```hcl
user_data = <<-EOF
  #!/bin/bash

  # Quoted delimiter prevents shell expansion inside
  cat > /etc/myapp/config.conf << 'INNEREOF'
  [database]
  host=localhost
  port=5432
  user=$${DB_USER}
  password=$${DB_PASSWORD}
  INNEREOF

  chmod 600 /etc/myapp/config.conf
EOF
```

### Writing Config Files

```hcl
user_data = <<-EOF
  #!/bin/bash
  cat > /etc/app/config.yml << 'CONFIG'
  environment: ${var.environment}
  port: ${var.app_port}
  database:
    host: ${aws_db_instance.main.endpoint}
  CONFIG
EOF
```

Note: Using a quoted heredoc delimiter (`'CONFIG'`) inside the script prevents the inner shell from expanding variables — only Terraform interpolates.

### Conditional Script Sections

```hcl
user_data = <<-EOF
  #!/bin/bash
  %{ if var.install_monitoring }
  yum install -y cloudwatch-agent
  systemctl enable --now amazon-cloudwatch-agent
  %{ endif }
  
  %{ if var.mount_efs }
  yum install -y amazon-efs-utils
  mkdir -p /mnt/efs
  mount -t efs ${var.efs_id}:/ /mnt/efs
  %{ endif }
EOF
```

## Debugging Userdata

```bash
# Check cloud-init logs on the instance
cat /var/log/cloud-init.log
cat /var/log/cloud-init-output.log

# View the rendered userdata (from instance)
curl -s http://169.254.169.254/latest/user-data

# Check if cloud-init completed
cloud-init status

# Re-run cloud-init (for testing)
cloud-init clean
cloud-init init
```

### Verify in Terraform Before Applying

```bash
# Render the userdata locally to inspect
terraform console
> templatefile("scripts/userdata.sh.tpl", { environment = "dev", app_name = "myapp" })
```

## Multi-Layer Backslash Escaping

When a string passes through multiple interpretation layers, you need additional backslashes at each layer:

| Layers of Interpretation | Backslashes Needed | Example |
|--------------------------|-------------------|---------|
| 1 layer (YAML only) | `\` | `\"max [0-9]+\"` |
| 2 layers (Terraform string → YAML) | `\\` | `\\\"max [0-9]+\\\"` |
| 3 layers (Terraform → Shell → YAML) | `\\\\` | `\\\\\"max [0-9]+\\\\\"` |

### Visual Guide

```hcl
# ❌ Won't work — Terraform eats the backslash
data = "\"max [0-9]+\""
# Terraform sees: "max [0-9]+"  (backslashes consumed)

# ✅ Works — double backslash survives Terraform string processing
data = "\\\"max [0-9]+\\\""
# Terraform produces: \"max [0-9]+\"
# YAML/JSON then sees: "max [0-9]+"

# ✅ Also works — heredoc treats backslashes as literal
data = <<-EOF
  \"max [0-9]+\"
EOF
# Terraform preserves: \"max [0-9]+\"  (backslashes kept as-is)
```

### Regular String vs Heredoc Comparison

```hcl
# Regular string — Terraform processes escape sequences
output "string_test" {
  value = "\"test\""
}
# Output: "test"  (backslashes removed, quotes unescaped)

# Heredoc — backslashes are literal
output "heredoc_test" {
  value = <<-EOF
    \"test\"
  EOF
}
# Output: \"test\"  (backslashes preserved in output)
```

**Key takeaway:** Add one more backslash for each layer that will interpret/process the string. Heredocs and external files skip Terraform's string interpretation, so they need fewer escapes.

## What Heredocs DO and DON'T Interpret

### Heredocs DO Interpret

- `${...}` — Terraform variable interpolation
- `$$` — escaped dollar sign (becomes `$`)
- `%{...}` — template directives
- `%%{` — escaped template directive (becomes `%{`)

### Heredocs DON'T Interpret

- `\n`, `\t`, `\"` — these are literal characters, not escape sequences
- Regular backslashes are passed through unchanged
- `\\` stays as `\\` (not collapsed to `\` like in regular strings)

```hcl
# Test: heredoc with various characters
output "heredoc_demo" {
  value = <<-EOF
    Backslash-quote: \"test\"
    Dollar sign: $$HOME
    Interpolation: ${var.name}
    Literal newline marker: \n (this stays as backslash-n)
  EOF
}
# Result:
#   Backslash-quote: \"test\"   ← backslash preserved
#   Dollar sign: $HOME          ← $$ becomes $
#   Interpolation: my-value     ← interpolated
#   Literal newline marker: \n  ← NOT a newline, literal \n
```

## Proxmox Cloud-Init Examples

### Option 1: Using Heredoc (Recommended)

```hcl
resource "proxmox_virtual_environment_file" "cloud_init" {
  content_type = "snippets"
  datastore_id = "local"
  node_name    = "pve"

  source_raw {
    data = <<-EOF
      #cloud-config
      write_files:
        - path: /etc/docker/daemon.json
          content: |
            {
              "LogFilterPatterns": [
                "~unable to parse \"max [0-9]+\" as a uint from Cgroup file"
              ]
            }
          permissions: '0644'
        - path: /etc/profile.d/custom.sh
          content: |
            export PRICE="$$4.99"
            export HOME_DIR="$$HOME"
            echo "User: $$USER"
      
      runcmd:
        - echo "Current path: $$PATH"
        - systemctl restart docker
    EOF
    file_name = "cloud-init.yaml"
  }
}
```

### Option 2: Using External Template File

**cloud-init.yaml.tpl:**

```yaml
#cloud-config
write_files:
  - path: /etc/docker/daemon.json
    content: |
      {
        "LogFilterPatterns": [
          "~unable to parse \"max [0-9]+\" as a uint from Cgroup file"
        ]
      }
    permissions: '0644'

runcmd:
  - systemctl restart docker
```

**main.tf:**

```hcl
resource "proxmox_virtual_environment_file" "cloud_init" {
  source_raw {
    data      = templatefile("${path.module}/cloud-init.yaml.tpl", {})
    file_name = "cloud-init.yaml"
  }
}
```

### Option 3: Using Static File (No Interpolation)

```hcl
resource "proxmox_virtual_environment_file" "cloud_init" {
  source_raw {
    data      = file("${path.module}/cloud-init.yaml")
    file_name = "cloud-init.yaml"
  }
}
# file() reads content as-is — no interpolation, no escaping needed in the YAML
```

## Escaping Reference Table

| Character | In Regular String | In Heredoc | Result in Output |
|-----------|------------------|------------|------------------|
| `$` | `$$` | `$$` | `$` |
| `\` | `\\` | `\` (literal) | `\` |
| `"` | `\"` (unescapes) | `\"` (stays as `\"`) | `"` vs `\"` |
| `%` | `%%` | `%%` | `%` |
| `${var}` | `${var.x}` | `${var.x}` | Interpolated value |
| Literal `${` | `$${` | `$${` | `${` |

## Common Pitfalls

### Loops — Forgetting to Escape the Iterator

```hcl
# WRONG — Terraform tries to interpolate $i
user_data = <<-EOF
  for i in {1..5}; do
    echo "Iteration $i"
  done
EOF

# CORRECT
user_data = <<-EOF
  for i in {1..5}; do
    echo "Iteration $$i"
  done
EOF
```

### Arithmetic — Shell $(( )) Expressions

```hcl
# WRONG — Terraform sees $(( as interpolation
user_data = <<-EOF
  RESULT=$((5 + 3))
EOF

# CORRECT
user_data = <<-EOF
  RESULT=$$((5 + 3))
EOF
```

### AWS CLI with Command Substitution

```hcl
# WRONG — Terraform tries to interpolate $(aws ...)
user_data = <<-EOF
  INSTANCE_ID=$(aws ec2 describe-instances --query 'Reservations[0].Instances[0].InstanceId' --output text)
EOF

# CORRECT
user_data = <<-EOF
  INSTANCE_ID=$$(aws ec2 describe-instances --query 'Reservations[0].Instances[0].InstanceId' --output text)
EOF
```

## Userdata Logging Pattern

Always add logging to your userdata scripts — makes debugging much easier when things fail silently:

```hcl
user_data = <<-EOF
  #!/bin/bash
  set -x
  exec > >(tee /var/log/userdata.log) 2>&1

  echo "=== Userdata started at $$(date) ==="

  # Your script here
  yum update -y
  yum install -y ${join(" ", var.packages)}

  echo "=== Userdata completed at $$(date) ==="
EOF
```

## Testing Userdata

### Validate Syntax Locally

```bash
# Render the userdata and check bash syntax
terraform console <<< 'aws_instance.web.user_data' | base64 -d > test-userdata.sh
bash -n test-userdata.sh       # Check for syntax errors
shellcheck test-userdata.sh    # Lint the script
```

### Preview with terraform console

```bash
terraform console

# For inline heredoc userdata
> aws_instance.web.user_data

# For templatefile
> templatefile("scripts/userdata.sh.tpl", { environment = "dev", app_name = "myapp" })
```

### Check on the Instance

```bash
# View rendered userdata from inside the instance
curl -s http://169.254.169.254/latest/user-data

# Check cloud-init logs
cat /var/log/cloud-init-output.log
cat /var/log/userdata.log    # If you added the logging pattern above
```

## Best Practices

1. **Use `templatefile()` for complex scripts** — avoids escaping headaches entirely
2. **Use heredoc `<<-EOF`** for simple inline scripts (5-10 lines max)
3. **Escape `$` with `$$`** in heredocs for shell variables/commands
4. **Don't escape Terraform variables** — `${var.x}` should be interpolated
5. **Use `base64encode()`** when the provider expects base64 userdata
6. **Use `cloudinit_config`** data source for multipart userdata
7. **Test userdata** — check `/var/log/cloud-init-output.log` on the instance
8. **Use `terraform console`** to preview rendered templates before applying
9. **Prefer cloud-init modules** (packages, write_files, runcmd) over raw bash when possible
10. **Use quoted inner heredocs** (`<< 'EOF'`) to prevent shell from expanding variables in config files
