# HP-UX Filesystem Management (HFS, JFS/VxFS)

Managing filesystems on HP-UX — the HFS vs JFS/VxFS (Base vs Online) differences, creating and mounting filesystems, defragmentation, repair, space reclamation, growing and shrinking across the LVM stack (VG → LV → FS), and the VxFS `ioerror` policies. Covers HP-UX 11i v1–v3. For the underlying volume manager, see [HP-UX LVM](articles/hpux-lvm.md).

## Concepts: How HP-UX Filesystems Fit Together

An HP-UX filesystem never lives directly on a raw disk in production; it sits on top of a logical volume, which is carved out of a volume group built from one or more physical volumes. Understanding that stack — **PV → VG → LV → FS** — is the key to every grow/shrink operation later in this article. The filesystem only knows about the block device handed to it (the logical volume); it neither knows nor cares how many physical disks back that device. That separation is exactly what lets LVM add disks, mirror, and stripe underneath a live filesystem without the filesystem noticing.

Two filesystem families dominate HP-UX. **HFS** (the High-performance FileSystem, HP's Berkeley FFS derivative) is the legacy local filesystem — it is what the boot area (`/stand`) still uses, because the boot loader understands HFS but not VxFS on older releases. **JFS** is HP's branding of the VERITAS **VxFS** journaling filesystem, and it is the default for essentially all data filesystems from 11i onward. The reason VxFS won out is journaling: it keeps an **intent log** of pending metadata changes, so after a crash it replays that log in seconds instead of walking the entire filesystem with `fsck`. HFS has no journal, so an HFS `fsck` after an unclean shutdown scales with filesystem size and can take a very long time on large volumes.

### The intent log and why it matters

Every VxFS filesystem reserves a small circular **intent log** area. Before VxFS changes on-disk metadata (allocating an inode, extending a file, updating a directory), it records the intended change in the log. If power is lost mid-operation, the next mount replays the log to bring metadata to a consistent state. This is why the fast form of `fsck` (log replay) is nearly instantaneous, and why the `nolog` option forces the slow, thorough structural check instead. A larger `logsize` improves metadata-heavy workload throughput at a small space cost, which is why you sometimes set it explicitly at `mkfs` time.

## HFS vs JFS/VxFS

**JFS** on HP-UX is VERITAS **VxFS**, and comes in two flavors: **Base JFS** (bundled) and **Online JFS** (a licensed add-on). The distinction matters constantly in day-to-day administration: almost every "online" operation you want to perform — growing a mounted filesystem, shrinking it, defragmenting it — requires the **Online JFS** license (product `OnlineJFS`, sometimes called Advanced VxFS). Without it you have plain Base JFS, which still journals and performs well but forces you to unmount the filesystem for any structural change. Confirm which one you own before planning a change that assumes online resize. The key capability differences:

| Capability | HFS | Base JFS | Online JFS |
|-----------|-----|----------|------------|
| Reduce filesystem | Unmount, then recreate | Must unmount | **Online** (mounted) via `fsadm` |
| Defragmentation | No utility | No | **Yes** (`fsadm`) |
| Online extend | No | No | **Yes** (`fsadm -b`) |
| Convert HFS → JFS | — | `vxfsconvert` (JFS 3.3+) | `vxfsconvert` |

- HFS provides **no** mechanism to reduce a filesystem online or offline (you recreate it); it has **no** defrag utility.
- Only **Online JFS** can extend/reduce/defragment a **mounted** filesystem; Base JFS still requires an unmount for those.
- `vxfsconvert` performs an **in-place** HFS-to-JFS conversion (the data is not copied elsewhere), so it needs free space inside the filesystem for the new VxFS metadata and is inherently risky — always have a verified backup first.

A practical rule of thumb: keep `/stand` as HFS (the boot loader depends on it), and make everything else VxFS. On 11i v3, `/stand` may be VxFS on newer hardware, but do not convert it yourself without checking the boot firmware's support.

```bash
# Filesystem type
mount -v                          # all mounts with types
fstyp /dev/vg00/rlvol1            # type of one FS (raw device)
fstyp -v /dev/vg00/rlvol3        # verbose
fstyp -l                          # all supported FS types

# Do you have Online JFS?
swlist -l product OnlineJFS*
```

## Creating a Filesystem (newfs / mkfs)

The default FS type comes from `/etc/default/fs` — check that file before running a bare `newfs` so you know whether you are getting HFS or VxFS. `newfs` is really a front-end that calls `mkfs` with sensible defaults; use `newfs` for quick creation and `mkfs` when you need to control low-level VxFS layout parameters (block size, intent-log size, inode size, layout version). Always point these commands at the **raw/character** device (`/dev/vg01/rdatavol`, with the leading `r`), not the block device — writing filesystem metadata is a raw-device operation.

A common gotcha: if you forget `-o largefiles`, the filesystem caps individual files at 2 GB minus one block. You will not notice until an application (a database, a large backup, a log) tries to cross that boundary and fails with EFBIG. You can flip the flag later on a mounted VxFS filesystem with `fsadm -F vxfs -o largefiles`, so this is recoverable, but it is cleaner to set it at creation time for any data filesystem.

```bash
newfs /dev/vg01/rdatavol          # default type
newfs -F hfs  /dev/rdisk/disk1
newfs -F vxfs /dev/rdisk/disk1
```

Common `newfs` options (both types):

| Option | Meaning |
|--------|---------|
| `-F hfs\|vxfs` | Filesystem type |
| `-o largefiles` | Allow files > 2 GB (toggle later on a mounted FS with `fsadm`) |
| `-s <size>` | Size in blocks (or k/m/g); default uses all available space |
| `-b <block-size>` | Block size (HFS default 8192 bytes) |
| `-R <mb>` | Reserve space at end of disk for swap |

```bash
# Reserve 200 MB at end of disk for swap
newfs -F hfs  -R 200 /dev/rdisk/disk1
newfs -F vxfs -R 200 /dev/rdisk/disk1

# Show a VxFS FS's layout attributes
mkfs -F vxfs -m /dev/vg00/lvol3

# Create VxFS with explicit options and a 2 MB intent log (16384 = size in blocks)
mkfs -F vxfs -o ninode=unlimited,bsize=1024,version=6,inosize=256,logsize=2048,largefiles \
     /dev/vg01/rdatavol 16384
```

Upgrade a VxFS filesystem's on-disk layout version:

```bash
vxupgrade -n 7 /data              # upgrade metadata to layout version 7
```

## Mounting Filesystems

Mounting attaches a filesystem to a directory (the *mount point*) in the existing tree and records it in the kernel mount table (`/etc/mnttab`, maintained automatically — do not edit it by hand). The *persistent* configuration lives in `/etc/fstab`; entries there are what `mount -a`, `mountall`, and the boot-time `localmount` script consult. A frequent mistake is mounting by hand and forgetting to add the `/etc/fstab` entry, so the filesystem silently disappears after the next reboot. Conversely, a bad `/etc/fstab` entry can stall the boot, so validate new entries with `mount <mountpoint>` (which reads options from `/etc/fstab`) before rebooting.

The block device is used for the mount; the mount point directory must already exist and should be empty (anything already in it is hidden, not deleted, while the filesystem is mounted).

```bash
mkdir /data1
mount -F hfs  /dev/vg01/lvol1 /data1      # (or /sbin/fs/hfs/mount ...)
mkdir /data2
mount -F vxfs /dev/vg01/lvol2 /data2      # (or /sbin/fs/vxfs/mount ...)

mount -aF vxfs               # mount all FS of a type
mount -l                     # list mounted local FS
mount -v                     # list all mounted FS (with types)
mount -o defaults            # rw,suid,largefiles,noquota

mountall                     # = mount -a
unmountall                   # = umount -a

# Force-unmount a VxFS FS even if in use
/sbin/fs/vxfs/vxumount -o force /data
```

`/sbin/init.d/localmount` auto-mounts the filesystems listed in `/etc/fstab` at boot.

### /etc/fstab entries

Each `/etc/fstab` line has six whitespace-separated fields: device, mount point, FS type, options, dump frequency, and fsck pass order.

```bash
# device               mount   type   options              dump  pass
/dev/vg00/lvol3        /       vxfs   delaylog             0     1
/dev/vg01/lvol1        /data   vxfs   rw,suid,largefiles   0     2
```

Useful VxFS mount options worth knowing:

| Option | Effect |
|--------|--------|
| `delaylog` | Default logging mode — good balance of performance and integrity |
| `tmplog` | Fastest, weakest — log flushes are delayed; only for scratch data |
| `log` | Strictest — metadata is logged synchronously; slowest, safest |
| `nodatainlog` | Do not log data on I/O-error-prone media |
| `largefiles` / `nolargefiles` | Allow / forbid files larger than 2 GB |
| `mincache=direct` | Bypass the buffer cache (useful under a database that does its own caching) |

The `delaylog` vs `log` choice is the usual performance/safety knob: `log` guarantees the on-disk log is written before returning from a metadata operation, `delaylog` batches log writes for throughput at the cost of a slightly larger window of metadata loss on a crash (file *data* integrity is unaffected — this is about metadata ordering).

### Mounting Optical / ISO / Loopback

```bash
# CD/DVD (CDFS)
mount -F cdfs -o ro /dev/disk/disk1 /dvd
mount -F cdfs -o ro,cdcase /dev/disk/disk1 /dvd        # disable 8.3;1, uppercase names
mount -F cdfs -o ro,cdcase,rr /dev/disk/disk1 /dvd     # Rock Ridge extensions

# Mount an ISO file (11i v3)
mount -F cdfs /root/myapp.iso /dvd

# Create an ISO (needs the IGNITE product)
swlist IGNITE
/opt/ignite/lbin/mkisofs -R -o /root/user1.iso /home/user1

# Loopback (LOFS) — same hierarchy under a second mount point
mkdir /opt/data
mount -F lofs /data /opt/data
```

## Tuning

```bash
vxtunefs /home                    # display VxFS tunables for a mounted FS
fsadm -F vxfs -o largefiles /data # enable large files on a mounted FS
fsadm -F vxfs /data               # check the largefiles setting
```

`vxtunefs` exposes the read-ahead and write-behind behavior that governs sequential I/O performance. The two tunables you touch most often are `read_pref_io`/`read_nstream` (how much VxFS reads ahead) and `write_pref_io`/`write_nstream` (how much it batches for write-behind). For a filesystem sitting on a striped LVM volume, aligning `read_pref_io` with the stripe unit and `read_nstream` with the number of columns lets VxFS issue full-stripe reads, which is the single biggest sequential-throughput win. Persist tunables in `/etc/vx/tunefstab` so they survive remounts:

```bash
# /etc/vx/tunefstab — applied at mount time
/dev/vg01/datavol read_pref_io=1048576,read_nstream=4,write_pref_io=1048576,write_nstream=4
```

## Defragmenting (Online JFS)

VxFS allocates files in **extents** (runs of contiguous blocks) rather than single blocks, which keeps most files contiguous naturally. Over time, though, a busy filesystem with lots of create/delete churn accumulates two kinds of fragmentation: **extent** fragmentation (a file's data scattered across many small extents, hurting sequential read speed) and **directory** fragmentation (directory blocks scattered, slowing lookups). `fsadm` reports and repairs both, and because this is an Online JFS feature it runs against the mounted, live filesystem. Schedule defrag during a quiet window and bound it with `-t` so it does not run indefinitely; a good habit is to run the report first, and only defrag if the "before" numbers show meaningful fragmentation.

```bash
fsadm -F vxfs -E /data            # report extent fragmentation
fsadm -F vxfs -D /data            # report directory fragmentation
fsadm -F vxfs -DE /data           # both reports
fsadm -F vxfs -DEde /data         # report, defrag (d/e), then report again
fsadm -F vxfs -DEde -t 600 /data  # defrag for max 600s
```

## Repairing a Corrupted Filesystem (fsck)

Run `fsck` only on an **unmounted** filesystem (or a filesystem the OS considers dirty at boot). Running it against a mounted, live filesystem can corrupt it, because `fsck` and the kernel would both be writing metadata. Always use the raw device.

```bash
# Replay the VxFS intent log (fast; use the RAW device)
fsck -F vxfs /dev/vg01/rdatavol

# Full metadata check, bypassing the intent log (JFS)
fsck -F vxfs -o full,nolog /dev/vg01/rdatavol
```

The default (log replay) is what runs automatically at boot for a filesystem that was not cleanly unmounted, and it is nearly instant. You only reach for `-o full,nolog` when log replay itself fails or the filesystem is structurally damaged (VxFS sets a `VX_FULLFSCK` flag that forces this) — the full check walks every inode and extent map and can take a long time on a large filesystem. For HFS, there is no intent log, so `fsck` is *always* the slow structural walk; this is the main operational reason HFS fell out of favor for large data volumes.

### Orphaned Files (lost+found)

`fsck` puts recovered orphaned files in `lost+found`, prompting `REMOVE` (delete) or `RECONNECT` (keep). To triage them:

```bash
cd /data/lost+found
ll  '#1743'                        # who owns it
file '#1743'                       # what type it is
strings '#1743'                    # peek at contents
mv '#1743' new_file_name           # rename/move as appropriate
```

## Reclaiming Space

```bash
bdf                                # free space per FS (HP-UX df-style)
bdf -t hfs                         # only HFS
bdf -t vxfs                        # only VxFS
df                                 # blocks/free in 512-byte units
df -k                              # in 1 KB blocks
du -sk /data/*                     # per-directory usage (KB)
quot /var                          # space usage by user

# Truncate accounting logs
> /var/adm/wtmp ; > /var/adm/wtmps
> /var/adm/btmp ; > /var/adm/btmps

# Find/purge core files and large old files
find / -name core -exec ll -d {} \;
find / -name core -exec rm -i {} \;
find / -atime +30 -size +1000c -exec ll -ud {} \;
```

## Growing and Shrinking Across the LVM Stack

To enlarge a filesystem you grow each layer bottom-up: **VG → LV → FS**. To shrink, go top-down (FS → LV).

### Extend a volume group

```bash
pvcreate -f /dev/rdisk/disk3
vgextend vg01 /dev/disk/disk3
vgdisplay -v vg01
```

### Extend a logical volume

```bash
lvextend -L 32 /dev/vg01/datavol /dev/disk/disk1
lvdisplay -v /dev/vg01/datavol
```

### Extend the filesystem

```bash
# Online (Online JFS) — no unmount
fsadm -F vxfs -b 32m /data
bdf /data

# Without Online JFS — unmount first
umount /data
extendfs -F vxfs /dev/vg01/rdatavol
mount /data
bdf /data
```

### Reduce a filesystem

```bash
# With Online JFS
fsadm -F vxfs -b 16M /data
bdf /data

# Without Online JFS — back up, recreate smaller, restore
tar -c /data
umount /data
newfs -F hfs -s 16384 /dev/vg01/rdatavol    # size in 1024-byte blocks
mount /data
tar -x /data
bdf /data
```

### Reduce / remove a logical volume

```bash
lvreduce -L 16 /dev/vg01/datavol            # shrink LV (after shrinking FS!)
lvdisplay -v /dev/vg01/datavol

umount /data                                 # remove LV
lvremove -f /dev/vg01/datavol
vi /etc/fstab
```

> Always shrink the **filesystem before** the logical volume — shrinking the LV under a larger FS destroys data.

### Reduce a volume group

```bash
pvdisplay -v /dev/disk/disk3
pvmove /dev/disk/disk3 /dev/disk/disk2       # evacuate extents off the PV
vgreduce vg01 /dev/disk/disk3
vgdisplay -v vg01
```

### Remove a volume group

A VG can be removed only when it has exactly one PV and no LVs.

```bash
# 11i v1/v2
vgreduce vg01 /dev/disk/disk2
vgremove vg01
rm -ir /dev/vg01

# 11i v3 (-X removes the VG and its group device file)
vgreduce vg01 /dev/disk/disk2
vgremove -X vg01

# Alternative: export it out of /etc/lvmtab
vgchange -a n vg01
vgexport vg01
```

### Remove a physical volume

```bash
pvremove /dev/rdisk/disk1                    # remove LVM headers
mediainit -S -c 0 -t 3 /dev/rdisk/disk1      # scrub remaining data (11i v3 only)
```

## Upgrading HFS to JFS (vxfsconvert)

```bash
bdf /data                                    # need ~15% free before converting
umount /data
/sbin/fs/vxfs/vxfsconvert /dev/vg01/rdatavol
fsck -F vxfs -y -o full /dev/vg01/rdatavol
mount /dev/vg01/datavol /data
vi /etc/fstab                                # change the FS type to vxfs
```

## VxFS ioerror Mount Option

The `ioerror` mount option sets how VxFS reacts to I/O errors:

| Policy | Behavior |
|--------|----------|
| `disable` | Disable the FS — refuse further I/O; requires unmount + full `fsck` |
| `nodisable` | Keep running degraded; set `VX_DATAIOERR` (and `VX_FULLFSCK`/`VX_METAIOERR` on metadata failures) |
| `wdisable` | Disable on **write** failure, else degrade |
| `mwdisable` | Disable on **metadata-write** failure, else degrade — **default for local** filesystems |
| `mdisable` | Disable only on metadata **read/write** failure; degrade on data errors — **default for shared cluster** filesystems |

```bash
mount -F vxfs -o ioerror=mwdisable /dev/vg01/lvol2 /data
```

## Command Reference

| Task | Command |
|------|---------|
| FS type | `fstyp [-v] <rdev>`, `mount -v` |
| Create FS | `newfs -F hfs\|vxfs [-o largefiles] [-s size] <rdev>` |
| Layout attributes | `mkfs -F vxfs -m <dev>` |
| Upgrade layout | `vxupgrade -n <ver> <mount>` |
| Mount / list | `mount -F <type> <dev> <dir>` / `mount -v` / `bdf` |
| Force unmount VxFS | `/sbin/fs/vxfs/vxumount -o force <mount>` |
| Defrag (Online JFS) | `fsadm -F vxfs -DEde <mount>` |
| Repair | `fsck -F vxfs [-o full,nolog] <rdev>` |
| Extend FS | `fsadm -F vxfs -b <size> <mount>` (online) / `extendfs` (offline) |
| Reduce FS | `fsadm -F vxfs -b <size>` (online) / recreate (offline) |
| Grow stack | `vgextend` → `lvextend` → `fsadm/extendfs` |
| Space usage | `bdf`, `df -k`, `du -sk`, `quot` |
| Convert HFS→JFS | `vxfsconvert <rdev>` |
| ioerror policy | `mount -o ioerror=<policy>` |

## Troubleshooting

### "Device busy" on unmount

`umount` fails with *device busy* when a process still has a file open on the filesystem or a shell's working directory is inside it. Find the culprit before resorting to a force-unmount:

```bash
fuser -cu /data                 # list PIDs (and users) using the mount point
fuser -ck /data                 # kill those processes (use with care)
/sbin/fs/vxfs/vxumount -o force /data   # last resort — force-unmount VxFS
```

`fuser -ck` is a blunt instrument; prefer identifying and stopping the application cleanly. A force-unmount can leave an application with open handles in an undefined state.

### Filesystem full but `du` disagrees with `bdf`

If `bdf` shows the filesystem full but `du` accounts for far less, a process is almost certainly holding a **deleted-but-still-open** file — the space is not released until the last file descriptor closes. Restarting the offending process (often a logger or database) reclaims it:

```bash
fuser -c /var                   # who has files open here
# restart the process holding the deleted file
```

### Wrong filesystem type at mount

A `mount` that fails with an invalid-argument or superblock error is often a type mismatch — you tried to mount VxFS as HFS or vice versa. Confirm the on-disk type and mount explicitly:

```bash
fstyp /dev/vg01/rdatavol        # what is it really?
mount -F vxfs /dev/vg01/lvol2 /data
```

### Inode exhaustion

A filesystem can report *no space* while `bdf` shows free blocks if it has run out of **inodes** (too many tiny files). `bdf -i` shows inode usage. VxFS allocates inodes dynamically, so this is mostly an HFS problem, but a filesystem created with a small `ninode` value can still hit it; the fix is to recreate with `ninode=unlimited` or a higher static count.

```bash
bdf -i /data                    # inode usage per filesystem
```

## Related Articles

- [HP-UX LVM](articles/hpux-lvm.md) — the PV/VG/LV stack that filesystems sit on
- [HP-UX Swap Management](articles/hpux-swap-management.md) — swap on device vs filesystem
- [HP-UX Device Management (ioscan, scsimgr, DSFs)](articles/hpux-device-management-ioscan.md) — discovering the disks behind volume groups
- [HP-UX NFS (Server and Client)](articles/hpux-nfs.md) — sharing these filesystems over the network
- [HP-UX Disaster Recovery](articles/hpux-disaster-recovery.md) — backup and restore of filesystems
- [HP-UX Boot Process](articles/hpux-boot-process.md) — why `/stand` stays HFS
