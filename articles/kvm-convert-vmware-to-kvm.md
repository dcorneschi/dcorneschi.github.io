# Converting VMware VMs to KVM with virt-v2v

`virt-v2v` converts virtual machines — including disk images and metadata — from foreign hypervisors (VMware, Xen, Hyper-V) to KVM managed by libvirt. This guide covers converting Linux VMs from VMware vCenter to RHEL KVM.

## Overview

On successful conversion, `virt-v2v` creates:

- A qcow2 disk image in `/var/tmp/` (or specified output location)
- A libvirt domain XML file with the same name as the original VM
- The VM can then be started using `virsh` or `virt-manager`

## Prerequisites

- `virt-v2v` must be run on a RHEL 64-bit host (7, 8, 9, or 10)
- The source VM **must be stopped** before conversion
- Minimum network speed: 1 Gbps
- Disk space: enough to store the VM's disk image + 1 GB
- The source VM must have 100+ free inodes
- Linux VMs must use a filesystem supported by the RHEL kernel (btrfs is not supported)

### Free Space Requirements (Inside the VM)

| Filesystem | Minimum Free Space |
|---|---|
| Root filesystem (`/`) | 100 MB |
| `/boot` | 50 MB |
| Every other mountable filesystem | 10 MB |

## Installation

```bash
# RHEL 7
yum install -y virt-v2v

# RHEL 8 / 9 / 10
dnf install -y virt-v2v
```

## Converting from VMware vCenter

### Basic Conversion

```bash
# Stop the VM in VMware first!

# Convert from vCenter
virt-v2v -ic vpx://username@vcenter.example.com/Datacenter/esxi "guestvm1"
```

Replace:
- `username` — your vCenter username
- `vcenter.example.com` — your vCenter FQDN
- `Datacenter/esxi` — path to the datacenter/ESXi host
- `guestvm1` — name of the VM to convert

### With Self-Signed Certificate (Skip Verification)

```bash
virt-v2v -ic vpx://username@vcenter.example.com/Datacenter/esxi?no_verify=1 "guestvm1"
```

### With Password File (Non-Interactive)

```bash
# Create password file
echo "MyVCenterPassword" > /tmp/vcenter-pass
chmod 600 /tmp/vcenter-pass

# Convert using password file
virt-v2v -ic vpx://username@vcenter.example.com/Datacenter/esxi "guestvm1" \
  -ip /tmp/vcenter-pass

# Clean up
rm -f /tmp/vcenter-pass
```

### Special Characters in Username or Path

| Character | URI Escape |
|---|---|
| `\` (backslash, e.g., `DOMAIN\USER`) | `%5c` → `DOMAIN%5cUSER` |
| Space (e.g., `My Datacenter`) | `%20` → `My%20Datacenter` |

```bash
# Domain user with backslash
virt-v2v -ic vpx://DOMAIN%5cuser@vcenter.example.com/My%20Datacenter/esxi "guestvm1"
```

## Converting from VMware OVA File

If you exported the VM as an OVA:

### Export from VMware

1. Shut down the VM gracefully in VMware (ensure it's not in hibernation or fast startup mode)
2. Export the VM as an OVA file using the vSphere Client:
   - Right-click the VM → **Export** → **Export OVF Template**
   - Choose "Single file" (`.ova`) or "Folder of files" (`.ovf`)
3. Transfer the OVA to the conversion host:

```bash
scp guestvm1.ova root@kvm-host:/var/tmp/
```

> **Note:** `.ova` files are uncompressed `.tar` archives. You can inspect their content with `tar tf guestvm1.ova`.

### Convert the OVA

```bash
# Convert from OVA file (single .ova archive)
virt-v2v -i ova /var/tmp/guestvm1.ova -of qcow2

# Convert from OVF folder (exported as "Folder of files")
virt-v2v -i ova /path/to/ovf-folder/ -of qcow2
```

### For Windows VMs (Requires virtio-win)

```bash
# Install virtio-win drivers package
dnf install -y virt-v2v virtio-win

# Convert Windows OVA
virt-v2v -i ova guestvm1.ova -of qcow2
```

The `virtio-win` package provides Windows VirtIO drivers that are injected during conversion.

## Converting from VMware via SSH (ESXi Direct)

```bash
# Connect directly to ESXi host via SSH
virt-v2v -ic esx://root@esxi.example.com "guestvm1" -ip /tmp/esxi-pass
```

## Output Options

### Default (libvirt)

By default, `virt-v2v` creates a libvirt domain:

```bash
virt-v2v -ic vpx://user@vcenter.example.com/DC/esxi "guestvm1"

# VM is created in libvirt
virsh list --all
virsh start guestvm1
```

### Output to a Specific Directory

```bash
# Output to local directory (disk images + XML)
virt-v2v -ic vpx://user@vcenter.example.com/DC/esxi "guestvm1" \
  -o local -os /var/lib/libvirt/images/
```

### Output to libvirt with Specific Pool

```bash
virt-v2v -ic vpx://user@vcenter.example.com/DC/esxi "guestvm1" \
  -o libvirt -os default
```

### Output to OpenStack (Glance)

```bash
virt-v2v -ic vpx://user@vcenter.example.com/DC/esxi "guestvm1" \
  -o glance
```

### Output Format

```bash
# Output as qcow2 (default, supports snapshots)
virt-v2v -ic vpx://user@vcenter.example.com/DC/esxi "guestvm1" \
  -of qcow2

# Output as raw (better performance)
virt-v2v -ic vpx://user@vcenter.example.com/DC/esxi "guestvm1" \
  -of raw
```

## Conversion Process Output

A successful conversion looks like:

```
[   0.0] Opening the source -ic vpx://user@vcenter.example.com/DC/esxi
[   1.1] Creating an overlay to protect the source from being modified
[   1.7] Initializing the target
[   1.7] Opening the overlay
[  25.8] Inspecting the overlay
[  30.9] Checking for sufficient free disk space in the guest
[  30.9] Estimating space required on target for each disk
[  30.9] Converting Red Hat Enterprise Linux Server release 8.9 to run on KVM
virt-v2v: This guest has virtio drivers installed.
[  45.4] Mapping filesystem data to avoid copying unused and blank areas
[  45.5] Closing the overlay
[  45.7] Checking if the guest needs BIOS or UEFI to boot
[  45.7] Assigning disks to buses
[  45.7] Copying disk 1/1 to /var/tmp/guestvm1-sda (qcow2)
[  93.0] Creating output metadata
[  93.0] Finished off
```

## Post-Conversion

### Verify the VM

```bash
# List VMs
virsh list --all

# Start the converted VM
virsh start guestvm1

# Connect to console
virsh console guestvm1
```

### Verify Functionality

After booting the converted VM:

```bash
# Check network
ip a
ping -c 3 gateway

# Check disk
df -h
lsblk

# Check services
systemctl status sshd
```

### Move Disk Image (if needed)

```bash
# Default output location is /var/tmp/
# Move to libvirt images directory
mv /var/tmp/guestvm1-sda /var/lib/libvirt/images/guestvm1.qcow2

# Update the VM config
virsh edit guestvm1
# Change the <source file="..."/> path
```

## Converting from Xen

```bash
# From Xen via SSH
virt-v2v -ic xen+ssh://root@xen-host.example.com "guestvm1"

# From a local Xen domain (using libvirt XML)
virt-v2v -i libvirtxml /etc/libvirt/xen/guestvm1.xml
```

## Converting from Hyper-V

```bash
# Export from Hyper-V first, then convert the disk
virt-v2v -i disk /path/to/disk.vhdx
```

## Converting a Local Disk Image

```bash
# Convert a VMDK file
virt-v2v -i disk /path/to/guestvm1.vmdk

# Convert a VHD/VHDX file
virt-v2v -i disk /path/to/guestvm1.vhdx

# Convert a raw disk
virt-v2v -i disk /path/to/guestvm1.raw
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `CURL: Error opening file` | VM is still running | Stop the VM before conversion |
| `error: cannot open connection` | vCenter unreachable | Check network, DNS, firewall (port 443) |
| SSL certificate error | Self-signed cert on vCenter | Add `?no_verify=1` to the URI |
| `virt-v2v: error: no root device found` | Unsupported filesystem (btrfs) | Only ext4/xfs/LVM supported |
| `insufficient free disk space` | Not enough space inside the VM | Free space per requirements table |
| Conversion very slow | Low inode count (XFS with many files) | Free inodes or reduce file count |
| Network not working post-conversion | Old VMware network driver | Install virtio drivers, check network config |

### Debug Mode

```bash
# Verbose output
virt-v2v -v -ic vpx://user@vcenter.example.com/DC/esxi "guestvm1"

# Very verbose (for bug reports)
virt-v2v -v -x -ic vpx://user@vcenter.example.com/DC/esxi "guestvm1"
```

### Check virt-v2v Capabilities

```bash
# Show supported input/output modes and features
virt-v2v --machine-readable
```

## Quick Reference

```bash
# Install
dnf install -y virt-v2v

# Convert from vCenter
virt-v2v -ic vpx://user@vcenter.example.com/Datacenter/esxi "vmname"

# Convert with self-signed cert
virt-v2v -ic vpx://user@vcenter.example.com/DC/esxi?no_verify=1 "vmname"

# Convert with password file
virt-v2v -ic vpx://user@vcenter.example.com/DC/esxi "vmname" -ip /tmp/pass

# Convert from OVA
virt-v2v -i ova /path/to/vm.ova

# Convert from disk image (VMDK, VHD, VHDX)
virt-v2v -i disk /path/to/disk.vmdk

# Output to specific directory
virt-v2v -ic vpx://... "vmname" -o local -os /var/lib/libvirt/images/

# Verify after conversion
virsh list --all
virsh start vmname
```
