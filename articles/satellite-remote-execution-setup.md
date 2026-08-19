# Configure Red Hat Satellite Content Host with Remote Execution

Remote Execution (REX) replaces the deprecated `katello-agent` for managing content on RHEL hosts through Red Hat Satellite. It uses SSH-based communication from the Satellite Server or Capsule to target hosts, allowing you to run jobs, install packages, and apply errata without logging into individual systems.

This guide covers SSH key distribution, verifying access, and using Remote Execution for package and errata management.

## Prerequisites

- Red Hat Satellite 6.2 or later (REX introduced in 6.2)
- SSH access from Satellite/Capsule to content hosts
- Root access on remote hosts (or a user with appropriate sudo privileges)

## Distributing SSH Keys for Remote Execution

Satellite Server or Capsule needs SSH key-based access to remote hosts to perform Remote Execution jobs. There are multiple methods to distribute the public key.

### Method 1: Download SSH Key from Satellite API

Create the `~/.ssh` directory on the remote host if it does not exist, then download and append the public key from the Satellite or Capsule API:

```bash
# Create the .ssh directory (if missing)
mkdir -p ~/.ssh

# Download the public key from Satellite/Capsule
curl https://satellite.example.com:9090/ssh/pubkey >> ~/.ssh/authorized_keys
```

Set proper permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Method 2: Copy SSH Key Manually with ssh-copy-id

From the Satellite or Capsule server, use `ssh-copy-id` to push the foreman-proxy key to the remote host:

```bash
ssh-copy-id -i ~foreman-proxy/.ssh/id_rsa_foreman_proxy.pub root@remote-host.example.com
```

### Method 3: Manual Copy

If neither method works, manually copy the public key content from the Satellite/Capsule into the remote host's `authorized_keys`:

```bash
# On Satellite/Capsule — view the public key
cat ~foreman-proxy/.ssh/id_rsa_foreman_proxy.pub

# On the remote host — paste the key into authorized_keys
vi ~/.ssh/authorized_keys

# Set permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

## Verify SSH Key Access

Once the key is deployed, test passwordless SSH from the Satellite or Capsule server to the remote host:

```bash
ssh -i ~foreman-proxy/.ssh/id_rsa_foreman_proxy root@remote-host.example.com
```

You should get a shell without being prompted for a password.

## Run a Test Job from Satellite GUI

1. Navigate to **Monitor -> Jobs -> Run Job**
2. Apply a filter to select the target host(s)
3. Enter a simple command to test (e.g., `uptime`, `date`, `hostname`)
4. Submit the job

The job details screen shows execution status. Click on individual hosts to view command output.

## Running Commands via Hammer CLI

Hammer CLI provides a powerful way to run Remote Execution jobs from the command line without using the Satellite GUI.

### Enable SSH Push Mode

Ensure Remote Execution is configured on the Satellite or Capsule:

```bash
satellite-installer --foreman-proxy-plugin-remote-execution-script-mode ssh

# Verify the REX SSH key exists
ls ~foreman-proxy/.ssh/id_rsa_foreman_proxy.pub
```

### Run a Command on a Single Host

```bash
hammer job-invocation create \
    --job-template "Run Command - Script Default" \
    --inputs "command=uptime" \
    --search-query "name = webserver1.example.com"
```

Check the result:

```bash
hammer job-invocation output \
    --id 1 \
    --host "webserver1.example.com"
```

### Run a Command on Multiple Hosts

```bash
# Run on all hosts in a host group
hammer job-invocation create \
    --job-template "Run Command - Script Default" \
    --inputs "command=df -h" \
    --search-query "hostgroup = WebServers"

# Run on hosts matching an OS query
hammer job-invocation create \
    --job-template "Run Command - Script Default" \
    --inputs "command=cat /etc/redhat-release" \
    --search-query "os = RedHat 9.3"
```

### Run a Multi-Line Script

```bash
hammer job-invocation create \
    --job-template "Run Command - Script Default" \
    --inputs "command=#!/bin/bash
echo 'Checking disk usage...'
df -h /
echo 'Checking memory...'
free -m
echo 'Last 5 login attempts:'
last -5" \
    --search-query "hostgroup = WebServers"
```

### Apply Errata via CLI

```bash
hammer job-invocation create \
    --feature katello_errata_install \
    --inputs "errata=RHSA-2026:0123" \
    --search-query "hostgroup = Production"
```

### Run Ansible Roles

```bash
# Import Ansible roles from a Smart Proxy (use 'fetch' to list available roles first)
hammer ansible roles import --proxy-id 1

# Run assigned Ansible roles on a host
hammer job-invocation create \
    --job-template "Ansible Roles - Ansible Default" \
    --search-query "name = webserver1.example.com"
```

### Schedule a Job for Later

```bash
hammer job-invocation create \
    --job-template "Run Command - Script Default" \
    --inputs "command=dnf update -y" \
    --search-query "hostgroup = WebServers" \
    --start-at "2026-03-08 02:00:00"
```

### Monitor Job Status

```bash
# List recent job invocations
hammer job-invocation list

# Check status of a specific job
hammer job-invocation info --id 42

# View output from a specific host
hammer job-invocation output --id 42 --host "webserver1.example.com"
```

### Search Query Examples

The `--search-query` option is powerful and flexible. Common patterns:

| Query | Targets |
|-------|---------|
| `name = webserver1.example.com` | A specific host |
| `os = RedHat and os_major = 9` | All RHEL 9 hosts |
| `os = RedHat and os_major >= 8` | RHEL 8 and later |
| `hostgroup = Production` | All hosts in a host group |
| `registered_at > "4 July 2024"` | Hosts registered after a date |

You can test queries interactively in the Satellite GUI under **Hosts -> All Hosts** using the filter box, and save complex queries as bookmarks for reuse.

### View Output for All Hosts at Once

By default, you need to check output host by host. This bash loop displays output from all hosts in a single command:

```bash
MY_ID=32
for host in $(hammer job-invocation info --id $MY_ID | grep "^ - " | awk '{print $2}'); do
    echo "=== $host ==="
    hammer job-invocation output --id $MY_ID --host $host
    echo
done
```

## Job Templates

Remote Execution uses job templates to define reusable tasks. Satellite ships with built-in templates and you can create custom ones.

### Built-In Templates

| Template | Purpose |
|----------|---------|
| Run Command - SSH Default | Run any arbitrary command |
| Service Action - SSH Default | Start, stop, restart, or check status of a service |
| Ansible Roles - Ansible Default | Run assigned Ansible roles |
| Package Action - SSH Default | Install, update, or remove packages |

### How Job Templates Work

Templates use embedded Ruby (ERB) to handle logic such as OS version differences. For example, the "Service Action - SSH Default" template:

```erb
<% if @host.operatingsystem.family == "Redhat" && @host.operatingsystem.major.to_i > 6 %>
systemctl <%= input("action") %> <%= input("service") %>
<% else %>
service <%= input("service") %> <%= input("action") %>
<% end -%>
```

This automatically uses `systemctl` on RHEL 7+ and the legacy `service` command on RHEL 6 and earlier.

### Running a Job Template from CLI

```bash
# Restart httpd on all RHEL 9 web servers
hammer job-invocation create \
    --job-template "Service Action - SSH Default" \
    --inputs "action=restart,service=httpd" \
    --search-query "hostgroup = WebServers and os_major = 9"
```

### Creating Custom Job Templates

1. Navigate to **Hosts -> Job Templates**
2. Click **New Job Template**
3. Write your ERB template using `<%= input("variable_name") %>` for user inputs
4. Define inputs on the **Job** tab (type, default values, allowed options)
5. Assign the template to appropriate organizations and locations

## Installing Packages Using Remote Execution

Once Remote Execution is configured for content hosts, you can install packages directly from the Satellite GUI:

1. Navigate to **Hosts -> Content Hosts**
2. Select the target host(s)
3. From the **Action** menu, select **Manage Packages**
4. Enter the package name (e.g., `tree`)
5. Select the action: **Install**, **Update**, or **Remove**
6. Choose **via Remote Execution**
7. Submit the job

Verify the installation on the remote host:

```bash
yum list installed tree
```

Example output:

```
Installed Packages
tree.x86_64    1.7.0-15.el8    @rhel-8-for-x86_64-baseos-rpms
```

## Installing Errata Using Remote Execution

1. Navigate to **Hosts -> Content Hosts**
2. Select the target host(s)
3. From the **Action** menu, select **Manage Errata**
4. Select the required errata (or use **Select All**)
5. Click **Install Selected -> via remote execution**
6. Wait for the job to complete

Monitor progress from **Monitor -> Jobs** — logs are available directly in the Satellite GUI without needing to log into individual hosts.

## Advantages Over katello-agent

| Feature | katello-agent (deprecated) | Remote Execution |
|---------|----------------------------|------------------|
| Communication | goferd + AMQP (Qpid) | SSH |
| Agent required | Yes | No |
| Scalability | Limited by message broker | Better (direct SSH) |
| Job templates | No | Yes (customizable) |
| Bulk operations | Limited | Full support |
| Real-time output | No | Yes |

## Using a Non-Root User for Remote Execution

By default, REX connects as `root`. For environments where direct root SSH is restricted, you can configure a dedicated non-root user with passwordless sudo.

### Create the REX User on the Client

```bash
# Create user
useradd rexuser
passwd rexuser

# Grant passwordless sudo
echo "rexuser   ALL=NOPASSWD:   ALL" | tee -a /etc/sudoers.d/rexuser
```

Verify sudo works without a password prompt:

```bash
su - rexuser
sudo yum repolist
```

### Deploy the SSH Key to the Non-Root User

From the Satellite or Capsule server:

```bash
ssh-copy-id -i ~foreman-proxy/.ssh/id_rsa_foreman_proxy.pub rexuser@client.example.com
```

Test the connection:

```bash
ssh -i ~foreman-proxy/.ssh/id_rsa_foreman_proxy rexuser@client.example.com 'sudo yum repolist'
```

### Configure the SSH User in Satellite

You can set the `remote_execution_ssh_user` parameter at multiple levels:

**Per host (GUI):**

1. Navigate to **Hosts -> All Hosts**
2. Edit the target host
3. Go to the **Parameters** tab
4. Add parameter: Name = `remote_execution_ssh_user`, Value = `rexuser`
5. Submit

**Per host (Hammer CLI):**

```bash
hammer host set-parameter \
  --host-id=XX \
  --name='remote_execution_ssh_user' \
  --parameter-type='string' \
  --value='rexuser'
```

**Globally:**

```bash
hammer settings set --name remote_execution_ssh_user --value rexuser
hammer settings set --name remote_execution_effective_user --value root
```

Or via the GUI: **Administer -> Settings -> Remote Execution** tab.

The parameter can also be set at the Host Group, Operating System, Domain, Location, or Organization level.

### Satellite 6.4+: Password-Based Authentication

Starting with Satellite 6.4, Remote Execution can work without pre-deployed SSH keys. When scheduling a job, click **Display advanced fields** and specify:

- **Effective user** — the user to run commands as (e.g., `root`)
- **Password** — SSH password for the remote user
- **Sudo password** — if sudo requires a password

This allows REX jobs to run without configuring SSH keys or `NOPASSWD` in sudoers.

> **Note:** When running Ansible roles with a non-root user, the SSH user and Effective user must be different. Ansible requires "becoming" another user distinct from the login user.

## Troubleshooting

### Connection refused or timeout

```bash
# Verify SSH port is open on the remote host
firewall-cmd --list-ports
firewall-cmd --add-port=22/tcp --permanent
firewall-cmd --reload
```

### Permission denied

```bash
# Check key permissions on the remote host
ls -la ~/.ssh/
# authorized_keys should be 600, .ssh directory should be 700

# Check SELinux context
restorecon -Rv ~/.ssh/
```

### Verify the correct key is deployed

```bash
# On Satellite/Capsule
cat ~foreman-proxy/.ssh/id_rsa_foreman_proxy.pub

# Compare with authorized_keys on the remote host
cat ~/.ssh/authorized_keys
```

### Check foreman-proxy service

```bash
# On Satellite/Capsule
systemctl status foreman-proxy
```

## References

- [Red Hat Satellite — Managing Hosts (official docs)](https://docs.redhat.com/en/documentation/red_hat_satellite/)
- [Getting Started with Remote Execution](https://docs.redhat.com/en/documentation/red_hat_satellite/6.16/html/managing_hosts/configuring_and_setting_up_remote_jobs_managing-hosts)
