# Extend a SAN LUN Online with Multipath and GFS2

How to expand a SAN-attached LUN on a Linux cluster using multipath, LVM, and GFS2 without downtime.

## Prerequisites

- Ensure the cluster is healthy before starting (`clustat`, all nodes online)
- Have a current backup of the data on the filesystem
- Confirm the LUN extension has been completed on the storage array
- Know the mpath device name on **each** node (they may differ)
- This procedure is performed online — no unmount required

> **Warning:** GFS2 filesystems cannot be shrunk. This operation is non-reversible.

## Increase the Size of LUN on the Storage Array

After extending the LUN on the storage side, the OS should detect the change. Verify with `dmesg`:

```sh
dmesg
```

You should see messages like:

```
sd 0:0:1:3: Warning! Received an indication that the operating parameters on this target have changed. The Linux SCSI layer does not automatically adjust these parameters.
sd 3:0:0:3: Warning! Received an indication that the operating parameters on this target have changed. The Linux SCSI layer does not automatically adjust these parameters.
sd 3:0:1:3: Warning! Received an indication that the operating parameters on this target have changed. The Linux SCSI layer does not automatically adjust these parameters.
sd 0:0:0:3: Warning! Received an indication that the operating parameters on this target have changed. The Linux SCSI layer does not automatically adjust these parameters.
```

## On Every Node in the Cluster

### Find the LUN's Paths

Identify the underlying SCSI devices for the multipath device:

```sh
multipath -l /dev/mapper/mpathd
```

```
mpathd (3600601600a20330030f0ff07f8dce211) dm-4 DGC,RAID 10
size=400G features='1 retain_attached_hw_handler' hwhandler='1 alua' wp=rw
|-+- policy='round-robin 0' prio=0 status=active
| |- 0:0:1:3 sdh 8:112 active undef running
| `- 3:0:1:3 sdq 65:0  active undef running
`-+- policy='round-robin 0' prio=0 status=enabled
  |- 0:0:0:3 sdd 8:48  active undef running
  `- 3:0:0:3 sdm 8:192 active undef running
```

### Rescan SCSI Paths for the LUN

Trigger a rescan on each path device:

```sh
echo 1 > /sys/block/sdh/device/rescan
echo 1 > /sys/block/sdq/device/rescan
echo 1 > /sys/block/sdd/device/rescan
echo 1 > /sys/block/sdm/device/rescan
```

Confirm the new capacity is detected:

```sh
less /var/log/messages
```

```
Jan 16 03:09:50 localhost kernel: sd 0:0:1:3: [sdh] 1384120320 512-byte logical blocks: (708 GB/660 GiB)
Jan 16 03:09:50 localhost kernel: sdh: detected capacity change from 429496729600 to 708669603840
Jan 16 03:09:57 localhost kernel: sd 3:0:1:3: [sdq] 1384120320 512-byte logical blocks: (708 GB/660 GiB)
Jan 16 03:09:57 localhost kernel: sdq: detected capacity change from 429496729600 to 708669603840
Jan 16 03:10:06 localhost kernel: sd 0:0:0:3: [sdd] 1384120320 512-byte logical blocks: (708 GB/660 GiB)
Jan 16 03:10:06 localhost kernel: sdd: detected capacity change from 429496729600 to 708669603840
Jan 16 03:10:13 localhost kernel: sd 3:0:0:3: [sdm] 1384120320 512-byte logical blocks: (708 GB/660 GiB)
Jan 16 03:10:13 localhost kernel: sdm: detected capacity change from 429496729600 to 708669603840
```

### Resize the Multipath Device

> The mpath device name might differ between cluster nodes. Make sure you resize the correct device on each node.

```sh
multipathd -k"resize map mpathd"
multipathd -k"resize map mpathb"
```

Verify the new size is reflected:

```sh
multipath -l /dev/mapper/mpathd
```

Check the logs:

```sh
less /var/log/messages
```

```
Jan 16 03:13:35 localhost multipathd: mpathd: resize map (operator)
Jan 16 03:13:35 localhost multipathd: mpathd: load table [0 1384120320 multipath 1 retain_attached_hw_handler 1 alua 2 1 round-robin 0 2 1 8:112 100 65:0 100 round-robin 0 2 1 8:48 100 8:192 100]
Jan 16 03:13:35 localhost kernel: sd 0:0:1:3: alua: port group 02 state A preferred supports tolUsNA
Jan 16 03:13:35 localhost kernel: sd 3:0:1:3: alua: port group 02 state A preferred supports tolUsNA
```

## On the First Node Only

### Resize the Physical Volume

```sh
pvresize /dev/mapper/mpathd
```

Verify the PV reflects the new size:

```sh
pvs /dev/mapper/mpathd
```

### Extend the Logical Volume

Use all available free space:

```sh
lvextend -l +100%FREE /dev/<vg>/<lv>
```

Verify the LV size:

```sh
lvs /dev/<vg>/<lv>
```

### Grow the GFS2 File System

`gfs2_grow` can be run online and is cluster-aware — all nodes will see the new size immediately.

```sh
gfs2_grow /apps
```

```
FS: Mount Point: /apps
FS: Device:      /dev/dm-11
FS: Size:        104854775 (0x63ff4f7)
FS: RG size:     65533 (0xfffd)
DEV: Size:       173014016 (0xa4ffc00)
The file system grew by 266247MB.
gfs2_grow complete.
```

Log output:

```
Jan 16 03:16:12 localhost kernel: GFS2: fsid=gfs2cls:gfs2pv.1: File system extended by 65528 blocks.
Jan 16 03:16:12 localhost kernel: GFS2: fsid=gfs2cls:gfs2pv.1: File system extended by 1310560 blocks.
Jan 16 03:16:12 localhost kernel: GFS2: fsid=gfs2cls:gfs2pv.1: File system extended by 2817704 blocks.
Jan 16 03:16:12 localhost kernel: GFS2: fsid=gfs2cls:gfs2pv.1: File system extended by 2752176 blocks.
Jan 16 03:16:12 localhost kernel: GFS2: fsid=gfs2cls:gfs2pv.1: File system extended by 2817704 blocks.
Jan 16 03:16:12 localhost kernel: GFS2: fsid=gfs2cls:gfs2pv.1: File system extended by 2817704 blocks.
Jan 16 03:16:12 localhost kernel: GFS2: fsid=gfs2cls:gfs2pv.1: File system extended by 2752176 blocks.
Jan 16 03:16:12 localhost kernel: GFS2: fsid=gfs2cls:gfs2pv.1: File system extended by 2817704 blocks.
```

### Verify the New Size

Confirm the new filesystem size on both nodes:

```sh
df -h /apps
```

## Notes

- `gfs2_grow` only needs to run on one node; all other nodes see the change automatically
- GFS2 does not support shrinking — plan capacity carefully
- If `pvresize` does not pick up the new size, verify the multipath device was resized correctly with `multipath -l`
- Always confirm each step before proceeding to the next — especially that all SCSI paths report the same new capacity
