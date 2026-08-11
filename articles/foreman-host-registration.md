# Registering Hosts in Foreman / Satellite

This guide covers registering RHEL and Ubuntu hosts to Foreman (with Katello) or Red Hat Satellite for content management, patching, and remote execution.

## Prerequisites

Before registering a host, ensure:

- Foreman/Satellite server is operational (`hammer ping`)
- An activation key exists for the target lifecycle environment and content view
- DNS resolves the Foreman/Satellite FQDN from the host
- Port 443 is open between the host and the Foreman/Satellite server (or Capsule)

## Registration Methods

| Method | Best For |
|--------|----------|
| Global Registration (curl command) | Simplest, works for both RHEL and Ubuntu |
| subscription-manager | RHEL/CentOS traditional registration |
| Bootstrap script | Legacy hosts, bulk migration |
| Provisioning | Hosts built by Foreman (auto-registered) |

## Method 1: Global Registration (Recommended)

Available since Foreman 3.0+. Generates a single `curl` command from the web UI that handles all steps automatically.

### Generate Registration Command (Web UI)

1. Navigate to **Hosts → Register Host**
2. Select:
   - **Organization** and **Location**
   - **Host Group** (optional)
   - **Activation Key(s)**
   - **Insecure** (skip CA verification — useful for initial setup)
3. Click **Generate**
4. Copy the generated `curl` command

### Generate Registration Command (Hammer)

```bash
hammer registration generate-command \
  --organization "ACME" \
  --location "Default Location" \
  --activation-key "rhel9-production" \
  --insecure true
```

### Run on the Host

The generated command looks like:

```bash
curl -sS 'https://foreman.example.com/register?activation_keys=rhel9-production&organization_id=1&location_id=1' \
  -H 'Authorization: Bearer <token>' | bash
```

This single command:

1. Downloads and installs the Foreman CA certificate
2. Installs `subscription-manager` (or `rhsm` for Ubuntu)
3. Registers the host with the specified activation key
4. Configures repositories from the content view
5. Installs and configures `katello-host-tools`

## Method 2: RHEL Registration with subscription-manager

### Install the CA Certificate

```bash
# Download and install Foreman CA cert
curl -o /etc/pki/rpm-gpg/RPM-GPG-KEY-foreman \
  https://foreman.example.com/pub/katello-server-ca.crt

# Install the katello CA consumer RPM
yum -y install https://foreman.example.com/pub/katello-ca-consumer-latest.noarch.rpm
```

### Register the Host

```bash
# Register with activation key (no username/password needed)
subscription-manager register \
  --org="ACME" \
  --activationkey="rhel9-production"

# Or register with multiple activation keys
subscription-manager register \
  --org="ACME" \
  --activationkey="rhel9-base,enable-epel,enable-monitoring"
```

### Install Katello Host Tools

```bash
yum -y install katello-host-tools
yum -y install katello-host-tools-tracer    # optional: tracks services needing restart
yum -y install katello-agent                # deprecated, use REX instead
```

### Verify Registration

```bash
# Check subscription status
subscription-manager status
subscription-manager identity

# List enabled repos
subscription-manager repos --list-enabled

# Check Foreman sees the host
hammer host info --name "$(hostname -f)"
```

## Method 3: Ubuntu Registration

### Using Global Registration (Recommended)

The Global Registration method works for Ubuntu out of the box:

```bash
# Generated from Hosts → Register Host in the web UI
curl -sS 'https://foreman.example.com/register?activation_keys=ubuntu22-production&organization_id=1&location_id=1' \
  -H 'Authorization: Bearer <token>' | bash
```

### Manual Ubuntu Registration

#### Install the CA Certificate

```bash
# Download and install Foreman CA cert
curl -o /usr/local/share/ca-certificates/foreman-ca.crt \
  https://foreman.example.com/pub/katello-server-ca.crt
update-ca-certificates
```

#### Install subscription-manager

```bash
# Install subscription-manager (available in Ubuntu repos)
apt update
apt install -y subscription-manager

# Or install from Foreman's consumer RPM equivalent
curl -o /tmp/katello-rhsm-consumer.deb \
  https://foreman.example.com/pub/katello-rhsm-consumer-latest.deb
dpkg -i /tmp/katello-rhsm-consumer.deb
```

#### Configure RHSM for Foreman

```bash
# /etc/rhsm/rhsm.conf
[server]
hostname = foreman.example.com
port = 443
prefix = /rhsm

[rhsm]
baseurl = https://foreman.example.com/pulp/deb/
repo_ca_cert = /etc/rhsm/ca/katello-server-ca.pem
full_refresh_on_yum = 1
```

#### Register

```bash
subscription-manager register \
  --org="ACME" \
  --activationkey="ubuntu22-production"
```

#### Configure APT Repositories

After registration, the repositories are configured in:

```bash
# Check configured repos
cat /etc/apt/sources.list.d/rhsm.sources

# Update package lists
apt update
```

### Verify Ubuntu Registration

```bash
subscription-manager status
subscription-manager identity
subscription-manager repos --list-enabled
apt update
```

## Registration via Capsule (Smart Proxy)

For hosts that can't reach the main Foreman server directly, register through a Capsule:

### RHEL via Capsule

```bash
# Install CA cert from the Capsule
yum -y install https://capsule01.example.com/pub/katello-ca-consumer-latest.noarch.rpm

# Register pointing to the Capsule
subscription-manager register \
  --org="ACME" \
  --activationkey="rhel9-production" \
  --serverurl="https://capsule01.example.com:8443/rhsm" \
  --baseurl="https://capsule01.example.com/pulp/repos"
```

### Global Registration via Capsule

In the registration form, select the Capsule as the **Smart Proxy** before generating the command.

## Host Group Assignment

Assign a host group during registration to automatically configure:

- Puppet classes / Ansible roles
- Content view and lifecycle environment
- Provisioning settings
- Parameters

```bash
# Register with host group via Global Registration
hammer registration generate-command \
  --organization "ACME" \
  --activation-key "rhel9-production" \
  --hostgroup "Production-Servers"
```

## Activation Key Setup for Registration

### RHEL Activation Key

```bash
hammer activation-key create \
  --name "rhel9-production" \
  --organization "ACME" \
  --lifecycle-environment "Production" \
  --content-view "RHEL9-CCV" \
  --unlimited-hosts

# Add subscription
hammer activation-key add-subscription \
  --name "rhel9-production" \
  --organization "ACME" \
  --subscription-id <sub-id>

# Enable specific repos by default
hammer activation-key content-override \
  --name "rhel9-production" \
  --organization "ACME" \
  --content-label "rhel-9-for-x86_64-appstream-rpms" \
  --value 1
```

### Ubuntu Activation Key

```bash
hammer activation-key create \
  --name "ubuntu22-production" \
  --organization "ACME" \
  --lifecycle-environment "Production" \
  --content-view "Ubuntu22-CCV" \
  --unlimited-hosts
```

## Post-Registration Steps

### Install Remote Execution SSH Key

If using REX (Remote Execution), distribute the SSH key:

```bash
# Automatically via the registration command (if configured)
# Or manually:
curl https://foreman.example.com:9090/ssh/pubkey >> /root/.ssh/authorized_keys
```

### Enable katello-host-tools (RHEL)

```bash
yum -y install katello-host-tools
systemctl enable --now goferd    # if using katello-agent (deprecated)
```

### Verify Host Appears in Foreman

```bash
# From the Foreman server
hammer host list --search "name = newhost.example.com"
hammer host info --name "newhost.example.com"
```

## Unregistering Hosts

### RHEL

```bash
# Unregister from Foreman
subscription-manager unregister

# Clean local subscription data
subscription-manager clean

# Remove the CA certificate
yum -y remove katello-ca-consumer-*
```

### Ubuntu

```bash
subscription-manager unregister
subscription-manager clean
apt remove -y katello-ca-consumer* 2>/dev/null
rm -f /etc/apt/sources.list.d/rhsm.sources
apt update
```

### Remove from Foreman (server side)

```bash
hammer host delete --name "oldhost.example.com"
```

## Bulk Registration

### Using a Script

```bash
#!/bin/bash
# Register multiple RHEL hosts
HOSTS="host1.example.com host2.example.com host3.example.com"
AK="rhel9-production"
ORG="ACME"

for host in $HOSTS; do
  echo "Registering $host..."
  ssh root@$host "
    yum -y install https://foreman.example.com/pub/katello-ca-consumer-latest.noarch.rpm
    subscription-manager register --org='$ORG' --activationkey='$AK'
    yum -y install katello-host-tools
  "
done
```

### Using Foreman REX

```bash
# Run registration on multiple unregistered hosts (if SSH access exists)
hammer job-invocation create \
  --job-template "Run Command - SSH Default" \
  --inputs "command=curl -sS 'https://foreman.example.com/register?activation_keys=rhel9-production&organization_id=1' -H 'Authorization: Bearer <token>' | bash" \
  --search-query "hostgroup = Unregistered"
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `Unable to verify server's identity` | CA cert not installed | Install katello-ca-consumer RPM first |
| `Network error` | Can't reach Foreman on 443 | Check DNS, firewall, `curl -v https://foreman.example.com` |
| `Invalid credentials` | Wrong org or AK name | Verify with `hammer activation-key list --organization "ACME"` |
| `No subscriptions available` | AK has no subscriptions attached | Add subscriptions to the activation key |
| `Repo not found` after registration | Content view not published/promoted | Publish and promote the CV to the correct lifecycle |
| Host doesn't appear in Foreman | Registration succeeded but host tools missing | Install `katello-host-tools`, check `rhsmcertd` is running |

### Debug Registration

```bash
# Verbose registration
subscription-manager register --org="ACME" --activationkey="my-key" --debug

# Check RHSM configuration
subscription-manager config

# Check RHSM identity
subscription-manager identity

# Check RHSM logs
tail -f /var/log/rhsm/rhsm.log

# Test connectivity to Foreman
curl -v https://foreman.example.com/rhsm

# On the Foreman server, check for the host
hammer host list --search "name ~ newhost"
```

## Additional Setup Notes

### Edit Hosts File (if hostname not in DNS)

If the Foreman server is not resolvable via DNS, add it to `/etc/hosts` on the client:

```bash
echo '192.168.50.210 foreman.example.com' >> /etc/hosts
```

### Disable Certificate Validation (Lab/Testing)

If using self-signed certificates, disable validation in RHSM:

```bash
sed -i 's/insecure = 0/insecure = 1/' /etc/rhsm/rhsm.conf
```

Or in `/etc/rhsm/rhsm.conf`:

```ini
[server]
insecure = 1
```

### Generate Registration Command (Alternative Syntax)

```bash
# Older hammer versions use host-registration subcommand
hammer host-registration generate-command --activation-keys "rhel9_reg_key" --insecure true

# Newer hammer versions
hammer registration generate-command --activation-key "rhel9_reg_key" --insecure true
```

### Interactive Registration (with Environment Selection)

If no activation key is used, `subscription-manager` prompts for environment:

```
$ subscription-manager register
Registering to: foreman.example.com:443/rhsm
Username: admin
Password:
Hint: Organization "Home" contains following environments: Library, Library/rhel_9, dev/rhel_9
Environment: dev/rhel_9
The system has been registered with ID: 69d2c2d7-0153-4111-aa4b-6c477846eb0b
The registered system name is: server01
```

Using an activation key bypasses this prompt (key already specifies the environment).

## Ubuntu 20.04 Registration (ATIX Method)

For Ubuntu 20.04 when Global Registration is not available, use the ATIX subscription-manager packages:

```bash
# Add ATIX repository and GPG key
wget -qO - https://apt.atix.de/atix_gpg.pub | apt-key add -
add-apt-repository 'deb https://apt.atix.de/Ubuntu20LTS/ stable main'

# Install subscription-manager
apt install -y wget python3-subscription-manager

# Download and run the katello-rhsm-consumer script
cd /root
wget --no-check-certificate -O katello-rhsm-consumer http://foreman.example.com/pub/katello-rhsm-consumer
bash katello-rhsm-consumer

# Disable certificate validation (if using self-signed certs)
sed -i 's/insecure = 0/insecure = 1/' /etc/rhsm/rhsm.conf

# Register
subscription-manager register --org="ACME" --activationkey="ubuntu20_reg_key"

# Optionally disable default Ubuntu repositories (use only Foreman content)
sed -i 's/^/#/' /etc/apt/sources.list
apt update
```

Reference: [ATIX Ubuntu Support Documentation](https://oss.atix.de/html/ubuntu.html)

## Quick Reference

```bash
# RHEL registration (activation key)
yum -y localinstall https://foreman.example.com/pub/katello-ca-consumer-latest.noarch.rpm
subscription-manager register --org="ACME" --activationkey="rhel9-production"
yum -y install katello-host-tools

# Ubuntu registration (global registration)
curl -sS 'https://foreman.example.com/register?activation_keys=ubuntu22-production&organization_id=1' -H 'Authorization: Bearer <token>' | bash

# Unregister
subscription-manager unregister && subscription-manager clean

# Verify
subscription-manager status
subscription-manager repos --list-enabled
```
