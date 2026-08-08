# Linux Tools for Executing Scripts via SSH

Running commands and scripts on remote hosts is a core sysadmin task. This guide covers tools ranging from raw SSH one-liners to full configuration management systems — from simplest to most capable.

## SSH (Native)

The simplest approach — no additional tools required.

### Run a Single Command

```bash
ssh user@host "uptime"
ssh user@host "df -h && free -m"

# Multiple commands (semicolons — run all regardless of success/failure)
ssh user@host 'uptime; df -h; whoami'

# Conditional execution (next command only runs if previous succeeds)
ssh user@host "cd /var/www && git pull && systemctl restart nginx"

# Alternative execution (second runs only if first fails)
ssh user@host "command1 || command2"

# With environment variables
ssh user@host 'export VAR=value; ./script.sh'
```

### Pipe Commands to SSH

```bash
# Pipe multiple commands via echo
echo -e "command1\ncommand2\ncommand3" | ssh user@host

# Pipe from a command list
printf '%s\n' "uptime" "df -h" "whoami" | ssh user@host
```

### Run a Local Script on a Remote Host

```bash
# Pipe script via stdin
ssh user@host 'bash -s' < local_script.sh

# With arguments (using --)
ssh user@host 'bash -s' -- arg1 arg2 < local_script.sh

# With arguments (inline)
ssh user@host 'bash -s arg1 arg2' < local_script.sh

# Specify interpreter
ssh user@host 'python3 -' < local_script.py
```

### Run a Multi-line Command

```bash
# Here document (variables NOT expanded — single-quoted delimiter)
ssh user@host << 'EOF'
echo "Hostname: $(hostname)"
echo "Uptime: $(uptime)"
echo "Disk: $(df -h / | tail -1)"
EOF

# Here document (variables expanded locally — unquoted delimiter)
ssh user@host << EOF
echo "Connecting from $(hostname) to remote"
uptime
df -h
EOF

# Multi-line with double quotes (supports cd, exports, multi-step logic)
ssh user@host "
cd /path/to/directory
export VAR=value
command1
command2
"

# Practical example: stop, update, restart
ssh user@host << 'EOF'
sudo systemctl stop myapp
sudo cp /tmp/newconfig /etc/myapp/
sudo systemctl start myapp
sudo systemctl status myapp
EOF
```

### Quoting Rules for Remote Commands

| Quoting | Behavior |
|---------|----------|
| `ssh host "command $VAR"` | `$VAR` expanded **locally** before sending |
| `ssh host 'command $VAR'` | `$VAR` expanded **remotely** on the server |
| `ssh host << 'EOF'` | Heredoc content sent literally (remote expansion) |
| `ssh host << EOF` | Heredoc content expanded locally first |

```bash
# Local variable sent to remote
LOCAL_PATH="/tmp/backup"
ssh user@host "mkdir -p $LOCAL_PATH"    # expands locally

# Remote variable used on remote
ssh user@host 'echo $HOME'             # expands on server

# Mixed: local variable in remote context
REMOTE_DIR="/opt/app"
ssh user@host "cd $REMOTE_DIR && ls"   # $REMOTE_DIR expanded locally
```

### Run with sudo

```bash
# Interactive (prompts for password)
ssh -t user@host "sudo systemctl restart nginx"

# Non-interactive (requires NOPASSWD in sudoers)
ssh user@host "sudo -n systemctl restart nginx"
```

### Run on Multiple Hosts (Loop)

```bash
for host in server1 server2 server3; do
    echo "=== $host ==="
    ssh user@$host "uptime"
done
```

```bash
# From a file
while read -r host; do
    ssh user@$host "df -h /" 
done < hosts.txt
```

### Copy and Execute

```bash
# Copy script, execute, clean up
scp script.sh user@host:/tmp/ && ssh user@host "bash /tmp/script.sh && rm /tmp/script.sh"

# Copy, make executable, then execute
scp script.sh user@host:/tmp/ && ssh user@host "chmod +x /tmp/script.sh && /tmp/script.sh"
```

### Tar Pipe (Transfer and Execute)

```bash
# Send a directory and run a script inside it
tar czf - ./deploy/ | ssh user@host "tar xzf - -C /tmp && bash /tmp/deploy/install.sh"
```

## parallel-ssh (pssh)

Run commands on multiple hosts in parallel. Part of the `pssh` package.

### Install

```bash
# Debian/Ubuntu
sudo apt install pssh

# RHEL/CentOS
sudo yum install pssh

# macOS
brew install pssh

# pip
pip install parallel-ssh
```

### Basic Usage

```bash
# Run command on multiple hosts
parallel-ssh -h hosts.txt -l user "uptime"

# Inline host list
parallel-ssh -H "host1 host2 host3" -l user "df -h"

# Inline output (show stdout from each host)
parallel-ssh -h hosts.txt -l user -i "df -h /"

# With timeout
parallel-ssh -h hosts.txt -l user -t 30 "systemctl status nginx"

# Unlimited timeout (useful with sudo)
parallel-ssh -h hosts.txt -l user -t 0 "sudo apt upgrade -y"

# Parallel threads (default: 32)
parallel-ssh -h hosts.txt -l user -p 10 "apt update"

# Execute script on multiple hosts
parallel-ssh -h hosts.txt -l user -I < script.sh
```

### hosts.txt Format

```
192.168.1.10
192.168.1.11
192.168.1.12:2222
user@192.168.1.13
user@192.168.1.14:2222
```

### Copy Files to Multiple Hosts

```bash
# parallel-scp
parallel-scp -h hosts.txt -l user local_file.conf /etc/app/

# parallel-rsync
parallel-rsync -h hosts.txt -l user -a ./configs/ /etc/app/
```

### Save Output Per Host

```bash
parallel-ssh -h hosts.txt -l user -o /tmp/output -e /tmp/errors "uptime"
# Creates /tmp/output/host1, /tmp/output/host2, etc.
```

### Key Options

| Flag | Description |
|------|-------------|
| `-h file` | Hosts file |
| `-H "host1 host2"` | Inline host list |
| `-l user` | Username |
| `-p N` | Parallelism (concurrent connections) |
| `-t N` | Command timeout (seconds) |
| `-O opt` | SSH option (e.g., `-O StrictHostKeyChecking=no`) |
| `-i` | Inline output (print stdout) |
| `-o dir` | Output directory (per-host files) |
| `-e dir` | Error directory (per-host stderr) |
| `-I` | Read stdin and send to each host |
| `-A` | Prompt for password (no key auth) |
| `-x args` | Extra SSH arguments |

## pdsh (Parallel Distributed Shell)

Similar to pssh but designed for HPC and cluster environments.

### Install

```bash
# Debian/Ubuntu
sudo apt install pdsh

# RHEL/CentOS
sudo yum install pdsh

# macOS
brew install pdsh
```

### Basic Usage

```bash
# Run command on hosts
pdsh -w "host[1-5]" "uptime"

# Range syntax
pdsh -w "web[01-10].example.com" "systemctl status nginx"

# Comma-separated
pdsh -w "host1,host2,host3" "df -h"

# Exclude hosts
pdsh -w "host[1-10]" -x "host3,host7" "uptime"
```

### Using SSH (Default May Be rsh)

```bash
# Set SSH as transport
export PDSH_RCMD_TYPE=ssh

# Or per-command
pdsh -R ssh -w "host[1-5]" "uptime"
```

### Output Formatting

```bash
# Default output (prefixed with hostname)
pdsh -w "host[1-3]" "uptime"
# host1: 10:30:01 up 5 days...
# host2: 10:30:01 up 12 days...

# Sort output by hostname
pdsh -w "host[1-3]" "uptime" | sort

# Consolidate identical output (dshbak)
pdsh -w "host[1-10]" "cat /etc/redhat-release" | dshbak -c
```

### Key Options

| Flag | Description |
|------|-------------|
| `-w hosts` | Target hosts (ranges, comma-separated) |
| `-x hosts` | Exclude hosts |
| `-l user` | Remote username |
| `-R ssh` | Use SSH as transport |
| `-t N` | Command timeout |
| `-f N` | Max concurrent connections (fanout) |

## ClusterSSH (cssh)

Opens multiple terminal windows — one per host — and a shared input window. Commands typed in the input window are sent to all hosts simultaneously.

### Install

```bash
# Debian/Ubuntu
sudo apt install clusterssh

# macOS
brew install clusterssh
```

### Usage

```bash
# Open terminals to multiple hosts
cssh host1 host2 host3

# From a config file
cssh -c clusters.conf webservers

# With username
cssh user@host1 user@host2
```

### clusters.conf

```
webservers = web1 web2 web3
dbservers = db1 db2
all = web1 web2 web3 db1 db2
```

Best for interactive/ad-hoc work where you need to see output and respond in real-time.

## MuSSH (Multi SSH)

A lightweight multi-host SSH tool.

### Install

```bash
sudo apt install mussh
```

### Usage

```bash
# Execute on multiple hosts
mussh -h hosts.txt -c 'uptime'

# Execute with user
mussh -h hosts.txt -l username -c 'sudo systemctl restart apache2'

# Execute script
mussh -h hosts.txt -c 'bash -s' < script.sh

# Print hostnames with output
mussh -h hosts.txt -p -c 'hostname; uptime'
```

## DSH (Distributed Shell)

### Install

```bash
sudo apt install dsh
```

### Configuration

```bash
# Create machine list
echo "host1" >> ~/.dsh/machines.list
echo "host2" >> ~/.dsh/machines.list
```

### Usage

```bash
# Execute on all machines in list
dsh -a uptime

# Execute on specific group
dsh -g webservers 'systemctl status nginx'

# Execute script
dsh -a 'bash -s' < script.sh
```

## Ansible (Ad-hoc Commands)

Ansible is primarily a configuration management tool, but its ad-hoc mode is excellent for one-off commands across many hosts.

### Install

```bash
pip install ansible
# or
sudo apt install ansible
```

### Inventory File (hosts.ini)

```ini
[webservers]
web1 ansible_host=192.168.1.10
web2 ansible_host=192.168.1.11

[dbservers]
db1 ansible_host=192.168.1.20

[all:vars]
ansible_user=admin
ansible_ssh_private_key_file=~/.ssh/homelab
```

### Ad-hoc Commands

```bash
# Run a command on all hosts
ansible all -i hosts.ini -m command -a "uptime"

# Shell module (supports pipes, redirects)
ansible all -i hosts.ini -m shell -a "df -h | grep sda"

# Copy a file
ansible all -i hosts.ini -m copy -a "src=app.conf dest=/etc/app/app.conf"

# Run a script
ansible all -i hosts.ini -m script -a "./local_script.sh"

# Install a package
ansible webservers -i hosts.ini -m apt -a "name=nginx state=present" --become

# Restart a service
ansible webservers -i hosts.ini -m service -a "name=nginx state=restarted" --become

# Run with parallelism (forks)
ansible all -i hosts.ini -m command -a "uptime" -f 20
```

### Key Options

| Flag | Description |
|------|-------------|
| `-i inventory` | Inventory file |
| `-m module` | Module to use (command, shell, script, copy, etc.) |
| `-a "args"` | Module arguments |
| `--become` | Escalate privileges (sudo) |
| `-f N` | Number of parallel forks (default: 5) |
| `--limit pattern` | Limit to specific hosts |
| `-u user` | Remote username |
| `--key-file path` | SSH key to use |

### Playbook (For Repeated Tasks)

```yaml
# deploy.yml
---
- hosts: webservers
  become: yes
  tasks:
    - name: Copy script
      copy:
        src: deploy.sh
        dest: /tmp/deploy.sh
        mode: '0755'

    - name: Execute script
      command: /tmp/deploy.sh

    - name: Clean up
      file:
        path: /tmp/deploy.sh
        state: absent
```

```bash
ansible-playbook -i hosts.ini deploy.yml
```

## Fabric (Python)

Python library for executing commands over SSH. Useful when you need programmatic logic around your remote execution.

### Install

```bash
pip install fabric
```

### Basic fabfile.py

```python
from fabric import Connection, task

@task
def deploy(c):
    """Deploy application to remote host."""
    c.run("systemctl stop app")
    c.put("app.tar.gz", "/tmp/app.tar.gz")
    c.run("tar xzf /tmp/app.tar.gz -C /opt/app")
    c.run("systemctl start app")

@task
def uptime(c):
    """Check uptime on remote host."""
    result = c.run("uptime", hide=True)
    print(f"{c.host}: {result.stdout.strip()}")
```

```bash
# Run on a single host
fab -H user@host deploy

# Run on multiple hosts
fab -H user@host1,user@host2 uptime
```

### Group Execution

```python
from fabric import SerialGroup, ThreadingGroup

# Sequential
group = SerialGroup("host1", "host2", "host3", user="admin")
results = group.run("uptime")

# Parallel
group = ThreadingGroup("host1", "host2", "host3", user="admin")
results = group.run("df -h /")

for conn, result in results.items():
    print(f"{conn.host}: {result.stdout.strip()}")
```

### With sudo

```python
from fabric import Connection

c = Connection("host", user="admin")
c.sudo("systemctl restart nginx", password="secret")
```

## Salt SSH

SaltStack's agentless mode — uses SSH instead of the usual minion/master architecture.

### Install

```bash
pip install salt-ssh
# or
sudo apt install salt-ssh
```

### Roster File (/etc/salt/roster)

```yaml
web1:
  host: 192.168.1.10
  user: admin
  priv: /home/user/.ssh/homelab

web2:
  host: 192.168.1.11
  user: admin
  priv: /home/user/.ssh/homelab
```

### Usage

```bash
# Run a command
salt-ssh '*' cmd.run "uptime"

# Target specific hosts
salt-ssh 'web1' cmd.run "df -h"

# Install a package
salt-ssh '*' pkg.install nginx

# Copy a file
salt-ssh '*' cp.get_file salt://configs/app.conf /etc/app/app.conf

# Apply a state
salt-ssh '*' state.apply webserver
```

## Capistrano (Ruby-based Deployment)

### Install

```bash
gem install capistrano
```

### Sample config/deploy.rb

```ruby
set :application, 'myapp'
server 'server1.example.com', roles: [:web, :app]
server 'server2.example.com', roles: [:web, :app]

task :execute_script do
  on roles(:all) do
    upload! 'local-script.sh', '/tmp/script.sh'
    execute 'chmod +x /tmp/script.sh'
    execute '/tmp/script.sh'
    execute 'rm /tmp/script.sh'
  end
end
```

```bash
cap production execute_script
```

## Rundeck (Web-based)

A web-based job scheduler and runbook automation tool:

- GUI for job execution across multiple nodes
- Workflow management with step-by-step execution
- Role-based access control
- Scheduling and cron-like capabilities
- Audit logging of all executions
- Integrates with Ansible, scripts, and APIs

Install via package manager or Docker. Access the web UI on port 4440 by default.

## Custom Bash Scripts

### Parallel Execution with Background Jobs

```bash
#!/bin/bash
# parallel-exec.sh

HOSTS="server1 server2 server3"
USERNAME="admin"

execute_on_host() {
    local host=$1
    echo "Starting execution on $host"
    ssh $USERNAME@$host 'bash -s' < script.sh
    echo "Completed on $host"
}

# Execute in parallel (background jobs)
for host in $HOSTS; do
    execute_on_host $host &
done

# Wait for all to complete
wait
echo "All executions completed"
```

### Script with Error Handling and Logging

```bash
#!/bin/bash
# robust-ssh.sh

HOSTS_FILE="hosts.txt"
USERNAME="admin"
SCRIPT="maintenance.sh"
LOG_DIR="logs"

mkdir -p $LOG_DIR

while read -r host; do
    echo "Executing on $host..."
    
    if ssh -n -o ConnectTimeout=10 -o BatchMode=yes $USERNAME@$host 'bash -s' < $SCRIPT > "$LOG_DIR/$host.log" 2>&1; then
        echo "✓ Success on $host"
    else
        echo "✗ Failed on $host - check $LOG_DIR/$host.log"
    fi
done < $HOSTS_FILE
```

## xargs + ssh

Combine `xargs` with SSH for quick parallel execution without extra tools:

```bash
# Run on 4 hosts in parallel
cat hosts.txt | xargs -I{} -P4 ssh user@{} "uptime"

# With a command
echo "host1 host2 host3" | tr ' ' '\n' | xargs -I{} -P3 ssh user@{} "df -h /"

# Capture output
cat hosts.txt | xargs -I{} -P4 sh -c 'echo "=== {} ==="; ssh user@{} "uptime"'
```

## GNU Parallel + ssh

More powerful than xargs for parallel remote execution:

### Install

```bash
sudo apt install parallel    # Debian/Ubuntu
sudo yum install parallel    # RHEL/CentOS
brew install parallel        # macOS
```

### Usage

```bash
# Run command on multiple hosts
parallel ssh user@{} "uptime" ::: host1 host2 host3

# From a hosts file
parallel --slf hosts.txt "uptime"

# From a hosts file (alternative syntax)
parallel -j 10 ssh {} 'bash -s' < script.sh :::: hosts.txt

# With different users per host (tab-separated file)
parallel --colsep '\t' ssh {1}@{2} 'uptime' :::: hosts_with_users.txt

# Transfer and execute
parallel --slf hosts.txt --transferfile script.sh --return results.txt \
    "bash script.sh > results.txt"

# With progress bar
parallel --slf hosts.txt --bar "apt update && apt upgrade -y"

# Limit concurrent jobs
parallel -j 5 ssh {} 'uptime' ::: host1 host2 host3 host4 host5
```

### hosts.txt Format for GNU Parallel

```
user@host1
user@host2
4/user@host3    # max 4 jobs on host3
```

## sshpass + Loops

For environments where key-based auth isn't available (legacy systems):

```bash
# Single host
sshpass -p 'password' ssh -o StrictHostKeyChecking=no user@host "uptime"

# Multiple hosts
while read -r host; do
    sshpass -p 'password' ssh -o StrictHostKeyChecking=no user@$host "uptime"
done < hosts.txt
```

> **Warning:** Passing passwords on the command line is insecure (visible in `ps` output). Use key-based auth whenever possible. If you must use sshpass, use `-f` to read from a file:

```bash
sshpass -f /path/to/password_file ssh user@host "command"
```

## Comparison Table

| Tool | Parallelism | Agentless | Learning Curve | Best For |
|------|-------------|-----------|----------------|----------|
| SSH (native) | Manual (loops) | Yes | None | One-off commands, simple scripts |
| parallel-ssh (pssh) | Built-in | Yes | Low | Quick parallel commands, homogeneous fleets |
| pdsh | Built-in | Yes | Low | HPC clusters, host ranges |
| ClusterSSH (cssh) | Visual | Yes | Low | Interactive/ad-hoc multi-host sessions |
| MuSSH | Built-in | Yes | Low | Simple multi-host execution |
| DSH | Built-in | Yes | Low | Group-based distributed commands |
| xargs + ssh | Built-in (`-P`) | Yes | Low | Quick parallel without extra tools |
| GNU Parallel | Built-in | Yes | Medium | Complex parallel workflows, data transfer |
| Ansible | Built-in (forks) | Yes | Medium | Config management, idempotent tasks, playbooks |
| Fabric | ThreadingGroup | Yes | Medium | Python-driven automation, deploy scripts |
| Salt SSH | Built-in | Yes | High | Existing SaltStack users, state management |
| Capistrano | Built-in | Yes | Medium | Ruby-based deploy workflows |
| Rundeck | Built-in | Yes | High | Enterprise workflows, GUI, scheduling |
| sshpass | Manual | Yes | Low | Legacy systems without key auth (last resort) |

## Tips and Best Practices

### Avoid Interactive Prompts

```bash
# Disable host key checking (testing/CI only)
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null user@host "command"

# Accept new keys automatically
ssh -o StrictHostKeyChecking=accept-new user@host "command"
```

### Handle Failures Gracefully

```bash
# Continue on error (loop)
for host in $(cat hosts.txt); do
    ssh user@$host "command" || echo "FAILED: $host"
done

# Set timeout
ssh -o ConnectTimeout=5 user@host "command"

# Check connectivity before executing
if ssh -o BatchMode=yes -o ConnectTimeout=5 user@host exit; then
    ssh user@host 'your-command'
else
    echo "Cannot connect to host"
fi
```

### Avoid Stdin Consumption in Loops

SSH consumes stdin by default, which breaks `while read` loops:

```bash
# WRONG — ssh eats the rest of hosts.txt
while read -r host; do
    ssh user@$host "uptime"
done < hosts.txt

# RIGHT — redirect ssh stdin from /dev/null
while read -r host; do
    ssh -n user@$host "uptime"
done < hosts.txt
```

The `-n` flag (or `< /dev/null`) prevents SSH from consuming the loop's stdin.

### Use Multiplexing for Repeated Connections

When running multiple commands against the same host:

```
Host *
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 300
```

### Run with Consistent Environment

Remote SSH commands don't load `.bashrc` or `.profile` by default (non-login, non-interactive shell):

```bash
# Source profile explicitly
ssh user@host "source /etc/profile && command"

# Or use bash -l (login shell)
ssh user@host "bash -lc 'command'"

# Set PATH explicitly
ssh user@host "PATH=/usr/local/bin:/usr/bin:/bin command"
```

### Capture Exit Codes

```bash
ssh user@host "command"
exit_code=$?

if [ $exit_code -ne 0 ]; then
    echo "Command failed with exit code $exit_code"
fi
```

### Log Output

```bash
# Log per-host output
for host in $(cat hosts.txt); do
    ssh -n user@$host "command" > "/tmp/output_${host}.log" 2>&1
done

# Log with timestamps
ssh user@host 'script.sh' 2>&1 | while read -r line; do
    echo "$(date '+%Y-%m-%d %H:%M:%S'): $line"
done
```

## Quick Reference

```bash
# Execute script on single host
ssh user@host 'bash -s' < script.sh

# Execute on multiple hosts (basic loop)
for host in host1 host2 host3; do ssh -n $host 'command'; done

# Execute in parallel with GNU parallel
parallel ssh {} 'command' ::: host1 host2 host3

# Execute with PSSH
pssh -h hosts.txt -l user 'command'

# Copy and execute
scp script.sh user@host:/tmp/ && ssh user@host '/tmp/script.sh'

# Parallel background jobs
for host in host1 host2 host3; do ssh -n user@$host 'command' & done; wait
```
