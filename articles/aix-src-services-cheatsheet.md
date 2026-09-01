# AIX System Resource Controller (SRC) Cheatsheet

Command reference for managing services on IBM AIX with the **System Resource Controller (SRC)** — starting, stopping, and refreshing subsystems and groups (`startsrc`, `stopsrc`, `refresh`), listing subsystem status (`lssrc`), the `inetd`-managed services, the `srcmstr` master, and the `tcp.clean` helper for TCP/IP daemons.

> The SRC manages daemons as **subsystems** (single services) grouped into **subsystem groups** (e.g. `tcpip`, `nfs`) and reachable individually or by `inetd` subserver. Most control commands require `root`. Stopping a group such as `tcpip` takes down every daemon in it — confirm scope with `lssrc` before acting.

## Starting Subsystems

```sh
# Start a subsystem by name (-s)
startsrc -s xntpd

# Start a subserver by type (-t), e.g. an inetd-managed service
startsrc -t ftp
```

## Stopping Subsystems

```sh
# Stop a subserver by type
stopsrc -t ftp

# Stop an entire subsystem group (-g), e.g. all NFS services
stopsrc -g nfs

# Stop all running TCP/IP daemons
stopsrc -g tcpip
```

| Flag | Selects | Example |
|------|---------|---------|
| `-s` | A single subsystem by name | `startsrc -s xntpd` |
| `-t` | A subserver by type (often an `inetd` service) | `stopsrc -t ftp` |
| `-g` | A subsystem group | `stopsrc -g nfs` |

## Refreshing a Subsystem

`refresh` tells a subsystem to re-read its configuration without a full restart (for subsystems that support it, such as `named` and `syslogd`).

```sh
# Refresh the named (DNS) service
refresh -s named
```

## Listing Status (lssrc)

```sh
# List all registered subsystems and their status
lssrc -a

# Status of a single subsystem
lssrc -s syslogd

# List all services (subservers) represented by inetd
lssrc -ls inetd
```

| Option | Purpose |
|--------|---------|
| `-a` | List all subsystems on the system |
| `-s <name>` | Status of one subsystem |
| `-g <group>` | Status of a subsystem group |
| `-ls <name>` | Long/detailed status (e.g. `inetd` subservers) |

## The SRC Master (srcmstr)

`srcmstr` is the SRC master process. Every subsystem it starts is a child of it, so its PID is the parent (PPID) of SRC-managed daemons.

```sh
# Show the SRC master process
ps -ef | grep srcmstr
```

## Tracing a Subsystem

Subsystems that support SRC tracing can have it toggled without a restart:

```sh
# Turn tracing on / off for a subsystem
traceson -s <subsystem>
tracesoff -s <subsystem>

# Long form (works for groups too)
traceson -g <group>
tracesoff -g <group>
```

> `-l` requests long-format trace output where the subsystem supports it (e.g. `traceson -l -s <subsystem>`).

## TCP/IP Cleanup Helper

```sh
# Stop all running TCP/IP daemons (except portmap and nfsd)
# and remove all /etc/locks/lpd lock files
sh /etc/tcp.clean
```

## Quick Reference

| Task | Command |
|------|---------|
| Start a subsystem | `startsrc -s xntpd` |
| Start a subserver | `startsrc -t ftp` |
| Stop a subserver | `stopsrc -t ftp` |
| Stop a subsystem group | `stopsrc -g nfs` |
| Stop all TCP/IP daemons | `stopsrc -g tcpip` |
| Refresh a subsystem | `refresh -s named` |
| List all subsystems | `lssrc -a` |
| Status of one subsystem | `lssrc -s syslogd` |
| List inetd subservers | `lssrc -ls inetd` |
| Clean up TCP/IP daemons | `sh /etc/tcp.clean` |
| Find the SRC master | `ps -ef \| grep srcmstr` |

## Related

- [AIX Networking Cheatsheet](articles/aix-networking-cheatsheet.md) — the TCP/IP daemons and `inetd` subservers managed through the SRC.
