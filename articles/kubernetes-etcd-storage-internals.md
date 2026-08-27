# How etcd Stores and Retrieves Kubernetes Objects

How Kubernetes persists cluster state in etcd — key structure, protobuf encoding, the watch mechanism, compaction, defragmentation, and backup/restore.

## High-Level Architecture

```
┌───────────────┐                    ┌───────────────────────────────┐
│   API Server  │──── gRPC ─────────▶│           etcd                │
│               │                    │                               │
│  Serializes   │                    │  Key-Value Store              │
│  objects to   │                    │  (BoltDB / bbolt)             │
│  protobuf     │                    │                               │
│               │                    │  Raft consensus for HA        │
│  Reads objects│◀─── gRPC ───────── │  (leader + followers)         │
│  from etcd    │                    │                               │
└───────────────┘                    └───────────────────────────────┘
```

Only the API server talks to etcd directly. No other component (kubelet, scheduler, controllers) reads from or writes to etcd.

## Key Structure

Every Kubernetes object is stored under a key path in etcd's `/registry` prefix:

```
/registry/{resource}/{namespace}/{name}
/registry/{resource}/{name}                 (cluster-scoped)
```

### Examples

```
/registry/pods/default/nginx
/registry/pods/production/web-abc123
/registry/deployments/default/my-app
/registry/services/specs/default/kubernetes
/registry/services/endpoints/default/kubernetes
/registry/secrets/kube-system/coredns-token
/registry/configmaps/kube-system/coredns
/registry/namespaces/production
/registry/nodes/ip-10-0-1-50.ec2.internal
/registry/clusterroles/cluster-admin
/registry/leases/kube-node-lease/node-1
/registry/events/default/my-pod.17abc123
```

### Key Patterns by Resource Type

| Resource | Key Pattern |
|----------|-------------|
| Pods | `/registry/pods/{namespace}/{name}` |
| Deployments | `/registry/deployments/{namespace}/{name}` |
| Services | `/registry/services/specs/{namespace}/{name}` |
| Endpoints | `/registry/services/endpoints/{namespace}/{name}` |
| Secrets | `/registry/secrets/{namespace}/{name}` |
| ConfigMaps | `/registry/configmaps/{namespace}/{name}` |
| Nodes | `/registry/minions/{name}` (legacy path) |
| Namespaces | `/registry/namespaces/{name}` |
| PVs | `/registry/persistentvolumes/{name}` |
| PVCs | `/registry/persistentvolumeclaims/{namespace}/{name}` |
| CRDs | `/registry/apiextensions.k8s.io/customresourcedefinitions/{name}` |
| Custom Resources | `/registry/{group}/{resource}/{namespace}/{name}` |

```bash
# List all keys (requires direct etcd access):
etcdctl get /registry --prefix --keys-only | head -50

# Get a specific object:
etcdctl get /registry/pods/default/nginx

# Count objects by type:
etcdctl get /registry/pods --prefix --keys-only | wc -l
```

## Value Encoding — Protobuf

Objects are stored as **protobuf** (not JSON/YAML). This is more compact and faster to serialize/deserialize:

```
┌──────────────────────────────────────────────────────────────┐
│  Storage format:                                             │
│                                                              │
│  [magic prefix][protobuf-encoded object]                     │
│                                                              │
│  Magic prefix: "k8s\x00" (4 bytes)                           │
│  Then: protobuf binary of the internal API version           │
│                                                              │
│  The API server handles all version conversion:              │
│  - External: v1, apps/v1, batch/v1 (what you see in YAML)    │
│  - Internal: __internal (used in code)                       │
│  - Storage: protobuf of the preferred storage version        │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Raw etcd value is binary (not human-readable):
etcdctl get /registry/pods/default/nginx -w fields
# Value is protobuf bytes

# Decode with auger (etcd object decoder):
etcdctl get /registry/pods/default/nginx | auger decode
# Shows decoded YAML/JSON
```

### Encryption at Rest

By default, etcd stores objects unencrypted. You can enable encryption:

```yaml
# EncryptionConfiguration (API server flag --encryption-provider-config):
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  - configmaps
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}     # Fallback: no encryption (for reading old data)
```

With encryption, the storage path becomes:
```
[magic prefix][encryption envelope][encrypted protobuf]
```

## resourceVersion — etcd Revision

Every object has a `metadata.resourceVersion`. This is actually the **etcd revision** at which the object was last modified:

```
┌────────────────────────────────────────────────────────────────┐
│  resourceVersion = etcd modification revision                  │
│                                                                │
│  etcd revision is a cluster-wide monotonic counter:            │
│  - Every write (put, delete) increments the global revision    │
│  - Each key also tracks its own create/mod revision            │
│                                                                │
│  Example:                                                      │
│    etcd revision 1000: Pod A created                           │
│    etcd revision 1001: Pod B created                           │
│    etcd revision 1002: Pod A updated (label added)             │
│    etcd revision 1003: Secret X created                        │
│                                                                │
│  Pod A's resourceVersion = "1002" (last mod)                   │
│  Pod B's resourceVersion = "1001" (last mod)                   │
│  Secret X's resourceVersion = "1003"                           │
│                                                                │
│  LIST pods returns resourceVersion = "1003" (latest revision)  │
│  WATCH from rv=1001 returns events after that revision         │
└────────────────────────────────────────────────────────────────┘
```

### Optimistic Concurrency (Compare-and-Swap)

When updating an object, the API server uses the resourceVersion for conflict detection:

```
1. Client reads pod (rv=1002)
2. Client sends update with rv=1002
3. API server checks: is current rv still 1002?
   YES → update succeeds, new rv=1005
   NO  → 409 Conflict: "the object has been modified"
```

```bash
# See resourceVersion:
kubectl get pod nginx -o jsonpath='{.metadata.resourceVersion}'
```

## Watches — etcd Event Stream

The API server's watch mechanism is backed by etcd's watch:

```
┌───────────────┐                    ┌───────────┐
│   API Server  │── Watch prefix ───▶│   etcd    │
│  watch cache  │   /registry/pods/  │           │
│               │                    │  Returns  │
│               │◀── Event stream ───│  events   │
│               │   (PUT, DELETE)    │  from rev │
└───────────────┘                    └───────────┘
```

etcd watches are on key prefixes:
- Watch all pods: prefix `/registry/pods/`
- Watch pods in namespace: prefix `/registry/pods/default/`
- Watch specific pod: key `/registry/pods/default/nginx`

The API server maintains a **watch cache** (in-memory ring buffer) so that multiple Kubernetes watchers share one etcd watch.

## Compaction

etcd keeps a history of all revisions (for watches to replay). Compaction removes old revisions to save disk space:

```
┌────────────────────────────────────────────────────────────────┐
│  Before compaction (rev 1-1000 stored):                        │
│                                                                │
│  Rev 500: Pod A created                                        │
│  Rev 600: Pod A updated (label change)                         │
│  Rev 700: Pod A updated (image change)                         │
│  Rev 800: Pod A status update                                  │
│                                                                │
│  After compaction at rev 900:                                  │
│                                                                │
│  Revisions 1-900 deleted (can't watch from those anymore)      │
│  Only rev 900+ available for watches                           │
│  Current state of all objects preserved (latest revision)      │
│                                                                │
│  A client trying to WATCH from rev 500 gets: 410 Gone          │
│  Must re-LIST to get current state                             │
└────────────────────────────────────────────────────────────────┘
```

### Auto-Compaction

```bash
# etcd auto-compaction (default for kube clusters):
# --auto-compaction-mode=periodic
# --auto-compaction-retention=5m (keep 5 minutes of history)

# Check current revision and compacted revision:
etcdctl endpoint status --write-out=table
# +----------+--------+---------+---------+
# | ENDPOINT | DB SIZE| RAFT IDX| COMPACTED|
# +----------+--------+---------+---------+
# | :2379    | 100 MB | 1500000 | 1490000 |
# +----------+--------+---------+---------+
```

## Defragmentation

Compaction marks old revisions as free space, but doesn't return it to the OS. Defrag reclaims this space:

```
┌────────────────────────────────────────────────────────────────┐
│  etcd database file (BoltDB):                                  │
│                                                                │
│  Before defrag: 500 MB on disk, 200 MB actual data             │
│  (300 MB is freed space from compacted revisions)              │
│                                                                │
│  After defrag:  200 MB on disk, 200 MB actual data             │
│  (freed space returned to OS)                                  │
└────────────────────────────────────────────────────────────────┘
```

```bash
# Defrag a single member (blocks reads/writes briefly):
etcdctl defrag --endpoints=https://etcd-0:2379

# Defrag all members (do one at a time in production):
etcdctl defrag --endpoints=https://etcd-0:2379,https://etcd-1:2379,https://etcd-2:2379

# Check DB size before/after:
etcdctl endpoint status --write-out=table
```

**Warning**: Defrag blocks the etcd member for the duration. In a 3-node cluster, defrag one at a time.

## Raft Consensus

etcd uses Raft for consistency across members:

```
┌──────────┐     ┌───────────┐     ┌───────────┐
│  Leader  │────▶│ Follower  │     │ Follower  │
│  etcd-0  │────▶│  etcd-1   │     │  etcd-2   │
│          │     │           │     │           │
│ Handles  │     │Replicates │     │Replicates │
│all writes│     │from leader│     │from leader│
│          │     │           │     │           │
└──────────┘     └───────────┘     └───────────┘

Write path:
1. Client sends write to leader
2. Leader appends to its log
3. Leader replicates to followers
4. Majority (2/3) acknowledge → commit
5. Leader responds to client: success
```

### Quorum

| Cluster Size | Quorum | Tolerated Failures |
|:------------:|:------:|:------------------:|
| 1 | 1 | 0 |
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

Production clusters typically use 3 or 5 etcd members.

## Size Limits

| Limit | Default | Configurable |
|-------|---------|:------------:|
| Max DB size | 2 GB (EKS: 8 GB) | Yes (`--quota-backend-bytes`) |
| Max request size | 1.5 MB | Yes (`--max-request-bytes`) |
| Max key size | 1.5 MB | No |
| Max value size | 1.5 MB | No |

```bash
# Check current DB size vs quota:
etcdctl endpoint status --write-out=table

# If DB hits quota → etcd goes read-only (ALARM: NOSPACE)
# Fix:
etcdctl alarm list
etcdctl compact <revision>
etcdctl defrag
etcdctl alarm disarm
```

### What Uses the Most Space

| Object Type | Typical Size | Notes |
|------------|-------------|-------|
| Events | Small but very numerous | Auto-expire (TTL 1h default) |
| Secrets/ConfigMaps | Can be up to 1 MB each | Large configs are problematic |
| CRDs with status | Varies | Frequent status updates create many revisions |
| Endpoints (legacy) | Can be huge for large services | Use EndpointSlices instead |

## Backup and Restore

### Snapshot Backup

```bash
# Take a snapshot:
etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot:
etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | abc12345 | 1500000  |    50000   |  100 MB    |
# +----------+----------+------------+------------+
```

### Restore from Snapshot

```bash
# Stop kube-apiserver and etcd

# Restore (creates a new data directory):
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --name=etcd-0 \
  --initial-cluster=etcd-0=https://10.0.0.1:2380 \
  --initial-advertise-peer-urls=https://10.0.0.1:2380

# Point etcd to the new data directory and start
# Start kube-apiserver
```

**Critical**: After restore, all objects have the state from snapshot time. Any changes made between snapshot and restore are lost.

## API Server ↔ etcd Interaction

```
┌────────────────────────────────────────────────────────────────────┐
│  API Server etcd Operations                                        │
│                                                                    │
│  CREATE (kubectl apply new object):                                │
│    → etcd txn: if key not exists → put(key, protobuf)              │
│    → Returns: created object with new resourceVersion              │
│                                                                    │
│  GET (kubectl get pod nginx):                                      │
│    → etcd get: key = /registry/pods/default/nginx                  │
│    → Returns: protobuf → decode → version convert → JSON/YAML      │
│                                                                    │
│  LIST (kubectl get pods):                                          │
│    → etcd get: prefix = /registry/pods/default/ (with limit/cont)  │
│    → Returns: list of protobuf objects                             │
│    → API server paginates (default 500 per chunk)                  │
│                                                                    │
│  UPDATE (kubectl apply changed object):                            │
│    → etcd txn: if mod_revision == expected → put(key, new protobuf)│
│    → Compare-and-swap: fails if someone else modified (409)        │
│                                                                    │
│  DELETE (kubectl delete pod nginx):                                │
│    → etcd delete: key = /registry/pods/default/nginx               │
│    → (with tombstone if using Lease-based TTL)                     │
│                                                                    │
│  WATCH (controller watching pods):                                 │
│    → etcd watch: prefix = /registry/pods/ from revision X          │
│    → Streams PUT/DELETE events                                     │
└────────────────────────────────────────────────────────────────────┘
```

## Monitoring etcd

```bash
# Key metrics to watch:
# etcd_server_has_leader (should be 1)
# etcd_disk_wal_fsync_duration_seconds (disk latency)
# etcd_disk_backend_commit_duration_seconds
# etcd_server_proposals_failed_total (raft failures)
# etcd_network_peer_round_trip_time_seconds (inter-member latency)
# etcd_debugging_mvcc_db_total_size_in_bytes (DB size)

# Check etcd health:
etcdctl endpoint health --write-out=table

# Check leader:
etcdctl endpoint status --write-out=table

# Check member list:
etcdctl member list --write-out=table

# Performance benchmark:
etcdctl check perf --load="s"  # small load test
```

## Quick Reference

```bash
# Key structure: /registry/{resource}/{namespace}/{name}
# Encoding: protobuf (not JSON)
# Only API server talks to etcd

# resourceVersion = etcd revision (global monotonic counter)
# Used for: optimistic concurrency, watch resumption

# Compaction: removes old revisions (frees logical space)
# Defrag: reclaims physical disk space (blocks briefly)

# Quorum: 2/3 for writes (3-member cluster tolerates 1 failure)

# Limits:
# Default DB quota: 2 GB (NOSPACE alarm if exceeded)
# Max object size: 1.5 MB

# Backup:
etcdctl snapshot save /backup/snapshot.db
etcdctl snapshot status /backup/snapshot.db --write-out=table

# Health:
etcdctl endpoint health
etcdctl endpoint status --write-out=table
etcdctl alarm list

# Direct inspection (requires certs):
etcdctl get /registry --prefix --keys-only | head
etcdctl get /registry/pods/default/nginx
```
