# Linux /etc/fstab Guide

A complete reference for `/etc/fstab` — the file that defines which filesystems are mounted at boot. Covers syntax, mount sources, filesystem types, common options, and practical examples for local disks, NFS, CIFS, tmpfs, swap, and more.

## Syntax

Each line in `/etc/fstab` has six fields separated by whitespace:

```
<source>  <mount_point>  <type>  <options>  <dump>  <pass>
```

| Field | Description |
|-------|-------------|
| `source` | What to mount — device path, UUID, LABEL, NFS share, etc. |
| `mount_point` | Where to mount it — absolute path (must exist) |
| `type` | Filesystem type — ext4, xfs, nfs, cifs, tmpfs, swap, etc. |
| `options` | Comma-separated mount options — defaults, noatime, ro, etc. |
| `dump` | Backup flag for `dump` utility — `0` = skip, `1` = include (rarely used today) |
| `pass` | fsck order at boot — `0` = skip, `1` = root fs, `2` = other filesystems |

### Example Line

```
UUID=a1b2c3d4-5678-90ab-cdef-1234567890ab  /data  ext4  defaults,noatime  0  2
```

## Identifying Devices

### By UUID (Recommended)

UUIDs are stable across reboots even if device order changes (e.g., `/dev/sdb` becomes `/dev/sdc`).

```bash
# Find UUIDs for all block devices
blkid

# Find UUID for a specific device
blkid /dev/sda1

# List by UUID symlinks
ls -la /dev/disk/by-uuid/
```

```
UUID=a1b2c3d4-5678-90ab-cdef-1234567890ab  /data  ext4  defaults  0  2
```

### By LABEL

Labels are human-readable but must be unique on the system.

```bash
# Set a label on ext4
e2label /dev/sda1 data

# Set a label on XFS
xfs_admin -L data /dev/sda1

# Find labels
blkid -s LABEL
```

```
LABEL=data  /data  ext4  defaults  0  2
```

### By Device Path

Simplest form, but device names can shift between reboots (especially with USB or hot-pluggable storage).

```
/dev/sdb1  /data  ext4  defaults  0  2
```

### By /dev/disk/by-id (SAN/iSCSI)

Stable identifiers based on hardware serial numbers. Useful for SAN LUNs.

```bash
ls -la /dev/disk/by-id/
```

```
/dev/disk/by-id/scsi-360000000000000001  /san-data  xfs  defaults,nofail  0  2
```

### By /dev/disk/by-path

Based on the physical connection path (PCI slot, target, LUN).

```bash
ls -la /dev/disk/by-path/
```

```
/dev/disk/by-path/pci-0000:03:00.0-scsi-0:0:1:0-part1  /data  xfs  defaults  0  2
```

## Filesystem Types

| Type | Description |
|------|-------------|
| `ext4` | Standard Linux filesystem — journaling, mature, widely supported |
| `xfs` | High-performance, default on RHEL — good for large files and parallelism |
| `btrfs` | Copy-on-write, snapshots, compression — default on openSUSE, Fedora |
| `nfs` | Network File System — mount remote directories over the network |
| `nfs4` | NFSv4 specifically (uses a single port, supports Kerberos) |
| `cifs` | SMB/CIFS shares — Windows/Samba file shares |
| `tmpfs` | RAM-based filesystem — fast, volatile (cleared on reboot) |
| `swap` | Swap space — not mounted to a directory |
| `vfat` | FAT32 — USB drives, EFI system partitions |
| `ntfs` / `ntfs3` | NTFS — Windows drives (`ntfs3` is the in-kernel driver since 5.15) |
| `iso9660` | CD/DVD ISO images |
| `overlay` | Union filesystem — used by Docker for container layers |
| `fuse` | Userspace filesystem — SSHFS, S3FS, GlusterFS clients |
| `devtmpfs` | Device nodes (`/dev`) — auto-managed by kernel |
| `proc` | Process information (`/proc`) — virtual filesystem |
| `sysfs` | Kernel/device info (`/sys`) — virtual filesystem |

## Mount Options

### Universal Options (All Filesystems)

| Option | Description |
|--------|-------------|
| `defaults` | Equivalent to: `rw,suid,dev,exec,auto,nouser,async` |
| `rw` | Mount read-write (default) |
| `ro` | Mount read-only |
| `noatime` | Don't update access time on reads — improves performance |
| `relatime` | Update atime only if older than mtime — balance of compatibility and performance (default since kernel 2.6.30) |
| `nodiratime` | Don't update directory access times |
| `noexec` | Prevent executing binaries — security hardening for data partitions |
| `nosuid` | Ignore SUID/SGID bits — prevent privilege escalation |
| `nodev` | Don't interpret block/character device files — security for user-writable mounts |
| `auto` | Mount at boot (default) |
| `noauto` | Don't mount at boot — only mount manually with `mount /mount_point` |
| `user` | Allow non-root users to mount (implies `noexec,nosuid,nodev`) |
| `users` | Allow any user to mount and unmount |
| `nofail` | Don't fail boot if the device is missing — critical for removable/cloud storage |
| `x-systemd.automount` | Mount on first access (lazy mount via systemd) |
| `x-systemd.device-timeout=30s` | Wait up to 30s for device to appear before giving up |
| `_netdev` | Mark as network device — wait for network before mounting |
| `sync` | Synchronous I/O — write immediately to disk (slow but safe) |
| `async` | Asynchronous I/O — buffer writes (default, faster) |
| `discard` | Enable TRIM for SSDs — pass discard commands to the device |
| `errors=remount-ro` | Remount read-only on filesystem errors (ext4 default) |
| `errors=continue` | Continue on errors |
| `errors=panic` | Kernel panic on errors |

### ext4-Specific Options

| Option | Description |
|--------|-------------|
| `data=journal` | Journal both data and metadata (safest, slowest) |
| `data=ordered` | Journal metadata, flush data before metadata commit (default) |
| `data=writeback` | Journal only metadata — fastest, risk of stale data after crash |
| `commit=N` | Sync journal every N seconds (default: 5) |
| `barrier=0` / `barrier=1` | Write barriers — disable for battery-backed RAID controllers |
| `stripe=N` | Optimize for RAID stripe size in filesystem blocks |
| `acl` | Enable POSIX access control lists |
| `user_xattr` | Enable extended user attributes |
| `delalloc` | Delayed allocation (default) — better contiguous allocation |
| `nodelalloc` | Disable delayed allocation — reduces risk of data loss on crash |

### XFS-Specific Options

| Option | Description |
|--------|-------------|
| `logbufs=N` | Number of in-memory log buffers (2-8, default: auto) |
| `logbsize=N` | Size of each log buffer (default: 32KB or 64KB) |
| `allocsize=N` | Preallocation size for streaming writes (e.g., `allocsize=64m`) |
| `inode64` | Allow inodes to be allocated anywhere (required for >1TB) |
| `nobarrier` | Disable write barriers (removed in kernel 5.10+ — barriers are always enabled) |
| `largeio` | Report large preferred I/O sizes in stat() |
| `wsync` | Synchronous directory updates |

### NFS Options

| Option | Description |
|--------|-------------|
| `vers=4` / `nfsvers=4` | NFS protocol version |
| `rsize=1048576` | Read buffer size in bytes (1MB max for NFSv3+) |
| `wsize=1048576` | Write buffer size in bytes |
| `hard` | Retry NFS requests indefinitely (default) — process hangs until server responds |
| `soft` | Return error after `retrans` retries — process gets EIO, may corrupt data |
| `intr` | Allow interrupt of hung NFS operations (deprecated in newer kernels) |
| `timeo=600` | Timeout in tenths of a second (60s) before retry |
| `retrans=2` | Number of retries before returning error (soft mounts) |
| `sec=krb5` | Kerberos authentication (also: `krb5i`, `krb5p`) |
| `bg` | Retry mount in background if first attempt fails — prevents blocking boot |
| `fg` | Retry mount in foreground (default) |
| `nolock` | Disable NFS file locking (for read-only or legacy mounts) |
| `noacl` | Disable ACL support |
| `tcp` | Force TCP transport (default for NFSv4) |
| `_netdev` | Wait for network before mounting (important for NFS) |

### CIFS/SMB Options

| Option | Description |
|--------|-------------|
| `username=user` | SMB username |
| `password=pass` | SMB password (insecure — use `credentials=` instead) |
| `credentials=/path/to/file` | File containing username/password/domain |
| `domain=DOMAIN` | Windows domain or workgroup |
| `uid=1000` | Map file ownership to this local UID |
| `gid=1000` | Map file ownership to this local GID |
| `file_mode=0644` | Permission mask for files |
| `dir_mode=0755` | Permission mask for directories |
| `vers=3.0` | SMB protocol version (2.0, 2.1, 3.0, 3.1.1) |
| `seal` | Enable SMB3 encryption |
| `sec=ntlmssp` | Authentication type (also: `krb5`, `krb5i`, `ntlmv2`) |
| `iocharset=utf8` | Character encoding for filenames |
| `noperm` | Don't check permissions client-side (server enforces) |
| `cifsacl` | Map CIFS ACLs to Linux permissions |
| `multiuser` | Allow multiple users with different credentials |
| `_netdev` | Wait for network before mounting |

### tmpfs Options

| Option | Description |
|--------|-------------|
| `size=512M` | Maximum size of the filesystem (default: 50% of RAM) |
| `nr_inodes=1000000` | Maximum number of inodes |
| `mode=1777` | Permission bits for the root directory |
| `uid=0` | Owner UID |
| `gid=0` | Owner GID |
| `noexec` | Prevent executing binaries from tmpfs |
| `nosuid` | Ignore SUID bits |

## Practical Examples

### Local ext4 Partition

```
# Data disk — standard options with noatime for performance
UUID=a1b2c3d4-5678-90ab-cdef-1234567890ab  /data  ext4  defaults,noatime,nofail  0  2
```

### Local XFS Partition

```
# XFS data volume — common on RHEL/CentOS
UUID=12345678-abcd-efgh-ijkl-123456789012  /var/lib/docker  xfs  defaults,noatime,pquota  0  2
```

### SSD with TRIM

```
# SSD with periodic TRIM via discard option
UUID=abcdef12-3456-7890-abcd-ef1234567890  /  ext4  defaults,noatime,discard,errors=remount-ro  0  1
```

> **Tip:** Instead of the continuous `discard` option, many prefer running `fstrim` weekly via timer for better SSD performance. See `systemctl enable fstrim.timer`.

### NFS Mount

```
# NFSv4 share from a NAS
192.168.1.100:/export/data  /mnt/nas  nfs4  defaults,_netdev,noatime,bg,hard,rsize=1048576,wsize=1048576  0  0

# NFSv3 share with specific version
nas.internal:/backups  /mnt/backups  nfs  vers=3,_netdev,bg,hard,rsize=1048576,wsize=1048576,noatime  0  0
```

### CIFS/SMB Share (Windows/Samba)

```
# SMB share with credentials file
//fileserver.internal/shared  /mnt/shared  cifs  credentials=/etc/samba/creds,uid=1000,gid=1000,file_mode=0644,dir_mode=0755,vers=3.0,_netdev,nofail  0  0
```

Credentials file (`/etc/samba/creds`, mode 0600):

```
username=svcaccount
password=s3cret
domain=CORP
```

### tmpfs (RAM Disk)

```
# In-memory filesystem for temp files (cleared on reboot)
tmpfs  /tmp  tmpfs  defaults,noatime,nosuid,nodev,noexec,size=2G,mode=1777  0  0

# Shared memory
tmpfs  /dev/shm  tmpfs  defaults,nosuid,nodev,size=4G  0  0

# Application cache in RAM
tmpfs  /var/cache/app  tmpfs  defaults,noatime,size=512M,uid=1000,gid=1000,mode=0750  0  0
```

### Swap Space

```
# Swap partition
UUID=98765432-1234-5678-9012-abcdef123456  none  swap  sw  0  0

# Swap file (works the same way)
/swapfile  none  swap  sw  0  0

# Swap with priority (higher = used first)
UUID=aaaa-bbbb  none  swap  sw,pri=10  0  0
UUID=cccc-dddd  none  swap  sw,pri=5   0  0
```

### EFI System Partition

```
# EFI partition — FAT32, mounted at /boot/efi
UUID=ABCD-1234  /boot/efi  vfat  umask=0077,shortname=winnt  0  2
```

### USB Drive (Removable)

```
# USB drive — noauto (only mounted manually), nofail (don't block boot)
UUID=1234-5678  /mnt/usb  vfat  noauto,nofail,user,uid=1000,gid=1000,umask=0022  0  0
```

### iSCSI LUN

```
# iSCSI target — _netdev ensures mount waits for network
/dev/disk/by-path/ip-10.0.0.50:3260-iscsi-iqn.2024-01.com.example:storage-lun-0  /iscsi-data  xfs  defaults,_netdev,noatime,nofail  0  2
```

### Bind Mount

```
# Expose a directory at a second location (not a new filesystem)
/var/www  /home/deploy/www  none  bind  0  0

# Read-only bind mount
/etc/ssl/certs  /container/certs  none  bind,ro  0  0
```

### GlusterFS

```
# GlusterFS distributed volume
gluster-node1:/gv0  /mnt/gluster  glusterfs  defaults,_netdev,backup-volfile-servers=gluster-node2  0  0
```

### systemd Automount (Lazy Mount)

```
# Only mount when first accessed — reduces boot time
192.168.1.100:/export/data  /mnt/nas  nfs4  noauto,x-systemd.automount,x-systemd.idle-timeout=300,_netdev  0  0
```

This mounts the NFS share only when something accesses `/mnt/nas`, and unmounts it after 5 minutes of inactivity.

### Cloud Disk with Timeout

```
# Cloud volume that might take time to attach
UUID=cloud-disk-uuid  /data  xfs  defaults,nofail,x-systemd.device-timeout=60s  0  2
```

## The dump and pass Fields

### dump (Field 5)

| Value | Meaning |
|-------|---------|
| `0` | Don't back up with `dump` (use this for almost everything today) |
| `1` | Include in `dump` backups |

The `dump` utility is rarely used in modern systems. Set to `0` for all entries.

### pass / fsck order (Field 6)

| Value | Meaning |
|-------|---------|
| `0` | Don't check with `fsck` at boot (NFS, tmpfs, swap, btrfs, xfs) |
| `1` | Check first — reserved for the root filesystem (`/`) |
| `2` | Check after root — other local filesystems |

> **Note:** XFS and Btrfs have their own repair tools and should use `0`. Network filesystems (NFS, CIFS) always use `0`. Only ext-family filesystems benefit from fsck at boot.

## Applying Changes

```bash
# Test fstab syntax (mount everything that isn't already mounted)
sudo mount -a

# Mount a specific entry
sudo mount /data

# Reload systemd's view of fstab (for systemd.automount entries)
sudo systemctl daemon-reload

# Check for syntax errors before rebooting
findmnt --verify
```

### Recovering from a Bad fstab

If a broken fstab entry prevents boot:

1. Boot into single-user mode or rescue mode
2. Remount root as read-write: `mount -o remount,rw /`
3. Edit `/etc/fstab` and fix the offending line
4. Reboot: `reboot`

Or add `nofail` to entries for non-critical mounts to prevent boot failures.

## Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Missing `nofail` on cloud/removable disk | System hangs at boot if disk is missing | Add `nofail` |
| Missing `_netdev` on NFS/CIFS | Mount attempted before network is up — fails | Add `_netdev` |
| Using `/dev/sdX` instead of UUID | Device name changes → mounts wrong disk or fails | Use `UUID=` |
| Mount point doesn't exist | Mount fails at boot | `mkdir -p /mount/point` |
| Missing `bg` on NFS | Boot blocks if NFS server is down | Add `bg` |
| Wrong permissions in credentials file | Other users can read SMB password | `chmod 600 /etc/samba/creds` |
| `pass` field set to non-zero for XFS/NFS | Unnecessary fsck attempts or errors | Set to `0` |
| `discard` on HDD | No benefit, slight overhead | Remove `discard` (only for SSDs) |

## Security Hardening with fstab

Use mount options to restrict what can happen on a filesystem:

```
# /tmp — no binaries, no SUID, no device files
tmpfs  /tmp  tmpfs  defaults,noatime,nosuid,nodev,noexec,size=2G,mode=1777  0  0

# /var/tmp — same restrictions
tmpfs  /var/tmp  tmpfs  defaults,noatime,nosuid,nodev,noexec,size=1G,mode=1777  0  0

# /home — no SUID binaries, no device files
UUID=xxxx  /home  ext4  defaults,noatime,nosuid,nodev  0  2

# /var — no SUID, no device files
UUID=xxxx  /var  ext4  defaults,noatime,nosuid,nodev  0  2

# /var/log — no execution, no SUID, no devices
UUID=xxxx  /var/log  ext4  defaults,noatime,nosuid,nodev,noexec  0  2

# /boot — read-only after initial setup
UUID=xxxx  /boot  ext4  defaults,noatime,nosuid,nodev,noexec,ro  0  2
```

| Option | Security Benefit |
|--------|-----------------|
| `nosuid` | Prevents SUID privilege escalation from that partition |
| `nodev` | Blocks creation of device files (prevents `/dev/sda` tricks) |
| `noexec` | No binary execution — prevents malware from running in /tmp |
| `ro` | Read-only — nothing can be modified |

## Quick Reference

```bash
# Show current mounts with options
mount | column -t

# Show fstab-style output for current mounts
findmnt --fstab

# Verify fstab without mounting
findmnt --verify

# List block devices with UUIDs and labels
lsblk -f

# Get UUID for a device
blkid /dev/sda1

# Test mount a specific fstab entry
mount -v /data

# Mount everything in fstab
mount -a

# Check if a filesystem is mounted
mountpoint -q /data && echo "mounted" || echo "not mounted"
```
