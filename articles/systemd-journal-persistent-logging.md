# Enable Persistent systemd Journal Logging

By default on RHEL 7, 8, 9, and 10, systemd stores journal logs only in a small ring-buffer at `/run/log/journal` which is not persistent — logs do not survive a reboot. This guide shows how to enable persistent logging to `/var/log/journal`.

## Why Logs Are Lost by Default

- Journal data is stored in `/run/log/journal` (tmpfs, cleared on reboot)
- The directory `/var/log/journal` does not exist by default
- Without persistence, you cannot investigate issues that occurred before the last boot

## Method 1: Create the Directory

The simplest approach — create `/var/log/journal` and the system starts storing logs there automatically.

### RHEL 7 and 8

```bash
mkdir -p /var/log/journal
systemctl restart systemd-journald.service
```

### RHEL 9 and 10

```bash
mkdir -p /var/log/journal
systemctl restart systemd-journald.service
journalctl --flush
```

> **Note:** On RHEL 9+, `journalctl --flush` is required to move in-memory logs to disk. This handles systems where the root filesystem is writable early but `/var` needs additional setup (ACLs, NFS mounts, etc.) before journald writes to it.

## Method 2: Set Storage=persistent in journald.conf

Explicitly configure persistent storage in the journal configuration.

### RHEL 7, 8, and 9

```bash
sed -i 's/#Storage=auto/Storage=persistent/' /etc/systemd/journald.conf
```

### RHEL 10

In RHEL 10, main configuration files moved to `/usr/lib/systemd/`. Use an override in `/etc/systemd/`:

```bash
cat > /etc/systemd/journald.conf << EOF
[Journal]
Storage=persistent
EOF
```

### Restart the Service

#### RHEL 7 and 8

```bash
systemctl restart systemd-journald.service
```

#### RHEL 9 and 10

```bash
systemctl restart systemd-journald.service
journalctl --flush
```

## Storage Options

| Value | Behavior |
|-------|----------|
| `persistent` | Store on disk at `/var/log/journal` (created if needed), fallback to `/run/log/journal` during early boot |
| `volatile` | Store only in memory at `/run/log/journal` |
| `auto` (default) | Store to `/var/log/journal` if directory exists, otherwise `/run/log/journal` |
| `none` | Disable all journal storage |

## Verify

```bash
# Check journal is writing to /var/log/journal
ls -la /var/log/journal/

# Show logs from a previous boot
journalctl -b -1

# Show disk usage
journalctl --disk-usage
```

## Configure Retention

Edit `/etc/systemd/journald.conf` to control log size and rotation:

```bash
[Journal]
Storage=persistent
SystemMaxUse=500M
SystemKeepFree=1G
MaxRetentionSec=1month
```

| Option | Description |
|--------|-------------|
| `SystemMaxUse` | Maximum disk space for journal files |
| `SystemKeepFree` | Minimum free space to leave on disk |
| `RuntimeMaxUse` | Maximum memory for volatile journal |
| `MaxRetentionSec` | Delete entries older than this duration |
| `MaxFileSec` | Rotate journal files after this duration |

Apply changes:

```bash
systemctl restart systemd-journald.service
```

## Manual Rotation

```bash
# Rotate journal files
journalctl --rotate

# Vacuum by size (keep only 200M)
journalctl --vacuum-size=200M

# Vacuum by time (keep only 7 days)
journalctl --vacuum-time=7d
```

## Configuration File Locations

| RHEL Version | Config File |
|-------------|-------------|
| RHEL 7, 8, 9 | `/etc/systemd/journald.conf` |
| RHEL 10 | `/usr/lib/systemd/journald.conf` (defaults), `/etc/systemd/journald.conf` (overrides) |

## rsyslog Consideration

After enabling persistent journal logging, evaluate whether you still need rsyslog. The journal can replace rsyslog for many use cases, but rsyslog may still be needed for:

- Remote log forwarding to a central syslog server
- Custom log file formats required by third-party tools
- Compliance requirements specifying traditional log files
