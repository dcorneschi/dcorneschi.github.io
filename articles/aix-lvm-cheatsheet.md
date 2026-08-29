# AIX LVM Cheatsheet

Reference for the IBM AIX Logical Volume Manager (LVM) — volume group types and internals, allocation policies, physical volume states, and the full command set for managing volume groups, logical volumes, physical volumes, mirroring, and common maintenance tasks.

> Most commands require `root`. LVM sits below filesystems — for creating/resizing filesystems on top of logical volumes, see the [AIX Filesystems Cheatsheet](articles/aix-filesystems-cheatsheet.md).

## Volume Group Types

Before AIX 5L V5.3 there were two VG types; V5.3 added scalable VGs.

| Type | Created with | Max LVs | Notes |
|------|--------------|---------|-------|
| Normal (standard/original/default) | `mkvg` (no `-B`/`-S`) | 256 | The classic default type |
| Big | `mkvg -B` (since V4.3.2) | 512 | Cannot be imported into V4.3.1 or earlier |
| Scalable | `mkvg -S` (since 5L V5.3) | 4096 | See below |

**Scalable volume groups (AIX 5L V5.3+):**

- Up to 1024 disks per VG and 4096 LVs per VG
- Maximum number of physical partitions (PPs) is VG-dependent, not PV-dependent
- LV control information is kept in the VGDA
- Max values need not be set at creation time — they can be raised later

## LVM Data Structures

**VGDA (Volume Group Descriptor Area)**
- The most important LVM data structure; global to the VG (identical on each disk)
- One or two copies per disk

**VGSA (Volume Group Status Area)**
- Tracks the state of mirrored copies
- One or two copies per disk

**LVCB (Logical Volume Control Block)**
- Historically occupies the first 512 bytes of each LV
- Holds LV attributes (policies, number of copies)
- Must **not** be overwritten by applications using raw devices

Maximum LV size: 1 TB (32-bit kernel) or 128 TB (64-bit kernel).

## Allocation Policies

**Inter-physical policy** (how an LV spreads across disks):

- `MIN` — the LV stays on as few disks as possible (ideally one)
- `MAX` — the LV spreads across as many PVs as possible (for throughput)

**Intra-physical policy** (where on a disk PPs are placed): one of `edge`, `middle`, or `center`, chosen when the LV is created.

## Physical Volume States

During `varyonvg`, a PV can be in one of these states:

| State | Meaning |
|-------|---------|
| `active` | Disk is accessible after varyon |
| `missing` | Disk not accessible but quorum is available; after repair, `varyonvg` returns it to active |
| `removed` | No access and quorum unavailable; recover with `varyonvg -f`, then `chpv -va <disk>`, then `varyonvg` |

## Important Files

| Path | Purpose |
|------|---------|
| `/etc/vg/vgVGID` | Handle to the in-memory VGDA copy |
| `/dev/hdiskX` | Special file for a disk |
| `/dev/VGname` | Special file for administrative access to a VG |
| `/dev/LVname` | Special file for a logical volume |
| `/etc/filesystems` | Maps LV name, FS log, and mount point (used by `mount`) |

## Volume Groups

### Query

```sh
lsvg                 # all volume groups
lsvg -o              # only varied-on VGs
lsvg rootvg          # VG details
lsvg -l rootvg       # logical volumes in the VG
lsvg -p rootvg       # physical volumes in the VG
lsvg -M rootvg       # PVname:PPnum [LVname:LPnum[:Copynum][PPstate]]
lsvg -n hdisk0       # VG info read from the VGDA on a specific disk (compare disks)
```

### Create

```sh
# Normal VG on hdisk1 with 64 MB physical partitions
mkvg -y datavg -s 64 hdisk1

# Big VG spanning three disks
mkvg -B -y bigvg hdisk2 hdisk3 hdisk4
```

### Vary on / off

```sh
varyonvg datavg          # vary on (calls syncvg -v)
varyonvg -f datavg       # force varyon
varyonvg -s datavg       # maintenance mode
varyonvg -r datavg       # read-only
varyonvg -M <LTGsize>    # set the logical track group size
varyoffvg datavg         # vary off
```

### Extend / reduce / import / export

```sh
extendvg datavg hdisk1           # add a PV
extendvg -f datavg hdisk1        # force-add a previously used disk

reducevg -d datavg hdisk2        # remove PV, deleting its LV data automatically
reducevg datavg <PVID>           # drop a vanished disk's PVID from the VGDA

exportvg datavg                  # export (removes the VG definition!)
importvg -y datavg hdisk1        # import a VG from a disk
importvg hdisk1                  # import, auto-named vg00, vg01, ...
importvg -y datavg 000989udxyfdj # import by PVID
importvg -L datavg               # learn about changes without full import
```

### Change VG attributes (chvg)

```sh
chvg -a y datavg        # auto vary-on at boot
chvg -a n datavg        # don't auto vary-on
chvg -g datavg          # detect underlying disks that have grown
chvg -Q y|n datavg      # quorum checking on/off
chvg -B datavg          # convert normal -> big VG
chvg -b y|n datavg      # bad-block relocation on/off
chvg -t 2 rootvg        # change the PP-limit factor (1-16)
chvg -s y datavg        # auto-sync stale partitions
chvg -v 512 datavg      # max LVs per VG
chvg -u datavg          # unlock the VG
chvg -L 256 datavg      # LTG size (KB) for I/O performance
chvg -h y -s y datavg   # hot-spare: auto-migrate PPs off a failing disk + auto-sync
```

## Logical Volumes

### Query

```sh
lslv lv01            # LV details
lslv -l lv01         # which PV(s) the LV is on
lslv -p hdisk0       # PP map for a disk
lslv -m lv01         # LV characteristics / map
lslv -l hd5          # determine the boot disk (hd5 = BLV)
```

### Create

```sh
# Log device on datavg (1 LP), then format it (LV must be closed)
mklv -t jfs2log -y datalog1 datavg 1
logform /dev/datalog1
logform -V jfs2 /dev/jfs2log     # rebuild a jfs2 log

# Data LVs (by LP count or size)
mklv -t jfs2 -y datalv datavg 16
mklv -t jfs2 -y datalv datavg 1G

# Recreate the boot LV (BLV) hd5
mklv -y hd5 -t boot rootvg 1
```

### Modify / remove

```sh
extendlv lv04 40           # grow by 40 LPs
extendlv lvraw 64M         # grow by size
chlv -n lv05 lv00          # rename
chlv -w n lv00             # turn off Mirror Write Consistency (then syncvg -f -l <LV> after a crash)
chlv -w y oracle           # turn on MWC
chlv -x <new_max_lps> <lv> # raise max LPs
rmlv lv04                  # remove an LV
```

### Copy on existing LVs

```sh
crfs -v jfs2 -d datalv -m /data01 -A y   # put a filesystem on an existing LV
# Inline JFS2 log inside the filesystem
crfs -v jfs2 -g datavg -a size=1G -m /data -a logname=INLINE -a logsize=<MB>
```

## Physical Volumes

### Query

```sh
lspv                 # all physical volumes
getlvodm -C          # same listing as lspv
lspv hdisk0          # PV details
lspv -l hdisk0       # LVs on the PV
lspv -p hdisk0       # PP usage for the PV
lspv -M hdisk0       # PP mapping and stale PPs
```

### PVID and state (chpv / chdev)

```sh
chdev -l hdisk1 -a pv=yes      # assign a PVID
chdev -l hdisk1 -a pv=clear    # remove the PVID

chpv -hy hdisk1     # mark as hot-spare
chpv -hn hdisk1     # remove from hot-spare pool
chpv -vr hdisk1     # close PV (unavailable)
chpv -va hdisk1     # open PV (available)
chpv -an hdisk1     # stop PP allocation
chpv -ay hdisk1     # allow PP allocation
chpv -c hdisk1      # clear the boot record
```

### Migrate and replace

```sh
migratepv hdisk1 hdisk3 hdisk5      # move all PPs off hdisk1 onto others
migratepv -l datalv01 hdisk4 hdisk5 # move one LV between PVs
migratelp datalv/23 hdisk3/105      # move a single LP to a specific PP
replacepv hdisk1 hdisk6             # replace one PV with another
```

## Mirroring

```sh
mirrorvg -s rootvg           # mirror rootvg without synchronizing yet
mirrorvg -S -c 3 rootvg      # triple mirror, background sync
mirrorvg -m datavg hdisk3    # exact-mapped mirror of datavg's LVs
unmirrorvg rootvg            # remove mirroring

mklvcopy lv01 2 hdisk7       # add a 2nd copy of lv01 on hdisk7
mklvcopy -a e -k lv02 2 hdisk4  # copies on the outer edge
rmlvcopy lvuser 2            # reduce to 2 copies

syncvg -v datavg             # sync copies across the VG
syncvg -p hdisk3             # sync copies on a PV

splitlvcopy -y newlv oldlv 2 # split one copy into a new LV
```

## Snapshots (Split/Join VG)

```sh
# Split the 2nd mirror of datavg into a snapshot VG
splitvg -y snapvg -c 2 datavg

# Rejoin the snapshot with the original VG
joinvg datavg
```

## Low-Level / ODM Tools

```sh
lqueryvg -Atp hdisk1     # extract from the VGDA on a disk
lquerylv <lv>            # query LV attributes
lquerypv -M hdisk0       # LTG size of a disk
lquerypv -h hdisk0       # disk header
getlvcb -AT hd4          # read the LVCB of an LV
putlvcb -t jfs lvdata    # write LV type to the LVCB
readvgda hdisk0          # show disk/VGDA details

redefinevg -d hdisk1 datavg  # sync ODM from VGDA
synclvodm datavg             # sync VGDA from ODM
rvgrecover                   # repair the ODM for a VG
cplv -v datavg -y lvnew lvold  # copy an LV's contents to a new LV
```

## Common Tasks

### Mirror rootvg to a second disk

```sh
extendvg rootvg hdisk1
mirrorvg rootvg
bosboot -ad hdisk0
bosboot -ad hdisk1
bootlist -m normal hdisk0 hdisk1
```

### Move a VG from hdisk1 to hdisk2 (via mirror)

```sh
extendvg datavg hdisk2
mirrorvg datavg hdisk2
unmirrorvg datavg hdisk1
reducevg datavg hdisk1
```

Equivalent with an explicit copy count:

```sh
extendvg vg02 hdisk2
mirrorvg -c 2 vg02
unmirrorvg vg02 hdisk1
reducevg vg02 hdisk1
```

### Replace a mirrored disk (disk still working)

```sh
unmirrorvg vg_name hdiskX
reducevg vg_name hdiskX
rmdev -l hdiskX -d
extendvg vg_name hdiskY
mirrorvg vg_name hdiskY
syncvg vg_name
```

### Replace a mirrored disk (disk failed)

```sh
extendvg vg_name hdiskY
migratepv hdiskX hdiskY
reducevg vg_name hdiskX
rmdev -l hdiskX -d
```

### Rename a volume group

```sh
lsvg -l vg01
varyoffvg vg01
exportvg vg01
importvg -y vg02 hdisk3
```

## Related

- [AIX Filesystems Cheatsheet](articles/aix-filesystems-cheatsheet.md) — lsfs/chfs, snapshots, and the default rootvg layout
- [AIX Backup and Recovery Cheatsheet](articles/aix-backup-recovery-cheatsheet.md) — mksysb, savevg/restvg, and alt_disk_install
- [AIX Boot and Init Cheatsheet](articles/aix-boot-init-cheatsheet.md) — bootlist, bosboot, and inittab
