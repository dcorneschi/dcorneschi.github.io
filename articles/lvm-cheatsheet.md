# LVM Cheatsheet

Logical Volume Manager (LVM) commands for creating, extending, reducing, and managing physical volumes, volume groups, and logical volumes on Linux.

---

## LVM Architecture

```
Physical Disks → Physical Volumes (PV) → Volume Groups (VG) → Logical Volumes (LV) → Filesystems

Example:
/dev/sdb1 (PV) ──┐
/dev/sdc1 (PV) ──┤── vg_data (VG) ──┬── lv_root (LV) → /
/dev/sdd1 (PV) ──┘                  ├── lv_swap (LV) → swap
                                    └── lv_home (LV) → /home
```

| Layer | Description |
|-------|-------------|
| **PV** (Physical Volume) | A disk or partition initialized for LVM |
| **VG** (Volume Group) | A pool of storage combining one or more PVs |
| **LV** (Logical Volume) | A virtual partition carved from a VG, formatted and mounted |
| **PE** (Physical Extent) | The smallest allocatable unit (default 4 MiB) |
| **LE** (Logical Extent) | Maps 1:1 to a PE within a logical volume |

> **Key benefit:** LVM decouples logical storage from physical disks. You can resize, move, and snapshot volumes without unmounting or rebooting.

### Device Naming Convention

```bash
/dev/<volume_group>/<logical_volume>
```

LVM volumes are accessed as `/dev/vg_name/lv_name` or equivalently `/dev/mapper/vg_name-lv_name`.

---

## Physical Volumes (PV)

### PV Creation and Removal

```bash
# Create physical volumes
pvcreate /dev/sdb                    # On whole disk
pvcreate /dev/sdb1                   # On partition
pvcreate /dev/sdb1 /dev/sdc1         # Multiple PVs
pvcreate --force /dev/sdb1           # Force overwrite
pvcreate --metadatasize 64M /dev/sdb1 # Custom metadata size

# Remove physical volumes
pvremove /dev/sdb1                   # Remove PV
pvremove -f /dev/sdb1                # Force removal
```

### PV Information Commands

```bash
# Basic information
pvs                                  # List all PVs (summary)
pvdisplay                            # Detailed PV information
pvdisplay /dev/sdb1                  # Specific PV details
pvscan                               # Scan for PVs
pvck /dev/sdb1                       # Check PV consistency
pvck --dump headers /dev/sdb1        # Dump and check PV headers and metadata structures

# Custom output formats
pvs -o pv_name,vg_name,pv_size,pv_free,pv_used
pvs -o +uuid                        # Show PVs and UUID
pvs --units g                       # Show sizes in GB
pvs --units m                       # Show sizes in MB
pvs --aligned                       # Align output columns
pvs -o pv_name,pv_uuid              # Show UUIDs
pvs --segments -o +seg_size,lv_name,lv_size,seg_pe_ranges  # Full PV details

# Show physical extent map (which LV uses which extents)
pvdisplay -m /dev/sdb1
pvdisplay -m                        # Display mapping of physical extents to LVs

# Show all attributes in columns
pvs -a -o+pv_used,pv_pe_count,pv_pe_alloc_count
```

### PV Operations

```bash
# Move data off PV
pvmove /dev/sdb1                    # Move to any available space
pvmove /dev/sdb1 /dev/sdc1          # Move to specific PV
pvmove -n lv_data /dev/sdb1 /dev/sdc1  # Move only extents belonging to a specific LV
pvmove -b /dev/sdb1 /dev/sdc1       # Run the daemon in the background
pvmove -i 5 /dev/sdb1              # Report progress every 5 seconds
pvmove /dev/sdb1:1000-1999          # Move a range of Physical Extents

# Check progress (pvmove runs in background for large volumes)
lvs -a -o+devices

# Resize PV (after partition resize)
pvresize /dev/sdb1                  # Auto-detect new size
pvresize --setphysicalvolumesize 100G /dev/sdb1  # Set specific size

# Change PV allocation policy
pvchange -x n /dev/sdb1             # Prohibit allocation
pvchange -x y /dev/sdb1             # Allow allocation

# Display PV attributes
pvs -o pv_name,pv_attr              # Show attributes

# Scan and activate all volume groups found
pvscan --activate ay
```

---

## Volume Groups (VG)

### VG Creation and Removal

```bash
# Create volume groups
vgcreate vg_data /dev/sdb1                      # Single PV
vgcreate vg_storage /dev/sdb1 /dev/sdc1         # Multiple PVs
vgcreate vg_data /dev/sd[b-e]                   # Multiple disks using glob
vgcreate --physicalextentsize 32M vg_data /dev/sdb1  # Custom extent size
vgcreate --maxlogicalvolumes 256 vg_data /dev/sdb1   # Max LVs limit

# Remove volume group
vgremove vg_data                    # Remove VG (must be empty)
vgremove -f vg_data                 # Force remove (removes all LVs)
```

### VG Information Commands

```bash
# Basic information
vgs                                # List all VGs (summary)
vgdisplay                          # Detailed VG information
vgdisplay vg_data                  # Specific VG details
vgdisplay --partial --verbose      # Display VGs with missing PVs
vgscan                             # Scan for VGs and rebuild caches
vgck vg_data                       # Check VG consistency

# Custom output formats
vgs -o vg_name,pv_count,lv_count,vg_size,vg_free,vg_uuid
vgs -o +pv_name                    # Show VGs and PVs
vgs -o vg_tags                     # Check tags for a VG
vgs --units g                      # Show sizes in GB
vgs -o vg_name,vg_free_percent     # Show free space percentage

# Show all VGs including hidden
vgs -a
```

### VG Operations

```bash
# Extend/reduce volume groups
vgextend vg_data /dev/sdc1                # Add PV to VG
vgextend vg_data /dev/sdd1 /dev/sde1      # Add multiple PVs
vgreduce vg_data /dev/sdc1                # Remove PV from VG
vgreduce --removemissing vg_data          # Remove all missing PVs
vgreduce --removemissing --force vg_data  # Remove missing PVs and LVs from VG
vgreduce --all vg_data                    # Remove all empty PVs

# VG state management
vgchange -ay vg_data                 # Activate VG
vgchange -an vg_data                 # Deactivate VG
vgchange -ay                         # Activate all known VGs in the system
vgchange --sysinit -ay vg_data       # System initialization activate
vgchange -ay --ignorelockingfailure  # Proceed with read-only metadata ops if locking fails
vgchange -ay --partial vg_data       # Activate LVs even if PVs are missing/failed

# VG manipulation
vgrename vg_old vg_new               # Rename VG
vgsplit vg_source vg_dest /dev/sdc1  # Split VG
vgmerge vg_dest vg_source            # Merge VGs

# Import/Export VGs (for moving disks between systems)
vgchange -an vg_data               # Deactivate first
vgexport vg_data                   # Export VG
# Move disks to target system
pvscan                             # Scan on target
vgimport vg_data                   # Import VG
vgchange -ay vg_data               # Activate

# Backup/Restore VG metadata
vgcfgbackup vg_data                              # Backup VG config
vgcfgbackup                                      # Backup all VGs
vgcfgrestore vg_data                             # Restore VG config
vgcfgbackup -f /tmp/vg_data.backup vg_data       # Backup to specific file
vgcfgrestore -f /etc/lvm/backup/vg_data vg_data  # Restore from specific file
```

---

## Logical Volumes (LV)

### LV Creation

```bash
# Basic LV creation
lvcreate -L 10G -n lv_data vg_storage          # Fixed size
lvcreate -l 1000 -n lv_test vg_storage         # By extents
lvcreate -l 50%VG -n lv_logs vg_storage        # 50% of VG
lvcreate -l 100%FREE -n lv_backup vg_storage   # All free space
lvcreate -L 20G -n lv_fast vg_data /dev/ssd1   # On specific PVs only

# Advanced LV creation
lvcreate -L 20G -i 3 -I 64K -n lv_striped vg_storage    # Striped
lvcreate -L 10G -m 1 -n lv_mirror vg_storage            # Mirrored
lvcreate --type raid1 -L 10G -n lv_raid1 vg_storage     # RAID 1
lvcreate --type raid5 -L 20G -n lv_raid5 vg_storage     # RAID 5

# Thin provisioning
lvcreate -L 100G --thinpool thin_pool vg_data           # Thin pool
lvcreate -V 50G --thin vg_data/thin_pool -n lv_thin     # Thin volume
```

### Size Specification

| Flag | Meaning | Example |
|------|---------|---------|
| `-L` | Absolute size | `-L 50G`, `-L 500M`, `-L 1T` |
| `-l` | Extents or percentage | `-l 100` (100 extents) |
| `-l 100%FREE` | All free space in VG | — |
| `-l 100%VG` | All space in VG | — |
| `-l 50%FREE` | Half of free space | — |
| `-l +100%FREE` | Extend by all free space | — |
| `-l +100%PVS` | Extend by all free space on the PV | — |
| `-l 10%ORIGIN` | Percentage of origin (snapshots) | — |

### LV Information Commands

```bash
# Basic information
lvs                                # List all LVs (summary)
lvdisplay                         # Detailed LV information
lvdisplay /dev/vg_data/lv_home    # Specific LV details
lvdisplay -m /dev/vg_data/lv_home # Display mapping of logical extents to PVs and PEs
lvscan                            # Scan for LVs

# Custom output formats
lvs -o lv_name,vg_name,lv_size,lv_attr,seg_count
lvs -o lv_name,lv_path            # Show device paths
lvs -o lv_name,origin,snap_percent # Snapshot information
lvs -o lv_name,data_percent,metadata_percent # Thin pool usage
lvs -o +devices                   # Show which LV is using which PV
lvs -a -o +devices                # All LVs, even those not accessible
lvs -o +tags                      # Show tags
lvs --units g                     # Show sizes in GB

# Status information
lvs -o lv_name,copy_percent       # Mirror sync status
lvs -o lv_name,sync_percent       # RAID sync status
lvs -o lv_name,lv_attr | grep 'swi-a-s'  # Find snapshots

# Show all (including internal/hidden)
lvs -a
```

### LV Attribute Flags

The `lv_attr` column in `lvs` shows a 10-character string:

| Position | Meaning |
|----------|---------|
| 1 | Volume type: `-` (normal), `v` (virtual/thin), `t` (thin pool), `V` (thin volume), `s` (snapshot), `m` (mirror), `r` (RAID) |
| 2 | Permissions: `w` (writeable), `r` (read-only) |
| 3 | Allocation policy: `n` (normal), `a` (anywhere), `c` (contiguous), `i` (inherited) |
| 4 | Fixed minor: `-` (no), `m` (yes) |
| 5 | State: `a` (active), `s` (suspended), `I` (invalid snapshot) |
| 6 | Device open: `o` (open), `-` (closed) |
| 7 | Target type: `-` (default/mirror), `t` (thin), `r` (RAID) |
| 8 | Zero: `z` (newly allocated blocks are zeroed) |
| 9 | Health: `-` (ok), `p` (partial), `r` (refresh needed) |
| 10 | Skip activation: `-` (no), `k` (skip) |

### LV Operations

```bash
# Extend logical volumes
lvextend -L +5G /dev/vg_data/lv_home      # Add 5GB
lvextend -L 20G /dev/vg_data/lv_home      # Resize to 20GB total
lvextend -l +100%FREE /dev/vg_data/lv_home # Use all free space
lvextend -L +10G -r /dev/vg_data/lv_home   # Extend and resize filesystem
lvextend /dev/vg_data/lv_home /dev/sdb     # Extend by the amount of free space on PV
lvextend -l +100%PVS /dev/vg_data/lv_home  # Extend by all free space on the PV

# Reduce logical volumes (DANGEROUS!)
umount /dev/vg_data/lv_home
e2fsck -f /dev/vg_data/lv_home
resize2fs /dev/vg_data/lv_home 15G
lvreduce -L 15G /dev/vg_data/lv_home

# LV state management
lvchange -ay /dev/vg_data/lv_home         # Activate LV
lvchange -an /dev/vg_data/lv_home         # Deactivate LV
lvchange -a n /dev/vg_data/lv_home        # Deactivate LV (alternate form)
lvchange -pr /dev/vg_data/lv_home         # Activate read-only

# Remove and rename
lvrename vg_data lv_old lv_new            # Rename LV
lvrename /dev/vg_data/lv_old /dev/vg_data/lv_new  # Rename using full paths
lvremove /dev/vg_data/lv_temp             # Remove LV
lvremove -f /dev/vg_data/lv_temp          # Force remove
```

> After renaming, update `/etc/fstab` and any references to the old device path.

---

## Extending Logical Volumes

### Extend the LV

```bash
# Add 10G to the LV
lvextend -L +10G /dev/vg_data/lv_app

# Extend to a specific total size
lvextend -L 80G /dev/vg_data/lv_app

# Extend using all remaining free space in VG
lvextend -l +100%FREE /dev/vg_data/lv_app

# Extend using a specific PV
lvextend -L +10G /dev/vg_data/lv_app /dev/sdc1
```

### Resize the Filesystem

```bash
# ext4: resize after extending LV
resize2fs /dev/vg_data/lv_app

# ext4: resize to a specific size
resize2fs /dev/vg_data/lv_app 70G

# XFS: grow to fill LV (must be mounted)
xfs_growfs /mount/point

# XFS: can only grow, never shrink
```

### Extend LV + Filesystem in One Step

```bash
# lvextend with -r flag resizes the filesystem automatically
lvextend -L +10G -r /dev/vg_data/lv_app

# Extend to use all free space and resize filesystem
lvextend -l +100%FREE -r /dev/vg_data/lv_app
```

> The `-r` (`--resizefs`) flag works with ext4, XFS, and other supported filesystems. This is the recommended approach.

---

## Reducing Logical Volumes

> **Warning:** Reducing an LV can destroy data. Always backup first. XFS cannot be shrunk.

### Reduce ext4

```bash
# 1. Unmount the filesystem
umount /mnt/app

# 2. Check filesystem integrity
e2fsck -f /dev/vg_data/lv_app

# 3. Shrink the filesystem first
resize2fs /dev/vg_data/lv_app 40G

# 4. Reduce the LV
lvreduce -L 40G /dev/vg_data/lv_app

# 5. Remount
mount /dev/vg_data/lv_app /mnt/app
```

### Reduce LV + Filesystem in One Step

```bash
# lvreduce with -r flag handles filesystem resize automatically
# Still requires unmount for ext4
umount /mnt/app
lvreduce -L 40G -r /dev/vg_data/lv_app
mount /dev/vg_data/lv_app /mnt/app
```

> **XFS does not support shrinking.** To reduce an XFS volume: backup data, remove the LV, recreate smaller, restore data.

---

## LVM Snapshots

### Snapshot Creation and Management

```bash
# Create snapshots
lvcreate -L 5G -s -n lv_app_snap /dev/vg_data/lv_app       # Fixed size
lvcreate -l 10%ORIGIN -s -n lv_snap /dev/vg_data/lv_orig   # Percentage of origin

# Create a thin snapshot (from thin volume, no pre-allocation)
lvcreate -s -n lv_app_snap /dev/vg_data/lv_app

# Monitor snapshot usage
lvs -o lv_name,origin,snap_percent

# Extend snapshot space
lvextend -L +1G /dev/vg_data/lv_app_snap

# Revert to snapshot (merge back into origin)
# Requires unmounting the origin LV (or reboot if root)
umount /mnt/app
lvconvert --merge /dev/vg_data/lv_app_snap
mount /dev/vg_data/lv_app /mnt/app

# Mount snapshot read-only
mkdir -p /mnt/snapshot
mount -o ro /dev/vg_data/lv_app_snap /mnt/snapshot

# Remove snapshot
umount /mnt/snapshot
lvremove /dev/vg_data/lv_app_snap
```

> If a snapshot reaches 100% usage, it becomes invalid and unusable.

### Snapshot Backup Script

```bash
#!/bin/bash
# snapshot_backup.sh

LV="$1"
SNAP_SIZE="2G"
BACKUP_DIR="/backup"
DATE=$(date +%Y%m%d)

# Create snapshot
SNAP_NAME="$(basename $LV)_snap_$(date +%H%M%S)"
lvcreate -L $SNAP_SIZE -s -n $SNAP_NAME $LV

# Mount and backup
mkdir -p /mnt/snap
mount /dev/$(dirname $LV | sed 's|/dev/||')/$SNAP_NAME /mnt/snap
rsync -av /mnt/snap/ $BACKUP_DIR/$DATE/

# Cleanup
umount /mnt/snap
lvremove -f /dev/$(dirname $LV | sed 's|/dev/||')/$SNAP_NAME
```

---

## Thin Provisioning

Allocate more logical storage than physically available. Space is consumed on write.

### Thin Pool Management

```bash
# Create thin pools
lvcreate -L 100G --thinpool thin_pool vg_data
lvcreate -L 1G --poolmetadata thin_meta vg_data
lvcreate -L 100G --thinpool pool --poolmetadata meta vg_data

# Manage thin pools
lvextend -L +50G vg_data/thin_pool              # Extend pool
lvs -o lv_name,data_percent,metadata_percent   # Monitor usage
dmsetup status vg_data-thin_pool               # Detailed status
```

### Thin Volume Management

```bash
# Create thin volumes (overcommit — total exceeds pool size)
lvcreate -V 80G --thin -n lv_vm1 vg_data/thin_pool
lvcreate -V 80G --thin -n lv_vm2 vg_data/thin_pool
lvcreate -V 80G --thin -n lv_vm3 vg_data/thin_pool

# Thin snapshots
lvcreate -s vg_data/lv_vm1 -n lv_vm1_snap

# Monitor thin usage
lvs -o lv_name,lv_size,data_percent vg_data
df -h  # Actual filesystem usage
```

> **Warning:** Monitor thin pool usage. If the pool fills to 100%, thin volumes become unresponsive.

---

## LVM RAID

### RAID Level Creation

```bash
# RAID 1 (mirror)
lvcreate --type raid1 -m 1 -L 50G -n lv_raid1 vg_data

# RAID 1 (3-way mirror)
lvcreate --type raid1 -m 2 -L 50G -n lv_raid1 vg_data

# RAID 5 (striping with parity, requires 3+ PVs)
lvcreate --type raid5 -i 2 -L 100G -n lv_raid5 vg_data

# RAID 6 (dual parity, requires 5+ PVs)
lvcreate --type raid6 -i 3 -L 100G -n lv_raid6 vg_data

# RAID 10 (stripe of mirrors)
lvcreate --type raid10 -L 40G -n lv_raid10 vg_data

# Striped LV (RAID0-like, no redundancy)
lvcreate --type striped -i 3 -L 90G -n lv_stripe vg_data
lvcreate --type striped -i 3 -I 64k -L 90G -n lv_stripe vg_data  # Custom stripe size
```

### RAID Operations

```bash
# Check RAID status
lvs -o lv_name,sync_percent,lv_attr
lvs -a -o name,copy_percent,devices
lvdisplay /dev/vg_data/lv_raid5 | grep -i sync

# Repair RAID volume
lvconvert --repair /dev/vg_data/lv_raid5

# Replace failed device
lvconvert --replace /dev/sdb1 /dev/vg_data/lv_raid5 /dev/sde1
```

---

## LVM Cache

Use a fast device (SSD/NVMe) to cache a slower device (HDD):

### Cache Setup

```bash
# Create cache pool (on fast storage)
lvcreate -L 10G -n cache_pool vg_data /dev/nvme0n1p1
lvcreate -L 100M -n cache_meta vg_data /dev/nvme0n1p1

# Convert to cache pool
lvconvert --type cache-pool --poolmetadata cache_meta vg_data/cache_pool

# Attach cache to an existing LV
lvconvert --type cache --cachepool vg_data/cache_pool vg_data/lv_data

# Remove cache
lvconvert --uncache vg_data/lv_data
```

### Cache Monitoring

```bash
# Cache statistics
lvs -o+cache_read_hits,cache_read_misses
lvs -o lv_name,cache_used_blocks,cache_total_blocks
dmsetup status | grep cache
```

---

## Device Paths

LVM devices are accessible via multiple paths:

```bash
# Canonical path
/dev/vg_data/lv_app

# Device-mapper path (what the kernel actually uses)
/dev/mapper/vg_data-lv_app

# dm- device
/dev/dm-0

# All three reference the same block device
ls -la /dev/vg_data/lv_app
ls -la /dev/mapper/vg_data-lv_app
```

> For `/etc/fstab`, use either `/dev/mapper/vg_name-lv_name` or the UUID. Avoid `/dev/dm-*` as numbering can change.

---

## Configuration and Metadata

### LVM Configuration File

```bash
# Main configuration
/etc/lvm/lvm.conf

# Metadata backups (automatic)
/etc/lvm/backup/       # Latest backup per VG
/etc/lvm/archive/      # Historical backups

# Dump current config
lvm dumpconfig
```

### Metadata Backup and Restore

```bash
# Manual backup
vgcfgbackup vg_data

# Backup all VGs
vgcfgbackup

# Backup to custom location
vgcfgbackup -f /backup/vg_data.backup vg_data

# Restore from backup
vgcfgrestore vg_data

# Restore from a specific archive file
vgcfgrestore -f /etc/lvm/archive/vg_data_00042-123456789.vg vg_data

# Archive current config
tar -czf /backup/lvm_config_$(date +%Y%m%d).tar.gz /etc/lvm/
```

### Scanning and Cache

```bash
# Rebuild LVM cache
pvscan
vgscan
lvscan

# Force metadata re-read
pvscan --cache

# Rebuild with node creation
vgscan --mknodes
```

---

## Other LVM Commands

### LVM Configuration Tools

```bash
# Display LVM configuration
lvmconfig

# Validate current configuration
lvmconfig --validate
lvm dumpconfig --validate

# Generate lvm.conf with all defaults and comments
lvmconfig --typeconfig default --withcomments

# Create lvm2 information dumps for diagnostic purposes
lvmdump -a -m

# Scan for all devices visible to LVM2
lvmdiskscan
```

### Device Mapper (dmsetup)

```bash
# Outputs brief information about device-mapper devices
dmsetup info -c

# Show low level device names
dmsetup ls

# Show device names as a tree
dmsetup ls --tree

# Show device mapper status
dmsetup status

# Show device mapper info
dmsetup info

# Remove a LV from device mapper table
dmsetup remove <vg_name>-<lv_name>
```

### LVM Filter Management

```bash
# Generate a suggested lvm filter based on the LVM devices found
pvs -a --config 'devices { filter = [ "a|.*|" ] }' --noheadings -opv_name,fmt,vg_name | \
  awk 'BEGIN { f = ""; } NF == 3 { n = "\42a|"$1"|\42, "; f = f n; } END { print "Suggested filter line for /etc/lvm/lvm.conf:\n filter = [ "f"\"r|.*|\" ]" }'

# Test a filter on the fly
lvs --config 'devices{ filter = [ "a|/dev/hda2|", "a|/dev/mapper/mpath|", "r|.*|" ] }'
```

### Disk Operations

```bash
# Destroy the partition table on a disk (before pvcreate on whole disk)
dd if=/dev/zero of=/dev/sdb bs=512 count=1
```

---

## Practical Recipes

### Recipe 1: Complete LVM Setup from Scratch

```bash
#!/bin/bash
set -e

DISK="/dev/sdb"
VG_NAME="vg_data"

# 1. Create partition for LVM
parted $DISK --script mklabel gpt
parted $DISK --script mkpart primary 1MiB 100%
parted $DISK --script set 1 lvm on

# 2. Create physical volume
pvcreate ${DISK}1

# 3. Create volume group
vgcreate $VG_NAME ${DISK}1

# 4. Create logical volumes
lvcreate -L 4G -n lv_swap $VG_NAME
lvcreate -L 20G -n lv_root $VG_NAME
lvcreate -l 100%FREE -n lv_home $VG_NAME

# 5. Create filesystems
mkswap /dev/$VG_NAME/lv_swap
mkfs.ext4 /dev/$VG_NAME/lv_root
mkfs.ext4 /dev/$VG_NAME/lv_home

# 6. Create mount points and mount
mkdir -p /mnt/{root,home}
swapon /dev/$VG_NAME/lv_swap
mount /dev/$VG_NAME/lv_root /mnt/root
mount /dev/$VG_NAME/lv_home /mnt/home

# 7. Add to /etc/fstab
echo "/dev/mapper/${VG_NAME}-lv_root  /      ext4 defaults 0 1" >> /etc/fstab
echo "/dev/mapper/${VG_NAME}-lv_home  /home  ext4 defaults 0 2" >> /etc/fstab
echo "/dev/mapper/${VG_NAME}-lv_swap  swap   swap defaults 0 0" >> /etc/fstab

echo "LVM setup complete!"
lvs $VG_NAME
```

### Recipe 2: Add Storage to Existing LVM

```bash
#!/bin/bash

NEW_DISK="/dev/sdc"
EXISTING_VG="vg_data"
LV_TO_EXTEND="lv_home"

# Prepare new disk
parted $NEW_DISK --script mklabel gpt
parted $NEW_DISK --script mkpart primary 1MiB 100%
parted $NEW_DISK --script set 1 lvm on

# Add to LVM
pvcreate ${NEW_DISK}1
vgextend $EXISTING_VG ${NEW_DISK}1

# Extend logical volume and filesystem
lvextend -l +100%FREE /dev/$EXISTING_VG/$LV_TO_EXTEND
resize2fs /dev/$EXISTING_VG/$LV_TO_EXTEND

echo "Storage added successfully!"
df -h | grep $LV_TO_EXTEND
```

### Recipe 3: Replace a Disk in a VG

```bash
# 1. Add new disk to VG
pvcreate /dev/sde1
vgextend vg_data /dev/sde1

# 2. Move data off old disk
pvmove /dev/sdb1 /dev/sde1

# 3. Remove old disk from VG
vgreduce vg_data /dev/sdb1

# 4. Remove PV label
pvremove /dev/sdb1

# 5. Physically remove the disk
```

### Recipe 4: LVM with Performance Optimization

```bash
#!/bin/bash

# Create striped LVM for database performance
DISKS="/dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1"
VG_NAME="vg_database"

# Create PVs
for disk in $DISKS; do
    pvcreate $disk
done

# Create VG with optimal extent size
vgcreate --physicalextentsize 64M $VG_NAME $DISKS

# Create striped LV for performance
lvcreate -L 100G -i 4 -I 1M -n lv_db_data $VG_NAME
lvcreate -L 20G -i 4 -I 256K -n lv_db_logs $VG_NAME

# Format with appropriate options
mkfs.ext4 -b 4096 -E stride=256,stripe-width=1024 /dev/$VG_NAME/lv_db_data
mkfs.ext4 -b 4096 /dev/$VG_NAME/lv_db_logs

echo "High-performance LVM setup complete!"
```

### Recipe 5: LVM Snapshot Backup Strategy

```bash
#!/bin/bash
# automated_lvm_backup.sh

VG="vg_data"
LV="lv_home"
SNAP_SIZE="10G"
BACKUP_BASE="/backup/lvm"
RETENTION_DAYS=7

DATE=$(date +%Y%m%d_%H%M%S)
SNAP_NAME="${LV}_snap_${DATE}"
BACKUP_DIR="$BACKUP_BASE/$DATE"

# Create snapshot
echo "Creating snapshot: $SNAP_NAME"
lvcreate -L $SNAP_SIZE -s -n $SNAP_NAME /dev/$VG/$LV

# Mount snapshot
mkdir -p /mnt/snap
mount /dev/$VG/$SNAP_NAME /mnt/snap

# Create backup
echo "Creating backup in: $BACKUP_DIR"
mkdir -p $BACKUP_DIR
rsync -avH --delete /mnt/snap/ $BACKUP_DIR/

# Cleanup snapshot
umount /mnt/snap
lvremove -f /dev/$VG/$SNAP_NAME

# Remove old backups
find $BACKUP_BASE -type d -name "20*" -mtime +$RETENTION_DAYS -exec rm -rf {} +

echo "Backup completed: $BACKUP_DIR"
```

### Recipe 6: Migrate LVM Between Servers

```bash
#!/bin/bash
# Migration process

# On source server:
# 1. Deactivate VG
vgchange -an vg_migrate

# 2. Export VG
vgexport vg_migrate

# 3. Physical disk move (shutdown both servers)

# On destination server:
# 4. Scan for new PVs
pvscan

# 5. Import VG
vgimport vg_migrate

# 6. Activate VG
vgchange -ay vg_migrate

# 7. Mount filesystems
lvscan
mount /dev/vg_migrate/lv_data /mnt/data
```

### Recipe 7: Extend Root LV (on a Running System)

```bash
# Check current space
df -h /
lvs /dev/mapper/rhel-root

# Extend the LV (no unmount needed for ext4/XFS)
lvextend -L +10G -r /dev/mapper/rhel-root

# Or use all available space
lvextend -l +100%FREE -r /dev/mapper/rhel-root

# Verify
df -h /
```

> Both ext4 and XFS support online (mounted) growth.

### Recipe 8: Extend a Logical Volume by Resizing the Disk

```bash
# After the underlying disk has been expanded (e.g., in a VM)
pvresize /dev/sdc
lvextend -L +10GB /dev/<vg_name>/<lv_name>
resize2fs /dev/<vg_name>/<lv_name>
df -h /<filesystem>
```

### Recipe 9: Shrink a Logical Volume

```bash
umount /<filesystem>
e2fsck -f /dev/<vg_name>/<lv_name>
lvreduce -r -L 10G /dev/<vg_name>/<lv_name>
e2fsck -f /dev/<vg_name>/<lv_name>
mount /<filesystem>
lvdisplay /dev/<vg_name>/<lv_name>
df -h /<filesystem>
```

### Recipe 10: Extend Swap Partition

```bash
swapoff -v /dev/vgroot/swap
lvresize -L +8GB /dev/vgroot/swap
mkswap /dev/vgroot/swap
swapon -va
swapon -s
```

### Recipe 11: Recover LVM from Corrupted Physical Volume

```bash
vgdisplay --partial --verbose
pvremove -ff /dev/sdc
pvcreate --uuid=xxxxxx /dev/sdc --restorefile=/etc/lvm/archive/<vg_name>.vg
vgcfgrestore -f /etc/lvm/archive/<vg_name>.vg <vg_name>
vgchange -ay <vg_name>
e2fsck /dev/<vg_name>/<lv_name>
```

> **Note:** This procedure will not restore any data lost from a physical volume that has failed and been replaced. If a physical volume has been partially overwritten (for example, the label or metadata regions have been damaged or destroyed) then user data may still exist in the data area of the volume and this may be recovered using standard tools after restoring access to the volume group using these steps.

### Recipe 12: Snapshot and Restore (Rescue Mode)

Make sure you have enough free space in VG.

```bash
# Make the snapshot (using all free space)
lvcreate -l 100%FREE -s -n lvrootsnap /dev/VolGroup/lv_root

# Boot in rescue mode with a basic shell (filesystems unmounted)

# Activate the VG
vgchange -ay VolGroup

# Restore the snapshot
lvconvert --merge /dev/VolGroup/lvrootsnap
```

---

## Monitoring and Maintenance

### Daily Monitoring Commands

```bash
# Quick status overview
vgs && pvs && lvs

# Check space usage
vgs -o vg_name,vg_free_percent
lvs -o lv_name,data_percent      # For thin pools
df -h                            # Filesystem usage

# Health checks
pvck /dev/sdb1                   # Check PV
vgck vg_data                     # Check VG
```

---

## Performance Tuning

### Optimal Settings

```bash
# Create performance-optimized LVM
# 1. Use appropriate extent sizes
vgcreate --physicalextentsize 64M vg_perf /dev/sdb1

# 2. Stripe across multiple devices
lvcreate -L 50G -i 4 -I 1024K -n lv_database vg_perf

# 3. Align with filesystem
mkfs.ext4 -E stride=256,stripe-width=1024 /dev/vg_perf/lv_database

# 4. Optimize mount options
mount -o noatime,data=writeback /dev/vg_perf/lv_database /opt/database
```

### Performance Monitoring

```bash
# Check stripe configuration
lvdisplay /dev/vg_data/lv_stripe | grep -E "Stripes|Stripe"

# Monitor I/O per PV
iostat -x 1 sdb sdc sdd

# Check device mapper statistics
dmsetup status
dmsetup info

# LVM-specific I/O stats
cat /proc/diskstats | grep dm-
```

---

## Integration Examples

### LVM with Docker

```bash
# Create Docker storage
vgcreate vg_docker /dev/sdb1
lvcreate -L 100G -n lv_docker vg_docker
mkfs.ext4 /dev/vg_docker/lv_docker
mount /dev/vg_docker/lv_docker /var/lib/docker

# Add to fstab
echo "/dev/vg_docker/lv_docker /var/lib/docker ext4 defaults 0 2" >> /etc/fstab
```

### LVM with Database

```bash
# MySQL/PostgreSQL optimized LVM
vgcreate vg_mysql /dev/sdb1 /dev/sdc1

# Separate data and logs
lvcreate -L 50G -i 2 -I 64K -n lv_mysql_data vg_mysql
lvcreate -L 10G -n lv_mysql_logs vg_mysql

# Format with database-optimized settings
mkfs.ext4 -b 4096 -E stride=16,stripe-width=32 /dev/vg_mysql/lv_mysql_data
mkfs.ext4 /dev/vg_mysql/lv_mysql_logs
```

### LVM with Virtualization

```bash
# VM storage pool with thin provisioning
vgcreate vg_vms /dev/sdb1 /dev/sdc1 /dev/sdd1
lvcreate -L 500G --thinpool vm_pool vg_vms

# Create VM disks
lvcreate -V 50G --thin vg_vms/vm_pool -n vm1_disk
lvcreate -V 100G --thin vg_vms/vm_pool -n vm2_disk
lvcreate -V 75G --thin vg_vms/vm_pool -n vm3_disk

# Monitor overcommitment
lvs -o lv_name,lv_size,data_percent vg_vms
```

---

## Troubleshooting

### VG Issues

```bash
# VG not found after reboot
vgscan --cache
vgscan --mknodes
vgchange -ay

# VG shows as partial
vgdisplay vg_name | grep -i partial
vgchange -ay --partial vg_name
vgreduce --removemissing vg_name

# Duplicate VG names
vgs -o vg_name,vg_uuid
vgrename <uuid> new_vg_name
```

### LV Issues

```bash
# LV not available
lvscan
lvchange -ay /dev/vg_data/lv_home

# LV shows as inactive
lvdisplay /dev/vg_data/lv_home | grep "LV Status"
lvchange -ay /dev/vg_data/lv_home

# Cannot remove LV (busy)
lsof | grep vg_data
fuser -v /dev/vg_data/lv_home
fuser -km /dev/vg_data/lv_home
lvremove /dev/vg_data/lv_home

# Check for mounts
mount | grep vg_data

# Check for swap
swapon --show
```

### PV Issues

```bash
# Duplicate PV UUIDs (after cloning disks)
pvs --all
pvchange --uuid /dev/sdc1

# pvmove interrupted or stuck
lvs -a | grep pvmove
pvmove /dev/sdb1             # Resume interrupted pvmove
pvmove --abort               # Abort stuck pvmove
```

### "Insufficient free space" When Extending

```bash
# Check free space in the VG
vgs -o vg_name,vg_free

# Check free space per PV
pvs -o pv_name,vg_name,pv_free

# Add another disk to the VG
pvcreate /dev/sde1
vgextend vg_data /dev/sde1
```

### Recovery Procedures

```bash
# Emergency VG activation
vgchange --sysinit -ay vg_data

# Recover from missing PV
vgdisplay --partial vg_data
vgreduce --removemissing --force vg_data

# Restore VG configuration
ls /etc/lvm/archive/vg_data*
vgcfgrestore -f /etc/lvm/archive/vg_data_00001.vg vg_data

# Force restore
vgcfgrestore --force vg_data
```

---

## Emergency Procedures

### Boot Issues with LVM

```bash
# From rescue environment
# 1. Scan for LVM
pvscan
vgscan
lvscan

# 2. Activate VGs
vgchange -ay

# 3. Mount root LV
mkdir /mnt/rescue
mount /dev/vg_system/lv_root /mnt/rescue

# 4. Chroot and fix
mount --bind /dev /mnt/rescue/dev
mount --bind /proc /mnt/rescue/proc
mount --bind /sys /mnt/rescue/sys
chroot /mnt/rescue
```

### Corrupted LVM Recovery

```bash
# Try automatic recovery
vgck vg_data
pvck /dev/sdb1

# Force scan and rebuild
vgscan --cache
pvscan --cache

# Restore from backup
vgcfgrestore -f /etc/lvm/backup/vg_data vg_data

# Last resort: rebuild metadata
vgcfgrestore --force vg_data
```

### Quick Emergency Reference

```bash
# Emergency activation
vgchange -ay                    # Activate all VGs
lvchange -ay /dev/vg/lv        # Activate specific LV
vgchange -ay --partial vg      # Activate with missing PVs

# Emergency info
lsblk                          # Show all block devices
pvs && vgs && lvs             # LVM overview
cat /proc/mdstat              # RAID status
df -h                         # Mounted filesystems

# Emergency repair
fsck -y /dev/vg/lv            # Fix filesystem
vgck vg_name                  # Check VG
pvck /dev/sdb1               # Check PV
```

---

## Backup Strategies

### Data Backup with Snapshots

```bash
#!/bin/bash
# Full backup strategy

VG="vg_production"
LVS="lv_database lv_web lv_logs"
BACKUP_ROOT="/backup"
DATE=$(date +%Y%m%d)

for lv in $LVS; do
    echo "Backing up $lv..."
    
    # Create snapshot
    lvcreate -L 5G -s -n ${lv}_snap /dev/$VG/$lv
    
    # Mount and backup
    mkdir -p /mnt/${lv}_snap
    mount /dev/$VG/${lv}_snap /mnt/${lv}_snap
    
    rsync -avH /mnt/${lv}_snap/ $BACKUP_ROOT/$DATE/$lv/
    
    # Cleanup
    umount /mnt/${lv}_snap
    lvremove -f /dev/$VG/${lv}_snap
    
    echo "$lv backup completed"
done
```

---

## Complete Command Reference

### All PV Commands

```bash
pvcreate <device>               # Create physical volume
pvremove <device>               # Remove physical volume
pvdisplay [device]              # Display PV information
pvs [options]                   # List PVs
pvscan                          # Scan for PVs
pvck <device>                   # Check PV
pvmove <source> [dest]          # Move extents
pvresize <device>               # Resize PV
pvchange -x [y|n] <device>      # Change allocation permission
```

### All VG Commands

```bash
vgcreate <vg_name> <pv>         # Create volume group
vgremove <vg_name>              # Remove volume group
vgdisplay [vg_name]             # Display VG information
vgs [options]                   # List VGs
vgscan                          # Scan for VGs
vgck <vg_name>                  # Check VG
vgextend <vg_name> <pv>         # Extend VG
vgreduce <vg_name> <pv>         # Reduce VG
vgrename <old> <new>            # Rename VG
vgchange -ay <vg_name>          # Activate VG
vgchange -an <vg_name>          # Deactivate VG
vgsplit <source> <dest> <pv>    # Split VG
vgmerge <dest> <source>         # Merge VGs
vgexport <vg_name>              # Export VG
vgimport <vg_name>              # Import VG
vgcfgbackup [vg_name]          # Backup VG config
vgcfgrestore <vg_name>          # Restore VG config
```

### All LV Commands

```bash
lvcreate [options] <vg_name>    # Create logical volume
lvremove <lv_path>              # Remove logical volume
lvdisplay [lv_path]             # Display LV information
lvs [options]                   # List LVs
lvscan                          # Scan for LVs
lvextend [options] <lv_path>    # Extend LV
lvreduce [options] <lv_path>    # Reduce LV (dangerous)
lvrename <vg> <old> <new>       # Rename LV
lvchange [options] <lv_path>    # Change LV attributes
lvconvert [options] <lv_path>   # Convert LV type
lvresize [options] <lv_path>    # Resize LV
```

---

## LVM Commands Naming Convention

All LVM commands follow a consistent pattern:

| Prefix | Layer | Commands |
|--------|-------|----------|
| `pv*` | Physical Volume | `pvcreate`, `pvdisplay`, `pvs`, `pvremove`, `pvmove`, `pvresize`, `pvscan`, `pvck`, `pvchange` |
| `vg*` | Volume Group | `vgcreate`, `vgdisplay`, `vgs`, `vgremove`, `vgextend`, `vgreduce`, `vgrename`, `vgscan`, `vgchange`, `vgexport`, `vgimport`, `vgck`, `vgsplit`, `vgmerge`, `vgcfgbackup`, `vgcfgrestore` |
| `lv*` | Logical Volume | `lvcreate`, `lvdisplay`, `lvs`, `lvremove`, `lvextend`, `lvreduce`, `lvrename`, `lvscan`, `lvchange`, `lvconvert`, `lvresize` |

---

## Quick Reference

### Most Common Daily Commands

```bash
# Status check
pvs && vgs && lvs              # Overview of all LVM components
df -h                          # Check filesystem usage
lsblk                          # View block device tree

# Create new LV
lvcreate -L 10G -n lv_new vg_data
mkfs.ext4 /dev/vg_data/lv_new

# Extend existing LV
lvextend -L +5G -r /dev/vg_data/lv_home

# Create snapshot for backup
lvcreate -L 2G -s -n backup_snap /dev/vg_data/lv_important

# Add storage to VG
pvcreate /dev/sdc1
vgextend vg_data /dev/sdc1
```

### Quick Reference Table

| Command | Description |
|---------|-------------|
| `pvs` | List physical volumes |
| `vgs` | List volume groups |
| `lvs` | List logical volumes |
| `pvdisplay` | Detailed PV info |
| `vgdisplay` | Detailed VG info |
| `lvdisplay` | Detailed LV info |
| `pvcreate /dev/sdb1` | Initialize a PV |
| `vgcreate vg_name /dev/sdb1` | Create a VG |
| `lvcreate -L 50G -n lv_name vg_name` | Create a 50G LV |
| `lvcreate -l 100%FREE -n lv_name vg_name` | Create LV using all free space |
| `lvextend -L +10G -r /dev/vg/lv` | Extend LV by 10G and resize FS |
| `lvextend -l +100%FREE -r /dev/vg/lv` | Extend LV to fill VG and resize FS |
| `lvreduce -L 40G -r /dev/vg/lv` | Shrink LV to 40G (unmount first) |
| `lvremove /dev/vg/lv` | Remove a logical volume |
| `vgextend vg_name /dev/sdc1` | Add PV to VG |
| `vgreduce vg_name /dev/sdb1` | Remove PV from VG |
| `pvmove /dev/sdb1` | Move data off a PV |
| `lvcreate -L 5G -s -n snap /dev/vg/lv` | Create snapshot |
| `lvconvert --merge /dev/vg/snap` | Revert to snapshot |
| `vgchange -ay vg_name` | Activate VG |
| `vgchange -an vg_name` | Deactivate VG |
| `vgexport vg_name` | Export VG (for disk migration) |
| `vgimport vg_name` | Import VG |
| `vgsplit vg_source vg_dest /dev/pv` | Split VG |
| `vgmerge vg_dest vg_source` | Merge VGs |
| `lvrename vg lv_old lv_new` | Rename a LV |
| `vgrename vg_old vg_new` | Rename a VG |

---

## Best Practices Summary

### Design Guidelines

1. **Naming**: Use descriptive VG and LV names (`vg_database`, `lv_mysql_data`)
2. **Sizing**: Leave 20-30% free space in VGs for snapshots and growth
3. **Extent Size**: Use 32MB or 64MB for large VGs (>1TB)
4. **Redundancy**: Use RAID or mirroring for critical data
5. **Performance**: Stripe across multiple physical devices

### Operational Best Practices

1. **Monitoring**: Regular health checks and space monitoring
2. **Backups**: Automate snapshot-based backups
3. **Documentation**: Maintain mapping of LVs to applications
4. **Testing**: Regular disaster recovery testing
5. **Updates**: Keep LVM tools and kernel updated

---

## Files

| Path | Description |
|------|-------------|
| `/etc/lvm/lvm.conf` | Main LVM configuration file |
| `/etc/lvm/backup/` | Latest metadata backup per VG |
| `/etc/lvm/archive/` | Historical metadata backups |
| `/etc/lvm/cache/.cache` | Metadata map (device cache) |

---

## Safety Warnings

1. **Always backup data** before reducing or removing logical volumes
2. **XFS cannot be shrunk** — only ext4 (and ext2/ext3) supports filesystem reduction
3. **Shrink the filesystem before shrinking the LV** — or use `-r` to do it atomically
4. **Never shrink an LV below the filesystem size** — this destroys data
5. **Monitor thin pool usage** — thin volumes fail when the pool is full
6. **pvmove can take hours** on large volumes — run in `screen` or `tmux`
7. **Snapshots degrade performance** and can fill up — remove them when no longer needed
8. **Test in non-production first** — LV removal and reduction are destructive operations

Remember: Always backup data and test procedures in non-production environments first!
