# Nagios check_by_ssh Plugin Guide

`check_by_ssh` executes Nagios plugins on remote hosts via SSH, eliminating the need for NRPE. It uses the Nagios server's SSH key to connect and run checks remotely.

## How It Works

```
Nagios Server → SSH → Remote Host → Execute Plugin → Return Output → Nagios Server
```

- No agent required on the remote host (only SSH and the plugin)
- Uses SSH key-based authentication (no passwords)
- Encrypted communication by default
- Plugin output and exit code are passed back to Nagios

## Prerequisites

### On the Nagios Server

```bash
# Verify check_by_ssh is installed
ls /usr/local/nagios/libexec/check_by_ssh

# Generate SSH key for the nagios user (no passphrase)
sudo -u nagios ssh-keygen -t ed25519 -N "" -f /var/spool/nagios/.ssh/id_ed25519

# Or if nagios home is /usr/local/nagios
sudo -u nagios ssh-keygen -t ed25519 -N "" -f /usr/local/nagios/.ssh/id_ed25519
```

### On the Remote Host

```bash
# Install the plugins you want to run remotely
sudo dnf install -y nagios-plugins-all    # RHEL/CentOS
sudo apt install -y nagios-plugins        # Ubuntu/Debian

# Or install from source (same path as Nagios server)
# Plugins should be in /usr/local/nagios/libexec/ or /usr/lib64/nagios/plugins/

# Or copy plugins directly from the Nagios server
# On the Nagios server:
sudo -u nagios scp -p /usr/local/nagios/libexec/* nagios@remote-host:/home/nagios/bin/

# Copy the Nagios server's public key to the remote host
# From the Nagios server:
sudo -u nagios ssh-copy-id -i /usr/local/nagios/.ssh/id_ed25519.pub user@remote-host

# Test the connection (should not prompt for password)
sudo -u nagios ssh user@remote-host "hostname"
```

## Basic Syntax

```bash
check_by_ssh -H <host> -C '<command>'

# Options:
# -H  Host to connect to
# -C  Command to execute on remote host
# -l  SSH username (default: nagios user running the check)
# -p  SSH port (default: 22)
# -t  Timeout in seconds (default: 10)
# -i  Identity file (SSH private key)
# -E  Ignore SSH stderr output
# -n  Host name for substitution in command
# -O  SSH options (passed to ssh command)
# -4  Use IPv4 only
# -6  Use IPv6 only
# -q  Quiet mode (suppress SSH warnings)
```

## Examples

### Check Disk Usage

```bash
# Command line test
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -C '/usr/local/nagios/libexec/check_disk -w 20% -c 10% -p /'

# Nagios command definition
define command {
    command_name    check_by_ssh_disk
    command_line    $USER1$/check_by_ssh -H $HOSTADDRESS$ -l nagios -C '$USER1$/check_disk -w $ARG1$ -c $ARG2$ -p $ARG3$'
}

# Service definition
define service {
    use                     generic-service
    host_name               webserver01
    service_description     Disk Usage /
    check_command           check_by_ssh_disk!20%!10%!/
}
```

### Check CPU Load

```bash
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -C '/usr/local/nagios/libexec/check_load -w 5,4,3 -c 10,8,6'

# Command definition
define command {
    command_name    check_by_ssh_load
    command_line    $USER1$/check_by_ssh -H $HOSTADDRESS$ -l nagios -C '$USER1$/check_load -w $ARG1$ -c $ARG2$'
}

# Service
define service {
    use                     generic-service
    host_name               webserver01
    service_description     CPU Load
    check_command           check_by_ssh_load!5,4,3!10,8,6
}
```

### Check Memory (Swap)

```bash
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -C '/usr/local/nagios/libexec/check_swap -w 20% -c 10%'

# Command definition
define command {
    command_name    check_by_ssh_swap
    command_line    $USER1$/check_by_ssh -H $HOSTADDRESS$ -l nagios -C '$USER1$/check_swap -w $ARG1$ -c $ARG2$'
}
```

### Check Running Processes

```bash
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -C '/usr/local/nagios/libexec/check_procs -w 250 -c 400'

# Check specific process
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -C '/usr/local/nagios/libexec/check_procs -c 1: -C httpd'
```

### Check Service Status (systemd)

```bash
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -C 'systemctl is-active nginx && echo "OK: nginx running" || (echo "CRITICAL: nginx not running"; exit 2)'
```

### Check File Age

```bash
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -C '/usr/local/nagios/libexec/check_file_age -w 86400 -c 172800 -f /var/log/backup.log'
```

### Check NTP Offset

```bash
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -C '/usr/local/nagios/libexec/check_ntp_time -H pool.ntp.org -w 0.5 -c 1.0'
```

### Custom Script Execution

```bash
# Run any custom script on the remote host
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -C '/usr/local/nagios/libexec/custom/check_app_health.sh'

# With arguments
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -C '/usr/local/nagios/libexec/custom/check_db_connections.sh --max 100 --warn 80'
```

## Generic Command Definition

A single flexible command definition that works for any remote check:

```bash
# Generic check_by_ssh command (pass full remote command as ARG1)
define command {
    command_name    check_by_ssh
    command_line    $USER1$/check_by_ssh -H $HOSTADDRESS$ -l nagios -t 30 -C '$ARG1$'
}

# Usage in services
define service {
    use                     generic-service
    host_name               webserver01
    service_description     Disk /
    check_command           check_by_ssh!/usr/local/nagios/libexec/check_disk -w 20% -c 10% -p /
}

define service {
    use                     generic-service
    host_name               webserver01
    service_description     CPU Load
    check_command           check_by_ssh!/usr/local/nagios/libexec/check_load -w 5,4,3 -c 10,8,6
}
```

## Using a Different SSH Port

```bash
# Specify port with -p
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -p 2222 \
    -l nagios \
    -C '/usr/local/nagios/libexec/check_load -w 5,4,3 -c 10,8,6'

# Command definition with port as argument
define command {
    command_name    check_by_ssh_port
    command_line    $USER1$/check_by_ssh -H $HOSTADDRESS$ -p $ARG1$ -l nagios -C '$ARG2$'
}
```

## Using a Specific SSH Key

```bash
# Specify identity file
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -i /usr/local/nagios/.ssh/id_ed25519 \
    -C '/usr/local/nagios/libexec/check_disk -w 20% -c 10% -p /'
```

## Multiple Commands in One SSH Session

```bash
# Execute multiple commands (reduces SSH overhead)
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -C '/usr/local/nagios/libexec/check_disk -w 20% -c 10% -p /' \
    -C '/usr/local/nagios/libexec/check_load -w 5,4,3 -c 10,8,6'

# Returns multiple lines, one per command
# Each command's exit status is checked
```

## SSH Options

```bash
# Disable strict host key checking (first connection or lab environments)
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -O "StrictHostKeyChecking=no" \
    -C '/usr/local/nagios/libexec/check_load -w 5,4,3 -c 10,8,6'

# Set connection timeout
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -O "ConnectTimeout=5" \
    -C '/usr/local/nagios/libexec/check_disk -w 20% -c 10% -p /'

# Combine multiple SSH options
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -O "StrictHostKeyChecking=no" \
    -O "UserKnownHostsFile=/dev/null" \
    -O "LogLevel=ERROR" \
    -C '/usr/local/nagios/libexec/check_load -w 5,4,3 -c 10,8,6'
```

## Tips and Tricks

### Suppress SSH Warnings

```bash
# Use -E to ignore stderr from SSH (motd, banners, etc.)
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -E \
    -C '/usr/local/nagios/libexec/check_disk -w 20% -c 10% -p /'

# Or use -q for quiet mode
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -q \
    -C '/usr/local/nagios/libexec/check_disk -w 20% -c 10% -p /'
```

### Pre-Accept Host Keys

```bash
# Accept host keys automatically (run once as nagios user)
sudo -u nagios ssh -o StrictHostKeyChecking=accept-new nagios@remote-host "exit"

# Or scan and add to known_hosts
sudo -u nagios ssh-keyscan remote-host >> /usr/local/nagios/.ssh/known_hosts
```

### Increase Timeout for Slow Checks

```bash
# Default timeout is 10 seconds — increase for slow checks
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -t 60 \
    -C '/usr/local/nagios/libexec/check_mysql_health --mode long-running-queries'
```

### Restrict SSH Access (Authorized Keys)

Lock down what the nagios user can do on the remote host:

```bash
# On the remote host, edit ~nagios/.ssh/authorized_keys
# Restrict to specific command or path:
command="/usr/local/nagios/libexec/*",no-agent-forwarding,no-port-forwarding,no-X11-forwarding ssh-ed25519 AAAA... nagios@nagios-server

# Or restrict to a wrapper script:
command="/usr/local/nagios/libexec/nagios-wrapper.sh",no-agent-forwarding,no-port-forwarding ssh-ed25519 AAAA... nagios@nagios-server
```

### Wrapper Script for sudo Checks

Some checks need root access (e.g., checking RAID status):

```bash
# On the remote host: /usr/local/nagios/libexec/check_raid_wrapper.sh
#!/bin/bash
sudo /usr/local/nagios/libexec/check_raid "$@"
```

```bash
# sudoers entry on remote host
nagios ALL=(root) NOPASSWD: /usr/local/nagios/libexec/check_raid

# Nagios command
/usr/local/nagios/libexec/check_by_ssh \
    -H 192.168.1.100 \
    -l nagios \
    -C '/usr/local/nagios/libexec/check_raid_wrapper.sh'
```

### SSH ControlMaster (Connection Multiplexing)

Reduce SSH overhead when running many checks against the same host:

```bash
# Create SSH config for the nagios user
# /usr/local/nagios/.ssh/config
Host *
    ControlMaster auto
    ControlPath /tmp/ssh-%r@%h:%p
    ControlPersist 600
    StrictHostKeyChecking accept-new
    ConnectTimeout 5
```

This keeps SSH connections open for 10 minutes, so subsequent checks reuse the same connection.

### Handle Plugin Path Differences

Different distros install plugins in different paths:

```bash
# Create a symlink on remote hosts for consistency
sudo ln -s /usr/lib64/nagios/plugins /usr/local/nagios/libexec

# Or use a variable in the command
define command {
    command_name    check_by_ssh_generic
    command_line    $USER1$/check_by_ssh -H $HOSTADDRESS$ -l nagios -C '$USER2$/$ARG1$'
}

# Set $USER2$ in resource.cfg based on remote plugin path
# $USER2$=/usr/lib64/nagios/plugins
```

## check_by_ssh vs NRPE

| Feature | check_by_ssh | NRPE |
|---------|-------------|------|
| Encryption | Built-in (SSH) | TLS (must configure) |
| Agent required | No (just SSH) | Yes (nrpe daemon) |
| Firewall | Port 22 (usually open) | Port 5666 (must open) |
| Overhead | Higher (SSH per check) | Lower (persistent daemon) |
| Configuration | Centralized (Nagios server) | Distributed (each host) |
| Flexibility | Any command | Pre-defined commands only |
| Argument passing | Full | Restricted by default |
| Security | SSH key auth | Allowed hosts + TLS |
| Scalability | Lower (SSH overhead) | Higher |
| Setup complexity | Simple (just SSH keys) | Medium (install + config on each host) |

### When to Use check_by_ssh

- Small to medium environments (< 200 hosts)
- Hosts already accessible via SSH
- Don't want to install/manage NRPE on every host
- Need to run arbitrary commands
- Security-sensitive environments (SSH is well-audited)

### When to Use NRPE

- Large environments (> 200 hosts)
- High check frequency (every 30 seconds)
- Hosts that don't have SSH enabled
- Want pre-defined, locked-down checks
- Performance is critical

## Troubleshooting

### "Remote command execution failed"

```bash
# Test manually as nagios user
sudo -u nagios /usr/local/nagios/libexec/check_by_ssh \
    -H remote-host -l nagios -C 'hostname'

# Check SSH connectivity
sudo -u nagios ssh nagios@remote-host "echo OK"

# Check if the plugin exists on remote host
sudo -u nagios ssh nagios@remote-host "ls -la /usr/local/nagios/libexec/check_disk"
```

### "Connection refused" or Timeout

```bash
# Verify SSH is running on remote host
ssh remote-host "systemctl status sshd"

# Check port
nc -zv remote-host 22

# Check firewall on remote host
ssh remote-host "firewall-cmd --list-ports"
```

### "Host key verification failed"

```bash
# Add the remote host to known_hosts
sudo -u nagios ssh-keyscan remote-host >> /usr/local/nagios/.ssh/known_hosts
chown nagios:nagios /usr/local/nagios/.ssh/known_hosts
```

### "Permission denied (publickey)"

```bash
# Verify the key is copied to the remote host
sudo -u nagios ssh -v nagios@remote-host

# Check permissions on remote host
# ~/.ssh should be 700
# ~/.ssh/authorized_keys should be 600

# Re-copy the key
sudo -u nagios ssh-copy-id nagios@remote-host
```

### Plugin Returns "UNKNOWN" with SSH Errors in Output

```bash
# Use -E to suppress SSH stderr
/usr/local/nagios/libexec/check_by_ssh -H host -l nagios -E -C '...'

# Or redirect in the command itself
/usr/local/nagios/libexec/check_by_ssh -H host -l nagios \
    -C '/usr/local/nagios/libexec/check_disk -w 20% -c 10% -p / 2>/dev/null'
```

## Quick Reference

| Action | Command |
|--------|---------|
| Basic check | `check_by_ssh -H host -C 'command'` |
| Specify user | `check_by_ssh -H host -l nagios -C 'command'` |
| Specify key | `check_by_ssh -H host -i /path/to/key -C 'command'` |
| Custom port | `check_by_ssh -H host -p 2222 -C 'command'` |
| Increase timeout | `check_by_ssh -H host -t 60 -C 'command'` |
| Suppress warnings | `check_by_ssh -H host -E -C 'command'` |
| SSH options | `check_by_ssh -H host -O "StrictHostKeyChecking=no" -C 'command'` |
| Multiple commands | `check_by_ssh -H host -C 'cmd1' -C 'cmd2'` |
