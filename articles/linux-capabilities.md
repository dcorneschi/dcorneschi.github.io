# Linux Capabilities

## What Are Linux Capabilities?

Traditionally, Linux divided processes into two categories: privileged (UID 0 / root) with full access, and unprivileged (everything else). Capabilities break the monolithic root privilege into distinct units that can be independently granted or revoked.

Instead of running a process as root to bind to port 80, you can grant only `CAP_NET_BIND_SERVICE`. Instead of running as root to change file ownership, you can grant only `CAP_CHOWN`. This follows the principle of least privilege.

## How Capabilities Work

Capabilities are stored in three sets for each process:

| Set | Description |
|-----|-------------|
| **Permitted** | Upper limit of capabilities the process can use. Can be dropped but never re-acquired. |
| **Effective** | Capabilities currently active (checked by the kernel for permission). |
| **Inheritable** | Capabilities preserved across `execve()` (for passing to child processes). |

Additionally, there are two more sets:

| Set | Description |
|-----|-------------|
| **Bounding** | Upper limit of capabilities that can ever be gained. Cannot be re-added once removed. |
| **Ambient** | Capabilities preserved across `execve()` for non-privileged programs (since kernel 4.3). |

### How the Kernel Checks Capabilities

When a process performs a privileged operation, the kernel checks if the corresponding capability is in the process's **effective** set — not whether the UID is 0.

```
Process attempts privileged operation
    → Kernel checks effective capability set
        → Capability present? → Allow
        → Capability absent?  → Deny (EPERM)
```

## Capability List

### Network

| Capability | Description |
|---|---|
| `CAP_NET_ADMIN` | Network administration (interfaces, routing, firewall rules) |
| `CAP_NET_BIND_SERVICE` | Bind to ports below 1024 |
| `CAP_NET_BROADCAST` | Allow broadcasting and listening to multicast |
| `CAP_NET_RAW` | Use RAW and PACKET sockets (ping, tcpdump) |

### File and Filesystem

| Capability | Description |
|---|---|
| `CAP_CHOWN` | Change file ownership |
| `CAP_DAC_OVERRIDE` | Bypass file read/write/execute permission checks |
| `CAP_DAC_READ_SEARCH` | Bypass file read and directory search permissions |
| `CAP_FOWNER` | Bypass permission checks on operations that require UID match |
| `CAP_FSETID` | Don't clear set-user-ID and set-group-ID bits on file modification |
| `CAP_MKNOD` | Create special files with `mknod()` |
| `CAP_SETFCAP` | Set file capabilities |

### Process and User Management

| Capability | Description |
|---|---|
| `CAP_KILL` | Send signals to any process (bypass permission checks) |
| `CAP_SETUID` | Change process UID (setuid, setreuid, setresuid) |
| `CAP_SETGID` | Change process GID (setgid, setregid, setresgid) |
| `CAP_SETPCAP` | Modify process capabilities |
| `CAP_SYS_NICE` | Set nice values, scheduling policies, CPU affinity |
| `CAP_SYS_PACCT` | Use `acct()` for process accounting |
| `CAP_SYS_PTRACE` | Trace any process with ptrace, read /proc/PID/mem |
| `CAP_SYS_RESOURCE` | Override resource limits (ulimit, disk quota) |

### System Administration

| Capability | Description |
|---|---|
| `CAP_SYS_ADMIN` | Catch-all for various admin operations (mount, swapon, sethostname, etc.) |
| `CAP_SYS_BOOT` | Reboot the system |
| `CAP_SYS_CHROOT` | Use `chroot()` |
| `CAP_SYS_MODULE` | Load and unload kernel modules |
| `CAP_SYS_RAWIO` | Perform raw I/O (iopl, ioperm, /dev/mem access) |
| `CAP_SYS_TIME` | Set system clock |
| `CAP_SYSLOG` | Perform syslog operations, read kernel ring buffer |
| `CAP_WAKE_ALARM` | Set CLOCK_REALTIME/CLOCK_BOOTTIME alarms |
| `CAP_BLOCK_SUSPEND` | Prevent system suspend |

### Security

| Capability | Description |
|---|---|
| `CAP_AUDIT_CONTROL` | Enable/disable kernel auditing, change audit rules |
| `CAP_AUDIT_READ` | Read audit log |
| `CAP_AUDIT_WRITE` | Write audit log entries |
| `CAP_IPC_LOCK` | Lock memory (mlock, mlockall, mmap MAP_LOCKED) |
| `CAP_IPC_OWNER` | Bypass IPC ownership checks |
| `CAP_LEASE` | Set file leases on arbitrary files |
| `CAP_LINUX_IMMUTABLE` | Set immutable/append-only file attributes |
| `CAP_MAC_ADMIN` | Change MAC configuration (Smack, AppArmor) |
| `CAP_MAC_OVERRIDE` | Override MAC policies |
| `CAP_SYS_TTY_CONFIG` | Use vhangup, configure virtual terminals |

### Containers (newer)

| Capability | Description |
|---|---|
| `CAP_BPF` | BPF operations (since kernel 5.8) |
| `CAP_CHECKPOINT_RESTORE` | Checkpoint/restore operations (since kernel 5.9) |
| `CAP_PERFMON` | Performance monitoring (since kernel 5.8) |

## Managing File Capabilities

Capabilities are stored as extended attributes (xattr) on the file itself in the `security.capability` namespace.

### View File Capabilities

```bash
# Show capabilities on a file
getcap /usr/bin/ping
# /usr/bin/ping cap_net_raw=ep

# Recursively find all files with capabilities
getcap -r / 2>/dev/null

# Find all files with capabilities in common paths
getcap -r /usr/bin /usr/sbin /usr/lib 2>/dev/null
```

### Set File Capabilities

```bash
# Allow binary to bind to privileged ports
setcap 'cap_net_bind_service=+ep' /usr/local/bin/mywebserver

# Allow ping without setuid root
setcap 'cap_net_raw=+ep' /usr/bin/ping

# Multiple capabilities
setcap 'cap_net_bind_service,cap_net_raw=+ep' /path/to/binary

# Allow raw socket + network admin
setcap 'cap_net_raw,cap_net_admin=+ep' /usr/sbin/tcpdump
```

### Remove File Capabilities

```bash
# Remove all capabilities from a file
setcap -r /path/to/binary

# Verify removal
getcap /path/to/binary
```

### Capability Flags (ep, ei, ip, etc.)

The letters after `=` indicate which sets to modify:

| Flag | Set | Description |
|------|-----|-------------|
| `e` | Effective | Capability is active |
| `p` | Permitted | Capability is allowed |
| `i` | Inheritable | Capability can be passed to child processes |

Common combinations:

| Setting | Meaning |
|---------|---------|
| `=ep` | Effective and Permitted (most common for binaries) |
| `=eip` | Effective, Inheritable, and Permitted |
| `=p` | Permitted only (process must explicitly enable it) |
| `=ei` | Effective and Inheritable |

```bash
# +ep: add to effective and permitted sets
setcap 'cap_net_bind_service=+ep' /path/to/binary

# = (without +): set exactly these, clear all others
setcap 'cap_net_bind_service=ep' /path/to/binary
```

## Managing Process Capabilities

### View Process Capabilities

```bash
# Current process capabilities
cat /proc/self/status | grep -i cap

# Specific process
cat /proc/<PID>/status | grep -i cap

# Decoded (human-readable) — requires libcap-ng-utils
capsh --decode=<hex_value>

# Show current shell capabilities
capsh --print

# Using getpcaps (libcap)
getpcaps <PID>
getpcaps $$
```

### Decode Capability Hex Values

```bash
# From /proc/PID/status:
# CapInh: 0000000000000000
# CapPrm: 000001ffffffffff
# CapEff: 000001ffffffffff
# CapBnd: 000001ffffffffff
# CapAmb: 0000000000000000

# Decode with capsh
capsh --decode=000001ffffffffff
```

### Run a Process with Specific Capabilities

```bash
# Drop all capabilities except specified ones
capsh --caps="cap_net_bind_service+eip" -- -c "/usr/local/bin/myserver"

# Run command with reduced capabilities
capsh --drop=cap_sys_admin -- -c "some_command"

# Run as non-root with specific capabilities (requires ambient caps)
capsh --user=nobody --addamb=cap_net_bind_service -- -c "/usr/local/bin/myserver"
```

## Practical Examples

### Web Server on Port 80 Without Root

```bash
# Instead of running as root:
setcap 'cap_net_bind_service=+ep' /usr/local/bin/nginx

# Now nginx can be run as non-root user and still bind to port 80
sudo -u www-data /usr/local/bin/nginx
```

### Ping Without Setuid

```bash
# Traditional (setuid root):
ls -la /usr/bin/ping
# -rwsr-xr-x 1 root root ... /usr/bin/ping

# Modern (capabilities):
chmod u-s /usr/bin/ping
setcap 'cap_net_raw=+ep' /usr/bin/ping
ls -la /usr/bin/ping
# -rwxr-xr-x 1 root root ... /usr/bin/ping
getcap /usr/bin/ping
# /usr/bin/ping cap_net_raw=ep
```

### tcpdump Without Root

```bash
setcap 'cap_net_raw,cap_net_admin=+ep' /usr/sbin/tcpdump

# Run as regular user
tcpdump -i eth0
```

### Allow Non-Root to Read dmesg

```bash
setcap 'cap_syslog=+ep' /usr/bin/dmesg
```

### Container Runtime Capabilities

Docker drops most capabilities by default. Only these are kept:

```
AUDIT_WRITE, CHOWN, DAC_OVERRIDE, FOWNER, FSETID, KILL,
MKNOD, NET_BIND_SERVICE, NET_RAW, SETFCAP, SETGID, SETPCAP,
SETUID, SYS_CHROOT
```

```bash
# Run container with additional capability
docker run --cap-add=NET_ADMIN alpine

# Run container with dropped capability
docker run --cap-drop=NET_RAW alpine

# Run with no capabilities
docker run --cap-drop=ALL alpine

# Run with specific capabilities only
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE alpine

# Run fully privileged (all capabilities)
docker run --privileged alpine
```

## Capabilities and systemd

systemd services can specify capabilities directly:

```ini
[Service]
# Drop all capabilities, add only what's needed
CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_NET_RAW
AmbientCapabilities=CAP_NET_BIND_SERVICE

# Run as non-root
User=www-data
Group=www-data

# Additional hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
```

```bash
# View effective capabilities of a running service
systemctl show <service> | grep -i cap

# Check what capabilities a service has
getpcaps $(pgrep -f <service_name>)
```

## Security Considerations

### CAP_SYS_ADMIN — The "New Root"

`CAP_SYS_ADMIN` is a catch-all capability that covers:

- `mount()` / `umount()`
- `swapon()` / `swapoff()`
- `sethostname()` / `setdomainname()`
- Various ioctl operations
- Namespace operations
- Many more

Granting `CAP_SYS_ADMIN` is almost equivalent to granting root. Avoid it when possible.

### Bounding Set Lockdown

```bash
# View bounding set
cat /proc/self/status | grep CapBnd

# Drop capability from bounding set (irreversible until reboot)
capsh --drop=cap_sys_admin -- -c "exec bash"

# In systemd service:
# CapabilityBoundingSet=~CAP_SYS_ADMIN  (exclude SYS_ADMIN)
```

### NoNewPrivileges

Prevents a process from gaining new privileges through execve (SUID binaries, file capabilities):

```bash
# Set on a process
prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0)

# In systemd:
# NoNewPrivileges=true

# Check
cat /proc/<PID>/status | grep NoNewPrivs
```

## Troubleshooting

### Permission Denied Despite Capabilities Being Set

```bash
# Check if the filesystem supports capabilities (requires xattr)
mount | grep -i "nosuid\|noexec"
# Capabilities don't work on nosuid/noexec mounts

# Check if the binary was rewritten (capabilities are cleared on write)
getcap /path/to/binary

# Check if SELinux is blocking
ausearch -m AVC -ts recent

# Check bounding set
cat /proc/self/status | grep CapBnd
capsh --decode=$(cat /proc/self/status | grep CapBnd | awk '{print $2}')
```

### Capabilities Cleared After Package Update

File capabilities are stored in extended attributes. Package updates replace the binary, clearing capabilities:

```bash
# Re-apply after update
setcap 'cap_net_bind_service=+ep' /usr/local/bin/myserver

# Or use a systemd service with AmbientCapabilities (preferred)
```

### Common Error Messages

| Error | Likely Missing Capability |
|-------|--------------------------|
| `bind: Permission denied` (port < 1024) | `CAP_NET_BIND_SERVICE` |
| `Operation not permitted` (raw socket) | `CAP_NET_RAW` |
| `Operation not permitted` (chown) | `CAP_CHOWN` |
| `Permission denied` (kill signal) | `CAP_KILL` |
| `Operation not permitted` (module load) | `CAP_SYS_MODULE` |
| `Operation not permitted` (mount) | `CAP_SYS_ADMIN` |
| `Operation not permitted` (nice) | `CAP_SYS_NICE` |
| `Operation not permitted` (mlock) | `CAP_IPC_LOCK` |

## Required Packages

```bash
# RHEL / CentOS
yum install libcap libcap-ng-utils    # RHEL 6/7
dnf install libcap libcap-ng-utils    # RHEL 8+

# Ubuntu / Debian
apt install libcap2-bin libcap-ng-utils

# Provides: getcap, setcap, capsh, getpcaps, filecap, pscap, netcap
```

## Useful Commands

```bash
# List all files with capabilities on the system
getcap -r / 2>/dev/null

# List capabilities of all running processes
for pid in /proc/[0-9]*; do
    getpcaps ${pid##*/} 2>/dev/null
done

# Find processes with specific capability
pscap | grep net_bind

# Find network-related capabilities
netcap

# Show file capabilities in a directory
filecap /usr/bin

# Audit capability usage (requires auditd)
auditctl -a always,exit -F arch=b64 -S capset -k capability_changes
ausearch -k capability_changes
```

### When Capabilities Are Automatically Dropped

Capabilities are stored in xattr and are cleared in certain situations:

```bash
# File is modified — capabilities are removed
echo "data" >> /usr/bin/ping
getcap /usr/bin/ping
# No output — capabilities gone

# File is copied — capabilities are NOT copied
cp /usr/bin/ping /usr/bin/ping-copy
getcap /usr/bin/ping-copy
# No output — no capabilities

# File is moved — capabilities ARE preserved
mv /usr/bin/ping /usr/bin/ping2
getcap /usr/bin/ping2
# Still has capabilities
```

Capabilities don't survive:
- File modification (write to the binary)
- Copying the file (even as root)
- Package updates that replace the binary

Capabilities do survive:
- Moving/renaming the file
- Changing ownership (chown)

## Checking Kernel Capability Support

```bash
# Check for file capabilities support
grep CONFIG_SECURITY_FILE_CAPABILITIES /boot/config-$(uname -r)

# Check for SquashFS xattr support (needed for snap)
grep CONFIG_SQUASHFS_XATTR /boot/config-$(uname -r)

# Alternative (if /proc/config.gz available)
zcat /proc/config.gz | grep CONFIG_SECURITY_FILE_CAPABILITIES
zcat /proc/config.gz | grep CONFIG_SQUASHFS_XATTR

# Test xattr support on a file
touch /tmp/test
setfattr -n user.test -v "hello" /tmp/test
getfattr -n user.test /tmp/test
# If this works, xattr support exists
```

Expected output for full capability support:

```
CONFIG_SECURITY_FILE_CAPABILITIES=y
CONFIG_SQUASHFS_XATTR=y
```

## Capability Number Reference

Full list of capabilities with their numeric values (kernel 5.15+):

```
CAP_CHOWN                = 0
CAP_DAC_OVERRIDE         = 1
CAP_DAC_READ_SEARCH      = 2
CAP_FOWNER               = 3
CAP_FSETID               = 4
CAP_KILL                 = 5
CAP_SETGID               = 6
CAP_SETUID               = 7
CAP_SETPCAP              = 8
CAP_LINUX_IMMUTABLE      = 9
CAP_NET_BIND_SERVICE     = 10
CAP_NET_BROADCAST        = 11
CAP_NET_ADMIN            = 12
CAP_NET_RAW              = 13
CAP_IPC_LOCK             = 14
CAP_IPC_OWNER            = 15
CAP_SYS_MODULE           = 16
CAP_SYS_RAWIO            = 17
CAP_SYS_CHROOT           = 18
CAP_SYS_PTRACE           = 19
CAP_SYS_PACCT            = 20
CAP_SYS_ADMIN            = 21
CAP_SYS_BOOT             = 22
CAP_SYS_NICE             = 23
CAP_SYS_RESOURCE         = 24
CAP_SYS_TIME             = 25
CAP_SYS_TTY_CONFIG       = 26
CAP_MKNOD                = 27
CAP_LEASE                = 28
CAP_AUDIT_WRITE          = 29
CAP_AUDIT_CONTROL        = 30
CAP_SETFCAP              = 31
CAP_MAC_OVERRIDE         = 32
CAP_MAC_ADMIN            = 33
CAP_SYSLOG               = 34
CAP_WAKE_ALARM           = 35
CAP_BLOCK_SUSPEND        = 36
CAP_AUDIT_READ           = 37
CAP_PERFMON              = 38
CAP_BPF                  = 39
CAP_CHECKPOINT_RESTORE   = 40
```

```bash
# View all capabilities your kernel supports
cat /usr/include/linux/capability.h | grep "define CAP_"
```

## Capabilities vs setuid: Attack Surface

### setuid Binary (Old Way)

```bash
ls -l /usr/bin/program
-rwsr-xr-x 1 root root 500K /usr/bin/program
# The 's' means setuid — runs as root with ALL privileges
```

- Binary runs with UID 0
- Has every possible privilege (~40 capabilities)
- If exploited: full root access to entire system

### Capabilities Binary (New Way)

```bash
ls -l /usr/bin/program
-rwxr-xr-x 1 root root 500K /usr/bin/program

getcap /usr/bin/program
/usr/bin/program cap_net_raw=ep
# Only has raw socket access
```

- Binary runs with regular UID
- Has only necessary privileges (1–5 typically)
- If exploited: limited access (only the granted capabilities)

### Example: ping

**setuid (old):**
```bash
ls -l /bin/ping
-rwsr-xr-x 1 root root 64K /bin/ping
# If compromised: attacker gets full root
```

**capabilities (modern):**
```bash
ls -l /bin/ping
-rwxr-xr-x 1 root root 64K /bin/ping
getcap /bin/ping
/bin/ping cap_net_raw=ep
# If compromised: attacker can only create raw sockets
```

## Snapd Capabilities Transition (2.70)

Snapd 2.70 transitioned `snap-confine` from setuid to capabilities:

**Before (setuid):**
```bash
ls -l /usr/lib/snapd/snap-confine
-rwsr-xr-x 1 root root 424K snap-confine
# Runs with ALL root privileges
```

**After (capabilities):**
```bash
ls -l /usr/lib/snapd/snap-confine
-rwxr-xr-x 1 root root 424K snap-confine

getcap /usr/lib/snapd/snap-confine
snap-confine cap_sys_admin,cap_setuid,cap_setgid=ep
# Only 3 capabilities instead of all ~40
```

What snap-confine needs:
- `CAP_SYS_ADMIN` — create mount namespaces and bind mounts
- `CAP_SETUID` — switch to the user running the snap
- `CAP_SETGID` — switch group permissions

This breaks on systems where the kernel lacks `CONFIG_SQUASHFS_XATTR=y` because capabilities stored in SquashFS extended attributes cannot be read.

## Testing Capabilities

```bash
# Test if the capability system works
cat > /tmp/captest.c << 'EOF'
#include <sys/capability.h>
#include <stdio.h>

int main() {
    cap_t caps = cap_get_proc();
    printf("Capabilities: %s\n", cap_to_text(caps, NULL));
    cap_free(caps);
    return 0;
}
EOF

gcc /tmp/captest.c -lcap -o /tmp/captest
/tmp/captest
rm /tmp/captest /tmp/captest.c
```

```bash
# Check audit logs for capability denials
ausearch -m AVC -ts recent

# Enable capability auditing
auditctl -a always,exit -F arch=b64 -S capset

# Check kernel messages
dmesg | grep -i capab
```

## Quick Reference

```bash
# View file caps:       getcap /path/to/binary
# Set file caps:        setcap 'cap_name=+ep' /path/to/binary
# Remove file caps:     setcap -r /path/to/binary
# Find all file caps:   getcap -r / 2>/dev/null
# View process caps:    getpcaps <PID>
# Decode hex caps:      capsh --decode=<hex>
# Print current caps:   capsh --print
# Run with caps:        capsh --caps="cap_name+eip" -- -c "command"
```
