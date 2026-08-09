# Installing KVM

KVM (Kernel-based Virtual Machine) turns Linux into a type-1 hypervisor. This guide covers installation on RHEL 7–10 and Ubuntu 22.04/24.04.

## Prerequisites

### Verify CPU Virtualization Support

```bash
# Check for Intel VT-x or AMD-V
grep -Ec '(vmx|svm)' /proc/cpuinfo

# If the result is > 0, hardware virtualization is supported
# vmx = Intel VT-x
# svm = AMD-V

# Alternative: lscpu
lscpu | grep Virtualization
# Virtualization:        VT-x

# With color highlight
egrep '(vmx|svm)' --color=always /proc/cpuinfo
```

If the output is `0`, enable virtualization in BIOS/UEFI (Intel VT-x or AMD SVM).

### Check if KVM Module is Loaded

```bash
lsmod | grep kvm
# kvm_intel (Intel) or kvm_amd (AMD) should appear

# Load manually if needed
sudo modprobe kvm
sudo modprobe kvm_intel   # Intel
sudo modprobe kvm_amd     # AMD
```

### Verify with kvm-ok (Ubuntu)

```bash
sudo apt install cpu-checker
kvm-ok
# INFO: /dev/kvm exists
# KVM acceleration can be used
```

## Installation on RHEL / CentOS / Rocky / AlmaLinux

### RHEL 7

```bash
# Install KVM and management tools
sudo yum install -y qemu-kvm libvirt libvirt-python libguestfs-tools virt-install

# Optional: GUI tools
sudo yum install -y virt-manager

# Start and enable libvirtd
sudo systemctl start libvirtd
sudo systemctl enable libvirtd

# Verify
sudo systemctl status libvirtd
virsh list --all
```

### RHEL 8

```bash
# Install virtualization module
sudo dnf module install virt

# Install additional tools
sudo dnf install -y qemu-kvm libvirt virt-install virt-viewer

# Optional: GUI and cockpit
sudo dnf install -y virt-manager
sudo dnf install -y cockpit-machines

# Start and enable libvirtd
sudo systemctl start libvirtd
sudo systemctl enable libvirtd

# Verify
virt-host-validate
virsh list --all
```

### RHEL 9

```bash
# Install virtualization packages
sudo dnf install -y qemu-kvm libvirt virt-install virt-viewer

# Optional: GUI tools
sudo dnf install -y virt-manager

# Optional: Cockpit integration
sudo dnf install -y cockpit-machines

# Start and enable libvirtd
sudo systemctl start libvirtd
sudo systemctl enable libvirtd

# Verify
virt-host-validate
virsh list --all
```

### RHEL 10

```bash
# Install virtualization packages
sudo dnf install -y qemu-kvm libvirt virt-install virt-viewer

# Optional: Cockpit integration (virt-manager deprecated in RHEL 10)
sudo dnf install -y cockpit-machines

# Start and enable libvirtd
sudo systemctl start libvirtd
sudo systemctl enable libvirtd

# Verify
virt-host-validate
virsh list --all
```

> **Note:** Starting from RHEL 10, `virt-manager` is deprecated. Use Cockpit with `cockpit-machines` for GUI management.

### All RHEL Versions — Add User to libvirt Group

```bash
# Allow non-root user to manage VMs
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER

# Apply without logout
newgrp libvirt
```

## Installation on Ubuntu

### Ubuntu 22.04 (Jammy)

```bash
# Install KVM and management tools
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-viewer

# Optional: GUI manager
sudo apt install -y virt-manager

# Verify libvirtd is running
sudo systemctl status libvirtd

# Add user to required groups
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER

# Apply without logout
newgrp libvirt

# Verify installation
virsh list --all
sudo kvm-ok
```

### Ubuntu 24.04 (Noble)

```bash
# Install KVM and management tools
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-viewer

# Optional: GUI manager
sudo apt install -y virt-manager

# Optional: Cockpit integration
sudo apt install -y cockpit cockpit-machines

# Verify libvirtd is running
sudo systemctl status libvirtd

# Add user to required groups
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER

# Apply without logout
newgrp libvirt

# Verify
virsh list --all
virt-host-validate
```

## Post-Installation Verification

```bash
# Validate host configuration
sudo virt-host-validate

# Expected output:
# QEMU: Checking for hardware virtualization          : PASS
# QEMU: Checking if device /dev/kvm exists            : PASS
# QEMU: Checking if device /dev/kvm is accessible     : PASS
# QEMU: Checking if device /dev/vhost-net exists      : PASS
# QEMU: Checking for cgroup 'cpu' controller support  : PASS
# QEMU: Checking for cgroup 'memory' controller support: PASS

# Check libvirt version
virsh version

# Check QEMU version
qemu-system-x86_64 --version

# List default network
virsh net-list --all

# List default storage pool
virsh pool-list --all

# Check KVM kernel modules
lsmod | grep kvm
```

## Default Network (NAT)

KVM creates a `default` NAT network on installation:

```bash
# Check default network status
virsh net-list --all

# Start if not active
virsh net-start default

# Set to autostart
virsh net-autostart default

# View network details
virsh net-info default

# View network XML
virsh net-dumpxml default
```

Default network provides:
- NAT for VM internet access
- DHCP (192.168.122.0/24 by default)
- DNS forwarding via dnsmasq

## Bridge Networking (Direct Access)

For VMs to be on the same network as the host (reachable from LAN):

### Ubuntu (Netplan)

```yaml
# /etc/netplan/01-bridge.yaml
network:
  version: 2
  ethernets:
    enp3s0:
      dhcp4: false
  bridges:
    br0:
      interfaces: [enp3s0]
      dhcp4: true
      # or static:
      # addresses: [192.168.1.10/24]
      # routes:
      #   - to: default
      #     via: 192.168.1.1
      # nameservers:
      #   addresses: [8.8.8.8, 8.8.4.4]
```

```bash
sudo netplan apply
```

### RHEL 8/9/10 (nmcli)

```bash
# Create bridge
sudo nmcli connection add type bridge con-name br0 ifname br0

# Add physical interface as bridge slave
sudo nmcli connection add type ethernet con-name br0-slave ifname enp3s0 master br0

# Configure bridge IP (DHCP)
sudo nmcli connection modify br0 ipv4.method auto

# Or static IP
sudo nmcli connection modify br0 ipv4.addresses 192.168.1.10/24
sudo nmcli connection modify br0 ipv4.gateway 192.168.1.1
sudo nmcli connection modify br0 ipv4.dns "8.8.8.8 8.8.4.4"
sudo nmcli connection modify br0 ipv4.method manual

# Bring up
sudo nmcli connection up br0

# Verify
ip addr show br0
bridge link
```

### RHEL 7 (ifcfg files)

```bash
# /etc/sysconfig/network-scripts/ifcfg-br0
TYPE=Bridge
DEVICE=br0
ONBOOT=yes
BOOTPROTO=static
IPADDR=192.168.1.10
PREFIX=24
GATEWAY=192.168.1.1
DNS1=192.168.1.1

# /etc/sysconfig/network-scripts/ifcfg-enp3s0 (or ifcfg-eth0)
TYPE=Ethernet
DEVICE=enp3s0
ONBOOT=yes
BRIDGE=br0
```

```bash
# Disable NetworkManager (RHEL 7 — may conflict with bridge)
sudo systemctl stop NetworkManager
sudo systemctl disable NetworkManager

# Restart network service
sudo systemctl restart network
```

> **Note:** On RHEL 7, NetworkManager may conflict with manual bridge configuration. Disable it if using ifcfg files directly. On RHEL 8+, NetworkManager handles bridges natively via nmcli.

## Storage Pool

### Default Pool

```bash
# Default pool location: /var/lib/libvirt/images/
virsh pool-list --all

# Start if not active
virsh pool-start default
virsh pool-autostart default
```

### Create Custom Pool

```bash
# Create directory
sudo mkdir -p /data/vms

# Define pool
virsh pool-define-as mypool dir - - - - /data/vms

# Build, start, and autostart
virsh pool-build mypool
virsh pool-start mypool
virsh pool-autostart mypool

# Verify
virsh pool-list --all
virsh pool-info mypool
```

### Create Pool on LVM

```bash
# Create logical volume
sudo lvcreate -n lv_kvm -L 100G <vg-name>
sudo mkfs.xfs /dev/<vg-name>/lv_kvm

# Create mount point
sudo mkdir /kvm-pool

# Add to fstab
echo "/dev/mapper/<vg-name>-lv_kvm   /kvm-pool   xfs   defaults   0 0" | sudo tee -a /etc/fstab

# Mount
sudo mount /kvm-pool
```

### Delete and Recreate Default Pool

```bash
# List current pools
virsh pool-list

# Destroy (stop) and undefine the default pool
virsh pool-destroy default
virsh pool-undefine default

# Redefine pointing to new location
virsh pool-define-as --name default --type dir --target /kvm-pool
virsh pool-autostart default
virsh pool-start default

# Verify
virsh pool-list
```

## Create a Virtual Machine

### Using virt-install (CLI)

```bash
# Download ISO first, then:
virt-install \
    --name ubuntu-server \
    --ram 2048 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/ubuntu-server.qcow2,size=20 \
    --os-variant ubuntu22.04 \
    --network bridge=br0 \
    --graphics vnc,listen=0.0.0.0 \
    --console pty,target_type=serial \
    --cdrom /var/lib/libvirt/images/ubuntu-22.04-live-server-amd64.iso

# Minimal CentOS/Rocky VM
virt-install \
    --name rocky9 \
    --ram 2048 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/rocky9.qcow2,size=20 \
    --os-variant rocky9.0 \
    --network network=default \
    --graphics spice \
    --cdrom /var/lib/libvirt/images/Rocky-9-latest-x86_64-dvd.iso

# Headless VM (no graphics, serial console)
virt-install \
    --name headless-vm \
    --ram 1024 \
    --vcpus 1 \
    --disk path=/var/lib/libvirt/images/headless.qcow2,size=10 \
    --os-variant generic \
    --network network=default \
    --graphics none \
    --extra-args='console=ttyS0,115200n8 serial' \
    --location /var/lib/libvirt/images/Rocky-9-latest-x86_64-dvd.iso
```

### List Available OS Variants

```bash
# All variants
osinfo-query os

# Filter
osinfo-query os | grep -i ubuntu
osinfo-query os | grep -i rhel
osinfo-query os | grep -i rocky
```

## Basic virsh Commands

For a full virsh command reference, see [KVM / virsh Cheatsheet](articles/kvm-cheatsheet.md).

```bash
# List all VMs
virsh list --all

# Start / shutdown / force-stop
virsh start <vm-name>
virsh shutdown <vm-name>
virsh destroy <vm-name>

# Console access
virsh console <vm-name>
```

## Snapshots

For snapshot management, see [KVM / virsh Cheatsheet](articles/kvm-cheatsheet.md).

## Cloning

```bash
virt-clone --original <source-vm> --name <new-vm> --auto-clone
```

## Firewall Configuration

```bash
# Option 1: Open specific ports for VNC/Spice access
sudo firewall-cmd --permanent --add-service=libvirt
sudo firewall-cmd --permanent --add-port=5900-5999/tcp   # VNC
sudo firewall-cmd --permanent --add-port=5634/tcp        # Spice
sudo firewall-cmd --reload

# Option 2: Disable firewall entirely (lab/homelab only)
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

## SELinux Configuration

```bash
# Check current mode
getenforce

# Set permissive (temporary, for testing)
sudo setenforce 0

# Disable permanently (not recommended for production)
sudo sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
# Reboot required for full disable

# Better: keep SELinux enabled and set booleans
sudo setsebool -P virt_use_nfs 1
sudo setsebool -P virt_use_samba 1
```

> **Recommendation:** Keep SELinux in enforcing mode and use booleans. Disabling SELinux is only appropriate for isolated lab environments.

## Troubleshooting

### Permission Denied on /dev/kvm

```bash
# Check ownership
ls -la /dev/kvm

# Fix permissions
sudo chown root:kvm /dev/kvm
sudo chmod 660 /dev/kvm

# Ensure user is in kvm group
sudo usermod -aG kvm $USER
```

### libvirtd Fails to Start

```bash
# Check logs
journalctl -u libvirtd -f
sudo cat /var/log/libvirt/libvirtd.log

# Check SELinux (RHEL)
getenforce
sudo setsebool -P virt_use_nfs 1
```

### Default Network Not Starting

```bash
# Check if another process uses the same subnet
ip addr | grep 192.168.122

# Restart default network
virsh net-destroy default
virsh net-start default

# Check iptables/nftables conflicts
sudo iptables -L -n | grep virbr
```

### VM Cannot Access Internet

```bash
# Check IP forwarding
cat /proc/sys/net/ipv4/ip_forward
# Should be 1

# Enable if needed
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# Make permanent
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/99-kvm.conf
sudo sysctl -p /etc/sysctl.d/99-kvm.conf
```

### Nested Virtualization (VMs Inside VMs)

```bash
# Check if enabled
cat /sys/module/kvm_intel/parameters/nested   # Intel
cat /sys/module/kvm_amd/parameters/nested     # AMD

# Enable (Intel)
echo "options kvm_intel nested=1" | sudo tee /etc/modprobe.d/kvm.conf
sudo modprobe -r kvm_intel && sudo modprobe kvm_intel

# Enable (AMD)
echo "options kvm_amd nested=1" | sudo tee /etc/modprobe.d/kvm.conf
sudo modprobe -r kvm_amd && sudo modprobe kvm_amd

# Verify
cat /sys/module/kvm_intel/parameters/nested
# Y
```

## Package Summary

| Package | Purpose |
|---------|---------|
| `qemu-kvm` | KVM hypervisor and QEMU emulator |
| `libvirt` / `libvirt-daemon-system` | Virtualization management daemon |
| `libvirt-clients` | `virsh` CLI tool |
| `virt-install` / `virtinst` | CLI VM creation tool |
| `virt-viewer` | Graphical VM console viewer |
| `virt-manager` | GUI management tool |
| `bridge-utils` | Bridge networking utilities |
| `cockpit-machines` | Web-based VM management (Cockpit) |
| `libguestfs-tools` | VM image manipulation tools |
| `cpu-checker` | `kvm-ok` verification (Ubuntu) |
