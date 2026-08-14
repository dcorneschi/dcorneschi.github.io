# KVM libguestfs Tools: virt-edit, virt-cat, and Friends

The `libguestfs-tools` package provides utilities for accessing and modifying VM disk images without booting the guest. These work on qcow2, raw, VMDK, and other formats.

## Installation

```sh
# RHEL / CentOS / Fedora
sudo dnf install libguestfs-tools

# Ubuntu / Debian
sudo apt install libguestfs-tools

# Verify
virt-cat --version
```

## virt-cat — Read Files from a Disk Image

Display file contents from a guest image without mounting it.

```sh
# Read a file from a running domain
virt-cat -d <domain> /etc/hostname

# Read from a disk image file
virt-cat -a /var/lib/libvirt/images/vm.qcow2 /etc/fstab

# Read from a domain's disk
virt-cat -d myvm /etc/resolv.conf

# Read cloud-init logs
virt-cat -d myvm /var/log/cloud-init-output.log

# Check network config
virt-cat -d myvm /etc/sysconfig/network-scripts/ifcfg-eth0
```

## virt-edit — Edit Files in a Disk Image

Edit files inside a guest disk image. The VM must be shut down.

```sh
# Edit a file interactively (opens $EDITOR)
virt-edit -d <domain> /etc/fstab

# Edit from a disk image
virt-edit -a /var/lib/libvirt/images/vm.qcow2 /etc/hostname

# Non-interactive: use -e for sed-like expressions
virt-edit -d <domain> /etc/selinux/config -e 's/SELINUX=enforcing/SELINUX=disabled/'

# Change hostname
virt-edit -d myvm /etc/hostname -e 's/.*/newhost/'

# Fix a broken fstab entry
virt-edit -d myvm /etc/fstab -e 's|/dev/sdb1|#/dev/sdb1|'

# Enable root login via SSH
virt-edit -d myvm /etc/ssh/sshd_config -e 's/^PermitRootLogin.*/PermitRootLogin yes/'

# Set a static IP (RHEL/CentOS)
virt-edit -d myvm /etc/sysconfig/network-scripts/ifcfg-eth0 -e 's/BOOTPROTO=dhcp/BOOTPROTO=static/'
```

### Robust sed Patterns for virt-edit

The `-e` flag accepts sed substitute expressions. Use `^#*` to match both commented and uncommented lines:

```sh
# Pattern: s/^#*Setting.*/Setting value/
# ^    = start of line
# #*   = zero or more # characters (handles commented or uncommented)
# .*   = rest of line (any current value)

# Enable password authentication (handles #PasswordAuthentication no, PasswordAuthentication no, etc.)
virt-edit -a "$image" /etc/ssh/sshd_config -e 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/'

# Enable root login
virt-edit -a "$image" /etc/ssh/sshd_config -e 's/^#*PermitRootLogin.*/PermitRootLogin yes/'

# Enable pubkey auth
virt-edit -a "$image" /etc/ssh/sshd_config -e 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/'
```

What `^#*PasswordAuthentication.*` matches:

```
PasswordAuthentication no           ✓ uncommented
#PasswordAuthentication no          ✓ commented
##PasswordAuthentication no         ✓ multiple comment chars
PasswordAuthentication yes          ✓ already correct (idempotent)
```

### Multiple -e Expressions in One Command

Chain multiple edits in a single invocation (faster than multiple virt-edit calls):

```sh
virt-edit -a "$image" /etc/ssh/sshd_config \
  -e 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' \
  -e 's/^#*PermitRootLogin.*/PermitRootLogin yes/' \
  -e 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/' \
  -e 's/^#*UseDNS.*/UseDNS no/'
```

### Verify Changes with virt-cat

Always verify before and after:

```sh
# Before
virt-cat -a "$image" /etc/ssh/sshd_config | grep -E "^#*(Password|PermitRoot|Pubkey)"

# Apply
virt-edit -a "$image" /etc/ssh/sshd_config \
  -e 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' \
  -e 's/^#*PermitRootLogin.*/PermitRootLogin yes/'

# After
virt-cat -a "$image" /etc/ssh/sshd_config | grep -E "^(Password|PermitRoot|Pubkey)"
```

### Test Patterns Before Applying

```sh
# Dry run: see what lines would match
virt-cat -a "$image" /etc/ssh/sshd_config | grep -E "^#*PasswordAuthentication"

# Preview the replacement
virt-cat -a "$image" /etc/ssh/sshd_config | sed 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' | grep PasswordAuthentication
```

> **Warning:** Never use virt-edit on a running VM — it will corrupt the filesystem.

## virt-ls — List Files and Directories

```sh
# List directory contents
virt-ls -d <domain> /etc/

# Long format (permissions, size, dates)
virt-ls -l -d <domain> /var/log/

# Recursive listing
virt-ls -R -d <domain> /etc/yum.repos.d/

# From a disk image
virt-ls -a /var/lib/libvirt/images/vm.qcow2 /home/
```

## virt-copy-out — Copy Files Out of an Image

```sh
# Copy a file from the guest to the host
virt-copy-out -d <domain> /etc/passwd /tmp/

# Copy a directory
virt-copy-out -d <domain> /var/log/ /tmp/guest-logs/

# From a disk image
virt-copy-out -a vm.qcow2 /etc/ssh/sshd_config /tmp/
```

## virt-copy-in — Copy Files Into an Image

VM must be shut down.

```sh
# Copy a file into the guest
virt-copy-in -d <domain> /tmp/authorized_keys /root/.ssh/

# Copy a directory into the guest
virt-copy-in -d <domain> /tmp/configs/ /etc/

# From a disk image
virt-copy-in -a vm.qcow2 /tmp/resolv.conf /etc/
```

## virt-customize — Bulk Modify a Disk Image

Apply multiple changes in a single pass. Very useful for image preparation.

```sh
# Set root password
virt-customize -a vm.qcow2 --root-password password:changeme

# Install packages
virt-customize -a vm.qcow2 --install vim,curl,wget

# Run a command
virt-customize -a vm.qcow2 --run-command 'systemctl enable nginx'

# Upload a file
virt-customize -a vm.qcow2 --upload /tmp/motd:/etc/motd

# Set hostname
virt-customize -a vm.qcow2 --hostname newhost.example.com

# Inject SSH key
virt-customize -a vm.qcow2 --ssh-inject root:file:/home/user/.ssh/id_ed25519.pub

# Combine multiple operations
virt-customize -a vm.qcow2 \
  --root-password password:changeme \
  --hostname webserver01 \
  --install httpd,mod_ssl \
  --run-command 'systemctl enable httpd' \
  --ssh-inject root:file:~/.ssh/id_ed25519.pub \
  --selinux-relabel

# First boot script
virt-customize -a vm.qcow2 --firstboot /tmp/setup.sh

# Delete a user
virt-customize -a vm.qcow2 --run-command 'userdel -r olduser'

# Timezone
virt-customize -a vm.qcow2 --timezone Europe/Amsterdam
```

## virt-sysprep — Prepare an Image as a Template

Remove instance-specific data (SSH keys, hostname, MAC addresses, logs) to make the image reusable as a template.

```sh
# Basic sysprep
virt-sysprep -a vm.qcow2

# Sysprep a domain (must be shut down)
virt-sysprep -d <domain>

# Keep SSH keys but remove everything else
virt-sysprep -a vm.qcow2 --enable defaults --disable ssh-hostkeys

# List available operations
virt-sysprep --list-operations

# Only run specific operations
virt-sysprep -a vm.qcow2 --operations hostname,logfiles,net-hwaddr,udev-persistent-net
```

What virt-sysprep removes by default:
- SSH host keys
- Machine ID
- Hostname
- Network MAC references (udev rules)
- Log files
- Bash history
- tmp files

## virt-df — Show Disk Usage

```sh
# Show disk usage for all guests
virt-df

# Human-readable
virt-df -h

# For a specific domain
virt-df -d <domain> -h

# From a disk image
virt-df -a vm.qcow2 -h
```

## virt-filesystems — List Filesystems

```sh
# List filesystems in an image
virt-filesystems -a vm.qcow2

# Long format with sizes
virt-filesystems -a vm.qcow2 --long --human-readable

# Show all (filesystems, partitions, block devices, LVs)
virt-filesystems -a vm.qcow2 --all --long -h

# For a running domain
virt-filesystems -d <domain> --all --long -h
```

## virt-inspector — Detailed Guest Info

```sh
# Get OS info, package list, etc.
virt-inspector -a vm.qcow2

# Output as XML
virt-inspector -d <domain> > guest-info.xml

# Just get the OS info
virt-inspector -a vm.qcow2 | virt-inspector --xpath '//operatingsystem'
```

## virt-resize — Resize Disk Images

```sh
# Expand a disk image
truncate -s 50G new-disk.qcow2
virt-resize --expand /dev/sda2 old-disk.qcow2 new-disk.qcow2

# Expand LVM
virt-resize --expand /dev/sda2 --LV-expand /dev/vg/lv old-disk.qcow2 new-disk.qcow2
```

## virt-sparsify — Reclaim Empty Space

```sh
# In-place (thin-provisioned)
virt-sparsify --in-place vm.qcow2

# Create a new sparse copy
virt-sparsify vm.qcow2 vm-sparse.qcow2

# Convert to compressed qcow2
virt-sparsify --convert qcow2 --compress vm.raw vm-compressed.qcow2
```

## virt-log — View Guest Logs

```sh
# View system journal from a guest
virt-log -d <domain>

# From an image
virt-log -a vm.qcow2
```

## guestfish — Interactive Shell

For complex operations, `guestfish` provides an interactive shell into the disk image.

```sh
# Interactive access
guestfish -a vm.qcow2 -i

# One-liner: read a file
guestfish -a vm.qcow2 -i cat /etc/hostname

# One-liner: upload a file
guestfish -a vm.qcow2 -i upload /tmp/local-file /etc/remote-file

# One-liner: set permissions
guestfish -a vm.qcow2 -i chmod 0600 /root/.ssh/authorized_keys

# Scripted operations
guestfish -a vm.qcow2 -i <<EOF
write /etc/hostname "myhost\n"
upload /tmp/resolv.conf /etc/resolv.conf
chmod 0644 /etc/resolv.conf
EOF
```

## One-Liners and Recipes

```sh
# Quick password reset for a locked VM
virt-customize -d myvm --root-password password:temppass123

# Fix SELinux after editing files (forces relabel on next boot)
virt-customize -a vm.qcow2 --selinux-relabel

# Extract all SSH host keys from a guest
virt-copy-out -d myvm /etc/ssh/ssh_host_* /tmp/keys/

# Check if cloud-init ran successfully
virt-cat -d myvm /var/lib/cloud/data/result.json

# Get IP from DHCP leases inside the guest
virt-cat -d myvm /var/lib/dhclient/dhclient.leases

# Disable firewalld in an image
virt-customize -a vm.qcow2 --run-command 'systemctl disable firewalld'

# Inject a cron job
virt-customize -a vm.qcow2 --write '/var/spool/cron/root:0 * * * * /usr/local/bin/healthcheck.sh'

# Diff two disk images (compare files)
guestfish -a old.qcow2 -i cat /etc/passwd > /tmp/old-passwd
guestfish -a new.qcow2 -i cat /etc/passwd > /tmp/new-passwd
diff /tmp/old-passwd /tmp/new-passwd

# Batch rename VMs by changing hostname in all images
for img in /var/lib/libvirt/images/web*.qcow2; do
  name=$(basename "$img" .qcow2)
  virt-customize -a "$img" --hostname "$name.example.com"
done

# Check disk usage across all VMs
for dom in $(virsh list --all --name); do
  echo "=== $dom ==="
  virt-df -d "$dom" -h 2>/dev/null
done
```

## Tips and Gotchas

- **VM must be shut down** for write operations (virt-edit, virt-copy-in, virt-customize, virt-sysprep). Read operations (virt-cat, virt-ls, virt-copy-out) work on running VMs but may show inconsistent data.
- **Always backup before editing**: `cp image.qcow2 image.qcow2.backup` or use `virt-copy-out` to save the original file.
- **SELinux relabeling**: After editing files with virt-edit or virt-customize, the security contexts may be wrong. Always add `--selinux-relabel` or `touch /.autorelabel` to force a relabel on next boot.
- **LVM inside guests**: libguestfs handles LVM automatically — you don't need to manually activate VGs.
- **Windows guests**: These tools work on NTFS too. Use `virt-cat -d winvm /Windows/System32/drivers/etc/hosts`.
- **Performance**: For multiple operations, use `virt-customize` with chained options rather than calling virt-edit multiple times (each invocation boots a helper VM).
- **Snapshots**: If the disk has snapshots, you may need to specify the backing file explicitly with `-a`.
- **QEMU user**: On some distros, libguestfs runs as the `qemu` user. Ensure the image file is readable by that user, or run commands with `sudo`.
- **Debugging**: If commands hang or fail, try `LIBGUESTFS_DEBUG=1 LIBGUESTFS_TRACE=1 virt-cat -d myvm /etc/hostname` for verbose output.


## virt-edit sed Expressions: Append vs Replace

### Appending Lines

The `$a\` sed command appends text at the end of the file — equivalent to `echo "text" >> file`:

```sh
# Append a line at end of file
virt-edit -a "$image" /etc/ssh/sshd_config -e '$a\ChallengeResponseAuthentication yes'

# Insert at the beginning
virt-edit -a "$image" /etc/ssh/sshd_config -e '1i\# Managed by automation'

# Append after line 3
virt-edit -a "$image" /etc/ssh/sshd_config -e '3a\# Added setting'
```

**Problems with appending:**
- Creates duplicates if the setting already exists
- SSH uses the last occurrence, but leaves a messy config
- No validation of existing state

### Replacing Lines (Preferred)

The `s/^#*Setting.*/Setting value/` pattern is always better for existing config files:

```sh
virt-edit -a "$image" /etc/ssh/sshd_config -e 's/^#*ChallengeResponseAuthentication.*/ChallengeResponseAuthentication yes/'
```

**Benefits:**
- No duplicates
- Handles commented and uncommented lines
- Clean config files
- Idempotent (safe to run multiple times)

### Mixed Approach: Replace + Append Fallback

For settings that might not exist in the file at all:

```sh
virt-edit -a "$image" /etc/ssh/sshd_config \
  -e 's/^#*ChallengeResponseAuthentication.*/ChallengeResponseAuthentication yes/' \
  -e '/^ChallengeResponseAuthentication/!{$a\ChallengeResponseAuthentication yes
}'
```

This replaces the line if it exists, or appends it if it doesn't.

### sed Expression Quick Reference

| Task | Expression | Example |
|------|-----------|---------|
| Append at end | `$a\text` | `$a\NewSetting yes` |
| Insert at start | `1i\text` | `1i\# Header` |
| Replace line | `s/pattern/replacement/` | `s/^#*Setting.*/Setting yes/` |
| Change line | `/pattern/c\text` | `/^#*Setting/c\Setting yes` |
| Delete line | `/pattern/d` | `/^#.*legacy/d` |
| Comment a line | `s/^pattern/#&/` | `s/^PermitRootLogin/#&/` |
| Uncomment a line | `s/^#\(pattern\)/\1/` | `s/^#\(PermitRootLogin\)/\1/` |

The `c\` (change) command replaces the entire matched line — useful as an alternative to `s///`:

```sh
# Replace the whole line matching the pattern
virt-edit -a "$image" /etc/ssh/sshd_config -e '/^#*PermitRootLogin/c\PermitRootLogin yes'
```

### Using a sed Script File

For complex edits, store expressions in a file and apply with `-f`:

```sh
# Create a sed script
cat > ssh_changes.sed << 'EOF'
s/^#*PermitRootLogin.*/PermitRootLogin yes/
s/^#*PasswordAuthentication.*/PasswordAuthentication yes/
s/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/
s/^#*UseDNS.*/UseDNS no/
EOF

# Apply the script
virt-edit -a "$image" /etc/ssh/sshd_config -f ssh_changes.sed

# Clean up
rm ssh_changes.sed
```

### Append vs Replace: When to Use Each

| Method | Use When |
|--------|----------|
| Replace (`s///`) | Modifying existing configurations (most cases) |
| Append (`$a\`) | Building new configs from scratch, setting guaranteed absent |

## SSH Configuration Recipes

### Development Environment (Permissive)

```sh
#!/bin/bash
image="dev-vm.qcow2"

virt-edit -a "$image" /etc/ssh/sshd_config \
  -e 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' \
  -e 's/^#*ChallengeResponseAuthentication.*/ChallengeResponseAuthentication yes/' \
  -e 's/^#*PermitRootLogin.*/PermitRootLogin yes/' \
  -e 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/' \
  -e 's/^#*X11Forwarding.*/X11Forwarding yes/' \
  -e 's/^#*PermitEmptyPasswords.*/PermitEmptyPasswords yes/' \
  -e 's/^#*UseDNS.*/UseDNS no/'
```

### Production Environment (Hardened)

```sh
#!/bin/bash
image="prod-vm.qcow2"

virt-edit -a "$image" /etc/ssh/sshd_config \
  -e 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' \
  -e 's/^#*ChallengeResponseAuthentication.*/ChallengeResponseAuthentication no/' \
  -e 's/^#*PermitRootLogin.*/PermitRootLogin no/' \
  -e 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/' \
  -e 's/^#*X11Forwarding.*/X11Forwarding no/' \
  -e 's/^#*MaxAuthTries.*/MaxAuthTries 3/' \
  -e 's/^#*PermitEmptyPasswords.*/PermitEmptyPasswords no/'
```

### Verify SSH Configuration Changes

```sh
# Before: check current state
virt-cat -a "$image" /etc/ssh/sshd_config | grep -E "^#*(Password|Challenge|Permit|Pubkey|X11|MaxAuth|UseDNS)"

# After: confirm changes applied
virt-cat -a "$image" /etc/ssh/sshd_config | grep -E "^(Password|Challenge|Permit|Pubkey|X11|MaxAuth|UseDNS)"

# Check for duplicates
virt-cat -a "$image" /etc/ssh/sshd_config | grep -c "PasswordAuthentication"

# View all active (uncommented) settings
virt-cat -a "$image" /etc/ssh/sshd_config | grep -v '^#' | grep -v '^$'

# Test SSH config syntax inside the image
virt-customize -a "$image" --run-command 'sshd -t'
```

### Backup and Restore sshd_config

```sh
# Backup before editing
virt-copy-out -a "$image" /etc/ssh/sshd_config ./

# Make changes
virt-edit -a "$image" /etc/ssh/sshd_config \
  -e 's/^#*PermitRootLogin.*/PermitRootLogin yes/' \
  -e 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/'

# Restore if something went wrong
virt-copy-in -a "$image" sshd_config /etc/ssh/
```

### Cloud-Init Alternative

When cloud-init is unavailable or impractical, modify the image directly:

```sh
virt-edit -a "ubuntu-22.04.qcow2" /etc/ssh/sshd_config \
  -e 's/^#*PermitRootLogin.*/PermitRootLogin yes/' \
  -e 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/'

virt-customize -a "ubuntu-22.04.qcow2" \
  --root-password password:changeme \
  --selinux-relabel
```
