<p align="center">
  <img src="/cheatsheets/cloud-init/images/cloud-init-logo.svg" alt="cloud-init logo" width="300"/>
</p>

<h1 align="center">cloud-init Cheatsheet</h1>

Comprehensive cloud-init reference guide covering instance initialization, user-data formats, module configuration, networking, debugging, and common cloud-config patterns for provisioning cloud instances across all major providers.

## Basic Structure

```yaml
#cloud-config
# Comments start with #
# YAML format is required
```

## Overview

cloud-init is the industry standard for cross-platform cloud instance initialization. It runs during early boot to configure networking, storage, users, packages, and custom scripts — driven by user-data provided at instance launch.

### Supported Platforms

| Provider | Datasource |
|----------|-----------|
| AWS EC2 | `DataSourceEc2` |
| Azure | `DataSourceAzure` |
| GCP | `DataSourceGCE` |
| OpenStack | `DataSourceOpenStack` |
| Oracle Cloud | `DataSourceOracle` |
| DigitalOcean | `DataSourceDigitalOcean` |
| Hetzner | `DataSourceHetzner` |
| Vultr | `DataSourceVultr` |
| VMware (guestinfo) | `DataSourceVMware` |
| NoCloud (local/ISO) | `DataSourceNoCloud` |
| LXD | `DataSourceLXD` |

### Boot Stages

| Stage | Systemd Service | Description |
|-------|----------------|-------------|
| Detect | `cloud-init-generator` | Runs ds-identify to detect platform |
| Local | `cloud-init-local.service` | Applies network config, no network available yet |
| Network (init) | `cloud-init-network.service` | Network up, runs `cloud_init_modules` (bootcmd, write_files, mounts, users, ssh) |
| Config | `cloud-config.service` | Runs `cloud_config_modules` (locale, ntp, apt, runcmd) |
| Final | `cloud-final.service` | Runs `cloud_final_modules` (packages, scripts, phone_home) |

> **Note:** On older distributions (Ubuntu 22.04 and earlier), the Network stage service is named `cloud-init.service`.

## Installation

| Distro | Command |
|--------|---------|
| Ubuntu/Debian | `sudo apt install cloud-init` |
| RHEL/CentOS/Fedora | `sudo dnf install cloud-init` |
| Alpine | `apk add cloud-init` |
| Arch | `pacman -S cloud-init` |
| openSUSE | `zypper install cloud-init` |

### Check Version

```bash
cloud-init --version
```

## CLI Commands

### Status & Information

| Command | Description |
|---------|-------------|
| `cloud-init status` | Show current status (running, done, error, disabled) |
| `cloud-init status --long` | Detailed status with extended info |
| `cloud-init status --wait` | Block until cloud-init completes |
| `cloud-init status --format json` | Machine-readable status output |
| `cloud-init query instance_id` | Query instance metadata |
| `cloud-init query region` | Query region |
| `cloud-init query cloud_name` | Query cloud provider name |
| `cloud-init query ds.meta_data` | Query full datasource metadata |
| `cloud-init query userdata` | Query raw user-data |
| `cloud-init query --format '{{v1.cloud_name}}'` | Custom format query |
| `cloud-init query --list-keys` | List available query keys |
| `cloud-init query --all` | Dump all instance data |

### Validation & Testing

| Command | Description |
|---------|-------------|
| `cloud-init schema --config-file config.yaml` | Validate cloud-config file |
| `cloud-init schema --system` | Validate system cloud-config |
| `cloud-init schema --config-file config.yaml --annotate` | Validate with inline error annotations |
| `cloud-init devel render /path/to/userdata.yaml` | Render user-data with Jinja templates |
| `cloud-init devel hotlog-hook` | Live tail cloud-init log |

### Re-running cloud-init

| Command | Description |
|---------|-------------|
| `sudo cloud-init clean` | Remove logs and artifacts (re-runs on next boot) |
| `sudo cloud-init clean --reboot` | Clean and immediately reboot |
| `sudo cloud-init clean --logs` | Remove only logs, keep instance state |
| `sudo cloud-init clean && sudo cloud-init init --local` | Clean state and re-run local stage |
| `sudo cloud-init single --name runcmd --frequency once` | Re-run a single module |
| `sudo cloud-init single --name set_hostname --frequency always` | Re-run module ignoring frequency |
| `sudo cloud-init init` | Re-run the init stage |
| `sudo cloud-init modules --mode config` | Re-run config stage modules |
| `sudo cloud-init modules --mode final` | Re-run final stage modules |

### Analysis & Debugging

| Command | Description |
|---------|-------------|
| `cloud-init analyze show` | Show timestamps for each module |
| `cloud-init analyze blame` | Show slowest modules (descending) |
| `cloud-init analyze dump` | Dump raw event data |
| `cloud-init analyze boot` | Show time from kernel boot to cloud-init completion |
| `cloud-init collect-logs` | Collect debug tarball for bug reports |
| `cloud-init features` | List supported features |
| `tail -f /var/log/cloud-init.log` | Follow main cloud-init log |
| `tail -f /var/log/cloud-init-output.log` | Follow script output log |
| `journalctl -u cloud-init.service -f` | Follow cloud-init service journal |
| `grep -i error /var/log/cloud-init.log` | Search for errors in log |

## User-Data Formats

cloud-init accepts multiple user-data formats, identified by the first line:

| Format | First Line / Marker | Description |
|--------|-------------------|-------------|
| Cloud-config | `#cloud-config` | YAML configuration (most common) |
| Shell script | `#!/bin/bash` (or any shebang) | Runs as a script in final stage |
| Include file | `#include` | List of URLs to fetch and process |
| Cloud boothook | `#cloud-boothook` | Runs very early, every boot (before bootcmd) |
| Part handler | `#part-handler` | Python code to handle custom MIME parts |
| Jinja template | `## template: jinja` | Jinja2 templated cloud-config |
| MIME multipart | `Content-Type: multipart/mixed` | Combine multiple formats |
| Gzip compressed | (binary gzip header) | Any format above, gzip-compressed |

### MIME Multipart Example

```bash
# Create a multipart user-data combining cloud-config and a script
cat > userdata.txt <<'EOF'
Content-Type: multipart/mixed; boundary="==BOUNDARY=="
MIME-Version: 1.0

--==BOUNDARY==
Content-Type: text/cloud-config; charset="us-ascii"

#cloud-config
packages:
  - nginx

--==BOUNDARY==
Content-Type: text/x-shellscript; charset="us-ascii"

#!/bin/bash
echo "Hello from script" > /tmp/hello.txt

--==BOUNDARY==--
EOF
```

## Cloud-Config Reference

### Users & Groups

```yaml
#cloud-config
users:
  - name: deploy
    groups: [sudo, docker]
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: true
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... user@host

  - name: developer
    groups: docker
    shell: /bin/bash
    lock_passwd: false
    passwd: $6$rounds=4096$saltsalt$hash...

  - name: appuser
    system: true
    shell: /usr/sbin/nologin
    no_create_home: true

groups:
  - docker
  - monitoring
```

### Default User Configuration

```yaml
system_info:
  default_user:
    name: ubuntu
    lock_passwd: true
    gecos: Ubuntu
    groups: [adm, audio, cdrom, dialout, dip, floppy, lxd, netdev, plugdev, sudo, video]
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    shell: /bin/bash
```

### SSH Configuration

```yaml
#cloud-config
ssh_authorized_keys:
  - ssh-ed25519 AAAA... admin@workstation
  - ssh-rsa AAAAB3NzaC1yc2E... user1@example.com

# Disable password auth
ssh_pwauth: false

# Disable root login
disable_root: true

# Regenerate host keys
ssh_deletekeys: true
ssh_genkeytypes: [ed25519, rsa]

# Import keys from GitHub/Launchpad
ssh_import_id:
  - gh:username
  - lp:username
```

### SSH Host Keys

```yaml
#cloud-config
ssh_keys:
  rsa_private: |
    -----BEGIN RSA PRIVATE KEY-----
    MIIEowIBAAKCAQEA...
    -----END RSA PRIVATE KEY-----
  rsa_public: ssh-rsa AAAAB3NzaC1yc2EAAAA...
```

### Package Management

```yaml
#cloud-config
# Update apt database
package_update: true

# Upgrade installed packages
package_upgrade: true

# Reboot if required after upgrade
package_reboot_if_required: true

# Install packages
packages:
  - nginx
  - curl
  - jq
  - git
  - docker.io
  - python3-pip
  - [python3-pip, 23.0]  # Specific version

# Add apt repositories
apt:
  sources:
    docker.list:
      source: "deb [arch=amd64] https://download.docker.com/linux/ubuntu $RELEASE stable"
      keyid: 9DC858229FC7DD38854AE2D88D81803C0EBFCD88
  conf: |
    APT {
      Get {
        Assume-Yes "true";
        Fix-Broken "true";
      }
    }
```

### Write Files

```yaml
#cloud-config
write_files:
  # Write before other modules (init stage)
  - path: /etc/myapp/config.json
    content: |
      {"port": 8080, "debug": false}
    owner: root:root
    permissions: '0644'

  # Write in final stage (after packages installed)
  - path: /etc/nginx/conf.d/app.conf
    defer: true
    content: |
      server {
          listen 80;
          server_name app.example.com;
          location / { proxy_pass http://127.0.0.1:8080; }
      }

  # Binary content (base64)
  - path: /usr/local/bin/tool
    encoding: b64
    content: f0VMRgIBAQAAAA...
    permissions: '0755'

  # Gzip + base64 encoding
  - path: /tmp/binary_file
    content: !!binary |
      H4sIAIxdvF8AA+3BMQEAAADCoPVPbQwfoAAAAAAAAAAAAAAAAAAAAIC3AYbSVKsAQAAA
    encoding: gzip+base64

  # Append to existing file
  - path: /etc/hosts
    append: true
    content: |
      10.0.0.5 db.internal

  # Environment variables
  - path: /etc/environment
    content: |
      PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
      EDITOR=nano
      JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
    append: true
```

### bootcmd & runcmd

```yaml
#cloud-config
# Runs early, every boot (Network stage)
bootcmd:
  - sysctl -w vm.swappiness=10
  - [cloud-init-per, once, fmt-disk, mkfs.ext4, /dev/nvme1n1]
  - [cloud-init-per, always, mnt-disk, mount, /dev/nvme1n1, /data]
  - echo 'Early boot command' > /tmp/bootcmd.log
  - mount /dev/sdb1 /mnt

# Runs once per instance (Final stage)
runcmd:
  - echo 'Hello World' > /tmp/hello.txt
  - [systemctl, enable, --now, nginx]
  - systemctl enable docker
  - systemctl start docker
  - usermod -aG docker ubuntu
  - 'curl -fsSL https://get.docker.com | sh'
  - [wget, "-O", "/tmp/file.tar.gz", "https://example.com/file.tar.gz"]
```

### Disk Setup & Mounts

```yaml
#cloud-config
disk_setup:
  /dev/nvme1n1:
    table_type: gpt
    layout: true
    overwrite: false

fs_setup:
  - label: data
    filesystem: ext4
    device: /dev/nvme1n1p1

mounts:
  - [/dev/nvme1n1p1, /data, ext4, "defaults,nofail", "0", "2"]
  - ["/dev/sdb1", "/mnt/data", "ext4", "defaults", "0", "2"]
  - [tmpfs, /run/shm, tmpfs, "defaults,size=512M", "0", "0"]
  - ["tmpfs", "/tmp", "tmpfs", "nodev,nosuid,size=1G", "0", "0"]

# Swap file
swap:
  filename: /swapfile
  size: 2G
  maxsize: 4G
```

### Networking

```yaml
#cloud-config
# Manage /etc/hosts
manage_etc_hosts: true
hostname: webserver
fqdn: webserver.internal.example.com

# Prefer IPv4
prefer_fqdn_over_hostname: true

# Locale and keyboard
locale: en_US.UTF-8
keyboard:
  layout: us
  variant: ""

# Disable cloud-init network config (use your own)
network:
  config: disabled
```

### Network Config v2 (Netplan-style)

```yaml
#cloud-config
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
    eth1:
      addresses:
        - 10.0.0.10/24
      routes:
        - to: 10.10.0.0/16
          via: 10.0.0.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

### Static IP Configuration

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

### Timezone & NTP

```yaml
#cloud-config
timezone: Europe/Bucharest

ntp:
  enabled: true
  servers:
    - 0.pool.ntp.org
    - 1.pool.ntp.org
  pools:
    - ntp.ubuntu.com
```

### CA Certificates

```yaml
#cloud-config
ca_certs:
  trusted:
    - |
      -----BEGIN CERTIFICATE-----
      MIIEkjCCA3qgAwIBAgIQCg...
      -----END CERTIFICATE-----
  remove_defaults: false
```

### Power State

```yaml
#cloud-config
power_state:
  delay: now
  mode: reboot
  message: "Rebooting after initial setup"
  timeout: 30
  condition: true
```

```yaml
# Delayed reboot with message
power_state:
  delay: 30
  mode: reboot
  message: "System will reboot in 30 seconds"
  condition: true
```

### Phone Home

```yaml
#cloud-config
phone_home:
  url: https://provisioning.example.com/callback
  post:
    - instance_id
    - hostname
    - fqdn
  tries: 5
```

### Final Message

```yaml
#cloud-config
final_message: |
  cloud-init completed
  version: $version
  timestamp: $timestamp
  datasource: $datasource
  uptime: $uptime
```

### Ansible Pull

```yaml
#cloud-config
ansible:
  install_method: pip
  package_name: ansible
  run_user: root
  pull:
    url: https://github.com/org/ansible-playbooks.git
    playbook_name: site.yml
    accept_host_key: true
    full: true
```

### Growpart & Resize

```yaml
#cloud-config
growpart:
  mode: auto
  devices: [/]
  ignore_growpart_resize: false

resize_rootfs: true
```

### Debug & Output Configuration

```yaml
#cloud-config
# Enable debug logging
debug: true

# Configure output redirection
output:
  all: ">> /var/log/cloud-init-output.log"
```

## Cloud-Init Module Lists

Full module lists as configured in `/etc/cloud/cloud.cfg`:

```yaml
cloud_init_modules:
  - migrator
  - seed_random
  - bootcmd
  - write-files
  - growpart
  - resizefs
  - disk_setup
  - mounts
  - set_hostname
  - update_hostname
  - update_etc_hosts
  - ca-certs
  - rsyslog
  - users-groups
  - ssh

cloud_config_modules:
  - emit_upstart
  - snap
  - ssh-import-id
  - locale
  - set-passwords
  - grub-dpkg
  - apt-pipelining
  - apt-configure
  - ubuntu-advantage
  - ntp
  - timezone
  - disable-ec2-metadata
  - runcmd
  - byobu

cloud_final_modules:
  - package-update-upgrade-install
  - fan
  - landscape
  - lxd
  - ubuntu-drivers
  - puppet
  - chef
  - mcollective
  - salt-minion
  - rightscale_userdata
  - scripts-vendor
  - scripts-per-once
  - scripts-per-boot
  - scripts-per-instance
  - scripts-user
  - ssh-authkey-fingerprints
  - keys-to-console
  - phone-home
  - final-message
  - power-state-change
```

## cloud-init-per

The `cloud-init-per` utility controls execution frequency of commands within `bootcmd`:

| Frequency | Semaphore Location | Description |
|-----------|-------------------|-------------|
| `once` | `/var/lib/cloud/sem/` | Runs once ever, persists across instance ID changes |
| `instance` | `/var/lib/cloud/instance/sem/` | Runs once per instance, resets on new instance ID |
| `always` | (no semaphore) | Runs every boot |

```bash
# Syntax: cloud-init-per <frequency> <name> <cmd> [args...]
cloud-init-per once    setup-raid  mdadm --assemble /dev/md0 /dev/sdb1 /dev/sdc1
cloud-init-per instance format-disk mkfs.ext4 /dev/nvme1n1
cloud-init-per always  mount-data  mount /dev/nvme1n1 /data
```

## Log Files & Directories

| Path | Description |
|------|-------------|
| `/var/log/cloud-init.log` | Main cloud-init log (module-level detail) |
| `/var/log/cloud-init-output.log` | stdout/stderr from scripts and commands |
| `/run/cloud-init/` | Runtime state (result.json, instance-id, datasource info) |
| `/run/cloud-init/result.json` | Final result summary with errors |
| `/var/lib/cloud/` | Persistent cloud-init state directory |
| `/var/lib/cloud/instance/` | Symlink to current instance data |
| `/var/lib/cloud/instance/sem/` | Per-instance semaphores (module run tracking) |
| `/var/lib/cloud/instance/scripts/` | Scripts written by modules (runcmd, per-boot, etc.) |
| `/var/lib/cloud/instance/user-data.txt` | Raw user-data as received |
| `/var/lib/cloud/instance/vendor-data.txt` | Raw vendor-data |
| `/var/lib/cloud/sem/` | Per-once semaphores (persist across instances) |
| `/var/lib/cloud/data/` | Instance IDs, previous datasource info |
| `/var/lib/cloud/scripts/per-boot/` | Scripts that run every boot |
| `/var/lib/cloud/scripts/per-instance/` | Scripts that run once per instance |
| `/var/lib/cloud/scripts/per-once/` | Scripts that run once ever |
| `/etc/cloud/cloud.cfg` | Main cloud-init config (module lists, datasource) |
| `/etc/cloud/cloud.cfg.d/*.cfg` | Drop-in configuration files |

## Configuration Files

### /etc/cloud/cloud.cfg Structure

```yaml
# Module lists (order matters within each stage)
cloud_init_modules:
  - bootcmd
  - write_files
  - growpart
  - resizefs
  - disk_setup
  - mounts
  - set_hostname
  - update_hostname
  - users_groups
  - ssh

cloud_config_modules:
  - ssh_import_id
  - locale
  - set_passwords
  - apt_configure
  - ntp
  - timezone
  - runcmd

cloud_final_modules:
  - package_update_upgrade_install
  - write_files_deferred
  - puppet
  - chef
  - ansible
  - salt_minion
  - scripts_vendor
  - scripts_per_once
  - scripts_per_boot
  - scripts_per_instance
  - scripts_user
  - phone_home
  - final_message
  - power_state_change

# System info
system_info:
  default_user:
    name: ubuntu
    lock_passwd: true
    gecos: Ubuntu
    groups: [adm, sudo]
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    shell: /bin/bash
```

### Disable cloud-init

| Method | How |
|--------|-----|
| Touch file | `sudo touch /etc/cloud/cloud-init.disabled` |
| Kernel cmdline | Add `cloud-init=disabled` to GRUB |
| Datasource None | Set `datasource_list: [None]` in cloud.cfg |
| Systemd mask | `sudo systemctl mask cloud-init` |

### Restrict Datasources

```bash
# /etc/cloud/cloud.cfg.d/90-datasource.cfg
datasource_list: [NoCloud, None]
```

## Datasource: NoCloud

NoCloud is used for local/offline provisioning (VMs, Packer builds, bare metal):

### Seed Directory

```bash
# Create seed directory
sudo mkdir -p /var/lib/cloud/seed/nocloud

# meta-data (required, can be empty)
echo "instance-id: iid-local01" > /var/lib/cloud/seed/nocloud/meta-data

# user-data
cat > /var/lib/cloud/seed/nocloud/user-data <<'EOF'
#cloud-config
packages: [nginx]
runcmd:
  - systemctl enable --now nginx
EOF
```

### ISO for VMs (QEMU/KVM, VirtualBox)

```bash
# Create the seed ISO
genisoimage -output seed.iso -volid cidata -joliet -rock \
  user-data meta-data network-config

# Or with cloud-localds (from cloud-image-utils)
cloud-localds seed.iso user-data meta-data --network-config=network-config

# Attach to VM
qemu-system-x86_64 -cdrom seed.iso -hda disk.qcow2 ...
```

### Kernel Cmdline / SMBIOS

```bash
# Pass via kernel cmdline
qemu-system-x86_64 ... -append "ds=nocloud;s=http://10.0.0.1/cloud-init/"

# Pass via SMBIOS (for VMs without cmdline access)
qemu-system-x86_64 ... -smbios type=1,serial="ds=nocloud;s=http://10.0.0.1/cloud-init/"
```

## Instance Metadata (IMDS)

### AWS EC2

| Command | Description |
|---------|-------------|
| `TOKEN=$(curl -sX PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")` | Get IMDSv2 token |
| `curl -sH "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id` | Instance ID |
| `curl -sH "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4` | Private IP |
| `curl -sH "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4` | Public IP |
| `curl -sH "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone` | Availability zone |
| `curl -sH "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/user-data` | User-data |

### GCP

| Command | Description |
|---------|-------------|
| `curl -sH "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/name` | Instance name |
| `curl -sH "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/zone` | Zone |
| `curl -sH "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip` | Internal IP |
| `curl -sH "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/attributes/user-data` | User-data |

### Azure

| Command | Description |
|---------|-------------|
| `curl -sH "Metadata: true" "http://169.254.169.254/metadata/instance?api-version=2023-07-01"` | Full instance metadata (JSON) |
| `curl -sH "Metadata: true" "http://169.254.169.254/metadata/instance/compute/name?api-version=2023-07-01&format=text"` | VM name |
| `curl -sH "Metadata: true" "http://169.254.169.254/metadata/instance/compute/location?api-version=2023-07-01&format=text"` | Region |

## Debugging

### Common Troubleshooting Steps

```bash
# 1. Check status
cloud-init status --long

# 2. Check for errors in result
cat /run/cloud-init/result.json

# 3. Check what datasource was detected
cat /run/cloud-init/ds-identify.log

# 4. Check module logs
grep -E "(WARNING|ERROR|CRITICAL)" /var/log/cloud-init.log

# 5. Check script output
cat /var/log/cloud-init-output.log

# 6. Check user-data as received
cat /var/lib/cloud/instance/user-data.txt

# 7. Validate user-data syntax
cloud-init schema --config-file /var/lib/cloud/instance/user-data.txt --annotate

# 8. Check semaphores (what has run)
ls /var/lib/cloud/instance/sem/

# 9. Performance analysis
cloud-init analyze blame
```

### Common Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `status: disabled` | `cloud-init.disabled` file exists or datasource not found | Remove `/etc/cloud/cloud-init.disabled` or check datasource config |
| runcmd didn't run | Already ran (semaphore exists) | `sudo cloud-init clean` and reboot, or use `cloud-init single` |
| User-data ignored | Missing `#cloud-config` header | Add `#cloud-config` as first line |
| Packages not installed | `package_update: true` missing | Add `package_update: true` before `packages:` |
| Network config not applied | Cloud overrides local config | Add `/etc/cloud/cloud.cfg.d/99-disable-network.cfg` with `network: {config: disabled}` |
| write_files content missing in bootcmd | write_files runs after bootcmd | Use `runcmd` instead, or inline the content in bootcmd |
| Script runs only once | Default frequency is `once-per-instance` | Use `bootcmd` with `cloud-init-per always` for every-boot execution |
| Slow boot | Large packages or slow mirrors | Use `cloud-init analyze blame` to identify slow modules |

### Validate Before Deploying

```bash
# Local validation (no instance needed)
cloud-init schema --config-file userdata.yaml

# Check YAML syntax with Python
python3 -c "import yaml; yaml.safe_load(open('userdata.yaml'))"

# Dry-run with LXD
lxc launch ubuntu:22.04 test --config=user.user-data="$(cat userdata.yaml)"
lxc exec test -- cloud-init status --wait
lxc exec test -- cat /var/log/cloud-init-output.log
lxc delete test --force
```

## Module Frequency Reference

| Frequency | Keyword | Behavior |
|-----------|---------|----------|
| Per always | `always` | Runs on every boot |
| Per once | `once` | Runs only once ever |
| Per instance | `once-per-instance` | Runs once per instance ID |

### Override Module Frequency

```yaml
# In /etc/cloud/cloud.cfg — change runcmd to run every boot
cloud_config_modules:
  - [runcmd, always]
```

## Jinja Templating

cloud-init supports Jinja2 templates in user-data:

```yaml
## template: jinja
#cloud-config
hostname: {{ ds.meta_data.local_hostname }}
fqdn: {{ ds.meta_data.local_hostname }}.{{ v1.region }}.internal

runcmd:
{% if v1.cloud_name == "aws" %}
  - echo "Running on AWS in {{ v1.region }}"
{% elif v1.cloud_name == "gce" %}
  - echo "Running on GCP in {{ v1.availability_zone }}"
{% else %}
  - echo "Running on {{ v1.cloud_name }}"
{% endif %}
```

### Available Jinja Variables

| Variable | Description |
|----------|-------------|
| `v1.cloud_name` | Cloud provider (aws, azure, gce, openstack, etc.) |
| `v1.distro` | Distribution name (ubuntu, centos, etc.) |
| `v1.distro_release` | Release codename (jammy, noble, etc.) |
| `v1.distro_version` | Version string (22.04, 24.04) |
| `v1.instance_id` | Instance ID |
| `v1.local_hostname` | Local hostname |
| `v1.machine` | Architecture (x86_64, aarch64) |
| `v1.platform` | Platform identifier |
| `v1.region` | Cloud region |
| `v1.availability_zone` | Availability zone |
| `ds.meta_data` | Full datasource metadata dict |

## Common Patterns

### Bootstrap a Docker Host

```yaml
#cloud-config
package_update: true
packages:
  - apt-transport-https
  - ca-certificates
  - curl
  - gnupg

write_files:
  - path: /etc/docker/daemon.json
    content: |
      {
        "log-driver": "json-file",
        "log-opts": {"max-size": "10m", "max-file": "3"},
        "storage-driver": "overlay2"
      }

runcmd:
  - curl -fsSL https://get.docker.com | sh
  - systemctl enable --now docker
  - usermod -aG docker ubuntu
```

### Join a Kubernetes Cluster

```yaml
#cloud-config
package_update: true

bootcmd:
  - modprobe br_netfilter
  - sysctl -w net.bridge.bridge-nf-call-iptables=1
  - sysctl -w net.ipv4.ip_forward=1

write_files:
  - path: /etc/sysctl.d/k8s.conf
    content: |
      net.bridge.bridge-nf-call-iptables = 1
      net.ipv4.ip_forward = 1

runcmd:
  - curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  - echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" > /etc/apt/sources.list.d/kubernetes.list
  - apt-get update && apt-get install -y kubelet kubeadm kubectl
  - kubeadm join control-plane:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### Harden SSH on First Boot

```yaml
#cloud-config
ssh_pwauth: false
ssh_deletekeys: true
ssh_genkeytypes: [ed25519]

write_files:
  - path: /etc/ssh/sshd_config.d/hardening.conf
    content: |
      PermitRootLogin no
      PasswordAuthentication no
      MaxAuthTries 3
      ClientAliveInterval 300
      ClientAliveCountMax 2
      X11Forwarding no
      AllowTcpForwarding no

users:
  - name: admin
    groups: [sudo]
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: true
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... admin@workstation

runcmd:
  - systemctl restart sshd
```

### Every-Boot Script (Alternative to bootcmd)

```bash
# Place scripts in /var/lib/cloud/scripts/per-boot/
cat > /var/lib/cloud/scripts/per-boot/mount-ephemeral.sh <<'EOF'
#!/bin/bash
mount /dev/nvme1n1 /data 2>/dev/null || true
EOF
chmod +x /var/lib/cloud/scripts/per-boot/mount-ephemeral.sh
```

### Web Server Setup

```yaml
#cloud-config
packages:
  - nginx
  - certbot
  - python3-certbot-nginx

write_files:
  - path: /var/www/html/index.html
    content: |
      <h1>Welcome to my server!</h1>
    owner: www-data:www-data
    permissions: '0644'

runcmd:
  - systemctl enable nginx
  - systemctl start nginx
  - ufw allow 'Nginx Full'
  - ufw --force enable
```

### Docker Host Setup (GPG Key Method)

```yaml
#cloud-config
packages:
  - apt-transport-https
  - ca-certificates
  - curl
  - gnupg
  - lsb-release

runcmd:
  - curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
  - echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
  - apt update
  - apt install -y docker-ce docker-ce-cli containerd.io
  - systemctl enable docker
  - usermod -aG docker ubuntu
```

### Docker Run Container on Boot

```yaml
#cloud-config
packages:
  - docker.io

runcmd:
  - systemctl start docker
  - systemctl enable docker
  - usermod -aG docker ubuntu
  - docker run -d --name nginx -p 80:80 nginx:latest
```

### Development Environment

```yaml
#cloud-config
packages:
  - git
  - curl
  - wget
  - vim
  - htop
  - build-essential
  - python3-pip
  - nodejs
  - npm

users:
  - name: developer
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2E... dev@example.com

runcmd:
  - pip3 install --upgrade pip
  - npm install -g yarn
  - curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
```

## Test cloud-init Locally

```bash
# Full local test sequence
cloud-init clean
cloud-init init --local
cloud-init init
cloud-init modules --mode=config
cloud-init modules --mode=final
```

### View Logs in Real-Time

```bash
tail -f /var/log/cloud-init.log
tail -f /var/log/cloud-init-output.log
```

## Tips and Best Practices

1. **Always validate your YAML** — use `cloud-init schema` to validate syntax before deploying
2. **Test in a safe environment** — test configurations in LXD/VMs before production use
3. **Use proper indentation** — YAML is sensitive to indentation (use spaces, not tabs)
4. **Quote special characters** — quote strings with special YAML characters (`:`, `{`, `}`, `[`, `]`)
5. **Use absolute paths** — always use full paths for files and commands
6. **Check cloud-init version** — different versions support different features
7. **Use runcmd for complex operations** — for multi-step operations, use runcmd over bootcmd
8. **Handle secrets carefully** — avoid putting sensitive data directly in user-data (use instance roles or vault)
9. **Randomize instance-id** — if cloud-init detects the same instance-id as a previous run, the entire configuration is skipped
10. **Mind module execution order** — `write_files` runs before `users_groups`; use `defer: true` when writing to user home directories that don't exist yet
11. **Don't write to /tmp** — use `/run/somedir` instead, as early boot races with `systemd-tmpfiles-clean`
12. **Use `preserve_hostname: false`** — if you want cloud-init to manage the hostname on every boot, otherwise the OS may overwrite it
13. **Disable APT recommends** — reduce image bloat with `apt.conf: | APT::Install-Recommends "0";`
14. **Use output directive** — redirect cloud-init output for easier debugging: `output: { all: "| tee -a /var/log/cloud-init-output.log" }`

## Password Configuration

### Set Passwords

```yaml
#cloud-config
# Set password for default user
password: mypassword
chpasswd:
  expire: false

# Or set passwords for multiple users
chpasswd:
  expire: false
  users:
    - name: root
      password: $6$rounds=4096$saltsalt$hash...
      type: HASH
    - name: admin
      password: plaintext-password
      type: text
```

### Enable Password SSH (for initial access)

```yaml
#cloud-config
ssh_pwauth: true
password: temporary-password
chpasswd:
  expire: true  # Force password change on first login
```

## Advanced Re-running & Debugging

### Re-run a Single Module with Custom User-Data

```bash
# Edit the cached user-data, then re-run the specific module
sudo cloud-init single --name cc_ntp --frequency always \
  -f /var/lib/cloud/instances/<instance-id>/user-data.txt

# Re-run an entire stage with frequency override
sudo cloud-init modules --mode config --frequency always \
  -f /var/lib/cloud/instances/<instance-id>/user-data.txt
```

### Inspect Instance State

```bash
# List all instance runs (useful when instance-id changed)
ls -l /var/lib/cloud/instances/

# Show what cloud-init knows about this instance
cloud-init query --all

# List available query keys
cloud-init query --list-keys

# Check which datasource was used
cloud-init query ds

# View the actual user-data cloud-init received
cat /var/lib/cloud/instance/user-data.txt

# View rendered cloud-config (after includes/merging)
cat /var/lib/cloud/instance/cloud-config.txt

# Check the boot-finished marker
cat /var/lib/cloud/instance/boot-finished
```

### Verify SSH Configuration After Provisioning

```bash
# Test effective SSH config (very useful after hardening)
sshd -T | grep -iE 'permitrootlogin|passwordauthentication|pubkeyauthentication|usepam'
```

### Inspect Config Drive (VPS Providers)

```bash
# Many VPS providers attach a CIDATA ISO — inspect it
blkid /dev/sr0
mkdir -p /mnt/cidata
mount -t iso9660 -o ro /dev/sr0 /mnt/cidata
ls -la /mnt/cidata
cat /mnt/cidata/user-data
cat /mnt/cidata/meta-data
cat /mnt/cidata/network-config
umount /mnt/cidata
```

## Terraform Integration

### Using cloudinit_config Provider

```hcl
# main.tf
data "cloudinit_config" "server" {
  gzip          = false
  base64_encode = false

  part {
    content_type = "text/cloud-config"
    content = templatefile("cloud-config.yaml.tpl", {
      ssh_key  = var.ssh_public_key
      hostname = var.hostname
    })
  }
}

resource "hcloud_server" "main" {
  name        = var.hostname
  image       = "ubuntu-24.04"
  server_type = "cx21"
  user_data   = data.cloudinit_config.server.rendered
}
```

### AWS EC2 with User-Data

```bash
aws ec2 run-instances \
    --image-id ami-0c7217cdde317cfec \
    --instance-type t3.micro \
    --key-name my-key \
    --user-data file://cloud-config.yaml \
    --security-group-ids sg-xxxxxxxxx \
    --subnet-id subnet-xxxxxxxxx
```

### Hetzner Cloud

```bash
hcloud server create \
    --name my-server \
    --type cx21 \
    --image ubuntu-24.04 \
    --user-data-from-file cloud-config.yaml \
    --ssh-key my-key
```

### DigitalOcean

```bash
doctl compute droplet create my-server \
    --image ubuntu-24-04-x64 \
    --size s-1vcpu-1gb \
    --region nyc3 \
    --ssh-keys my-key-id \
    --user-data-file cloud-config.yaml
```

## Instance Identity & Re-Provisioning

### What Triggers a "New Instance"

| Trigger | Result |
|---------|--------|
| Different `instance-id` in datasource | Full re-run of all once-per-instance modules |
| `cloud-init clean` + reboot | Treats next boot as a new instance |
| Same `instance-id` after reboot | Only `always`-frequency modules run |
| Change hostname only | Does NOT trigger full re-run |

### Force New Instance Without Clean

```bash
# Change the instance-id in NoCloud seed (triggers full re-provisioning)
echo "instance-id: $(uuidgen)" > /var/lib/cloud/seed/nocloud/meta-data
reboot
```

## Incus / LXD Integration

### Launch with cloud-init User-Data

```bash
# Inline cloud-config
lxc launch ubuntu:24.04 my-container \
  --config=user.user-data="#cloud-config
packages: [nginx]
runcmd:
  - systemctl enable --now nginx"

# From file
lxc launch ubuntu:24.04 my-container \
  --config=user.user-data="$(cat cloud-config.yaml)"

# Watch provisioning
lxc exec my-container -- tail -f /var/log/cloud-init-output.log

# Verify completion
lxc exec my-container -- cloud-init status --wait
```

### Incus Profile with cloud-init

```yaml
name: my-profile
config:
  cloud-init.user-data: |
    #cloud-config
    package_update: true
    packages: [nginx, curl]
    runcmd:
      - systemctl enable --now nginx
  cloud-init.network-config: |
    version: 2
    ethernets:
      eth0:
        dhcp4: true
```

## KVM Provisioning with virt-install

### Launch VM with cloud-init Files

```bash
virt-install --name my-vm \
  --memory 4096 --os-variant ubuntu22.04 \
  --disk=size=10,backing_store="$(pwd)/image/ubuntu-22.04-minimal-cloudimg-amd64.img" \
  --cloud-init user-data="$(pwd)/user-data",meta-data="$(pwd)/meta-data",network-config="$(pwd)/network-config" \
  --network network=default
```

### Generate Instance ID for meta-data

```bash
# Generate a random instance-id (ensures cloud-init treats this as a new instance)
echo "instance-id: $(uuidgen)" > meta-data
```

### Wait for cloud-init to Complete (Scripting)

```bash
# Block until cloud-init finishes — useful in automation pipelines
cloud-init status --wait
echo $?  # 0 = success, 1 = error

# Get DHCP lease to find VM IP
virsh net-dhcp-leases default
```

## Proxmox Cloud-Init Templates

### Create a Template from Cloud Image

```bash
# Download cloud image
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# Create VM
qm create 9000

# Import disk
qm importdisk 9000 noble-server-cloudimg-amd64.img local-lvm

# Configure VM with cloud-init
qm set 9000 \
  --name ubuntu-template \
  --scsihw virtio-scsi-pci \
  --scsi0 local-lvm:vm-9000-disk-0 \
  --ide2 local-lvm:cloudinit \
  --boot c --bootdisk scsi0 \
  --serial0 socket --vga serial0 \
  --net0 virtio,bridge=vmbr0 \
  --ipconfig0 ip=dhcp \
  --sshkey ~/.ssh/authorized_keys \
  --agent 1

# Resize disk
qm resize 9000 scsi0 +50G

# Convert to template
qm template 9000
```

### Clone and Launch from Template

```bash
# Clone (linked clone)
qm clone 9000 100 --name my-server --full 0

# Start
qm start 100

# Get IP via guest agent
pvesh get nodes/$(hostname)/qemu/100/agent/network-get-interfaces --output-format=json | \
  jq -r '.result[] | select(.name=="eth0") | ."ip-addresses"[] | select(."ip-address-type"=="ipv4") | ."ip-address"'
```

### Custom User-Data via Snippets

```bash
# Write custom cloud-init user-data to snippets directory
cat > /var/lib/vz/snippets/vm-100-user-data.yaml <<'EOF'
#cloud-config
fqdn: my-server
ssh_pwauth: false
users:
  - name: admin
    groups: [sudo, docker]
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... admin@workstation
runcmd:
  - apt-get update
  - apt-get install -y qemu-guest-agent
  - systemctl start qemu-guest-agent
EOF

# Attach custom user-data to VM
qm set 100 --cicustom "user=local:snippets/vm-100-user-data.yaml"
```

### APT Proxy Configuration

```yaml
#cloud-config
apt:
  preserve_sources_list: true
  http_proxy: http://proxy.example.com:3128
  https_proxy: http://proxy.example.com:3128
  conf: |
    APT {
      Get {
        Assume-Yes "true";
        Fix-Broken "true";
      }
    }
```

## Networking Tips

### Disable cloud-init Network Management

```bash
# Prevent cloud-init from overwriting your network config
sudo tee /etc/cloud/cloud.cfg.d/99-disable-network.cfg <<'EOF'
network: {config: disabled}
EOF
```

### Pin Datasource to Avoid Probing Delays

```bash
# On non-cloud systems, restrict to NoCloud only
sudo tee /etc/cloud/cloud.cfg.d/90-datasource.cfg <<'EOF'
datasource_list: [NoCloud, None]
EOF
```

### Explicit NoCloud Seed Path

```bash
# Point NoCloud to a specific seed directory (removes ambiguity)
sudo tee /etc/cloud/cloud.cfg.d/91-nocloud-seed.cfg <<'EOF'
datasource:
  NoCloud:
    seedfrom: file:///var/lib/cloud/seed/nocloud/
EOF
```

## Security Considerations

When sharing cloud-init logs or configs, be aware these can contain secrets:

| File | Risk |
|------|------|
| `/var/lib/cloud/instance/user-data.txt` | May contain passwords, tokens, SSH keys |
| `/var/lib/cloud/instance/vendor-data.txt` | May contain provider tokens |
| `/var/log/cloud-init.log` | May echo rendered config with credentials |
| `/var/log/cloud-init-output.log` | May contain command output with secrets |
| `cloud-init query --all` | Includes sensitive instance data |

### Redaction Checklist Before Sharing

- Remove lines containing `password`, `token`, `secret`, `key`, `Authorization`, or `Bearer`
- Replace SSH public keys if you don't want them tied to a host identity
- Scan `cloud-init.log` for rendered config content, not just obvious credentials
- Use `cloud-init collect-logs` which creates a tarball — review before sharing


## Troubleshooting Guide

### System Service Status

```bash
# Check if cloud-init services are running
systemctl status cloud-init.service
systemctl status cloud-init-local.service
systemctl status cloud-config.service
systemctl status cloud-final.service

# Check service logs
journalctl -u cloud-init.service
journalctl -u cloud-config.service

# Boot logs
dmesg | grep -i cloud
```

### Log Analysis Commands

```bash
# Search for errors in logs
grep -i error /var/log/cloud-init.log
grep -i fail /var/log/cloud-init.log
grep -i warn /var/log/cloud-init.log

# Check specific stage execution
grep "modules-init" /var/log/cloud-init.log
grep "modules-config" /var/log/cloud-init.log
grep "modules-final" /var/log/cloud-init.log

# Filter by specific modules
grep "cc_users_groups" /var/log/cloud-init.log
grep "cc_ssh" /var/log/cloud-init.log
grep "cc_write_files" /var/log/cloud-init.log
grep "cc_runcmd" /var/log/cloud-init.log
grep "cc_package" /var/log/cloud-init.log
```

### Show Current Configuration

```bash
# Show merged cloud-init configuration
cloud-init query --all

# Show specific configuration keys
cloud-init query users
cloud-init query packages
cloud-init query write_files

# Show instance metadata
cloud-init query ds
```

### Test Specific Modules

```bash
# Run single-use modules again
cloud-init single --name cc_users_groups
cloud-init single --name cc_ssh
cloud-init single --name cc_write_files

# List available modules
cloud-init single --list-modules
```

### Issue: Cloud-Init Not Running

```bash
# Diagnosis
systemctl is-enabled cloud-init.service
systemctl is-enabled cloud-config.service
systemctl status cloud-init.service --no-pager

# Fix: Enable services
systemctl enable cloud-init.service
systemctl enable cloud-config.service
systemctl enable cloud-final.service
systemctl start cloud-init.service

# Fix: Remove disable file if exists
ls -la /etc/cloud/cloud-init.disabled
rm /etc/cloud/cloud-init.disabled
```

### Issue: YAML Syntax Errors

```yaml
# BAD: Mixed tabs and spaces
users:
	- name: ubuntu  # Tab used here
  - name: admin    # Spaces used here

# GOOD: Consistent spaces
users:
  - name: ubuntu
  - name: admin

# BAD: Incorrect indentation
write_files:
- path: /tmp/file
content: |
  Hello World

# GOOD: Proper indentation
write_files:
  - path: /tmp/file
    content: |
      Hello World
```

### Issue: User Creation Failures

```bash
# Diagnosis
grep "cc_users_groups" /var/log/cloud-init.log
id username
cat /etc/passwd | grep username
cat /home/username/.ssh/authorized_keys
sudo -l -U username
```

```yaml
# Ensure proper user configuration
users:
  - name: ubuntu
    sudo: ['ALL=(ALL) NOPASSWD:ALL']  # Use array format
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2E...  # Full key on one line
    lock_passwd: false  # Allow password if needed
```

### Issue: SSH Access Problems

```bash
# Diagnosis
systemctl status ssh
grep -E "(PasswordAuthentication|PubkeyAuthentication)" /etc/ssh/sshd_config
ls -la /home/username/.ssh/
cat /home/username/.ssh/authorized_keys
journalctl -u ssh.service
tail -f /var/log/auth.log

# Fix permissions
chmod 700 /home/username/.ssh
chmod 600 /home/username/.ssh/authorized_keys
chown -R username:username /home/username/.ssh
systemctl restart ssh
```

### Issue: Package Installation Failures

```bash
# Diagnosis
grep "cc_package" /var/log/cloud-init.log
cat /var/log/apt/history.log
cat /var/log/dpkg.log

# Test manually
apt update
apt install -y package-name
```

### Issue: File Write Problems

```bash
# Diagnosis
grep "cc_write_files" /var/log/cloud-init.log
ls -la /path/to/file
ls -ld /path/to/  # Check parent directory
```

```yaml
# Ensure permissions are quoted
write_files:
  - path: /opt/myapp/config.conf
    content: |
      setting=value
    owner: root:root
    permissions: '0644'  # Always quote permissions
```

### Issue: Command Execution Failures

```bash
# Diagnosis
grep "cc_runcmd" /var/log/cloud-init.log
grep -A 10 -B 5 "runcmd" /var/log/cloud-init-output.log
grep "Failed" /var/log/cloud-init.log
```

```yaml
runcmd:
  # Use full paths
  - /usr/bin/systemctl enable docker
  
  # Handle errors gracefully
  - systemctl enable nginx || echo "nginx enable failed"
  
  # Use proper quoting for complex commands
  - 'curl -fsSL https://get.docker.com | sh'
  
  # Use array format for commands with arguments
  - [wget, "-O", "/tmp/file.tar.gz", "https://example.com/file.tar.gz"]
```

### Issue: Network Configuration Problems

```bash
# Diagnosis
ip addr show
ip route show
cat /etc/resolv.conf
ls -la /etc/netplan/
netplan get

# Fix
netplan --debug apply
systemctl restart systemd-networkd
```

### Timing and Boot Analysis

```bash
# Check cloud-init timing
cloud-init analyze show
cloud-init analyze blame

# System boot timing
systemd-analyze
systemd-analyze blame
systemd-analyze critical-chain cloud-init.service
```

### Module Dependency Issues

```bash
# Check module execution order
grep -E "(modules-init|modules-config|modules-final)" /var/log/cloud-init.log

# Check disabled/skipped modules
grep -i "skip" /var/log/cloud-init.log
```

### Complete Reset (Recovery)

```bash
# Full cloud-init reset
cloud-init clean --seed --logs
rm -rf /var/lib/cloud/*
rm -rf /run/cloud-init
cloud-init init --local
```

### Emergency Access Recovery

```bash
# Add temporary user (run as root from console)
useradd -m -s /bin/bash emergency
echo 'emergency:password' | chpasswd
usermod -aG sudo emergency

# Enable password authentication temporarily
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
systemctl restart ssh
```

### Reduce Boot Time

```yaml
#cloud-config
# Disable unnecessary operations
package_update: false
package_upgrade: false

# Use faster mirror
apt:
  primary:
    - arches: [default]
      uri: http://us.archive.ubuntu.com/ubuntu/
```

### Cloud-Init Status Debug Script

```bash
#!/bin/bash
echo "=== Cloud-Init Status ==="
cloud-init status --long

echo -e "\n=== Service Status ==="
systemctl status cloud-init.service --no-pager -l
systemctl status cloud-config.service --no-pager -l

echo -e "\n=== Recent Errors ==="
grep -i error /var/log/cloud-init.log | tail -10

echo -e "\n=== Module Timing ==="
cloud-init analyze blame | head -10
```

### Network Debug Script

```bash
#!/bin/bash
echo "=== Network Configuration ==="
ip addr show
echo -e "\n=== Routing ==="
ip route show
echo -e "\n=== DNS ==="
cat /etc/resolv.conf
echo -e "\n=== Connectivity ==="
ping -c 3 8.8.8.8
```
