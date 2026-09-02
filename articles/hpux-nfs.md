# HP-UX NFS (Server and Client)

Configuring NFS on HP-UX — the supported protocol versions per release, sharing/exporting filesystems (the `exportfs` model on 11i v1/v2 vs the `share` model on 11i v3), the server and client daemons and config files, and monitoring. Covers HP-UX 11i v1–v3.

## Concepts: How NFS Works on HP-UX

NFS (Network File System) lets a server *export* (share) a directory tree so that clients can *mount* it and use it as though it were local. Under the hood it is built on **ONC RPC** (Remote Procedure Call) and, for v2/v3, **XDR** data encoding. That RPC foundation explains most of NFS's operational quirks: services register with the **portmapper** (`rpcbind`, port 111), clients ask the portmapper which dynamic port a given service landed on, and only then do they talk to that service. This is why `rpcinfo -p` is the first diagnostic you run — if `mountd`, `nfsd`, `statd`, or `lockd` are not registered, nothing will mount.

The main daemons and their jobs:

| Daemon | Role |
|--------|------|
| `rpcbind` / `portmap` | Maps RPC program numbers to ports; every other service registers here |
| `nfsd` | The server kernel threads that service file read/write requests |
| `rpc.mountd` | Handles the initial `mount` request and access checks (v2/v3) |
| `rpc.statd` | Status monitor — tracks peers so locks can be recovered after a crash |
| `rpc.lockd` | Network Lock Manager (NLM) — advisory file locking for v2/v3 |

A crucial architectural difference: in **NFSv2/v3**, locking (`lockd`) and status monitoring (`statd`) are *separate* side-band protocols on their own dynamic ports, which is what makes v2/v3 painful through firewalls. **NFSv4** folds locking, mounting, and ACLs into the single main protocol on one well-known port (2049), and adds a compound-operation model that reduces round trips. That consolidation is the practical reason to prefer v4 across untrusted networks.

## NFS Protocol Versions by Release

| HP-UX release | NFS version |
|---------------|-------------|
| 10.20 (unsupported) | NFSv2 (last supported there) |
| 11.00, 11i v1, 11i v2 | NFSv3 |
| 11i v3 | NFSv4 (plus v3/v2) |

- HP's **NFSv3** implementation is backward-compatible with NFSv2: an NFSv3 server accepts mounts from NFSv2 clients, and NFSv3 clients can mount from NFSv2 servers.
- **NFSv4** lets you assign **static port numbers** to `rpc.statd`, `rpc.lockd`, `rpc.mountd`, and `nfsd`, which greatly simplifies firewall configuration (v2/v3 use dynamic RPC ports via the portmapper).
- NFSv4 also changes identity handling: users and groups are exchanged as `user@domain` strings and mapped by the `nfsmapid` daemon, so a mismatched NFSv4 domain between client and server makes files appear owned by `nobody`. Keep the domain consistent (configured in `/etc/default/nfs`) across the estate.
- The client can request a specific protocol version at mount time with `-o vers=3` (or `vers=4`). If you do not specify, the client negotiates the highest version both ends support, which occasionally surprises you when a client silently falls back to v3.

```bash
swlist -l product NFS          # confirm NFS is installed
```

## Exporting / Sharing (Server)

The command set changed between releases: `exportfs`/`/etc/exports` on 11i v1/v2, and `share`/`/etc/dfs/dfstab` on 11i v3 (the Solaris-style model). This change trips up admins moving between releases: the *concepts* are identical (a persistent config file plus a command that activates it), but the file, its syntax, and the command names all differ. On 11i v3, `exportfs` still exists as a compatibility shim, but `share`/`shareall` is the supported interface.

The distinction between the config file and the running state matters. Editing `/etc/exports` (or `/etc/dfs/dfstab`) does *not* change what is currently exported — you must run `exportfs -a` (or `shareall`) to apply it. Conversely, `exportfs /data` or `share /data` on the command line exports *immediately* but does **not** persist across a reboot unless the entry is also in the config file. Keep the two in sync.

### 11i v1 and v2 — exportfs

```bash
exportfs -a                    # export everything in /etc/exports
exportfs /data                 # export one entry
exportfs -u /data              # unexport
exportfs                       # list current exports (no args)
```

`/etc/exports` entries:

```
/usr/share/man
/home        access=oakland:la
/opt/games   ro
/opt/appl    access=oakland:la,ro
/usr/local   rw=oakland
/etc/opt/appl root=oakland,access=la
```

### 11i v3 — share

```bash
shareall                       # share everything in /etc/dfs/dfstab
share /data                    # share one path
unshare /data                  # stop sharing
share                          # list current shares (no args)
```

`/etc/dfs/dfstab` entries (equivalent to the exports above):

```
share -d "man pages"      /usr/share/man
share -o access=oakland:la -d "user homes"  /home
share -o ro               -d "games"        /opt/games
share -o access=oakland:la,ro -d "application" /opt/appl
share -o rw=oakland       -d "open source"  /usr/local
share -o root=oakland,access=la -d "app config" /etc/opt/appl
share -o public,ro        -d "web NFS"       /docs
```

Common share/export options:

| Option | Meaning |
|--------|---------|
| `ro` / `rw` | Read-only / read-write (optionally `rw=host` to limit) |
| `access=host:host` | Restrict which clients may mount |
| `root=host` | Grant root access from the named host(s) |
| `public` | Mark as the WebNFS public filesystem |
| `anon=uid` | UID to map anonymous/unknown requests to (`-1` denies anonymous access) |
| `sec=sys\|krb5` | Security flavor: default AUTH_SYS, or Kerberos (`krb5`, `krb5i`, `krb5p`) on NFSv4 |
| `-d "text"` | Description (11i v3 `share`) |

A few pitfalls with these options are worth calling out. **`root=host` is a security decision, not a convenience** — it lets the named host's root map to server root, so a compromised client with that grant effectively has root on the exported tree; grant it narrowly. By default, without `root=`, remote root is *squashed* to the anonymous UID (root squashing), which is the safe default. Also note that access-list options like `access=` and `rw=` are matched by hostname, so they depend on name resolution being consistent — an entry that works by short name may fail if the client presents an FQDN, and vice versa. For anything crossing an untrusted boundary, prefer NFSv4 with `sec=krb5` over host-based `AUTH_SYS`, which trusts whatever UID the client asserts.

## Verifying Exports and Mounts

```bash
showmount -e                   # list exported filesystems (v1/v2/v3)
showmount -a                   # clients that have mounted from this server
rpcinfo -p                     # registered RPC programs (rpc.mountd, nfsd, etc.)
```

## Mounting (Client)

```bash
# Mount an NFS export
mount server:/home /mnt/home
mount -F nfs -o rw server:/home /mnt/home

# Persist in /etc/fstab
# server:/home  /mnt/home  nfs  rw,bg,intr  0 0

# Unmount all NFS filesystems
umount -aF nfs
```

### Client mount options that matter

| Option | Effect |
|--------|--------|
| `hard` | Retry forever on server timeout (default) — guarantees data integrity, but a process can hang if the server stays down |
| `soft` | Fail I/O after `retrans` retries — avoids hangs but can corrupt data; avoid for read-write mounts |
| `intr` | Allow signals to interrupt a process blocked on a hard mount (so you can `kill` it) |
| `bg` | Retry the mount in the background if the server is unavailable at boot |
| `rsize=`/`wsize=` | Read/write transfer sizes; larger values improve throughput on fast networks |
| `vers=3\|4` | Force a specific NFS protocol version |
| `proto=tcp\|udp` | Transport; TCP is the sensible default and required for NFSv4 |

The `hard` vs `soft` choice is the classic NFS trade-off. Use **`hard,intr`** for anything read-write: `hard` ensures a temporarily unreachable server never silently loses a write, and `intr` keeps you able to interrupt a stuck process. Reserve `soft` for read-only mounts where a hang would be worse than a failed read. The `bg` option is important in `/etc/fstab`: without it, a server that is down at boot will block the client's startup at the `localmount` stage until the mount times out.

## Daemons and Config Files

### Server

| File / script | Purpose |
|---------------|---------|
| `/etc/rc.config.d/nfsconf` | Enable server (and client) functionality (control variables) |
| `/etc/default/nfs` | NFS defaults incl. static ports (11i v3 only; reboot after changing) |
| `/etc/exports` | Export list (11i v1/v2) |
| `/etc/dfs/dfstab` | Share list (11i v3) |
| `/sbin/init.d/nfs.core` | Core RPC services (server and client) |
| `/sbin/init.d/lockmgr` | Lock manager (new in 11i v3; server and client) |
| `/sbin/init.d/nfs.server` | Server startup script |

### Client

| File / script | Purpose |
|---------------|---------|
| `/sbin/init.d/nfs.core` | Core RPC services |
| `/sbin/init.d/lockmgr` | Lock manager (11i v3) |
| `/sbin/init.d/nfs.client` | Client startup script |

Enable NFS in `/etc/rc.config.d/nfsconf` (set `NFS_SERVER=1` / `NFS_CLIENT=1`), then start the scripts (or reboot):

```bash
/sbin/init.d/nfs.core   start
/sbin/init.d/lockmgr    start      # 11i v3
/sbin/init.d/nfs.server start      # server
/sbin/init.d/nfs.client start      # client
```

## Monitoring

```bash
nfsstat                        # NFS usage statistics (server + client)
nfsstat -s                     # server side only
nfsstat -c                     # client side only
nfsstat -z                     # reset (zero) the statistics counters
nfsstat -m                     # per-mount timing (client): srtt, retransmits
```

When reading `nfsstat` output, the numbers to watch are **retransmissions** and **timeouts** on the client, and **badcalls** on the server. A high retransmit rate almost always points at the network or an overloaded server rather than at NFS itself — the client sent a request, got no reply in time, and re-sent it. `nfsstat -m` gives per-mount smoothed round-trip times, which is the quickest way to spot one slow export among many. Reset with `-z` before a test so you measure a clean window.

## Command Reference

| Task | 11i v1/v2 | 11i v3 |
|------|-----------|--------|
| Export all | `exportfs -a` | `shareall` |
| Export one | `exportfs /data` | `share /data` |
| Unexport | `exportfs -u /data` | `unshare /data` |
| List local exports | `exportfs` | `share` |
| Config file | `/etc/exports` | `/etc/dfs/dfstab` |
| Static NFS ports | — | `/etc/default/nfs` (NFSv4) |
| List exports (remote) | `showmount -e` | `showmount -e` |
| Clients mounted | `showmount -a` | `showmount -a` |
| RPC programs | `rpcinfo -p` | `rpcinfo -p` |
| Statistics | `nfsstat [-z]` | `nfsstat [-z]` |
| Unmount all NFS | `umount -aF nfs` | `umount -aF nfs` |

## Troubleshooting

Work the stack from the bottom up: network reachability, then RPC registration, then the export itself, then permissions.

### "RPC: Program not registered"

The server is reachable but a required daemon is not running or not registered with the portmapper. Check what is registered and restart the NFS services:

```bash
rpcinfo -p server              # is mountd/nfsd/statd/lockd registered?
/sbin/init.d/nfs.core   start
/sbin/init.d/nfs.server start
```

### Mount hangs or "server not responding"

The mount request is not getting a reply. Confirm basic reachability and that the portmapper answers, then verify the export is actually offered to this client:

```bash
ping server
rpcinfo -u server nfs          # does the NFS program respond over UDP?
showmount -e server            # is the path exported, and to us?
```

If `showmount -e` lists the path but the mount still fails, the cause is usually an `access=`/`rw=` host list that does not match how this client resolves — check forward and reverse DNS agree.

### Files owned by "nobody" on NFSv4

This is the classic NFSv4 domain mismatch: the client and server disagree on the `NFSv4 domain`, so identity strings fail to map. Align the domain in `/etc/default/nfs` on both ends and restart the mapping daemon.

### Stale file handle

`Stale NFS file handle` means the object the client cached was removed or the exported filesystem was recreated/reordered on the server (the file handle it holds no longer resolves). Unmount and remount the client:

```bash
umount -f /mnt/home
mount server:/home /mnt/home
```

### Permission denied despite correct export

Remember that NFS (AUTH_SYS) trusts UIDs, not usernames. If UID 500 is `alice` on the client but `bob` on the server, `alice` will access `bob`'s files. Keep UID/GID mappings consistent (via a shared name service) or use NFSv4 with Kerberos.

## Related Articles

- [HP-UX Filesystem Management (HFS, JFS/VxFS)](articles/hpux-filesystem-management.md) — the local filesystems you export
- [HP-UX Network Configuration](articles/hpux-network-configuration.md) — interfaces, routing, and name resolution NFS depends on
- [HP-UX Startup and Services](articles/hpux-startup-and-services.md) — the init.d scripts that start NFS daemons
- [HP-UX User and Password Administration](articles/hpux-user-password-administration.md) — keeping UID/GID mappings consistent across hosts
- [HP-UX System Information](articles/hpux-system-information.md) — checking installed products and release version
