# Adding a New Disk in KVM

This guide covers creating and attaching disk images to KVM virtual machines — both to running VMs (hot-plug) and to stopped VMs (persistent config).

## Method 1: Hot-Plug to a Running VM

### Create the Disk Image

```bash
# Create a qcow2 disk (thin provisioned — grows as data is written)
qemu-img create -f qcow2 /var/lib/libvirt/images/vm01-data.qcow2 50G

# Alternative: create via storage pool (virsh vol-create-as)
virsh vol-create-as default --format qcow2 vm01-data.qcow2 50G

# Create with metadata preallocation (better performance, still thin)
qemu-img create -f qcow2 -o preallocation=metadata /var/lib/libvirt/images/vm01-data.qcow2 50G

# Create a raw disk (full allocation, best I/O performance)
qemu-img create -f raw /var/lib/libvirt/images/vm01-data.raw 50G

# Verify the disk was created
qemu-img info /var/lib/libvirt/images/vm01-data.qcow2
```

### Attach the Disk (Hot-Plug)

```bash
# Attach qcow2 disk to a running VM
virsh attach-disk vm01 \
    /var/lib/libvirt/images/vm01-data.qcow2 vdb \
    --driver qemu --subdriver qcow2 --persistent

# Attach raw disk
virsh attach-disk vm01 \
    /var/lib/libvirt/images/vm01-data.raw vdb \
    --driver qemu --subdriver raw --persistent

# Attach with cache mode
virsh attach-disk vm01 \
    /var/lib/libvirt/images/vm01-data.qcow2 vdb \
    --driver qemu --subdriver qcow2 --cache none --persistent

# Attach as read-only
virsh attach-disk vm01 \
    /var/lib/libvirt/images/vm01-data.qcow2 vdb \
    --driver qemu --subdriver qcow2 --persistent --mode readonly
```

Flags:
- `--persistent` — saves to VM config (survives reboot)
- `--live` — applies to running VM (implied when VM is running)
- `--config` — saves to config only (takes effect on next boot)

### Verify Attachment (Host Side)

```bash
# List VM block devices
virsh domblklist vm01
# Target  Source
# ------------------------------------------------
# vda     /var/lib/libvirt/images/vm01.qcow2
# vdb     /var/lib/libvirt/images/vm01-data.qcow2

# Check block device info
virsh domblkinfo vm01 vdb

# Verify the XML config file directly
cat /etc/libvirt/qemu/vm01.xml | grep -A5 "vdb"
```

## Method 2: Attach to a Stopped VM

### Via virsh attach-disk

```bash
# VM must be shut down
virsh shutdown vm01

# Attach disk (config only)
virsh attach-disk vm01 \
    /var/lib/libvirt/images/vm01-data.qcow2 vdb \
    --driver qemu --subdriver qcow2 --persistent --config

# Start the VM
virsh start vm01
```

### Via virsh edit (XML)

```bash
virsh edit vm01
```

Add the following inside the `<devices>` section:

```xml
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none' discard='unmap'/>
  <source file='/var/lib/libvirt/images/vm01-data.qcow2'/>
  <target dev='vdb' bus='virtio'/>
</disk>
```

Start the VM:

```bash
virsh start vm01
```

### Via XML File (attach-device)

Create a disk XML file:

```bash
cat > /tmp/disk-vdb.xml << EOF
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none' discard='unmap'/>
  <source file='/var/lib/libvirt/images/vm01-data.qcow2'/>
  <target dev='vdb' bus='virtio'/>
</disk>
EOF
```

Attach it:

```bash
# To running VM (persistent)
virsh attach-device vm01 /tmp/disk-vdb.xml --persistent

# To stopped VM (config only)
virsh attach-device vm01 /tmp/disk-vdb.xml --config
```

## Method 3: Add Disk at VM Creation

```bash
# Single disk
virt-install \
    --name vm01 \
    --ram 2048 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/vm01-root.qcow2,size=20 \
    --os-variant rocky9.0 \
    --network network=default \
    --cdrom /path/to/installer.iso

# Multiple disks at creation
virt-install \
    --name vm01 \
    --ram 4096 \
    --vcpus 4 \
    --disk path=/var/lib/libvirt/images/vm01-root.qcow2,size=30 \
    --disk path=/var/lib/libvirt/images/vm01-data.qcow2,size=100 \
    --disk path=/var/lib/libvirt/images/vm01-logs.qcow2,size=50 \
    --os-variant rocky9.0 \
    --network network=default \
    --cdrom /path/to/installer.iso
```

## Inside the Guest (Partition, Format, Mount)

After attaching the disk, the guest OS sees a new block device. You need to partition, format, and mount it.

### Identify the New Disk

```bash
# List block devices
lsblk

# Example output:
# NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
# vda    252:0    0   20G  0 disk
# └─vda1 252:1    0   20G  0 part /
# vdb    252:16   0   50G  0 disk    <-- new disk

# Alternative: fdisk
fdisk -l /dev/vdb

# Check with dmesg (recent attachment)
dmesg | tail -10
```

### Option A: Use Entire Disk Without Partitioning

```bash
# Format directly (no partition table)
mkfs.xfs /dev/vdb

# Or ext4
mkfs.ext4 /dev/vdb

# Create mount point
mkdir -p /data

# Mount
mount /dev/vdb /data

# Verify
df -h /data
```

### Option B: Create a Partition First

```bash
# Create partition table and partition
fdisk /dev/vdb
# n → new partition
# p → primary
# 1 → partition number
# Enter → default first sector
# Enter → default last sector (use entire disk)
# w → write and exit

# Or non-interactive with parted
parted /dev/vdb --script mklabel gpt mkpart primary xfs 0% 100%

# Format the partition
mkfs.xfs /dev/vdb1

# Create mount point and mount
mkdir -p /data
mount /dev/vdb1 /data

# Verify
df -h /data
lsblk
```

### Option C: Use LVM

```bash
# Create physical volume
pvcreate /dev/vdb

# Create or extend a volume group
vgcreate vg_data /dev/vdb
# or extend existing: vgextend vg_data /dev/vdb

# Create logical volume
lvcreate -n lv_data -l 100%FREE vg_data

# Format
mkfs.xfs /dev/vg_data/lv_data

# Mount
mkdir -p /data
mount /dev/vg_data/lv_data /data
```

### Make It Persistent (fstab)

```bash
# Get the UUID (preferred over device names)
blkid /dev/vdb1
# or for LVM:
blkid /dev/vg_data/lv_data

# Add to /etc/fstab
echo "UUID=<your-uuid>  /data  xfs  defaults  0 0" >> /etc/fstab

# Or for LVM
echo "/dev/vg_data/lv_data  /data  xfs  defaults  0 0" >> /etc/fstab

# Test fstab entry (mount without reboot)
mount -a

# Verify
df -h /data
```

## Detach a Disk

```bash
# Detach from running VM (hot-unplug)
virsh detach-disk vm01 vdb --persistent

# Detach with live + persistent
virsh detach-disk --domain vm01 --persistent --live --target vdb

# Detach via XML
virsh detach-device vm01 /tmp/disk-vdb.xml --persistent

# Verify removal
virsh domblklist vm01
```

> **Important:** Unmount the filesystem inside the guest before detaching: `umount /data`

## Resize an Existing Disk

### Increase Disk Image Size (Host)

```bash
# Shutdown VM (recommended for qcow2 resize)
virsh shutdown vm01

# Resize the image file
qemu-img resize /var/lib/libvirt/images/vm01-data.qcow2 +20G

# Start VM
virsh start vm01
```

### Online Resize (VM Running)

```bash
# Resize block device from host (no shutdown needed)
virsh blockresize vm01 /var/lib/libvirt/images/vm01-data.qcow2 70G
```

### Expand Inside the Guest

After resizing the image, the guest needs to expand the filesystem:

```bash
# If no partition table (raw filesystem on /dev/vdb)
xfs_growfs /data        # XFS
resize2fs /dev/vdb      # ext4

# If partitioned (need to grow partition first)
growpart /dev/vdb 1
xfs_growfs /data        # XFS
resize2fs /dev/vdb1     # ext4

# If LVM
pvresize /dev/vdb
lvextend -l +100%FREE /dev/vg_data/lv_data
xfs_growfs /data        # XFS
resize2fs /dev/vg_data/lv_data   # ext4
```

## Disk Formats Comparison

| Format | Thin Provisioned | Snapshots | Performance | Use Case |
|--------|-----------------|-----------|-------------|----------|
| `qcow2` | Yes | Yes (internal) | Good | Default choice, flexible |
| `raw` | No (full allocation) | No | Best I/O | Performance-critical workloads |
| `qcow2` + preallocation | Partial | Yes | Better than default qcow2 | Balance of performance and flexibility |

## Cache Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `none` | No host caching, O_DIRECT | Best for production (data safety) |
| `writethrough` | Read cache, write-through | Safe, moderate performance |
| `writeback` | Full host cache | Best performance, risk of data loss on host crash |
| `unsafe` | No flush, no sync | Testing only (fastest, most dangerous) |

```bash
# Attach with specific cache mode
virsh attach-disk vm01 /path/to/disk.qcow2 vdb \
    --driver qemu --subdriver qcow2 --cache none --persistent
```

## Discard/TRIM Support

Enable discard to reclaim freed space in thin-provisioned qcow2 images:

```xml
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none' discard='unmap'/>
  <source file='/var/lib/libvirt/images/vm01-data.qcow2'/>
  <target dev='vdb' bus='virtio'/>
</disk>
```

Inside the guest:

```bash
# Mount with discard option
mount -o discard /dev/vdb /data

# Or add to fstab
UUID=<uuid>  /data  xfs  defaults,discard  0 0

# Manual TRIM
fstrim -v /data

# From host: trim all VM filesystems
virsh domfstrim vm01
```

## Troubleshooting

### Disk Not Visible in Guest

```bash
# Check if attached on host
virsh domblklist vm01

# Check dmesg in guest
dmesg | grep -i vd

# Check if virtio driver is loaded (guest)
lsmod | grep virtio

# If using IDE bus instead of virtio, device will be /dev/sdX
```

### Cannot Detach Disk

```bash
# Ensure it's unmounted in the guest first
# Inside guest:
umount /data

# Then detach from host
virsh detach-disk vm01 vdb --persistent

# If still failing, try with --live flag
virsh detach-disk vm01 vdb --persistent --live
```

### Permission Denied

```bash
# Check ownership of the disk image
ls -la /var/lib/libvirt/images/vm01-data.qcow2

# Should be owned by qemu:qemu (or libvirt-qemu on Ubuntu)
sudo chown qemu:qemu /var/lib/libvirt/images/vm01-data.qcow2

# Or on Ubuntu
sudo chown libvirt-qemu:kvm /var/lib/libvirt/images/vm01-data.qcow2

# Check SELinux context (RHEL)
ls -lZ /var/lib/libvirt/images/vm01-data.qcow2

# Fix SELinux label
sudo restorecon -v /var/lib/libvirt/images/vm01-data.qcow2
```

### Disk Full (qcow2 Growing)

```bash
# Check actual disk usage on host
qemu-img info /var/lib/libvirt/images/vm01-data.qcow2
# Look at "disk size" vs "virtual size"

# Reclaim space (after deleting files in guest + fstrim)
# Option 1: fstrim from inside guest, then:
qemu-img convert -O qcow2 vm01-data.qcow2 vm01-data-compact.qcow2
mv vm01-data-compact.qcow2 vm01-data.qcow2

# Option 2: Use virt-sparsify (VM must be off)
virt-sparsify --in-place /var/lib/libvirt/images/vm01-data.qcow2
```
