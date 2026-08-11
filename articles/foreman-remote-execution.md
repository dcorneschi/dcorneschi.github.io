# Setting Up Remote Execution in Foreman / Satellite

Remote Execution (REX) allows running commands, scripts, and Ansible playbooks on managed hosts directly from Foreman or Red Hat Satellite — without needing direct SSH access from your workstation.

## Overview

Remote Execution uses SSH (or Ansible) to connect from the Foreman/Smart Proxy to managed hosts and execute jobs. It supports:

- Running arbitrary commands on one or many hosts
- Executing scripts (bash, Python, etc.)
- Running Ansible playbooks and roles
- Scheduling jobs for later execution
- Package installation, errata application, Puppet runs

## Prerequisites

- Foreman or Red Hat Satellite installed and operational
- Smart Proxy (Capsule) with network access to managed hosts on port 22
- Managed hosts registered to Foreman/Satellite

## Installation

### Enable Remote Execution Plugin

```bash
foreman-installer \
  --enable-foreman-plugin-remote-execution \
  --enable-foreman-proxy-plugin-remote-execution-script
```

For Satellite:

```bash
satellite-installer \
  --enable-foreman-plugin-remote-execution \
  --enable-foreman-proxy-plugin-remote-execution-script
```

This installs and configures:

- `foreman_remote_execution` — the Foreman plugin (web UI and API)
- `smart_proxy_remote_execution_ssh` — the Smart Proxy plugin (SSH connectivity)

### Verify Installation

```bash
# Check the proxy features include remote execution
hammer proxy info --name "$(hostname -f)" | grep -i "remote"

# Should show:
# Features: ... Remote Execution ...

# Restart services if needed
foreman-maintain service restart
```

## SSH Key Distribution

Remote Execution uses SSH keys for authentication. The Smart Proxy generates a key pair during installation.

### View the Public Key

```bash
# Default key location
cat ~foreman-proxy/.ssh/id_rsa_foreman_proxy.pub

# Full path (same location)
cat /var/lib/foreman-proxy/ssh/id_rsa_foreman_proxy.pub
```

The key pair is located at:

- `~foreman-proxy/.ssh/id_rsa_foreman_proxy` (private key)
- `~foreman-proxy/.ssh/id_rsa_foreman_proxy.pub` (public key)

If you replace these files with your own key, either make sure the private key is not protected by a passphrase, or you will need to specify the passphrase for every job.

### Distribute Key to Managed Hosts

**Method 1: Using the Foreman API (recommended for bulk)**

```bash
# Push SSH key to a single host
hammer host update --name "server01.example.com" --manage-ssh-keys true

# The key is distributed automatically when hosts are provisioned
# or via the remote_execution_ssh_keys snippet in provisioning templates
```

**Method 2: Manual distribution**

```bash
# Copy key to a host manually
ssh-copy-id -i /var/lib/foreman-proxy/ssh/id_rsa_foreman_proxy.pub root@server01.example.com

# Or append to authorized_keys
cat /var/lib/foreman-proxy/ssh/id_rsa_foreman_proxy.pub | \
  ssh root@server01.example.com 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys'
```

**Method 3: Via Puppet/Ansible (scalable)**

Include the public key in your configuration management to deploy it to all hosts automatically.

**Method 4: Provisioning snippet**

Add the `remote_execution_ssh_keys` snippet to your provisioning templates. New hosts will get the key automatically during kickstart.

### Verify SSH Connectivity

```bash
# Test from the Smart Proxy
ssh -i /var/lib/foreman-proxy/ssh/id_rsa_foreman_proxy root@server01.example.com hostname
```

## Configuration

### Global Settings (Web UI)

Navigate to **Administer → Settings → Remote Execution**:

| Setting | Description | Default |
|---------|-------------|---------|
| SSH User | Default user for SSH connections | `root` |
| Effective User | User to run commands as (via sudo) | `root` |
| SSH Port | Default SSH port | `22` |
| Connection Timeout | Timeout for SSH connection (seconds) | `60` |
| Cleanup Working Dirs | Remove temp directories after execution | `true` |

### Per-Host Settings

Override the global SSH user or port for specific hosts:

```bash
# Set SSH user for a host
hammer host update \
  --name "server01.example.com" \
  --parameters "remote_execution_ssh_user=deploy"

# Set SSH port for a host
hammer host update \
  --name "server01.example.com" \
  --parameters "remote_execution_ssh_port=2222"
```

### Configure Effective User (sudo)

To run commands as a non-root user but escalate via sudo:

```bash
# On the managed host, add to /etc/sudoers or /etc/sudoers.d/foreman-proxy:
foreman-proxy ALL=(ALL) NOPASSWD: ALL
```

In Foreman settings, set:
- **SSH User** → `foreman-proxy`
- **Effective User** → `root`
- **Effective User Method** → `sudo`

## Running Jobs

### Via Hammer CLI

```bash
# Run a command on a single host
hammer job-invocation create \
  --job-template "Run Command - SSH Default" \
  --inputs "command=uptime" \
  --search-query "name = server01.example.com"

# Run on multiple hosts by search
hammer job-invocation create \
  --job-template "Run Command - SSH Default" \
  --inputs "command=df -h" \
  --search-query "hostgroup = Production"

# Run on all hosts in an organization
hammer job-invocation create \
  --job-template "Run Command - SSH Default" \
  --inputs "command=yum check-update" \
  --search-query "organization = ACME"

# Run a script
hammer job-invocation create \
  --job-template "Run Command - SSH Default" \
  --inputs "command=bash -c 'echo hello; hostname; date'"

# Schedule a job for later
hammer job-invocation create \
  --job-template "Run Command - SSH Default" \
  --inputs "command=yum update -y" \
  --search-query "hostgroup = Production" \
  --start-at "2024-12-01 02:00:00"

# Apply errata via REX
hammer job-invocation create \
  --job-template "Install Errata - Katello SSH Default" \
  --inputs "errata=RHSA-2024:1234" \
  --search-query "name = server01.example.com"
```

### Via Web UI

1. Navigate to **Monitor → Jobs → Run Job**
2. Select a job template (e.g., "Run Command - SSH Default")
3. Enter the command or inputs
4. Select target hosts (by search query, hostgroup, or individual selection)
5. Click **Submit**

To verify Remote Execution is working, run a simple test job with the command `uptime`.

> **Note:** Applying errata via Remote Execution requires SSH key authentication — it cannot be done with password-only authentication. However, a job can be scheduled manually using the same Katello template.

### Check Job Status

```bash
# List recent jobs
hammer job-invocation list

# Show job details
hammer job-invocation info --id <job-id>

# Show output for a specific host
hammer job-invocation output --id <job-id> --host "server01.example.com"
```

## Default Job Templates

Foreman ships with built-in job templates:

| Template | Purpose |
|----------|---------|
| Run Command - SSH Default | Execute arbitrary commands |
| Install Errata - Katello SSH Default | Apply errata packages |
| Install Package - Katello SSH Default | Install packages |
| Remove Package - Katello SSH Default | Remove packages |
| Update Package - Katello SSH Default | Update packages |
| Restart Service - SSH Default | Restart a systemd service |
| Power Action - SSH Default | Reboot/shutdown via SSH |
| Puppet Run Once - SSH Default | Trigger a Puppet run |
| Module Stream Actions - SSH Default | Manage DNF module streams |

### Create Custom Job Template

Navigate to **Hosts → Templates → Job Templates → New Job Template**:

```erb
<%# Template name: Check Disk Space %>
<%# Description: Check disk usage and alert if above threshold %>
<%# Job category: Commands %>

df -h | awk '$5+0 > <%= input("threshold") %> {print "WARNING:", $0}'
```

Via hammer:

```bash
hammer job-template create \
  --name "Check Disk Space" \
  --file /tmp/check_disk.erb \
  --provider-type SSH \
  --job-category "Commands"
```

## Ansible Integration

Remote Execution can use Ansible as a provider instead of (or alongside) SSH scripts:

```bash
# Enable Ansible plugin
foreman-installer \
  --enable-foreman-plugin-ansible \
  --enable-foreman-proxy-plugin-ansible

# Import Ansible roles
hammer ansible roles sync --proxy-id 1

# Run an Ansible role on hosts
hammer job-invocation create \
  --job-template "Ansible Roles - Ansible Default" \
  --search-query "hostgroup = Production"
```

## Troubleshooting

### Connection Issues

```bash
# Test SSH from the Smart Proxy to the host
sudo -u foreman-proxy \
  ssh -i /var/lib/foreman-proxy/ssh/id_rsa_foreman_proxy \
  root@server01.example.com hostname

# Check proxy logs
tail -f /var/log/foreman-proxy/proxy.log

# Check Foreman task logs
hammer task list --search "label = Actions::RemoteExecution::RunHostJob AND result = error"

# Verify the proxy has REX feature
hammer proxy refresh-features --name "$(hostname -f)"
hammer proxy info --name "$(hostname -f)"
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Connection refused | SSH not running on host or port mismatch | Verify sshd is running, check port |
| Permission denied | SSH key not deployed or wrong user | Distribute the proxy's public key |
| Host not found | DNS resolution failure | Check `/etc/hosts` or DNS |
| Timeout | Network or firewall blocking port 22 | Open port 22 from proxy to host |
| No proxy found | Smart Proxy not assigned to host | Assign proxy in host or hostgroup |

### Check SSH Key Fingerprint

```bash
# On the Smart Proxy
ssh-keygen -lf /var/lib/foreman-proxy/ssh/id_rsa_foreman_proxy.pub

# On the managed host (verify key is present)
grep "foreman-proxy" /root/.ssh/authorized_keys
```

### Re-generate SSH Key

```bash
# If the key is compromised or lost
rm /var/lib/foreman-proxy/ssh/id_rsa_foreman_proxy*
foreman-installer --enable-foreman-proxy-plugin-remote-execution-script
# Redistribute the new public key to all hosts
```

## Security Considerations

- The Smart Proxy SSH key grants root access to all managed hosts — protect it
- Use `Effective User` + sudo to avoid distributing root SSH keys
- Restrict which users can run REX jobs via Foreman roles/permissions
- Audit REX usage via Foreman's built-in audit log (**Monitor → Audits**)
- Consider network segmentation — only Smart Proxies should have SSH access to hosts

## Quick Reference

```bash
# Enable REX
foreman-installer --enable-foreman-plugin-remote-execution --enable-foreman-proxy-plugin-remote-execution-script

# View proxy SSH key
cat /var/lib/foreman-proxy/ssh/id_rsa_foreman_proxy.pub

# Distribute key to host
ssh-copy-id -i /var/lib/foreman-proxy/ssh/id_rsa_foreman_proxy.pub root@<host>

# Run a command
hammer job-invocation create --job-template "Run Command - SSH Default" --inputs "command=<cmd>" --search-query "name = <host>"

# Check job output
hammer job-invocation output --id <job-id> --host "<host>"

# List recent jobs
hammer job-invocation list
```
