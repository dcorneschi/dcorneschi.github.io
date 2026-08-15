# Configure Samba on RHEL 6–10 and Ubuntu

This guide covers installing and configuring Samba as a file server on RHEL 6–10 and Ubuntu 22.04/24.04. It includes anonymous shares, authenticated shares, SELinux contexts, firewall rules, and client access.

## Overview

Samba provides SMB/CIFS file sharing between Linux and Windows systems. It implements the SMB protocol, allowing Linux machines to act as file servers for Windows clients or join Active Directory domains.

| Component | Purpose |
|-----------|---------|
| `smbd` | Handles file sharing and printing services (SMB/CIFS) |
| `nmbd` | Provides NetBIOS name resolution and browsing |
| `winbindd` | Maps Windows SIDs to Linux UIDs/GIDs (AD integration) |
| `smbclient` | CLI client for accessing SMB shares |
| `smb.conf` | Main configuration file (`/etc/samba/smb.conf`) |

### SMB Ports

| Protocol | Port | Purpose |
|----------|------|---------|
| NetBIOS Name Service | 137/UDP | Name resolution |
| NetBIOS Datagram | 138/UDP | Browsing |
| NetBIOS Session | 139/TCP | File sharing (legacy) |
| SMB over TCP | 445/TCP | Direct file sharing (modern) |

---

## Installation

### RHEL 6

```bash
yum install samba samba-client samba-common
```

### RHEL 7

```bash
yum install samba samba-client samba-common
```

### RHEL 8 / 9 / 10

```bash
dnf install samba samba-client samba-common
```

### Ubuntu 22.04 / 24.04

```bash
apt install samba smbclient cifs-utils
```

### macOS

```bash
brew install samba
```

---

## Service Management

### RHEL 6 (SysVinit)

```bash
service smb start
service nmb start
chkconfig smb on
chkconfig nmb on
```

### RHEL 7–10 and Ubuntu (systemd)

```bash
systemctl start smb nmb
systemctl enable smb nmb
```

On Ubuntu, the service names are different:

```bash
systemctl start smbd nmbd
systemctl enable smbd nmbd
```

### Check Service Status

```bash
# RHEL 7–10
systemctl status smb nmb

# Ubuntu
systemctl status smbd nmbd

# RHEL 6
service smb status
service nmb status
```

### Reload Configuration (Without Restart)

```bash
# RHEL 7–10
systemctl reload smb

# Ubuntu
systemctl reload smbd
```

---

## Firewall Configuration

### RHEL 6 (iptables)

```bash
iptables -I INPUT 4 -m state --state NEW -m udp -p udp --dport 137 -j ACCEPT
iptables -I INPUT 5 -m state --state NEW -m udp -p udp --dport 138 -j ACCEPT
iptables -I INPUT 6 -m state --state NEW -m tcp -p tcp --dport 139 -j ACCEPT
iptables -I INPUT 7 -m state --state NEW -m tcp -p tcp --dport 445 -j ACCEPT
service iptables save
```

### RHEL 7–10 (firewalld)

```bash
firewall-cmd --permanent --add-service=samba
firewall-cmd --reload
```

### Ubuntu (ufw)

```bash
ufw allow samba
```

Or explicitly:

```bash
ufw allow 139/tcp
ufw allow 445/tcp
ufw allow 137/udp
ufw allow 138/udp
```

---

## SELinux Configuration (RHEL Only)

SELinux contexts and booleans must be set for Samba to access shared directories.

### Set Context on Share Directory

```bash
semanage fcontext -a -t samba_share_t "/srv/samba/share(/.*)?"
restorecon -Rv /srv/samba/share
```

### Common SELinux Booleans

```bash
# Allow Samba to share home directories
setsebool -P samba_enable_home_dirs on

# Allow Samba read/write access to files with public_content_rw_t
setsebool -P smbd_anon_write on

# Allow Samba to export NFS shares
setsebool -P samba_export_all_rw on

# Allow Samba to use fusefs (FUSE filesystems)
setsebool -P samba_share_fusefs on
```

### View Current Booleans

```bash
getsebool -a | grep samba
```

### Verify File Context

```bash
ls -lZ /srv/samba/share
```

---

## Basic Configuration: Anonymous Share

### Create the Share Directory

```bash
mkdir -p /srv/samba/public
chmod 0777 /srv/samba/public

# SELinux (RHEL only)
semanage fcontext -a -t samba_share_t "/srv/samba/public(/.*)?"
restorecon -Rv /srv/samba/public
```

### Configure smb.conf

Back up the original configuration:

```bash
cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

Edit `/etc/samba/smb.conf`:

```ini
[global]
   workgroup = WORKGROUP
   server string = Samba Server
   netbios name = fileserver
   security = user
   map to guest = Bad User
   dns proxy = no

[public]
   path = /srv/samba/public
   browsable = yes
   writable = yes
   guest ok = yes
   guest only = yes
   read only = no
   create mask = 0666
   directory mask = 0777
```

### Validate Configuration

```bash
testparm
```

### Restart Services

```bash
# RHEL 7–10
systemctl restart smb nmb

# Ubuntu
systemctl restart smbd nmbd

# RHEL 6
service smb restart
service nmb restart
```

---

## Authenticated Share

### Create the Share Directory

```bash
mkdir -p /srv/samba/secure
chmod 0770 /srv/samba/secure
chgrp sambashare /srv/samba/secure

# SELinux (RHEL only)
semanage fcontext -a -t samba_share_t "/srv/samba/secure(/.*)?"
restorecon -Rv /srv/samba/secure
```

### Create Samba Users

Samba maintains its own password database. The Linux user must exist first:

```bash
# Create a system user (no login shell)
useradd -M -s /sbin/nologin sambauser

# Set the Samba password
smbpasswd -a sambauser
smbpasswd -e sambauser
```

### Configure smb.conf

Add to `/etc/samba/smb.conf`:

```ini
[secure]
   path = /srv/samba/secure
   browsable = yes
   writable = yes
   guest ok = no
   valid users = sambauser @sambashare
   create mask = 0660
   directory mask = 0770
```

### Managing Samba Users

```bash
# List Samba users
pdbedit -L

# List with verbose details
pdbedit -Lv

# Show specific user details
pdbedit -Lv username

# Change password
smbpasswd sambauser

# Change password on remote server
smbpasswd -r HOSTNAME -U sambauser

# Disable user
smbpasswd -d sambauser

# Enable user
smbpasswd -e sambauser

# Remove user
smbpasswd -x sambauser
```

---

## Share Home Directories

Edit `/etc/samba/smb.conf`:

```ini
[homes]
   comment = Home Directories
   browsable = no
   writable = yes
   valid users = %S
   create mask = 0700
   directory mask = 0700
```

On RHEL, enable the SELinux boolean:

```bash
setsebool -P samba_enable_home_dirs on
```

---

## Read-Only Share with Write List

```ini
[documents]
   path = /srv/samba/documents
   browsable = yes
   read only = yes
   write list = admin @editors
   valid users = @staff @editors admin
   create mask = 0664
   directory mask = 0775
```

---

## Printer Share

```ini
[printers]
   comment = All Printers
   path = /var/spool/samba
   browseable = no
   guest ok = no
   writable = no
   printable = yes
```

---

## Multi-User Group Share

Useful when a team needs shared read/write access:

```bash
# Create the group
groupadd smbusers

# Create user and add to group
useradd sambauser -s /sbin/nologin
usermod -aG smbusers sambauser

# Set Samba password
smbpasswd -a sambauser

# Create and set permissions
mkdir /share
chown -R root:smbusers /share
chmod 2755 /share
```

The `2755` permission sets the SGID bit so new files inherit the group.

```ini
[share]
   comment = Share
   path = /share
   writable = yes
   valid users = @smbusers
```

---

## Samba Configuration Reference

### Global Options

| Directive | Description | Example |
|-----------|-------------|---------|
| `workgroup` | Windows workgroup or NT domain name | `WORKGROUP` |
| `server string` | Server description shown in browse lists | `Samba Server` |
| `netbios name` | NetBIOS name of the server | `fileserver` |
| `security` | Authentication mode | `user`, `ads` |
| `map to guest` | Map failed logins to guest | `Bad User`, `Never` |
| `interfaces` | Network interfaces to listen on | `eth0 lo`, `192.168.1.0/24` |
| `bind interfaces only` | Only bind to listed interfaces | `yes` |
| `log file` | Log file path (per-client with `%m`) | `/var/log/samba/log.%m` |
| `max log size` | Maximum log file size in KB | `1000` |
| `passdb backend` | Password database backend | `tdbsam` |
| `hosts allow` | Allowed client IPs/networks | `192.168.1. 10.0.0.` |
| `hosts deny` | Denied client IPs/networks | `ALL` |
| `socket options` | TCP performance tuning | `TCP_NODELAY SO_RCVBUF=16384 SO_SNDBUF=16384` |
| `wins support` | Act as a WINS server | `yes` |

### Share Options

| Directive | Description | Values |
|-----------|-------------|--------|
| `comment` | Description shown when browsing shares | `Shared Files` |
| `path` | Filesystem path to share | `/srv/samba/share` |
| `browsable` | Show in network browse lists | `yes` / `no` |
| `writable` | Allow write access | `yes` / `no` |
| `read only` | Opposite of writable | `yes` / `no` |
| `guest ok` | Allow anonymous access | `yes` / `no` |
| `valid users` | Users/groups allowed access | `user1 @group1` |
| `write list` | Users/groups with write even if read only | `admin @editors` |
| `read list` | Users/groups with read-only even if writable | `@viewers` |
| `create mask` | Max permissions for new files | `0664` |
| `directory mask` | Max permissions for new directories | `0775` |
| `force user` | Run all access as this user | `nobody` |
| `force group` | Run all access as this group | `sambashare` |
| `inherit permissions` | New files inherit parent directory permissions | `yes` / `no` |
| `printable` | Enable printer share | `yes` / `no` |
| `public` | Allow anonymous access (synonym for `guest ok`) | `yes` / `no` |

---

## Client Access

### Linux: smbclient

```bash
# List available shares on localhost
smbclient -L localhost -N

# List shares on a remote server
smbclient -L 192.168.1.32 -U sambauser

# Connect to a share
smbclient //192.168.1.32/share -U sambauser

# With workgroup specified
smbclient //192.168.1.32/share -U sambauser -W WORKGROUP

# Inside smbclient (FTP-like interface)
smb: \> ls
smb: \> cd directory
smb: \> get file.txt
smb: \> put local-file.txt
smb: \> mget *.txt
smb: \> mput *.txt
smb: \> mkdir newdir
smb: \> rm filename
smb: \> rmdir directory
smb: \> quit
```

### Network Discovery

```bash
# List machines responding to SMB name queries on the subnet
findsmb
```

### Linux: Mount with CIFS

```bash
# One-time mount (requires cifs-utils)
mount -t cifs //192.168.1.32/share /mnt/share -o username=sambauser,password=secret

# Alternative syntax
mount.cifs //192.168.1.32/share /mnt/share -o username=sambauser,password=secret

# Credentials file (recommended)
mount -t cifs //192.168.1.32/share /mnt/share -o credentials=/root/.smbcredentials

# With specific UID/GID
mount -t cifs //192.168.1.32/share /mnt/share -o credentials=/root/.smbcredentials,uid=1000,gid=1000

# Unmount
umount /mnt/share
```

Create `/root/.smbcredentials`:

```
username=sambauser
password=secret
domain=WORKGROUP
```

Secure it:

```bash
chmod 600 /root/.smbcredentials
```

### Persistent Mount via fstab

Add to `/etc/fstab`:

```
//server/secure  /mnt/share  cifs  credentials=/root/.smbcredentials,_netdev,x-systemd.automount  0  0
```

The `_netdev` option delays mount until the network is available. The `x-systemd.automount` mounts on first access.

### Windows

```cmd
:: Map a network drive
net use Z: \\server\secure /user:sambauser

:: List current connections
net use

:: Disconnect
net use Z: /delete
```

Or use File Explorer: `\\server\secure`

---

## SMB Protocol Versions

Control which protocol versions are allowed:

```ini
[global]
   # Minimum SMB version (security hardening)
   server min protocol = SMB2
   
   # Maximum SMB version
   server max protocol = SMB3
```

| Version | Introduced | Notes |
|---------|-----------|-------|
| SMB1/CIFS | Windows 2000 | Deprecated, insecure — disable if possible |
| SMB2 | Windows Vista | Improved performance, compound requests |
| SMB2.1 | Windows 7 | Leasing, large MTU support |
| SMB3 | Windows 8 | Encryption, multichannel, directory leasing |
| SMB3.0.2 | Windows 8.1 | Minor improvements |
| SMB3.1.1 | Windows 10 | Pre-auth integrity, AES-128-GCM |

---

## Samba as Active Directory Member

### RHEL 7–10

```bash
# Install required packages
dnf install samba samba-client samba-winbind samba-winbind-clients krb5-workstation

# Join the domain
realm discover HOMELAB.LOCAL
realm join HOMELAB.LOCAL -U Administrator

# Verify
realm list
wbinfo --ping-dc
```

### Ubuntu 22.04 / 24.04

```bash
apt install samba winbind libpam-winbind libnss-winbind krb5-user

# Join the domain
realm discover HOMELAB.LOCAL
realm join HOMELAB.LOCAL -U Administrator
```

### smb.conf for AD Member

```ini
[global]
   workgroup = HOMELAB
   security = ads
   realm = HOMELAB.LOCAL
   
   idmap config * : backend = tdb
   idmap config * : range = 10000-19999
   idmap config HOMELAB : backend = rid
   idmap config HOMELAB : range = 20000-99999
   
   winbind use default domain = yes
   winbind enum users = yes
   winbind enum groups = yes
   
   template homedir = /home/%U
   template shell = /bin/bash
```

---

## Troubleshooting

### Validate Configuration

```bash
testparm
testparm -s    # suppress prompts
```

### Check Samba Status

```bash
smbstatus               # active connections and locked files
smbstatus -b            # brief output
smbstatus -S            # shares in use
```

### Debug Logging

Increase log level in `smb.conf`:

```ini
[global]
   log level = 3
   log file = /var/log/samba/log.%m
   max log size = 5000
```

Log levels: 0 (errors only) → 10 (maximum debug). Level 3 is good for troubleshooting.

### View Logs

```bash
# Main Samba log
tail -f /var/log/samba/log.smbd

# Per-machine log (if log file uses %m)
tail -f /var/log/samba/log.HOSTNAME

# NetBIOS name service log
tail -f /var/log/samba/log.nmbd
```

### Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| Permission denied | SELinux context missing | `restorecon -Rv /path` |
| Permission denied | Samba user not created | `smbpasswd -a user` |
| Can't browse shares | nmb not running | `systemctl start nmb` |
| Connection refused | Firewall blocking | `firewall-cmd --add-service=samba --permanent` |
| NT_STATUS_ACCESS_DENIED | File permissions | Check directory ownership and `chmod` |
| Protocol negotiation failed | SMB version mismatch | Adjust `server min protocol` |
| Can't mount CIFS | Missing package | `dnf install cifs-utils` or `apt install cifs-utils` |

### Test Connectivity

```bash
# Test from client
smbclient -L server -N
smbclient -L server -U sambauser

# Check listening ports
ss -tlnp | grep -E '(smb|445|139)'
netstat -tlnp | grep smbd

# Test name resolution
nmblookup fileserver
nslookup fileserver
```

### RHEL 6–Specific Issues

RHEL 6 ships Samba 3.x with older defaults:

```bash
# Check what version is installed
smbd --version

# Default log location
/var/log/samba/

# Reset tdb files if corrupted
rm /var/lib/samba/*.tdb
service smb restart
```

---

## Version Differences by Distribution

| Feature | RHEL 6 | RHEL 7 | RHEL 8 | RHEL 9 | RHEL 10 | Ubuntu 22.04 | Ubuntu 24.04 |
|---------|--------|--------|--------|--------|---------|--------------|--------------|
| Samba version | 3.x | 4.x | 4.x | 4.x | 4.x | 4.x | 4.x |
| Init system | SysVinit | systemd | systemd | systemd | systemd | systemd | systemd |
| Service names | `smb`, `nmb` | `smb`, `nmb` | `smb`, `nmb` | `smb`, `nmb` | `smb`, `nmb` | `smbd`, `nmbd` | `smbd`, `nmbd` |
| SELinux | Yes | Yes | Yes | Yes | Yes | No (AppArmor) | No (AppArmor) |
| Firewall | iptables | firewalld | firewalld | firewalld | firewalld | ufw | ufw |
| Default SMB | SMB1 | SMB2+ | SMB2+ | SMB2+ | SMB3 | SMB2+ | SMB2+ |
| AD join tool | Manual | `realm` | `realm` | `realm` | `realm` | `realm` | `realm` |

---

## Quick Setup: Minimal Authenticated Share

A complete minimal example from install to access:

```bash
# 1. Install (RHEL 8+)
dnf install samba samba-client

# 2. Create user and directory
useradd -M -s /sbin/nologin shareuser
smbpasswd -a shareuser
mkdir -p /srv/samba/data
chown shareuser:shareuser /srv/samba/data
chmod 0755 /srv/samba/data

# 3. SELinux
semanage fcontext -a -t samba_share_t "/srv/samba/data(/.*)?"
restorecon -Rv /srv/samba/data

# 4. Configure smb.conf
cat >> /etc/samba/smb.conf << 'EOF'

[data]
   path = /srv/samba/data
   browsable = yes
   writable = yes
   valid users = shareuser
   create mask = 0644
   directory mask = 0755
EOF

# 5. Validate, start, and open firewall
testparm
systemctl enable --now smb nmb
firewall-cmd --permanent --add-service=samba
firewall-cmd --reload

# 6. Test
smbclient //localhost/data -U shareuser
```

---

## Security Tips

- Always use `security = user` in modern setups
- Disable guest access when not needed (`guest ok = no`)
- Use strong passwords for Samba users
- Restrict valid users to specific groups
- Set `server min protocol = SMB2` to disable insecure SMB1
- Enable encryption when possible (`server smb encrypt = required`)
- Keep Samba updated
- Monitor logs for unauthorized access attempts
- Use `hosts allow` to restrict access by IP/subnet

---

## Useful Links

- Official Samba Documentation: https://www.samba.org/samba/docs/
- Samba Wiki: https://wiki.samba.org/
