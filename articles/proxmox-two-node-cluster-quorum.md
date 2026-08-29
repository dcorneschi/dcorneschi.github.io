# Proxmox Two-Node Cluster Quorum

A Proxmox cluster uses corosync's votequorum to decide whether it has a **majority** of votes before allowing write operations to `/etc/pve` and running HA services. With two nodes, majority is 2 votes — so losing one node drops you to 1 vote, the cluster loses quorum, and it becomes read-only until quorum returns. This guide covers how to keep a two-node cluster usable when a node is offline, and how to shut a node down cleanly.

> Quorum protects against split-brain (two nodes each thinking they are authoritative). The techniques below trade some of that safety for availability. Understand the tradeoff before applying them, especially with shared storage or HA.

## Understanding the Problem

Each node normally contributes 1 vote. Quorum requires a strict majority:

| Nodes | Total votes | Votes for quorum | Survives losing |
|-------|-------------|------------------|-----------------|
| 2 | 2 | 2 | 0 nodes |
| 2 + QDevice | 3 | 2 | 1 node |
| 3 | 3 | 2 | 1 node |

So a plain two-node cluster cannot tolerate a single node going down without intervention. There are three practical ways to fix this: `two_node` mode, a QDevice witness, or manually overriding expected votes.

## Check Cluster and Quorum Status

Start here to see where you stand.

```bash
# Overall cluster status (nodes, votes, quorum)
pvecm status

# Detailed quorum view
corosync-quorumtool -s

# List members and their votes
corosync-quorumtool -l

# The live corosync configuration
cat /etc/pve/corosync.conf
```

## Option 1: Two-Node Mode (recommended for planned single-node operation)

Corosync has a dedicated `two_node` setting. When enabled, quorum is calculated as if only 1 vote is required, so the cluster stays quorate with a single node online. It also enables `wait_for_all` by default, meaning after a full restart both nodes must be seen once before the cluster becomes quorate.

Edit the cluster-wide corosync config (it lives in the replicated `/etc/pve`):

```bash
nano /etc/pve/corosync.conf
```

Set the quorum section and **bump `config_version`** so the change propagates:

```ini
quorum {
    provider: corosync_votequorum
    two_node: 1
}
```

```bash
# corosync reloads /etc/pve/corosync.conf automatically when config_version increases;
# you can also reload explicitly:
systemctl reload corosync

# Verify two-node mode is active
corosync-quorumtool -s
```

> Always increment the `totem { config_version: N }` value when editing `corosync.conf`, and edit the copy under `/etc/pve/` (not `/etc/corosync/`) so it replicates to all nodes. A malformed file can break the cluster — keep a backup.

## Option 2: Add a QDevice (recommended for production)

A QDevice adds an external tie-breaker (a "witness") that casts a vote, giving you 3 total votes so the cluster survives losing either real node. The witness runs `corosync-qnetd` on a separate lightweight host (it does not need to be a Proxmox node).

On the external witness host:

```bash
apt update
apt install -y corosync-qnetd
systemctl enable --now corosync-qnetd
```

On **both** cluster nodes, install the client:

```bash
apt update
apt install -y corosync-qdevice
```

Then register the QDevice from any one cluster node:

```bash
pvecm qdevice setup <qnetd-server-ip>
```

Verify:

```bash
pvecm status
corosync-quorumtool -s     # should show a Qdevice vote
```

This is the cleanest option — no manual vote juggling, and it keeps split-brain protection intact.

## Option 3: Manually Override Expected Votes (temporary)

If a node is already down and the cluster is stuck read-only, you can lower the expected vote count on the surviving node so it regains quorum:

```bash
# On the remaining online node
pvecm expected 1
```

This is a runtime-only change — it does not persist across reboots. Reset it once both nodes are back:

```bash
pvecm expected 2
```

> Use this only as a recovery measure. Forcing quorum with a node you cannot see risks split-brain if that node is actually still running and reachable by clients/storage.

## Shutting a Node Down Cleanly

### Migrate workloads first

Before powering off a node, move its running guests to the other node so they stay available:

```bash
# Migrate a VM (online/live migration)
qm migrate <vmid> <target-node> --online

# Migrate a container (usually requires a brief restart unless configured otherwise)
pct migrate <ctid> <target-node> --restart
```

Or use the web UI: right-click the guest → Migrate.

### Graceful shutdown via the web interface

1. Select the node in the left tree.
2. Click **Shutdown** in the top-right node menu.
3. Confirm.

### Graceful shutdown from the CLI

```bash
# SSH to the node, then:
shutdown -h now
# or
systemctl poweroff
```

### Bringing the node back

When the node powers back on and rejoins, reset expected votes if you changed them:

```bash
pvecm expected 2
corosync-quorumtool -s
```

## Important Considerations

- **Quorum**: a plain two-node cluster loses quorum when one node is offline — plan for `two_node` mode or a QDevice.
- **High Availability**: HA-managed guests only run where there is quorum; without it, HA will not start or recover services, and may fence a node.
- **Shared storage**: ensure any shared storage (Ceph, NFS, iSCSI) stays reachable from the surviving node, or guests will not start.
- **Split-brain**: `two_node` mode and forced expected votes reduce protection against two nodes acting independently. A QDevice is the safest way to tolerate a node loss.
- **Config edits**: always edit `/etc/pve/corosync.conf`, increment `config_version`, and keep a backup before reloading corosync.

## Recommended Approach

- **Node offline regularly (by design)** → enable `two_node: 1` mode.
- **Production, want automatic single-node survival with split-brain protection** → add a QDevice.
- **One-off recovery when a node died unexpectedly** → `pvecm expected 1`, then reset to `2` on recovery.
