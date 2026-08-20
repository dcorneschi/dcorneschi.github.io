# Setting Up Chroot SFTP on RHEL and Ubuntu

Configure a restricted SFTP environment where users are jailed to their home directory with no shell access and no ability to browse the filesystem.

## How Chroot SFTP Works

- Users connect via SFTP only (no SSH shell access)
- Users are jailed (chrooted) to a specific directory
- Users cannot navigate outside their jail
- Uses OpenSSH's built-in `internal-sftp` subsystem (no external binaries needed)

## Prerequisites

- OpenSSH server installed and running
- Root access to configure sshd

```bash
# Verify OpenSSH version (chroot SFTP requires 4.9+)
sshd -V 2>&1 | head -1

# Check sshd is running
systemctl status sshd    # RHEL
systemctl status ssh     # Ubuntu
```

## Step 1: Create SFTP Group

```bash
# Create a group for SFTP-only users
sudo groupadd sftpusers
```

## Step 2: Create SFTP User

```bash
# Create user with no shell access
sudo useradd -m -g sftpusers -s /sbin/nologin sftpuser1

# Set password
sudo passwd sftpuser1

# Or create without home directory (we'll set up the jail manually)
sudo useradd -g sftpusers -s /sbin/nologin -d /sftp/sftpuser1 sftpuser1
sudo passwd sftpuser1
```

## Step 3: Set Up Chroot Directory Structure

The chroot directory must be owned by root with no group/other write permissions. The user gets a writable subdirectory inside it.

```bash
# Option A: Jail under /sftp
sudo mkdir -p /sftp/sftpuser1/upload
sudo chown root:root /sftp/sftpuser1
sudo chmod 755 /sftp/sftpuser1

# Create writable directory for the user
sudo chown sftpuser1:sftpusers /sftp/sftpuser1/upload
sudo chmod 755 /sftp/sftpuser1/upload

# Option B: Jail to home directory
sudo mkdir -p /home/sftpuser1/upload
sudo chown root:root /home/sftpuser1
sudo chmod 755 /home/sftpuser1

sudo chown sftpuser1:sftpusers /home/sftpuser1/upload
sudo chmod 755 /home/sftpuser1/upload
```

> **Critical:** The chroot directory and all parent directories must be owned by `root:root` with no write permissions for group/other. If this is wrong, SSH will refuse the connection with "fatal: bad ownership or modes for chroot directory."

## Step 4: Configure sshd

Edit the SSH daemon configuration:

```bash
sudo vi /etc/ssh/sshd_config
```

### Option A: Match by Group

```bash
# Comment out or replace the default Subsystem line
#Subsystem sftp /usr/libexec/openssh/sftp-server
Subsystem sftp internal-sftp

# Add at the END of the file
Match Group sftpusers
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
    PasswordAuthentication yes
```

### Option B: Match by User

```bash
Subsystem sftp internal-sftp

Match User sftpuser1,sftpuser2
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

### Option C: Chroot to Home Directory

```bash
Subsystem sftp internal-sftp

Match Group sftpusers
    ChrootDirectory %h
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

### Configuration Directives Explained

| Directive | Purpose |
|-----------|---------|
| `Subsystem sftp internal-sftp` | Use built-in SFTP server (required for chroot) |
| `ChrootDirectory /sftp/%u` | Jail user to `/sftp/<username>` |
| `ChrootDirectory %h` | Jail user to their home directory |
| `ForceCommand internal-sftp` | Only allow SFTP (no shell, no scp) |
| `AllowTcpForwarding no` | Disable port forwarding |
| `X11Forwarding no` | Disable X11 forwarding |
| `PasswordAuthentication yes` | Allow password auth (or use keys) |

### Variables

| Variable | Expands To |
|----------|-----------|
| `%u` | Username |
| `%h` | User's home directory |
| `%%` | Literal `%` |

## Step 5: Validate and Restart SSH

```bash
# Test configuration for syntax errors (critical — don't skip this)
sudo sshd -t

# If no errors, restart SSH
sudo systemctl restart sshd    # RHEL
sudo systemctl restart ssh     # Ubuntu
```

## Step 6: Test the Connection

```bash
# Test SFTP connection
sftp sftpuser1@server-ip

# Should connect and jail to /sftp/sftpuser1
sftp> pwd
# /

sftp> ls
# upload

sftp> cd /etc
# Couldn't canonicalize: No such file or directory

# Test that SSH is blocked
ssh sftpuser1@server-ip
# This service allows sftp connections only.
# Connection to server-ip closed.
```

## Complete Setup Script (RHEL)

```bash
#!/bin/bash
# setup-sftp-user.sh — Create a chroot SFTP user on RHEL
set -euo pipefail

USERNAME="${1:?Usage: $0 <username>}"
SFTP_BASE="/sftp"

# Create group if it doesn't exist
getent group sftpusers >/dev/null || groupadd sftpusers

# Create user
useradd -g sftpusers -s /sbin/nologin -d "${SFTP_BASE}/${USERNAME}" "${USERNAME}"
echo "Set password for ${USERNAME}:"
passwd "${USERNAME}"

# Create jail directory structure
mkdir -p "${SFTP_BASE}/${USERNAME}/upload"
chown root:root "${SFTP_BASE}/${USERNAME}"
chmod 755 "${SFTP_BASE}/${USERNAME}"
chown "${USERNAME}:sftpusers" "${SFTP_BASE}/${USERNAME}/upload"
chmod 755 "${SFTP_BASE}/${USERNAME}/upload"

echo "User ${USERNAME} created. SFTP jail: ${SFTP_BASE}/${USERNAME}"
echo "Writable directory: ${SFTP_BASE}/${USERNAME}/upload"
echo ""
echo "Ensure /etc/ssh/sshd_config has:"
echo "  Match Group sftpusers"
echo "    ChrootDirectory ${SFTP_BASE}/%u"
echo "    ForceCommand internal-sftp"
```

## Complete Setup Script (Ubuntu)

```bash
#!/bin/bash
# setup-sftp-user.sh — Create a chroot SFTP user on Ubuntu
set -euo pipefail

USERNAME="${1:?Usage: $0 <username>}"
SFTP_BASE="/sftp"

# Create group if it doesn't exist
getent group sftpusers >/dev/null || groupadd sftpusers

# Create user (Ubuntu uses /usr/sbin/nologin)
useradd -g sftpusers -s /usr/sbin/nologin -d "${SFTP_BASE}/${USERNAME}" "${USERNAME}"
echo "Set password for ${USERNAME}:"
passwd "${USERNAME}"

# Create jail directory structure
mkdir -p "${SFTP_BASE}/${USERNAME}/upload"
chown root:root "${SFTP_BASE}/${USERNAME}"
chmod 755 "${SFTP_BASE}/${USERNAME}"
chown "${USERNAME}:sftpusers" "${SFTP_BASE}/${USERNAME}/upload"
chmod 755 "${SFTP_BASE}/${USERNAME}/upload"

echo "User ${USERNAME} created. SFTP jail: ${SFTP_BASE}/${USERNAME}"
echo "Writable directory: ${SFTP_BASE}/${USERNAME}/upload"
```

## SSH Key Authentication (Optional)

```bash
# Create .ssh directory inside the jail (owned by root, readable by user)
sudo mkdir -p /sftp/sftpuser1/.ssh
sudo chown root:root /sftp/sftpuser1/.ssh
sudo chmod 755 /sftp/sftpuser1/.ssh

# Add authorized key
sudo vi /sftp/sftpuser1/.ssh/authorized_keys
# Paste the user's public key

sudo chown root:root /sftp/sftpuser1/.ssh/authorized_keys
sudo chmod 644 /sftp/sftpuser1/.ssh/authorized_keys
```

> **Note:** With chroot, the `.ssh/authorized_keys` file must be inside the chroot directory and readable by sshd (which runs as root during auth). The file ownership by root is intentional.

Update sshd_config to use the correct path:

```bash
Match Group sftpusers
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AuthorizedKeysFile /sftp/%u/.ssh/authorized_keys
    PasswordAuthentication no
```

## Multiple Upload Directories

```bash
# Create multiple writable directories for one user
sudo mkdir -p /sftp/sftpuser1/{upload,incoming,reports}
sudo chown sftpuser1:sftpusers /sftp/sftpuser1/upload
sudo chown sftpuser1:sftpusers /sftp/sftpuser1/incoming
sudo chown sftpuser1:sftpusers /sftp/sftpuser1/reports
```

## Bind Mount Method (Jail with Shared Home)

An alternative approach: keep the user's real data in `/home` and bind-mount it into the chroot jail. Useful when you want the chroot directory separate from actual data storage.

```bash
# Create user and data directory
useradd sftpuser1
passwd sftpuser1
mkdir /home/sftpuser1/data
chown sftpuser1:sftpuser1 /home/sftpuser1/data
chmod 755 /home/sftpuser1/data

# Create chroot jail structure
mkdir -p /var/sftp-chroot/sftpuser1/data
chown root:root /var/sftp-chroot/sftpuser1
chmod 755 /var/sftp-chroot/sftpuser1
chown sftpuser1:sftpuser1 /var/sftp-chroot/sftpuser1/data

# Bind mount home data into the jail
echo '/home/sftpuser1/data  /var/sftp-chroot/sftpuser1/data  none  bind  0 0' >> /etc/fstab
mount /var/sftp-chroot/sftpuser1/data
```

sshd_config:

```bash
Match User sftpuser1
    ChrootDirectory /var/sftp-chroot/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

The user sees `/data` inside their jail, which is actually `/home/sftpuser1/data` on the host.

## Logging SFTP Activity

### Default Logging

SFTP activity is logged to auth log by default:

```bash
# RHEL
tail -f /var/log/secure | grep sftp

# Ubuntu
tail -f /var/log/auth.log | grep sftp
```

### Verbose Logging

Add logging level to the Match block:

```bash
Match Group sftpusers
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp -l INFO
    AllowTcpForwarding no
    X11Forwarding no
```

Log levels: `QUIET`, `FATAL`, `ERROR`, `INFO`, `VERBOSE`, `DEBUG`

### Separate SFTP Log File

```bash
# In sshd_config
Subsystem sftp internal-sftp -l INFO -f LOCAL6

# Create rsyslog rule
echo 'local6.* /var/log/sftp.log' | sudo tee /etc/rsyslog.d/sftp.conf
sudo systemctl restart rsyslog
sudo systemctl restart sshd
```

### Rsyslog Socket Inside Chroot

Since chrooted users can't reach `/dev/log`, create a logging socket inside the jail:

```bash
# Create dev directory inside the chroot
sudo mkdir -p /sftp/sftpuser1/dev

# Configure rsyslog to listen on a socket inside the jail
cat << 'EOF' | sudo tee /etc/rsyslog.d/sftp-chroot.conf
$ModLoad imuxsock
$AddUnixListenSocket /sftp/sftpuser1/dev/log
local3.info /var/log/sftp.log
EOF

# Use LOCAL3 facility in sshd_config
# Subsystem sftp internal-sftp -f LOCAL3 -l INFO
# ForceCommand internal-sftp -f LOCAL3 -l INFO

sudo systemctl restart rsyslog
sudo systemctl restart sshd
```

This ensures SFTP activity is logged even from within the chroot environment.

## SELinux Configuration (RHEL)

If using a non-standard chroot directory:

```bash
# Set correct SELinux context
sudo semanage fcontext -a -t ssh_home_t "/sftp(/.*)?"
sudo restorecon -Rv /sftp

# Or if using /home
sudo restorecon -Rv /home
```

## Bandwidth Limiting (Optional)

Use `internal-sftp` with rate limiting via pam_limits or traffic shaping:

```bash
# /etc/security/limits.conf — limit max file size per user
sftpuser1    hard    fsize    1048576    # 1 GB max file size
```

## Troubleshooting

### "fatal: bad ownership or modes for chroot directory"

```bash
# The chroot directory and ALL parents must be:
# - Owned by root:root
# - No write permission for group or other (max 755)

# Check ownership chain
ls -ld / /sftp /sftp/sftpuser1
namei -l /sftp/sftpuser1

# Fix
sudo chown root:root /sftp/sftpuser1
sudo chmod 755 /sftp/sftpuser1
```

### "This service allows sftp connections only" (Expected)

This message appears when a user tries SSH instead of SFTP. This is correct behavior — `ForceCommand internal-sftp` blocks shell access.

### User Can Connect But Can't Upload

```bash
# Check writable directory ownership
ls -la /sftp/sftpuser1/
# upload/ should be owned by the user

# Fix
sudo chown sftpuser1:sftpusers /sftp/sftpuser1/upload

# User uploads to /upload, not to /
sftp> cd upload
sftp> put myfile.txt
```

### "Permission denied" on SFTP Connect

```bash
# Check user shell is nologin
grep sftpuser1 /etc/passwd
# Should end with /sbin/nologin or /usr/sbin/nologin

# Check user is in sftpusers group
id sftpuser1

# Check sshd_config syntax
sudo sshd -t

# Check SSH logs
sudo tail -20 /var/log/secure    # RHEL
sudo tail -20 /var/log/auth.log  # Ubuntu
```

### Match Block Not Working

```bash
# Match blocks MUST be at the END of sshd_config
# Nothing after the Match block applies globally

# Verify order
tail -20 /etc/ssh/sshd_config

# Test with specific user
sudo sshd -T -C user=sftpuser1 | grep -i chroot
```

## Remove SFTP User

```bash
# Remove user
sudo userdel sftpuser1

# Remove jail directory
sudo rm -rf /sftp/sftpuser1

# Or keep data
sudo userdel sftpuser1
# Directory remains at /sftp/sftpuser1
```

## Quick Reference

| Action | Command |
|--------|---------|
| Create group | `groupadd sftpusers` |
| Create user | `useradd -g sftpusers -s /sbin/nologin username` |
| Set password | `passwd username` |
| Create jail | `mkdir -p /sftp/user/upload` |
| Set jail ownership | `chown root:root /sftp/user && chmod 755 /sftp/user` |
| Set upload ownership | `chown user:sftpusers /sftp/user/upload` |
| Test sshd config | `sshd -t` |
| Restart SSH (RHEL) | `systemctl restart sshd` |
| Restart SSH (Ubuntu) | `systemctl restart ssh` |
| Test connection | `sftp user@host` |
| Check logs (RHEL) | `tail -f /var/log/secure` |
| Check logs (Ubuntu) | `tail -f /var/log/auth.log` |
| Debug Match block | `sshd -T -C user=username \| grep chroot` |
