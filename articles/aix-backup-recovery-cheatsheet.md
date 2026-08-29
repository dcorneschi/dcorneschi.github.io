# AIX Backup and Recovery Cheatsheet

Command reference for backing up and restoring IBM AIX systems — `mksysb` system images, volume group backups (`savevg`/`restvg`), file-level archives (`cpio`, `tar`, `backup`/`restore`), bootable media (`mkcd`/`mkdvd`), and cloning the root volume group with `alt_disk_install`.

> Most of these commands require `root`. Device names like `/dev/rmt0` (tape), `/dev/cd0`, and `/dev/cd1` (optical) vary by system — check with `lsdev -Cc tape` and `lsdev -Cc cdrom`.

## mksysb — Full rootvg System Backup

`mksysb` creates a bootable backup image of the root volume group (`rootvg`).

```sh
# Back up rootvg to tape, creating/updating the /image.data and excluding
# files listed in /etc/exclude.rootvg (-e); capture errors to a log
mksysb -i -e /dev/rmt0 2>/tmp/mksysb.err

# Back up to a file, excluding files (-X extends /tmp if needed)
mksysb -ivX /mnt/sysimage.mksysb
```

Common flags:

| Flag | Meaning |
|------|---------|
| `-i` | Call `mkszfile` to generate `/image.data` (disk/LV/FS layout) |
| `-e` | Exclude files/dirs matching patterns in `/etc/exclude.rootvg` |
| `-v` | Verbose — list files as they are backed up |
| `-X` | Automatically expand `/tmp` if more space is needed |
| `-m` | Also generate map files (`mkszfile -m`) |

## Listing and Restoring from a mksysb (lsmksysb / listvgbackup)

`lsmksysb` and `listvgbackup` list contents of and restore files from a `mksysb` or `savevg` image. Restore paths are relative (prefix with `./`).

```sh
# List contents of the backup on the default device /dev/rmt0
lsmksysb

# List contents of the backup on /dev/cd0
lsmksysb -f /dev/cd0

# List a non-rootvg volume group backup (-s = the VG data, not rootvg)
lsmksysb -f /dev/cd0 -s

# Restore /etc/inittab from the backup on /dev/cd0
lsmksysb -f /dev/cd0 -r ./etc/inittab

# Display the volume group backup log
lsmksysb -B

# Restore everything under ./etc into /tmp/etc
lsmksysb -r -d /tmp/etc ./etc
```

`listvgbackup` is equivalent and often preferred for VG backups:

```sh
# List the contents of the backup on /dev/cd0
listvgbackup -f /dev/cd0

# Restore ./data/mydata from tape /dev/rmt0 (-s restores the VG's saved data)
listvgbackup -r -s ./data/mydata

# Show info/header about a mksysb image file
listvgbackup -l -f /mksys/machine/mksys-machine

# Restore /etc/resolv.conf from an image file
listvgbackup -f /mnt/xx11c17.mksysb -r /etc/resolv.conf
```

## restorevgfiles — Restore Individual Files from a VG Backup

```sh
# Restore ./data/mydata into /tmp from the backup on /dev/cd0
restorevgfiles -f /dev/cd0 -s -d /tmp ./data/mydata
```

## savevg / restvg — Non-rootvg Volume Groups

`savevg` backs up a user volume group; `restvg` restores it. `mkvgdata` generates the VG's `.data` layout file.

```sh
# Back up the datavg volume group to a file (-i builds the vgdata first)
savevg -i -f /tmp/mksysb-datavg datavg

# Generate /tmp/vgdata/uservg/uservg.data describing uservg
mkvgdata uservg

# Display VG information stored in a backup (-l = list, don't restore)
restvg -lf /tmp/mksysb-datavg

# Restore the volume group image
restvg -f /tmp/mksysb-datavg
```

## Bootable Media (mkcd / mkdvd)

```sh
# Create a bootable system backup on CD-R /dev/cd0
mkcd -d /dev/cd0

# Create a mksysb image in UDF format on DVD-RAM /dev/cd1 from rootvg
mkcd -U -d /dev/cd1 -V rootvg

# Create a mksysb backup to DVD from an existing mksysb file
mkdvd -d /dev/cd0 -m /path_to_mksysb_file -V rootvg

# Review the media-creation log
cat /var/adm/ras/mkcd.log
```

## File-Level Archives

### cpio

```sh
# Create a cpio archive of /home
find /home -print | cpio -ocv > filename

# List the archive contents
cpio -ict filename | more

# Restore a directory (create dirs -d, verbose -v)
cpio -icdv < filename "tcpip/*"

# Restore a single named file
cpio -icdv < filename "resolv.conf"
```

### Compressed archives

```sh
# Extract a gzip'd tar into a different directory
gzip -d -c /cdrom/bak.gz | (cd /stage; tar xf -)

# Extract a .tar.Z (compress) archive
zcat target.tar.Z | tar xf -
```

### tar

```sh
tar -cvf filename.tar "/usr/*"     # create
tar -tvf filename.tar              # list
tar -xvf filename.tar              # restore
tar -xvpf filename.tar             # restore preserving permissions
```

### backup / restore (AIX native, by name or inode)

```sh
# Back up by filename (-i = read names from stdin)
find /home -print | backup -iqvf filename.bff

# Back up by inode at level 0 (full)
backup -0 -f filename "/usr"

# List an archive
restore -qTvf filename.bff

# Restore all files
restore -qvxf filename.bff

# Restore a single file
restore -qvxf filename.bff "./etc/passwd"
```

## alt_disk_install — Clone rootvg to Another Disk

Cloning `rootvg` to a spare disk lets you apply updates to the clone and roll back by simply booting the original disk.

```sh
# Clone the running rootvg to hdisk2
alt_disk_install -C hdisk2

# Clone rootvg, apply an update_all bundle from /dev/cd0, and set hdisk4 to boot
alt_disk_install -C -b update_all -l /dev/cd0 hdisk4

# After booting the alternate disk successfully, remove the old rootvg clone
alt_disk_install -X old_rootvg
```

> On newer AIX, `alt_disk_copy` is the current interface for the clone operation (`alt_disk_install -C` still works as a wrapper). Verify the boot list afterward with `bootlist -m normal -o`.

## Quick Reference

| Task | Command |
|------|---------|
| Full rootvg backup to tape | `mksysb -i -e /dev/rmt0` |
| List a mksysb | `lsmksysb -f /dev/cd0` |
| Restore one file from mksysb | `listvgbackup -f <image> -r /etc/resolv.conf` |
| Back up a data VG | `savevg -i -f /tmp/bk-datavg datavg` |
| Restore a data VG | `restvg -f /tmp/bk-datavg` |
| Bootable backup CD | `mkcd -d /dev/cd0` |
| Clone rootvg | `alt_disk_install -C hdisk2` |
| Remove old clone | `alt_disk_install -X old_rootvg` |
