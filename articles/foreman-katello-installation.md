# Installing Foreman with Katello on RHEL 9

This guide covers installing Foreman 3.13 with Katello 4.15 on RHEL 9, including prerequisites, configuration, and post-install verification.

> **Note:** For RHEL 8, use Foreman 3.12 / Katello 4.14 and replace `el9` with `el8` in the repository URLs below.

## VM Specification

| Resource | Minimum |
|----------|---------|
| vCPU | 4 |
| RAM | 20 GB |
| Swap | 4 GB |
| Disk | 64 GB |

## Prerequisites

### Set Hostname

```sh
hostnamectl set-hostname foreman.example.com
echo '192.168.50.210 foreman.example.com foreman' >> /etc/hosts
```

Verify:

```sh
hostname -f
# Should return: foreman.example.com
```

The FQDN must resolve correctly — the installer will fail if it cannot determine the hostname.

### Disable SELinux

```sh
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

Reboot for the change to take effect, or set temporarily:

```sh
setenforce 0
```

### Configure Firewall

| Port | Protocol | Required For |
|------|----------|--------------|
| 53 | TCP & UDP | DNS Server |
| 67, 68 | UDP | DHCP Server |
| 69 | UDP | TFTP Server |
| 80, 443 | TCP | HTTP & HTTPS (Foreman web UI) |
| 3000 | TCP | HTTP (standalone WEBrick service) |
| 3306 | TCP | Separate MySQL database |
| 5432 | TCP | Separate PostgreSQL database |
| 5910–5930 | TCP | Server VNC Consoles |
| 8140 | TCP | Puppet Master |
| 8443 | TCP | Smart Proxy (open only to Foreman) |

```sh
firewall-cmd --permanent --add-port=53/tcp
firewall-cmd --permanent --add-port=53/udp
firewall-cmd --permanent --add-port=67-69/udp
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=3000/tcp
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --permanent --add-port=5432/tcp
firewall-cmd --permanent --add-port=5910-5930/tcp
firewall-cmd --permanent --add-port=8140/tcp
firewall-cmd --permanent --add-port=8443/tcp
firewall-cmd --reload
firewall-cmd --list-all
```

Or disable the firewall entirely (lab/homelab environments):

```sh
systemctl disable --now firewalld
```

### Install and Configure NTP

```sh
dnf install -y chrony
systemctl enable --now chronyd
chronyc sources
```

## Installation

### Register RHEL and Enable Repositories

```sh
subscription-manager register --auto-attach --username=<user> --password=<password>
subscription-manager repos --disable "*"
subscription-manager repos \
  --enable=rhel-9-for-x86_64-baseos-rpms \
  --enable=rhel-9-for-x86_64-appstream-rpms
dnf clean all
```

### Install Foreman and Katello Repositories

```sh
dnf -y install https://yum.theforeman.org/releases/3.13/el9/x86_64/foreman-release.rpm
dnf -y install https://yum.theforeman.org/katello/4.15/katello/el9/x86_64/katello-repos-latest.rpm
dnf -y install https://yum.puppet.com/puppet8-release-el-9.noarch.rpm
dnf -y module enable katello:el9
```

### Install Foreman Installer

```sh
dnf -y install foreman-installer-katello
```

### Run the Installer

```sh
foreman-installer --scenario katello \
  --foreman-initial-organization "Home" \
  --foreman-initial-location "Default" \
  --foreman-initial-admin-username admin \
  --foreman-initial-admin-password <password>
```

The installer takes 10–20 minutes depending on hardware. Logs are written to `/var/log/foreman-installer/`.

### Installer Options

| Option | Description |
|--------|-------------|
| `--scenario katello` | Install with Katello (content management) |
| `--foreman-initial-organization` | Default organization name |
| `--foreman-initial-location` | Default location name |
| `--foreman-initial-admin-username` | Admin username |
| `--foreman-initial-admin-password` | Admin password |
| `--foreman-proxy-dns true` | Enable DNS management |
| `--foreman-proxy-dhcp true` | Enable DHCP management |
| `--foreman-proxy-tftp true` | Enable TFTP for PXE boot |

To see all available options:

```sh
foreman-installer --scenario katello --help
```

## Post-Installation

### Verify Health Status

```sh
foreman-maintain service status -b
foreman-maintain health check
hammer ping
```

### Access Web UI

Open a browser and navigate to:

```
https://foreman.example.com
```

Login with the admin credentials set during installation.

### Configure Hammer CLI

```sh
mkdir -p ~/.hammer/cli.modules.d
cat > ~/.hammer/cli.modules.d/foreman.yml << EOF
:foreman:
  :host: 'https://foreman.example.com'
  :username: 'admin'
  :password: '<password>'
EOF
chmod 600 ~/.hammer/cli.modules.d/foreman.yml
```

Test:

```sh
hammer ping
hammer organization list
```

## Adding Content

### Upload Red Hat Manifest

```sh
hammer subscription upload \
  --organization "Home" \
  --file /tmp/manifest.zip
```

### Enable RHEL Repositories

```sh
hammer repository-set enable \
  --organization "Home" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --basearch "x86_64" \
  --releasever "8" \
  --name "Red Hat Enterprise Linux 8 for x86_64 - BaseOS (RPMs)"

hammer repository-set enable \
  --organization "Home" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --basearch "x86_64" \
  --releasever "8" \
  --name "Red Hat Enterprise Linux 8 for x86_64 - AppStream (RPMs)"
```

### Create Ubuntu Repository

Via the web interface or hammer:

```sh
# Create product
hammer product create \
  --organization "Home" \
  --name "Ubuntu 20.04"

# Create repository
hammer repository create \
  --organization "Home" \
  --product "Ubuntu 20.04" \
  --name "Ubuntu 20.04 Main" \
  --content-type "deb" \
  --url "http://archive.ubuntu.com/ubuntu/" \
  --deb-releases "focal,focal-updates,focal-security" \
  --deb-components "main,universe" \
  --deb-architectures "amd64" \
  --download-policy "on_demand" \
  --verify-ssl-on-sync false
```

### Sync Repositories

```sh
# Sync a specific product
hammer product synchronize --organization "Home" --name "Ubuntu 20.04" --async

# Sync all RHEL repos
hammer product synchronize --organization "Home" --name "Red Hat Enterprise Linux for x86_64" --async
```

### Create a Sync Plan

```sh
hammer sync-plan create \
  --organization "Home" \
  --name "Daily Sync" \
  --interval daily \
  --sync-date "$(date '+%Y-%m-%d') 02:00:00" \
  --enabled true

hammer product set-sync-plan \
  --organization "Home" \
  --name "Red Hat Enterprise Linux for x86_64" \
  --sync-plan "Daily Sync"

hammer product set-sync-plan \
  --organization "Home" \
  --name "Ubuntu 20.04" \
  --sync-plan "Daily Sync"
```

## Important Files

| Path | Purpose |
|------|---------|
| `/var/log/foreman-installer/foreman.log` | Installer log |
| `/var/log/foreman-installer/katello.log` | Katello installer log |
| `/etc/foreman-installer/scenarios.d/foreman-answers.yaml` | Installation parameters (answer file) |
| `/var/lib/pulp/` | Content storage (DO NOT modify manually) |
| `/etc/hammer/cli.modules.d/foreman.yml` | Global Hammer config |
| `~/.hammer/cli.modules.d/foreman.yml` | User Hammer credentials |

## Troubleshooting

```sh
# Re-run the installer (safe to run multiple times)
foreman-installer --scenario katello

# Check service status
foreman-maintain service status -b

# Restart all services
foreman-maintain service restart

# Check logs
tail -f /var/log/foreman/production.log
tail -f /var/log/foreman-proxy/proxy.log

# Check disk space (Pulp storage)
df -h /var/lib/pulp

# Run health checks
foreman-maintain health check
```
