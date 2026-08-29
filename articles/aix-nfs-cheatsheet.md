# AIX NFS Cheatsheet

Command reference for NFS (Network File System) on IBM AIX — exporting filesystems from a server, mounting them on clients, managing the NFS daemons, and diagnosing problems. AIX provides both the standard NFS tools (`exportfs`, `mount`, `showmount`, `nfsstat`) and its own ODM-aware wrappers (`mknfs`, `mknfsexp`, `mknfsmnt`) that make exports and mounts persistent across reboots.

> Most administrative commands require `root`. The ODM-aware `mknfs*` commands update the config files **and** the ODM so changes survive reboots — prefer them over hand-editing where possible. For the menu equivalents, see `smitty nfs` in the [AIX SMIT Cheatsheet](articles/aix-smit-cheatsheet.md).

## Key Files

| File | Purpose |
|------|---------|
| `/etc/exports` | Filesystems this host exports (server side) |
| `/etc/xtab` | Currently exported filesystems (maintained by `exportfs`) |
| `/etc/filesystems` | Persistent mount definitions (NFS mounts included) |
| `/etc/rc.nfs` | Startup script that starts the NFS daemons at boot |
| `/etc/nfs/hostkey` | NFS server host principal (NFSv4/Kerberos) |
| `/etc/nfs/local_domain` | NFSv4 local domain |
| `/etc/nfs/realm.map` | NFSv4 realm-to-domain mapping |

## NFS Daemons and Services

```sh
# Start / stop / restart all NFS subsystems as a group
startsrc -g nfs
stopsrc -g nfs

# Individual daemons
startsrc -s nfsd        # server: services client requests
startsrc -s rpc.mountd  # server: handles mount requests
startsrc -s biod        # client: block I/O daemon (legacy)
startsrc -s rpc.statd   # lock recovery
startsrc -s rpc.lockd   # file locking
startsrc -s rpc.idmapd  # NFSv4 user/group ID mapping

# Check status
lssrc -g nfs
lssrc -s nfsd

# Will NFS start at boot? (check the inittab entry)
lsitab rcnfs
```

> `rpc.mountd` and `nfsd` are required on the server; `portmap`/`rpcbind` must also be running. NFSv4 additionally uses `rpc.idmapd`.

### Configure NFS via SMIT

```sh
smit mknfs        # configure and start NFS services (now and at next boot)
smit rmnfs        # stop and unconfigure NFS services
smit mknfsexp     # add a directory to the exports list
smit chnfsexp     # change/show an exported directory
smit rmnfsexp     # remove a directory from the exports list
smit mknfsmnt     # add a filesystem for NFS mounting
```

## Server: Exporting Filesystems

### Persistent exports (mknfsexp — updates /etc/exports)

```sh
# Export /data read/write to everyone (also adds it to /etc/exports)
mknfsexp -d /data -t rw

# Export read-only to specific clients
mknfsexp -d /data -t ro -c host1,host2

# Read/write to some hosts, root access allowed for one
mknfsexp -d /data -t rw -a client1,client2 -r client1

# Export with a specific NFS version and (v4) reference name
mknfsexp -d /export -v 4 -t rw
```

| Flag | Meaning |
|------|---------|
| `-d` | Directory to export |
| `-t` | Access type: `rw`, `ro`, or `rm` (read-mostly) |
| `-c` | Clients allowed to mount |
| `-a` | Hosts granted read/write access |
| `-r` | Hosts granted root access |
| `-v` | NFS version (`2`, `3`, `4`) |

### Activate/refresh exports (exportfs)

`exportfs` reads `/etc/exports` and publishes entries in `/etc/xtab`.

```sh
exportfs -a              # export everything listed in /etc/exports
exportfs                 # list currently exported filesystems
exportfs -v              # verbose listing
exportfs -i /data        # export /data now, ignoring /etc/exports options
exportfs -u /data        # unexport /data
exportfs -ua             # unexport everything
```

### Remove a persistent export (rmnfsexp)

```sh
rmnfsexp -d /data        # unexport and remove from /etc/exports
```

## Client: Mounting NFS Filesystems

### One-off mount

```sh
# Mount an export now
mount server:/data /mnt/data

# With options (NFSv3 over TCP, soft mount, timeouts)
mount -o vers=3,proto=tcp,soft,timeo=30,retrans=3 server:/data /mnt/data

# NFSv4 mount
mount -o vers=4 server:/export /mnt/export

# Full real-world example (NFSv3, TCP, hard, tuned buffers)
mount -o rw,bg,hard,rsize=32768,wsize=32768,timeo=600,vers=3,proto=tcp,sec=sys \
  server:/usr/sap/trans /usr/sap/trans
```

### Persistent mount (mknfsmnt — updates /etc/filesystems)

```sh
# Add a persistent NFS mount that mounts at boot
mknfsmnt -f /mnt/data -d /data -h server -t rw -A -w bg

# Then mount everything defined in /etc/filesystems that is NFS
mount -a
```

| Flag | Meaning |
|------|---------|
| `-f` | Local mount point |
| `-d` | Remote directory (server path) |
| `-h` | NFS server hostname |
| `-t` | `rw` or `ro` |
| `-A` | Mount automatically at system restart |
| `-w` | `bg` (background) or `fg` (foreground) mount retry |

### Common mount options

| Option | Effect |
|--------|--------|
| `vers=3` / `vers=4` | NFS protocol version |
| `proto=tcp` / `proto=udp` | Transport |
| `hard` (default) | Retry indefinitely on server outage (safest for data) |
| `soft` | Give up after `retrans` retries (risk of I/O errors) |
| `bg` | Retry the mount in the background if the server is down |
| `intr` | Allow signals to interrupt a hung hard mount |
| `timeo=N` | RPC timeout in tenths of a second |
| `rsize=`/`wsize=` | Read/write buffer sizes |

### Unmount

```sh
umount /mnt/data
umount -f /mnt/data      # force (server gone / hung mount)
umount -a -t nfs         # unmount all NFS mounts
```

## Client Diagnostics

```sh
# What does a server export, and who has it mounted?
showmount -e server      # exports available on the server
showmount -a server      # all client:mount pairs
showmount -d server      # directories currently mounted by clients

# NFS/RPC statistics
nfsstat -s               # server-side stats (calls received/rejected)
nfsstat -c               # client-side stats (calls sent/rejected)
nfsstat -m               # per-mount options and performance (client)
nfsstat -z               # reset all statistics counters to zero

# Characteristics of NFS-mountable filesystems (from /etc/filesystems)
lsnfsmnt

# Are the server's RPC services registered?
rpcinfo -p server        # list RPC programs/ports on the server
rpcinfo -u server nfs    # ping the NFS service over UDP
rpcinfo -u host nfs 3    # ping NFS v3 over UDP specifically
```

### Trace NFS/network activity (netpmon)

```sh
# Capture all network stats, write the report on stop
netpmon -o /tmp/netp.log -O all
# ... reproduce the activity ...
trcstop
cat /tmp/netp.log
```

## NFSv4 Specifics

```sh
# Set/show the NFSv4 domain (must match between client and server for ID mapping)
chnfsdom example.com     # set the local NFSv4 domain
chnfsdom                 # show the current domain

# Manage the ID-mapping daemon
startsrc -s rpc.idmapd
stopsrc  -s rpc.idmapd

# NFSv4 uses a single pseudo-root; nfsroot sets the exported root
nfsroot -e /export       # show/set the NFSv4 root and exports
```

> NFSv4 maps users by `user@domain`. If files show up as `nobody`, the NFSv4 domains (`chnfsdom`) on client and server don't match, or `rpc.idmapd` isn't running.

## Tuning (nfso)

`nfso` displays and sets NFS network options. Changes can be scoped to the current boot, future boots, or both.

```sh
nfso -a                 # list all options and values
nfso -as                # list all options (short/current values)
nfso -o nfs_use_reserved_ports          # show one option
nfso -p -o nfs_use_reserved_ports=1     # set persistently (now + future boots)
```

**Scope flags** (which apply to `nfso`, and to many other AIX tunables like `vmo`/`ioo`):

| Flag | Scope |
|------|-------|
| `-p` | Now **and** future boots (persistent — the default effect of `-p`) |
| `-r` | Future boots only (not the running value) |
| (no flag) | Now only (not persisted across reboot) |

> Some AIX tunable commands document these as `-B` (both, default), `-I` (future boot only), and `-N` (now only) — the same three scopes under different letters depending on the tool.

**`nfs_use_reserved_ports`** controls which source ports the NFS client uses to talk to the server:

- `0` — the client uses non-reserved ports **above** 1024
- `1` — the client uses reserved ports **below** 1024 (some servers require this)

```sh
nfso -p -o nfs_use_reserved_ports=1
```

## Troubleshooting

| Symptom | Check |
|---------|-------|
| `mount` hangs | Server up? `showmount -e server`, `rpcinfo -p server`; try `bg`/`soft` to avoid hanging |
| Permission denied on mount | Client not in the export's `-c`/`-a` list; re-check `exportfs -v` |
| Files owned by `nobody` (v4) | `chnfsdom` mismatch or `rpc.idmapd` down |
| Stale file handle | Export was recreated; unmount/remount the client |
| Exports not surviving reboot | Use `mknfsexp` (updates `/etc/exports`), not just `exportfs -i` |
| Mounts not surviving reboot | Use `mknfsmnt -A` (updates `/etc/filesystems`) |

```sh
# Confirm daemons are up
lssrc -g nfs

# Re-export and verify
exportfs -va
showmount -e localhost
```

## Quick Reference

| Task | Command |
|------|---------|
| Start/stop all NFS | `startsrc -g nfs` / `stopsrc -g nfs` |
| Persistent export | `mknfsexp -d /data -t rw` |
| Re-export /etc/exports | `exportfs -a` |
| List local exports | `exportfs -v` |
| Unexport | `exportfs -u /data` |
| One-off mount | `mount server:/data /mnt/data` |
| Persistent mount | `mknfsmnt -f /mnt/data -d /data -h server -t rw -A` |
| List server exports | `showmount -e server` |
| Mount options/stats | `nfsstat -m` |
| RPC services on server | `rpcinfo -p server` |
| Set NFSv4 domain | `chnfsdom example.com` |

## Related

- [AIX SMIT Cheatsheet](articles/aix-smit-cheatsheet.md) — `smitty nfs`, `smitty mknfsexp`, `smitty mknfsmnt`
- [AIX Filesystems Cheatsheet](articles/aix-filesystems-cheatsheet.md) — local filesystems, mount, and `/etc/filesystems`
- [AIX NIM Cheatsheet](articles/aix-nim-cheatsheet.md) — NFS-exported mksysb/lpp_source for network installs
