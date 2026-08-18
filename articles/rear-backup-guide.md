# ReaR Backup Guide

Relax-and-Recover (ReaR) creates bootable disaster recovery images and full system backups. It produces an ISO image containing a minimal rescue system that can restore the entire server from backup. ReaR supports NFS, CIFS (Samba), USB, rsync, and local destinations as backup targets.

## How It Works

1. ReaR creates a bootable ISO with a minimal Linux rescue system
2. The rescue system contains the disk layout, bootloader config, and network settings
3. A full backup of the filesystem is stored on a remote target (NFS, CIFS, USB)
4. During recovery, the ISO recreates partitions/LVM/filesystems and restores files from backup

## Backup Server Setup (NFS)

### Install and Configure NFS

```bash
yum install nfs-utils
mkdir /backup
```

Configure the export:

```bash
vi /etc/exports
```

```bash
/backup  *(fsid=0,rw,sync,no_root_squash,no_subtree_check,crossmnt)
```

Start the NFS service:

```bash
# RHEL 6
service nfs start
chkconfig nfs on

# RHEL 7+
systemctl enable --now nfs-server
```

## Client Installation

### RHEL 6

```bash
yum install nfs-utils rpcbind rear genisoimage syslinux
service rpcbind start; chkconfig rpcbind on
service nfslock start; chkconfig nfslock on
chkconfig netfs on
```

### RHEL 7 / 8 / 9

```bash
yum install nfs-utils rpcbind rear genisoimage syslinux
systemctl start rpcbind nfs-lock
systemctl enable rpcbind nfs-lock
```

On RHEL 8/9 use `dnf`:

```bash
dnf install nfs-utils rear genisoimage syslinux
```

### Ubuntu / Debian

```bash
apt install rear nfs-common genisoimage syslinux isolinux
```

## Configuration

Edit `/etc/rear/local.conf`:

```bash
vi /etc/rear/local.conf
```

### NFS Backup (Most Common)

```bash
OUTPUT=ISO
OUTPUT_URL=nfs://192.168.1.32/backup
BACKUP=NETFS
BACKUP_URL=nfs://192.168.1.32/backup
BACKUP_PROG_EXCLUDE=("${BACKUP_PROG_EXCLUDE[@]}" '/media' '/var/tmp' '/var/crash')
NETFS_KEEP_OLD_BACKUP_COPY=
```

### CIFS (Samba) Backup

```bash
OUTPUT=ISO
OUTPUT_URL=cifs://192.168.1.32/backup
BACKUP=NETFS
BACKUP_URL=cifs://192.168.1.32/backup
BACKUP_OPTIONS="cred=/etc/rear/cifs_credentials"
```

Create credentials file:

```bash
cat > /etc/rear/cifs_credentials << EOF
username=backup_user
password=backup_pass
domain=WORKGROUP
EOF
chmod 600 /etc/rear/cifs_credentials
```

### USB Backup

```bash
OUTPUT=USB
USB_DEVICE=/dev/disk/by-label/REAR-000
BACKUP=NETFS
BACKUP_URL=usb:///dev/disk/by-label/REAR-000
```

Format the USB device first:

```bash
rear format /dev/sdX
```

### rsync Backup

```bash
OUTPUT=ISO
BACKUP=RSYNC
BACKUP_URL=rsync://backup_user@192.168.1.32/backup
```

## Configuration Options

| Option | Description |
|--------|-------------|
| `OUTPUT=ISO` | Create bootable ISO image |
| `OUTPUT=USB` | Create bootable USB |
| `OUTPUT=PXE` | Create PXE boot files |
| `OUTPUT_URL=` | Where to store the ISO |
| `BACKUP=NETFS` | Use tar backup over network filesystem |
| `BACKUP=RSYNC` | Use rsync for backup |
| `BACKUP_URL=` | Where to store the backup archive |
| `BACKUP_PROG_EXCLUDE=()` | Paths to exclude from backup |
| `NETFS_KEEP_OLD_BACKUP_COPY=` | Keep previous backup (empty = no) |
| `NETFS_KEEP_OLD_BACKUP_COPY=yes` | Keep one previous backup |
| `BACKUP_PROG_COMPRESS_OPTIONS=` | Compression options for tar |
| `BACKUP_TYPE=incremental` | Incremental backup (requires FULLBACKUPDAY) |
| `FULLBACKUPDAY="Mon"` | Day for full backup when using incremental |

## Running Backups

### Create Full Backup (ISO + Data)

```bash
rear -d -v mkbackup
```

This creates:
- Bootable ISO in `/var/lib/rear/output/`
- Full backup archive on the configured `BACKUP_URL`

### Create Only the Rescue ISO (No Data Backup)

```bash
rear -d -v mkrescue
```

### Create Only the Backup (No ISO)

```bash
rear -d -v mkbackuponly
```

### Verify Configuration

```bash
rear dump
```

### Check Disk Layout

```bash
rear savelayout
cat /var/lib/rear/layout/disklayout.conf
```

## Recovery Process

1. Burn the bootable ISO image to a CD/DVD or mount via IPMI/iLO/iDRAC
2. Boot the server from the recovery medium
3. Select "Recover hostname" from the boot menu
4. Login as root (no password required)
5. Run the recovery:

```bash
rear -d -v recover
```

6. Reboot after recovery completes:

```bash
reboot
```

## Scheduling Backups

### Cron Job (Daily at 2 AM)

```bash
echo "0 2 * * * root /usr/sbin/rear mkbackup" > /etc/cron.d/rear-backup
```

### Systemd Timer

```bash
# /etc/systemd/system/rear-backup.service
[Unit]
Description=ReaR Backup

[Service]
Type=oneshot
ExecStart=/usr/sbin/rear mkbackup
```

```bash
# /etc/systemd/system/rear-backup.timer
[Unit]
Description=Daily ReaR Backup

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
systemctl enable --now rear-backup.timer
```

## Incremental Backups

Reduce backup time and storage by only backing up changes:

```bash
# /etc/rear/local.conf
OUTPUT=ISO
OUTPUT_URL=nfs://192.168.1.32/backup
BACKUP=NETFS
BACKUP_URL=nfs://192.168.1.32/backup
BACKUP_TYPE=incremental
FULLBACKUPDAY="Sun"
```

Full backup runs on Sunday, incrementals on all other days.

## Excluding Paths

```bash
BACKUP_PROG_EXCLUDE=("${BACKUP_PROG_EXCLUDE[@]}" '/media' '/var/tmp' '/var/crash' '/var/log/journal' '/tmp')
```

Exclude large or non-essential paths to reduce backup size and time.

## Troubleshooting

| Issue | Fix |
|-------|-----|
| NFS mount fails | Verify NFS server export: `showmount -e NFS_SERVER` |
| ISO too large | Exclude unnecessary paths, use `BACKUP_PROG_COMPRESS_OPTIONS` |
| Rescue boot fails | Check BIOS/UEFI mode matches the original system |
| Recovery fails at disk layout | Edit `/var/lib/rear/layout/disklayout.conf` during recovery |
| Permission denied on NFS | Check `no_root_squash` in `/etc/exports` |
| Missing network during recovery | Add `NETWORKING_PREPARATION_COMMANDS` to local.conf |

### Debug Mode

```bash
# Verbose debug output
rear -d -v mkbackup

# Even more verbose
rear -D -v mkbackup
```

### Check Logs

```bash
# Backup logs
less /var/log/rear/rear-$(hostname).log

# List backup contents
ls -lh /backup/$(hostname)/
```

## Verify Backups

Always test recovery on a spare machine or VM:

```bash
# List what's in the backup archive
tar -tzf /backup/$(hostname)/backup.tar.gz | head -50

# Check ISO is bootable
file /var/lib/rear/output/rear-$(hostname).iso
```

## Best Practices

- Test recovery regularly on a spare VM — a backup you haven't tested is not a backup
- Schedule backups during off-peak hours
- Use `NETFS_KEEP_OLD_BACKUP_COPY=yes` to keep one previous backup
- Exclude `/tmp`, `/var/tmp`, `/var/crash`, `/media` to save space
- Monitor backup logs for failures
- Store the ISO separately from the data backup when possible
- For UEFI systems, ensure the rescue ISO matches the boot mode
- Use incremental backups for large systems with daily changes
