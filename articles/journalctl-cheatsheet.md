# journalctl Cheatsheet

Query and display log entries from the systemd journal — filtering by unit, time, priority, boot, and more.

## Basic Viewing

| Command | What It Does |
|---------|--------------|
| `journalctl` | Show all logs (oldest first), paged |
| `journalctl -e` | Jump to the end of the journal (most recent entries) |
| `journalctl -f` | Follow new entries in real time (like `tail -f`) |
| `journalctl -n 50` | Show the last 50 entries |
| `journalctl -n 50 --no-pager` | Last 50 entries without paging (for scripts) |
| `journalctl -r` | Reverse order (newest first) |
| `journalctl --no-pager` | Print all output without paging |

## Filtering by Unit

| Command | What It Does |
|---------|--------------|
| `journalctl -u nginx.service` | Logs for a specific service |
| `journalctl -u nginx.service -u php-fpm.service` | Logs for multiple services (ORed) |
| `journalctl -u nginx.service -f` | Follow a specific service's logs |
| `journalctl -u nginx.service -f --since "5 min ago"` | Follow with recent context before the live stream |
| `journalctl -u nginx.service -n 100` | Last 100 lines for a service |
| `journalctl -u 'nginx*'` | Pattern matching on unit names |
| `journalctl -u 'nginx@*'` | Match templated instances (`nginx@foo`, `nginx@bar`) |
| `journalctl --user-unit=myapp.service` | Logs for a user-level service |
| `journalctl -D /var/log/journal/<machine-id>/` | Read from a specific journal directory |

## Filtering by Time

| Command | What It Does |
|---------|--------------|
| `journalctl --since "2024-03-15 08:00:00"` | Since a specific date/time |
| `journalctl --since "1 hour ago"` | Last hour |
| `journalctl --since "30 min ago"` | Last 30 minutes |
| `journalctl --since today` | Since midnight today |
| `journalctl --since yesterday --until today` | All of yesterday |
| `journalctl --since "2024-03-15" --until "2024-03-16"` | Date range |
| `journalctl --since "-5m"` | Relative: last 5 minutes |
| `journalctl --since "-2h" --until "-1h"` | Between 2 hours ago and 1 hour ago |

Time format: `YYYY-MM-DD HH:MM:SS`, or relative with `-` prefix, or keywords `today`, `yesterday`, `tomorrow`, `now`.

## Filtering by Boot

| Command | What It Does |
|---------|--------------|
| `journalctl -b` | Current boot only |
| `journalctl -b -1` | Previous boot |
| `journalctl -b -2` | Two boots ago |
| `journalctl --list-boots` | List all recorded boots with IDs and timestamps |
| `journalctl -b <boot-id>` | Specific boot by ID |

## Filtering by Priority

Priority levels (same as syslog):

| Level | Name | Meaning |
|-------|------|---------|
| 0 | `emerg` | System is unusable |
| 1 | `alert` | Immediate action required |
| 2 | `crit` | Critical conditions |
| 3 | `err` | Error conditions |
| 4 | `warning` | Warning conditions |
| 5 | `notice` | Normal but significant |
| 6 | `info` | Informational |
| 7 | `debug` | Debug messages |

| Command | What It Does |
|---------|--------------|
| `journalctl -p err` | Errors and above (emerg, alert, crit, err) |
| `journalctl -p warning` | Warnings and above |
| `journalctl -p debug` | All messages including debug |
| `journalctl -p err..err` | Exactly err only (not crit, not warning) |
| `journalctl -p err..crit` | Range: only crit and err |
| `journalctl -p 0..3` | Same as above, numeric |
| `journalctl -b -p err` | Errors from current boot |
| `journalctl -b -p err -x` | Errors this boot with catalog explanations |

Note: `-p err` means "err **and worse**" (emerg, alert, crit, err). If you want exactly one level, use the range syntax: `-p err..err`.

## Kernel Messages

Unlike `dmesg`, `journalctl -k` persists across boots — you can read the kernel ring buffer from previous boots.

| Command | What It Does |
|---------|--------------|
| `journalctl -k` | Kernel messages (current boot, equivalent to `dmesg`) |
| `journalctl -k -b -1` | Kernel messages from previous boot (impossible with `dmesg`) |
| `journalctl -k -p err` | Only kernel errors |
| `journalctl -k --since "5 min ago"` | Recent kernel messages |

## Grep and Pattern Matching

| Command | What It Does |
|---------|--------------|
| `journalctl -g "error\|fail"` | PCRE regex search in MESSAGE field |
| `journalctl -g "OOM" -p err` | Combine grep with priority filter |
| `journalctl -g "timeout" -u nginx.service` | Grep within a unit's logs |
| `journalctl --case-sensitive=no -g "ERROR"` | Force case-insensitive match |

Note: `-g` uses PCRE (Perl-compatible) regex. If the pattern is all lowercase, matching is case-insensitive by default.

## Field Matching

| Command | What It Does |
|---------|--------------|
| `journalctl _SYSTEMD_UNIT=sshd.service` | Match by field directly |
| `journalctl _PID=1234` | Logs from a specific PID |
| `journalctl _UID=1000` | Logs from a specific user ID |
| `journalctl _COMM=nginx` | Logs from a specific command name |
| `journalctl _EXE=/usr/sbin/nginx` | Logs from a specific executable |
| `journalctl _HOSTNAME=webserver01` | Logs from a specific host |
| `journalctl _TRANSPORT=kernel` | Only kernel transport messages |
| `journalctl /usr/sbin/sshd` | Shortcut: match by executable path |

Multiple field matches are ANDed. Same-field matches are ORed. Use `+` for explicit OR between different fields:

```bash
# AND: entries must match both
journalctl _SYSTEMD_UNIT=sshd.service _PID=1234

# OR on same field: entries matching either unit
journalctl _SYSTEMD_UNIT=sshd.service _SYSTEMD_UNIT=nginx.service

# OR across fields: messages from sshd PID 1234 OR anything from nginx
journalctl _SYSTEMD_UNIT=sshd.service _PID=1234 + _SYSTEMD_UNIT=nginx.service
```

## Output Formats

| Command | What It Does |
|---------|--------------|
| `journalctl -o short` | Default syslog-like format |
| `journalctl -o short-iso` | ISO 8601 timestamps |
| `journalctl -o short-precise` | Microsecond precision timestamps |
| `journalctl -o short-full` | Full timestamps (same format `--since` accepts) |
| `journalctl -o verbose` | All fields, structured |
| `journalctl -o json` | JSON format (one object per line) |
| `journalctl -o json-pretty` | Pretty-printed JSON |
| `journalctl -o cat` | Message text only (no metadata) |
| `journalctl -o with-unit` | Prefix with unit name instead of syslog identifier |
| `journalctl -o short-monotonic` | Monotonic timestamps (time since boot) |

### Output modifiers

| Option | What It Does |
|--------|--------------|
| `--output-fields=MESSAGE,_PID,_COMM` | Limit which fields appear (verbose/json/cat modes) |
| `--no-hostname` | Hide the hostname field |
| `--utc` | Show timestamps in UTC |
| `-x` | Add catalog explanation texts to known message IDs |
| `-a` | Show all fields in full (don't truncate binary data) |
| `--truncate-newline` | Show only first line of multi-line messages |

## Syslog Identifier

| Command | What It Does |
|---------|--------------|
| `journalctl -t sshd` | Filter by syslog identifier |
| `journalctl -t kernel` | Kernel syslog messages |
| `journalctl -t sudo` | All sudo messages |
| `journalctl -t systemd` | Messages from systemd itself |
| `journalctl -T cron` | Exclude a specific identifier |
| `journalctl -t sshd -t sudo` | Multiple identifiers (ORed) |

## Disk Usage and Maintenance

| Command | What It Does |
|---------|--------------|
| `journalctl --disk-usage` | Total disk space used by journal files |
| `journalctl --rotate` | Force rotation of journal files |
| `journalctl --vacuum-size=500M` | Remove old journals until under 500 MiB |
| `journalctl --vacuum-time=7d` | Remove journals older than 7 days |
| `journalctl --vacuum-files=5` | Keep only 5 journal files |
| `journalctl --rotate --vacuum-size=200M` | Rotate then vacuum (most effective) |
| `journalctl --sync` | Flush unwritten data to disk |
| `journalctl --flush` | Move runtime journal (`/run`) to persistent (`/var`) |
| `journalctl --verify` | Check journal file integrity |

## Listing Available Data

| Command | What It Does |
|---------|--------------|
| `journalctl -F _SYSTEMD_UNIT` | All unit names that have logged |
| `journalctl -F _COMM` | All command names that have logged |
| `journalctl -F PRIORITY` | All priority levels present |
| `journalctl -F _HOSTNAME` | All hostnames in the journal |
| `journalctl -N` | All field names currently in use |

## Recipes

### Troubleshoot a failed service

```bash
# Check current status (shows when service started)
systemctl status nginx.service

# See recent logs for the service
journalctl -u nginx.service -n 50 --no-pager

# See logs from the last start attempt
journalctl -u nginx.service -b --since "5 min ago"

# All available logs for the service (limited by journal retention)
journalctl -u nginx.service

# Follow logs while restarting
journalctl -u nginx.service -f &
systemctl restart nginx.service
```

### Find why a server rebooted

```bash
# List all boots
journalctl --list-boots

# Check end of previous boot for crash/shutdown reason
journalctl -b -1 -n 100 -r

# Kernel messages from previous boot (OOM, panic, etc.)
journalctl -b -1 -k -p err
```

### Audit SSH logins

```bash
# All SSH activity
journalctl -u sshd.service --since today

# Failed logins only
journalctl -u sshd.service -g "Failed|Invalid" --since today

# Accepted logins
journalctl -u sshd.service -g "Accepted" --since today
```

### Export logs for analysis

```bash
# JSON export for jq processing
journalctl -u myapp.service --since today -o json --no-pager | jq '.MESSAGE'

# Filter JSON output by priority with jq
journalctl -u ssh.service --since '1 hour ago' -o json | jq -r 'select(.PRIORITY | tonumber <= 4) | .MESSAGE'

# Export to a file for shipping
journalctl --since "1 hour ago" -o export > /tmp/journal-export.bin

# Import on another machine
systemd-journal-remote --output=/var/log/journal/remote/ /tmp/journal-export.bin
```

### Find OOM kills

```bash
journalctl -k -g "Out of memory|oom_reaper|oom-kill"
journalctl -k -g "Killed process"
```

### Continuous logging to a file with timestamps

```bash
journalctl -f -u myapp.service -o short-iso --no-pager >> /tmp/myapp-debug.log
```

### Logs from a specific process

```bash
# By PID (if you know it)
journalctl _PID=$(pidof nginx | awk '{print $1}')

# By executable path
journalctl /usr/sbin/nginx --since "1 hour ago"
```

## Persistent vs Volatile Storage

### Log Storage Locations

- **`/var/log/journal/<machine-id>/`** — Persistent storage (survives reboots)
- **`/run/log/journal/<machine-id>/`** — Runtime storage (cleared on reboot)
- Machine ID is stored in `/etc/machine-id`

Journal files are binary format (not plain text) — access them through `journalctl` rather than viewing directly. Files use the `.journal` extension.

### Storage Configuration

By default on some distros, journal logs are stored only in `/run/log/journal/` (volatile — lost on reboot). To enable persistent storage:

```bash
# Create the persistent directory
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal

# Restart journald to pick it up
systemctl restart systemd-journald
```

Or configure in `/etc/systemd/journald.conf`:

```ini
[Journal]
Storage=persistent
SystemMaxUse=2G
SystemMaxFileSize=256M
MaxRetentionSec=1month
```

| Storage option | Behavior |
|----------------|----------|
| `auto` (default) | Use `/var/log/journal/` if it exists, else `/run/log/journal/` |
| `persistent` | Store in `/var/log/journal/` (survives reboot) |
| `volatile` | Store in `/run/log/journal/` only (lost on reboot) |
| `none` | Disable all storage (only forwarding) |

### Checking Persistence

```bash
# If you see multiple boot entries, persistent logging is enabled
journalctl --list-boots

# Check the Storage setting
grep Storage /etc/systemd/journald.conf
```

## journald.conf Reference

**Location:** `/etc/systemd/journald.conf`

```ini
[Journal]
Storage=auto              # auto, persistent, volatile, none
Compress=yes              # Compress logs (yes/no)
Seal=yes                  # Forward Secure Sealing (yes/no)
SplitMode=uid             # Split journals by user ID

# Size limits
SystemMaxUse=             # Max disk space (default: 10% of filesystem, max 4G)
SystemKeepFree=           # Min free space to maintain (default: 15%)
SystemMaxFileSize=        # Max size per journal file (default: 1/8 of SystemMaxUse)
SystemMaxFiles=100        # Max number of journal files to keep

# Time limits
MaxRetentionSec=          # Max time to keep logs (default: none — size-based only)
MaxFileSec=1month         # Max time before rotating file

# Rate limiting
RateLimitIntervalSec=30s  # Rate limit interval
RateLimitBurst=10000      # Max messages per interval

# Forwarding
ForwardToSyslog=no        # Forward to syslog (yes/no)
ForwardToKMsg=no          # Forward to kernel log (yes/no)
ForwardToConsole=no       # Forward to console (yes/no)
ForwardToWall=yes         # Forward wall messages (yes/no)
```

### Key Configuration Settings

| Setting | Purpose |
|---------|---------|
| `SystemMaxUse=` | Max disk space for journal |
| `SystemKeepFree=` | Min free disk space to maintain |
| `SystemMaxFileSize=` | Max size per journal file (default: 1/8 of SystemMaxUse) |
| `SystemMaxFiles=` | Max number of journal files to keep |
| `MaxRetentionSec=` | Max time to keep logs |
| `MaxFileSec=` | Max time before rotating to new file |
| `RateLimitIntervalSec=` | Rate limit window |
| `RateLimitBurst=` | Max messages per rate limit window |

## Journal File Internals

### File Format

- **Binary format** — Structured binary, not plain text
- **Indexed** — Logs are indexed for fast querying and filtering
- **Compressed** — Entries are automatically compressed to save disk space
- **Signed** — Can be cryptographically signed (Forward Secure Sealing) to ensure integrity

### File Rotation

Journal files are automatically rotated based on:

| Trigger | Setting | Default |
|---------|---------|---------|
| File size | `SystemMaxFileSize=` | 1/8 of `SystemMaxUse` |
| File age | `MaxFileSec=` | 1 month |
| System boot | — | New journal file created after each boot |

### Performance Characteristics

- **Fast queries** — Binary format enables quick filtering without scanning entire files
- **Memory-mapped** — Journal files are memory-mapped for read efficiency
- **Compression** — Reduces disk space but adds slight CPU overhead on writes
- **Indexed** — Small write overhead vs plain text, but much faster reads

## Journal vs Syslog

| Feature | systemd Journal | Traditional Syslog |
|---------|----------------|--------------------|
| Format | Binary, indexed | Plain text |
| Metadata | Rich (PID, UID, boot ID, unit, etc.) | Limited |
| Query speed | Fast (indexed) | Slow (grep-based) |
| Compression | Built-in | Manual/external |
| Rotation | Automatic | Manual (logrotate) |
| Integrity | Cryptographic signing (FSS) | None |
| Boot separation | Native (`-b` flag) | Manual parsing |
| Structured fields | Yes (arbitrary key=value) | No |
| Tools | `journalctl` | `grep`, `awk`, `sed` |

## Troubleshooting

### If logs aren't appearing

```bash
# 1. Check journald is running
systemctl status systemd-journald

# 2. Check disk space
df -h /var/log

# 3. Check permissions
ls -la /var/log/journal/

# 4. Verify journal integrity
journalctl --verify

# 5. Check rate limiting (messages may be dropped)
journalctl -u systemd-journald --since "5 min ago" | grep -i "suppressed"

# 6. Restart journald
systemctl restart systemd-journald
```

### Journal corruption

```bash
# Verify all journal files
journalctl --verify

# If corrupted, rotate and vacuum to get fresh files
journalctl --rotate
journalctl --vacuum-time=1s  # remove all old (corrupted) files
```

## Security Considerations

- **Access control** — Journal files readable by root and `systemd-journal` group members only
- **Sensitive data** — Logs may contain passwords, tokens, or PII in command arguments
- **Forward Secure Sealing (FSS)** — Cryptographic signing prevents retroactive tampering (enable with `Seal=yes`)
- **Disk exhaustion** — Unlimited logging can fill disk; always set `SystemMaxUse=` and `SystemKeepFree=`
- **Log forwarding** — Consider forwarding to remote syslog or a log aggregator for audit trails and redundancy

## Access Control

By default, only `root` and members of specific groups can read the system journal:

| Group | Access |
|-------|--------|
| `systemd-journal` | Read access to system journal |
| `adm` | Read access (traditional admin group) |
| `wheel` | Read access (traditional admin group) |

Regular users can always read their own user journal.

```bash
# Grant a user access to system logs
usermod -aG systemd-journal username
```

## Gotchas

- **Not every log is in the journal.** Nginx access logs, Apache per-vhost logs, MySQL slow query logs, and applications that write directly to files still live under `/var/log`. `journalctl` covers stdout/stderr from systemd units plus the kernel and anything talking to the syslog socket — which is most daemons, but not all.
- **`-p err` includes everything worse too.** It means err + crit + alert + emerg. If you want exactly one level, use `-p err..err`.
- **Default view is oldest first.** Surprising if you expect newest. Use `-e` (jump to end) or `-r` (reverse) when you want recent.
- **`-f` follows; without it you get a snapshot.** Same difference as `tail` vs `tail -f`.
- **The pager is `less`.** Press `q` to quit, `/` to search, `G` to jump to end. `Ctrl-C` doesn't exit the pager — it just interrupts the search.
- **`-u nginx` won't match template instances.** If you have `nginx@foo.service` and `nginx@bar.service`, use `-u 'nginx@*'` or be explicit.
- **Rotated journal files** are usually merged transparently, but for forensics on archived journals use `-D /var/log/journal/<machine-id>/` to point at a specific directory.

## Tips

- `journalctl -xe` is the quickest "show me what just went wrong" command — jumps to the end and adds catalog explanations.
- `journalctl -u <service> -f` is the workhorse — the `tail -f` reflex unified and structured. Pair with `--since '5 min ago'` to get recent context before the live stream starts.
- Combine `-b` with `-u` and `-p` for focused troubleshooting: `journalctl -b -u sshd -p err`.
- Use `-o verbose` to see all structured fields when the short output doesn't give enough context.
- Use `-o cat` for just the message text — pipes cleanly to `grep`, `awk`, or `less` without timestamp/host noise.
- The `-q` flag suppresses informational banners ("-- Journal begins at...") — useful in scripts.
- When attaching logs to bug reports, don't use `-x` (catalog messages add noise for developers).
- `journalctl` pages through `less` by default. Press `/` to search, `G` to jump to end, `g` to jump to start, `q` to quit.
- If you find yourself building a big `grep` pipeline, check whether a structured filter already exists (`-u`, `-p`, `-t`, `-g`, `_FIELD=value`) — it's almost always faster and more precise.

## See Also

- [top Cheatsheet](articles/top-cheatsheet.md) — process-level CPU and memory view
- [ps Cheatsheet](articles/ps-cheatsheet.md) — snapshot of running processes
- [Bash Essentials Guide](articles/bash-essentials-guide.md) — shell fundamentals for scripting log pipelines
