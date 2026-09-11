# fuser Cheatsheet

`fuser` identifies processes using files, directories, mounted filesystems, devices, and TCP or UDP ports. On Linux it is normally provided by the `psmisc` package. The behavior described here follows the Linux [`fuser(1)` manual](https://manpages.ubuntu.com/manpages/noble/en/man1/fuser.1.html); other Unix systems may use different options.

> **Warning:** `fuser -k` sends `SIGKILL` by default. Combined with `-m`, it can abruptly terminate every process accessing any file on a filesystem. Inspect first, prefer normal service shutdown or `SIGTERM`, and reserve `SIGKILL` for processes that do not stop.

## Installation

```bash
# Debian and Ubuntu
sudo apt install psmisc

# RHEL, Rocky Linux, AlmaLinux, and Fedora
sudo dnf install psmisc

# Arch Linux
sudo pacman -S psmisc
```

## Syntax

```text
fuser [options] NAME...
fuser -n NAMESPACE [options] NAME...
fuser NAME/NAMESPACE
```

The default namespace is `file`. The other supported namespaces are `tcp` and `udp`.

## Common Options

| Option | Description |
|--------|-------------|
| `-v`, `--verbose` | Show PID, user, access mode, and command in a `ps`-like table |
| `-u`, `--user` | Append each process owner's username to the compact output |
| `-m`, `--mount` | Find processes accessing any file on the mounted filesystem containing `NAME` |
| `-M`, `--ismountpoint` | Continue only when `NAME` is a mount point; an important safeguard with `-m` and `-k` |
| `-n NAMESPACE`, `--namespace NAMESPACE` | Search the `file`, `tcp`, or `udp` namespace |
| `-4`, `--ipv4` | Search IPv4 sockets only; valid with `tcp` and `udp` |
| `-6`, `--ipv6` | Search IPv6 sockets only; valid with `tcp` and `udp` |
| `-a`, `--all` | Display every named target, including unused targets |
| `-s`, `--silent` | Print nothing; useful when only the exit status is needed |
| `-I`, `--inode` | Compare file inodes instead of names, including on network filesystems |
| `-k`, `--kill` | Signal matching processes; defaults to `SIGKILL` |
| `-SIGNAL` | Select the signal used with `-k`, such as `-TERM` or `-HUP` |
| `-i`, `--interactive` | Ask before signaling each process; requires `-k` |
| `-w` | With `-k`, signal only processes that have write access |
| `-l`, `--list-signals` | List recognized signal names |
| `-V`, `--version` | Show the installed version |

`-c` is a POSIX-compatible alias for `-m`. Linux also accepts `-f` for POSIX compatibility but ignores it.

## Files, Directories, and Filesystems

### Find processes using a file

```bash
fuser /var/log/syslog
sudo fuser -v /var/log/syslog
```

The compact form prints PIDs. Use `-v` to see the process owner, command, and access type. Root privileges may be required to see processes owned by other users.

### Find processes using a directory itself

```bash
sudo fuser -v /srv/app
```

Without `-m`, naming a directory does **not** recursively search every file below it. It finds direct uses of that directory, such as a process whose current directory is `/srv/app`.

### Find processes using an entire mounted filesystem

```bash
# Identify all processes using the filesystem that contains /var
sudo fuser -vm /var
```

`-m` broadens the query to every file on the same mounted filesystem. If `/var` is not a separate mount, this may search the root filesystem and return many system processes.

Confirm filesystem boundaries before acting:

```bash
findmnt -T /var
mountpoint /var
sudo fuser -vm /var
```

### Require the target to be a mount point

```bash
sudo fuser -vmM /mnt/data
```

`-M` prevents the request from proceeding unless `/mnt/data` is an actual mount point. Use it as a safety check when a typo or an unexpectedly unmounted path could make `-m` target a broader filesystem.

### Inspect a block device

```bash
# Processes using the device node itself
sudo fuser -v /dev/sdb1

# Processes accessing files on the filesystem mounted from this device
sudo fuser -vm /dev/sdb1
```

Verify the device and mount relationship with `findmnt` or `lsblk` before using any kill option.

### Check multiple targets

```bash
sudo fuser -v /var/log/syslog /var/log/auth.log /dev/ttyUSB0
```

Use `-a` if every target must appear even when no process is using it:

```bash
sudo fuser -av /var/log/syslog /var/log/auth.log
```

## TCP and UDP Ports

Socket lookups require the `tcp` or `udp` namespace. By default, `fuser` searches both IPv4 and IPv6.

### Find processes using TCP port 80

```bash
# List all processes using local TCP port 80
sudo fuser -vn tcp 80

# Equivalent shortcut notation
sudo fuser -v 80/tcp
```

This searches the **local** port, so results can include listeners and connected sockets using that local port.

### Find processes using a UDP port

```bash
sudo fuser -vn udp 53
sudo fuser -v 53/udp
```

### Restrict the address family

```bash
# IPv4 only
sudo fuser -4 -vn tcp 443

# IPv6 only
sudo fuser -6 -vn tcp 443
```

Do not combine `-4` and `-6`.

### Use a service name

```bash
fuser -v ssh/tcp
fuser -v domain/udp
```

Service names are resolved through the system service database, commonly `/etc/services`. Numeric ports are less ambiguous in scripts.

### Inspect the returned process

```bash
sudo fuser -vn tcp 8080
ps -fp <PID>
sudo readlink -f /proc/<PID>/exe
```

Replace `<PID>` with a PID returned by `fuser`.

## Understanding Output

Verbose output resembles:

```text
                     USER        PID ACCESS COMMAND
/var:                root       1234 ..c.. bash
                     app        5678 f.... worker
```

The `ACCESS` field describes how each process uses the target:

| Letter | Meaning |
|--------|---------|
| `c` | Current working directory |
| `e` | Executable being run |
| `f` | Open file |
| `F` | File open for writing |
| `r` | Process root directory |
| `m` | Memory-mapped file or shared library |
| `.` | Placeholder for an unused access position |

In compact output, each PID may be followed by access letters. Lowercase `f` and uppercase `F` are omitted from the default compact display, so use `-v` when the access type matters.

`fuser` sends PIDs to standard output and labels or other explanatory text to standard error. Keep that split in mind when redirecting output:

```bash
# Capture only matching PIDs
pids=$(fuser /var/log/syslog 2>/dev/null)

# Show a normal verbose report, including diagnostic messages
sudo fuser -v /var/log/syslog
```

## Signals and Safe Process Termination

### Inspect before signaling

```bash
sudo fuser -vmM /mnt/data
```

Where possible, identify the owning service and stop it cleanly:

```bash
sudo systemctl stop example.service
sudo fuser -vmM /mnt/data
```

### Request graceful termination

```bash
sudo fuser -k -TERM -m -M /mnt/data
sleep 5
sudo fuser -vmM /mnt/data
```

`SIGTERM` allows applications to run shutdown handlers, flush data, and release locks. Recheck the filesystem after waiting.

### Ask before signaling each process

```bash
sudo fuser -k -i -TERM -m -M /mnt/data
```

Interactive confirmation reduces accidental termination but does not make a broad target inherently safe.

### Signal only writers

```bash
sudo fuser -k -w -TERM /var/log/example.log
```

`-w` has an effect only with `-k`.

### Force termination only as a last resort

```bash
sudo fuser -k -KILL -m -M /mnt/data
```

`-KILL` cannot be caught and does not permit cleanup. Confirm the exact target, review every matching PID, and expect possible lost work or application recovery on restart.

### Dangerous broad-filesystem example

```bash
# DANGEROUS: defaults to SIGKILL for every matching process
sudo fuser -km /home
```

This requested command can kill shells, editors, desktop sessions, SSH sessions, and services accessing any file on `/home`. If `/home` is not separately mounted, `-m` may target the filesystem containing it. Never run the equivalent against `/`, `/var`, `/usr`, or another broad system filesystem without understanding the full impact.

Prefer this sequence instead:

```bash
findmnt -T /home
sudo fuser -vmM /home
sudo fuser -k -i -TERM -m -M /home
sleep 5
sudo fuser -vmM /home
```

## Practical Workflows

### Diagnose a busy mount before unmounting

```bash
findmnt /mnt/data
sudo fuser -vmM /mnt/data
```

Common blockers include a shell whose current directory is on the mount, a service with open files, and a process with a memory-mapped file. Change directory or stop the responsible service, then recheck:

```bash
cd /
sudo systemctl stop example.service
sudo fuser -vmM /mnt/data
sudo umount /mnt/data
```

Do not use `fuser -k` as the first step in a routine unmount.

### Diagnose an APT or dpkg lock

```bash
sudo fuser -v /var/lib/dpkg/lock-frontend
sudo fuser -v /var/lib/dpkg/lock
sudo fuser -v /var/cache/apt/archives/lock
```

Inspect the reported process and allow active package operations or automatic updates to finish. Killing a package manager can leave package configuration incomplete.

### Find a process blocking a serial device

```bash
sudo fuser -v /dev/ttyUSB0
```

### Reload processes using a configuration file

```bash
# Inspect first
sudo fuser -v /etc/example/example.conf

# Send SIGHUP only if the application documents reload support
sudo fuser -k -HUP /etc/example/example.conf
```

A process may read a configuration file and close it immediately, in which case `fuser` will not find it later. Prefer the application's documented reload command.

## Scripting and Exit Status

`fuser` exits with status `0` when it finds at least one access. It returns nonzero when no matching access is found **or** when a fatal error occurs.

```bash
if fuser -s /dev/ttyUSB0; then
    echo "Device is in use"
else
    echo "No access found, or fuser encountered an error"
fi
```

Because an unused target and a fatal error both produce nonzero status, validate the target and permissions when scripts need to distinguish failure modes.

Avoid using `-s` with `-a`; silent mode also ignores `-u` and `-v`.

## Troubleshooting

### No process is reported for a known open file or port

Try with appropriate privileges:

```bash
sudo fuser -v /path/to/file
sudo fuser -vn tcp 8080
```

Linux reads process information from `/proc`. Permission restrictions can hide file descriptors and socket ownership from unprivileged users. Do not make `fuser` setuid root merely to bypass this limitation.

### A directory query misses files below it

A plain directory query is not recursive. Use `-m` only when you intend to inspect the entire mounted filesystem:

```bash
sudo fuser -vm /srv/app
```

For a recursive directory tree rather than the whole filesystem, use `lsof +D` carefully; it can be slow on large trees.

### A process in another mount namespace is missing

Containers and services can have separate mount namespaces. `fuser` may not match mounted block devices seen through another namespace. Inspect from the relevant container or namespace, or enter it with an appropriate administration tool before repeating the query.

### Kernel access appears instead of a PID

Verbose mode can report `kernel` for a mount point, NFS export, or swap file. `fuser -k` acts on processes and cannot terminate kernel access. Remove the underlying kernel use safely, such as disabling swap or unexporting a filesystem, rather than trying to kill it.

## Alternatives

| Goal | Alternative |
|------|-------------|
| Detailed open-file information | `sudo lsof /path/to/file` |
| Files open on a mounted filesystem | `sudo lsof +f -- /mnt/data` |
| Recursive directory scan | `sudo lsof +D /srv/app` |
| TCP listener and process on port 80 | `sudo ss -ltnp 'sport = :80'` |
| UDP socket and process on port 53 | `sudo ss -lunp 'sport = :53'` |
| Process details after finding a PID | `ps -fp <PID>` |

Use `fuser` for a fast path-to-PID or local-port-to-PID lookup, `lsof` for richer file-descriptor details, and `ss` for socket state and endpoint information.

## Quick Reference

```bash
# Specific file
sudo fuser -v /path/to/file

# Directory itself, not its full tree
sudo fuser -v /path/to/directory

# Entire filesystem containing /var
sudo fuser -vm /var

# Mount-point-checked filesystem query
sudo fuser -vmM /mnt/data

# Local TCP port 80
sudo fuser -vn tcp 80
sudo fuser -v 80/tcp

# Local UDP port 53
sudo fuser -vn udp 53

# Graceful termination on a verified mount point
sudo fuser -k -TERM -m -M /mnt/data

# Interactive graceful termination
sudo fuser -k -i -TERM -m -M /mnt/data

# Default SIGKILL behavior — inspect before using
sudo fuser -km /home

# List signal names
fuser -l
```

Content based on the linked Linux manual was rephrased for compliance with licensing restrictions.
