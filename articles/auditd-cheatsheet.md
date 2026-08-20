# Linux Audit (auditd) Cheatsheet

The Linux Audit system (`auditd`) tracks security-relevant events on the system — file access, system calls, user logins, and privilege escalation. Essential for compliance (PCI-DSS, HIPAA, SOX) and security monitoring.

## Installation and Status

```bash
# Install (usually pre-installed on RHEL-based systems)
sudo dnf install -y audit audit-libs    # RHEL/CentOS/Fedora
sudo apt install -y auditd audispd-plugins    # Ubuntu/Debian

# Enable and start
sudo systemctl enable --now auditd

# Check status
sudo systemctl status auditd
sudo auditctl -s

# Check audit daemon config
cat /etc/audit/auditd.conf
```

## Core Components

| Component | Purpose |
|-----------|---------|
| `auditd` | Daemon that writes audit records to disk |
| `auditctl` | Control the audit system (add/delete rules, check status) |
| `ausearch` | Search audit logs |
| `aureport` | Generate summary reports from audit logs |
| `augenrules` | Merge rules from `/etc/audit/rules.d/` and load them |
| `autrace` | Trace a process (similar to strace using audit) |
| `aulast` | Like `last` but from audit logs |
| `aulastlog` | Like `lastlog` but from audit logs |

## auditctl — Manage Rules

### Check Current Rules

```bash
# List all active rules
sudo auditctl -l

# Check audit system status
sudo auditctl -s

# Count loaded rules
sudo auditctl -l | wc -l
```

### File/Directory Watches

```bash
# Watch a file for any access
sudo auditctl -w /etc/passwd -p rwxa -k passwd_changes

# Watch a directory recursively
sudo auditctl -w /etc/ssh/ -p rwxa -k ssh_config

# Watch for read access only
sudo auditctl -w /etc/shadow -p r -k shadow_read

# Watch for write/append
sudo auditctl -w /var/log/secure -p wa -k log_tamper

# Watch for attribute changes
sudo auditctl -w /etc/sudoers -p a -k sudoers_change

# Watch for execution
sudo auditctl -w /usr/bin/passwd -p x -k passwd_exec
```

Permission flags:

| Flag | Meaning |
|------|---------|
| `r` | Read |
| `w` | Write |
| `x` | Execute |
| `a` | Attribute change (chmod, chown, etc.) |

### System Call Rules

```bash
# Monitor all execve calls (command execution) by a specific user
sudo auditctl -a always,exit -F arch=b64 -S execve -F uid=1000 -k user_commands

# Monitor file deletions
sudo auditctl -a always,exit -F arch=b64 -S unlink -S unlinkat -S rename -S renameat -k file_delete

# Monitor mount/unmount operations
sudo auditctl -a always,exit -F arch=b64 -S mount -S umount2 -k mount_ops

# Monitor time changes
sudo auditctl -a always,exit -F arch=b64 -S adjtimex -S settimeofday -S clock_settime -k time_change

# Monitor user/group changes
sudo auditctl -a always,exit -F arch=b64 -S setuid -S setgid -S setreuid -S setregid -k priv_escalation

# Monitor module loading
sudo auditctl -a always,exit -F arch=b64 -S init_module -S delete_module -S finit_module -k kernel_modules

# Monitor network connections
sudo auditctl -a always,exit -F arch=b64 -S connect -S accept -S bind -k network_connections

# Monitor all commands by root
sudo auditctl -a always,exit -F arch=b64 -S execve -F euid=0 -k root_commands
```

### Delete Rules

```bash
# Delete a specific watch rule
sudo auditctl -W /etc/passwd -p rwxa -k passwd_changes

# Delete all rules
sudo auditctl -D

# Delete all rules and reset
sudo auditctl -D -k
```

### Temporary vs Permanent Rules

```bash
# Rules added with auditctl are temporary (lost on reboot)
# To make permanent, add to rules files:

# RHEL 7+: use rules.d directory
sudo vi /etc/audit/rules.d/custom.rules

# Or the main rules file
sudo vi /etc/audit/audit.rules

# Load rules from files
sudo augenrules --load

# Check for rule errors
sudo augenrules --check
```

## Permanent Rules File

```bash
# /etc/audit/rules.d/99-custom.rules

# Delete all existing rules first
-D

# Set buffer size
-b 8192

# Failure mode (0=silent, 1=printk, 2=panic)
-f 1

# File watches
-w /etc/passwd -p rwxa -k identity
-w /etc/shadow -p rwxa -k identity
-w /etc/group -p rwxa -k identity
-w /etc/gshadow -p rwxa -k identity
-w /etc/sudoers -p rwxa -k sudoers
-w /etc/sudoers.d/ -p rwxa -k sudoers

# SSH configuration
-w /etc/ssh/sshd_config -p rwxa -k sshd_config

# Cron
-w /etc/crontab -p rwxa -k cron
-w /etc/cron.d/ -p rwxa -k cron
-w /var/spool/cron/ -p rwxa -k cron

# System calls — command execution
-a always,exit -F arch=b64 -S execve -F euid=0 -k root_commands

# File deletions
-a always,exit -F arch=b64 -S unlink -S unlinkat -S rename -S renameat -k file_delete

# Privilege escalation
-a always,exit -F arch=b64 -S setuid -S setgid -k priv_esc

# Module loading
-a always,exit -F arch=b64 -S init_module -S finit_module -S delete_module -k modules

# Time changes
-a always,exit -F arch=b64 -S adjtimex -S settimeofday -S clock_settime -k time_change

# Make audit configuration immutable (requires reboot to change)
-e 2
```

```bash
# Load the rules
sudo augenrules --load

# Verify
sudo auditctl -l
```

## ausearch — Search Audit Logs

### Basic Searches

```bash
# Search by key
sudo ausearch -k passwd_changes

# Search recent events (last 10 minutes)
sudo ausearch -ts recent

# Search today's events
sudo ausearch -ts today

# Search by time range
sudo ausearch -ts 09:00:00 -te 17:00:00
sudo ausearch -ts "12/01/2024" -te "12/31/2024"

# Search specific date and time
sudo ausearch -ts "2024-06-15 09:00:00"
```

### Filter by Type

```bash
# Login events
sudo ausearch -m USER_LOGIN

# Failed logins
sudo ausearch -m USER_LOGIN --success no

# Authentication events
sudo ausearch -m USER_AUTH

# All user session events
sudo ausearch -m USER_START -m USER_END

# AVC (SELinux) denials
sudo ausearch -m AVC

# System calls
sudo ausearch -m SYSCALL

# Configuration changes
sudo ausearch -m CONFIG_CHANGE
```

### Filter by User/Process

```bash
# Search by user ID (any UID field)
sudo ausearch -ua 1000

# Search by user ID (specific UID field)
sudo ausearch -ui 1000

# Search by effective UID
sudo ausearch -ue 0

# Search by login UID (original user before su/sudo)
sudo ausearch -ul 1000

# Search by process ID
sudo ausearch -p 12345

# Search by executable
sudo ausearch -x /usr/bin/sudo

# Search by filename
sudo ausearch -f /etc/passwd

# Search by hostname
sudo ausearch -hn webserver01
```

### Output Formatting

```bash
# Interpret numeric values (UIDs → usernames, syscalls → names)
sudo ausearch -k passwd_changes -i

# Raw output
sudo ausearch -k passwd_changes --raw

# Extract raw logs for today (useful for forwarding/archiving)
sudo ausearch --start today --raw > audit-raw.log

# CSV output
sudo ausearch -k passwd_changes --format csv

# Text output
sudo ausearch -k passwd_changes --format text

# Only successful events
sudo ausearch -k passwd_changes --success yes

# Only failed events
sudo ausearch -k passwd_changes --success no
```

### Combining Filters

```bash
# Failed SSH logins today
sudo ausearch -m USER_LOGIN -ts today --success no -i

# Root commands in the last hour
sudo ausearch -k root_commands -ts recent -i

# File changes by a specific user today
sudo ausearch -k file_delete -ua 1000 -ts today -i

# SELinux denials for httpd
sudo ausearch -m AVC -c httpd -ts today
```

## aureport — Generate Reports

### Summary Reports

```bash
# Overall summary
sudo aureport

# Summary for today
sudo aureport -ts today

# Summary for a time range
sudo aureport -ts "06/01/2024" -te "06/30/2024"
```

### Specific Reports

```bash
# Authentication report
sudo aureport -au

# Failed authentication
sudo aureport -au --failed

# Login report
sudo aureport -l

# Failed logins
sudo aureport -l --failed

# User report (all user activity)
sudo aureport -u

# File access report
sudo aureport -f

# Executable report
sudo aureport -x

# System call report
sudo aureport -s

# Anomaly report
sudo aureport --anomaly

# Response to anomaly events report
sudo aureport -r

# Process ID report
sudo aureport -p

# Key report (events by key)
sudo aureport -k

# Terminal report
sudo aureport --tty

# Configuration change report
sudo aureport -c

# Crypto report (SSH, TLS)
sudo aureport --crypto
```

### Report with Interpretation

```bash
# Interpreted output (names instead of numbers)
sudo aureport -au -i

# Summary with node names
sudo aureport --summary

# Report in CSV format
sudo aureport -au --format csv
```

## autrace — Trace a Process

```bash
# Trace a command (like strace but using audit)
sudo autrace /bin/ls /tmp

# Trace and search results
sudo autrace /bin/ls /tmp
sudo ausearch -ts recent -p <PID_FROM_TRACE>

# Clean up autrace rules after tracing
sudo auditctl -D
sudo augenrules --load
```

## aulast / aulastlog

```bash
# Show login history from audit logs (like 'last')
sudo aulast

# Show bad (failed) logins
sudo aulast --bad

# Show last login for each user (like 'lastlog')
sudo aulastlog
```

## Log Management

### Log Files

```bash
# Default log location
/var/log/audit/audit.log

# Rotated logs
ls -la /var/log/audit/

# Real-time log monitoring
sudo tail -f /var/log/audit/audit.log
```

### Configuration (/etc/audit/auditd.conf)

```bash
# Key settings
log_file = /var/log/audit/audit.log
log_format = ENRICHED              # Include user/group names
num_logs = 5                       # Number of rotated logs
max_log_file = 50                  # Max size in MB
max_log_file_action = ROTATE       # What to do when full
space_left = 75                    # MB free before warning
space_left_action = SYSLOG         # Action on low space
admin_space_left = 50              # MB before admin action
admin_space_left_action = SUSPEND  # Suspend logging
disk_full_action = SUSPEND         # Action when disk full
disk_error_action = SUSPEND        # Action on write error
flush = INCREMENTAL_ASYNC          # Write strategy
```

### Rotate Logs Manually

```bash
# Rotate audit logs
sudo service auditd rotate

# Or on systemd systems
sudo kill -USR1 $(pidof auditd)
```

## Common Audit Events (Message Types)

| Type | Description |
|------|-------------|
| `SYSCALL` | System call event |
| `PATH` | File path in a syscall event |
| `CWD` | Current working directory |
| `EXECVE` | Command execution arguments |
| `USER_LOGIN` | User login attempt |
| `USER_AUTH` | User authentication |
| `USER_START` | Session opened |
| `USER_END` | Session closed |
| `USER_CMD` | User ran a command (sudo) |
| `CRED_ACQ` | Credentials acquired |
| `CRED_DISP` | Credentials disposed |
| `AVC` | SELinux access vector cache |
| `CONFIG_CHANGE` | Audit config modified |
| `SERVICE_START` | Service started |
| `SERVICE_STOP` | Service stopped |
| `CRYPTO_SESSION` | SSH/TLS session |

## Compliance Rule Sets

### PCI-DSS Basics

```bash
# /etc/audit/rules.d/pci-dss.rules
-w /etc/passwd -p rwxa -k identity
-w /etc/shadow -p rwxa -k identity
-w /etc/group -p rwxa -k identity
-w /etc/security/ -p rwxa -k security_config
-w /etc/audit/ -p rwxa -k audit_config
-a always,exit -F arch=b64 -S execve -F euid=0 -k root_activity
-a always,exit -F arch=b64 -S unlink -S rmdir -k file_deletion
-w /var/log/ -p rwxa -k log_access
```

### CIS Benchmark Rules

```bash
# /etc/audit/rules.d/cis.rules
# Time changes
-a always,exit -F arch=b64 -S adjtimex -S settimeofday -k time-change
-w /etc/localtime -p wa -k time-change

# User/group changes
-w /etc/group -p wa -k identity
-w /etc/passwd -p wa -k identity
-w /etc/gshadow -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/security/opasswd -p wa -k identity

# Network configuration
-a always,exit -F arch=b64 -S sethostname -S setdomainname -k system-locale
-w /etc/issue -p wa -k system-locale
-w /etc/issue.net -p wa -k system-locale
-w /etc/hosts -p wa -k system-locale
-w /etc/hostname -p wa -k system-locale

# Login/logout events
-w /var/log/lastlog -p wa -k logins
-w /var/run/faillock/ -p wa -k logins
-w /var/log/tallylog -p wa -k logins

# Session initiation
-w /var/run/utmp -p wa -k session
-w /var/log/wtmp -p wa -k logins
-w /var/log/btmp -p wa -k logins

# Discretionary access control
-a always,exit -F arch=b64 -S chmod -S fchmod -S fchmodat -k perm_mod
-a always,exit -F arch=b64 -S chown -S fchown -S fchownat -S lchown -k perm_mod

# Unauthorized access attempts
-a always,exit -F arch=b64 -S open -S openat -S creat -F exit=-EACCES -k access
-a always,exit -F arch=b64 -S open -S openat -S creat -F exit=-EPERM -k access

# Privileged commands
-a always,exit -F path=/usr/bin/sudo -F perm=x -k privileged
-a always,exit -F path=/usr/bin/su -F perm=x -k privileged
-a always,exit -F path=/usr/bin/chage -F perm=x -k privileged
-a always,exit -F path=/usr/bin/passwd -F perm=x -k privileged

# Make immutable
-e 2
```

## Troubleshooting

### Audit Not Logging

```bash
# Check if auditd is running
systemctl status auditd

# Check buffer status (lost events = buffer too small)
sudo auditctl -s
# If "lost" > 0, increase buffer:
sudo auditctl -b 16384

# Check disk space
df -h /var/log/audit/

# Check permissions
ls -la /var/log/audit/audit.log
```

### Too Many Events (Noisy Rules)

```bash
# Check which rules generate the most events
sudo aureport -k --summary

# Exclude specific events
# Add to rules: exclude directories you don't care about
-a never,exit -F dir=/proc -k exclude_proc
-a never,exit -F dir=/sys -k exclude_sys

# Exclude specific programs
-a never,exit -F exe=/usr/sbin/crond -k exclude_cron
```

### Performance Impact

```bash
# Check audit backlog
sudo auditctl -s | grep backlog

# If backlog is consistently high:
# 1. Increase backlog limit
sudo auditctl -b 32768

# 2. Reduce rule specificity
# 3. Use async flush mode in auditd.conf
# flush = INCREMENTAL_ASYNC

# 4. Exclude noisy paths
```

### Lost Events

```bash
# Check for lost events
sudo auditctl -s | grep lost

# If losing events:
# 1. Increase buffer: -b 65536
# 2. Set failure mode: -f 1 (log to kernel)
# 3. Check I/O performance on /var/log/audit/
# 4. Consider remote logging with audisp
```

## Quick Reference

| Action | Command |
|--------|---------|
| Check status | `auditctl -s` |
| List rules | `auditctl -l` |
| Watch a file | `auditctl -w /path -p rwxa -k keyname` |
| Delete all rules | `auditctl -D` |
| Search by key | `ausearch -k keyname` |
| Search recent | `ausearch -ts recent` |
| Search by user | `ausearch -ua UID` |
| Failed logins | `ausearch -m USER_LOGIN --success no` |
| Auth report | `aureport -au` |
| Login report | `aureport -l --failed` |
| Key summary | `aureport -k --summary` |
| Load rules | `augenrules --load` |
| Rotate logs | `service auditd rotate` |
| Trace process | `autrace /bin/command` |
| Interpret output | `ausearch -i -k keyname` |
