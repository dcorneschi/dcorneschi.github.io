# GFS2 & RHEL Cluster Cheatsheet

## Cluster Configuration Methods

- Luci web interface
- Manual configuration
- ccs tool

## Configuration File

```
/etc/cluster/cluster.conf
```

## Cluster Commands

### cman_tool

| Command | Description |
|---------|-------------|
| `cman_tool status` | Display the local view of the cluster status |
| `cman_tool nodes` | Display the local view of the cluster nodes |
| `cman_tool services` | Display the local view of subsystems using cman |
| `cman_tool version -r` | Update the cluster configuration (first increase `config_version=<3>`) |
| `cman_tool status \| grep "Config Version:"` | Check the config version |

### clusvcadm

| Command | Description |
|---------|-------------|
| `clusvcadm -d <service_name>` | Stop and disable the service |
| `clusvcadm -e <service_name>` | Enable and start the service |
| `clusvcadm -r <service> <nodename>` | Relocate the service to another cluster member |

### ccs_tool

| Command | Description |
|---------|-------------|
| `ccs_tool create -2 <cluster_name>` | Create a 2-node cman cluster config file |
| `ccs_tool lsnode` | List nodes |
| `ccs_tool lsfence` | List fence devices |
| `ccs_tool delnode <nodename>` | Delete a node |
| `ccs_tool addnode <nodename> -n <nodeid>` | Add a node |

### ccs

| Command | Description |
|---------|-------------|
| `ccs -h hostname --sync --activate` | Sync and activate config file |
| `ccs -h hostname --checkconf` | Verify all nodes have the same cluster.conf |
| `ccs --getconf` | Print current cluster.conf file |
| `ccs_config_validate` | Verify the configuration |
| `ccs -h <nodename> --lsfencedev` | List all configured fence devices |
| `ccs -h <nodename> --lsfenceinst` | List all fence methods and instances |
| `ccs -h <nodename> --stop` | Stop and disable cluster services on reboot |
| `ccs -h <nodename> --start` | Start and enable cluster services on reboot |
| `ccs --stopall` | Stop and disable cluster services on all nodes |
| `ccs --startall` | Start and enable cluster services on all nodes |
| `ccs --lsservices` | List currently configured services and resources |

### Cluster Status

| Command | Description |
|---------|-------------|
| `clustat` | Display the status of the cluster |
| `clustat -i 1` | Display cluster status, refresh every 1 second |

### GFS2 Tools

| Command | Description |
|---------|-------------|
| `gfs2_tool journals <mountpoint>` | List the file system's journals |
| `gfs2_tool gettune <mountpoint>` | Print current values of tuning parameters |
| `tunegfs2 -l /dev/<vg>/<lv>` | Display filesystem parameters (newer replacement for `gfs2_tool gettune`) |

### DLM & Lockspaces

| Command | Description |
|---------|-------------|
| `dlm_tool ls` | List DLM lockspaces and their status |
| `dlm_tool status` | Show DLM status overview |
| `group_tool` | List lockspaces and their members |

### Fencing

| Command | Description |
|---------|-------------|
| `fence_tool ls` | Display internal fenced state |

## Service Management

### Stop all services

```sh
service rgmanager stop
service gfs2 stop
service clvmd stop
service cman stop
```

### Start all services

```sh
service cman start
service clvmd start
service gfs2 start
service rgmanager start
```

### Enable/disable services at boot

```sh
# Disable
for i in cman rgmanager gfs2 clvmd; do chkconfig $i off; done

# Enable
for i in cman rgmanager gfs2 clvmd; do chkconfig $i on; done

# Check
for i in rgmanager gfs2 clvmd cman; do chkconfig --list $i; done
```

## Check GFS2 Filesystem

1. Make sure the filesystem is not mounted anywhere
2. Activate the VG if clvmd is not running:

```sh
vgchange -ay <VG> --config 'global {locking_type = 0}'
```

3. Run fsck:

```sh
fsck.gfs2 -v -y /dev/<VG>/<LV> 2>&1 | tee /root/gfs2_fsck.log
```

## Mount GFS2 Without Cluster Infrastructure

1. Make sure the filesystem is not mounted anywhere
2. Activate the VG:

```sh
vgchange -ay <VG> --config 'global {locking_type = 0}'
```

3. Mount with no lock protocol:

```sh
mount -t gfs2 -o lockproto=lock_nolock <block device> <mount point>
```

## Check for Withdrawal

If the value is `1`, the filesystem has withdrawn:

```sh
cat /sys/fs/gfs2/myCluster:myFS/withdraw
```

## Manual Fencing

Using cluster.conf:

```sh
fence_node -vv <nodename>
```

IPMI over LAN:

```sh
fence_ipmilan -a 192.168.100.30 -l <username> -p <password> -o reboot
```

VMware:

```sh
fence_vmware_soap -o reboot -a 192.168.100.30 -l <username> -p <password> -z -n <uuid>
```

## Fixing a Failed Service

Convince rgmanager the problem is fixed by disabling the service first, then re-enabling it:

```sh
clusvcadm -d <service>
clusvcadm -e <service>
```

## Crash the OS (Testing)

```sh
echo 1 > /proc/sys/kernel/sysrq
echo c > /proc/sysrq-trigger
```

## Create a GFS2 Filesystem

```sh
mkfs.gfs2 -p lock_dlm -t <cluster_name>:<fs_name> -j <number_of_journals> /dev/<vg>/<lv>
```

- `-p lock_dlm` — use the DLM lock protocol (required for clustered access)
- `-t <cluster_name>:<fs_name>` — lock table name, must match `cluster.conf` cluster name
- `-j <number_of_journals>` — one journal per node that will mount the filesystem

Example for a 2-node cluster:

```sh
mkfs.gfs2 -p lock_dlm -t mycluster:myfs -j 2 /dev/vg_shared/lv_data
```

## Grow a GFS2 Filesystem

`gfs2_grow` expands the filesystem to fill the underlying device. It runs online and is cluster-aware — all nodes see the new size immediately.

```sh
gfs2_grow <mountpoint>
```

> GFS2 cannot be shrunk. This operation is non-reversible.

## Add Journals

Before a new node can mount the filesystem, you must add a journal for it. Run from any node that already has the filesystem mounted:

```sh
gfs2_jadd -j 1 <mountpoint>
```

Verify journals:

```sh
gfs2_tool journals <mountpoint>
```

## Quorum and 2-Node Clusters

In a 2-node cluster, losing one node means losing quorum. The `two_node` directive in `cluster.conf` tells cman to allow the remaining node to continue operating:

```xml
<cman two_node="1" expected_votes="1"/>
```

Without this, the surviving node will fence itself when it loses quorum.

## tunegfs2

`tunegfs2` replaces `gfs2_tool gettune` on newer systems. View or modify filesystem parameters:

```sh
# Display filesystem parameters
tunegfs2 -l /dev/<vg>/<lv>

# Set a parameter (e.g., quota enforcement)
tunegfs2 -o quota=on /dev/<vg>/<lv>
```

## Troubleshooting

### GFS2 Withdrawal

A withdrawal means GFS2 detected an inconsistency and has gone read-only to protect data. Check for withdrawal:

```sh
cat /sys/fs/gfs2/<cluster_name>:<fs_name>/withdraw
```

A value of `1` means the filesystem has withdrawn. Check logs for the root cause:

```sh
grep -i "withdraw" /var/log/messages
grep -i "GFS2" /var/log/messages
```

Common causes:
- Storage I/O errors or path failures
- Journal replay failures
- Memory pressure causing metadata writeback failures

Recovery typically requires unmounting the filesystem on the affected node, running `fsck.gfs2`, and remounting.

### Unexpected Fencing

When a node is fenced unexpectedly, check:

```sh
# On the surviving node — check fenced logs
grep -i "fence" /var/log/messages

# Check if the fenced node lost quorum
grep -i "quorum" /var/log/messages

# Check corosync/cman communication
grep -i "cman" /var/log/messages

# DLM lockspace status
dlm_tool ls
```

Common causes:
- Network issues between nodes (corosync heartbeat timeout)
- Node hanging or unresponsive (high load, D-state processes)
- Fencing agent misconfiguration

### Hung or Slow Filesystem

```sh
# Check for D-state processes waiting on GFS2
ps aux | grep " D"

# Check DLM lock contention
dlm_tool lockdebug <lockspace_name>

# Check for pending fencing operations
fence_tool ls
```
## Fencing Methods

Fencing disconnects a misbehaving node from shared storage to guarantee data integrity, and can remove it from the cluster. Red Hat requires fencing in all clustered environments. Supported methods include:

- **Network-power fencing** — cut power via a network power switch (e.g. `fence_apc`).
- **Network-switch fencing** — disable the node's switch ports (Fibre Channel or Ethernet).
- **Virtual-machine fencing** — kill/reboot a guest via the hypervisor (e.g. `fence_vmware_soap`).
- **Storage fencing / SCSI persistent reservations** — `fence_scsi` revokes the node's registration on the shared device instead of powering it off. Only works if the storage supports SCSI-3 persistent reservations.

A node can be configured with one method or several. With multiple methods, they cascade in the order listed in `cluster.conf` until one succeeds.

> Manual fencing (`fence_manual`) is **not supported in production** — it can cause file-system corruption if an admin acknowledges a fence before the failed node is truly cut off from I/O. Use `fence_ack_manual` (RHEL 6+) only after verifying the node is powered off or physically disconnected. `fence_manual` was removed in RHEL 6.

## Fencing a Node with Dual Power Supplies

A node with two power supplies needs one fence device per supply, and the method must turn **both off, then both on** — otherwise the node keeps running on the remaining supply and the fence never completes.

```xml
<clusternode name="node-01" votes="1">
  <fence>
    <method name="1">
      <device name="pwr01" action="off" switch="1" port="1"/>
      <device name="pwr02" action="off" switch="1" port="1"/>
      <device name="pwr01" action="on"  switch="1" port="1"/>
      <device name="pwr02" action="on"  switch="1" port="1"/>
    </method>
  </fence>
</clusternode>
...
<fencedevices>
  <fencedevice agent="fence_apc" ipaddr="192.168.0.101" login="admin" name="pwr01" passwd="XXXX"/>
  <fencedevice agent="fence_apc" ipaddr="192.168.1.101" login="admin" name="pwr02" passwd="XXXX"/>
</fencedevices>
```

## Fence-Race Delay in Two-Node Clusters

In `two_node` mode both nodes always have quorum, so a network partition triggers a **fence race**: each node tries to fence the other at the same time. With integrated power management (e.g. HP iLO) this can power off *both* nodes, leaving the cluster fully down since no power-on command is issued.

Fix it by giving one node's fence device a `delay`, so the other node gets a head start and wins the race:

```xml
<fencedevices>
  <fencedevice name="node1-fence" agent="fence_ilo"
    ipaddr="node1-ilo" login="user" passwd="passwd" delay="30" />
  <fencedevice name="node2-fence" agent="fence_ilo"
    ipaddr="node2-ilo" login="user" passwd="passwd" />
</fencedevices>
```

Here the cluster waits 30s before fencing `node1`, so `node2` (no delay) wins. Tune the delay per cluster: long enough for a fence to complete plus a margin for network congestion.

## Calling a Fence Agent from stdin

When invoked by `fenced`, agents read arguments from standard input. To test manually you can feed stdin directly:

```sh
# Redirect args from a file
cat /tmp/fence_ipmilan-args
# ipaddr=192.168.1.42
# login=<username>
# password=<password>
# action=status
/usr/sbin/fence_ipmilan < /tmp/fence_ipmilan-args

# Or echo them inline
echo -e "ipaddr=192.168.1.42\nlogin=<username>\npasswd=<password>\naction=status" | fence_ipmilan
```

Agents live in `/sbin/fence_*` (RHEL 5) or `/usr/sbin/fence_*` (RHEL 6). If `fence_node` fails but a direct agent call works, the problem is in `cluster.conf`; if the direct call also fails, the agent options or the fence device itself are wrong.
## GFS2 Best Practices & Tuning

> These are RHEL 5/6-era Red Hat recommendations. The principles (small filesystems, 4K blocks, sensible journals, per-node locality) still hold, but some specifics (DLM table tuning, the 768MB resource-group limit) are historical — newer kernels self-tune more. Benchmark on a test cluster before applying in production.

### Formatting

| Guideline | Recommendation |
|-----------|----------------|
| Filesystem size | Smaller is better — prefer 10×1TB over 1×10TB (faster/cheaper `fsck.gfs2`, backups, fewer resource groups) |
| Block size | Stick with the default **4K** — matches the Linux page size, so the kernel does less buffer work |
| Journal size | Default **128MB** is usually optimal. Don't shrink to 8/32MB — small journals fill fast and stall on writes. 256MB may help larger filesystems |
| Journal count | One journal per node that mounts (add later with `gfs2_jadd`) |
| Resource group size | Keep **smaller than 768MB** (Red Hat lab found problems above that). Override the auto-estimate with `mkfs.gfs2 -r <MB>` |

Too many tiny resource groups → allocations waste time searching for free blocks (each search takes a cluster-wide lock). Too few large ones → nodes contend for the same RG lock. Experiment to find the balance.

### Mount Options

Mount with `noatime` and `nodiratime` so GFS2 isn't updating inode access times on every read:

```sh
mount -t gfs2 -o noatime,nodiratime /dev/<vg>/<lv> <mountpoint>
```

### Block Allocation (reduce DLM lock contention)

GFS2 with DLM performs best when each node works in its own resource group. The first node to lock a resource becomes its **lock master**; locking on the master is fast, asking a remote master is slow.

- **Let each node allocate its own files** — avoid one node creating files that others then append to.
- **Keep each node in its own resource group** — GFS2 already starts nodes in different RGs; don't fight it.
- **Preallocate when possible** — use the `fallocate` syscall to reserve blocks up front and skip per-write allocation.
- Worst case: a central file that every node appends to — all nodes fight for the same RG lock.

### DLM Table Tuning (RHEL 6.0 and earlier)

Increasing DLM table sizes could improve performance on older kernels. RHEL 6.1+ raised the defaults, so this is rarely needed now:

```sh
echo 1024 > /sys/kernel/config/dlm/cluster/lkbtbl_size
echo 1024 > /sys/kernel/config/dlm/cluster/rsbtbl_size
echo 1024 > /sys/kernel/config/dlm/cluster/dirtbl_size
```

Not persistent — add to a startup script if used. GFS2 itself is self-tuning; the old GFS knobs (`demote_secs`, `glock_purge`, `inoded_secs`) no longer exist.

### VFS Tuning

Tune the VFS layer underneath GFS2 via `sysctl`, then persist in `/etc/sysctl.conf`:

```sh
# Read current values
sysctl -n vm.dirty_background_ratio
sysctl -n vm.vfs_cache_pressure

# Adjust (example values — test for your workload)
sysctl -w vm.dirty_background_ratio=20
sysctl -w vm.vfs_cache_pressure=500
```

### Backups

Make regular backups regardless of RAID/mirroring/snapshots — redundancy is not a backup.

- Reading the whole filesystem from one node holds locks/cache and hurts other nodes. Drop caches when done:

```sh
echo -n 3 > /proc/sys/vm/drop_caches
```

- Best approach: take a **hardware SAN snapshot**, present it to a non-cluster host, and back it up there — mount that copy with `-o lockproto=lock_nolock`.
- Faster still: have each node back up its own directories (e.g. a per-node `rsync` script) to split the load.

### SELinux

Avoid SELinux on GFS2. It stores extended attributes on every filesystem object, and maintaining them slows GFS2 down considerably.

### NFS over GFS2

NFS is not cluster-aware. Red Hat supports **active/passive only** (one node serves NFS at a time). If clients use POSIX/`fcntl()` locks, mount with `localflocks` so locks stay local to each server:

```sh
mount -t gfs2 -o localflocks /dev/<vg>/<lv> <mountpoint>
```

Have all clients mount from a single server to avoid two servers granting the same lock independently. If unsure whether an app uses POSIX locks, use `localflocks` for safety.

### Samba (SMB/Windows) over GFS2

Use Samba with **CTDB** for active/active serving. Historically a tech preview with no GFS2 cluster-lease support (which slows serving) — verify current support status for your release.

### Cluster & Hardware Sizing

- **Node count**: more nodes = better fault tolerance but more lock/network traffic. Red Hat recommends **no more than 16 nodes**.
- **Storage**: works on iSCSI/FCoE, but SAN with Fibre Channel (Red Hat's primary test platform) performs best. Bigger cache helps.
- **Network**: faster gear helps, but test first — some high-end switches mishandle the multicast used for `fcntl` flocks, while commodity switches sometimes do better.

### Converting GFS1 → GFS2

Use `gfs2_convert`, but beware: if the old GFS filesystem was grown with `gfs_grow`, its resource groups may be unevenly spaced, which can break `fsck.gfs2`. Safest path is to `mkfs.gfs2` a fresh device (evenly spaced RGs) and copy the data across. RHEL 6's `fsck.gfs2` handles uneven RGs better.
## Performance & Diagnosing Hangs

GFS2 behaves like a local filesystem except for **caching**, which is coordinated cluster-wide by **glocks** (GFS locks) over DLM. Understanding glocks is the key to diagnosing slowness and hangs.

### Glock Concepts

- **Glock** — a cluster-wide lock on a filesystem resource. All cross-node cache coordination happens here; there is no other inter-node communication.
- **Holder** — a process using (or waiting for) a glock. Holders are either *granted* or *waiting*.
- **Caching is split** — only one node caches a part of the FS at a time for writes; multiple nodes may cache the same part read-only. When a node requests exclusive (write) access to something cached elsewhere, every other node must flush and drop that cache — far slower than a local access.
- **Glock types** in a dump are shown as `type,number`:
  - **type 2** = inode (the `number` is the inode's disk location in FS blocks, and its inode number)
  - **type 3** = resource group
  - **type 6** = flock
  - Most contention shows up on types 2 and 3.

Directories behave like files: a create/unlink needs exclusive access to the directory glock, forcing other nodes to reread it. Split write-heavy directories into hashed subdirectories to cut contention.

### Is a Task Stuck or Just Slow?

Take two glock dumps a few seconds/minutes apart and compare the **waiting** holders (ignore granted):

- Same glocks, identical waiting-holder list in both dumps → the cluster is **stuck**.
- List changed → it's just **slow** (making progress).

Other signals: heavy DLM network traffic means lots of locking activity (an indicator, not proof, of a problem). On RHEL 6+ use GFS2 tracepoints to watch activity.

Map a contended type-2 glock to a file by converting its hex number to decimal and searching (run on an otherwise-idle FS — it reads every inode):

```sh
find <mountpoint> -inum <inode_number>
```

`gfs2_quotad` appearing stuck (even with no quotas) is usually a *symptom* — it also updates `statfs`, so it queues behind whatever the real problem is.

### Triage Checklist

When a cluster is slow or hung, determine:

- Is it a **performance** problem (slow), a **bug** (stuck / kernel panic / assertion), or **corruption** (usually a filesystem withdraw)?
- Reproducible or one-off? Single node or only in the cluster? Same node or moving around?
- Does it correlate with a time of day or event (nightly backups, cron)?
- **Reproduce on a single node if possible** — if it still happens with one node mounted, it's likely not a clustering issue at all.

### Mount Options for Performance

```sh
# noatime/nodiratime: stop reads from becoming atime writes (biggest win)
mount -t gfs2 -o noatime,nodiratime /dev/<vg>/<lv> <mountpoint>
```

- **`nobarrier`** — GFS2 uses I/O barriers by default (RHEL 6+) when flushing the log. Safe to disable *only* if the storage has no write cache or a non-volatile (battery/UPS-backed) cache. Auto-disabled if the device doesn't support barriers.
- **`discard`** — issue TRIM/discard on deallocation for thin-provisioned/SSD backing (needs volume-manager and device support). Small overhead; a discard implies a barrier.
- **`statfs_quantum` / `statfs_percent`** (RHEL 6+) — speed up `statfs`/`df`; the modern replacement for the old `statfs_fast` tunable.
- Avoid journaled-data mode (`chattr +j`) unless required; default ordered-data mode is fine. Use `fsync(2)` on the file *and its parent dir* to guarantee data is on disk.

### File Locking

| Lock type | Mechanism | Notes |
|-----------|-----------|-------|
| `flock` | type 6 glock (via DLM) | Faster; preferred on performance grounds, especially at higher node counts |
| `fcntl` / POSIX | userspace via corosync (not DLM) | Rate-limited to **100 locks/sec** by default; raise in `cluster.conf` (0 = unlimited). Not suited to high-performance locking |
| Leases | — | Not supported on GFS or GFS2 |

`localflocks` makes both flock and POSIX locks node-local — required for NFS-exported GFS2, and NFS-over-GFS2 is only supported active/passive (single active server), not alongside Samba or local apps.

### Application Patterns

- **Email (IMAP/sendmail)** — give each user a "home" node they normally connect to (files stay cached there); use `maildir` not `mbox` so each message has its own lock. Contention in maildir is almost always the directory lock.
- **Web servers** — a good fit (content caches read-only on all nodes). To update, build a fresh copy and swap it in with a bind mount + `mount --move` rather than editing files in place.
- **Backups** — best to back up each node's own working set from that node (keeps caches warm, spreads load); or take a hardware SAN snapshot. If a single node reads the whole FS, run `echo -n 3 > /proc/sys/vm/drop_caches` afterward.

### The `ls --color` Trap

On a slow GFS2 filesystem, people instinctively run `ls` — but `--color=tty` forces a `stat()` on every entry, generating extra lock requests and *worsening* contention. Disable the color aliases on cluster nodes:

```sh
# add to /etc/profile (or ~/.bash_profile per user)
alias ll='ls -l' 2>/dev/null
alias l.='ls -d .*' 2>/dev/null
unalias ls
```

Generally, avoid heavy `ls` use on a contended GFS2 mount.

### Legacy Tunables (RHEL 4/5 only — removed in RHEL 6)

These `gfs_tool settune` knobs no longer exist on RHEL 6+ (GFS2 self-tunes; control caching via `fsync`/`fadvise`/`madvise` instead). Listed for reference on old GFS/RHEL 5 systems:

```sh
# Percentage of unused glocks to purge every 5s (0 disables; 30-60 typical). Not persistent.
gfs_tool settune /path/to/mount glock_purge 50

# Demote write locks / flush cached data every N seconds (default 300)
gfs_tool settune /mnt/gfs1 demote_secs 200

# Speed up statfs on GFS (RHEL 4.5+)
gfs_tool settune /path/to/mount statfs_fast 1
```

### Solid-State & Network Notes

- **SSD** helps most where glock contention is high (much lower seek time).
- **Network**: latency matters more than throughput for GFS2. Ensure multicast works between all nodes (used for flocks). If cluster traffic shares a NIC with storage/app traffic, use `tc` to cap bandwidth. Avoid jumbograms unless the network also carries storage traffic.
## Quorum Disk (QDisk)

A quorum disk (`qdiskd`) is a shared-storage tie-breaker that casts extra quorum votes and runs fitness *heuristics* to decide which nodes stay in the cluster.

> Red Hat recommends **not** deploying QDisk unless genuinely needed — it adds complexity and misconfiguration risk. For the classic two-node fence-race case, the fencing `delay` option (see above) is simpler and is the preferred solution on RHEL 5.6+/6.1+.

### When QDisk Helps

- **Fence race with separate fencing/heartbeat networks** — where a node that lost its cluster link can still reach its fence device, both nodes can fence simultaneously ("fence death", both powered off). QDisk (or, preferably, `delay`) picks a predetermined winner.
- **Fencing loops** — a fenced node reboots, still can't rejoin, fences the survivor, repeat. QDisk breaks the loop (alternative: disable cluster services at boot via `chkconfig`).
- **Last man standing** — keep a cluster quorate with fewer nodes than a normal majority (accepting that one node may not handle full load).
- **Service-owner preference / join prerequisites** — heuristics score nodes so the service owner wins, or gate a node's membership on a check (e.g. a required network is reachable).

### Requirements & Caveats

- Needs concurrent, synchronous, real-time access to **shared storage** (most SAN arrays are fine). Timings must tolerate multipath failover.
- **QDisk is not a fencing substitute** — it still requires real I/O fencing (power fencing recommended); `qdiskd` uses the cluster's fence mechanism to evict stalled nodes.
- Use the **`deadline`** I/O scheduler on the quorum LUN.
- **Not supported** on distributed or replicated storage.
- With multipath, use **longer** cluster timeouts (see below).

### cluster.conf Structure

```xml
<cman expected_votes="3" quorum_dev_poll="21000"/>

<quorumd label="myQDisk" interval="1" tko="10" min_score="1" votes="1">
  <heuristic program="ping -c1 -w1 192.168.2.1" score="1" interval="2" tko="4"/>
</quorumd>

<totem token="21000"/>
```

> **RHEL 6.1+ auto-calculates most of this.** `interval`, `tko`, `votes`, and `quorum_dev_poll` are derived from `cluster.conf`; focus on getting `totem token` right rather than hand-tuning the rest. In a two-node cluster with no heuristics, `qdiskd` enables `master_wins` automatically.

### Timing Rules (manual tuning, RHEL 5 / 6.0)

For a cluster of **N** nodes:

| Setting | Rule |
|---------|------|
| `totem token` | Greater than **2 × (quorumd `tko` × `interval`)**. Value is in **milliseconds** (other values are whole seconds) |
| `cman quorum_dev_poll` | Equal to `totem token` |
| `cman expected_votes` | **2N − 1** |
| `cman two_node` | `0` or absent |
| `quorumd votes` | **N − 1** |
| Each `clusternode votes` | `1` |
| Heuristic `tko × interval` | ≤ quorumd `interval × (tko − 1)` |

Example above: quorumd `tko×interval` = 10s, so `totem token` = 21000ms (21s).

### Multipath Strategies

- **Minimize failover time** — if any node can take over cheaply, don't wait for path failover. Use normal QDisk settings; a node with misbehaving storage is evicted after `quorumd interval × tko` seconds.
- **Allow time for recovery** — if evicting a node is costly, make `quorumd interval × tko` **greater than the measured multipath failover time**, then adjust `totem token` and `quorum_dev_poll` to match. Failover time varies by environment (an unresponsive target/switch can take minutes) — test real failure scenarios.

```xml
<quorumd label="myqdisk" min_score="1" interval="2" tko="10">
  <heuristic program="ping -c1 -w1 192.168.2.1" score="1" interval="2" tko="4"/>
</quorumd>
```

> On RHEL 5.5 a bug fix made cluster failover time roughly **3 × token**. If `totem token` = multipath_failover × 2.7, expect failover ≈ multipath_failover × 2.7 × 3 (e.g. 30s → ~243s / 4+ min). Budget for this.

### Heuristics

Each heuristic returns a `score` when its `program` succeeds. Scores are summed; if the total drops below `min_score`, the node removes itself (usually reboots).

- **Ping** — remove a node that can't reach a critical resource (e.g. the client-facing router). Never gate membership on a single packet — use heuristic `tko > 1` (RHEL 5).
- **SAN connectivity** — QDisk already implicitly monitors the path *to the quorum disk*; add a heuristic to watch other storage paths if needed.
- **Network link state** — monitor lower-level link status (Ethernet/InfiniBand) rather than just ping, using standard RHEL tools.
