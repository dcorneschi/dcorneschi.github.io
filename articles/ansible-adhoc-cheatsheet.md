# Ansible Ad-Hoc Commands Cheatsheet

Ad-hoc commands let you execute tasks on remote hosts without writing a playbook. They're ideal for one-off operations, quick checks, and live troubleshooting across your fleet.

## Basic Syntax

```bash
ansible <pattern> -m <module> -a "<arguments>" [options]
ansible <pattern> -i <inventory> -m <module> -a "<arguments>" [options]
```

## Common Options

| Option | Short | Description |
|--------|-------|-------------|
| `--become` | `-b` | Execute with sudo/privilege escalation |
| `--become-user=USER` | | Run as specific user (default: root) |
| `--ask-become-pass` | `-K` | Prompt for sudo password |
| `--ask-pass` | `-k` | Prompt for SSH password |
| `-i <inventory>` | | Specify inventory file |
| `-u <user>` | | Connect as this user |
| `--private-key=KEY` | | SSH private key file |
| `--forks=N` | `-f N` | Number of parallel processes (default: 5) |
| `--timeout=SECS` | `-T` | Connection timeout |
| `--limit SUBSET` | `-l` | Limit to specific hosts/groups |
| `--check` | `-C` | Dry run (no changes made) |
| `--diff` | `-D` | Show file differences |
| `--one-line` | `-o` | Condensed output (one line per host) |
| `-v` / `-vv` / `-vvv` / `-vvvv` | | Increase verbosity |
| `--list-hosts` | | Show matched hosts without executing |

## Host Patterns

```bash
# All hosts
ansible all -m ping

# List all hosts from inventory
ansible all --list-hosts
ansible-inventory --list
ansible-inventory --graph

# Specific group
ansible webservers -m ping

# Specific host
ansible web01.example.com -m ping

# Multiple hosts (comma-separated)
ansible web01,web02,db01 -m ping

# Multiple groups (union)
ansible webservers:databases -m ping

# Exclude hosts from a group
ansible webservers:!web02 -m ping

# Intersection of groups (hosts in BOTH groups)
ansible webservers:&production -m ping

# Wildcard patterns
ansible "web*" -m ping
ansible "192.168.1.*" -m ping

# Regex pattern (prefix with ~)
ansible ~web[0-9]+ -m ping

# Range of hosts
ansible webserver[1:5] -m ping

# First N hosts in a group
ansible webservers[0:2] -m ping

# Every Nth host
ansible webservers[::3] -m ping

# Limit execution to subset
ansible all --limit "web01,web02" -m ping
ansible all --limit @failed_hosts.txt -m ping

# List matched hosts without running anything
ansible webservers --list-hosts
```

## Connection & Authentication

```bash
# Specify inventory
ansible all -i inventory.ini -m ping
ansible all -i /etc/ansible/hosts -m ping

# Connect as specific user
ansible all -u ubuntu -m ping
ansible all -u root --ask-pass -m ping

# Use specific SSH key
ansible all --private-key=~/.ssh/id_rsa -m ping

# Use sudo
ansible all --become -m ping
ansible all --become --become-user=root -m ping
ansible all --become --ask-become-pass -m ping

# Combined options
ansible webservers -i inventory.ini -u admin --become \
  --private-key=~/.ssh/admin_key -m ping

# Enable SSH pipelining (faster)
ansible all -m ping -e 'ansible_ssh_pipelining=true'

# SSH connection multiplexing
ansible all -m ping -e 'ansible_ssh_args="-o ControlMaster=auto -o ControlPersist=60s"'
```

## Ping & Connectivity

```bash
# Basic connectivity test
ansible all -m ping

# Test specific group
ansible webservers -m ping

# Verbose ping
ansible all -m ping -v

# Test with specific user
ansible all -m ping -u ansible

# Test with sudo
ansible all -m ping -b

# Disable fact gathering (faster ping)
ansible all -m ping --gather-facts=no
```

## Command Execution

### command Module (Safe — No Shell Features)

```bash
# Simple commands (no pipes, no redirects, no shell expansion)
ansible all -m command -a "date"
ansible all -m command -a "whoami"
ansible all -m command -a "hostname"
ansible all -m command -a "uptime"
ansible all -m command -a "uname -a"
ansible all -m command -a "cat /etc/os-release"
ansible all -m command -a "ls -la /tmp"
ansible all -m command -a "du -sh /var/log"

# Change working directory
ansible all -m command -a "ls -la chdir=/var/log"

# Only run if file does NOT exist (creates)
ansible all -m command -a "touch /tmp/marker creates=/tmp/marker"

# Only run if file DOES exist (removes)
ansible all -m command -a "rm /tmp/marker removes=/tmp/marker"

# Run as different user
ansible all -m command -a "whoami" -b --become-user=nginx
```

### shell Module (Supports Pipes, Redirects, Variables)

```bash
# Commands with pipes
ansible all -m shell -a "ps aux | grep nginx | grep -v grep"
ansible all -m shell -a "ps aux | head -10"
ansible all -m shell -a "cat /proc/loadavg"
ansible all -m shell -a "netstat -tlnp | grep :80"
ansible all -m shell -a "ss -tlnp"

# Commands with redirects
ansible all -m shell -a "echo 'test' > /tmp/test.txt"
ansible all -m shell -a "cat /var/log/syslog | tail -20 > /tmp/last-logs.txt"

# Environment variables
ansible all -m shell -a "echo $HOME"
ansible all -m shell -a "APP_ENV=production /opt/app/deploy.sh" -b

# Complex shell commands
ansible all -m shell -a "df -h | grep -vE '^Filesystem|tmpfs|cdrom'"
ansible all -m shell -a "hostname && whoami && date"
ansible webservers -m shell -a "ps aux | grep nginx | grep -v grep | wc -l"

# Specify shell
ansible all -m shell -a "ps aux" -e "ansible_shell_executable=/bin/bash"
```

### script Module (Run Local Script Remotely)

```bash
# Copy and execute a local script on all remote hosts
ansible all -m script -a "./scripts/setup.sh"
ansible all -m script -a "./scripts/setup.sh" -b

# Execute on specific group
ansible webservers -m script -a "./scripts/deploy.sh" -b

# Script with arguments
ansible all -m script -a "./scripts/configure.sh arg1 arg2"

# Script with working directory
ansible all -m script -a "./scripts/install.sh chdir=/opt"

# Execute with specific interpreter
ansible all -m script -a "./scripts/setup.py" -a "executable=python3"
```

### raw Module (No Python Required on Target)

```bash
# Useful for bootstrapping systems without Python
ansible all -m raw -a "uptime"
ansible all -m raw -a "apt-get install -y python3" -b
ansible all -m raw -a "yum install -y python3" -b
```

## Package Management

### APT (Ubuntu/Debian)

```bash
# Update cache
ansible ubuntu -m apt -a "update_cache=yes" -b

# Install package
ansible ubuntu -m apt -a "name=nginx state=present" -b

# Install multiple packages
ansible ubuntu -m apt -a "name=nginx,git,vim,curl state=present update_cache=yes" -b

# Install specific version
ansible ubuntu -m apt -a "name=nginx=1.18.0-0ubuntu1 state=present" -b

# Upgrade package to latest
ansible ubuntu -m apt -a "name=nginx state=latest" -b

# Remove package
ansible ubuntu -m apt -a "name=apache2 state=absent" -b

# Purge package (remove + delete configs)
ansible ubuntu -m apt -a "name=apache2 state=absent purge=yes" -b

# Full system upgrade
ansible ubuntu -m apt -a "upgrade=dist update_cache=yes" -b

# Autoremove unused packages
ansible ubuntu -m apt -a "autoremove=yes" -b

# Clean package cache
ansible ubuntu -m apt -a "autoclean=yes" -b

# Install from .deb file
ansible ubuntu -m apt -a "deb=/tmp/package.deb state=present" -b
```

### YUM/DNF (RHEL/CentOS/Fedora)

```bash
# Install package
ansible centos -m yum -a "name=nginx state=present" -b

# Install multiple packages
ansible centos -m yum -a "name=git,vim,curl state=present" -b

# Install specific version
ansible centos -m yum -a "name=httpd-2.4.6 state=present" -b

# Update package
ansible centos -m yum -a "name=nginx state=latest" -b

# Update all packages
ansible centos -m yum -a "name=* state=latest" -b

# Remove package
ansible centos -m yum -a "name=httpd state=absent" -b

# Install from URL
ansible centos -m yum -a "name=https://example.com/package.rpm state=present" -b

# Install package group
ansible centos -m yum -a "name='@Development Tools' state=present" -b

# Clean cache + update
ansible centos -m yum -a "name=* state=latest update_cache=yes" -b

# Install a specific Red Hat errata/advisory
ansible all -m command -a "yum update -y --advisory RHBA-2019:3982" -b -k
ansible centos -m command -a "yum update -y --advisory RHBA-2019:3057 warn=no" -b
```

### package Module (Cross-Platform)

```bash
# Works on both apt and yum systems
ansible all -m package -a "name=git state=present" -b
ansible all -m package -a "name=vim state=latest" -b
ansible all -m package -a "name=httpd state=absent" -b
```

## Service Management

```bash
# Start service
ansible all -m service -a "name=nginx state=started" -b

# Stop service
ansible all -m service -a "name=apache2 state=stopped" -b

# Restart service
ansible all -m service -a "name=ssh state=restarted" -b

# Reload service
ansible all -m service -a "name=nginx state=reloaded" -b

# Enable at boot
ansible all -m service -a "name=nginx enabled=yes" -b

# Start + enable
ansible all -m service -a "name=nginx state=started enabled=yes" -b

# Disable at boot
ansible all -m service -a "name=apache2 enabled=no" -b

# Check status (via shell)
ansible all -m shell -a "systemctl status nginx" -b

# systemd-specific (supports daemon_reload)
ansible all -m systemd -a "name=nginx state=started" -b
ansible all -m systemd -a "name=nginx state=restarted daemon_reload=yes" -b
ansible all -m systemd -a "name=nginx enabled=yes" -b
```

## File Operations

### Copy (Local → Remote)

```bash
# Copy file
ansible all -m copy -a "src=/local/file.txt dest=/remote/file.txt"

# Copy with ownership and permissions
ansible all -m copy -a "src=index.html dest=/var/www/html/ owner=www-data group=www-data mode=0644" -b

# Copy with backup of existing file
ansible all -m copy -a "src=nginx.conf dest=/etc/nginx/nginx.conf backup=yes" -b

# Copy directory
ansible all -m copy -a "src=/local/configs/ dest=/etc/myapp/" -b

# Create file with inline content
ansible all -m copy -a "content='Hello World\n' dest=/tmp/hello.txt"

# Copy and validate before placing
ansible all -m copy -a "src=sshd_config dest=/etc/ssh/sshd_config validate='sshd -t -f %s' backup=yes" -b
```

### File Module (Manage File Properties)

```bash
# Create empty file
ansible all -m file -a "path=/tmp/test.txt state=touch"

# Create directory
ansible all -m file -a "path=/opt/myapp state=directory mode=0755" -b

# Create nested directories
ansible all -m file -a "path=/opt/myapp/config/ssl state=directory mode=0750" -b

# Remove file
ansible all -m file -a "path=/tmp/unwanted.txt state=absent"

# Remove directory
ansible all -m file -a "path=/tmp/olddir state=absent"

# Change permissions
ansible all -m file -a "path=/opt/script.sh mode=0755"

# Change ownership
ansible all -m file -a "path=/var/www/html owner=www-data group=www-data recurse=yes" -b

# Create symbolic link
ansible all -m file -a "src=/opt/myapp/current dest=/opt/myapp/live state=link"
```

### Fetch (Remote → Local)

```bash
# Download file from all remote hosts
ansible all -m fetch -a "src=/etc/hostname dest=/tmp/hostnames/"

# Fetch with flat structure (no host subdirectories)
ansible all -m fetch -a "src=/var/log/messages dest=/tmp/logs/ flat=yes"

# Fetch with checksum verification
ansible all -m fetch -a "src=/etc/nginx/nginx.conf dest=/tmp/configs/ validate_checksum=yes"
```

### File Content Editing

```bash
# Add a line to a file
ansible all -m lineinfile -a "path=/etc/hosts line='192.168.1.100 myserver' state=present" -b

# Remove a line matching pattern
ansible all -m lineinfile -a "path=/etc/hosts regexp='^192.168.1.100' state=absent" -b

# Replace text in file
ansible all -m replace -a "path=/etc/config regexp='old_value' replace='new_value'" -b

# Insert a block of text
ansible all -m blockinfile -a "path=/etc/hosts block='192.168.1.100 server1\n192.168.1.101 server2'" -b

# Template a file
ansible all -m template -a "src=nginx.conf.j2 dest=/etc/nginx/nginx.conf" -b
```

## User & Group Management

```bash
# Create user
ansible all -m user -a "name=john state=present" -b

# Create user with details
ansible all -m user -a "name=alice state=present create_home=yes shell=/bin/bash" -b

# Create user with specific UID
ansible all -m user -a "name=bob uid=1001 state=present" -b

# Add user to groups
ansible all -m user -a "name=charlie groups=sudo,docker append=yes" -b

# Set password (use vault in production!)
ansible all -m user -a "name=dave password={{ 'secretpass' | password_hash('sha512') }}" -b

# Lock user account
ansible all -m user -a "name=john password_lock=yes" -b

# Change shell
ansible all -m user -a "name=john shell=/bin/zsh" -b

# Remove user
ansible all -m user -a "name=olduser state=absent remove=yes" -b

# Create user with full details and disable password aging
ansible all -m user -a "name=daniel uid=501 create_home=yes shell=/bin/bash comment='Daniel'" -b -k
ansible all -m shell -a "chage -I -1 -m -1 -M -1 -W -1 daniel" -b -k

# Create group
ansible all -m group -a "name=developers state=present" -b

# Create group with GID
ansible all -m group -a "name=apps gid=1500 state=present" -b

# Remove group
ansible all -m group -a "name=oldgroup state=absent" -b

# Deploy SSH authorized key
ansible all -m authorized_key -a "user=ansible key='{{ lookup(\"file\", \"~/.ssh/id_rsa.pub\") }}'" -b
```

## System Information (Facts)

```bash
# Gather all facts
ansible all -m setup

# Filter specific facts
ansible all -m setup -a "filter=ansible_distribution*"
ansible all -m setup -a "filter=ansible_os_family"
ansible all -m setup -a "filter=ansible_memory_mb"
ansible all -m setup -a "filter=ansible_processor*"
ansible all -m setup -a "filter=ansible_mounts"
ansible all -m setup -a "filter=ansible_default_ipv4"
ansible all -m setup -a "filter=ansible_interfaces"
ansible all -m setup -a "filter=ansible_network*"

# Gather subset of facts (faster)
ansible all -m setup -a "gather_subset=network,hardware"
ansible all -m setup -a "gather_subset=!all"

# Save facts to files (one per host)
ansible all -m setup --tree /tmp/facts

# Custom facts directory
ansible all -m setup -a "fact_path=/etc/ansible/facts.d"
```

## Cron Jobs

```bash
# Add cron job
ansible all -m cron -a "name='backup logs' minute=0 hour=2 job='/opt/backup.sh'" -b

# Add cron job for specific user
ansible all -m cron -a "name='user backup' user=backup minute=30 hour=1 job='/home/backup/script.sh'" -b

# Remove cron job
ansible all -m cron -a "name='old backup' state=absent" -b

# List cron jobs
ansible all -m shell -a "crontab -l" -b
ansible all -m shell -a "crontab -l -u backup" -b
```

## Mount Operations

```bash
# Mount filesystem
ansible all -m mount -a "path=/mnt/data src=/dev/sdb1 fstype=ext4 state=mounted" -b

# Unmount
ansible all -m mount -a "path=/mnt/data state=unmounted" -b

# Add to fstab (without mounting now)
ansible all -m mount -a "path=/mnt/data src=/dev/sdb1 fstype=ext4 state=present" -b

# Check mounts
ansible all -m shell -a "df -h"
ansible all -m shell -a "mount | grep -v tmpfs"
```

## Archive Operations

```bash
# Create archive
ansible all -m archive -a "path=/var/log dest=/tmp/logs.tar.gz format=gz" -b

# Create zip
ansible all -m archive -a "path=/etc/nginx dest=/tmp/nginx-config.zip format=zip" -b

# Extract archive (already on remote)
ansible all -m unarchive -a "src=/tmp/archive.tar.gz dest=/opt/ remote_src=yes" -b

# Extract archive from control machine
ansible all -m unarchive -a "src=/local/archive.tar.gz dest=/opt/" -b

# Download and extract from URL
ansible all -m unarchive -a "src=https://example.com/file.tar.gz dest=/opt/ remote_src=yes" -b
```

## Download & URL Operations

```bash
# Download file
ansible all -m get_url -a "url=https://example.com/file.tar.gz dest=/tmp/file.tar.gz"

# Download with permissions
ansible all -m get_url -a "url=https://example.com/script.sh dest=/opt/script.sh mode=0755" -b

# Download with checksum
ansible all -m get_url -a "url=https://example.com/file.tar.gz dest=/tmp/file.tar.gz checksum=sha256:abc123..."

# Download with custom headers
ansible all -m get_url -a "url=https://api.example.com/data dest=/tmp/data.json headers='Authorization: Bearer token123'"

# Test URL connectivity
ansible all -m uri -a "url=https://google.com method=GET"
ansible all -m uri -a "url=https://api.example.com/health method=GET return_content=yes"

# Check port connectivity
ansible all -m wait_for -a "host=google.com port=443 timeout=10"
ansible all -m wait_for -a "port=22 timeout=1"
```

## Synchronize (rsync)

```bash
# Sync files from control machine to targets (push)
ansible all -m synchronize -a "src=/local/path/ dest=/remote/path/"

# Sync with delete (mirror — removes extra files on remote)
ansible all -m synchronize -a "src=/local/path/ dest=/remote/path/ delete=yes"

# Pull files from remote to local
ansible all -m synchronize -a "src=/remote/path/ dest=/local/path/ mode=pull"
```

## SSL/TLS Operations

```bash
# Generate self-signed certificate
ansible all -m openssl_certificate -a "path=/etc/ssl/certs/server.crt provider=selfsigned privatekey_path=/etc/ssl/private/server.key" -b

# Generate private key
ansible all -m openssl_privatekey -a "path=/etc/ssl/private/server.key size=2048" -b
```

## Git Operations

```bash
# Clone repository
ansible all -m git -a "repo=https://github.com/user/repo.git dest=/opt/repo"

# Clone specific branch
ansible all -m git -a "repo=https://github.com/user/repo.git dest=/opt/repo version=develop"

# Update to latest
ansible all -m git -a "repo=https://github.com/user/repo.git dest=/opt/repo version=HEAD"
```

## Firewall

```bash
# UFW (Ubuntu)
ansible ubuntu -m ufw -a "rule=allow port=22 proto=tcp" -b
ansible ubuntu -m ufw -a "rule=allow port=80 proto=tcp" -b
ansible ubuntu -m ufw -a "rule=allow port=443 proto=tcp" -b
ansible ubuntu -m ufw -a "state=enabled" -b

# Firewalld (RHEL/CentOS)
ansible centos -m firewalld -a "service=http state=enabled permanent=yes" -b
ansible centos -m firewalld -a "port=8080/tcp state=enabled permanent=yes" -b

# iptables
ansible all -m iptables -a "chain=INPUT protocol=tcp destination_port=22 jump=ACCEPT" -b

# Check status
ansible ubuntu -m shell -a "ufw status" -b
ansible centos -m shell -a "firewall-cmd --list-all" -b
```

## Database Operations

```bash
# MySQL — create database
ansible db -m mysql_db -a "name=mydb state=present" -b

# MySQL — create user
ansible db -m mysql_user -a "name=dbuser password=secret priv='mydb.*:ALL' state=present" -b

# MySQL — backup
ansible db -m mysql_db -a "name=mydb state=dump target=/tmp/mydb.sql" -b

# PostgreSQL — create database
ansible db -m postgresql_db -a "name=mydb" -b

# PostgreSQL — create user
ansible db -m postgresql_user -a "name=dbuser password=secret db=mydb priv=ALL" -b
```

## Process Management

```bash
# List processes
ansible all -m shell -a "ps aux | head -20"

# Find specific process
ansible all -m shell -a "ps aux | grep nginx"
ansible all -m shell -a "pgrep -f httpd"

# Kill process
ansible all -m shell -a "pkill nginx" -b
ansible all -m shell -a "killall httpd" -b

# Check listening ports
ansible all -m shell -a "ss -tlnp"
ansible all -m shell -a "netstat -tlnp" -b

# Check system load
ansible all -m shell -a "top -b -n1 | head -5"
ansible all -m shell -a "cat /proc/loadavg"
```

## Log Analysis

```bash
# System logs
ansible all -m shell -a "tail -20 /var/log/syslog" -b
ansible all -m shell -a "tail -20 /var/log/messages" -b

# Auth logs
ansible all -m shell -a "tail -20 /var/log/auth.log" -b
ansible all -m shell -a "grep 'Failed password' /var/log/auth.log | tail -5" -b

# Service logs
ansible webservers -m shell -a "tail -20 /var/log/nginx/error.log" -b

# Journald
ansible all -m shell -a "journalctl -n 20" -b
ansible all -m shell -a "journalctl -u nginx -n 10" -b

# Search for errors
ansible all -m shell -a "grep -i error /var/log/syslog | tail -10" -b

# Boot messages
ansible all -m shell -a "dmesg | tail -20"
```

## Reboot

```bash
# Reboot all systems
ansible all -m reboot -b

# Reboot with custom timeout
ansible all -m reboot -a "reboot_timeout=600" -b

# Reboot with message
ansible all -m reboot -a "msg='Scheduled maintenance reboot'" -b
```

## Reboot

```bash
# Reboot all systems
ansible all -m reboot -b

# Reboot with custom timeout
ansible all -m reboot -a "reboot_timeout=600" -b

# Reboot with message
ansible all -m reboot -a "msg='Scheduled maintenance reboot'" -b
```

## Practical Recipes

### System Health Check

```bash
ansible all -m shell -a "uptime && free -h && df -h /"
```

### Quick Security Audit

```bash
ansible all -m shell -a "grep 'Failed password' /var/log/auth.log | wc -l" -b
```

### Deploy SSH Key

```bash
ansible all -m authorized_key -a "user=ansible key='{{ lookup(\"file\", \"~/.ssh/id_rsa.pub\") }}'"
```

### Mass Service Restart

```bash
ansible webservers -m service -a "name=nginx state=restarted" -b
```

### Backup Config Files

```bash
ansible all -m fetch -a "src=/etc/nginx/nginx.conf dest=/backup/{{ inventory_hostname }}-nginx.conf flat=yes"
```

### Install → Configure → Start Pattern

```bash
ansible webservers -m yum -a "name=nginx state=present" -b && \
ansible webservers -m copy -a "src=nginx.conf dest=/etc/nginx/nginx.conf backup=yes" -b && \
ansible webservers -m service -a "name=nginx state=restarted enabled=yes" -b
```

### Copy Script → Execute → Clean Up

```bash
ansible all -m copy -a "src=./scripts/setup.sh dest=/tmp/setup.sh mode=0755" -b
ansible all -m shell -a "/tmp/setup.sh" -b
ansible all -m file -a "path=/tmp/setup.sh state=absent" -b
```

### Disk Cleanup

```bash
ansible ubuntu -m shell -a "apt autoremove -y && apt autoclean" -b
```

### Update SSH Config

```bash
ansible all -m lineinfile -a "path=/etc/ssh/sshd_config regexp='^PermitRootLogin' line='PermitRootLogin no'" -b
ansible all -m service -a "name=sshd state=restarted" -b
```

## Performance Tips

```bash
# Increase parallelism (default: 5)
ansible all -m ping -f 50

# Disable fact gathering (faster for simple commands)
# Ad-hoc commands don't gather facts by default (only setup module does)
# For playbooks: gather_facts: no
ansible all -m shell -a "uptime"

# Enable pipelining
ansible all -m ping -e 'ansible_ssh_pipelining=true'

# SSH connection multiplexing
ansible all -m ping -e 'ansible_ssh_args="-o ControlMaster=auto -o ControlPersist=60s"'
```

## Error Handling & Advanced Options

```bash
# Set custom timeout for connections
ansible all -m shell -a "long_command" --timeout=60

# Run long tasks asynchronously (background with polling)
ansible all -m shell -a "/opt/long-script.sh" -b --async=3600 --poll=10

# Fire and forget (poll=0 — don't wait for result)
ansible all -m shell -a "/opt/long-script.sh" -b --async=3600 --poll=0

# Log output to file (set via environment variable)
ANSIBLE_LOG_PATH=/tmp/ansible.log ansible all -m shell -a "uptime"

# Or configure in ansible.cfg:
# [defaults]
# log_path = /var/log/ansible.log
```

> **Note:** `ignore_errors` and `retries` are playbook-only directives. For ad-hoc commands, handle errors by examining exit codes or piping output.

## Inventory Patterns

```bash
# Standard inventory file
ansible -i inventory.ini webservers -m ping

# Dynamic inventory script
ansible -i inventory.py all -m setup

# Multiple inventory sources
ansible -i staging -i production all -m ping

# Limit to specific hosts from inventory
ansible all -m ping --limit webserver1,webserver2

# Limit using a file of hosts
ansible all -m ping --limit @failed_hosts.txt

# Target by inventory variable/tag
ansible tag_Environment_production -m ping
```

## Common Patterns

```bash
# Pattern: Install → Configure → Start
ansible webservers -m yum -a "name=nginx state=present" -b && \
ansible webservers -m copy -a "src=nginx.conf dest=/etc/nginx/nginx.conf backup=yes" -b && \
ansible webservers -m service -a "name=nginx state=restarted enabled=yes" -b

# Pattern: Backup before change (using remote_src)
ansible all -m copy -a "src=/etc/hosts dest=/etc/hosts.backup remote_src=yes" -b && \
ansible all -m lineinfile -a "path=/etc/hosts line='192.168.1.100 newhost'" -b

# Pattern: Conditional execution
ansible all -m shell -a "systemctl restart nginx" -b -e "ansible_os_family=='RedHat'"

# Pattern: Copy → Execute → Clean Up
ansible all -m copy -a "src=./scripts/setup.sh dest=/tmp/setup.sh mode=0755" -b
ansible all -m shell -a "/tmp/setup.sh" -b
ansible all -m file -a "path=/tmp/setup.sh state=absent" -b
```

## Real-World Script Execution

```bash
# System update script (limited parallelism to avoid overload)
ansible all -i inventory.ini -m script -a "./scripts/system-update.sh" -b --forks=3

# Log rotation on web servers only
ansible webservers -i inventory.ini -m script -a "./scripts/rotate-logs.sh" -b

# Database backup with extended timeout
ansible databases -i inventory.ini -m script -a "./scripts/db-backup.sh" -b --timeout=600

# Health check with condensed output
ansible all -i inventory.ini -m shell -a "systemctl status nginx mysql" -b --one-line

# Execute with environment variables
ansible all -m shell -a "APP_ENV=production ./scripts/deploy.sh" -b

# Conditional: only run if file exists
ansible all -m shell -a "test -f /etc/nginx/nginx.conf && /opt/scripts/nginx-check.sh" -b
```

## Tips

1. **Use `script` module** for local scripts you want to execute remotely
2. **Use `shell` module** for complex commands with pipes/redirects
3. **Use `command` module** for simple, secure command execution (no shell injection)
4. **Always test** with `--check` or on a single host first (`--limit host1`)
5. **Use `--limit`** to target specific hosts for testing
6. **Set appropriate timeouts** for long-running scripts (`--timeout=300`)
7. **Use verbosity flags** (`-v`, `-vv`, `-vvv`) for debugging
8. **Logging** — set `ANSIBLE_LOG_PATH=/path/to/file` before running commands for audit
9. **Background tasks** — use `--async` and `--poll` for long-running operations
10. **Security** — use `--ask-become-pass` instead of passwordless sudo in sensitive environments

## Debugging

```bash
# Verbose output levels
ansible all -m ping -v        # Show results
ansible all -m ping -vv       # Show task inputs
ansible all -m ping -vvv      # Show connection info
ansible all -m ping -vvvv     # Show everything (SSH debug)

# Dry run
ansible all -m copy -a "src=file dest=/remote/file" --check

# Dry run with diff
ansible all -m copy -a "src=file dest=/remote/file" --check --diff

# Test sudo access
ansible all -m shell -a "whoami" -b

# Check Python version on targets
ansible all -m shell -a "python3 --version"

# Save output to file
ansible all -m shell -a "uptime" > results.txt
ansible all -m shell -a "uptime" -o > results-oneline.txt
```

## Useful Aliases

```bash
# Add to ~/.bashrc or ~/.zshrc
alias a='ansible'
alias ap='ansible-playbook'
alias aping='ansible all -m ping'
alias afacts='ansible all -m setup'
alias auptime='ansible all -m shell -a "uptime"'
alias adf='ansible all -m shell -a "df -h"'
alias amem='ansible all -m shell -a "free -h"'
```

## Module Quick Reference

| Module | Purpose | Example |
|--------|---------|---------|
| `ping` | Test connectivity | `ansible all -m ping` |
| `command` | Run command (no shell) | `ansible all -m command -a "date"` |
| `shell` | Run command (with shell) | `ansible all -m shell -a "ps aux \| head"` |
| `script` | Run local script remotely | `ansible all -m script -a "./setup.sh"` |
| `raw` | Run without Python | `ansible all -m raw -a "uptime"` |
| `copy` | Copy files to remote | `ansible all -m copy -a "src=f dest=/tmp/f"` |
| `fetch` | Fetch files from remote | `ansible all -m fetch -a "src=/etc/hosts dest=./"` |
| `file` | Manage files/dirs | `ansible all -m file -a "path=/tmp/d state=directory"` |
| `apt` | APT packages | `ansible all -m apt -a "name=nginx state=present" -b` |
| `yum` | YUM packages | `ansible all -m yum -a "name=nginx state=present" -b` |
| `package` | Cross-platform packages | `ansible all -m package -a "name=git state=present" -b` |
| `service` | Manage services | `ansible all -m service -a "name=nginx state=started" -b` |
| `systemd` | Manage systemd units | `ansible all -m systemd -a "name=nginx state=started" -b` |
| `user` | Manage users | `ansible all -m user -a "name=john state=present" -b` |
| `group` | Manage groups | `ansible all -m group -a "name=devs state=present" -b` |
| `cron` | Manage cron jobs | `ansible all -m cron -a "name='job' job='/opt/run.sh'" -b` |
| `mount` | Manage mounts | `ansible all -m mount -a "path=/mnt src=/dev/sdb1 ..." -b` |
| `lineinfile` | Edit single line | `ansible all -m lineinfile -a "path=/etc/f line='text'" -b` |
| `replace` | Search and replace | `ansible all -m replace -a "path=/etc/f regexp='old' replace='new'" -b` |
| `blockinfile` | Insert text block | `ansible all -m blockinfile -a "path=/etc/f block='text'" -b` |
| `get_url` | Download files | `ansible all -m get_url -a "url=http://... dest=/tmp/"` |
| `uri` | HTTP requests | `ansible all -m uri -a "url=http://... method=GET"` |
| `git` | Git operations | `ansible all -m git -a "repo=url dest=/opt/repo"` |
| `archive` | Create archives | `ansible all -m archive -a "path=/var/log dest=/tmp/a.gz"` |
| `unarchive` | Extract archives | `ansible all -m unarchive -a "src=/tmp/a.gz dest=/opt/"` |
| `setup` | Gather facts | `ansible all -m setup` |
| `debug` | Debug output | `ansible all -m debug -a "msg='hello'"` |
| `wait_for` | Wait for condition | `ansible all -m wait_for -a "port=80 timeout=10"` |
| `reboot` | Reboot system | `ansible all -m reboot -b` |
