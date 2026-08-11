# Troubleshooting cloud-init on Proxmox

This guide covers common cloud-init issues on Proxmox VE, how to debug them, and how to properly configure cloud-init for VM templates.

## How cloud-init Works in Proxmox

Proxmox generates a cloud-init NoCloud ISO and attaches it as a CD-ROM drive to the VM. On first boot, cloud-init inside the guest reads this ISO and applies the configuration (hostname, users, network, SSH keys).

```
Proxmox (host)                          Guest VM
┌─────────────────┐                    ┌──────────────────┐
│ qm set --ci*    │ → generates →      │ cloud-init reads │
│ parameters      │   NoCloud ISO      │ meta-data        │
│                 │   (attached as     │ user-data        │
│                 │    IDE/SCSI CD)    │ network-config   │
└─────────────────┘                    └──────────────────┘
```

## Setting Up cloud-init in Proxmox

### Add cloud-init Drive

```bash
# Add cloud-init CD-ROM drive
qm set <vmid> --ide2 local-lvm:cloudinit

# Or on SCSI bus
qm set <vmid> --scsi1 local-lvm:cloudinit
```

### Configure cloud-init Parameters

```bash
# User and password
qm set <vmid> --ciuser admin
qm set <vmid> --cipassword "SecurePassword123"

# SSH keys
qm set <vmid> --sshkeys ~/.ssh/id_ed25519.pub

# Multiple SSH keys (from file with one key per line)
qm set <vmid> --sshkeys /tmp/authorized_keys

# Network (static IP)
qm set <vmid> --ipconfig0 ip=192.168.50.100/24,gw=192.168.50.1

# Network (DHCP)
qm set <vmid> --ipconfig0 ip=dhcp

# DNS
qm set <vmid> --nameserver 192.168.50.10
qm set <vmid> --searchdomain homelab.local

# Hostname (derived from VM name by default)
# To override:
qm set <vmid> --ciuser admin --name custom-hostname
```

### View Generated cloud-init Config

```bash
# Dump the generated cloud-init user-data
qm cloudinit dump <vmid> user

# Dump network-config
qm cloudinit dump <vmid> network

# Dump meta-data
qm cloudinit dump <vmid> meta
```

## Common Issues and Fixes

### 1. cloud-init Not Running at All

**Symptoms:** VM boots but has no configured hostname, no user, default network (DHCP or no network).

**Causes:**
- cloud-init is not installed in the guest image
- cloud-init drive not attached
- cloud-init already ran (instance-id cached)

**Fix:**

```bash
# Check if cloud-init drive is attached
qm config <vmid> | grep -i cloudinit

# If missing, add it:
qm set <vmid> --ide2 local-lvm:cloudinit

# Inside the guest, check if cloud-init is installed
dpkg -l cloud-init       # Ubuntu/Debian
rpm -q cloud-init        # RHEL/CentOS
```

### 2. cloud-init Ran But Didn't Apply Changes

**Symptoms:** First boot applied correctly, but subsequent changes to `qm set --ci*` don't take effect.

**Cause:** cloud-init caches the instance-id. If it hasn't changed, cloud-init skips re-configuration on subsequent boots.

**Fix (inside the guest):**

```bash
# Clean cloud-init state (forces re-run on next boot)
cloud-init clean

# Or remove the instance marker manually
rm -rf /var/lib/cloud/instances/
rm -f /var/lib/cloud/instance

# Reboot
reboot
```

**Fix (from Proxmox host):**

```bash
# Regenerate the cloud-init image (forces new instance-id)
qm set <vmid> --ciuser admin   # any change triggers regeneration
qm start <vmid>
```

### 3. Network Not Configured

**Symptoms:** VM boots but has no IP or wrong IP. `ip a` shows no configured address.

**Causes:**
- `--ipconfig0` not set
- Network interface name mismatch
- cloud-init network module not rendering for the guest's network manager

**Debug:**

```bash
# Check what Proxmox generates
qm cloudinit dump <vmid> network

# Inside the guest, check cloud-init network log
cat /var/lib/cloud/instance/boot-finished
cat /var/log/cloud-init.log | grep -i network
cat /var/log/cloud-init-output.log
```

**Fix:**

```bash
# Ensure ipconfig0 is set
qm set <vmid> --ipconfig0 ip=192.168.50.100/24,gw=192.168.50.1

# If using multiple NICs:
qm set <vmid> --ipconfig0 ip=192.168.50.100/24,gw=192.168.50.1
qm set <vmid> --ipconfig1 ip=10.0.0.100/24
```

**Interface name mismatch:** Some images use `ens18` instead of `eth0`. cloud-init's network config from Proxmox uses device indices, not names, so this usually works. If not:

```bash
# Inside the guest, check interface names
ip link show

# Cloud-init log shows what it tried to configure
grep -i "network" /var/log/cloud-init.log
```

### 4. SSH Key Not Working

**Symptoms:** Can't SSH with key, only password works (or nothing works).

**Causes:**
- SSH key format issue
- Key not properly URL-encoded
- Wrong user (key is injected for `--ciuser`, not root)

**Fix:**

```bash
# Verify the key is set
qm cloudinit dump <vmid> user | grep ssh

# Ensure the key file is in correct format (one key per line)
cat ~/.ssh/id_ed25519.pub

# Re-set the key
qm set <vmid> --sshkeys ~/.ssh/id_ed25519.pub

# SSH to the correct user (not root unless ciuser=root)
ssh admin@192.168.50.100    # if --ciuser admin
```

**URL encoding issue:** If setting keys via API or with special characters in the path, the key content must be URL-encoded. Using `--sshkeys` with a file path handles this automatically.

### 5. Password Authentication Not Working

**Symptoms:** Can't login with the password set via `--cipassword`.

**Causes:**
- SSH password authentication disabled in guest sshd_config
- Password expired

**Fix (from Proxmox):**

```bash
# Check generated user-data
qm cloudinit dump <vmid> user | grep -A5 chpasswd
```

**Fix (inside guest, via console):**

```bash
# Check if password auth is enabled
grep -i passwordauth /etc/ssh/sshd_config

# Enable password auth
sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
systemctl restart sshd

# Check password expiry
chage -l admin
```

### 6. cloud-init Drive Not Detected by Guest

**Symptoms:** Guest boots but cloud-init logs show "No datasource found".

**Causes:**
- Wrong bus type for the cloud-init drive
- Guest kernel doesn't have the right drivers
- cloud-init datasource not configured for NoCloud

**Fix:**

```bash
# Try different bus for cloud-init drive
qm set <vmid> --delete ide2
qm set <vmid> --ide2 local-lvm:cloudinit

# Or use SCSI
qm set <vmid> --delete ide2
qm set <vmid> --scsi1 local-lvm:cloudinit

# Inside guest, check if the CD-ROM is visible
lsblk
blkid | grep -i cidata

# Check cloud-init datasource config
cat /etc/cloud/cloud.cfg.d/90_dpkg.cfg    # Ubuntu
cat /etc/cloud/cloud.cfg                   # RHEL
```

Ensure the datasource list includes `NoCloud`:

```yaml
# /etc/cloud/cloud.cfg.d/99-proxmox.cfg
datasource_list: [NoCloud, ConfigDrive, None]
```

### 7. Hostname Not Set

**Symptoms:** VM has generic hostname like `localhost` instead of the VM name.

**Cause:** cloud-init uses the VM name from Proxmox meta-data.

**Fix:**

```bash
# Set VM name (used as hostname)
qm set <vmid> --name my-hostname

# Or verify what meta-data provides
qm cloudinit dump <vmid> meta
```

### 8. Custom cloud-init Config (Advanced)

Proxmox only exposes basic cloud-init options. For advanced configuration, use a custom user-data snippet:

```bash
# Create a snippets directory on storage
pvesm set local --content snippets

# Create custom user-data
cat > /var/lib/vz/snippets/custom-cloud-init.yaml << 'EOF'
#cloud-config
packages:
  - vim
  - htop
  - qemu-guest-agent

runcmd:
  - systemctl enable --now qemu-guest-agent
  - echo "Custom cloud-init ran" > /tmp/cloud-init-done

write_files:
  - path: /etc/motd
    content: |
      Welcome to this Proxmox VM
      Managed by cloud-init
EOF

# Set custom user-data on the VM
qm set <vmid> --cicustom "user=local:snippets/custom-cloud-init.yaml"

# Can also customize network and meta
qm set <vmid> --cicustom "user=local:snippets/user.yaml,network=local:snippets/network.yaml,meta=local:snippets/meta.yaml"
```

## Debugging cloud-init Inside the Guest

### Check cloud-init Status

```bash
# Show overall status
cloud-init status

# Show status with details
cloud-init status --long

# Wait for cloud-init to complete (useful in scripts)
cloud-init status --wait
```

### Check Logs

```bash
# Main cloud-init log
cat /var/log/cloud-init.log

# Output log (stdout/stderr from runcmd, etc.)
cat /var/log/cloud-init-output.log

# Check what modules ran and their results
cat /var/lib/cloud/data/status.json | python3 -m json.tool

# Check instance metadata
cat /var/lib/cloud/instance/user-data.txt
cat /var/lib/cloud/instance/meta-data
cat /var/lib/cloud/instance/network-config
```

### Manually Re-Run cloud-init

```bash
# Clean all cloud-init state
cloud-init clean

# Re-run all modules
cloud-init init
cloud-init modules --mode config
cloud-init modules --mode final

# Or reboot (cleanest way after cloud-init clean)
cloud-init clean && reboot
```

### Check What cloud-init Configured

```bash
# Show rendered network config
cat /etc/netplan/*.yaml              # Ubuntu
cat /etc/NetworkManager/system-connections/*   # RHEL 9
cat /etc/sysconfig/network-scripts/ifcfg-*    # RHEL 7/8

# Check created users
cat /etc/passwd | grep -v nologin | tail -5

# Check SSH keys were deployed
cat /home/admin/.ssh/authorized_keys
```

## cloud-init Modules and Boot Stages

cloud-init runs in 4 stages:

| Stage | When | Purpose |
|-------|------|---------|
| Generator | systemd boot | Detect datasource |
| Network (init) | After network up | Set hostname, mount, grow disk |
| Config | Mid-boot | SSH keys, users, packages |
| Final | Late boot | Run commands, scripts |

```bash
# See which modules are configured to run
cat /etc/cloud/cloud.cfg | grep -A50 "cloud_init_modules"
```

## Reset cloud-init for Template Creation

Before converting a VM to a template, clean cloud-init so it runs fresh on each clone:

```bash
# Inside the guest
cloud-init clean
rm -rf /var/lib/cloud/
truncate -s 0 /etc/machine-id

# Then from Proxmox host
qm shutdown <vmid>
qm template <vmid>
```

## Debugging Without SSH Access

### Use the Proxmox Serial Console

If the VM has no network or SSH isn't working, use the serial console:

1. In Proxmox UI: **VM → Console**. If screen is blank, switch to Serial console.
2. Ensure the VM has a serial device:
   - **Hardware → Add → Serial Port** (if not present)
   - **Options → Display → Serial terminal**

From the Proxmox host CLI:

```bash
# List VMs
qm list

# Attach to serial console (Ctrl+] to exit)
qm terminal <vmid>
```

> **Tip:** Set display to serial temporarily so all boot/cloud-init output is visible during debugging.

### Pull Logs via QEMU Guest Agent (No SSH)

If the guest agent is running inside the VM:

```bash
# Pull cloud-init logs to the Proxmox host
qm guest file pull <vmid> /var/log/cloud-init.log /root/cloud-init-<vmid>.log
qm guest file pull <vmid> /var/log/cloud-init-output.log /root/cloud-init-output-<vmid>.log

# Query cloud-init status
qm guest exec <vmid> -- cloud-init status --long

# If a PID is printed, inspect the result
qm guest exec-status <vmid> <PID>
```

> **Tip:** Install `qemu-guest-agent` in the `packages` list (not `runcmd`) so it's available earlier in the boot process.

### Read Logs Offline by Mounting the VM Disk

**qcow2 storage (e.g., local directory):**

```bash
VMID=<vmid>
IMG=/var/lib/vz/images/$VMID/vm-$VMID-disk-0.qcow2

modprobe nbd max_part=8
qemu-nbd -r -c /dev/nbd0 "$IMG"
partprobe /dev/nbd0
mkdir -p /mnt/vm
mount -o ro /dev/nbd0p2 /mnt/vm   # adjust partition number as needed

# Inspect logs
less /mnt/vm/var/log/cloud-init.log
less /mnt/vm/var/log/cloud-init-output.log

# Clean up
umount /mnt/vm
qemu-nbd -d /dev/nbd0
```

**LVM-thin storage (e.g., local-lvm):**

```bash
VMID=<vmid>
LV=/dev/pve/vm-$VMID-disk-0   # Verify with: lvdisplay; qm config $VMID

lvchange -ay "$LV"
kpartx -av "$LV"
mkdir -p /mnt/vm
mount -o ro /dev/mapper/$(basename $LV)p2 /mnt/vm   # adjust partition number

# Inspect logs
less /mnt/vm/var/log/cloud-init.log
less /mnt/vm/var/log/cloud-init-output.log

# Clean up
umount /mnt/vm
kpartx -dv "$LV"
```

> **Important:** Always mount read-only (`-o ro`) when inspecting a VM disk from the host.

### Enable Debug Logging for Next Boot

Create a custom cloud-init snippet with debug enabled:

```yaml
#cloud-config
debug: true
bootcmd:
  - systemctl enable --now serial-getty@ttyS0.service
output:
  all: '| tee -a /var/log/cloud-init-output.log'
```

### Temporary Password Login (For Debugging Only)

If SSH keys aren't working and you need console access:

```yaml
#cloud-config
ssh_pwauth: true
users:
  - name: debug
    lock_passwd: false
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
chpasswd:
  list: |
    debug:ChangeMeNow!
  expire: false
```

> **Remove temporary password login after debugging!**

## What to Look for in Logs

### Common Failures

| Pattern in Logs | Likely Cause | Fix |
|---|---|---|
| `apt update` failures | DNS not reachable from VM | Check DNS in `--nameserver` or network config |
| `Failed to start ssh` | sshd_config syntax error | Check custom write_files that modify sshd |
| `WARNING: Unhandled non-multipart userdata` | Invalid YAML in user-data | Validate YAML syntax |
| `DataSourceNone: No data found` | cloud-init drive not detected | Check drive attachment and datasource config |
| `url_helper.py ... Connection refused` | VM can't reach metadata endpoint | Normal for NoCloud — ignore if using Proxmox |

### DNS/Network Issues

If packages fail to install, it's usually DNS:

```bash
# Check DNS configured by cloud-init
cat /etc/resolv.conf

# Prefer configuring DNS via netplan or systemd-resolved
# rather than writing /etc/resolv.conf directly
```

### Validate YAML Before Deploying

```bash
# Validate cloud-init user-data syntax
cloud-init schema --config-file /path/to/user-data.yaml

# Or use Python
python3 -c "import yaml; yaml.safe_load(open('user-data.yaml'))"
```

## After Fixing

- Remove temporary password login
- Set display back from serial to default (if changed)
- Re-apply configuration and monitor serial console during first boot
- Verify with `cloud-init status --long` inside the guest

## Quick Reference

```bash
# Add cloud-init drive
qm set <vmid> --ide2 local-lvm:cloudinit

# Set basic config
qm set <vmid> --ciuser admin --cipassword pass
qm set <vmid> --sshkeys ~/.ssh/id_ed25519.pub
qm set <vmid> --ipconfig0 ip=192.168.50.100/24,gw=192.168.50.1
qm set <vmid> --nameserver 192.168.50.10

# Dump generated config
qm cloudinit dump <vmid> user
qm cloudinit dump <vmid> network

# Custom user-data
qm set <vmid> --cicustom "user=local:snippets/custom.yaml"

# Inside guest: debug
cloud-init status --long
cat /var/log/cloud-init.log
cloud-init clean && reboot

# Template workflow
cloud-init clean && rm -rf /var/lib/cloud/ && truncate -s 0 /etc/machine-id
# then from host: qm template <vmid>
```
