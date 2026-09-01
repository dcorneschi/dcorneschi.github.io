# AIX Storage Provisioning Tasks

Task-oriented recipes for day-to-day storage work on IBM AIX — discovering and adding a new LUN, creating logical volumes and JFS2 filesystems, spanning a filesystem across LUNs, mapping disks through a VIOS to an LPAR, extending paging space, EMC/HDS BCV clone workflows, and copying or swapping logical volumes.

> These are real-world provisioning sequences (SAP/Oracle layouts, EMC PowerPath `hdiskpowerNN` devices). Adjust sizes, disk names, mount points, and VG names to your environment. Anything that changes reservations, mirrors, or BCV state is high-impact — verify disk identity (PVID/serial) before running and take a backup before destructive steps. See the [AIX LVM Cheatsheet](articles/aix-lvm-cheatsheet.md) for the underlying VG/LV/PV command reference.

## Add a New LUN

Snapshot the disk list before and after `cfgmgr` to identify the newly discovered device:

```sh
lspv > /tmp/lspv
cfgmgr
lspv > /tmp/lspv2
diff /tmp/lspv /tmp/lspv2
```

Prepare the disk, add it to a VG, and create the LV and filesystem:

```sh
# Assign a PVID and disable SCSI reservation
chdev -l hdiskpower23 -a pv=yes
chdev -l hdiskpower23 -a reserve_lock=no

# Add the disk to the volume group
extendvg asevg hdiskpower23

# Create a JFS2 logical volume (536 LPs, max 1200 LPs) on that disk
/usr/sbin/mklv -y ASEdata9 -t 'jfs2' -x '1200' asevg 536 hdiskpower23

# Create the filesystem on the LV (not auto-mounted, rw, 4K block size)
/usr/sbin/crfs -v jfs2 -d ASEdata9 -m /oracle/ASE/sapdata9 -A 'no' -p 'rw' -a agblksize='4096'
```

### With a mirror copy

```sh
/usr/sbin/mklv -y data12GP1 -t 'jfs2' -x '1200' gp1vg 268 hdiskpower44
/usr/sbin/crfs -v jfs2 -d data12GP1 -m /oracle/GP1/sapdata12 -A 'no' -p 'rw' -a agblksize='4096'
mount /oracle/GP1/sapdata12

# Add a second copy on another disk and synchronize
mklvcopy data13GP1 2 hdiskpower42
syncvg -l data12GP1 &
```

## Spanning a Filesystem Across Two LUNs

Extend the underlying LV onto an additional disk, then grow the filesystem:

```sh
# Extend the LV by 32 LPs onto another disk
/usr/sbin/extendlv transferHCP 32 hdiskpower0

# Grow the filesystem to the new size
chfs -a size=XG /filesystem
```

## Add a LUN Mapped Through a VIOS

### On the VIOS

```sh
# Drop to the OEM (root) shell from the padmin restricted shell
oem_setup_env    # 'r oem' in some environments

lspv > /tmp/lun
cfgmgr
lspv > /tmp/lun2
diff /tmp/lun /tmp/lun2

chdev -l hdiskpower51 -a pv=yes
chdev -l hdiskpower51 -a reserve_lock=no

# Map the backing device to a virtual host adapter with a stable name
mkvdev -vdev hdiskpower51 -vadapter vhost7 -dev swp_38disk11
```

### On the LPAR (VIO client)

```sh
lspv > /tmp/lun
cfgmgr
lspv > /tmp/lun2
diff /tmp/lun /tmp/lun2

chdev -l hdisk15 -a queue_depth=32
extendvg swpvg hdisk15

/usr/sbin/mklv -y data8SWP -t 'jfs2' -x '538' swpvg 538 hdisk15
/usr/sbin/crfs -v jfs2 -d data8SWP -m /oracle/SWP/sapdata8 -A 'no' -p 'rw' -a agblksize='4096'
mount /oracle/SWP/sapdata8
```

## Extend Paging Space

```sh
# Grow the default paging space (hd6) by 76 PPs
chps -s 76 hd6

# Add disks to rootvg, then create/mirror a paging space
extendvg rootvg hdisk2 hdisk3
mkps -a -n -s 80 rootvg hdisk2
mklvcopy paging00 2 hdisk3
syncvg -l paging00
```

See the [AIX Paging Space Cheatsheet](articles/aix-paging-space-cheatsheet.md) for full paging-space management.

## EMC/HDS BCV Clone Workflow

Business Continuance Volumes (BCV) create point-in-time clones on EMC Symmetrix arrays using the SYMCLI tools, mirrored to an AIX volume group.

### OS side

1. Run `cfgmgr` on all three servers to discover the new disks.
2. Create the `MIPsapdata` device-mapping file under `/root/sys*/bin`.
3. Generate the VG/FS creation scripts:
   ```sh
   sh create_FS_for_HDS_EMC_hdisk-plus-mkvg-plus-log <MIPdata-file> mip2vg
   ```
4. On the source hosts (230x): set `reserve_lock=no` and `pv=yes` on the new `hdiskpower` disks.
5. On the target hosts (276x): set `reserve_lock=no` on the new `hdiskpower` disks.
6. Run the generated `mklv`, `mkfs`, `mklvcopy`, and `syncvg` scripts from the `create_fs` directory.

### BCV / SYMCLI side

```sh
# Discover Symmetrix config on all three servers
symcfg discover

# Add the standard (source) device to the device group (all servers)
symld -sid 059 -g MIPVG add dev 109C DEV_MIPdata35

# Add the BCV (clone) device to the group (all servers)
symbcv -sid 059 -g MIPVG add dev 109D BCV_MIPdata35

# Establish a full mirror (on ONE server only)
symmir -g MIPVG -full establish DEV_MIPdata35 BCV ld BCV_MIPdata35

# Once synced (check with the query), split the pair
symmir -g MIPVG query
symmir -g MIPVG split DEV_MIPdata35

# On the target hosts (276x), assign a PVID to the split BCV
chdev -l hdiskpower -a pv=yes

# Export the device group definition (all servers)
symdg export MIPVG -f /root/sys_sich/bin/BCV/BCV-MIPVG-export.txt
```

## Copy a Logical Volume (same VG)

Unmount the filesystems, mark the target as a copy type, then copy:

```sh
umount /opt/pluto
umount /opt/pluto_bak
chlv -t copy fslv01
cplv -e fslv01 -f fslv00
mount /opt/pluto
mount /opt/pluto_bak
```

## Swap Filesystem Mount Points

Reassign mount points between logical volumes with `chfs -m`:

```sh
chfs -m /opt/pluto_err /opt/pluto
chfs -m /opt/pluto /opt/pluto_bak
mount /opt/pluto_err
mount /opt/pluto
```

## Copy a Logical Volume to a Different VG

`cplv` can target another VG; create and format a JFS2 log there as well:

```sh
umount /opt/pluto
cplv -v apps_vg -y pluto_lv fslv00
mklv -t jfs2log -y jfslog_lv apps_vg 1
logform /dev/jfslog_lv
```

## Quick Reference

| Task | Key commands |
|------|--------------|
| Discover new disk | `lspv` → `cfgmgr` → `lspv` → `diff` |
| Prepare a LUN | `chdev -l <disk> -a pv=yes -a reserve_lock=no` |
| Add disk to VG | `extendvg <vg> <disk>` |
| Create LV | `mklv -y <lv> -t jfs2 -x <maxlp> <vg> <lps> <disk>` |
| Create filesystem | `crfs -v jfs2 -d <lv> -m <mount> -p rw -a agblksize=4096` |
| Grow LV / filesystem | `extendlv <lv> <lps> <disk>` ; `chfs -a size=XG <fs>` |
| Mirror an LV | `mklvcopy <lv> 2 <disk>` ; `syncvg -l <lv>` |
| Map disk on VIOS | `mkvdev -vdev <disk> -vadapter vhost<n> -dev <name>` |
| Extend paging | `chps -s <pps> hd6` |
| Copy an LV | `cplv -e <target> -f <source>` |
| Copy LV to another VG | `cplv -v <vg> -y <lv> <source>` |
| Change mount point | `chfs -m <newmount> <fs>` |
