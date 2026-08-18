# rclone Cheatsheet

rclone is a command-line tool for managing files on cloud storage. It supports 70+ backends including S3, Google Drive, Azure Blob, SFTP, FTP, and local filesystems. Think of it as rsync for cloud storage.

## Installation

```bash
# Script install (Linux/macOS)
curl https://rclone.org/install.sh | sudo bash

# Debian/Ubuntu
sudo apt install rclone

# RHEL/CentOS/Fedora
sudo dnf install rclone

# macOS
brew install rclone

# Verify
rclone version
```

## Configuration

### Interactive Setup

```bash
rclone config
```

This walks through adding a remote interactively. Config is stored in `~/.config/rclone/rclone.conf`.

### Manage Remotes

```bash
rclone config show              # Show remotes with passwords
rclone config update myremote   # Update existing remote
rclone config delete myremote   # Delete remote
rclone config file              # Show config file path
```

### Non-Interactive Config (CLI)

Create remotes directly from the command line without the interactive wizard:

```bash
# Create SMB/Samba remote
rclone config create samba smb host 192.168.50.90 user myuser pass mypassword

# Create SFTP remote
rclone config create mysftp sftp host server.example.com user backup key_file ~/.ssh/id_ed25519

# Create S3 remote
rclone config create mys3 s3 provider AWS access_key_id AKIA... secret_access_key ... region us-east-1
```

### Manual Config File

```bash
vi ~/.config/rclone/rclone.conf
```

```ini
[myremote]
type = s3
provider = AWS
access_key_id = AKIAXXXXXXXXXXXXXXXX
secret_access_key = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
region = us-east-1

[gdrive]
type = drive
client_id = xxxxx.apps.googleusercontent.com
client_secret = xxxxx
token = {"access_token":"...","token_type":"Bearer","refresh_token":"...","expiry":"..."}

[mysftp]
type = sftp
host = server.example.com
user = backup
key_file = ~/.ssh/id_ed25519
```

### List Configured Remotes

```bash
rclone listremotes
```

## Basic Commands

### Copy

```bash
# Copy files from source to destination
rclone copy /local/path remote:bucket/path

# Copy from remote to local
rclone copy remote:bucket/path /local/path

# Copy between remotes
rclone copy remote1:bucket1 remote2:bucket2
```

### Sync (Mirror)

```bash
# Make destination identical to source (deletes extra files on dest)
rclone sync /local/path remote:bucket/path

# Dry run first to see what would change
rclone sync /local/path remote:bucket/path --dry-run

# Sync with progress
rclone sync /local/path remote:bucket/path -P
```

### Move

```bash
# Move files (deletes from source after transfer)
rclone move /local/path remote:bucket/path

# Move with delete-empty-src-dirs
rclone move /local/path remote:bucket/path --delete-empty-src-dirs
```

### List

```bash
# List directories in remote
rclone lsd remote:

# List files in remote path
rclone ls remote:bucket/path

# List with size and modification time
rclone lsl remote:bucket/path

# List only top-level
rclone lsf remote:bucket/path

# JSON output
rclone lsjson remote:bucket/path
```

### Delete

```bash
# Delete files in path
rclone delete remote:bucket/path

# Delete path and empty directories
rclone purge remote:bucket/path

# Delete empty directories only
rclone rmdirs remote:bucket/path
```

### Other Operations

```bash
# Check if source and destination match
rclone check /local/path remote:bucket/path

# Show file info
rclone cat remote:bucket/path/file.txt

# Create directory
rclone mkdir remote:bucket/new-dir

# Get total size
rclone size remote:bucket/path

# Show storage usage for remote
rclone about remote:
```

## Common Flags

| Flag | Description |
|------|-------------|
| `-P` / `--progress` | Show real-time transfer progress |
| `--dry-run` | Show what would happen without making changes |
| `-v` | Verbose output |
| `--transfers N` | Number of parallel transfers (default: 4) |
| `--checkers N` | Number of parallel checkers (default: 8) |
| `--bwlimit RATE` | Bandwidth limit (e.g., `10M`, `1G`) |
| `--min-size SIZE` | Skip files smaller than SIZE |
| `--max-size SIZE` | Skip files larger than SIZE |
| `--min-age DURATION` | Skip files newer than DURATION |
| `--max-age DURATION` | Skip files older than DURATION |
| `--include PATTERN` | Include files matching pattern |
| `--exclude PATTERN` | Exclude files matching pattern |
| `--filter-from FILE` | Read filter rules from file |
| `--log-file FILE` | Log output to file |
| `--log-level LEVEL` | Log level (DEBUG, INFO, NOTICE, ERROR) |
| `--retries N` | Retry operations N times (default: 3) |
| `--low-level-retries N` | Retry low-level operations |
| `--ignore-existing` | Skip files that already exist on dest |
| `--update` | Skip files that are newer on dest |
| `--no-traverse` | Don't list dest before transfer (faster for few files) |

## Filtering

### Include/Exclude

```bash
# Only sync .jpg files
rclone sync /photos remote:photos --include "*.jpg"

# Exclude .tmp and .log files
rclone sync /data remote:backup --exclude "*.tmp" --exclude "*.log"

# Exclude directories
rclone sync /data remote:backup --exclude "/node_modules/**" --exclude "/.git/**"
```

### Filter File

Create a filter file for complex rules:

```bash
# filters.txt
+ *.jpg
+ *.png
+ *.pdf
- *.tmp
- *.log
- /node_modules/**
- /.git/**
```

```bash
rclone sync /data remote:backup --filter-from filters.txt
```

### Age-Based Filtering

```bash
# Only files modified in last 7 days
rclone copy /data remote:backup --max-age 7d

# Only files older than 30 days
rclone copy /data remote:archive --min-age 30d
```

## Backup Patterns

### Simple Backup Script

```bash
#!/bin/bash
# backup.sh
DATE=$(date +%Y%m%d)
LOG="/var/log/rclone-backup-${DATE}.log"

rclone sync /data remote:backup/current \
  --transfers 8 \
  --checkers 16 \
  --log-file "$LOG" \
  --log-level INFO \
  --exclude "/tmp/**" \
  --exclude "*.tmp"

echo "Backup completed: $(date)" >> "$LOG"
```

### Backup with Versioning

```bash
# Keep old versions in a dated directory
rclone sync /data remote:backup/current --backup-dir remote:backup/versions/$(date +%Y%m%d)
```

### Incremental-Style Copy

```bash
# Only copy new/changed files (don't delete)
rclone copy /data remote:backup --update --ignore-existing
```

### Bandwidth-Limited Backup

```bash
# Limit to 50 MB/s during business hours
rclone sync /data remote:backup --bwlimit "08:00,50M 18:00,off"
```

## Mounting Remotes

Mount cloud storage as a local filesystem:

```bash
# Mount remote as local directory
rclone mount remote:bucket /mnt/cloud --daemon

# Mount with caching for better performance
rclone mount remote:bucket /mnt/cloud \
  --daemon \
  --vfs-cache-mode full \
  --vfs-cache-max-size 10G \
  --vfs-read-ahead 128M

# Unmount
fusermount -u /mnt/cloud
```

### systemd Mount Service

```bash
# /etc/systemd/system/rclone-mount.service
[Unit]
Description=rclone mount
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/rclone mount remote:bucket /mnt/cloud \
  --vfs-cache-mode full \
  --vfs-cache-max-size 10G \
  --allow-other
ExecStop=/bin/fusermount -u /mnt/cloud
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable --now rclone-mount
```

## Serving Files

```bash
# HTTP server
rclone serve http remote:bucket --addr :8080

# WebDAV server
rclone serve webdav remote:bucket --addr :8080

# SFTP server
rclone serve sftp remote:bucket --addr :2022

# FTP server
rclone serve ftp remote:bucket --addr :2121
```

## Encryption

### Create Encrypted Remote (crypt)

```bash
rclone config
# Choose "crypt" as type
# Point it to an existing remote as the backend
```

Config result:

```ini
[secret]
type = crypt
remote = remote:bucket/encrypted
password = *** ENCRYPTED ***
password2 = *** ENCRYPTED ***
```

```bash
# Write to encrypted remote
rclone copy /sensitive-data secret:

# Read from encrypted remote (decrypted transparently)
rclone ls secret:
```

## S3-Specific Commands

```bash
# List buckets
rclone lsd s3remote:

# Create bucket
rclone mkdir s3remote:new-bucket

# Set storage class
rclone copy /data s3remote:bucket --s3-storage-class GLACIER

# Multipart upload threshold
rclone copy /data s3remote:bucket --s3-upload-cutoff 200M --s3-chunk-size 50M
```

## Performance Tuning

```bash
# Increase parallelism for many small files
rclone sync /data remote:backup --transfers 16 --checkers 32

# Increase buffer size for large files
rclone copy /data remote:backup --buffer-size 64M

# Disable checksum checking (faster, less safe)
rclone sync /data remote:backup --size-only

# Use server-side copy when possible (between same provider)
rclone copy remote1:bucket1/path remote1:bucket2/path --server-side-across-configs
```

## Scheduling with Cron

```bash
# Daily backup at 3 AM with logging
0 3 * * * /usr/bin/rclone sync /data remote:backup --log-file /var/log/rclone.log --log-level INFO

# Weekly deep sync on Sunday
0 2 * * 0 /usr/bin/rclone sync /data remote:backup --log-file /var/log/rclone-weekly.log

# Proxmox backups to Samba share (hourly)
2 * * * * /usr/bin/rclone sync /mnt/pve/backup/dump samba:/Backup/Proxmox >> /tmp/rclone.log 2>&1

# Encrypted backup with sftp-idle-timeout (prevents timeouts on long jobs)
0 2 * * * /usr/bin/rclone sync --sftp-idle-timeout=0 /mnt/pve/datastore/dump crypt:/dump >> /var/log/rclone_backup.log 2>&1
```

## Bisync (Bidirectional Sync)

```bash
# Sync two paths bidirectionally
rclone bisync /local/path remote:path

# Force initial resync (required on first run)
rclone bisync --resync /local/path remote:path

# Dry run to preview changes
rclone bisync --dry-run /local/path remote:path
```

## Deduplication

```bash
# Remove duplicate files interactively
rclone dedupe remote:path

# Rename duplicates (keeps all, adds suffix)
rclone dedupe --dedupe-mode rename remote:path

# Keep newest and delete older duplicates
rclone dedupe --dedupe-mode newest remote:path
```

## Delete Single File

```bash
rclone deletefile remote:bucket/path/file.txt
```

## Filter Operators Reference

| Operator | Description |
|----------|-------------|
| `+` | Include |
| `-` | Exclude |
| `!` | Negate |
| `*` | Match any characters (single level) |
| `**` | Match any depth |
| `?` | Match single character |
| `[abc]` | Character set |
| `[a-z]` | Range |
| `{a,b}` | Alternatives |

### Filter File Example

```text
+ *.mp3
+ *.flac
+ Documents/
- **/Cache/
- *.tmp
- *.log
+ */
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `RCLONE_CONFIG` | Path to config file |
| `RCLONE_CACHE_DIR` | Cache directory |
| `RCLONE_LOG_LEVEL` | Log level (DEBUG, INFO, NOTICE, ERROR) |
| `RCLONE_BWLIMIT` | Bandwidth limit |
| `RCLONE_TRANSFERS` | Number of parallel transfers |
| `RCLONE_<REMOTE>_TYPE` | Set remote type via env |
| `RCLONE_<REMOTE>_<OPTION>` | Set any remote option via env |

## Debugging

```bash
# Increasing verbosity
rclone copy /data remote:backup -v      # Verbose
rclone copy /data remote:backup -vv     # Very verbose
rclone copy /data remote:backup -vvv    # Extremely verbose

# Debug HTTP traffic
rclone copy /data remote:backup --dump headers

# Log to file
rclone sync /data remote:backup --log-file /var/log/rclone.log --log-level DEBUG
```

## Useful Commands

```bash
rclone version                  # Show version info
rclone help                     # Show help
rclone help sync                # Help for specific command
rclone selfupdate               # Update rclone to latest
rclone backend <command> remote: # Backend-specific commands
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Auth expired | Re-run `rclone config reconnect remote:` |
| Slow transfers | Increase `--transfers` and `--checkers` |
| Rate limited | Add `--tpslimit 10` to throttle API calls |
| Out of memory on large dirs | Use `--fast-list` or reduce `--checkers` |
| Permission denied on mount | Add `--allow-other` and edit `/etc/fuse.conf` |
| Connection timeout | Increase `--timeout 60s` and `--contimeout 30s` |
| Files not updating | Check `--update` flag or use `--checksum` |

### Debug

```bash
# Verbose debug output
rclone copy /data remote:backup -vv

# Show config location
rclone config file

# Test connectivity
rclone lsd remote: --timeout 10s
```

## Supported Backends (Common)

| Backend | Type | Use Case |
|---------|------|----------|
| Amazon S3 | `s3` | Object storage, backups |
| Google Drive | `drive` | File storage, sharing |
| Azure Blob | `azureblob` | Object storage |
| Backblaze B2 | `b2` | Low-cost backup |
| SFTP | `sftp` | Remote servers |
| FTP | `ftp` | Legacy servers |
| Local | `local` | Disk-to-disk |
| Crypt | `crypt` | Encryption layer on any backend |
| Google Cloud Storage | `gcs` | GCP object storage |
| Dropbox | `dropbox` | Personal cloud |
| OneDrive | `onedrive` | Microsoft cloud |
| MinIO | `s3` | Self-hosted S3 |
| Wasabi | `s3` | Low-cost S3-compatible |
