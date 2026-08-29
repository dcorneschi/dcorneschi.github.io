# Migrating VMs and LXC Containers Between Proxmox Nodes

Moving guests between nodes in a Proxmox cluster — for maintenance, load balancing, or before a [node shutdown](articles/proxmox-two-node-cluster-quorum.md). This covers live vs offline migration, bulk scripting, the CLI (`qm`/`pct`) and API (`pvesh`) approaches, local-disk migration, pre/post checks, and troubleshooting.

> Migration commands run on the **Proxmox host** as `root`. VM commands use `qm`, container commands use `pct`, and the cluster API is reached with `pvesh`.

## Prerequisites

1. **Cluster**: both nodes are members of the same Proxmox cluster (`pvecm status`).
2. **Storage**: shared storage (Ceph, NFS, GlusterFS) for seamless migration, or local storage with disk migration enabled.
3. **Network**: bridges (e.g. `vmbr0`) and any VLANs referenced by the guest must exist on the target node.
4. **Resources**: enough CPU, RAM, and storage headroom on the target.

> **VM vs container downtime**: QEMU VMs support true live migration with no downtime. LXC containers cannot be migrated fully live — an online container migration uses `--restart`, which briefly stops and restarts the container on the target.

## Method 1: Live Migration (recommended)

Best for production with minimal downtime.

### Single guest

```bash
# VM — live migration (no downtime)
qm migrate <vmid> <target-node> --online

# Container — restart migration (brief stop/start)
pct migrate <ctid> <target-node> --restart
```

### All guests on a node (bulk, via the API)

```bash
SOURCE_NODE="pve1"
TARGET_NODE="pve2"

# Migrate all VMs
for vmid in $(pvesh get /nodes/$SOURCE_NODE/qemu --output-format=json | jq -r '.[].vmid'); do
    echo "Migrating VM $vmid..."
    pvesh create /nodes/$SOURCE_NODE/qemu/$vmid/migrate --target $TARGET_NODE --online 1
done

# Migrate all containers
for ctid in $(pvesh get /nodes/$SOURCE_NODE/lxc --output-format=json | jq -r '.[].vmid'); do
    echo "Migrating Container $ctid..."
    pvesh create /nodes/$SOURCE_NODE/lxc/$ctid/migrate --target $TARGET_NODE --restart 1
done
```

## Method 2: Offline Migration

Best when live migration is not possible (incompatible CPU flags, certain local-device passthrough) or during a maintenance window.

```bash
# VMs — stop, then migrate
for vmid in $(pvesh get /nodes/$SOURCE_NODE/qemu --output-format=json | jq -r '.[].vmid'); do
    qm stop $vmid
    qm migrate $vmid $TARGET_NODE
done

# Containers — stop, then migrate
for ctid in $(pvesh get /nodes/$SOURCE_NODE/lxc --output-format=json | jq -r '.[].vmid'); do
    pct stop $ctid
    pct migrate $ctid $TARGET_NODE
done
```

## Method 3: Proxmox Web UI

1. Log into the web interface.
2. Select the guest under the source node.
3. Click **Migrate** (top-right), or right-click the guest → **Migrate**.
4. Choose the target node and options, then confirm.

## Method 4: Backup and Restore (non-clustered)

When the nodes are not in a cluster, move guests via a backup archive on shared/backup storage.

```bash
# On the source node — back everything up
vzdump --all --storage BACKUP_STORAGE

# On the target node — restore
qmrestore <backup-file> <vmid> --storage TARGET_STORAGE   # VM
pct restore <ctid> <backup-file> --storage TARGET_STORAGE  # container
```

## Local-Disk Migration

If the guest's disks live on local storage rather than shared storage, the disk contents must move too.

```bash
# VM with local disks
qm migrate <vmid> <target-node> --online --with-local-disks

# Container on local storage
pct migrate <ctid> <target-node> --restart
```

## Advanced Options

```bash
# Limit migration bandwidth (MiB/s) to protect the network
pvesh create /nodes/$SOURCE_NODE/qemu/<vmid>/migrate \
    --target $TARGET_NODE --online 1 --bwlimit 100

# Send disks to a specific storage on the target
qm migrate <vmid> <target-node> --targetstorage <storage-name>

# Force migration past certain checks (use with care)
qm migrate <vmid> <target-node> --force
```

## Complete Migration Script

Moves every VM and container off a node, with a pause between each to avoid saturating the network.

```bash
#!/bin/bash
set -euo pipefail

SOURCE_NODE="pve1"
TARGET_NODE="pve2"

echo "Starting migration from $SOURCE_NODE to $TARGET_NODE"

echo "Migrating VMs..."
for vmid in $(pvesh get /nodes/$SOURCE_NODE/qemu --output-format=json | jq -r '.[].vmid'); do
    echo "  VM $vmid"
    pvesh create /nodes/$SOURCE_NODE/qemu/$vmid/migrate --target $TARGET_NODE --online 1
    sleep 5
done

echo "Migrating LXC containers..."
for ctid in $(pvesh get /nodes/$SOURCE_NODE/lxc --output-format=json | jq -r '.[].vmid'); do
    echo "  CT $ctid"
    pvesh create /nodes/$SOURCE_NODE/lxc/$ctid/migrate --target $TARGET_NODE --restart 1
    sleep 5
done

echo "Migration completed."
```

## Pre-Migration Checks

```bash
# Cluster health and node membership
pvecm status
pvecm nodes

# Target node resources (CPU, memory, uptime)
pvesh get /nodes/$TARGET_NODE/status

# Storage available on the target
pvesh get /nodes/$TARGET_NODE/storage

# What is on the source node?
qm list
pct list
```

## Post-Migration Verification

```bash
# VM landed and is running
qm status <vmid>
qm config <vmid>

# Container landed and is running
pct status <ctid>
pct config <ctid>

# Watch running cluster tasks
pvesh get /cluster/tasks
tail -f /var/log/pve/tasks/active
```

## Troubleshooting

| Issue | What to check |
|-------|---------------|
| Storage not accessible on target | Storage is mounted/enabled on the target node; content type and permissions match |
| Network mismatch | The bridge (`vmbr0`, etc.) and any VLAN tags used by the guest exist on the target |
| Insufficient resources | Free RAM/CPU/storage on the target; migrate in smaller batches |
| Migration stuck or slow | Network path between nodes, bandwidth saturation; try `--bwlimit` or offline migration |
| VM live migration refused | CPU model/flags differ between nodes; use a common CPU type or migrate offline |

```bash
# Diagnostics
pvesh get /cluster/tasks     # migration task status
iftop                        # migration bandwidth
ping <target-node>
ssh <target-node> hostname   # inter-node SSH must work
```

## Important Notes

1. **Back up first** — take a `vzdump` before large migrations.
2. **Test in a lab** before doing it in production.
3. **Pick a low-usage window**, especially for local-disk migrations that copy data.
4. **Monitor progress** and node resources during the run.
5. **Have a rollback plan** — you can migrate back to the source if something misbehaves.
6. **Mind quorum** — on a small cluster, don't shut the source node down until the target is confirmed healthy. See [Two-Node Cluster Quorum](articles/proxmox-two-node-cluster-quorum.md).

## Example Workflow

```bash
# 1. Prepare
pvecm status
pvesh get /nodes/pve2/status
qm list; pct list

# 2. Execute (bulk script, or per-guest)
qm migrate 100 pve2 --online
pct migrate 200 pve2 --restart

# 3. Verify
qm status 100
pct status 200

# 4. Only now, if draining the node for maintenance, shut it down
```
