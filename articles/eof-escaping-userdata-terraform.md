# EOF Escaping in Userdata, Terraform, and Shell Scripts

## Overview

Here-documents (heredocs) are a common way to embed multi-line strings in shell scripts, cloud-init userdata, and Terraform configurations. A critical but often misunderstood detail is whether the delimiter is **quoted or unquoted** — this single difference controls whether variable expansion and command substitution happen inside the heredoc. Getting this wrong leads to empty variables, broken scripts, or unintended content in your userdata.

## The Problem: Multiple Interpretation Layers

The core challenge arises when you have multiple layers of interpretation:

1. **Terraform HCL parser** — interprets `${...}` as Terraform variables
2. **Shell script interpreter** — interprets `$VAR` as shell variables
3. **Nested heredocs** — may have their own escaping requirements
4. **Cloud-init** — processes userdata before passing to shell

Each layer tries to interpret special characters, leading to conflicts.

## Shell Heredoc Basics

A heredoc redirects a block of text to a command's stdin. The syntax is:

```bash
command <<DELIMITER
content
DELIMITER
```

The delimiter (commonly `EOF`, `END`, `HEREDOC`, etc.) marks the start and end of the block. The name itself doesn't matter — what matters is whether you **quote** it.

### Unquoted Delimiter — Variables ARE Expanded

When the delimiter is unquoted, the shell performs variable expansion, command substitution, and backslash interpretation inside the heredoc:

```bash
NAME="world"

cat <<EOF
Hello, $NAME
Today is $(date +%A)
A backslash-n: \n
EOF
```

Output:

```
Hello, world
Today is Monday
A backslash-n: \n
```

`$NAME` is replaced with its value, `$(date +%A)` executes the command, and `\n` remains literal (backslash only escapes `$`, `` ` ``, `\`, and newline in heredocs).

### Quoted Delimiter — Variables Are NOT Expanded

When the delimiter is quoted (single quotes, double quotes, or even a backslash on any character), the shell treats the entire heredoc as a literal string — no expansion, no substitution:

```bash
NAME="world"

cat <<'EOF'
Hello, $NAME
Today is $(date +%A)
A backslash-n: \n
EOF
```

Output:

```
Hello, $NAME
Today is $(date +%A)
A backslash-n: \n
```

Everything is preserved exactly as written.

### All Quoting Forms That Suppress Expansion

Any of these suppress expansion — they're all equivalent:

```bash
cat <<'EOF'     # Single-quoted — most common
cat <<"EOF"     # Double-quoted
cat <<\EOF      # Backslash on first character
cat <<EO\F      # Backslash on any character
```

The most readable and conventional choice is `<<'EOF'`.

### The <<- Variant (Strip Leading Tabs)

Adding a hyphen (`<<-`) strips leading **tabs** (not spaces) from the heredoc body and the closing delimiter. This allows indentation in scripts without affecting the output:

```bash
if true; then
	cat <<-EOF
	This line has no leading whitespace in output.
	Neither does this one.
	EOF
fi
```

This works with both quoted and unquoted delimiters:

```bash
	cat <<-'EOF'
	$NOT_EXPANDED
	EOF
```

> **Note:** The indentation must be **tabs**, not spaces. Editors that convert tabs to spaces will break `<<-` heredocs silently.

## The Problem in Userdata Scripts

EC2 userdata scripts (and cloud-init shell scripts) run on the instance. If your userdata generates configuration files using heredocs, you need to control which variables expand **at write time** (on the instance) versus **at template render time** (in Terraform, Packer, or wherever you build the userdata).

### Example: Writing a systemd Unit File

Suppose your userdata script creates a systemd unit that references an environment variable set at boot:

```bash
#!/bin/bash
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)

cat > /etc/systemd/system/myapp.service <<EOF
[Unit]
Description=My Application
After=network.target

[Service]
Environment=INSTANCE_ID=$INSTANCE_ID
ExecStart=/usr/local/bin/myapp --id $INSTANCE_ID
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

Here `<<EOF` (unquoted) is correct — we **want** `$INSTANCE_ID` to expand when the script runs on the instance, so the generated file contains the actual instance ID.

### Example: Writing a Script That Uses Its Own Variables

Now suppose your userdata writes a helper script that should contain literal `$1`, `$HOME`, or other variables:

```bash
#!/bin/bash

cat > /usr/local/bin/backup.sh <<'EOF'
#!/bin/bash
BACKUP_DIR="$HOME/backups"
mkdir -p "$BACKUP_DIR"
tar czf "$BACKUP_DIR/backup-$(date +%Y%m%d).tar.gz" "$1"
echo "Backed up $1 to $BACKUP_DIR"
EOF

chmod +x /usr/local/bin/backup.sh
```

Here `<<'EOF'` (quoted) is correct — we want the script file to contain literal `$HOME`, `$1`, and `$(date ...)` that will expand when `backup.sh` is executed later, not when the userdata runs.

### Mixing Both: Partial Expansion

Sometimes you need **some** variables to expand and others to stay literal. You have two options:

**Option 1: Unquoted heredoc with escaped variables**

```bash
#!/bin/bash
APP_VERSION="2.1.0"

cat > /usr/local/bin/run-app.sh <<EOF
#!/bin/bash
# APP_VERSION is baked in at instance creation time
VERSION="$APP_VERSION"

# These expand when the script runs
HOSTNAME=\$(hostname)
echo "Running version \$VERSION on \$HOSTNAME"
exec /opt/app/bin/app-\$VERSION
EOF
```

The generated file:

```bash
#!/bin/bash
# APP_VERSION is baked in at instance creation time
VERSION="2.1.0"

# These expand when the script runs
HOSTNAME=$(hostname)
echo "Running version $VERSION on $HOSTNAME"
exec /opt/app/bin/app-$VERSION
```

**Option 2: Quoted heredoc with variable injection via sed or envsubst**

```bash
#!/bin/bash
APP_VERSION="2.1.0"

cat > /usr/local/bin/run-app.sh <<'EOF'
#!/bin/bash
VERSION="__APP_VERSION__"
HOSTNAME=$(hostname)
echo "Running version $VERSION on $HOSTNAME"
exec /opt/app/bin/app-$VERSION
EOF

sed -i "s/__APP_VERSION__/$APP_VERSION/" /usr/local/bin/run-app.sh
```

Option 1 is simpler for a few variables. Option 2 is cleaner when the heredoc is large and you only need to inject one or two values.

## Terraform and Heredocs

Terraform adds another layer of complexity. You're now dealing with **two levels of interpretation**: Terraform's template engine (HCL interpolation) and the shell on the instance.

### Problem: Terraform vs Shell Variables

```hcl
resource "aws_instance" "example" {
  user_data = <<-EOF
    #!/bin/bash
    echo "Instance ID: ${aws_instance.example.id}"  # Terraform variable
    echo "Hostname: $(hostname)"                     # Shell command — passes through fine
    echo "User: $USER"                               # Shell variable — passes through fine
    echo "Home: ${HOME}"                             # PROBLEM! Terraform tries to interpolate
  EOF
}
```

**Issue**: Terraform interprets `${HOME}` as a Terraform expression, causing an error. Only the `${...}` syntax (dollar + brace) triggers Terraform interpolation. Bare `$USER` and `$(hostname)` pass through unchanged.

### Solution 1: Double Dollar Signs

```hcl
resource "aws_instance" "example" {
  user_data = <<-EOF
    #!/bin/bash
    echo "Instance ID: ${aws_instance.example.id}"  # Terraform variable
    echo "Hostname: $(hostname)"                     # Shell command — no escaping needed
    echo "User: $USER"                               # No braces — no escaping needed
    echo "Home: $${HOME}"                            # $$ escapes the ${} for shell
    export DB_HOST="${aws_db_instance.main.endpoint}"
    export APP_HOME="$${HOME}/app"
  EOF
}
```

**Rule**: Use `$$` before `{` to escape shell `${...}` expressions in Terraform heredocs. Bare `$VAR` and `$(cmd)` don't need escaping.

### Solution 2: Avoid Braces for Shell Variables

```hcl
resource "aws_instance" "example" {
  user_data = <<-EOF
    #!/bin/bash
    echo "User: $USER"                               # No braces — passes through
    echo "Home: $HOME"                               # No braces — passes through
    echo "Path: $${PATH}"                            # Braces needed — escape with $$
  EOF
}
```

Since Terraform only interprets `${...}`, you can often avoid escaping entirely by using the brace-less form `$VAR` instead of `${VAR}` in your shell scripts.

### Solution 3: templatefile() — The Recommended Approach

The `templatefile()` function processes a file with Terraform's template syntax. Variables use `${var}` notation — which conflicts with shell variable syntax.

**userdata.sh.tpl:**

```bash
#!/bin/bash
# Terraform variables — expanded at plan/apply time
APP_VERSION="${app_version}"
S3_BUCKET="${s3_bucket}"
REGION="${region}"

# Shell commands and variables — pass through without escaping
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
HOSTNAME=$(hostname -f)

echo "Deploying $APP_VERSION to $HOSTNAME in $REGION"
```

**main.tf:**

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.medium"

  user_data = templatefile("${path.module}/userdata.sh.tpl", {
    app_version = var.app_version
    s3_bucket   = var.s3_bucket
    region      = var.aws_region
  })
}
```

### The Dollar Sign Problem in templatefile()

Terraform's template engine interprets `${...}` as interpolation. When your shell script uses `${VARIABLE}` (with braces), you need to escape it for Terraform using `$$`:

```bash
# In a .tpl file:

# Terraform interpolation — single $
BUCKET="${s3_bucket}"

# Shell command substitution — NO escaping needed (parentheses, not braces)
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)

# Shell variable without braces — NO escaping needed
echo "Instance $INSTANCE_ID is running"

# Shell variable WITH braces — MUST escape with $$
echo "Instance $${INSTANCE_ID} is running"

# Shell parameter expansion — MUST escape with $$
echo "Default: $${VAR:-fallback}"
```

The rules:

| Syntax in .tpl file | Terraform sees | Result on instance |
|---------------------|----------------|--------------------|
| `${var_name}` | Interpolation | Value of the Terraform variable |
| `$${VARIABLE}` | Literal `${VARIABLE}` | `${VARIABLE}` (shell expands it) |
| `$(command)` | Literal (not a template sequence) | `$(command)` (shell executes it) |
| `$VARIABLE` | Literal (not a template sequence) | `$VARIABLE` (shell expands it) |
| `$${` | Escape sequence | Literal `${` in output |

> **Key insight:** Only `${...}` (dollar + open brace) triggers Terraform interpolation. `$(...)` (dollar + parenthesis) and bare `$VARIABLE` (dollar without brace) are NOT template sequences and pass through untouched. You only need `$$` escaping when using `${...}` syntax for shell variables.

### Heredocs Inside templatefile() Templates

When your userdata template itself contains heredocs, you're at three levels: Terraform template → shell script → generated file. This is where things get tricky:

**userdata.sh.tpl:**

```bash
#!/bin/bash

# Terraform variable
DB_HOST="${db_endpoint}"

# Write a config file using a shell heredoc
# Use quoted heredoc so shell variables stay literal
cat > /etc/myapp/config.env <<'CONF'
# These will be set at app runtime
APP_HOME=/opt/myapp
LOG_LEVEL=info
CONF

# Write another file using unquoted heredoc — shell expansion happens
cat > /etc/myapp/database.env <<CONF
# DB_HOST was set from Terraform, now baked into this file
DATABASE_URL=postgresql://app:secret@$${DB_HOST}:5432/mydb
INSTANCE=$(hostname)
CONF
```

After Terraform renders this template (with `db_endpoint = "db.example.com"`), the userdata script becomes:

```bash
#!/bin/bash

# Terraform variable
DB_HOST="db.example.com"

# Write a config file using a shell heredoc
cat > /etc/myapp/config.env <<'CONF'
# These will be set at app runtime
APP_HOME=/opt/myapp
LOG_LEVEL=info
CONF

# Write another file using unquoted heredoc — shell expansion happens
cat > /etc/myapp/database.env <<CONF
# DB_HOST was set from Terraform, now baked into this file
DATABASE_URL=postgresql://app:secret@${DB_HOST}:5432/mydb
INSTANCE=$(hostname)
CONF
```

When the script runs on the instance, `database.env` will contain:

```
DATABASE_URL=postgresql://app:secret@db.example.com:5432/mydb
INSTANCE=ip-10-0-1-42.ec2.internal
```

Note: `$(hostname)` didn't need escaping in the template because `$(` is not a Terraform template sequence. Only `${DB_HOST}` needed `$$` escaping because it uses brace syntax.

### Inline Heredocs in HCL (Without templatefile)

You can embed userdata directly in HCL using Terraform's heredoc syntax:

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.medium"

  user_data = <<-USERDATA
    #!/bin/bash
    APP_VERSION="${var.app_version}"
    INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)

    cat > /etc/app.conf <<'EOF'
    version=${var.app_version}
    instance=$INSTANCE_ID
    EOF
  USERDATA
}
```

**Problem:** Terraform interpolates `${var.app_version}` in both places — including inside the inner `<<'EOF'` heredoc, because Terraform processes the entire string before the shell ever sees it. The `<<'EOF'` quoting only matters to the shell, not to Terraform's HCL parser.

**Also a problem:** `$(curl ...)` is not a Terraform interpolation (no braces), so it passes through. But `$INSTANCE_ID` in the inner heredoc also passes through — which is correct here since we want the shell to expand it.

This gets confusing fast. For anything beyond trivial scripts, use `templatefile()` with a separate `.tpl` file.

### Nested Heredocs

#### The Challenge

When an outer script creates an inner script, you need different delimiter names and careful quoting:

```bash
# Outer script creates an inner script
cat <<'OUTER' > /usr/local/bin/setup.sh
#!/bin/bash
cat <<'INNER' > /etc/config.txt
Server: $HOSTNAME
User: $USER
INNER
OUTER
```

**Key**: Use different delimiter names (`OUTER` vs `INNER`) and quote both to prevent premature expansion.

#### Terraform with Nested Heredocs

```hcl
resource "aws_instance" "example" {
  user_data = <<-EOF
    #!/bin/bash
    
    # Create a systemd service file
    # Shell variables pass through — no braces, no escaping needed
    cat <<'SERVICE' > /etc/systemd/system/myapp.service
    [Unit]
    Description=My Application
    
    [Service]
    Environment="DB_HOST=$DB_HOST"
    ExecStart=/usr/local/bin/myapp
    SERVICE
    
    systemctl enable myapp
  EOF
}
```

### Terraform Heredoc: Escaping Template Sequences

Unlike shell heredocs, Terraform does **not** support quoting the heredoc delimiter to suppress interpolation. The `<<-'EOF'` syntax does not work in HCL — it will produce a parse error.

Instead, Terraform provides only two escape sequences inside heredocs:

| Sequence | Produces |
|----------|----------|
| `$${` | Literal `${` |
| `%%{` | Literal `%{` |

If you need to embed literal `${...}` in your Terraform heredoc (e.g., shell brace-expansion syntax), use `$${`:

```hcl
# Terraform interpolation IS active (always in heredocs)
user_data = <<-USERDATA
  echo "${var.greeting}"
  echo "$${SHELL_VAR}"
USERDATA
```

There is no way to disable template processing for an entire Terraform heredoc. If your script heavily uses `${...}` syntax, use `templatefile()` with a separate file where you can systematically escape with `$$`.

## Three Levels of Interpretation: A Mental Model

When working with Terraform + userdata + generated files, think of three stages:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Level 1: Terraform (plan/apply time)                                │
│                                                                     │
│ • Processes ${...} interpolations in HCL or templatefile()          │
│ • $${ becomes literal ${                                            │
│ • Produces the final userdata string                                │
├─────────────────────────────────────────────────────────────────────┤
│ Level 2: Shell (instance boot time)                                 │
│                                                                     │
│ • Executes the userdata script                                      │
│ • Expands $VAR, ${VAR}, $(cmd) in unquoted heredocs                 │
│ • Preserves everything literally in quoted heredocs (<<'EOF')       │
│ • Produces config files, scripts, etc.                              │
├─────────────────────────────────────────────────────────────────────┤
│ Level 3: Generated files (application runtime)                      │
│                                                                     │
│ • Scripts written by userdata are executed                          │
│ • Config files are read by applications                             │
│ • Variables expand (or don't) based on the consuming application    │
└─────────────────────────────────────────────────────────────────────┘
```

At each level, ask: "Who should expand this variable?"

| Who expands it | How to write it in .tpl |
|----------------|-------------------------|
| Terraform | `${terraform_var}` |
| Shell (userdata, no braces) | `$SHELL_VAR` (passes through, no escaping needed) |
| Shell (userdata, with braces) | `$${SHELL_VAR}` |
| Shell (command substitution) | `$(command)` (passes through, no escaping needed) |
| Nobody (literal in output) | Use `<<'EOF'` in the shell script |

## Common Escaping Scenarios

### Escaping Backticks in Terraform

```hcl
user_data = <<-EOF
  #!/bin/bash
  # Backticks work but are deprecated in favor of $()
  HOSTNAME=`hostname`
  
  # Preferred — modern command substitution
  HOSTNAME=$(hostname)
EOF
```

Backticks are not a Terraform syntax issue (they don't trigger interpolation), but `$()` is preferred as it's nestable, more readable, and avoids quoting ambiguities.

### Escaping Quotes

```hcl
user_data = <<-EOF
  #!/bin/bash
  # Quotes don't need escaping in Terraform heredocs
  echo "He said \"Hello\""           # Double quotes in double quotes (shell escaping)
  echo 'Single quotes work: "Hi"'   # Single quotes protect double quotes
  MESSAGE="It's working"            # Apostrophe in double quotes is fine
EOF
```

Terraform heredocs are delimited by the marker, not by quotes, so quote characters pass through literally.

### Escaping Backslashes

```hcl
user_data = <<-EOF
  #!/bin/bash
  # Backslashes are literal in Terraform heredocs — no escaping needed
  echo "Path: /usr/local/bin"
  
  # sed and regex work normally
  sed -i 's/old/new/g' file.txt
  
  # Regex with backslashes — written exactly as shell expects
  grep "\bword\b" file.txt
EOF
```

> **Note:** Terraform explicitly does NOT interpret backslash escape sequences in heredoc strings. Backslashes are passed through literally. This is different from quoted strings where `\\`, `\n`, `\t` etc. are interpreted.

### Escaping Curly Braces

```hcl
user_data = <<-EOF
  #!/bin/bash
  # Shell parameter expansion — must escape because of ${
  echo "$${VAR:-default}"           # $$ escapes the ${, shell sees ${VAR:-default}
  
  # Terraform variable
  echo "${var.environment}"         # Terraform interpolation
  
  # Shell variable without braces — no conflict, no escaping needed
  echo "User: $USER"               # Passes through to shell as-is
EOF
```

## Cloud-Init Specific Considerations

### Cloud-Config Format

```yaml
#cloud-config
write_files:
  - path: /usr/local/bin/setup.sh
    content: |
      #!/bin/bash
      echo "User: $USER"              # Shell variable (literal in YAML)
      echo "Home: $HOME"
    permissions: '0755'
```

In YAML's `|` literal block, variables are preserved as-is — no escaping needed.

### Terraform + Cloud-Init

```hcl
data "cloudinit_config" "example" {
  part {
    content_type = "text/cloud-config"
    content = <<-EOF
      #cloud-config
      write_files:
        - path: /etc/app/config.sh
          content: |
            export DB_HOST="${aws_db_instance.main.endpoint}"
            export REGION="${var.aws_region}"
            export USER_HOME="$${HOME}"
    EOF
  }
}
```

## Common Mistakes and Fixes

### Mistake 1: Forgetting to Quote the Heredoc Delimiter

```bash
#!/bin/bash
# WRONG — $1 and $BACKUP_DIR expand to empty strings (or current shell values)
cat > /usr/local/bin/backup.sh <<EOF
#!/bin/bash
tar czf /backups/backup-$(date +%Y%m%d).tar.gz $1
EOF

# CORRECT — all variables are literal in the generated script
cat > /usr/local/bin/backup.sh <<'EOF'
#!/bin/bash
tar czf /backups/backup-$(date +%Y%m%d).tar.gz $1
EOF
```

### Mistake 2: Not Escaping ${} in Terraform Templates

```bash
# WRONG in a .tpl file — Terraform tries to interpolate ${INSTANCE_ID}
echo "Running on ${INSTANCE_ID}"

# CORRECT — $$ tells Terraform to produce a literal ${
echo "Running on $${INSTANCE_ID}"

# NOTE: $(command) does NOT need escaping — only ${} does
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)  # Fine as-is
```

The error you'll see:

```
Error: Invalid reference
│ A reference to "INSTANCE_ID" is not valid in this context.
```

### Mistake 3: Double Expansion

```bash
#!/bin/bash
PASSWORD="s3cr3t\$pecial"  # Contains a literal dollar sign

# WRONG — unquoted heredoc tries to expand $pecial
cat > /etc/app/credentials <<EOF
DB_PASSWORD=$PASSWORD
EOF

# Result: DB_PASSWORD=s3cr3t (if $pecial is empty)

# CORRECT — escape the content, or use a quoted heredoc and sed
cat > /etc/app/credentials <<EOF
DB_PASSWORD=${PASSWORD}
EOF
```

Wait — this is actually correct because we **want** `$PASSWORD` to expand. The problem is the variable's **value** contains a `$`. Fix the assignment:

```bash
PASSWORD='s3cr3t$pecial'  # Single quotes — no expansion in assignment
```

### Mistake 4: Using ${} for Shell Variables in Terraform Heredocs

```hcl
# WRONG — Terraform interprets ${AZ} as a Terraform expression
user_data = <<-EOF
  #!/bin/bash
  REGION="${var.region}"
  AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
  echo "Region: ${var.region}, AZ: ${AZ}"   # <- Terraform error on ${AZ}
EOF
```

Terraform sees `${AZ}` and tries to resolve it as a Terraform expression. Fix:

```hcl
user_data = <<-EOF
  #!/bin/bash
  REGION="${var.region}"
  AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
  echo "Region: ${var.region}, AZ: $${AZ}"
EOF
```

Or simply avoid braces for the shell variable:

```hcl
  echo "Region: ${var.region}, AZ: $AZ"
```

### Mistake 5: Indentation With Spaces Instead of Tabs for <<-

```bash
# WRONG — spaces won't be stripped by <<-
if true; then
    cat <<-EOF
    This still has leading whitespace in the output
    EOF    # <-- this delimiter won't match (leading spaces, not tabs)
fi

# CORRECT — use tabs (shown as → here)
if true; then
→   cat <<-EOF
→   This has no leading whitespace in the output
→   EOF
fi
```

### Mistake 6: Unterminated Heredoc

```hcl
# Wrong - the closing delimiter must be on its own line
user_data = <<-EOF
  #!/bin/bash
  echo "test"
  EOF  # This won't match — "EOF" has leading spaces and trailing comment

# Fix - closing delimiter alone on its own line (Terraform strips leading whitespace with <<-)
user_data = <<-EOF
  #!/bin/bash
  echo "test"
  EOF
```

> **Note:** Terraform's `<<-` strips leading **spaces** (finding the line with the fewest leading spaces and removing that many from all lines). This is different from shell's `<<-` which only strips leading tabs.

### Mistake 7: Nested Heredoc Closes Outer Heredoc

```bash
# Wrong - same delimiter name
cat <<EOF
  cat <<EOF
    nested
  EOF
EOF

# Fix - use different delimiters
cat <<OUTER
  cat <<INNER
    nested
  INNER
OUTER
```

## Real-World Examples

### Rsyslog Configuration (Literal $ Required)

Rsyslog uses `$programname` as part of its own syntax — these must remain literal in the final config file:

```hcl
variable "app_name" {
  default = "myapp"
}

resource "aws_instance" "example" {
  user_data = <<-EOT
    #!/bin/bash

    # Single-quoted heredoc prevents bash from expanding $programname
    cat <<'EOF' > /etc/rsyslog.d/filter.conf
    if $programname == '${var.app_name}' then stop
    EOF
  EOT
}
```

Terraform interpolates `${var.app_name}` → `myapp` (because it processes the entire string first). The `<<'EOF'` then prevents bash from touching `$programname`. Result in the file:

```
if $programname == 'myapp' then stop
```

### Environment File with Mixed Variables

Some values are known at deploy time (Terraform), others at boot time (bash):

```hcl
variable "app_version" {
  default = "v1.2.3"
}

variable "environment" {
  default = "production"
}

resource "aws_instance" "app_server" {
  user_data = <<-EOT
    #!/bin/bash

    # Get runtime information
    INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
    PRIVATE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)

    # Unquoted heredoc — bash expands $INSTANCE_ID and $PRIVATE_IP
    cat <<EOF > /etc/app/.env
    # Set at deployment time (Terraform)
    APP_VERSION=${var.app_version}
    ENVIRONMENT=${var.environment}

    # Set at boot time (bash)
    INSTANCE_ID=$INSTANCE_ID
    PRIVATE_IP=$PRIVATE_IP
    STARTED_AT=$(date -Iseconds)
    EOF
  EOT
}
```

Result in `/etc/app/.env`:

```
APP_VERSION=v1.2.3
ENVIRONMENT=production
INSTANCE_ID=i-1234567890abcdef0
PRIVATE_IP=10.0.1.50
STARTED_AT=2024-01-17T20:30:00+00:00
```

### Docker Daemon Configuration (JSON File)

```hcl
variable "log_max_size" {
  default = "10m"
}

variable "log_max_file" {
  default = "3"
}

resource "aws_instance" "docker_host" {
  user_data = <<-EOT
    #!/bin/bash

    cat <<'EOF' > /etc/docker/daemon.json
    {
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "${var.log_max_size}",
        "max-file": "${var.log_max_file}"
      },
      "storage-driver": "overlay2"
    }
    EOF

    systemctl restart docker
  EOT
}
```

Terraform replaces the variables before the script runs. The `<<'EOF'` ensures no bash expansion happens on the JSON content. Result:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
```

### EKS Node Bootstrap

```hcl
variable "cluster_name" {
  default = "my-eks-cluster"
}

variable "cluster_endpoint" {
  default = "https://ABC123.gr7.us-east-1.eks.amazonaws.com"
}

resource "aws_launch_template" "eks_nodes" {
  user_data = base64encode(<<-EOT
    #!/bin/bash
    set -ex

    # Rsyslog filter — needs literal $ for rsyslog syntax
    cat <<'RSYSLOG' > /etc/rsyslog.d/00-containerd-filter.conf
    if $programname == 'containerd' then {
        if $msg contains 'unable to parse' then {
            stop
        }
    }
    RSYSLOG

    # Bootstrap EKS node
    /etc/eks/bootstrap.sh "${var.cluster_name}" \
      --apiserver-endpoint "${var.cluster_endpoint}"
  EOT
  )
}
```

## Decision Tree: Which Quoting to Use

```
Do you need Terraform variables in the generated file?
│
├─ YES → Use ${var.name} in the heredoc (Terraform processes it first)
│   │
│   └─ Do you also need literal $ characters preserved?
│       ├─ YES → Use single-quoted inner heredoc: cat <<'EOF'
│       └─ NO  → Use unquoted inner heredoc: cat <<EOF
│
└─ NO
    │
    └─ Do you need bash variables expanded in the file?
        ├─ YES → Use unquoted inner heredoc: cat <<EOF
        └─ NO  → Use single-quoted inner heredoc: cat <<'EOF'
```

## Packer Considerations

Packer's shell provisioner also interprets variables. Similar rules apply:

```hcl
provisioner "shell" {
  inline = [
    "echo '${var.app_version}' > /opt/app/version",
    "cat > /etc/profile.d/app.sh <<'EOF'",
    "export APP_HOME=/opt/app",
    "export PATH=$APP_HOME/bin:$PATH",
    "EOF"
  ]
}
```

For complex scripts, use `templatefile()` with Packer's `file` provisioner or the `shell` provisioner's `script` argument pointing to a rendered template.

## Debugging Tips

### Check Terraform Output

```bash
# After apply, inspect the computed user_data value
terraform console
> aws_instance.example.user_data
```

This shows what Terraform computed and sent to AWS (base64-encoded for aws_instance).

### Use terraform show

```bash
# After apply, inspect user_data from the state (note: may be base64-encoded)
terraform show -json | jq -r '.values.root_module.resources[] | select(.address=="aws_instance.example") | .values.user_data'
```

### Enable Cloud-Init Logging

```hcl
user_data = <<-EOF
  #!/bin/bash
  exec > >(tee /var/log/user-data.log)
  exec 2>&1
  
  echo "Starting userdata execution at $(date)"
  # Your script here
EOF
```

### Test Locally with Heredoc

Create a test file to verify shell behavior independently:

```bash
#!/bin/bash
cat <<-'EOF'
# Paste your Terraform heredoc content here
# Replace $$ with $ to test shell behavior
EOF
```

### Test with terraform console

```bash
# Render the template and inspect the output
terraform console <<< 'templatefile("userdata.sh.tpl", {app_version="1.0", region="us-east-1"})'
```

## Best Practices

### Choose the Right Approach

- **Simple scripts with `$VAR`**: No escaping needed — bare `$VAR` and `$(cmd)` pass through Terraform
- **Scripts using `${VAR}`**: Escape with `$${VAR}` in Terraform heredocs
- **Complex scripts**: Use `templatefile()` with a separate `.tpl` file

### Use templatefile() for Non-Trivial Userdata

It separates concerns and makes the template testable independently.

### Always Use <<'EOF' When Writing Files That Contain Shell Syntax

Unless you explicitly need expansion from the outer script.

### Use $$ Only When Needed in Terraform

You only need `$$` before `{` — when using `${...}` syntax for shell variables. Bare `$VAR` and `$(cmd)` don't conflict with Terraform's template engine.

### Test Incrementally

```bash
# Test your heredoc locally first
cat <<-'EOF'
  echo "Test: $USER"
EOF

# Then wrap in Terraform — verify with terraform console
terraform console <<< '"Test: $${HOME}"'
```

### Use Consistent Delimiters

```hcl
# Good - clear hierarchy with different names
user_data = <<-USERDATA
  cat <<'SCRIPT' > /tmp/script.sh
    echo "nested"
  SCRIPT
USERDATA

# Avoid - same delimiter name at multiple levels (confusing)
user_data = <<-EOF
  cat <<'EOF' > /tmp/script.sh
    echo "nested"
  EOF
EOF
```

### Validate Rendered Output

Run `terraform plan` and inspect the userdata with `terraform show` or:

```bash
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan | jq -r '.planned_values.root_module.resources[] | select(.type=="aws_instance") | .values.user_data' | base64 -d
```

### Test Userdata Locally

Render the template with test values and run it through `shellcheck`:

```bash
terraform console <<< 'templatefile("userdata.sh.tpl", {app_version="1.0", region="us-east-1"})' > /tmp/userdata.sh
shellcheck /tmp/userdata.sh
```

### Keep Heredocs Shallow

If you have heredocs inside heredocs inside templates, you've gone too far. Extract the inner content into a separate file and copy it with `write_files` (cloud-init) or a `file` provisioner (Packer).

### Comment the Intent

When mixing Terraform and shell variables, add comments explaining which level handles which variable:

```bash
# In a .tpl file:
# ${var} = Terraform template interpolation
# $${VAR} = escaped, becomes ${VAR} for shell
# $VAR = passes through to shell (no braces, no conflict)
# $(cmd) = passes through to shell (parentheses, no conflict)

# Expanded by Terraform at build time:
REGION="${region}"

# Expanded by shell at boot time (braces need escaping):
echo "Home is $${HOME}"

# Expanded by shell at boot time (no braces, no escaping needed):
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
```

## Quick Reference Table

| Context | Want expansion | Don't want expansion |
|---------|---------------|---------------------|
| Shell script | `<<EOF` | `<<'EOF'` |
| Shell script (specific var) | `$VAR` or `${VAR}` | `\$VAR` or `\${VAR}` |
| Terraform .tpl (Terraform var) | `${tf_var}` | N/A |
| Terraform .tpl (shell var, no braces) | `$SHELL_VAR` (passes through) | N/A |
| Terraform .tpl (shell var, with braces) | `$${SHELL_VAR}` (becomes `${SHELL_VAR}`) | N/A |
| Terraform .tpl (shell command sub) | `$(command)` (passes through) | N/A |
| Terraform HCL heredoc (Terraform var) | `${var.name}` | N/A |
| Terraform HCL heredoc (shell `${...}`) | `$${VARIABLE}` | N/A |

### By Context and Variable Type

| Context | Shell Variable | Terraform Variable | Command Substitution |
|---------|---------------|-------------------|---------------------|
| Terraform heredoc (`$VAR`) | `$VAR` (passes through) | N/A | `$(cmd)` (passes through) |
| Terraform heredoc (`${VAR}`) | `$${VAR}` | `${var.name}` | N/A |
| templatefile() (`$VAR`) | `$VAR` (passes through) | N/A | `$(cmd)` (passes through) |
| templatefile() (`${VAR}`) | `$${VAR}` | `${var_name}` | N/A |
| Plain shell script | `$VAR` or `${VAR}` | N/A | `$(cmd)` |
| Nested heredoc | Depends on quoting | Depends on outer | Depends on quoting |

## Summary

The core rule is simple: **quoted delimiters suppress expansion, unquoted delimiters allow it** — this applies to shell heredocs. In Terraform, there is no quoted-delimiter feature; instead you escape `${` as `$${` to prevent template interpolation. The complexity comes from having multiple layers — Terraform, the shell running userdata, and generated files — each with their own expansion rules. Keep these layers distinct in your mind, use `templatefile()` to separate Terraform interpolation from shell logic, and use `<<'EOF'` in your shell scripts whenever the heredoc content should be treated as literal text.

**Golden Rule**: Only `${...}` triggers Terraform interpolation. Escape it with `$${` when you need literal `${` for shell. Bare `$VAR` and `$(cmd)` pass through safely. Test your output with `terraform console` before applying.
