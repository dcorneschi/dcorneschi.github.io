# Installing KVM Guests

This guide covers creating KVM virtual machines using `virt-install` — from ISO images, kickstart files, network installs, and cloud images.

## Prerequisites

```bash
# Verify KVM is available
lsmod | grep kvm
virsh list --all

# Required packages
dnf install -y qemu-kvm libvirt virt-install libguestfs-tools   # RHEL 8+
yum install -y qemu-kvm libvirt virt-install libguestfs-tools   # RHEL 7

# Ensure libvirtd is running
systemctl enable --now libvirtd
```

## Method 1: Install from ISO (Interactive)

### Download or Copy the ISO

```bash
# Place ISO in libvirt images directory
cp rhel-9.3-x86_64-dvd.iso /var/lib/libvirt/images/
```

### Create VM with virt-install

```bash
virt-install \
  --name rhel9-vm01 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/rhel9-vm01.qcow2,size=20,format=qcow2 \
  --os-variant rhel9.3 \
  --network bridge=br0 \
  --graphics vnc,listen=0.0.0.0 \
  --cdrom /var/lib/libvirt/images/rhel-9.3-x86_64-dvd.iso \
  --boot cdrom
```

Then connect via VNC or virt-manager to complete the graphical installer.

### With Console Access (No GUI)

```bash
virt-install \
  --name rhel9-vm01 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/rhel9-vm01.qcow2,size=20,format=qcow2 \
  --os-variant rhel9.3 \
  --network bridge=br0 \
  --graphics none \
  --console pty,target_type=serial \
  --extra-args "console=ttyS0,115200n8" \
  --location /var/lib/libvirt/images/rhel-9.3-x86_64-dvd.iso
```

> **Note:** `--extra-args` requires `--location` instead of `--cdrom`.

## Method 2: Install from Kickstart (Automated)

### Create a Kickstart File

```bash
vi /var/lib/libvirt/images/ks-rhel9.cfg
```

```kickstart
#version=RHEL9
ignoredisk --only-use=vda
autopart --type=lvm
clearpart --all --initlabel

# System language and keyboard
lang en_US.UTF-8
keyboard us
timezone Europe/Bucharest --utc

# Network
network --bootproto=static --device=eth0 --ip=192.168.50.100 --netmask=255.255.255.0 --gateway=192.168.50.1 --nameserver=192.168.50.10 --hostname=vm01.homelab.local --activate
network --bootproto=dhcp --device=eth0 --activate

# Root password (use openssl passwd -6 to generate)
rootpw --iscrypted $6$rounds=4096$randomsalt$hashedpasswordhere

# User
user --name=admin --groups=wheel --iscrypted --password=$6$rounds=4096$randomsalt$hashedpasswordhere

# System services
firewall --enabled --service=ssh
selinux --enforcing
services --enabled=sshd,chronyd

# Packages
%packages
@^minimal-environment
chrony
vim-enhanced
bash-completion
%end

# Post-install script
%post
echo "192.168.50.10 foreman.homelab.local foreman" >> /etc/hosts
%end

# Reboot after installation
reboot
```

### Install with Local Kickstart File

```bash
virt-install \
  --name rhel9-vm01 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/rhel9-vm01.qcow2,size=20,format=qcow2 \
  --os-variant rhel9.3 \
  --network bridge=br0 \
  --graphics none \
  --console pty,target_type=serial \
  --location /var/lib/libvirt/images/rhel-9.3-x86_64-dvd.iso \
  --initrd-inject=/var/lib/libvirt/images/ks-rhel9.cfg \
  --extra-args "inst.ks=file:/ks-rhel9.cfg console=ttyS0,115200n8"
```

### Install with HTTP Kickstart

```bash
virt-install \
  --name rhel9-vm01 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/rhel9-vm01.qcow2,size=20,format=qcow2 \
  --os-variant rhel9.3 \
  --network bridge=br0 \
  --graphics none \
  --console pty,target_type=serial \
  --location /var/lib/libvirt/images/rhel-9.3-x86_64-dvd.iso \
  --extra-args "inst.ks=http://192.168.50.10/ks/ks-rhel9.cfg console=ttyS0,115200n8"
```

### Install with NFS Kickstart

```bash
virt-install \
  --name rhel9-vm01 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/rhel9-vm01.qcow2,size=20,format=qcow2 \
  --os-variant rhel9.3 \
  --network bridge=br0 \
  --graphics none \
  --console pty,target_type=serial \
  --location nfs://192.168.50.10:/export/rhel9-iso \
  --extra-args "inst.ks=nfs://192.168.50.10:/export/ks/ks-rhel9.cfg console=ttyS0,115200n8"
```

## Method 3: Network Install (HTTP/FTP/NFS)

Install directly from a network repository without a local ISO:

### From HTTP

```bash
virt-install \
  --name rhel9-vm01 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/rhel9-vm01.qcow2,size=20,format=qcow2 \
  --os-variant rhel9.3 \
  --network bridge=br0 \
  --graphics none \
  --console pty,target_type=serial \
  --location http://192.168.50.10/rhel9/BaseOS/x86_64/os/ \
  --extra-args "console=ttyS0,115200n8"
```

### From FTP

```bash
virt-install \
  --name rhel9-vm01 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/rhel9-vm01.qcow2,size=20,format=qcow2 \
  --os-variant rhel9.3 \
  --network bridge=br0 \
  --graphics none \
  --console pty,target_type=serial \
  --location ftp://192.168.50.10/pub/rhel9/ \
  --extra-args "console=ttyS0,115200n8"
```

### From NFS

```bash
virt-install \
  --name rhel9-vm01 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/rhel9-vm01.qcow2,size=20,format=qcow2 \
  --os-variant rhel9.3 \
  --network bridge=br0 \
  --graphics none \
  --console pty,target_type=serial \
  --location nfs://192.168.50.10:/export/rhel9-iso \
  --extra-args "console=ttyS0,115200n8"
```

## Method 4: Cloud Image (qcow2 + cloud-init)

Use pre-built cloud images for fast deployment without running an installer.

### Download Cloud Image

```bash
# RHEL 9 (from Red Hat portal)
# CentOS Stream 9
curl -o /var/lib/libvirt/images/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2 \
  https://cloud.centos.org/centos/9-stream/x86_64/images/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2

# Ubuntu 22.04
curl -o /var/lib/libvirt/images/jammy-server-cloudimg-amd64.img \
  https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
```

### Create a Backing Disk (Copy-on-Write)

```bash
qemu-img create -f qcow2 -b /var/lib/libvirt/images/CentOS-Stream-GenericCloud-9-latest.x86_64.qcow2 \
  -F qcow2 /var/lib/libvirt/images/vm01.qcow2 20G
```

### Create cloud-init ISO

```bash
# Create meta-data
cat > /tmp/meta-data << EOF
instance-id: vm01
local-hostname: vm01.homelab.local
EOF

# Create user-data
cat > /tmp/user-data << EOF
#cloud-config
hostname: vm01
fqdn: vm01.homelab.local
manage_etc_hosts: true

users:
  - name: admin
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: wheel
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... your-key-here

# Set root password
chpasswd:
  list: |
    root:changeme
  expire: false

# Install packages
packages:
  - vim
  - bash-completion
  - chrony

# Run commands on first boot
runcmd:
  - systemctl enable --now chronyd
EOF

# Generate the cloud-init ISO
genisoimage -output /var/lib/libvirt/images/vm01-cidata.iso \
  -volid cidata -joliet -rock /tmp/meta-data /tmp/user-data
```

### Create the VM

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

The `--import` flag skips the installer and boots directly from the disk image.

## Method 5: PXE Boot

```bash
virt-install \
  --name rhel9-vm01 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/rhel9-vm01.qcow2,size=20,format=qcow2 \
  --os-variant rhel9.3 \
  --network bridge=br0 \
  --graphics vnc,listen=0.0.0.0 \
  --pxe \
  --boot network
```

Requires a PXE/TFTP/DHCP server configured on the network.

## Method 6: Import Existing Disk

```bash
# Import a pre-existing disk image
virt-install \
  --name imported-vm \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/existing-disk.qcow2 \
  --os-variant rhel9.3 \
  --network bridge=br0 \
  --graphics none \
  --import
```

## virt-install Options Reference

### Compute

| Option | Description |
|--------|-------------|
| `--name` | VM name (must be unique) |
| `--ram` / `--memory` | RAM in MB |
| `--vcpus` | Number of virtual CPUs |
| `--cpu host` | Pass through host CPU model |

### Storage

| Option | Description |
|--------|-------------|
| `--disk path=,size=,format=` | Create or attach a disk |
| `--disk none` | No disk (diskless boot) |
| `--filesystem` | Share host directory with guest |

Disk formats: `qcow2` (default, supports snapshots), `raw` (better performance)

Short form (auto-creates disk in default pool):

```bash
--disk size=20           # Creates 20 GB qcow2 in /var/lib/libvirt/images/
--disk size=20,format=raw
```

### Network

| Option | Description |
|--------|-------------|
| `--network bridge=br0` | Attach to a bridge |
| `--network network=default` | Use libvirt NAT network |
| `--network none` | No network |
| `--network type=direct,source=eth0` | macvtap (direct attach) |

### Installation Source

| Option | Description |
|--------|-------------|
| `--cdrom /path/to/iso` | Boot from ISO (no extra-args) |
| `--location /path/or/url` | Install tree (supports extra-args) |
| `--pxe` | PXE network boot |
| `--import` | Skip installer, boot from disk |
| `--boot cdrom` | Boot from CD |
| `--boot hd` | Boot from hard disk |
| `--boot network` | Boot from network (PXE) |

### Display

| Option | Description |
|--------|-------------|
| `--graphics vnc,listen=0.0.0.0` | VNC console (connect with viewer) |
| `--graphics spice` | SPICE console |
| `--graphics none` | No graphics (serial console only) |
| `--console pty,target_type=serial` | Serial console |
| `--noautoconsole` | Don't attach to console after creation |

### Kickstart / Automation

| Option | Description |
|--------|-------------|
| `--extra-args "..."` | Kernel command-line arguments (requires --location) |
| `--initrd-inject=/path` | Inject file into initrd |

### OS Variant

```bash
# List available OS variants
virt-install --osinfo list
# or (older versions)
osinfo-query os
```

Common variants:

| Variant | OS |
|---------|-----|
| `rhel9.3` | RHEL 9.3 |
| `rhel8.9` | RHEL 8.9 |
| `centos-stream9` | CentOS Stream 9 |
| `ubuntu22.04` | Ubuntu 22.04 |
| `debian12` | Debian 12 |
| `fedora39` | Fedora 39 |
| `win10` | Windows 10 |
| `win2k22` | Windows Server 2022 |

## Ubuntu Guest Installation

### From ISO

```bash
virt-install \
  --name ubuntu22-vm01 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/ubuntu22-vm01.qcow2,size=20,format=qcow2 \
  --os-variant ubuntu22.04 \
  --network bridge=br0 \
  --graphics vnc,listen=0.0.0.0 \
  --cdrom /var/lib/libvirt/images/ubuntu-22.04-live-server-amd64.iso
```

### With Autoinstall (Ubuntu 20.04+)

```bash
virt-install \
  --name ubuntu22-vm01 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/ubuntu22-vm01.qcow2,size=20,format=qcow2 \
  --os-variant ubuntu22.04 \
  --network bridge=br0 \
  --graphics none \
  --console pty,target_type=serial \
  --location /var/lib/libvirt/images/ubuntu-22.04-live-server-amd64.iso,kernel=casper/vmlinuz,initrd=casper/initrd \
  --extra-args "autoinstall ds=nocloud-net;s=http://192.168.50.10/autoinstall/ console=ttyS0,115200n8"
```

## Post-Installation

### Connect via VNC (Remote Access)

If the VM was created with `--graphics vnc`:

```bash
# Find the VNC port assigned to the VM
virsh dumpxml vm01 | grep vnc
# <graphics type='vnc' port='5900' autoport='yes' listen='127.0.0.1'>
```

**SSH tunnel for VNC (from Windows/Mac/Linux workstation):**

```bash
# Forward VNC port to your local machine
ssh -L 5900:127.0.0.1:5900 root@kvm-host-ip
```

Then connect a VNC viewer (RealVNC, TigerVNC) to `127.0.0.1:5900`.

### Connect to Console

```bash
# Attach to serial console
virsh console rhel9-vm01

# Detach: Ctrl+]
```

### Start/Stop VM

```bash
virsh start rhel9-vm01
virsh shutdown rhel9-vm01
virsh destroy rhel9-vm01       # force stop
virsh reboot rhel9-vm01
```

### Enable Autostart

```bash
virsh autostart rhel9-vm01
```

### View VM Info

```bash
virsh dominfo rhel9-vm01
virsh domblklist rhel9-vm01
virsh domiflist rhel9-vm01
```

## Cloning a VM

```bash
# Shut down the source VM first
virsh shutdown rhel9-vm01

# Clone
virt-clone \
  --original rhel9-vm01 \
  --name rhel9-vm02 \
  --auto-clone

# Start the clone
virsh start rhel9-vm02
```

## Scripting Multiple VMs

```bash
#!/bin/bash
# Create multiple VMs from a template
for i in 01 02 03; do
  virt-install \
    --name "vm${i}" \
    --ram 2048 \
    --vcpus 2 \
    --disk path="/var/lib/libvirt/images/vm${i}.qcow2,size=20,format=qcow2" \
    --os-variant rhel9.3 \
    --network bridge=br0 \
    --graphics none \
    --console pty,target_type=serial \
    --location /var/lib/libvirt/images/rhel-9.3-x86_64-dvd.iso \
    --initrd-inject=/var/lib/libvirt/images/ks-rhel9.cfg \
    --extra-args "inst.ks=file:/ks-rhel9.cfg console=ttyS0,115200n8" \
    --noautoconsole
done
```

## Quick Reference

```bash
# From ISO (graphical)
virt-install --name vm --ram 2048 --vcpus 2 --disk size=20 --cdrom /path/to.iso --os-variant rhel9.3

# From ISO (console)
virt-install --name vm --ram 2048 --vcpus 2 --disk size=20 --location /path/to.iso --os-variant rhel9.3 --graphics none --extra-args "console=ttyS0"

# With kickstart
virt-install --name vm --ram 2048 --vcpus 2 --disk size=20 --location /path/to.iso --os-variant rhel9.3 --graphics none --initrd-inject=/path/ks.cfg --extra-args "inst.ks=file:/ks.cfg console=ttyS0"

# Cloud image (import)
virt-install --name vm --ram 2048 --vcpus 2 --disk /path/to/image.qcow2 --os-variant centos-stream9 --import --noautoconsole

# PXE
virt-install --name vm --ram 2048 --vcpus 2 --disk size=20 --os-variant rhel9.3 --network bridge=br0 --pxe

# List OS variants
virt-install --osinfo list
```
