# Installing Red Hat Satellite from ISO on RHEL 9

This guide covers installing Red Hat Satellite 6.16 on RHEL 9 using the disconnected (ISO) method — useful when the Satellite server has no internet access.

## Prerequisites

### System Requirements

| Resource | Minimum |
|----------|---------|
| CPU | 4 cores |
| RAM | 20 GB |
| Swap | 4 GB |
| `/` (root) | 30 GB |
| `/var` | 50 GB (content storage grows over time) |
| `/var/lib/pulp` | Depends on synced content (100 GB+ recommended) |

### Required ISOs

Download from the [Red Hat Customer Portal](https://access.redhat.com/downloads):

1. **Red Hat Enterprise Linux 9 Binary DVD** — for base OS installation
2. **Red Hat Satellite 6.16 ISO** — contains all Satellite packages

### Set Hostname

```bash
hostnamectl set-hostname satellite.example.com
echo "192.168.1.100 satellite.example.com satellite" >> /etc/hosts

# Verify
hostname -f
# Should return: satellite.example.com
```

### Configure Chronyd (NTP)

```bash
dnf install -y chrony
systemctl enable --now chronyd
chronyc sources
```

### Disable Firewall (Lab) or Open Ports

```bash
# Option 1: Disable firewall (lab environments only)
systemctl disable --now firewalld

# Option 2: Open required ports
firewall-cmd --permanent --add-port={53/udp,53/tcp,67/udp,68/udp,69/udp,80/tcp,443/tcp,5000/tcp,5647/tcp,8000/tcp,8140/tcp,8443/tcp,9090/tcp}
firewall-cmd --reload
firewall-cmd --list-all
```

Required ports:

| Port | Protocol | Service |
|------|----------|---------|
| 53 | TCP/UDP | DNS |
| 67, 68 | UDP | DHCP |
| 69 | UDP | TFTP |
| 80, 443 | TCP | HTTP/HTTPS (Web UI) |
| 5000 | TCP | Docker registry |
| 5647 | TCP | Qpid (Capsule communication) |
| 8000 | TCP | Anaconda provisioning |
| 8140 | TCP | Puppet |
| 8443 | TCP | Smart Proxy (subscription-manager) |
| 9090 | TCP | Smart Proxy HTTPS |

## Step 1: Register RHEL and Enable Base Repos

Even for disconnected installs, the base RHEL repos are needed for dependencies:

```bash
# Register the system
subscription-manager register --username=<user> --password=<password>
subscription-manager attach --pool=<satellite-pool-id>

# Disable all repos, enable only what's needed
subscription-manager repos --disable "*"
subscription-manager repos \
  --enable=rhel-9-for-x86_64-baseos-rpms \
  --enable=rhel-9-for-x86_64-appstream-rpms \
  --enable=satellite-6.16-for-rhel-9-x86_64-rpms \
  --enable=satellite-maintenance-6.16-for-rhel-9-x86_64-rpms
```

If the server is fully disconnected (no internet), skip registration and use the RHEL DVD as a local repo instead (see next section).

## Step 2: Configure RHEL DVD as Local Repository (Disconnected)

If the system cannot reach Red Hat CDN:

```bash
# Mount the RHEL 9 DVD ISO
mkdir -p /mnt/rhel-dvd
mount -o loop /path/to/rhel-9-x86_64-dvd.iso /mnt/rhel-dvd

# Create a local repo file
cat > /etc/yum.repos.d/rhel9-dvd.repo << 'EOF'
[rhel9-dvd-baseos]
name=RHEL 9 DVD - BaseOS
baseurl=file:///mnt/rhel-dvd/BaseOS
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[rhel9-dvd-appstream]
name=RHEL 9 DVD - AppStream
baseurl=file:///mnt/rhel-dvd/AppStream
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF

dnf clean all
dnf repolist
```

## Step 3: Mount and Install the Satellite ISO

```bash
# Copy the ISO to the server (if needed)
scp satellite-6.16-rhel-9-x86_64.iso root@satellite.example.com:/tmp/

# Import Red Hat GPG key
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

# Create mount point
mkdir -p /mnt/satellite-iso

# Mount the Satellite ISO
mount -o loop /tmp/satellite-6.16-rhel-9-x86_64.iso /mnt/satellite-iso

# Run the installer script from the ISO
cd /mnt/satellite-iso
./install_packages

# This script:
# - Configures a local yum repository from the ISO content
# - Installs satellite packages
```

The `install_packages` script creates a repo file at `/etc/yum.repos.d/satellite-local.repo` and installs the `satellite` package group.

### Alternative: Manual Package Install

If the script fails or you prefer manual control:

```bash
# Create local repo from ISO
cat > /etc/yum.repos.d/satellite-local.repo << 'EOF'
[satellite-local]
name=Satellite 6.16 Local
baseurl=file:///mnt/satellite-iso
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF

dnf clean all
dnf install -y satellite
```

## Step 4: Run the Satellite Installer

```bash
satellite-installer --scenario satellite \
  --foreman-initial-organization "MyOrg" \
  --foreman-initial-location "MyLocation" \
  --foreman-initial-admin-username admin \
  --foreman-initial-admin-password "SecurePassword123"
```

The installer takes 15–30 minutes. Logs are written to `/var/log/foreman-installer/satellite.log`.

### Additional Installer Options

```bash
# With DNS, DHCP, and TFTP enabled
satellite-installer --scenario satellite \
  --foreman-initial-organization "MyOrg" \
  --foreman-initial-location "MyLocation" \
  --foreman-initial-admin-username admin \
  --foreman-initial-admin-password "SecurePassword123" \
  --foreman-proxy-dns true \
  --foreman-proxy-dns-managed true \
  --foreman-proxy-dns-zone "example.com" \
  --foreman-proxy-dns-reverse "1.168.192.in-addr.arpa" \
  --foreman-proxy-dhcp true \
  --foreman-proxy-dhcp-managed true \
  --foreman-proxy-dhcp-range "192.168.1.100 192.168.1.200" \
  --foreman-proxy-tftp true \
  --foreman-proxy-tftp-managed true
```

To see all options:

```bash
satellite-installer --scenario satellite --help
```

## Step 5: Verify Installation

```bash
# Check all services
satellite-maintain service status

# Health check
satellite-maintain health check

# Ping all components
hammer ping

# Access web UI
echo "https://$(hostname -f)"
```

## Step 6: Import the Subscription Manifest

Download the manifest from the [Red Hat Customer Portal](https://access.redhat.com/management/subscription_allocations) and copy it to the Satellite server.

```bash
hammer subscription upload \
  --organization "MyOrg" \
  --file /path/to/manifest.zip
```

## Step 7: Enable and Sync Repositories

### Using the ISO for Initial Content (Disconnected Sync)

For a disconnected Satellite, you can import content from ISOs using Inter-Satellite Sync (ISS) or content import:

```bash
# On a connected Satellite: export content
hammer content-export complete \
  --organization "MyOrg" \
  --content-view "RHEL9-CV" \
  --version 1.0

# Transfer the export to the disconnected Satellite
# Then import:
hammer content-import version \
  --organization "MyOrg" \
  --path /var/lib/pulp/imports/RHEL9-CV/1.0/
```

### Enable Red Hat Repos (if connected)

```bash
hammer repository-set enable \
  --organization "MyOrg" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --basearch "x86_64" \
  --releasever "9" \
  --name "Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)"

hammer repository-set enable \
  --organization "MyOrg" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --basearch "x86_64" \
  --releasever "9" \
  --name "Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)"

hammer product synchronize \
  --organization "MyOrg" \
  --name "Red Hat Enterprise Linux for x86_64" \
  --async
```

## Post-Installation Configuration

### Configure Hammer CLI

```bash
mkdir -p ~/.hammer/cli.modules.d
cat > ~/.hammer/cli.modules.d/foreman.yml << EOF
:foreman:
  :host: 'https://satellite.example.com'
  :username: 'admin'
  :password: 'SecurePassword123'
EOF
chmod 600 ~/.hammer/cli.modules.d/foreman.yml
```

### Create Lifecycle Environments

```bash
hammer lifecycle-environment create \
  --organization "MyOrg" \
  --name "Development" \
  --prior "Library"

hammer lifecycle-environment create \
  --organization "MyOrg" \
  --name "Production" \
  --prior "Development"
```

### Create a Sync Plan

```bash
hammer sync-plan create \
  --organization "MyOrg" \
  --name "Daily Sync" \
  --interval daily \
  --sync-date "$(date '+%Y-%m-%d') 02:00:00" \
  --enabled true
```

## Unmount ISOs

After installation is complete:

```bash
cd /
umount /mnt/satellite-iso
umount /mnt/rhel-dvd

# Remove if no longer needed
rm -f /etc/yum.repos.d/satellite-local.repo
rm -f /etc/yum.repos.d/rhel9-dvd.repo
```

## Important Files

| Path | Purpose |
|------|---------|
| `/var/log/foreman-installer/satellite.log` | Installer log |
| `/etc/foreman-installer/scenarios.d/satellite-answers.yaml` | Answer file (all installer parameters) |
| `/var/lib/pulp/` | Content storage (DO NOT modify manually) |
| `/etc/hammer/cli.modules.d/foreman.yml` | Global Hammer config |
| `~/.hammer/cli.modules.d/foreman.yml` | User Hammer credentials |
| `/etc/cron.weekly/pulp-maintenance` | Identifies and deletes orphaned content (packages, errata, OSTree) not associated with any repository |

## Errata Terminology

| Term | Definition |
|------|-----------|
| **Applicable** | Errata contains RPMs that can update a system in the environment. One or more systems are known to require it. |
| **Installable** | Errata has been promoted into a lifecycle environment where the affected system has access to it (i.e., `yum update` will install it). |

## Satellite Maintain Commands

```bash
# Service management
satellite-maintain service list
satellite-maintain service status
satellite-maintain service status -b          # brief
satellite-maintain service status -f          # full details
satellite-maintain service status --only rh-mongodb34-mongod
satellite-maintain service stop
satellite-maintain service start
satellite-maintain service restart

# Health checks
satellite-maintain health list
satellite-maintain health check
```

## Managing Paused Tasks

1. Navigate to **Monitor → Tasks**
2. Search for paused tasks: `state = paused`
3. Click on the paused task to see details
4. If a **Resume** button is available, click it

If Resume doesn't work, use the Dynflow console:

1. On the task details screen, click **Dynflow console**
2. Find the blocked subtask (will have a blue **skip** link)
3. Click **skip**, then click **Resume** at the top of the screen
4. Refresh the page — the task should change to `stopped:error` state

## Useful Satellite Commands

```bash
# Incremental update of a content view (add specific packages without full publish)
hammer content-view version incremental-update \
  --content-view-version-id 56 \
  --package-ids 11165

# Apply a security errata to a host
hammer host errata apply --host server01.example.com --errata-ids RHSA-2019:3755

# List all RHEL hosts with a specific package installed
hammer host list --search 'installed_package = sudo-1.8.6p3-29.el6_10.2.x86_64 and os_major = 6'

# List all RHEL hosts where a specific package is NOT installed
hammer host list --search 'not installed_package = sudo-1.8.6p3-29.el6_10.2.x86_64 and os_major = 6'
```

## Enabling a Repository on a Client Host

First add the repository to the Content View in the web UI and re-publish the content view. Then on the client:

```bash
# List all available repos (enabled and disabled)
yum repolist all

# Enable a specific repo
subscription-manager repos --enable=rhel-9-for-x86_64-appstream-rpms
```

## Troubleshooting

```bash
# Re-run installer (safe to run multiple times)
satellite-installer --scenario satellite

# Check service status
satellite-maintain service status

# Restart all services
satellite-maintain service restart

# Check logs
tail -f /var/log/foreman/production.log
tail -f /var/log/foreman-proxy/proxy.log
journalctl -u foreman -f

# Check disk space
df -h /var/lib/pulp

# Health check
satellite-maintain health check

# Verify selinux isn't blocking (if enabled)
ausearch -m AVC -ts recent
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Installer fails at DB migration | Insufficient RAM | Ensure 20 GB+ RAM |
| `package not found` during install | ISO not mounted or repo not configured | Verify `dnf repolist` shows satellite-local |
| Web UI not accessible | Firewall blocking 443 | Open port or disable firewall |
| `satellite-maintain` not found | Package not installed | `dnf install rubygem-foreman_maintain` |
| Slow installer | Low disk I/O | Use SSD for `/var/lib/pulp` |
