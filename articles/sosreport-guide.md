# sosreport Guide

`sos` (formerly `sosreport`) is a diagnostic data collection tool for Red Hat Enterprise Linux. It gathers system configuration, logs, hardware info, and the state of running services into a compressed archive that can be attached to Red Hat support cases. The tool uses a plugin architecture — each plugin collects data from a specific subsystem.

## Install

```bash
# RHEL 8+ (pre-installed)
rpm -q sos

# RHEL 7 / CentOS 7
sudo yum install sos

# RHEL 8+ / Fedora
sudo dnf install sos

# Verify version
sos report --version
```

## Generate a Report

```bash
# Basic report (interactive — prompts for case ID)
sudo sos report

# Non-interactive (skip prompts)
sudo sos report --batch

# With a Red Hat support case number
sudo sos report --case-id=03456789

# With a custom temporary directory (useful if /tmp is small)
sudo sos report --tmp-dir=/var/tmp

# Specify output directory for the archive
sudo sos report --tmp-dir=/opt/sosreports/

# Set a custom archive name
sudo sos report --label=webserver-issue

# Limit log size collected per file (default: 25 MB)
sudo sos report --log-size=10

# Compress with a specific algorithm
sudo sos report --compression-type=xz
sudo sos report --compression-type=gzip

# Generate without compression (plain directory, not tarball)
sudo sos report --build
```

## Plugin Management

```bash
# List all available plugins and their status
sudo sos report --list-plugins

# List only enabled plugins
sudo sos report --list-plugins | grep -A1 "enabled"

# Run only specific plugins
sudo sos report --only-plugins=networking,firewalld,kernel

# Enable additional plugins (that are disabled by default)
sudo sos report --enable-plugins=docker,podman

# Skip specific plugins
sudo sos report --skip-plugins=yum,dnf,rpm

# Pass options to a specific plugin
sudo sos report -k networking.traceroute=off
sudo sos report -k logs.all_logs=on

# Verify what a plugin would collect (dry run)
sudo sos report --dry-run --only-plugins=networking
```

## Common Plugins

| Plugin | Collects |
|--------|----------|
| `kernel` | Kernel parameters, modules, sysctl, dmesg |
| `networking` | Interfaces, routes, iptables/nftables, ss, bonding |
| `firewalld` | Firewall rules, zones, services |
| `filesys` | Mount points, fstab, df, lsblk |
| `block` | Block devices, schedulers, queue parameters |
| `lvm2` | PVs, VGs, LVs, LVM configuration |
| `multipath` | DM-Multipath configuration and status |
| `devicemapper` | Device mapper tables and status |
| `systemd` | Unit files, journal, timers, targets |
| `logs` | System logs (/var/log/messages, journal) |
| `yum` / `dnf` | Package history, repo config, installed RPMs |
| `rpm` | Installed packages, verification |
| `subscription_manager` | RHSM registration, subscriptions, repos |
| `selinux` | SELinux status, booleans, policy info |
| `pam` | PAM configuration files |
| `sssd` | SSSD configuration and logs |
| `cron` | Crontabs, anacron, systemd timers |
| `docker` | Container info, images, volumes (if Docker installed) |
| `podman` | Pods, containers, images (if Podman installed) |
| `kubernetes` | Pods, nodes, services (if kubectl available) |
| `openshift` | OCP resources and logs |
| `pacemaker` | Cluster status, resources, constraints |
| `corosync` | Cluster communication layer config |
| `virsh` | KVM domains, networks, storage pools |
| `ovirt` | oVirt/RHV host agent and VDSM logs |
| `insights` | Insights client configuration and status |
| `satellite` | Satellite/Foreman services and logs |
| `apache` | httpd configuration and logs |
| `nginx` | NGINX configuration and logs |
| `mysql` | MySQL/MariaDB configuration (no data) |
| `postgresql` | PostgreSQL configuration and logs |

## Targeted Reports

```bash
# Networking issues only
sudo sos report --batch --only-plugins=networking,firewalld,openvswitch,ethtool

# Storage issues only
sudo sos report --batch --only-plugins=filesys,block,lvm2,multipath,scsi,iscsi

# Cluster/HA issues
sudo sos report --batch --only-plugins=pacemaker,corosync,dlm

# Virtualization issues (KVM)
sudo sos report --batch --only-plugins=virsh,libvirt,qemu

# Container issues
sudo sos report --batch --only-plugins=podman,containers_common,crio

# Kernel/boot issues
sudo sos report --batch --only-plugins=kernel,grub2,boot,systemd

# Security/authentication issues
sudo sos report --batch --only-plugins=selinux,sssd,pam,ldap,kerberos

# Performance issues
sudo sos report --batch --only-plugins=kernel,processor,memory,block,filesys
```

## Collect from Multiple Nodes

`sos collect` runs `sos report` on multiple cluster nodes concurrently and packages all reports into a single archive.

```bash
# Collect from a list of nodes (SSH key auth required)
sudo sos collect --nodes=node1,node2,node3

# Collect from nodes with password auth
sudo sos collect --nodes=node1,node2,node3 --password

# Auto-detect cluster type and nodes (run from a cluster node)
sudo sos collect

# Specify cluster type manually
sudo sos collect --cluster-type=pacemaker
sudo sos collect --cluster-type=kubernetes
sudo sos collect --cluster-type=ovirt

# Collect from a remote master node (run from workstation)
sudo sos collect --master=cluster-node1 --nodes=node1,node2,node3

# Collect with specific plugins on all nodes
sudo sos collect --nodes=node1,node2 --only-plugins=pacemaker,corosync

# Collect with a case number
sudo sos collect --nodes=node1,node2 --case-id=03456789 --batch

# Specify SSH user and key
sudo sos collect --nodes=node1,node2 --ssh-user=admin --ssh-key=/root/.ssh/id_rsa

# Set a timeout per node (seconds)
sudo sos collect --nodes=node1,node2 --timeout=600
```

## Obfuscate Sensitive Data

`sos clean` (RHEL 8.4+) redacts hostnames, IP/MAC addresses, and custom keywords from an existing report.

```bash
# Obfuscate an existing sos report archive
sudo sos clean /var/tmp/sosreport-server1-2024-01-15.tar.xz

# Obfuscate during report generation
sudo sos report --clean

# Add custom keywords to obfuscate
sudo sos clean /var/tmp/sosreport-server1.tar.xz \
  --keywords=mycompany,internal-app,secretproject

# Add specific domains to obfuscate
sudo sos clean /var/tmp/sosreport-server1.tar.xz \
  --domains=example.com,internal.corp

# Obfuscate usernames
sudo sos clean /var/tmp/sosreport-server1.tar.xz \
  --usernames=admin,deploy,dbuser

# Obfuscate a sos collect archive
sudo sos clean /var/tmp/sos-collector-2024-01-15.tar.xz

# Output produces:
#   - A new *-obfuscated.tar.xz archive
#   - A mapping file showing original -> obfuscated values
```

## Upload to Red Hat Support

```bash
# Upload directly to a Red Hat support case (RHEL 8+)
# Requires system to be registered with subscription-manager
sudo sos report --upload --case-id=03456789

# Upload an existing archive
sudo sos upload /var/tmp/sosreport-server1-2024-01-15.tar.xz --case-id=03456789

# Manual upload via SFTP (alternative)
sftp sftp://dropbox.redhat.com/incoming/
# put /var/tmp/sosreport-server1-2024-01-15.tar.xz

# Manual upload via curl to the Red Hat Customer Portal
# (attach via support case web UI if automated upload fails)
```

## Generate from Rescue Environment

```bash
# When system won't boot — boot from rescue/live media, then:

# Mount the root filesystem
mount /dev/sda2 /mnt/sysimage
mount --bind /proc /mnt/sysimage/proc
mount --bind /sys /mnt/sysimage/sys
mount --bind /dev /mnt/sysimage/dev

# Chroot and generate
chroot /mnt/sysimage
sos report --batch

# Or run without chroot (limited data)
sos report --sysroot=/mnt/sysimage --batch
```

## Configuration File

```bash
# Global configuration: /etc/sos/sos.conf (RHEL 8+)
# Legacy location: /etc/sos.conf (RHEL 7)
cat /etc/sos/sos.conf
```

```ini
# /etc/sos/sos.conf
[global]
# Default compression type
compression-type = xz

# Default temporary directory
tmp-dir = /var/tmp

[report]
# Always skip these plugins
skip-plugins = rpm,dnf

# Always enable these plugins
enable-plugins = podman

# Default log size limit per file (MB)
log-size = 25

# Non-interactive by default
batch = true

[collect]
# Default SSH user for sos collect
ssh-user = root

# Default timeout per node
timeout = 600
```

## Inspect the Archive

```bash
# List contents of a sosreport archive
tar -tf /var/tmp/sosreport-server1-2024-01-15.tar.xz | head -50

# Extract the archive
mkdir /tmp/sosreport && tar -xf /var/tmp/sosreport-server1-2024-01-15.tar.xz -C /tmp/sosreport

# Key directories inside the archive
ls /tmp/sosreport/sosreport-server1-*/
# sos_commands/    — output of commands run by each plugin
# etc/             — configuration files
# var/log/         — log files
# proc/            — /proc entries
# sos_logs/        — sos tool's own logs
# sos_reports/     — plugin execution summary

# Check which plugins ran and their status
cat /tmp/sosreport/sosreport-server1-*/sos_reports/manifest.json | python3 -m json.tool

# Quick look at network config collected
ls /tmp/sosreport/sosreport-server1-*/sos_commands/networking/

# Check collected kernel parameters
cat /tmp/sosreport/sosreport-server1-*/sos_commands/kernel/sysctl_-a

# View collected firewall rules
cat /tmp/sosreport/sosreport-server1-*/sos_commands/firewalld/firewall-cmd_--list-all-zones
```

## Useful Options Reference

| Option | Description |
|--------|-------------|
| `--batch` | Non-interactive, skip all prompts |
| `--case-id=ID` | Associate with a support case |
| `--label=TEXT` | Add a custom label to the archive name |
| `--only-plugins=LIST` | Run only the specified plugins |
| `--skip-plugins=LIST` | Skip the specified plugins |
| `--enable-plugins=LIST` | Enable plugins that are disabled by default |
| `-k PLUGIN.OPT=VAL` | Pass an option to a plugin |
| `--list-plugins` | List all plugins and their status |
| `--dry-run` | Show what would be collected without running |
| `--tmp-dir=PATH` | Use a custom temp directory (also controls output location) |
| `--log-size=MB` | Max log file size to collect (default: 25) |
| `--compression-type=TYPE` | xz, gzip, bzip2, or auto |
| `--build` | Generate as uncompressed directory (no tarball) |
| `--clean` | Obfuscate sensitive data after collection |
| `--upload` | Upload the archive to Red Hat (requires registration) |
| `--all-logs` | Collect full logs (no size truncation) |
| `--since=DATE` | Collect journal/logs since DATE |
| `--verify` | Run rpm -V on installed packages |
| `--timeout=SECONDS` | Plugin timeout (default: 300) |
| `--cmd-timeout=SECONDS` | Per-command timeout (default: 300) |

## Analyze with xsos

[xsos](https://github.com/ryran/xsos) is a standalone tool that summarizes sosreport data into a readable overview — BIOS, OS, CPU, memory, networking, storage, and more.

```bash
# Install xsos
curl -Lo /usr/local/bin/xsos bit.ly/xsos-direct
chmod +x /usr/local/bin/xsos

# Extract a sosreport archive
tar -xvJf /var/tmp/sosreport-server1-2024-01-15.tar.xz -C /tmp/

# Show full system summary from extracted sosreport
xsos -a /tmp/sosreport-server1-2024-01-15/

# Show only specific sections
xsos --bios /tmp/sosreport-server1-2024-01-15/
xsos --os /tmp/sosreport-server1-2024-01-15/
xsos --cpu /tmp/sosreport-server1-2024-01-15/
xsos --mem /tmp/sosreport-server1-2024-01-15/
xsos --net /tmp/sosreport-server1-2024-01-15/
xsos --disk /tmp/sosreport-server1-2024-01-15/

# Run xsos against the live system (no sosreport needed)
xsos -a
```

## Useful One-Liners

```bash
# Quick sosreport for a support case (non-interactive)
sudo sos report --batch --case-id=03456789

# Minimal report — just networking and logs
sudo sos report --batch --only-plugins=networking,logs

# Full logs, no truncation
sudo sos report --batch --all-logs

# Collect journal logs from the last 3 days only
sudo sos report --batch --since="3 days ago"

# Generate and obfuscate in one step
sudo sos report --batch --clean --case-id=03456789

# Check archive size before uploading
ls -lh /var/tmp/sosreport-*.tar.xz

# Find old sosreports taking up space
find /var/tmp -name "sosreport-*" -mtime +30 -exec ls -lh {} \;

# Clean up old reports
find /var/tmp -name "sosreport-*" -mtime +30 -delete

# Collect from all pacemaker cluster nodes
sudo sos collect --cluster-type=pacemaker --batch --case-id=03456789

# Compare plugins between two RHEL versions
diff <(ssh rhel8 "sos report --list-plugins 2>/dev/null") \
     <(ssh rhel9 "sos report --list-plugins 2>/dev/null")
```
