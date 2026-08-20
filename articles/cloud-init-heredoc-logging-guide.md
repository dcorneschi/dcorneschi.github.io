# Cloud-Init Heredoc and Logging Guide

Patterns for using heredocs in cloud-init user data scripts, capturing output to log files, and debugging cloud-init execution.

## Heredocs in runcmd

### Basic Heredoc to Create a File

```yaml
#cloud-config
runcmd:
  - |
    cat << 'EOF' > /etc/myapp/config.yml
    server:
      host: 0.0.0.0
      port: 8080
    database:
      host: localhost
      name: myapp
    EOF
```

### Heredoc with Variable Expansion

```yaml
#cloud-config
runcmd:
  - |
    HOSTNAME=$(hostname)
    INSTANCE_ID=$(cloud-init query instance-id)
    cat << EOF > /etc/motd
    ==========================================
    Hostname: ${HOSTNAME}
    Instance: ${INSTANCE_ID}
    Built:    $(date)
    ==========================================
    EOF
```

### Heredoc Without Variable Expansion (Quoted EOF)

```yaml
#cloud-config
runcmd:
  - |
    cat << 'EOF' > /usr/local/bin/health-check.sh
    #!/bin/bash
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health)
    if [ "$STATUS" != "200" ]; then
        echo "UNHEALTHY: HTTP $STATUS" | logger -t health-check
        exit 1
    fi
    EOF
    chmod +x /usr/local/bin/health-check.sh
```

### Appending with Heredoc

```yaml
#cloud-config
runcmd:
  - |
    cat << 'EOF' >> /etc/sysctl.d/99-custom.conf
    net.core.somaxconn = 65535
    net.ipv4.tcp_max_syn_backlog = 65535
    vm.swappiness = 10
    EOF
    sysctl --system
```

## Logging Patterns

### Redirect All Output to a Log File

```yaml
#cloud-config
runcmd:
  - |
    exec > /var/log/cloud-init-custom.log 2>&1
    set -x
    echo "=== Starting custom setup at $(date) ==="
    apt-get update
    apt-get install -y nginx
    systemctl enable nginx
    echo "=== Completed at $(date) ==="
```

### Log Individual Commands

```yaml
#cloud-config
runcmd:
  - echo "Starting package install" | tee -a /var/log/setup.log
  - apt-get update >> /var/log/setup.log 2>&1
  - apt-get install -y nginx >> /var/log/setup.log 2>&1
  - echo "Package install done" | tee -a /var/log/setup.log
```

### Structured Logging Function

```yaml
#cloud-config
runcmd:
  - |
    LOG_FILE="/var/log/cloud-init-custom.log"

    log() {
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
    }

    log "Starting instance configuration"
    log "Installing packages..."
    apt-get update -qq >> "$LOG_FILE" 2>&1
    apt-get install -y nginx certbot >> "$LOG_FILE" 2>&1
    log "Packages installed"

    log "Configuring nginx..."
    systemctl enable --now nginx >> "$LOG_FILE" 2>&1
    log "Configuration complete"
```

### Logging with Exit Status

```yaml
#cloud-config
runcmd:
  - |
    LOG="/var/log/cloud-init-custom.log"

    run_cmd() {
        echo "[$(date '+%H:%M:%S')] Running: $*" >> "$LOG"
        "$@" >> "$LOG" 2>&1
        local rc=$?
        if [ $rc -ne 0 ]; then
            echo "[$(date '+%H:%M:%S')] FAILED (exit $rc): $*" >> "$LOG"
        else
            echo "[$(date '+%H:%M:%S')] OK: $*" >> "$LOG"
        fi
        return $rc
    }

    run_cmd apt-get update
    run_cmd apt-get install -y docker.io
    run_cmd systemctl enable docker
    run_cmd systemctl start docker
```

## Cloud-Init Output Directive

The `output` directive in cloud-config controls where cloud-init sends its own stdout/stderr:

```yaml
#cloud-config
output:
  # All output to a file
  all: "| tee -a /var/log/cloud-init-output.log"

  # Or split init/config/final stages
  init: "> /var/log/cloud-init-init.log 2>&1"
  config: "> /var/log/cloud-init-config.log 2>&1"
  final:
    output: "| tee -a /var/log/cloud-init-final.log"
    error: "| tee -a /var/log/cloud-init-final-errors.log"
```

## Default Log Locations

| File | Contents |
|------|----------|
| `/var/log/cloud-init.log` | Detailed cloud-init execution log (debug) |
| `/var/log/cloud-init-output.log` | stdout/stderr from scripts |
| `/run/cloud-init/` | Runtime data (current boot) |
| `/var/lib/cloud/instance/` | Instance-specific data and semaphores |
| `/var/lib/cloud/data/` | Persistent metadata |

## write_files with Heredoc-Style Content

```yaml
#cloud-config
write_files:
  # Script with bash heredoc inside it
  - path: /usr/local/bin/setup-db.sh
    permissions: '0755'
    content: |
      #!/bin/bash
      set -euo pipefail
      exec > /var/log/db-setup.log 2>&1

      echo "Creating database..."
      mysql -u root << 'SQL'
      CREATE DATABASE IF NOT EXISTS myapp;
      CREATE USER IF NOT EXISTS 'appuser'@'localhost' IDENTIFIED BY 'secret';
      GRANT ALL ON myapp.* TO 'appuser'@'localhost';
      FLUSH PRIVILEGES;
      SQL

      echo "Database setup complete"

  # Systemd unit file
  - path: /etc/systemd/system/myapp.service
    content: |
      [Unit]
      Description=My Application
      After=network.target

      [Service]
      Type=simple
      ExecStart=/usr/local/bin/myapp
      Restart=always
      StandardOutput=append:/var/log/myapp.log
      StandardError=append:/var/log/myapp-error.log

      [Install]
      WantedBy=multi-user.target

  # Logrotate config
  - path: /etc/logrotate.d/cloud-init-custom
    content: |
      /var/log/cloud-init-custom.log {
          daily
          rotate 7
          compress
          missingok
          notifempty
      }
```

## Script with Full Logging Template

```yaml
#cloud-config
write_files:
  - path: /var/lib/cloud/scripts/per-instance/setup.sh
    permissions: '0755'
    content: |
      #!/bin/bash
      set -euo pipefail

      LOG="/var/log/instance-setup.log"
      exec > >(tee -a "$LOG") 2>&1

      echo "=========================================="
      echo "Instance Setup Started: $(date)"
      echo "Hostname: $(hostname)"
      echo "Instance ID: $(cloud-init query instance-id)"
      echo "=========================================="

      # Install packages
      echo "[STEP 1] Installing packages..."
      apt-get update -qq
      apt-get install -y nginx certbot python3-certbot-nginx

      # Configure
      echo "[STEP 2] Configuring nginx..."
      cat << 'NGINX' > /etc/nginx/sites-available/default
      server {
          listen 80 default_server;
          server_name _;
          root /var/www/html;
          index index.html;
      }
      NGINX
      systemctl restart nginx

      # Verify
      echo "[STEP 3] Verifying..."
      curl -s -o /dev/null -w "%{http_code}" http://localhost/ | grep -q 200 && \
          echo "OK: nginx responding" || echo "FAIL: nginx not responding"

      echo "=========================================="
      echo "Instance Setup Complete: $(date)"
      echo "=========================================="
```

## Debugging Cloud-Init

### View Logs

```bash
# Main cloud-init log
sudo cat /var/log/cloud-init.log

# Script output
sudo cat /var/log/cloud-init-output.log

# Tail in real-time during boot
sudo tail -f /var/log/cloud-init-output.log

# Filter for errors
sudo grep -i "error\|fail\|traceback" /var/log/cloud-init.log
```

### Check Status

```bash
# Overall status
cloud-init status
cloud-init status --long

# Wait for cloud-init to finish
cloud-init status --wait
```

### Validate User Data

```bash
# Check user data syntax before deploying
cloud-init schema --config-file userdata.yaml

# View rendered user data on a running instance
sudo cloud-init query userdata

# View what cloud-init received
sudo cat /var/lib/cloud/instance/user-data.txt
```

### Re-Run for Testing

```bash
# Re-run runcmd module
sudo cloud-init single --name runcmd --frequency always

# Clean and re-run everything (testing only)
sudo cloud-init clean --reboot
```

## Heredoc Content Invisible in Logs

### The Problem

`set -x` only traces the command itself, not heredoc content. When you run:

```bash
set -x
cat << 'EOF' > /etc/config.yml
database:
  host: localhost
  port: 5432
EOF
```

The log only shows:

```
+ cat
```

You don't see the actual content being written, making debugging difficult.

### Strategy 1: Echo Before/After (Minimal)

```bash
echo "Writing configuration to /etc/myapp/config.yml"
cat << 'EOF' > /etc/myapp/config.yml
server:
    listen 80;
    server_name example.com;
EOF
echo "Configuration written successfully"
```

### Strategy 2: Write Then Display (Recommended)

```bash
cat << 'EOF' > /etc/myapp/config.yml
database:
  host: localhost
  port: 5432
  name: myapp
EOF
echo "=== /etc/myapp/config.yml ==="
cat /etc/myapp/config.yml
echo "=== End ==="
```

This shows the exact file content in the logs, verifies the write succeeded, and reveals any variable interpolation issues.

### Strategy 3: Using tee (Simultaneous Write + Display)

```bash
cat << 'EOF' | tee /etc/systemd/system/myapp.service
[Unit]
Description=My Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/myapp

[Install]
WantedBy=multi-user.target
EOF
```

Content is displayed as it's written — visible in `/var/log/cloud-init-output.log`.

### Strategy 4: Capture into Variable First

```bash
CONFIG_CONTENT=$(cat << 'EOF'
database:
  host: localhost
  port: 5432
EOF
)
echo "Writing content:"
echo "$CONFIG_CONTENT"
echo "$CONFIG_CONTENT" > /etc/myapp/config.yml
```

### Strategy Comparison

| Strategy | Content Visible | Verifies Write | Complexity | Best For |
|----------|:--------------:|:--------------:|:----------:|----------|
| Echo Before/After | No | No | Simple | Quick operations |
| Write Then Display | Yes | Yes | Simple | Cloud-init debugging |
| tee | Yes | Implicit | Simple | Interactive scripts |
| Capture into Variable | Yes | No | Medium | Small configs |

### Recommended Pattern for Cloud-Init

```yaml
#cloud-config
runcmd:
  - |
    set -euo pipefail
    set -x

    cat << 'EOF' > /etc/myapp/config.yml
    database:
      host: ${DB_HOST}
      port: 5432
    EOF
    echo "=== /etc/myapp/config.yml ==="
    cat /etc/myapp/config.yml
    echo "=== End ==="

    cat << 'EOF' > /etc/systemd/system/myapp.service
    [Unit]
    Description=My Application
    After=network.target

    [Service]
    Type=simple
    ExecStart=/usr/local/bin/myapp
    Restart=always

    [Install]
    WantedBy=multi-user.target
    EOF
    echo "=== /etc/systemd/system/myapp.service ==="
    cat /etc/systemd/system/myapp.service
    echo "=== End ==="

    systemctl daemon-reload
    systemctl enable myapp
```

In `/var/log/cloud-init-output.log`, you'll see the full file content with proper formatting, making it obvious whether variables expanded correctly.

## Common Pitfalls

### Indentation in YAML

```yaml
#cloud-config
runcmd:
  # WRONG — heredoc content must be part of the | block
  - |
    cat << 'EOF' > /tmp/test.txt
  content here    # This is OUTSIDE the block because of indentation
    EOF

  # CORRECT — maintain consistent indentation
  - |
    cat << 'EOF' > /tmp/test.txt
    content here
    EOF
```

### Quoting EOF vs Unquoted

```yaml
#cloud-config
runcmd:
  # Quoted 'EOF' — NO variable expansion (write literal $HOSTNAME)
  - |
    cat << 'EOF' > /tmp/no-expand.txt
    Host is $HOSTNAME
    EOF

  # Unquoted EOF — variables ARE expanded
  - |
    cat << EOF > /tmp/expanded.txt
    Host is $HOSTNAME
    EOF
```

### Special Characters in YAML

```yaml
#cloud-config
runcmd:
  # Colons in content need the | block scalar
  - |
    echo "key: value" > /tmp/test.yaml

  # WRONG — YAML interprets the colon
  # - echo "key: value" > /tmp/test.yaml
```

### tee vs Redirect

```yaml
#cloud-config
runcmd:
  # tee writes to file AND stdout (visible in cloud-init-output.log)
  - echo "Setup starting" | tee /var/log/setup.log

  # Redirect only writes to file (silent in cloud-init-output.log)
  - echo "Setup starting" > /var/log/setup.log

  # tee -a appends instead of overwriting
  - echo "Step complete" | tee -a /var/log/setup.log
```

## Quick Reference

| Goal | Pattern |
|------|---------|
| Create file from heredoc | `cat << 'EOF' > /path/file` ... `EOF` |
| Expand variables in heredoc | `cat << EOF > /path/file` (unquoted) |
| Log all output | `exec > /var/log/file.log 2>&1` |
| Log + show on console | `exec > >(tee -a /var/log/file.log) 2>&1` |
| Timestamped logging | Function with `date` + `tee` |
| Redirect cloud-init output | `output:` directive in cloud-config |
| View script output | `/var/log/cloud-init-output.log` |
| View cloud-init debug | `/var/log/cloud-init.log` |
| Validate config | `cloud-init schema --config-file file.yaml` |
| Re-run module | `cloud-init single --name <module> --frequency always` |
