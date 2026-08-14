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

### Export from ESXi Web UI

If exporting directly from the ESXi host interface:

1. Go to **Virtual Machines** → select the VM → **Actions** → **Export**
2. This downloads an `.ovf` file and one or more `.vmdk` files
3. Copy both files to the KVM server
4. Create an OVA archive from the exported files:

```sh
tar -cvf vm03.ova vm03.ovf disk-1.vmdk
```

### Define a Libvirt Pool for Converted VMs

Create a dedicated storage pool for converted machines:

```sh
mkdir -p /kvm-pool/esx
virsh pool-define-as ESX --type dir --target /kvm-pool/esx
virsh pool-start ESX
virsh pool-autostart ESX
virsh pool-list
```

### Convert the OVA

```bash
# Convert from OVA file (single .ova archive)
virt-v2v -i ova /var/tmp/guestvm1.ova -of qcow2

# Convert from OVF folder (exported as "Folder of files")
virt-v2v -i ova /path/to/ovf-folder/ -of qcow2

# Convert and output to a specific libvirt pool
virt-v2v -i ova vm03.ova -o libvirt -of qcow2 -os ESX
```

Example output:

```
[   0.0] Opening the source -i ova vm03.ova
virt-v2v: warning: making OVA directory public readable to work around
libvirt bug https://bugzilla.redhat.com/1045069
[  66.7] Creating an overlay to protect the source from being modified
[  67.0] Opening the overlay
[  71.3] Inspecting the overlay
[  86.2] Checking for sufficient free disk space in the guest
[  86.2] Estimating space required on target for each disk
[  86.2] Converting CentOS release 6.10 (Final) to run on KVM
virt-v2v: This guest has virtio drivers installed.
[ 140.6] Mapping filesystem data to avoid copying unused and blank areas
[ 141.0] Closing the overlay
[ 141.3] Assigning disks to buses
[ 141.3] Checking if the guest needs BIOS or UEFI to boot
[ 141.3] Initializing the target -o libvirt -os ESX
[ 141.3] Copying disk 1/1 to /kvm-pool/esx/vm03-sda (qcow2)
(100.00/100%)
[ 319.2] Creating output metadata
Pool ESX refreshed
Domain vm03 defined from /tmp/v2vlibvirta2d6b9.xml
[ 319.3] Finishing off
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

When you don't have vCenter and need to connect directly to an ESXi host, there are two methods:

### Method 1: SSH to VMX File (No Proprietary Software)

This uses SSH to access the VMX and VMDK files directly on the ESXi datastore:

```sh
# Import using SSH transport (recommended for ESXi without vCenter)
virt-v2v -i vmx -it ssh \
  -ip /tmp/esxi-pass \
  "ssh://root@esxi.example.com/vmfs/volumes/datastore1/guestvm1/guestvm1.vmx"

# Output to a specific directory
virt-v2v -i vmx -it ssh \
  -ip /tmp/esxi-pass \
  "ssh://root@esxi.example.com/vmfs/volumes/datastore1/guestvm1/guestvm1.vmx" \
  -o local -os /var/lib/libvirt/images/
```

> **Note:** The SSH method does not work with guests that have snapshots. Collapse snapshots first or use the VDDK method.

### Method 2: VDDK Library (Fastest, Proprietary)

If you have VMware's VDDK (VixDiskLib), this is the fastest method:

```sh
# Using VDDK with ESXi direct
virt-v2v \
  -ic "esx://root@esxi.example.com?no_verify=1" \
  -it vddk \
  -io vddk-libdir=/path/to/vmware-vix-disklib-distrib \
  -ip /tmp/esxi-pass \
  "guestvm1" \
  -o local -os /var/lib/libvirt/images/

# Using VDDK with vCenter
virt-v2v \
  -ic "vpx://root@vcenter.example.com/Datacenter/esxi?no_verify=1" \
  -it vddk \
  -io vddk-libdir=/path/to/vmware-vix-disklib-distrib \
  -ip /tmp/esxi-pass \
  "guestvm1" \
  -o local -os /var/lib/libvirt/images/
```

### ESXi Direct Methods: Comparison

| Method | URI | Speed | Requirements |
|--------|-----|-------|--------------|
| `-i vmx -it ssh` | `ssh://root@esxi/vmfs/...` | Medium | SSH access, no snapshots |
| `-ic esx:// -it vddk` | `esx://root@esxi` | Fast | VDDK library (proprietary) |
| `-ic vpx://` (vCenter) | `vpx://user@vcenter/DC/esxi` | Slow | vCenter ≥ 5.0 |
| `-i ova` | Local file | Medium | OVA export from VMware |

### ESXi Direct: Prerequisites

- SSH must be enabled on the ESXi host (Host → Manage → Services → SSH → Start)
- The VM must be powered off
- The user must have access to the datastore containing the VM's disks
- Port 443 (HTTPS) and port 22 (SSH) must be open from the conversion host to ESXi

### ESXi Direct: Step-by-Step

```sh
# 1. Enable SSH on ESXi (via DCUI or vSphere Client)
#    Host → Manage → Services → TSM-SSH → Start

# 2. Create password file
echo "ESXiRootPassword" > /tmp/esxi-pass
chmod 600 /tmp/esxi-pass

# 3. Power off the VM on ESXi
ssh root@esxi.example.com "vim-cmd vmsvc/power.off \$(vim-cmd vmsvc/getallvms | grep guestvm1 | awk '{print \$1}')"

# 4. Find the VMX path on the ESXi datastore
ssh root@esxi.example.com "find /vmfs/volumes/ -name 'guestvm1.vmx'"
# Output: /vmfs/volumes/datastore1/guestvm1/guestvm1.vmx

# 5. Convert using SSH transport
virt-v2v -i vmx -it ssh \
  -ip /tmp/esxi-pass \
  "ssh://root@esxi.example.com/vmfs/volumes/datastore1/guestvm1/guestvm1.vmx" \
  -o local -os /var/lib/libvirt/images/ \
  -of qcow2

# 6. Clean up
rm -f /tmp/esxi-pass

# 7. Verify
virsh list --all
virsh start guestvm1
```

### ESXi Direct: Finding the VMX Path

The SSH method requires the full path to the VMX file on the ESXi datastore:

```sh
# List all VMs and their VMX paths
ssh root@esxi.example.com "vim-cmd vmsvc/getallvms"

# Output:
# Vmid  Name          File                                         Guest OS      Version
# 1     guestvm1      [datastore1] guestvm1/guestvm1.vmx           rhel8_64Guest  vmx-19
# 2     webserver     [datastore1] webserver/webserver.vmx          ubuntu64Guest  vmx-19

# Find a specific VMX file
ssh root@esxi.example.com "find /vmfs/volumes/ -name '*.vmx' | grep guestvm1"
# /vmfs/volumes/datastore1/guestvm1/guestvm1.vmx
```

The VMX path in the URI must be percent-encoded for spaces:

```sh
# If the path contains spaces:
# /vmfs/volumes/datastore1/my guest/my guest.vmx
# becomes:
"ssh://root@esxi.example.com/vmfs/volumes/datastore1/my%20guest/my%20guest.vmx"
```

### ESXi vs vCenter: When to Use Each

| Method | Use When |
|--------|----------|
| `vpx://` (vCenter) | Enterprise environments, multiple ESXi hosts, centralized management |
| `esx://` (ESXi direct) | No vCenter available, standalone ESXi, small environments, lab/homelab |

### ESXi Direct: Troubleshooting

```sh
# Test SSH access
ssh root@esxi.example.com "hostname"

# Test connectivity to ESXi HTTPS (needed for VDDK method)
curl -k https://esxi.example.com/

# Check if VM is powered off
ssh root@esxi.example.com "vim-cmd vmsvc/power.getstate \$(vim-cmd vmsvc/getallvms | grep guestvm1 | awk '{print \$1}')"

# Find all VMX files on the host
ssh root@esxi.example.com "find /vmfs/volumes/ -name '*.vmx'"

# If conversion is slow, check network
iperf3 -c esxi.example.com

# Debug mode (SSH method)
virt-v2v -v -x -i vmx -it ssh \
  -ip /tmp/esxi-pass \
  "ssh://root@esxi.example.com/vmfs/volumes/datastore1/guestvm1/guestvm1.vmx"
```

### ESXi Direct: Batch Conversion

```sh
#!/bin/bash
set -euo pipefail

ESXI_HOST="esxi.example.com"
PASS_FILE="/tmp/esxi-pass"
OUTPUT_DIR="/var/lib/libvirt/images"
DATASTORE="datastore1"
VMS=("vm1" "vm2" "vm3")

for VM in "${VMS[@]}"; do
  echo "=== Converting $VM ==="

  # Power off (ignore error if already off)
  ssh root@$ESXI_HOST "vim-cmd vmsvc/power.off \$(vim-cmd vmsvc/getallvms | grep '$VM' | awk '{print \$1}')" 2>/dev/null || true
  sleep 5

  # Convert via SSH
  virt-v2v -i vmx -it ssh \
    -ip "$PASS_FILE" \
    "ssh://root@${ESXI_HOST}/vmfs/volumes/${DATASTORE}/${VM}/${VM}.vmx" \
    -o local -os "$OUTPUT_DIR" \
    -of qcow2

  echo "$VM conversion complete."
done

echo "All conversions finished."
virsh list --all
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

### Modify Network Configuration

Edit the VM and set the correct bridge interface:

```sh
virsh edit guestvm1
```

Change the `<source bridge>` line to match your host bridge:

```xml
    <interface type='bridge'>
      <mac address='52:54:00:8d:b7:c3'/>
      <source bridge='br0'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
```

### Start the VM

```sh
virsh start guestvm1
```

### Post-Conversion OS Cleanup

After booting the converted VM for the first time, fix network and hardware references:

```sh
# Remove MAC address binding from network scripts (RHEL/CentOS 6/7)
# Comment or remove the HWADDR line in:
vi /etc/sysconfig/network-scripts/ifcfg-eth0

# Remove persistent network udev rules (prevents NIC renaming issues)
rm -f /etc/udev/rules.d/70-persistent-net.rules

# Reboot to pick up changes
reboot
```

For RHEL/CentOS 8+ and Ubuntu (using NetworkManager or netplan), the MAC address is usually not hardcoded in config files, but verify with:

```sh
# Check for MAC references in NetworkManager connections
grep -r "mac-address" /etc/NetworkManager/system-connections/

# Remove VMware tools if present
yum remove -y open-vm-tools || dnf remove -y open-vm-tools
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

# Convert from ESXi via SSH (no vCenter, no VDDK)
virt-v2v -i vmx -it ssh -ip /tmp/pass \
  "ssh://root@esxi/vmfs/volumes/datastore1/vmname/vmname.vmx"

# Convert from ESXi via VDDK (fastest, requires proprietary lib)
virt-v2v -ic "esx://root@esxi?no_verify=1" -it vddk \
  -io vddk-libdir=/path/to/vddk -ip /tmp/pass "vmname"

# Convert from OVA
virt-v2v -i ova /path/to/vm.ova

# Convert from OVA to specific libvirt pool
virt-v2v -i ova vm.ova -o libvirt -of qcow2 -os poolname

# Convert from disk image (VMDK, VHD, VHDX)
virt-v2v -i disk /path/to/disk.vmdk

# Output to specific directory
virt-v2v -ic vpx://... "vmname" -o local -os /var/lib/libvirt/images/

# Verify after conversion
virsh list --all
virsh start vmname
```
