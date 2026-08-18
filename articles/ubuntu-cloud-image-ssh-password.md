# Enable SSH Password Auth in Ubuntu Cloud Images

Ubuntu cloud images disable SSH password authentication by default for security. This guide shows how to enable it using `virt-edit` and `virt-customize` from the libguestfs toolkit — useful for lab environments, quick testing, or when key-based auth isn't practical.

## Prerequisites

- `libguestfs-tools` package installed
- Ubuntu cloud image file (qcow2 or img format)
- Proper permissions to modify the image file

## Install libguestfs-tools

### Ubuntu / Debian

```bash
sudo apt update
sudo apt install libguestfs-tools
```

### RHEL / CentOS / Fedora

```bash
sudo dnf install libguestfs-tools
```

## Configuration by Ubuntu Version

OpenSSH 8.9 (shipped with Ubuntu 22.04) renamed `ChallengeResponseAuthentication` to `KbdInteractiveAuthentication`. The behavior is identical — only the name changed.

### OpenSSH Version History

| Ubuntu Version | OpenSSH Version | Interactive Auth Setting |
|----------------|-----------------|-------------------------|
| 18.04 LTS | 7.6 | `ChallengeResponseAuthentication` |
| 20.04 LTS | 8.2 | `ChallengeResponseAuthentication` |
| 22.04 LTS | 8.9 | `KbdInteractiveAuthentication` |
| 24.04 LTS | 9.6 | `KbdInteractiveAuthentication` |

### Ubuntu 20.04 (OpenSSH < 8.9)

SSH settings live in the main `sshd_config` file:

```bash
virt-edit -a "$image_name" /etc/ssh/sshd_config \
  -e 's/^#*PermitRootLogin.*/PermitRootLogin yes/' \
  -e 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' \
  -e 's/^#*ChallengeResponseAuthentication.*/ChallengeResponseAuthentication yes/'
```

### Ubuntu 22.04 and 24.04 (OpenSSH 8.9+)

Cloud-specific settings moved to a drop-in file, and the interactive auth directive was renamed:

```bash
virt-edit -a "$image_name" /etc/ssh/sshd_config.d/60-cloudimg-settings.conf \
  -e 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/'

virt-edit -a "$image_name" /etc/ssh/sshd_config \
  -e 's/^#*KbdInteractiveAuthentication.*/KbdInteractiveAuthentication yes/' \
  -e 's/^#*PermitRootLogin.*/PermitRootLogin yes/'
```

### Universal Approach (All Versions)

Set both parameters for maximum compatibility — unrecognized directives are safely ignored:

```bash
virt-edit -a "$image_name" /etc/ssh/sshd_config \
  -e 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' \
  -e 's/^#*ChallengeResponseAuthentication.*/ChallengeResponseAuthentication yes/' \
  -e 's/^#*KbdInteractiveAuthentication.*/KbdInteractiveAuthentication yes/' \
  -e 's/^#*PermitRootLogin.*/PermitRootLogin yes/' \
  -e 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/'
```

## Version Differences

| Ubuntu Version | SSH Config Location | Notes |
|----------------|---------------------|-------|
| 20.04 | `/etc/ssh/sshd_config` | Direct modification of main config |
| 22.04 | `/etc/ssh/sshd_config.d/60-cloudimg-settings.conf` | Drop-in configuration directory |
| 24.04 | `/etc/ssh/sshd_config.d/60-cloudimg-settings.conf` | Same as 22.04 |

## Complete Script with Version Detection

A script that auto-detects which SSH config format the image uses:

```bash
#!/bin/bash

if [ -z "$1" ]; then
    echo "Usage: $0 <image_name>"
    echo "Example: $0 ubuntu-22.04-server-cloudimg-amd64.img"
    exit 1
fi

image_name="$1"

if [ ! -f "$image_name" ]; then
    echo "Error: Image file '$image_name' not found!"
    exit 1
fi

echo "Configuring SSH in image: $image_name"
echo "Detecting SSH configuration format..."

if virt-cat -a "$image_name" /etc/ssh/sshd_config | grep -q "KbdInteractiveAuthentication"; then
    echo "Detected modern SSH config (Ubuntu 22.04+/OpenSSH 8.9+)"

    virt-edit -a "$image_name" /etc/ssh/sshd_config \
      -e 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' \
      -e 's/^#*KbdInteractiveAuthentication.*/KbdInteractiveAuthentication yes/' \
      -e 's/^#*PermitRootLogin.*/PermitRootLogin yes/' \
      -e 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/'

    # Handle drop-in config if present
    virt-edit -a "$image_name" /etc/ssh/sshd_config.d/60-cloudimg-settings.conf \
      -e 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' 2>/dev/null || true

elif virt-cat -a "$image_name" /etc/ssh/sshd_config | grep -q "ChallengeResponseAuthentication"; then
    echo "Detected legacy SSH config (Ubuntu 20.04/OpenSSH < 8.9)"

    virt-edit -a "$image_name" /etc/ssh/sshd_config \
      -e 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' \
      -e 's/^#*ChallengeResponseAuthentication.*/ChallengeResponseAuthentication yes/' \
      -e 's/^#*PermitRootLogin.*/PermitRootLogin yes/' \
      -e 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/'
else
    echo "Could not detect SSH config format, using universal approach"

    virt-edit -a "$image_name" /etc/ssh/sshd_config \
      -e 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' \
      -e 's/^#*ChallengeResponseAuthentication.*/ChallengeResponseAuthentication yes/' \
      -e 's/^#*KbdInteractiveAuthentication.*/KbdInteractiveAuthentication yes/' \
      -e 's/^#*PermitRootLogin.*/PermitRootLogin yes/' \
      -e 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/'
fi

echo ""
echo "Applied SSH settings:"
virt-cat -a "$image_name" /etc/ssh/sshd_config | grep -E "^(Password|Challenge|KbdInteractive|Pubkey)Authentication|^PermitRootLogin"
echo ""
echo "SSH configuration updated successfully!"
echo "Remember to set passwords for users after boot!"
```

## Verification

Check the SSH configuration after modification:

```bash
# For Ubuntu 20.04
virt-cat -a "$image_name" /etc/ssh/sshd_config | grep -E "(PermitRootLogin|PasswordAuthentication|ChallengeResponseAuthentication)"

# For Ubuntu 22.04/24.04
virt-cat -a "$image_name" /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
virt-cat -a "$image_name" /etc/ssh/sshd_config | grep -i -E "(PasswordAuthentication|KbdInteractiveAuthentication|UsePAM)"
```

### Verify on a Running VM

After boot, check the effective runtime configuration (merges all config files):

```bash
# Show effective sshd config (most reliable method)
ssh user@server "sudo sshd -T | grep -i passwordauthentication"

# Check multiple authentication settings at once
ssh user@server "sudo grep -E '^(PasswordAuthentication|ChallengeResponseAuthentication|KbdInteractiveAuthentication|UsePAM)' /etc/ssh/sshd_config"

# Test that password auth actually works end-to-end
ssh -o PreferredAuthentications=password user@server
```

> **Note:** `sshd -T` shows the compiled runtime config and is more reliable than grepping files directly, since drop-in files and includes may override the main config.

## Conditional Enable (Idempotent)

Only modify settings if they are missing or explicitly disabled — safe for repeated runs:

```bash
# Add ChallengeResponseAuthentication only if not present
if ! virt-cat -a "$image_name" /etc/ssh/sshd_config | grep -q "ChallengeResponseAuthentication"; then
    virt-customize -a "$image_name" --run-command "echo 'ChallengeResponseAuthentication yes' >> /etc/ssh/sshd_config"
fi

# Flip PasswordAuthentication only if explicitly set to no
if virt-cat -a "$image_name" /etc/ssh/sshd_config | grep -q "PasswordAuthentication no"; then
    virt-customize -a "$image_name" --run-command "sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config"
fi
```

## Setting User Passwords After Boot

Once the VM is running, set passwords for users:

```bash
# Set root password
sudo passwd root

# Set ubuntu user password
sudo passwd ubuntu

# Or create a new user
sudo adduser newuser
```

## Alternative: Set Password in the Image

Use `virt-customize` to set a password before first boot:

```bash
# Set root password
virt-customize -a "$image_name" --root-password password:changeme

# Set ubuntu user password
virt-customize -a "$image_name" --password ubuntu:password:changeme
```

## Debug Commands

```bash
# List SSH configuration files in the image
virt-ls -a "$image_name" /etc/ssh/
virt-ls -a "$image_name" /etc/ssh/sshd_config.d/

# View full SSH configuration
virt-cat -a "$image_name" /etc/ssh/sshd_config
virt-cat -a "$image_name" /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
```

## Example Usage

```bash
# Make script executable
chmod +x configure-ssh.sh

# Run with your image
./configure-ssh.sh ubuntu-22.04-server-cloudimg-amd64.img

# Boot VM and connect
ssh root@your-vm-ip
```

## Security Considerations

Enabling password authentication reduces security. For production use, prefer:

1. **SSH key authentication** — use `ssh-copy-id` or cloud-init to deploy keys
2. **Strong passwords** — enforce complexity requirements if password auth is needed
3. **Network restrictions** — limit SSH access with firewall rules
4. **fail2ban** — protect against brute-force attacks
5. **Non-standard port** — reduce automated scanning noise

## Production-Ready Script (Keys Only)

For production images, disable password auth and harden SSH:

```bash
#!/bin/bash
image_name="$1"

if virt-cat -a "$image_name" /etc/ssh/sshd_config | grep -q "KbdInteractiveAuthentication"; then
    virt-edit -a "$image_name" /etc/ssh/sshd_config \
      -e 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' \
      -e 's/^#*KbdInteractiveAuthentication.*/KbdInteractiveAuthentication no/' \
      -e 's/^#*PermitRootLogin.*/PermitRootLogin no/' \
      -e 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/' \
      -e 's/^#*X11Forwarding.*/X11Forwarding no/' \
      -e 's/^#*MaxAuthTries.*/MaxAuthTries 3/'
else
    virt-edit -a "$image_name" /etc/ssh/sshd_config \
      -e 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' \
      -e 's/^#*ChallengeResponseAuthentication.*/ChallengeResponseAuthentication no/' \
      -e 's/^#*PermitRootLogin.*/PermitRootLogin no/' \
      -e 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/' \
      -e 's/^#*X11Forwarding.*/X11Forwarding no/' \
      -e 's/^#*MaxAuthTries.*/MaxAuthTries 3/'
fi
```

## Migrating from 20.04 to 22.04+

If upgrading images, replace the deprecated directive:

```bash
virt-edit -a "$image_name" /etc/ssh/sshd_config \
  -e 's/^ChallengeResponseAuthentication/KbdInteractiveAuthentication/' \
  -e 's/^#ChallengeResponseAuthentication/#KbdInteractiveAuthentication/'
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Permission denied running virt-edit | Run with `sudo` |
| File not found (20.04 path) | Image may be 22.04+ — use drop-in path |
| File not found (22.04 path) | Image may be 20.04 — use main sshd_config |
| Changes not taking effect | Verify with `virt-cat`, check `systemctl status sshd` after boot |
| Cannot connect after boot | Ensure password is set and firewall allows port 22 |
| SSH won't start after changes | Validate syntax: `virt-customize -a "$image_name" --run-command 'sshd -t'` |

### Checking Which Setting Exists

```bash
# Check for legacy format
virt-cat -a "$image_name" /etc/ssh/sshd_config | grep -i challengeresponse

# Check for modern format
virt-cat -a "$image_name" /etc/ssh/sshd_config | grep -i kbdinteractive

# Show all authentication-related settings
virt-cat -a "$image_name" /etc/ssh/sshd_config | grep -v '^#' | grep -i authentication
```
