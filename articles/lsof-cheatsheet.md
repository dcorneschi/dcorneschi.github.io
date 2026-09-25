# lsof Cheatsheet

`lsof` (list open files) reports which files are open and which processes hold them. On Unix-like systems "everything is a file" — so this includes regular files, directories, devices, pipes, and network sockets. That makes `lsof` a Swiss-army knife for answering "what's using this port / file / mount?" and "what does this process have open?".

Most complete output requires root; as a normal user you only see your own processes. Pair with the [ss Cheatsheet](articles/ss-cheatsheet.md) and [netstat Cheatsheet](articles/netstat-cheatsheet.md) for socket-focused views, and the [Linux Processes and Signals Cheatsheet](articles/ps-cheatsheet.md) for process inspection.

## Syntax

```text
lsof [options] [names]
```

Options with a `-` prefix generally **filter**; options with a `+` prefix are **inclusive** (e.g. `+D`, `+L1`). Multiple selection criteria are **OR**'d by default — use `-a` to **AND** them.

## Basic Usage

```bash
lsof                       # every open file on the system (large output)
lsof -p 1234               # files opened by PID 1234
lsof -u alice              # files opened by user alice
lsof -c nginx              # files opened by commands named nginx
lsof /path/to/file         # processes with this file open
lsof /path/to/dir          # processes using this directory
```

## Selecting by Process and User

```bash
lsof -p 1234,5678          # multiple PIDs
lsof -p ^1234              # all processes EXCEPT PID 1234
lsof -u alice,bob          # multiple users
lsof -u alice -u ^root     # user alice, excluding root
lsof -R                    # also show parent PID (PPID) column
```

## Network Connections

```bash
lsof -i                    # all network connections (IPv4 + IPv6)
lsof -i :80                # anything on port 80
lsof -i :443
lsof -i tcp                # TCP only
lsof -i udp                # UDP only
lsof -i TCP:3000           # TCP on a specific port
lsof -i @192.168.1.1       # connections to/from a host
lsof -i 4                  # IPv4 only
lsof -i 6                  # IPv6 only
lsof -i -sTCP:LISTEN       # listening sockets
lsof -i -sTCP:ESTABLISHED  # established connections
```

Add `-n` (no hostname resolution) and `-P` (no port-name resolution) for faster, numeric output:

```bash
lsof -i -n -P              # all connections, numeric, fast
lsof -i -n -P | grep LISTEN
```

## Files and Directories

```bash
lsof /path/to/file         # who has this file open
lsof +d /path/to/dir       # files in a directory (top level only, faster)
lsof +D /path/to/dir       # files under a directory (recursive, slower)
lsof +L1                   # deleted files still held open (link count < 1)
lsof /dev/sda1             # processes using a device
```

> `+d` searches only the directory's immediate contents; `+D` recurses the whole tree. Prefer `+d` when you don't need recursion — `+D` can be very slow on large trees.

## Output Formatting

```bash
lsof -t                    # terse: PIDs only (ideal for scripting)
lsof -t -i :8080           # PIDs holding port 8080
lsof -r 5                  # repeat every 5 seconds (0 = continuous)
lsof -d 0-2                # only fd 0,1,2 (stdin/stdout/stderr)
lsof -F pcfn               # machine-parseable fields: PID, command, fd, name
lsof -n -P                 # skip DNS and service-name lookups
```

## Combining Criteria with -a

By default multiple criteria are OR'd. `-a` requires **all** of them to match:

```bash
lsof -a -u alice -c nginx     # files that are alice's AND owned by nginx
lsof -a -u alice -i           # alice's network connections only
lsof -a -p 1234 -d 0-2        # fds 0-2 of PID 1234 only
```

## Common Tasks

### Which process is using a port?

```bash
lsof -i :8080
lsof -i TCP:3000
lsof -t -i :8080              # just the PID
```

### Free a port / kill what holds a file

```bash
kill $(lsof -t -i :8080)          # graceful
lsof -ti :8080 | xargs kill -9    # forceful (also common on macOS)
kill $(lsof -t /path/to/file)     # kill everything holding a file
```

> `kill -9` gives the process no chance to clean up. Try a normal `kill` (TERM) first and reserve `-9` for genuinely stuck processes.

### What's preventing an unmount?

```bash
lsof +D /mnt/usb             # everything open under the mount point
lsof /mnt/backup
```

### Reclaim disk from deleted-but-open files

A file that's been `rm`'d but is still held open by a process keeps consuming disk until that process closes it or is restarted:

```bash
lsof +L1                     # list deleted files still open
lsof | grep -i deleted       # alternative view
```

### Inspect a single process

```bash
lsof -p 1234                 # everything PID 1234 has open
lsof -p 1234 | wc -l         # count its open fds (spot fd leaks)
lsof -r 1 -p 1234            # watch it live, refresh every second
```

### Find the processes with the most open files

```bash
lsof | awk '{print $2}' | sort | uniq -c | sort -nr | head
```

## Security Checks

```bash
lsof | grep -E '(deleted|DEL)'     # processes running deleted executables
lsof /etc/passwd /etc/shadow       # who has sensitive files open
lsof -i -n | grep -v LISTEN        # active (non-listening) connections
```

Running-but-deleted binaries can indicate a process that was updated (or replaced maliciously) without a restart — worth investigating either way.

## Output Columns

| Column | Meaning |
|--------|---------|
| `COMMAND` | Process (command) name |
| `PID` | Process ID |
| `USER` | User owning the process |
| `FD` | File descriptor (see below) |
| `TYPE` | File type: `REG`, `DIR`, `CHR`, `IPv4`, `IPv6`, `unix`, `FIFO`, … |
| `DEVICE` | Device numbers |
| `SIZE/OFF` | File size or file offset |
| `NODE` | Inode number (or protocol like TCP/UDP for sockets) |
| `NAME` | File path, mount point, or connection endpoints |

## File Descriptor (FD) Values

| Value | Meaning |
|-------|---------|
| `cwd` | Current working directory |
| `rtd` | Root directory |
| `txt` | Program text (executable/code) |
| `mem` | Memory-mapped file |
| `0` | stdin |
| `1` | stdout |
| `2` | stderr |
| `3`+ | Other open descriptors (often suffixed `r`/`w`/`u` for read/write/both) |

## Connection States (for -i)

| State | Meaning |
|-------|---------|
| `LISTEN` | Waiting for incoming connections |
| `ESTABLISHED` | Active, open connection |
| `TIME_WAIT` | Closed, waiting out lingering packets |
| `CLOSE_WAIT` | Remote closed; local side hasn't closed yet (leak if it piles up) |

## Common Flags

| Flag | Meaning |
|------|---------|
| `-a` | AND the selection criteria (default is OR) |
| `-b` | Avoid kernel calls that may block |
| `-c <name>` | Select by command name |
| `-d <fds>` | Select by file descriptor (e.g. `0-2`, `cwd`) |
| `-i [spec]` | Select network (internet) files |
| `-n` | Don't resolve hostnames |
| `-P` | Don't resolve port names |
| `-p <pid>` | Select by PID (`^` to exclude) |
| `-r <secs>` | Repeat mode (0 = continuous) |
| `-R` | Show parent PID (PPID) |
| `-s [p:s]` | Select by protocol state, e.g. `-sTCP:LISTEN` |
| `-t` | Terse output — PIDs only |
| `-u <user>` | Select by user (`^` to exclude) |
| `-v` | Version / verbose info |
| `+d <dir>` | Directory contents, top level only |
| `+D <dir>` | Directory tree, recursive |
| `+L1` | Files with link count < 1 (deleted but open) |

## Performance Tips

- Use `-n` and `-P` together to skip DNS and service-name lookups — a big speedup for `-i` queries.
- Prefer `+d` over `+D` when you don't need recursion.
- Use `-t` in scripts so output is just PIDs, ready to pipe into `kill` or `xargs`.
- Full visibility usually needs root; without it you only see your own processes.

## Quick Reference

```bash
lsof -i :PORT                    # who's on a port
lsof -t -i :PORT | xargs kill    # free that port
lsof -p PID                      # what a process has open
lsof -u USER                     # what a user has open
lsof +D /mnt/x                   # what blocks an unmount
lsof +L1                         # deleted-but-open files (disk leaks)
lsof -i -n -P | grep LISTEN      # listening sockets, numeric
lsof -a -u USER -i               # AND criteria: a user's connections
```

For related material, see the [ss Cheatsheet](articles/ss-cheatsheet.md), the [netstat Cheatsheet](articles/netstat-cheatsheet.md), and the [Linux Processes and Signals Cheatsheet](articles/ps-cheatsheet.md).
