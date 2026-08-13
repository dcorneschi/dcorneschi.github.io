# Ansible Run Command Modules: command vs shell vs raw vs script

Ansible provides four modules for executing commands on remote hosts. Each has different capabilities and security characteristics. The general rule: **use run command modules as a last resort** — prefer purpose-built modules (`apt`, `service`, `file`, etc.) when they exist, as they're idempotent and self-documenting. When you must run a command, pick the safest module that meets your needs.

## When to Use Which

| Module | Shell Features | Python Required | Idempotent | Use Case |
|--------|---------------|-----------------|------------|----------|
| `command` | No | Yes | No (use `creates`/`removes`) | Simple commands — **safest choice** |
| `shell` | Yes (pipes, redirects, `$HOME`) | Yes | No (use `creates`/`removes`) | Commands needing shell features |
| `script` | Yes (local script) | No on target | No | Run a local script on remote hosts |
| `raw` | Depends on user's shell | No | No | Bootstrap systems without Python |

**Decision flow:**
1. Can a built-in module do it? → Use that module
2. Need pipes, redirects, or shell variables? → `shell`
3. Simple command with no shell features? → `command` (safer)
4. Need to run a local script remotely? → `script`
5. Target has no Python? → `raw`

## command Module (Safest)

The `command` module executes a command **without passing it through a shell**. This means:
- Shell variables like `$HOME`, `$USER` are NOT expanded
- Pipes (`|`), redirects (`>`, `>>`), and semicolons (`;`) don't work
- Glob patterns (`*.txt`) are NOT expanded
- No risk of shell injection attacks

This makes it the safest option for simple command execution.

### Basic Usage

```yaml
# Simple command
- name: Check disk space
  command: df -h /

# Command with arguments
- name: List files
  command: ls -la /opt/app

# Change working directory
- name: Run in specific directory
  command: ./migrate.sh
  args:
    chdir: /opt/app

# Only run if file does NOT exist (idempotent)
- name: Create marker file
  command: touch /opt/app/.initialized
  args:
    creates: /opt/app/.initialized

# Only run if file DOES exist (cleanup)
- name: Remove temp file
  command: rm /tmp/deploy.lock
  args:
    removes: /tmp/deploy.lock
```

### creates and removes (Idempotency)

Since `command` isn't inherently idempotent, use `creates` and `removes` to prevent unnecessary re-execution:

```yaml
# Only run the build if the binary doesn't exist yet
- name: Build application
  command: make install
  args:
    chdir: /opt/src/app
    creates: /usr/local/bin/myapp

# Only run cleanup if the temp directory exists
- name: Clean build artifacts
  command: rm -rf /tmp/build-output
  args:
    removes: /tmp/build-output
```

### Ad-Hoc

```bash
# command is the DEFAULT module — you can omit -m command
ansible all -a "uptime"
ansible all -a "df -h /"
ansible all -m command -a "hostname"
ansible all -m command -a "ls -la /tmp"

# With chdir
ansible all -m command -a "cat config.yml chdir=/opt/app"
```

### What Doesn't Work

```yaml
# FAILS — pipes don't work in command module
- command: ps aux | grep nginx

# FAILS — shell variables not expanded
- command: echo $HOME

# FAILS — redirects don't work
- command: ls /tmp > /tmp/listing.txt

# FAILS — glob not expanded
- command: rm /tmp/*.log

# Use shell module for all of the above
```

## shell Module

The `shell` module executes commands **through the remote user's shell** (`/bin/sh` by default). It supports:
- Pipes (`|`), redirects (`>`, `>>`), semicolons (`;`)
- Shell variables (`$HOME`, `$USER`, `$PATH`)
- Glob expansion (`*.txt`)
- Subshells, command substitution (`$(command)`)

### Basic Usage

```yaml
# Pipes and redirects
- name: Save process list
  shell: ps aux | grep nginx | grep -v grep > /tmp/nginx-procs.txt

# Shell variables
- name: Show user's home
  shell: echo $HOME

# Redirect and append
- name: Log deployment
  shell: echo "Deployed at $(date)" >> /var/log/deploy.log

# Complex command
- name: Find and remove old logs
  shell: find /var/log -name "*.log" -mtime +30 -delete
```

### Specify Shell and Working Directory

```yaml
- name: Run with bash (not /bin/sh)
  shell: |
    source /opt/app/env.sh
    /opt/app/start.sh
  args:
    executable: /bin/bash

- name: Run in specific directory
  shell: ls -lrt > temp.txt
  args:
    chdir: /root/ansible/shell_chdir_example
    executable: /bin/bash
```

### Register Output

```yaml
- name: Capture shell output
  shell: echo $USER
  register: command_result

- name: Display stdout
  debug:
    msg: "{{ command_result.stdout }}"

- name: Display full result (stdout, stderr, rc)
  debug:
    msg: "{{ command_result }}"

- name: Capture working directory
  shell: echo $PWD
  register: pwd_result

- name: Show just stdout
  debug:
    msg: "{{ pwd_result.stdout }}"
```

### creates and removes (Same as command)

```yaml
- name: Only run if output file doesn't exist
  shell: /opt/scripts/generate-report.sh > /tmp/report.txt
  args:
    creates: /tmp/report.txt
```

### Ad-Hoc

```bash
# Shell features available
ansible all -m shell -a "ps aux | grep nginx | wc -l"
ansible all -m shell -a "echo $HOME"
ansible all -m shell -a "df -h / | tail -1"
ansible all -m shell -a "cat /var/log/syslog | grep ERROR | tail -10"
```

### Risks

```yaml
# DANGEROUS — if variable contains shell metacharacters
- name: Risky — user input could contain injection
  shell: "grep {{ user_input }} /etc/passwd"
  # If user_input is "; rm -rf /" this is catastrophic

# SAFER — use command module when no shell features are needed
- name: Safe lookup
  command: "grep {{ user_input }} /etc/passwd"
  # command doesn't process shell metacharacters
```

## raw Module

The `raw` module executes a command over SSH **without any Ansible processing**. No Python is required on the target. Ansible doesn't do error checking, doesn't process the output, and returns STDOUT, STDERR, and return code as-is.

### When to Use

- Target has **no Python installed** (embedded systems, minimal containers, network devices)
- Bootstrapping a system (install Python so other modules can work)
- Managing devices with only SSH access (routers, switches, IoT)

### Basic Usage

```yaml
# Bootstrap: install Python on a fresh system
- name: Install Python (Debian/Ubuntu)
  raw: apt-get install -y python3
  become: yes

- name: Install Python (RHEL/CentOS)
  raw: yum install -y python3
  become: yes

# Manage minimal/embedded systems
- name: Check uptime on embedded device
  raw: uptime

# Run command on network device
- name: Show interface status
  raw: show interface brief
```

### Ad-Hoc

```bash
# Bootstrap Python on new systems
ansible all -m raw -a "apt-get install -y python3" -b
ansible all -m raw -a "yum install -y python3" -b

# Check systems without Python
ansible all -m raw -a "uptime"
ansible all -m raw -a "cat /etc/os-release"
```

### Characteristics

```yaml
# raw doesn't track changed status — always reports "changed"
# Use changed_when to suppress if needed
- name: Check something
  raw: cat /etc/hostname
  changed_when: false

# raw uses the user's default shell (not necessarily /bin/sh)
# There's no chdir, creates, or removes support
```

## script Module

The `script` module transfers a **local script** to the remote host and executes it there. The script runs on the remote machine but lives on the control node.

### When to Use

- You have a complex script that's easier to maintain as a separate file
- The script doesn't need to be permanently on the remote host
- Target doesn't need Python (script module uses raw transfer + execution)

### Basic Usage

```yaml
# Execute a local script on all remote hosts
- name: Run setup script
  script: scripts/setup.sh
  become: yes

# Script with arguments
- name: Run configure script
  script: scripts/configure.sh --env production --port 8080

# Only run if marker doesn't exist (idempotent)
- name: Initialize only once
  script: scripts/first-boot.sh
  args:
    creates: /opt/app/.initialized

# Script with specific interpreter
- name: Run Python script
  script: scripts/deploy.py
  args:
    executable: python3

# Run in specific directory
- name: Run build script
  script: scripts/build.sh
  args:
    chdir: /opt/src
```

### Ad-Hoc

```bash
# Copy and execute local script on all hosts
ansible all -m script -a "./scripts/health-check.sh"
ansible all -m script -a "./scripts/setup.sh" -b
ansible webservers -m script -a "./scripts/deploy.sh arg1 arg2" -b
```

## Error Handling with Run Command Modules

### failed_when (Custom Failure Conditions)

By default, a task fails when the command returns a non-zero exit code. Override this with `failed_when`:

```yaml
# Only fail if exit code is NOT 0 or 2 (2 means "already done")
- name: Run migration
  command: /opt/app/migrate.sh
  register: result
  failed_when: result.rc != 0 and result.rc != 2

# Fail based on output content
- name: Check service health
  shell: /opt/app/healthcheck.sh
  register: health
  failed_when: "'CRITICAL' in health.stdout"

# Never fail (equivalent to ignore_errors but more explicit)
- name: Optional cleanup
  command: rm /tmp/old-lock
  register: cleanup
  failed_when: false
```

### changed_when (Custom Change Detection)

Commands always report "changed" by default. Control this:

```yaml
# Never changed (read-only command)
- name: Check status
  command: systemctl is-active nginx
  register: nginx_status
  changed_when: false

# Changed only if output says so
- name: Apply configuration
  shell: /opt/app/apply-config.sh
  register: config_result
  changed_when: "'Configuration updated' in config_result.stdout"
```

### ignore_errors

```yaml
# Continue playbook even if this task fails
- name: Stop old service (might not exist)
  command: systemctl stop legacy-app
  ignore_errors: yes
```

## Comparison Examples

The same task implemented with each module:

```yaml
# COMMAND — safe, no shell, no pipes
- name: Get hostname
  command: hostname
  register: result

# SHELL — supports pipes and variables
- name: Get hostname and IP
  shell: "hostname && hostname -I | awk '{print $1}'"
  register: result

# RAW — no Python needed, no error handling
- name: Get hostname (no Python on target)
  raw: hostname
  register: result

# SCRIPT — execute a local file remotely
- name: Run info-gathering script
  script: scripts/gather-info.sh
  register: result
```

## Suppressing Warnings (warn Parameter)

In older Ansible versions (< 2.14), the `command` and `shell` modules warned you when running commands that a built-in module could handle (e.g., `apt-get install`, `chmod`). You could suppress with `warn: false`:

```yaml
# Ansible < 2.14 — suppress module usage warnings
- name: Run yum directly
  command: yum update -y --advisory RHBA-2024:1234
  args:
    warn: false
```

> **Note:** The `warn` parameter was **removed in Ansible Core 2.14** (ansible-core 2.14+). If you use it on newer versions, you'll get: `"Unsupported parameters for (ansible.legacy.command) module: warn"`. Simply remove the `warn: false` line — the warnings are no longer emitted.

## Multi-Line Shell Commands

Use YAML block scalars for complex multi-line scripts:

### Literal Block (`|`) — Preserves Newlines

```yaml
# Each line runs as a separate command in the shell
- name: Multi-step deployment
  shell: |
    cd /opt/app
    git pull origin main
    pip install -r requirements.txt
    systemctl restart myapp
  args:
    executable: /bin/bash
  become: yes

# With variables
- name: Configure and start service
  shell: |
    export APP_ENV={{ env }}
    export DB_HOST={{ db_host }}
    /opt/app/configure.sh
    /opt/app/start.sh
  args:
    executable: /bin/bash
```

### Folded Block (`>`) — Joins Lines (Single Command)

```yaml
# Lines are joined with spaces — useful for very long single commands
- name: Long curl command
  shell: >
    curl -s -X POST
    -H "Authorization: Bearer {{ api_token }}"
    -H "Content-Type: application/json"
    -d '{"status": "deployed", "version": "{{ app_version }}"}'
    https://api.example.com/deployments
```

### Heredoc Pattern

```yaml
# Create a file with a multi-line script, then execute
- name: Generate and run script
  shell: |
    cat << 'EOF' > /tmp/setup.sh
    #!/bin/bash
    set -euo pipefail
    echo "Starting setup..."
    apt-get update
    apt-get install -y nginx
    systemctl enable nginx
    echo "Setup complete."
    EOF
    chmod +x /tmp/setup.sh
    /tmp/setup.sh
  args:
    executable: /bin/bash
  become: yes
```

## Async and Poll (Long-Running Commands)

For commands that take a long time (backups, upgrades, builds), use `async` to prevent SSH timeout:

### Wait for Completion (Poll > 0)

```yaml
# Run for up to 1 hour, check every 30 seconds
- name: Full system upgrade (may take a while)
  shell: apt-get dist-upgrade -y
  async: 3600        # Maximum runtime in seconds
  poll: 30           # Check every 30 seconds
  become: yes

# Database backup with extended timeout
- name: Backup database
  command: /opt/scripts/full-backup.sh
  async: 7200        # Allow up to 2 hours
  poll: 60           # Check every minute
  become: yes
```

### Fire and Forget (Poll = 0)

```yaml
# Start and don't wait — move to next task immediately
- name: Start long data migration
  shell: /opt/scripts/migrate-data.sh >> /var/log/migration.log 2>&1
  async: 86400       # Allow up to 24 hours
  poll: 0            # Don't wait
  register: migration_job
  become: yes

# Check on it later
- name: Wait for migration to complete
  async_status:
    jid: "{{ migration_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 120
  delay: 60          # Check every 60 seconds, up to 120 times
```

### Ad-Hoc Async

```bash
# Run long command in background, poll every 10 seconds
ansible all -m shell -a "/opt/scripts/upgrade.sh" -b --async=3600 --poll=10

# Fire and forget (poll=0)
ansible all -m shell -a "/opt/scripts/reindex.sh" -b --async=7200 --poll=0
```

### When to Use Async

| Scenario | async | poll | Pattern |
|----------|-------|------|---------|
| Long upgrade/backup | 3600+ | 30-60 | Wait with periodic check |
| Background job (don't care about result) | large | 0 | Fire and forget |
| Parallel long tasks on same host | large | 0 + async_status | Start all, wait after |
| Normal command (< 30s) | Don't use | — | Default behavior |

## Best Practices

1. **Prefer built-in modules** — `apt`, `yum`, `service`, `file`, `template` are idempotent and self-documenting
2. **Use `command` over `shell`** when no shell features are needed — eliminates injection risk
3. **Always add `creates`/`removes`** to make command/shell tasks idempotent
4. **Set `changed_when: false`** for read-only commands (checks, queries)
5. **Use `failed_when`** for commands with non-standard exit codes
6. **Use `raw` only for bootstrapping** or Python-less targets
7. **Use `script` for complex logic** that's hard to express as inline shell
8. **Register output** when you need to make decisions based on command results
9. **Quote variables** in shell to prevent injection: `shell: "grep '{{ safe_var }}' /etc/file"`
10. **Document why** you're using a run command module instead of a purpose-built one

## Quick Reference

| Feature | `command` | `shell` | `raw` | `script` |
|---------|-----------|---------|-------|----------|
| Pipes (`\|`) | No | Yes | Depends | N/A |
| Redirects (`>`) | No | Yes | Depends | N/A |
| Shell variables | No | Yes | Depends | N/A |
| `creates`/`removes` | Yes | Yes | No | Yes |
| `chdir` | Yes | Yes | No | Yes |
| `executable` | No | Yes | No | Yes |
| Python required | Yes | Yes | No | No |
| Default module | Yes (`-a`) | No | No | No |
| Injection safe | Yes | No | No | N/A |
| Error handling | rc-based | rc-based | minimal | rc-based |
