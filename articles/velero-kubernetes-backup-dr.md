# Velero: Kubernetes Backup and Disaster Recovery

`etcd` holds your cluster's desired state, but it isn't a backup strategy. A fat-fingered
`kubectl delete namespace`, a bad Helm upgrade, a corrupted PVC, or a whole cluster lost to a
region outage all need something that captures **both** the Kubernetes objects and the data
behind them, and can put them back — into the same cluster or a fresh one. That's what
[Velero](https://velero.io) does.

Velero (formerly Heptio Ark) backs up Kubernetes API objects and, optionally, persistent
volume data to object storage (S3, GCS, Azure Blob, or any S3-compatible bucket). From there
you can restore a single namespace, migrate workloads between clusters, or rebuild a cluster
after a disaster. This article covers how it works, how to install it, and the backup/restore
workflows that matter for real DR.

> Commands and flags here target **Velero v1.16**. Check `velero <command> --help` for your
> version, since flags evolve between releases.

---

## How Velero works

Velero runs as a Deployment (the server) in the `velero` namespace, plus an optional
**node-agent** DaemonSet when you need to move volume *data*. Everything is driven by custom
resources — `Backup`, `Restore`, `Schedule`, `BackupStorageLocation` — that you create with the
`velero` CLI or plain YAML.

A backup has two halves:

1. **Kubernetes objects.** Velero lists the API resources you selected (by namespace, label, or
   resource type), serializes them, and uploads a tarball to the backup storage location.
2. **Volume data (optional).** For PersistentVolumes, Velero can capture the data three ways:

| Method | How it works | When to use |
|--------|--------------|-------------|
| **CSI snapshots** | Storage-layer snapshot via the CSI driver, kept on the storage side | Fast, same-cluster restores when your CSI driver supports snapshots and you want durable snapshots on the storage platform |
| **CSI Snapshot Data Movement** | Takes a CSI snapshot, then a data mover copies the data to object storage and releases the snapshot | Cross-cluster/cross-cloud DR, on-prem storage without durable snapshots, long-term retention |
| **File System Backup (FSB)** | Node-agent reads files directly from the live volume (Kopia) | Volumes with no CSI snapshot support — EFS, NFS, `emptyDir`, `local`, etc. |

Prefer CSI-based methods over FSB when you can: FSB reads from the live PV, so the data isn't
captured at a single point in time and is less consistent.

### Selecting the volume method

**CSI snapshots** need a `VolumeSnapshotClass` that Velero is allowed to use — mark it with the
`velero.io/csi-volumesnapshot-class: "true"` label so Velero picks it up for the matching CSI
driver:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: velero-snapshot-class
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: ebs.csi.aws.com        # your CSI driver
deletionPolicy: Retain
```

**File System Backup** is opt-in per pod unless you installed with
`--default-volumes-to-fs-backup`. Annotate the pod with the volume names to back up (a
comma-separated list matching the pod's `volumes[].name`):

```yaml
metadata:
  annotations:
    backup.velero.io/backup-volumes: data,config
```

A backup that finishes with warnings about PVCs usually means neither method was set up: no
labeled VolumeSnapshotClass / snapshot location, and no FSB annotation on the pod.

## Installing Velero

Velero needs a plugin for your object-storage provider and credentials for the bucket. The CLI
`velero install` generates and applies all the in-cluster resources. An AWS S3 example with the
node-agent and CSI support enabled:

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.0 \
  --bucket velero-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero \
  --use-node-agent \
  --features=EnableCSI
```

Key install flags:

- `--use-node-agent` — installs the node-agent DaemonSet, **required** for File System Backup
  and for CSI Snapshot Data Movement with the built-in data mover.
- `--features=EnableCSI` — turns on CSI snapshot integration. From Velero 1.14+ the CSI plugin
  is built into Velero, so you no longer install a separate CSI plugin.
- `--privileged-node-agent` — run the node-agent privileged (needed to back up block volumes).
- `--default-volumes-to-fs-backup` — make FSB the default for all pod volumes instead of
  annotating each pod.
- `--no-secret` — skip the credentials file when using IRSA / Workload Identity / kube2iam
  instead of static keys.

If you use IAM Roles for Service Accounts on EKS (recommended over static keys), attach the
role to the Velero service account and install with `--no-secret`. Prefer pinning the plugin
image to an explicit version rather than `latest`.

To review the manifests before applying (useful for GitOps), add `--dry-run -o yaml`.

## Taking backups

On-demand backup of one or more namespaces:

```bash
# Whole namespace, Kubernetes objects only
velero backup create app-backup --include-namespaces myapp

# Include volume data via the built-in data mover (snapshot → object storage)
velero backup create app-backup --include-namespaces myapp --snapshot-move-data

# Filter by label, and speed up file uploads with parallelism
velero backup create app-backup \
  --include-namespaces myapp \
  --selector app=web \
  --snapshot-move-data \
  --parallel-files-upload 4 \
  --wait

# Full-cluster backup, skipping noise that controllers recreate anyway
velero backup create cluster-backup \
  --exclude-namespaces kube-system,velero \
  --exclude-resources events,events.events.k8s.io,pods
```

Useful selectors and scoping flags:

- `--include-namespaces` / `--exclude-namespaces`
- `--include-resources` / `--exclude-resources` (e.g. `--include-resources deployments,services`)
- `--selector app=web` — label selector
- `--include-cluster-resources=true` — include cluster-scoped objects (ClusterRoles, PVs, etc.)
- `--snapshot-move-data` — move CSI snapshot data to object storage (built-in data mover)
- `--wait` — block until the backup finishes

You can also exclude an individual object regardless of selectors by labeling it:

```bash
kubectl label -n myapp secret/dev-only velero.io/exclude-from-backup=true
```

Inspect what happened:

```bash
velero backup get
velero backup describe app-backup --details
velero backup logs app-backup

# When using data movement, watch the DataUpload CRs
kubectl -n velero get datauploads -l velero.io/backup-name=app-backup -w
```

## Scheduling backups

DR needs *automated, recurring* backups, not one-offs. A `Schedule` runs `velero backup create`
on a cron and applies a TTL for retention:

```bash
# Every day at 3am, keep backups for 30 days, move volume data to object storage
velero schedule create daily-app \
  --schedule="0 3 * * *" \
  --include-namespaces myapp \
  --snapshot-move-data \
  --ttl 720h0m0s
```

Notes that matter for scheduling:

- Backups from a schedule are named `<SCHEDULE-NAME>-<YYYYMMDDhhmmss>`.
- `--ttl` sets retention (e.g. `720h` = 30 days); Velero garbage-collects expired backups.
- Pin the schedule to a timezone with `CRON_TZ` to avoid daylight-saving surprises:
  `--schedule="CRON_TZ=America/New_York 0 3 * * *"`.
- Trigger a scheduled template immediately with
  `velero backup create --from-schedule daily-app`.

```bash
velero schedule get
velero schedule describe daily-app
```

## Restoring

Restore is where DR is actually tested. Restoring re-creates objects (and data, if the backup
captured it) — by default only into namespaces that don't already have the objects.

```bash
# Restore an entire backup
velero restore create --from-backup app-backup

# Restore only specific namespaces from a larger backup
velero restore create --from-backup nightly-full --include-namespaces myapp

# Remap to a different namespace (great for testing restores safely)
velero restore create --from-backup app-backup \
  --namespace-mappings myapp:myapp-restore-test

# Exclude resource types from the restore
velero restore create --from-backup app-backup \
  --exclude-resources persistentvolumeclaims
```

By default a restore is **non-destructive**: if an object already exists in the target cluster,
Velero skips it (you'll see "already exists"–style skips). To make Velero reconcile existing
objects toward the backup instead, set the existing-resource policy:

```bash
velero restore create --from-backup app-backup --existing-resource-policy=update
```

`--existing-resource-policy` takes `none` (default, skip existing) or `update` (best-effort
patch of the existing resource to match the backup, tagging it with `velero.io/backup-name` and
`velero.io/restore-name` labels). Two caveats: it's best-effort — a failed update falls back to
the non-destructive behavior with a warning rather than failing the restore — and it only
updates the resource's spec, **not** PV data. Restoring the *contents* of a PVC still goes
through snapshots/data movement, not this flag.

For data-movement backups you don't specify anything extra on restore — Velero reads from the
backup whether data movement was used and which data mover to use.

```bash
velero restore get
velero restore describe <restore-name> --details
velero restore logs <restore-name>

# Watch data download progress
kubectl -n velero get datadownloads -l velero.io/restore-name=<restore-name> -w
```

**Cross-cluster / cross-cloud DR:** point a fresh cluster's Velero at the *same* backup storage
location, let it sync the backup metadata, then `velero restore create --from-backup`. For data
movement to restore volumes, create a StorageClass in the target cluster with the **same name**
as the source so PVCs provision correctly; otherwise remap the storage class during restore.

## Application-consistent backups with hooks

A snapshot taken mid-write can be crash-consistent but not application-consistent. Backup hooks
run commands in a pod's container before and after the backup so you can quiesce a database —
flush and lock, snapshot, then unlock. Annotate the pod:

```yaml
metadata:
  annotations:
    pre.hook.backup.velero.io/container: postgres
    pre.hook.backup.velero.io/command: '["/bin/sh","-c","pg_dump -U postgres mydb > /backup/dump.sql"]'
    post.hook.backup.velero.io/container: postgres
    post.hook.backup.velero.io/command: '["/bin/sh","-c","rm -f /backup/dump.sql"]'
```

Always validate the restore of a database backup end-to-end — a backup you've never restored is
a hope, not a plan.

## Deleting backups and storage lifecycle

There's a sharp distinction between deleting the *backup resource* and deleting the *data*:

```bash
# Deletes the Backup CR AND the data in object/block storage
velero backup delete app-backup

# Deletes ONLY the CR; data stays in the bucket (and Velero will re-sync the CR from it)
kubectl delete backup app-backup -n velero
```

Two lifecycle gotchas:

- For data-movement/FSB backups, deleting a backup removes the repository *snapshot reference*
  immediately, but the underlying data is reclaimed only when a **repository maintenance job**
  runs. Storage usage won't drop until then — make sure those maintenance jobs complete.
- Object-lock / immutability on the bucket can break Velero, because it rewrites backup metadata
  when moving a backup from `Finalizing` to `Complete`. On AWS S3 backups still work (new object
  versions), but deletions leave old versions behind.

## Troubleshooting

```bash
# Are the server and node-agent pods running?
kubectl get pods -n velero

# Is the backup repository healthy?
velero repo get

# Is the backup storage location reachable? (PHASE should be Available)
velero backup-location get
velero snapshot-location get

# What failed?
velero backup describe <name> --details
velero backup logs <name>
velero restore describe <name> --details
velero restore logs <name>

# Server / node-agent logs (bump verbosity with --log-level=debug on the container)
kubectl -n velero logs deploy/velero
```

Common issues:

- **Backup stuck `InProgress` / `Finalizing`** — usually an async volume operation; check the
  `DataUpload` CRs and node-agent logs.
- **PVs not backed up** — no CSI snapshot support and FSB not enabled; add `--use-node-agent`
  and either annotate the pods or install with `--default-volumes-to-fs-backup`.
- **Restore skips objects / "already exists"** — restore is non-destructive by default; restore
  into a mapped namespace to test cleanly, or use `--existing-resource-policy=update` to
  reconcile existing objects toward the backup.
- **`PartiallyFailed` backups** — read `velero backup logs`; often a single unsupported resource
  or a hook command that returned non-zero.

## A practical DR baseline

1. **Install** with the node-agent and CSI enabled; use IRSA/Workload Identity over static keys.
2. **Schedule** at least a daily namespace or full-cluster backup with a sane `--ttl`, using
   `--snapshot-move-data` so volume data lands in object storage (survives cluster loss).
3. **Add hooks** for stateful workloads that need application consistency.
4. **Test restores regularly** into a mapped namespace or a scratch cluster — the restore is the
   only part of DR that counts.
5. **Store backups off-cluster and ideally cross-region**, and verify repository maintenance
   jobs actually run so storage doesn't grow unbounded.

## Summary

Velero turns Kubernetes backup and DR into a few declarative resources: install the server (plus
node-agent for volume data), schedule recurring backups that move volume data to object storage,
add hooks where you need application consistency, and — most importantly — rehearse restores.
`etcd` snapshots protect the control plane; Velero protects your workloads and their data, and
lets you rebuild a cluster or migrate across clouds from a bucket.

---

### Sources

- [Velero documentation (v1.16)](https://velero.io/docs/v1.16/)
- [Customize Velero Install](https://velero.io/docs/v1.16/customize-installation/)
- [Backup Reference](https://velero.io/docs/v1.16/backup-reference/)
- [CSI Snapshot Data Movement](https://velero.io/docs/v1.16/csi-snapshot-data-movement/)
- [Backup Hooks](https://velero.io/docs/v1.16/backup-hooks/)
