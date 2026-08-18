# RHEL Post-Installation Steps

Essential configuration tasks to perform after a fresh Red Hat Enterprise Linux installation, covering RHEL 7 through 10. These steps establish a baseline for registration, networking, security, monitoring, and day-two operations.

## Register and Subscribe

```bash
# Register with Red Hat Customer Portal
sudo subscription-manager register --username=user@example.com --auto-attach

# Register with an activation key (preferred for automation)
sudo subscription-manager register --activationkey=mykey --org=myorg

# Register to Satellite
sudo subscription-manager register --org=myorg --activationkey=mykey \
  --serverurl=https://satellite.example.com/rhsm \
  --baseurl=https://satellite.example.com/pulp/repos

# Verify registration
sudo subscription-manager identity
sudo subscription-manager status

# List enabled repos
sudo subscription-manager repos --list-enabled

# Enable common repos (RHEL 8+)
sudo subscription-manager repos \
  --enable=rhel-8-for-x86_64-baseos-rpms \
  --enable=rhel-8-for-x86_64-appstream-rpms

# Enable EPEL (Extra Packages for Enterprise Linux)
# RHEL 7
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

# RHEL 8
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

# RHEL 9
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```

## Set Hostname

```bash
# Set a persistent hostname
sudo hostnamectl set-hostname server01.example.com

# Verify
hostnamectl

# Update /etc/hosts
echo "192.168.1.10  server01.example.com  server01" | sudo tee -a /etc/hosts
```

## Configure Networking

```bash
# List connections (RHEL 7+)
nmcli connection show

# Set a static IP
sudo nmcli connection modify ens192 \
  ipv4.addresses 192.168.1.10/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8,8.8.4.4" \
  ipv4.dns-search "example.com" \
  ipv4.method manual

# Bring the connection up with new settings
sudo nmcli connection up ens192

# Verify
ip addr show ens192
ip route show
cat /etc/resolv.conf

# Disable IPv6 (if not needed)
sudo nmcli connection modify ens192 ipv6.method disabled
sudo nmcli connection up ens192
```

### Legacy ifcfg Network Configuration (RHEL 6/7)

On RHEL 6 and early RHEL 7 systems, network interfaces are configured via ifcfg files.

```bash
# Edit the interface config file
vi /etc/sysconfig/network-scripts/ifcfg-eth0
```

```ini
DEVICE=eth0
HWADDR=00:0c:29:c5:d1:b0
TYPE=Ethernet
UUID=3309f090-b642-4997-9c22-c89fc2cc02aa
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=none
IPADDR=192.168.100.22
NETMASK=255.255.255.0
GATEWAY=192.168.100.1
DNS1=192.168.100.1
USERCTL=no
PEERDNS=no
IPV6INIT=no
```

```bash
# Configure default gateway (alternative location)
vi /etc/sysconfig/network
# GATEWAY=192.168.100.1

# Restart network (RHEL 6)
/etc/init.d/network restart

# Restart network (RHEL 7)
systemctl restart network
```

## Disable IPv6

```bash
# Method 1: sysctl (RHEL 7+, persistent)
cat <<EOF | sudo tee /etc/sysctl.d/ipv6.conf
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
EOF
sudo sysctl -p /etc/sysctl.d/ipv6.conf

# Rebuild initramfs to ensure IPv6 stays off at boot
sudo dracut -f

# Method 2: Disable the IPv6 kernel module (RHEL 6/7)
cat <<EOF | sudo tee /etc/modprobe.d/ipv6.conf
options ipv6 disable=1
EOF

# Disable ip6tables (RHEL 6)
sudo chkconfig ip6tables off

# Comment out IPv6 localhost in /etc/hosts
sudo sed -i 's/^[[:space:]]*::/#::/' /etc/hosts

# Method 3: nmcli (RHEL 7+)
sudo nmcli connection modify ens192 ipv6.method disabled
sudo nmcli connection up ens192
```

## Update the System

```bash
# RHEL 7
sudo yum update -y

# RHEL 8 / 9 / 10
sudo dnf update -y

# Check if reboot is needed (new kernel, glibc, systemd, etc.)
needs-restarting -r       # RHEL 7/8
dnf needs-restarting -r   # RHEL 9+

# List services that need restart (no reboot)
needs-restarting -s       # RHEL 7/8
dnf needs-restarting -s   # RHEL 9+
```

## Configure Time and NTP

```bash
# Set timezone
sudo timedatectl set-timezone Europe/Bucharest
timedatectl

# Enable NTP synchronization (chronyd is default on RHEL 7+)
sudo systemctl enable --now chronyd

# Verify NTP sources
chronyc sources -v
chronyc tracking

# Add custom NTP servers
sudo vi /etc/chrony.conf
# server ntp1.example.com iburst
# server ntp2.example.com iburst

sudo systemctl restart chronyd
```

## Configure SSH

```bash
# Harden SSH — edit /etc/ssh/sshd_config
sudo vi /etc/ssh/sshd_config
```

```bash
# Recommended settings
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
AllowUsers admin deploy
```

```bash
# Restart SSH
sudo systemctl restart sshd

# Copy SSH public key for passwordless access
ssh-copy-id admin@server01.example.com

# Verify SSH access before closing current session
ssh admin@server01.example.com
```

## Configure Firewall

```bash
# RHEL 7+ uses firewalld
sudo systemctl enable --now firewalld

# Check current zone and rules
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all

# Allow SSH (enabled by default in most zones)
sudo firewall-cmd --permanent --add-service=ssh

# Allow additional services
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https

# Allow a specific port
sudo firewall-cmd --permanent --add-port=8080/tcp

# Reload to apply permanent rules
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-all
```

### iptables (RHEL 6)

```bash
# Enable iptables on RHEL 6
chkconfig iptables on && service iptables start

# If minimal install is missing firewall tools
yum install system-config-firewall-base
lokkit --default=server
```

## Configure SELinux

```bash
# Check current status
getenforce
sestatus

# Set to enforcing (recommended for production)
sudo setenforce 1

# Make persistent across reboots
sudo vi /etc/selinux/config
# SELINUX=enforcing

# Common troubleshooting — check for denials
sudo ausearch -m AVC -ts recent
sudo sealert -a /var/log/audit/audit.log

# Install troubleshooting tools
sudo yum install setroubleshoot-server   # RHEL 7
sudo dnf install setroubleshoot-server   # RHEL 8+
```

## Configure Storage

```bash
# List block devices
lsblk

# Partition a new disk (if additional disks are attached)
sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary xfs 0% 100%

# Create filesystem
sudo mkfs.xfs /dev/sdb1

# Create mount point and mount
sudo mkdir -p /data
sudo mount /dev/sdb1 /data

# Add to /etc/fstab for persistence (use UUID)
UUID=$(blkid -s UUID -o value /dev/sdb1)
echo "UUID=$UUID  /data  xfs  defaults  0 0" | sudo tee -a /etc/fstab

# Verify fstab syntax
sudo mount -a
df -h /data
```

## Install Essential Packages

```bash
# RHEL 7
sudo yum install -y \
  vim wget curl net-tools bind-utils \
  yum-utils bash-completion \
  lsof strace tcpdump \
  sysstat iotop htop \
  tar unzip rsync tmux \
  policycoreutils-python \
  sos insights-client

# RHEL 8 / 9 / 10
sudo dnf install -y \
  vim wget curl net-tools bind-utils \
  dnf-utils bash-completion \
  lsof strace tcpdump \
  sysstat iotop htop \
  tar unzip rsync tmux \
  policycoreutils-python-utils \
  sos insights-client
```

## Configure kdump

```bash
# kdump captures kernel crash dumps for post-mortem analysis

# Verify kdump is installed
rpm -q kexec-tools

# Enable and start
sudo systemctl enable --now kdump

# Verify it's active
systemctl status kdump
kdumpctl status

# Check reserved memory (configured in GRUB)
cat /proc/cmdline | grep crashkernel

# Default crash dump location
cat /etc/kdump.conf | grep ^path
# path /var/crash

# Test kdump (WARNING: this will crash and reboot the system)
# echo c > /proc/sysrq-trigger
```

## Register with Red Hat Insights

```bash
# RHEL 8+ (insights-client pre-installed)
sudo insights-client --register

# RHEL 7 (install first)
sudo yum install insights-client
sudo insights-client --register

# Set a display name
sudo insights-client --display-name="server01.example.com"

# Verify
sudo insights-client --status
```

## Configure Automatic Updates (Optional)

```bash
# RHEL 7 (yum-cron)
sudo yum install yum-cron
sudo vi /etc/yum/yum-cron.conf
# apply_updates = yes (or no for download-only)
sudo systemctl enable --now yum-cron

# RHEL 8+ (dnf-automatic)
sudo dnf install dnf-automatic
sudo vi /etc/dnf/automatic.conf
# apply_updates = yes
sudo systemctl enable --now dnf-automatic.timer

# Verify timer
systemctl list-timers dnf-automatic.timer
```

## Configure Log Rotation and Persistence

```bash
# Enable persistent journal logging (survives reboots)
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald

# Or set in config
sudo vi /etc/systemd/journald.conf
# Storage=persistent

# Set journal size limit
# SystemMaxUse=500M

sudo systemctl restart systemd-journald

# Verify journal is persistent
journalctl --disk-usage
```

## Configure User Accounts

```bash
# Create an admin user
sudo useradd -m -s /bin/bash -G wheel admin

# Set password
sudo passwd admin

# Verify sudo access (wheel group has sudo on RHEL)
sudo -l -U admin

# Set password policies
sudo vi /etc/login.defs
# PASS_MAX_DAYS   90
# PASS_MIN_DAYS   7
# PASS_MIN_LEN    12
# PASS_WARN_AGE   14

# Lock root account for SSH (already done via sshd_config)
# Optionally lock root password entirely
# sudo passwd -l root
```

## Configure fail2ban (Optional)

```bash
# Install from EPEL
sudo yum install fail2ban       # RHEL 7
sudo dnf install fail2ban       # RHEL 8+

# Create local override
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo vi /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/secure
maxretry = 3
bantime = 3600
findtime = 600
```

```bash
# Enable and start
sudo systemctl enable --now fail2ban

# Check status
sudo fail2ban-client status sshd
```

## Configure SNMP or Monitoring Agent (Optional)

```bash
# Install net-snmp
sudo dnf install net-snmp net-snmp-utils

# Basic configuration
sudo vi /etc/snmp/snmpd.conf
# rocommunity public 192.168.1.0/24

sudo systemctl enable --now snmpd

# Or install Datadog / Zabbix / Prometheus node_exporter as needed
```

## Configure Audit Rules (Optional)

```bash
# auditd is installed and enabled by default

# Add custom rules for compliance
sudo vi /etc/audit/rules.d/custom.rules
```

```bash
# Watch /etc/passwd for changes
-w /etc/passwd -p wa -k user_changes

# Watch /etc/shadow for changes
-w /etc/shadow -p wa -k password_changes

# Watch /etc/sudoers for changes
-w /etc/sudoers -p wa -k sudoers_changes

# Log all commands run by root
-a always,exit -F arch=b64 -F euid=0 -S execve -k root_commands
```

```bash
# Reload audit rules
sudo augenrules --load
sudo systemctl restart auditd

# Verify rules loaded
sudo auditctl -l
```

## Lock Down RHEL Release (Optional)

```bash
# Pin to a specific minor release (prevent upgrades beyond it)
# RHEL 7
sudo subscription-manager release --set=7.9

# RHEL 8
sudo subscription-manager release --set=8.10

# RHEL 9
sudo subscription-manager release --set=9.4

# Verify
sudo subscription-manager release --show

# Remove lock (allow upgrades to latest minor)
sudo subscription-manager release --unset
```

## Configure MOTD Banner

```bash
# Set a login warning banner
cat <<'EOF' | sudo tee /etc/motd
UNAUTHORIZED ACCESS TO THIS DEVICE IS PROHIBITED
You must have explicit, authorized permission to access or configure this device.
Unauthorized attempts and actions to access or use this system may result in
civil and/or criminal penalties. All activities performed on this device are
logged and monitored.
EOF

# SSH pre-authentication banner (shown before login)
cat <<'EOF' | sudo tee /etc/issue.net
UNAUTHORIZED ACCESS TO THIS DEVICE IS PROHIBITED
EOF

# Enable the SSH banner
sudo sed -i 's/^#Banner.*/Banner \/etc\/issue.net/' /etc/ssh/sshd_config
sudo systemctl restart sshd

# Generate ASCII art banners: http://patorjk.com/software/taag
```

## Configure sysstat Collection Interval

```bash
# Default interval is 10 minutes — change to 1 minute for finer granularity
sudo vi /etc/cron.d/sysstat
```

```bash
# Collect every 1 minute instead of default 10
*/1 * * * * root /usr/lib64/sa/sa1 1 1
```

```bash
# RHEL 8+ uses a systemd timer instead of cron
sudo systemctl edit sysstat-collect.timer
# [Timer]
# OnCalendar=
# OnCalendar=*:*:00

sudo systemctl daemon-reload
sudo systemctl restart sysstat-collect.timer
```

## Install VMware Tools (Virtual Machines)

```bash
# Option 1: open-vm-tools (recommended — maintained in RHEL repos)
sudo dnf install open-vm-tools    # RHEL 8+
sudo yum install open-vm-tools    # RHEL 7
sudo systemctl enable --now vmtoolsd

# Option 2: VMware Tools from ISO (legacy method)
mount /dev/cdrom /mnt
cp /mnt/VMwareTools-*.tar.gz /tmp/
cd /tmp && tar -xzf VMwareTools-*.tar.gz
cd vmware-tools-distrib
./vmware-install.pl --default
umount /mnt

# Verify
vmware-toolbox-cmd stat speed
vmtoolsd --version
```

## Post-Install Verification

```bash
# Summary check after all steps
echo "=== Hostname ===" && hostnamectl | grep "Static hostname"
echo "=== Subscription ===" && sudo subscription-manager identity 2>&1 | head -3
echo "=== Insights ===" && sudo insights-client --status 2>&1 | tail -1
echo "=== SELinux ===" && getenforce
echo "=== Firewall ===" && sudo firewall-cmd --state
echo "=== NTP ===" && chronyc tracking | grep "Leap status"
echo "=== kdump ===" && systemctl is-active kdump
echo "=== Kernel ===" && uname -r
echo "=== Uptime ===" && uptime
echo "=== Disk ===" && df -h / /boot
echo "=== Memory ===" && free -h | head -2
```

## Quick Reference by RHEL Version

| Task | RHEL 7 | RHEL 8 | RHEL 9 / 10 |
|------|--------|--------|--------------|
| Package manager | `yum` | `dnf` (yum symlink) | `dnf` |
| Auto updates | `yum-cron` | `dnf-automatic` | `dnf-automatic` |
| Firewall | `firewalld` | `firewalld` | `firewalld` |
| NTP | `chronyd` | `chronyd` | `chronyd` |
| Default filesystem | XFS | XFS | XFS |
| Network config | `nmcli` / ifcfg files | `nmcli` / ifcfg files | `nmcli` / keyfiles (no ifcfg) |
| Python | 2.7 (default) | 3.6 / 3.8 / 3.9 | 3.9 / 3.11 / 3.12 |
| Init system | systemd | systemd | systemd |
| Insights client | Install manually | Pre-installed | Pre-installed |
| Default crypto | DEFAULT policy | DEFAULT / FUTURE | DEFAULT / FUTURE |
| Container runtime | docker | podman | podman |
| Module streams | N/A | `dnf module` | `dnf module` (limited in 9) |
