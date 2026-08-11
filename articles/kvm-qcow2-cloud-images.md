# Using Red Hat and Cloud qcow2 Images with KVM

Red Hat, CentOS, Ubuntu, and other distributions provide pre-built qcow2 cloud images that can be deployed as KVM guests in seconds — no installer needed. This guide covers downloading, customizing, and deploying these images.

## Where to Download Cloud Images

| Distribution | URL |
|---|---|
| RHEL 9 | [Red Hat Customer Portal](https://access.redhat.com/downloads/content/rhel) → "KVM Guest Image" |
| RHEL 8 | [Red Hat Customer Portal](https://access.redhat.com/downloads/content/rhel) → "KVM Guest Image" |
| CentOS Stream 9 | https://cloud.centos.org/centos/9-stream/x86_64/images/ |
| CentOS Stream 8 | https://cloud.centos.org/centos/8-stream/x86_64/images/ |
| Rocky Linux 9 | https://download.rockylinux.org/pub/rocky/9/images/x86_64/ |
| AlmaLinux 9 | https://repo.almalinux.org/almalinux/9/cloud/x86_64/images/ |
| Ubuntu 22.04 | https://cloud-images.ubuntu.com/jammy/current/ |
| Ubuntu 24.04 | https://cloud-images.ubuntu.com/noble/current/ |
| Debian 12 | https://cloud.debian.org/images/cloud/bookworm/latest/ |
| Fedora 42 | https://download.fedoraproject.org/pub/fedora/linux/releases/42/Cloud/x86_64/images/ |

### Download Example

```bash
# RHEL 9 KVM Guest Image (requires Red Hat subscription)
# Download from the Customer Portal manually

# CentOS Stream 9
curl -O https://cloud.centos.org/centos/9-stream/x86_64/images/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2

# Rocky Linux 9
curl -O https://download.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-GenericCloud-Base.latest.x86_64.qcow2

# Ubuntu 22.04
curl -O https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Store in libvirt images directory
mv *.qcow2 /var/lib/libvirt/images/
mv *.img /var/lib/libvirt/images/
```

## Image Characteristics

Cloud images differ from regular installations:

- **No root password set** — login is only possible via cloud-init or SSH key
- **cloud-init is pre-installed** — configures hostname, users, SSH keys on first boot
- **Minimal packages** — smaller footprint, add what you need
- **growpart/cloud-utils** — disk auto-expands on first boot
- **DHCP enabled by default** — gets IP from the network
- **Serial console enabled** — works with `virsh console`

## Method 1: cloud-init (Recommended)

The standard way to customize cloud images on first boot.

### Create cloud-init Configuration

```bash
mkdir -p /var/lib/libvirt/images/cloud-init
```

**meta-data** (instance identity):

```bash
cat > /var/lib/libvirt/images/cloud-init/meta-data << EOF
instance-id: vm01
local-hostname: vm01.homelab.local
EOF
```

**user-data** (users, packages, commands):

```bash
cat > /var/lib/libvirt/images/cloud-init/user-data << EOF
#cloud-config
hostname: vm01
fqdn: vm01.homelab.local
manage_etc_hosts: true

# Set root password
chpasswd:
  list: |
    root:MySecurePassword123
  expire: false

# Create a user
users:
  - name: admin
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: wheel
    shell: /bin/bash
    lock_passwd: false
    passwd: \$6\$rounds=4096\$randomsalt\$hashedpasswordhere
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... your-public-key-here

# Enable password SSH authentication
ssh_pwauth: true

# Install packages
packages:
  - vim
  - bash-completion
  - chrony
  - qemu-guest-agent

# Run commands
runcmd:
  - systemctl enable --now qemu-guest-agent
  - systemctl enable --now chronyd

# Grow root filesystem to fill disk
growpart:
  mode: auto
  devices: ['/']
  ignore_growroot_disabled: false
EOF
```

**network-config** (static IP):

```bash
cat > /var/lib/libvirt/images/cloud-init/network-config << EOF
version: 2
ethernets:
  eth0:
    dhcp4: false
    addresses:
      - 192.168.50.100/24
    gateway4: 192.168.50.1
    nameservers:
      addresses:
        - 192.168.50.10
        - 8.8.8.8
      search:
        - homelab.local
EOF
```

### Generate cloud-init ISO

```bash
genisoimage -output /var/lib/libvirt/images/vm01-cidata.iso \
  -volid cidata -joliet -rock \
  /var/lib/libvirt/images/cloud-init/meta-data \
  /var/lib/libvirt/images/cloud-init/user-data \
  /var/lib/libvirt/images/cloud-init/network-config

# Or using cloud-localds (from cloud-image-utils)
cloud-localds /var/lib/libvirt/images/vm01-cidata.iso \
  /var/lib/libvirt/images/cloud-init/user-data \
  --network-config /var/lib/libvirt/images/cloud-init/network-config
```

### Create VM Disk (Copy-on-Write)

```bash
# Create a disk backed by the cloud image (saves space)
qemu-img create -f qcow2 \
  -b /var/lib/libvirt/images/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2 \
  -F qcow2 \
  /var/lib/libvirt/images/vm01.qcow2 30G
```

Or copy and resize:

```bash
# Full copy (independent, larger)
cp /var/lib/libvirt/images/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2 \
   /var/lib/libvirt/images/vm01.qcow2

# Resize to 30 GB
qemu-img resize /var/lib/libvirt/images/vm01.qcow2 30G
```

### Deploy the VM

```bash
virt-install \
  --name vm01 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/vm01.qcow2 \
  --disk path=/var/lib/libvirt/images/vm01-cidata.iso,device=cdrom \
  --os-variant centos-stream9 \
  --network bridge=br0 \
  --graphics none \
  --console pty,target_type=serial \
  --import \
  --noautoconsole
```

### Connect

```bash
# Wait ~30 seconds for cloud-init to complete
virsh console vm01
# or
ssh admin@192.168.50.100
```

## Method 2: virt-customize (Modify Image Directly)

`virt-customize` from the `libguestfs-tools` package modifies the image offline — no cloud-init needed.

### Install libguestfs-tools

```bash
dnf install -y libguestfs-tools    # RHEL 8+
yum install -y libguestfs-tools    # RHEL 7
apt install -y libguestfs-tools    # Ubuntu
```

### Set Root Password

```bash
# Copy the base image first
cp /var/lib/libvirt/images/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2 \
   /var/lib/libvirt/images/vm01.qcow2

# Set root password
virt-customize -a /var/lib/libvirt/images/vm01.qcow2 \
  --root-password password:MySecurePassword123
```

### Full Customization

```bash
virt-customize -a /var/lib/libvirt/images/vm01.qcow2 \
  --root-password password:MySecurePassword123 \
  --hostname vm01.homelab.local \
  --install vim,bash-completion,chrony,qemu-guest-agent \
  --run-command 'systemctl enable chronyd' \
  --run-command 'systemctl enable qemu-guest-agent' \
  --ssh-inject root:file:/root/.ssh/id_ed25519.pub \
  --selinux-relabel \
  --timezone Europe/Bucharest
```

### Available virt-customize Operations

| Option | Description |
|--------|-------------|
| `--root-password password:PASS` | Set root password (plaintext) |
| `--root-password file:/path` | Set root password from file |
| `--password USER:password:PASS` | Set user password |
| `--hostname NAME` | Set hostname |
| `--install PKG1,PKG2` | Install packages |
| `--uninstall PKG1,PKG2` | Remove packages |
| `--run-command 'CMD'` | Run a command inside the image |
| `--run /path/to/script.sh` | Run a script inside the image |
| `--ssh-inject USER:file:KEY` | Inject SSH public key for user |
| `--copy-in /host/file:/guest/path` | Copy file into the image |
| `--write /path:CONTENT` | Write a file inside the image |
| `--append-line /path:LINE` | Append a line to a file |
| `--timezone TZ` | Set timezone |
| `--selinux-relabel` | Relabel SELinux contexts (important for RHEL!) |
| `--firstboot /path/script` | Run script on first boot |
| `--truncate /path` | Empty a file (e.g., logs) |
| `--delete /path` | Delete a file |

### Deploy After Customization

```bash
# Resize disk
qemu-img resize /var/lib/libvirt/images/vm01.qcow2 30G

# Create VM
virt-install \
  --name vm01 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/vm01.qcow2 \
  --os-variant centos-stream9 \
  --network bridge=br0 \
  --graphics none \
  --console pty,target_type=serial \
  --import \
  --noautoconsole
```

## Method 3: virt-sysprep (Prepare Image as Template)

`virt-sysprep` resets a VM image for cloning — removes SSH keys, machine ID, logs, etc.

```bash
# Prepare an image as a reusable template
virt-sysprep -a /var/lib/libvirt/images/template.qcow2

# Sysprep with specific operations
virt-sysprep -a /var/lib/libvirt/images/template.qcow2 \
  --operations defaults,-ssh-userdir \
  --hostname template.homelab.local
```

Then clone from the template:

```bash
cp /var/lib/libvirt/images/template.qcow2 /var/lib/libvirt/images/vm02.qcow2
virt-customize -a /var/lib/libvirt/images/vm02.qcow2 \
  --hostname vm02.homelab.local \
  --root-password password:NewPassword
```

## Method 4: guestfish (Interactive Shell)

`guestfish` gives you a shell inside the image for manual changes:

```bash
guestfish -a /var/lib/libvirt/images/vm01.qcow2 -i

# Inside guestfish:
><fs> cat /etc/hostname
><fs> write /etc/hostname "vm01.homelab.local\n"
><fs> cat /etc/shadow
><fs> edit /etc/sysconfig/network-scripts/ifcfg-eth0
><fs> exit
```

## Changing Passwords

### Method 1: virt-customize (Offline)

```bash
# Set root password
virt-customize -a /var/lib/libvirt/images/vm01.qcow2 \
  --root-password password:NewPassword123

# Set a user password
virt-customize -a /var/lib/libvirt/images/vm01.qcow2 \
  --password admin:password:UserPassword123

# Using a pre-computed hash (write directly to shadow via run-command)
HASH=$(openssl passwd -6 'MyPassword')
virt-customize -a /var/lib/libvirt/images/vm01.qcow2 \
  --run-command "usermod -p '$HASH' root"

# Remove cloud-init (useful when deploying without cloud-init datasource)
virt-customize -a /var/lib/libvirt/images/vm01.qcow2 \
  --uninstall cloud-init
```

### Method 2: cloud-init (First Boot)

```yaml
#cloud-config
chpasswd:
  list: |
    root:NewPassword123
    admin:UserPassword456
  expire: false
```

### Method 3: guestfish (Manual)

```bash
# Generate password hash
openssl passwd -6 'MyPassword'
# $6$rounds=...hash...

# Edit shadow file
guestfish -a /var/lib/libvirt/images/vm01.qcow2 -i
><fs> download /etc/shadow /tmp/shadow
><fs> exit

# Edit /tmp/shadow, replace root's hash
# Then upload back:
guestfish -a /var/lib/libvirt/images/vm01.qcow2 -i
><fs> upload /tmp/shadow /etc/shadow
><fs> exit
```

### Method 4: virt-edit (Edit Files Directly)

```bash
# Edit a file inside the image
virt-edit -a /var/lib/libvirt/images/vm01.qcow2 /etc/hostname

# Non-interactive edit with expression
virt-edit -a /var/lib/libvirt/images/vm01.qcow2 /etc/hostname \
  -e 's/localhost.localdomain/vm01.homelab.local/'
```

## Injecting SSH Keys

### With virt-customize

```bash
# Inject key for root
virt-customize -a /var/lib/libvirt/images/vm01.qcow2 \
  --ssh-inject root:file:/root/.ssh/id_ed25519.pub

# Inject key for a specific user
virt-customize -a /var/lib/libvirt/images/vm01.qcow2 \
  --ssh-inject admin:file:/home/admin/.ssh/id_ed25519.pub
```

### With cloud-init

```yaml
#cloud-config
users:
  - name: admin
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... key1
      - ssh-rsa AAAA... key2
```

## Setting Static IP

### With cloud-init network-config

```yaml
version: 2
ethernets:
  eth0:
    dhcp4: false
    addresses:
      - 192.168.50.100/24
    gateway4: 192.168.50.1
    nameservers:
      addresses: [192.168.50.10, 8.8.8.8]
      search: [homelab.local]
```

### With virt-customize (RHEL/CentOS)

```bash
virt-customize -a /var/lib/libvirt/images/vm01.qcow2 \
  --run-command 'nmcli con mod "System eth0" ipv4.method manual ipv4.addresses 192.168.50.100/24 ipv4.gateway 192.168.50.1 ipv4.dns "192.168.50.10 8.8.8.8"'
```

## Resizing and Shrinking Images

### Grow a Disk Image

```bash
# Resize the qcow2 file (offline)
qemu-img resize /var/lib/libvirt/images/vm01.qcow2 +10G

# The guest filesystem still needs expanding inside the VM:
# For LVM: pvresize, lvresize, resize2fs/xfs_growfs
# For cloud images: growpart handles this automatically on boot
```

### Resize with virt-resize (Offline, Safe)

`virt-resize` copies the image to a new file, expanding partitions in the process:

```bash
# Show current partitions
virt-filesystems --long --all -a /var/lib/libvirt/images/vm01.qcow2

# Create a larger target image
truncate -r /var/lib/libvirt/images/vm01.qcow2 /var/lib/libvirt/images/vm01-new.qcow2
truncate -s +10G /var/lib/libvirt/images/vm01-new.qcow2

# Resize: expand /dev/sda2 to fill available space
virt-resize --expand /dev/sda2 \
  /var/lib/libvirt/images/vm01.qcow2 \
  /var/lib/libvirt/images/vm01-new.qcow2

# Or resize /dev/sda1 by a fixed amount and expand /dev/sda2
virt-resize --resize /dev/sda1=+500M --expand /dev/sda2 \
  /var/lib/libvirt/images/vm01.qcow2 \
  /var/lib/libvirt/images/vm01-new.qcow2

# Replace the old image
mv /var/lib/libvirt/images/vm01-new.qcow2 /var/lib/libvirt/images/vm01.qcow2
```

### Shrink an Image (Sparsify)

Remove unused space from a qcow2 image to reduce file size on the host:

```bash
# Sparsify (reclaim unused blocks, no data loss)
virt-sparsify /var/lib/libvirt/images/vm01.qcow2 /var/lib/libvirt/images/vm01-sparse.qcow2

# In-place sparsify (overwrites original)
virt-sparsify --in-place /var/lib/libvirt/images/vm01.qcow2

# Compress while converting
qemu-img convert -O qcow2 -c /var/lib/libvirt/images/vm01.qcow2 /var/lib/libvirt/images/vm01-compressed.qcow2
```

## Inspecting Images

```bash
# Show image info (size, format, backing file)
qemu-img info /var/lib/libvirt/images/vm01.qcow2

# List filesystems in the image
virt-filesystems -a /var/lib/libvirt/images/vm01.qcow2 --long --all

# Show OS info
virt-inspector -a /var/lib/libvirt/images/vm01.qcow2

# List files
virt-ls -a /var/lib/libvirt/images/vm01.qcow2 /etc/

# Cat a file
virt-cat -a /var/lib/libvirt/images/vm01.qcow2 /etc/hostname

# Show disk usage
virt-df -a /var/lib/libvirt/images/vm01.qcow2
```

## Complete Deployment Script

```bash
#!/bin/bash
# Deploy a VM from a cloud image
# Usage: ./deploy-vm.sh <vm-name> <ip-address>

VM_NAME="${1:?Usage: $0 <vm-name> <ip-address>}"
VM_IP="${2:?Usage: $0 <vm-name> <ip-address>}"
BASE_IMAGE="/var/lib/libvirt/images/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2"
IMAGES_DIR="/var/lib/libvirt/images"
SSH_KEY="$(cat /root/.ssh/id_ed25519.pub)"

# Create disk
qemu-img create -f qcow2 -b "$BASE_IMAGE" -F qcow2 \
  "${IMAGES_DIR}/${VM_NAME}.qcow2" 30G

# Customize
virt-customize -a "${IMAGES_DIR}/${VM_NAME}.qcow2" \
  --hostname "${VM_NAME}.homelab.local" \
  --root-password password:changeme \
  --ssh-inject "root:string:${SSH_KEY}" \
  --install qemu-guest-agent \
  --run-command 'systemctl enable qemu-guest-agent' \
  --selinux-relabel

# Create cloud-init for network (static IP)
cat > /tmp/meta-data << EOF
instance-id: ${VM_NAME}
local-hostname: ${VM_NAME}.homelab.local
EOF

cat > /tmp/network-config << EOF
version: 2
ethernets:
  eth0:
    dhcp4: false
    addresses: [${VM_IP}/24]
    gateway4: 192.168.50.1
    nameservers:
      addresses: [192.168.50.10]
      search: [homelab.local]
EOF

cat > /tmp/user-data << EOF
#cloud-config
EOF

genisoimage -output "${IMAGES_DIR}/${VM_NAME}-cidata.iso" \
  -volid cidata -joliet -rock \
  /tmp/meta-data /tmp/user-data /tmp/network-config

# Create VM
virt-install \
  --name "${VM_NAME}" \
  --ram 2048 \
  --vcpus 2 \
  --disk path="${IMAGES_DIR}/${VM_NAME}.qcow2" \
  --disk path="${IMAGES_DIR}/${VM_NAME}-cidata.iso",device=cdrom \
  --os-variant centos-stream9 \
  --network bridge=br0 \
  --graphics none \
  --import \
  --noautoconsole

echo "VM ${VM_NAME} created. SSH: ssh root@${VM_IP}"
```

## Quick Reference

```bash
# Download image
curl -O https://cloud.centos.org/centos/9-stream/x86_64/images/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2

# Set root password (offline)
virt-customize -a image.qcow2 --root-password password:MyPassword

# Inject SSH key
virt-customize -a image.qcow2 --ssh-inject root:file:~/.ssh/id_ed25519.pub

# Set hostname
virt-customize -a image.qcow2 --hostname vm01.homelab.local

# Install packages
virt-customize -a image.qcow2 --install vim,chrony --selinux-relabel

# Resize disk
qemu-img resize image.qcow2 30G

# Create backing disk (CoW)
qemu-img create -f qcow2 -b base.qcow2 -F qcow2 vm.qcow2 30G

# Deploy (import, no installer)
virt-install --name vm --ram 2048 --vcpus 2 --disk vm.qcow2 --os-variant centos-stream9 --import --noautoconsole

# Inspect image
qemu-img info image.qcow2
virt-cat -a image.qcow2 /etc/hostname
```
