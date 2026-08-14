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
