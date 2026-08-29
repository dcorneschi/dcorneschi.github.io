# AIX Filesystems Cheatsheet

Command reference for working with filesystems on IBM AIX — the default `rootvg` logical volume layout, file timestamps, and managing JFS2 filesystems with `lsfs`/`chfs`, snapshots, VFS entries, and diagnostics like `fuser` and `fileplace`.

> Most administrative commands here require `root`. AIX filesystems sit on logical volumes; growing a filesystem grows its underlying LV automatically. For volume group and backup context, see the [AIX Backup and Recovery Cheatsheet](articles/aix-backup-recovery-cheatsheet.md).

## Default rootvg Layout

AIX creates a standard set of logical volumes and mount points in `rootvg`:

| LV | Mount / purpose |
|----|-----------------|
| `hd1` | `/home` |
| `hd2` | `/usr` |
| `hd3` | `/tmp` |
| `hd4` | `/` (root) |
| `hd5` | Boot logical volume |
| `hd6` | Paging (swap) space |
| `hd8` | JFS/JFS2 log device |
| `hd9var` | `/var` |
| `hd10opt` | `/opt` |
| `hd11admin` | `/admin` |

## File Timestamps

Every file tracks three timestamps in its inode:

| Timestamp | Meaning | View with |
|-----------|---------|-----------|
| `st_atime` | Last time file data was **accessed** (read/opened) | `ls -u` |
| `st_mtime` | Last time file **content** was modified | `ls -l` (default) |
| `st_ctime` | Last time file **metadata** changed (permissions, ownership) | `ls -c` |

View all three at once for a file with `istat`:

```sh
istat /etc/passwd
```

> **noatime for performance:** on filesystems with heavy read activity, disabling access-time updates avoids a metadata write on every read. Add it at mount time with `-o noatime`, or make it permanent:
> ```sh
> chfs -a options=noatime /data
> ```

## Listing Filesystems (lsfs)

```sh
lsfs                 # all filesystems, grep-friendly
lsfs -q /home        # extended info about /home
lsfs -l              # list format
lsfs -c              # column format
lsfs -v jfs2         # only JFS2 filesystems
lsfs -v nfs          # only NFS filesystems
```

## Changing Filesystems (chfs)

`chfs` resizes, moves, and sets options on a filesystem; growing it expands the underlying logical volume automatically.

```sh
# Grow /var by 1 GB (relative)
chfs -a size=+1G /var

# Set /var to exactly 1 GB (absolute)
chfs -a size=1G /var

# Shrink /var by 1 GB
chfs -a size=-1G /var

# Change the mount point from /test to /new
chfs -m /test /new

# Don't auto-mount at restart
chfs -A no /data

# Mount read-only
chfs -p ro /data

# Enable noatime permanently
chfs -a options=noatime /data
```

### Freeze and split-copy (for backups)

```sh
# Freeze the filesystem (no writes) for N seconds — e.g. during a split/backup;
# 'freeze=off' thaws it
chfs -a freeze=<seconds> /data
chfs -a freeze=off /data

# Split the 2nd mirror copy of /testfs and mount it read-only at /backup
chfs -a splitcopy=/backup -a copy=2 /testfs
```

## VFS Entries (/etc/vfs)

```sh
lsvfs                # list virtual filesystem type entries
crvfs                # create an entry
rmvfs                # remove an entry
```

## Volume Group Filesystems

```sh
# List filesystems belonging to a volume group
lsvgfs rootvg
```

## Removing a Filesystem

```sh
# Remove /mymount's entry and its logical volume (add -r to also remove the mount point dir)
rmfs /mymount
rmfs -r /mymount
```

## Space Usage

```sh
# Summarize usage on one filesystem (-s summary, -m in MB, -x stay on one filesystem)
du -smx /

# Largest entries under /home, sorted
du -smx /home/* | sort -rn | head
```

## JFS2 Snapshots

```sh
# Create a snapshot of /data on logical volume /dev/snapsb
snapshot -o snapfrom=/data /dev/snapsb

# Delete the snapshot and the LV holding it
snapshot -d /dev/snapsb
```

## Diagnostics and Repair

```sh
# Debug a filesystem interactively
fsdb /data

# Dump JFS2 superblock/structure in readable form
dumpfs /dev/hd4 | more

# Show the placement of a file's blocks on the LV/PV (reveals fragmentation)
fileplace <filename>
```

### fuser — Who's Using a File or Filesystem

```sh
# Processes using a specific file
fuser /etc/passwd

# Processes using a filesystem (-c), with user (-u) and file type (-x) info
fuser -cux /var

# Kill those processes (-k) — useful before unmounting
fuser -cuxk /var

# Show deleted-but-still-open files (inodes) holding space, with owning PIDs
fuser -dV /tmp
```

## Quick Reference

| Task | Command |
|------|---------|
| List all filesystems | `lsfs` |
| Extended info for one FS | `lsfs -q /home` |
| Grow a filesystem | `chfs -a size=+1G /var` |
| Move a mount point | `chfs -m /test /new` |
| Enable noatime | `chfs -a options=noatime /data` |
| List a VG's filesystems | `lsvgfs rootvg` |
| Remove a filesystem + LV | `rmfs -r /mymount` |
| Create / delete a snapshot | `snapshot -o snapfrom=/data /dev/snapsb` / `snapshot -d /dev/snapsb` |
| Who's using a filesystem | `fuser -cux /var` |
| Check file fragmentation | `fileplace <filename>` |
| All timestamps of a file | `istat <file>` |

## Related

- [AIX Backup and Recovery Cheatsheet](articles/aix-backup-recovery-cheatsheet.md) — mksysb, volume group, and file-level backups
- [AIX Boot and Init Cheatsheet](articles/aix-boot-init-cheatsheet.md) — bootlist, bosboot, inittab, and run levels
