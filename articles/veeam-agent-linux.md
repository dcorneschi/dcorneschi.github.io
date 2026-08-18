# Veeam Agent for Linux

Veeam Agent for Linux provides backup and recovery for physical and virtual Linux machines. It supports volume-level and file-level backups to NFS, CIFS, local, and Veeam repository targets.

## Installation

Download the release package from the Veeam website, then install:

```bash
# RHEL 8/9
yum localinstall veeam-release-el8-1.0.7-1.x86_64.rpm
yum install veeam

# RHEL 7
yum localinstall veeam-release-el7-1.0.7-1.x86_64.rpm
yum install veeam

# Ubuntu/Debian
dpkg -i veeam-release-deb_1.0.7_amd64.deb
apt update
apt install veeam
```

## Job Management

```bash
# List configured backup jobs
veeamconfig job list

# Create a new job (interactive wizard)
veeamconfig job create

# Start a backup job
veeamconfig job start --name "MyBackupJob"

# Show job details
veeamconfig job info --name "MyBackupJob"

# Delete a job
veeamconfig job delete --name "MyBackupJob"
```

## Backup Operations

```bash
# List all backups
veeamconfig backup list

# Show detailed info about a specific backup
veeamconfig backup show --id <backup_id>

# List restore points for a backup
veeamconfig point list --backupid <backup_id>
```

## Restore

### Volume-Level Restore

```bash
veeamconfig backup restore --id <backup_id> --targetdev <target_volume> --backupdev <volume_in_backup>
```

### File-Level Restore

Mount the restore point and copy files:

```bash
# Mount the restore point
veeamconfig point mount --id <point_id> --mountdir /mnt/efs

# Browse and copy files
ls /mnt/efs/FileLevelBackup_0/
cp /mnt/efs/FileLevelBackup_0/etc/resolv.conf /tmp/

# Unmount when done (stop the session)
veeamconfig session stop --id <session_id>
```

## Session Management

```bash
# List all sessions
veeamconfig session list

# View session log
veeamconfig session log --id <session_id>

# Stop a running session
veeamconfig session stop --id <session_id>
```

## Reset Session Database

If the session list becomes corrupted or needs clearing:

```bash
rm /var/lib/veeam/veeam_db.sqlite*
service veeamservice restart
```

## NFS Backup Server Setup

### Install NFS (RHEL 7+)

```bash
yum install nfs-utils rpcbind
systemctl start rpcbind nfs-lock nfs-server
systemctl enable rpcbind nfs-lock nfs-server
```

### Configure Export

```bash
vi /etc/exports
```

```bash
/backup 192.168.1.0/24(rw,sync,no_root_squash)
```

```bash
exportfs -a
mkdir /backup
chmod 755 /backup
chown nfsnobody:nfsnobody /backup
```

### Firewall Rules

```bash
yum install firewalld
systemctl enable --now firewalld

# Allow NFS from specific network
firewall-cmd --add-rich-rule 'rule family="ipv4" service name="nfs" source address="192.168.1.0/24" accept' --permanent
firewall-cmd --reload
```

## Disk Image Backup Over Network (dd + ssh)

For bare-metal disk-level backups using `dd` piped over SSH:

### Create Image

```bash
dd if=/dev/sda | gzip -1 - | ssh root@192.168.1.22 'dd of=/repo/server-img.gz'
```

### Restore Image

Boot into rescue mode, then:

```bash
ssh root@192.168.1.22 dd if=/repo/server-img.gz | gunzip -1 - | dd of=/dev/sda
```

## Scheduling

```bash
# Show backup schedule for a job
veeamconfig schedule show --jobname "MyBackupJob"

# Set daily schedule at 2 AM
veeamconfig schedule set --jobname "MyBackupJob" --daily --at 02:00

# Set schedule to run on specific days
veeamconfig schedule set --jobname "MyBackupJob" --weekdays Mon,Wed,Fri --at 03:00

# Disable schedule
veeamconfig schedule disable --jobname "MyBackupJob"

# Enable schedule
veeamconfig schedule enable --jobname "MyBackupJob"
```

## Pre/Post Job Scripts

Run custom scripts before and after backup jobs:

```bash
veeamconfig job create \
  --name "MyBackupJob" \
  --prescript /opt/scripts/pre-backup.sh \
  --postscript /opt/scripts/post-backup.sh
```

Example pre-backup script (stop a database):

```bash
#!/bin/bash
# /opt/scripts/pre-backup.sh
systemctl stop postgresql
sync
```

Example post-backup script (start it back):

```bash
#!/bin/bash
# /opt/scripts/post-backup.sh
systemctl start postgresql
```

## Encryption

Enable encryption for backup data at rest:

```bash
# Create encrypted backup job (interactive wizard prompts for password)
veeamconfig job create --encryption

# Or specify during job creation
veeamconfig job create \
  --name "EncryptedBackup" \
  --repopath /backup \
  --encryption
```

> **Note:** The encryption password is required during restore. Store it securely — without it, the backup is unrecoverable.

## Veeam Repository Target

Back up to a Veeam Backup & Replication server instead of NFS/local:

```bash
# Add Veeam repository
veeamconfig repo add --name "VBRServer" --type vbrserver --address 192.168.1.50 --port 10006

# List configured repositories
veeamconfig repo list

# Create job targeting Veeam repository
veeamconfig job create --name "ToVBR" --repo "VBRServer"
```

## License Management

```bash
# Show current license status
veeamconfig license show

# Install a license file
veeamconfig license install --path /tmp/veeam_license.lic

# Show license expiry and edition
veeamconfig license info
```

## Command Reference

| Command | Description |
|---------|-------------|
| `veeamconfig job list` | List backup jobs |
| `veeamconfig job start --name NAME` | Start a backup job |
| `veeamconfig backup list` | List all backups |
| `veeamconfig backup show --id ID` | Show backup details |
| `veeamconfig point list --backupid ID` | List restore points |
| `veeamconfig point mount --id ID --mountdir DIR` | Mount restore point |
| `veeamconfig session list` | List all sessions |
| `veeamconfig session log --id ID` | View session log |
| `veeamconfig session stop --id ID` | Stop/unmount session |
| `veeamconfig backup restore --id ID ...` | Volume-level restore |

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Job not starting | Check service: `systemctl status veeamservice` |
| NFS mount fails | Verify export: `showmount -e NFS_SERVER` |
| Session database corrupt | Remove `veeam_db.sqlite*` and restart service |
| Backup fails with permission error | Check NFS export has `no_root_squash` |
| Cannot mount restore point | Ensure mount directory exists and is empty |
| Service not running | Start: `systemctl start veeamservice` |
