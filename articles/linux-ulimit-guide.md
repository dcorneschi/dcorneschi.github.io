# Linux ulimit Guide

`ulimit` controls per-process resource limits in Linux. These limits prevent a single user or process from consuming all system resources — file descriptors, memory, CPU time, process count, and more.

## Soft vs Hard Limits

Every resource has two limits:

| Type | Description |
|------|-------------|
| **Soft limit** | The effective limit enforced right now. A process can raise it up to the hard limit. |
| **Hard limit** | The ceiling. Only root can raise it. Once lowered by a non-root user, it cannot be raised again. |

```bash
# View soft limits
ulimit -Sa

# View hard limits
ulimit -Ha

# View a specific soft limit (open files)
ulimit -Sn

# View a specific hard limit (open files)
ulimit -Hn
```

A process starts with the soft limit active. It can call `setrlimit()` to raise its soft limit up to the hard limit without needing root. This is why many applications set their own soft limits at startup — they rely on the hard limit being high enough.

## Common Limits

| Flag | Resource | Typical Default |
|------|----------|-----------------|
| `-n` | Open files (file descriptors) | 1024 |
| `-u` | Max user processes (nproc) | 4096–63000 |
| `-f` | Max file size (blocks) | unlimited |
| `-c` | Core file size (blocks) | 0 (disabled) |
| `-s` | Stack size (KB) | 8192 |
| `-v` | Virtual memory (KB) | unlimited |
| `-m` | Max resident set size (KB) | unlimited |
| `-l` | Max locked memory (KB) | 64 |
| `-t` | CPU time (seconds) | unlimited |
| `-i` | Pending signals | ~30000 |
| `-q` | POSIX message queue size (bytes) | 819200 |
| `-x` | Max file locks | unlimited |
| `-e` | Max scheduling priority | 0 |
| `-r` | Max real-time priority | 0 |
| `-p` | Pipe buffer size (512-byte blocks) | 8 |

## Viewing Current Limits

```bash
# All limits for current shell
ulimit -a

# All soft limits
ulimit -Sa

# All hard limits
ulimit -Ha

# Specific: open files
ulimit -n

# Specific: max user processes
ulimit -u

# Specific: stack size
ulimit -s

# Check limits for a running process (PID)
cat /proc/<PID>/limits
```

### /proc/PID/limits

The kernel exposes per-process limits in `/proc`:

```bash
cat /proc/1234/limits
```

```
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        0                    unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             63116                63116                processes
Max open files            1024                 1048576              files
Max locked memory         65536                65536                bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       63116                63116                signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us
```

## Setting Limits for the Current Session

```bash
# Set open files (soft)
ulimit -n 65536

# Set open files (hard) — requires root
ulimit -Hn 65536

# Set max processes
ulimit -u 8192

# Enable core dumps (unlimited size)
ulimit -c unlimited

# Set stack size to 16MB
ulimit -s 16384

# Disable core dumps
ulimit -c 0

# Limit max file size to 100KB (in 512-byte blocks)
ulimit -f 200

# Limit virtual memory to 2GB
ulimit -v 2097152

# Set multiple limits at once
ulimit -n 65536 -u 8192 -l unlimited
```

> **Note:** Changes made with `ulimit` only apply to the current shell session and its child processes. They do not persist across reboots or new logins.

### Preventing Fork Bombs

A fork bomb like `:(){ :|:& };:` spawns processes exponentially until the system is unresponsive. Limiting `nproc` contains the damage:

```bash
# Limit current session to 200 processes
ulimit -u 200

# Test — a fork bomb will now hit the wall:
# bash: fork: retry: Resource temporarily unavailable
```

### Limiting File Size

Prevent users from creating large files that fill disk:

```bash
# Limit to 100KB (value is in 512-byte blocks)
ulimit -f 200

# Try to create a larger file
cat /dev/zero > testfile
# File size limit exceeded (core dumped)

ls -lh testfile
# -rw-r--r-- 1 user user 100K ...
```

### Limiting Virtual Memory

Cap how much virtual address space a single process can map:

```bash
# Limit to 1GB
ulimit -v 1048576
```

Useful for catching memory leaks during testing — the process receives ENOMEM when it exceeds the cap rather than consuming all system RAM.

## Persistent Configuration — /etc/security/limits.conf

To make limits persistent across logins, configure `/etc/security/limits.conf` or drop-in files in `/etc/security/limits.d/`.

### Syntax

```
<domain>  <type>  <item>  <value>
```

| Field | Values |
|-------|--------|
| `domain` | Username, `@group`, `*` (all users), or `%group` |
| `type` | `soft`, `hard`, or `-` (both) |
| `item` | `nofile`, `nproc`, `core`, `stack`, `memlock`, etc. |
| `value` | Number or `unlimited` |

### Examples

```
# /etc/security/limits.conf

# All users — open files
*               soft    nofile          65536
*               hard    nofile          65536

# All users — max processes
*               soft    nproc           4096
*               hard    nproc           63116

# Specific user — higher limits
alice           soft    nofile          131072
alice           hard    nofile          131072

# Group — database team
@dba            soft    nofile          131072
@dba            hard    nofile          131072
@dba            soft    memlock         unlimited
@dba            hard    memlock         unlimited

# Disable core dumps for all
*               hard    core            0

# Enable core dumps for developers
@developers     soft    core            unlimited
@developers     hard    core            unlimited

# Increase stack size for Java apps
appuser         soft    stack           32768
appuser         hard    stack           32768
```

### Drop-in Files in /etc/security/limits.d/

```bash
# /etc/security/limits.d/90-nofile.conf
*  soft  nofile  65536
*  hard  nofile  65536

# /etc/security/limits.d/90-nproc.conf
*  soft  nproc   4096
*  hard  nproc   63116
```

Files are read in alphabetical order. Use numeric prefixes to control load order.

> **Note:** On RHEL/CentOS 6+, `/etc/security/limits.d/20-nproc.conf` ships by default and overrides `limits.conf` for `nproc`. Check this file if your nproc settings are being ignored.

## PAM — Why limits.conf Needs pam_limits

`/etc/security/limits.conf` is read by the `pam_limits.so` PAM module. If PAM is not configured to load it, your limits won't apply.

Verify that `/etc/pam.d/common-session` (Debian) or `/etc/pam.d/system-auth` (RHEL) includes:

```
session  required  pam_limits.so
```

Services that bypass PAM (like some systemd units or cron jobs) won't respect `limits.conf`. Use systemd's `LimitNOFILE=` directives for those instead.

## systemd Service Limits

For services managed by systemd, set limits in the unit file rather than `limits.conf`:

```ini
# /etc/systemd/system/myapp.service or override
[Service]
LimitNOFILE=65536
LimitNPROC=4096
LimitCORE=infinity
LimitMEMLOCK=infinity
LimitSTACK=33554432
```

| systemd Directive | Equivalent ulimit | limits.conf item |
|-------------------|-------------------|-------------------|
| `LimitNOFILE` | `-n` | `nofile` |
| `LimitNPROC` | `-u` | `nproc` |
| `LimitCORE` | `-c` | `core` |
| `LimitMEMLOCK` | `-l` | `memlock` |
| `LimitSTACK` | `-s` | `stack` |
| `LimitAS` | `-v` | `as` |
| `LimitCPU` | `-t` | `cpu` |
| `LimitFSIZE` | `-f` | `fsize` |
| `LimitSIGPENDING` | `-i` | `sigpending` |
| `LimitMSGQUEUE` | `-q` | `msgqueue` |

### Applying Changes

```bash
# Reload systemd after changing unit files
sudo systemctl daemon-reload
sudo systemctl restart myapp

# Check effective limits for a running service
sudo cat /proc/$(systemctl show -p MainPID --value myapp.service)/limits
```

### Override Without Editing the Unit File

```bash
sudo systemctl edit myapp.service
```

This creates `/etc/systemd/system/myapp.service.d/override.conf`:

```ini
[Service]
LimitNOFILE=131072
```

## System-Wide Limits — /etc/sysctl.conf

Some limits are kernel-wide and cannot be set per-user. These are configured via `sysctl`:

```bash
# Max file descriptors system-wide
cat /proc/sys/fs/file-max
sysctl fs.file-max

# Max inodes
sysctl fs.inode-max

# Async I/O requests
sysctl fs.aio-max-nr

# Max number of inotify watches
cat /proc/sys/fs/inotify/max_user_watches

# Max PID value (affects total processes)
cat /proc/sys/kernel/pid_max

# Shared memory max segment size (bytes)
sysctl kernel.shmmax

# Shared memory total pages
sysctl kernel.shmall

# Virtual memory map areas
sysctl vm.max_map_count

# Current system-wide FD usage
cat /proc/sys/fs/file-nr
# Output: allocated  free  max
```

### Setting System-Wide Limits

```bash
# Temporary (until reboot)
sudo sysctl -w fs.file-max=2097152
sudo sysctl -w fs.inotify.max_user_watches=524288
sudo sysctl -w kernel.pid_max=4194304

# Persistent — add to /etc/sysctl.conf or /etc/sysctl.d/
cat > /etc/sysctl.d/99-limits.conf << 'EOF'
fs.file-max = 2097152
fs.inode-max = 2097152
fs.aio-max-nr = 1048576
fs.inotify.max_user_watches = 524288
kernel.pid_max = 4194304
vm.max_map_count = 262144
EOF

# Apply
sudo sysctl -p /etc/sysctl.d/99-limits.conf
```

### Network-Related sysctls

For high-connection-count servers, raise these alongside `nofile`:

```bash
# Max listen backlog (connections waiting to be accepted)
sudo sysctl -w net.core.somaxconn=65535

# TCP SYN backlog
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=8192

# Ephemeral port range (more outbound connections)
sudo sysctl -w net.ipv4.ip_local_port_range="1024 65535"
```

### Relationship Between sysctl and ulimit

```
fs.file-max  →  system-wide ceiling (all processes combined)
ulimit -n    →  per-process ceiling (within that system-wide limit)
```

A user can set `ulimit -n 1000000`, but if `fs.file-max` is only `65536`, processes will still fail when the system-wide limit is exhausted.

## Common Scenarios

### High-Traffic Web Servers (nginx, Apache)

```
# /etc/security/limits.d/90-webserver.conf
www-data  soft  nofile  65536
www-data  hard  nofile  65536

# /etc/systemd/system/nginx.service.d/override.conf
[Service]
LimitNOFILE=65536
```

```bash
# Also set worker_connections in nginx.conf to match
sudo sysctl -w fs.file-max=2097152
```

### Database Servers (PostgreSQL, MySQL, Oracle)

```
# /etc/security/limits.d/90-database.conf
postgres  soft  nofile    131072
postgres  hard  nofile    131072
postgres  soft  nproc     65536
postgres  hard  nproc     65536
postgres  soft  memlock   unlimited
postgres  hard  memlock   unlimited

mysql     soft  nofile    131072
mysql     hard  nofile    131072
```

### Java Applications

Java processes often need high file descriptor and memory limits:

```
# /etc/security/limits.d/90-java.conf
appuser   soft  nofile    65536
appuser   hard  nofile    65536
appuser   soft  nproc     4096
appuser   hard  nproc     4096
appuser   soft  stack     32768
appuser   hard  stack     32768
```

### Docker and Containers

Docker inherits limits from the daemon. Set them on the Docker service:

```ini
# /etc/systemd/system/docker.service.d/override.conf
[Service]
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity
```

Set default limits for all containers in the Docker daemon config:

```json
// /etc/docker/daemon.json
{
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    },
    "nproc": {
      "Name": "nproc",
      "Hard": 32768,
      "Soft": 32768
    }
  }
}
```

Per-container limits via `docker run`:

```bash
docker run --ulimit nofile=65536:65536 --ulimit nproc=4096:4096 myimage
```

In `docker-compose.yml`:

```yaml
services:
  myapp:
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
      nproc:
        soft: 4096
        hard: 4096
```

### Enabling Core Dumps

```bash
# Enable for current session
ulimit -c unlimited

# Persistent
echo "* soft core unlimited" | sudo tee -a /etc/security/limits.d/90-core.conf
echo "* hard core unlimited" | sudo tee -a /etc/security/limits.d/90-core.conf

# Set core dump location
echo "/tmp/core.%e.%p.%t" | sudo tee /proc/sys/kernel/core_pattern

# Persistent location
echo "kernel.core_pattern = /tmp/core.%e.%p.%t" | sudo tee -a /etc/sysctl.d/99-core.conf
sudo sysctl -p /etc/sysctl.d/99-core.conf
```

### Elasticsearch / OpenSearch

```
# /etc/security/limits.d/90-elasticsearch.conf
elasticsearch  soft  nofile    65536
elasticsearch  hard  nofile    65536
elasticsearch  soft  memlock   unlimited
elasticsearch  hard  memlock   unlimited
elasticsearch  soft  nproc     4096
elasticsearch  hard  nproc     4096
```

```bash
# Required sysctl settings
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count = 262144" | sudo tee -a /etc/sysctl.d/99-elasticsearch.conf
```

## Troubleshooting

### Monitoring System Logs for Limit Errors

```bash
# Watch for limit-related errors in real time
tail -f /var/log/syslog | grep -i "too many\|cannot fork\|resource"
journalctl -f | grep -i "too many\|cannot fork"

# Check dmesg for fork failures
dmesg | grep -i "fork failed"
```

### Limits not taking effect after login

1. Verify PAM is loading `pam_limits.so`:
   ```bash
   grep pam_limits /etc/pam.d/common-session     # Debian/Ubuntu
   grep pam_limits /etc/pam.d/system-auth         # RHEL/CentOS
   ```

2. Check for overrides in `/etc/security/limits.d/`:
   ```bash
   ls -la /etc/security/limits.d/
   ```
   Files are read in alphabetical order — later files override earlier ones.

3. Log out and back in. `limits.conf` is applied at login time.

### Limits not applying to a systemd service

`limits.conf` only applies to PAM-authenticated sessions. systemd services bypass PAM. Set limits in the unit file:

```bash
sudo systemctl show myapp.service | grep Limit

# Check if DefaultLimitNOFILE is set system-wide
grep DefaultLimit /etc/systemd/system.conf
grep DefaultLimit /etc/systemd/user.conf
```

### "Too many open files" errors

```bash
# Check current usage for a process
ls /proc/<PID>/fd | wc -l

# Check the limit
cat /proc/<PID>/limits | grep "Max open files"

# Check limits for a named service using pgrep
cat /proc/$(pgrep nginx | head -1)/limits | grep "open files"

# System-wide FD usage
cat /proc/sys/fs/file-nr

# Find processes with the most open FDs using lsof
lsof | awk '{print $1}' | sort | uniq -c | sort -rn | head -10

# Same without lsof
for pid in /proc/[0-9]*; do
    echo "$(ls $pid/fd 2>/dev/null | wc -l) $pid $(cat $pid/cmdline 2>/dev/null | tr '\0' ' ')"
done | sort -rn | head -20

# Monitor FD usage in real time
watch -n 1 'cat /proc/sys/fs/file-nr'
```

### "fork: retry: Resource temporarily unavailable"

This means `nproc` (max processes) is exhausted:

```bash
# Check current process count for a user
ps -u username --no-headers | wc -l

# Process count by user (find who's consuming the most)
ps -eo user= | sort | uniq -c | sort -rn | head -10

# Check the limit
ulimit -u
cat /proc/<PID>/limits | grep "Max processes"

# On RHEL, check the default nproc override
cat /etc/security/limits.d/20-nproc.conf

# Check system PID max
cat /proc/sys/kernel/pid_max
```

### Limits reset after su or sudo

`su` and `sudo` start a new PAM session. Limits configured for the target user apply:

```bash
# Check limits after switching user
sudo -u appuser bash -c 'ulimit -a'
su - appuser -c 'ulimit -a'
```

## Increasing nofile Without Restarting the Process

You can raise the open files limit for a running process by writing directly to `/proc/PID/limits` — no restart needed.

```bash
# 1. Check the current limit for the user
su - username -s /bin/bash -c 'ulimit -n'

# 2. Count currently open file descriptors
ls -l /proc/<PID>/fd | wc -l

# 3. Increase the limit at runtime (soft:hard)
echo -n "Max open files=8192:8192" > /proc/<PID>/limits

# 4. Verify the new limit
grep "Max open files" /proc/<PID>/limits
```

> **Note:** Writing to `/proc/PID/limits` requires root and a kernel that supports it (Linux 2.6.24+, with `CONFIG_CHECKPOINT_RESTORE`). On older kernels or without that config, use `prlimit` instead:

```bash
# Using prlimit (util-linux)
prlimit --pid <PID> --nofile=8192:8192

# Verify
prlimit --pid <PID> --nofile
```

## Configuration Precedence

Multiple layers of configuration interact. When troubleshooting, check all of them:

| Layer | Applies to | Set by |
|-------|-----------|--------|
| Kernel sysctl (`fs.file-max`, `kernel.pid_max`) | Entire system | `/etc/sysctl.d/` |
| systemd service limits (`LimitNOFILE`) | Individual services | Unit file or override |
| `/etc/security/limits.conf` + `limits.d/` | PAM-authenticated logins | Root |
| `ulimit` command | Current shell session only | Any user (up to hard limit) |

For services managed by systemd, the effective limits come from the unit file — **not** from `limits.conf`. For interactive users over SSH or console, `limits.conf` applies via PAM.

## Monitoring Script

A simple script to monitor system-wide resource usage and alert when thresholds are crossed:

```bash
#!/bin/bash
# /usr/local/bin/monitor-limits.sh

FD_THRESHOLD=80  # alert at 80% usage

CURRENT_FD=$(awk '{print $1}' /proc/sys/fs/file-nr)
MAX_FD=$(cat /proc/sys/fs/file-max)
FD_PERCENT=$(awk "BEGIN {printf \"%.0f\", ($CURRENT_FD/$MAX_FD)*100}")

echo "File descriptors: ${CURRENT_FD}/${MAX_FD} (${FD_PERCENT}%)"
echo "Processes: $(ps aux --no-headers | wc -l) / $(cat /proc/sys/kernel/pid_max)"

if [ "$FD_PERCENT" -gt "$FD_THRESHOLD" ]; then
    echo "WARNING: File descriptor usage at ${FD_PERCENT}%"
fi

# Top 5 FD consumers
echo ""
echo "Top 5 processes by open FDs:"
for pid in /proc/[0-9]*; do
    echo "$(ls $pid/fd 2>/dev/null | wc -l) $(basename $pid) $(cat $pid/comm 2>/dev/null)"
done | sort -rn | head -5
```

```bash
chmod +x /usr/local/bin/monitor-limits.sh

# Run via cron every 5 minutes
# */5 * * * * /usr/local/bin/monitor-limits.sh >> /var/log/resource-limits.log
```

## Quick Reference

```bash
ulimit -a                      # show all current limits
ulimit -Sa                     # show all soft limits
ulimit -Ha                     # show all hard limits
ulimit -n                      # show open files limit
ulimit -n 65536                # set open files (soft)
ulimit -Hn 65536               # set open files (hard, needs root)
ulimit -u                      # show max processes
ulimit -u 4096                 # set max processes
ulimit -c unlimited            # enable core dumps
ulimit -c 0                    # disable core dumps
cat /proc/<PID>/limits         # limits for a running process
sysctl fs.file-max             # system-wide FD max
sysctl -w fs.file-max=2097152  # set system-wide FD max (temporary)
```

## Files

| File | Purpose |
|------|---------|
| `/etc/security/limits.conf` | Per-user/group persistent limits |
| `/etc/security/limits.d/*.conf` | Drop-in limit overrides |
| `/etc/sysctl.conf` | System-wide kernel parameters |
| `/etc/sysctl.d/*.conf` | Drop-in sysctl overrides |
| `/proc/sys/fs/file-max` | System-wide max file descriptors |
| `/proc/sys/fs/file-nr` | Current FD allocation stats |
| `/proc/<PID>/limits` | Per-process effective limits |
| `/proc/<PID>/fd/` | Open file descriptors for a process |
