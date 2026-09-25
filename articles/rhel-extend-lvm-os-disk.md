# Extending an LVM Logical Volume by Growing the OS Disk on RHEL

When an LVM-backed filesystem on the OS disk fills up on a RHEL virtual machine (VMware, KVM, or cloud), the fix is a chain: grow the virtual disk, grow the partition, grow the physical volume, grow the logical volume, then grow the filesystem. This guide walks the full path end to end, including the reboot caveat that trips people up when the disk is in use.

There are two ways to reclaim the newly added space on an existing partition layout:

- **Delete and recreate the partition** (e.g. `/dev/sda2`) at a larger size — covered in detail below.
- **Create a new partition** (e.g. `/dev/sda3`) and add it to the volume group as a second PV — a lower-risk alternative when you'd rather not touch the existing partition.

> Take a snapshot or backup before editing the partition table. Deleting and recreating a partition is safe *only* if the new partition starts on the exact same sector; a wrong start sector destroys data. On modern systems, [`growpart`](articles/growpart-extend-partitions.md) avoids the manual delete/recreate dance entirely — prefer it when available.

## The Full Chain

```
Grow virtual disk (hypervisor/cloud)
        ↓
Rescan disk on the guest
        ↓
Grow the partition (fdisk: delete + recreate, or growpart)
        ↓
pvresize        ← grow the physical volume to fill the partition
        ↓
lvextend        ← grow the logical volume from free VG space
        ↓
resize2fs / xfs_growfs   ← grow the filesystem to fill the LV
```

Each layer must be grown in order, because each one sits inside the one above it.

## Option A — Delete and Recreate /dev/sda2

### 1. Extend the Virtual Disk

Grow the disk in the VMware vSphere Client (or via CLI / your cloud console). The guest won't see the new size until you rescan.

### 2. Take a Snapshot

Snapshot the VM (or back up the data) before touching the partition table. This is your rollback path.

### 3. Rescan the Disk

Tell the kernel to re-read the disk geometry without a reboot:

```sh
echo 1 > /sys/block/sda/device/rescan
```

### 4. Confirm the New Size

```sh
lsblk
fdisk -l /dev/sda
```

The disk should now report the larger capacity even though the partition is still the old size.

### 5. Delete and Recreate the Partition

Open `fdisk` on the whole disk:

```sh
fdisk /dev/sda
```

Inside `fdisk`, delete partition 2 and recreate it larger, starting on the **same first sector**, then set the LVM type. The classic keystroke sequence (older `fdisk` with DOS-compatibility toggles):

```text
c        # turn off DOS compatibility
u        # switch units to sectors
d        # delete a partition
2        #   partition 2
n        # new partition
p        #   primary
2        #   number 2
<Enter>  #   accept default first sector (MUST match the original start)
<Enter>  #   accept default last sector (use full disk)
t        # change partition type
2        #   partition 2
8e       #   Linux LVM
p        # print the table to verify start sector is unchanged
w        # write and exit
```

> The single most important step is that the recreated partition's **start sector matches the original**. Print the table with `p` before writing and confirm the start sector is identical. If it differs, quit with `q` (no changes written) and reassess.

### 6. Reboot

`partprobe` or `kpartx -v -a /dev/sda` cannot re-read the partition table while the disk holds the mounted root/LVM in use, so the kernel keeps the old table until reboot:

```sh
reboot
```

### 7. Verify the Partition Picked Up the New Size

```sh
cat /proc/partitions | grep sda2
lsblk /dev/sda
```

### 8. Grow the Physical Volume

`pvresize` expands the PV to fill the enlarged partition:

```sh
pvresize /dev/sda2
pvs
```

`pvs` should now show increased free space (PFree) in the volume group.

### 9. Extend the Logical Volume

Add space to the target LV. Use `-L +<size>` for an absolute amount, or `-l +100%FREE` to consume all free VG space:

```sh
lvextend -L +10G /dev/<vg_name>/<lv_name>
# or take everything that's free:
lvextend -l +100%FREE /dev/<vg_name>/<lv_name>
lvs
```

> `lvextend -r` grows the filesystem in the same command (via `fsadm`, which in turn calls `resize2fs`/`xfs_growfs` for you), which lets you combine steps 9 and 10: `lvextend -r -l +100%FREE /dev/vg/lv`.

### 10. Grow the Filesystem

Finally, extend the filesystem to fill the enlarged LV. The command depends on the filesystem type:

```sh
# ext3 / ext4
resize2fs /dev/<vg_name>/<lv_name>

# xfs (grows online, must be mounted; takes the mount point or device)
xfs_growfs /dev/<vg_name>/<lv_name>
```

Confirm the new size:

```sh
df -h <filesystem>
```

## Option B — Add a New Partition Instead

If you'd rather not delete and recreate `/dev/sda2`, create a **new** partition in the free space and add it to the volume group. This avoids the risky partition-recreate step:

```sh
# After rescanning the disk (steps 1–4 above)
fdisk /dev/sda        # create /dev/sda3 as type 8e (Linux LVM)
partprobe /dev/sda    # or reboot if the kernel won't re-read the table

pvcreate /dev/sda3            # initialize the new PV
vgextend <vg_name> /dev/sda3  # add it to the volume group
lvextend -r -l +100%FREE /dev/<vg_name>/<lv_name>   # extend LV + FS
```

This is generally safer because it never touches the existing partition, though it does leave the VG spread across multiple partitions on the same disk.

## Key Takeaways

- Growth is a layered chain: disk → partition → PV → LV → filesystem, in that order.
- Rescan with `echo 1 > /sys/block/sda/device/rescan` to see new capacity without a reboot.
- When recreating a partition, the **start sector must be identical** — verify with `p` before writing.
- A partition change on an in-use disk needs a **reboot**; `partprobe`/`kpartx` can't re-read it live.
- `pvresize` then `lvextend` then `resize2fs`/`xfs_growfs` (or `lvextend -r` to combine the last two).
- Prefer [`growpart`](articles/growpart-extend-partitions.md) over manual delete/recreate when it's available, and always snapshot first.

## Quick Reference

```sh
# 1. Rescan after growing the disk in the hypervisor
echo 1 > /sys/block/sda/device/rescan
lsblk

# 2. Recreate the partition larger (same start sector!) — or use growpart
growpart /dev/sda 2          # modern alternative to the fdisk dance

# 3. Reboot if the disk is in use, then grow the stack
pvresize /dev/sda2
lvextend -r -l +100%FREE /dev/<vg_name>/<lv_name>   # extends LV and FS together

# Manual filesystem grow if not using -r
resize2fs /dev/<vg_name>/<lv_name>     # ext4
xfs_growfs /dev/<vg_name>/<lv_name>    # xfs

df -h
```

For related material, see [Extending Partitions with growpart](articles/growpart-extend-partitions.md), the [LVM Cheatsheet](articles/lvm-cheatsheet.md), the [fdisk Cheatsheet](articles/fdisk-cheatsheet.md), and [Extend a SAN LUN Online with Multipath and GFS2](articles/linux-extend-lun-multipath-gfs2.md).
