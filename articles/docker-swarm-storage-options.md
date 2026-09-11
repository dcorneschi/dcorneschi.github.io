# Shared Storage Options for Docker Swarm

By default a Docker volume is local to the node that created it, so a task rescheduled to another node loses its data. Stateful workloads on a swarm therefore need **shared or distributed storage** reachable from any node. This guide compares the main options, their trade-offs, and which to choose — it is a decision guide rather than a setup manual.

For hands-on NFS, GlusterFS, and Ceph configuration with stack examples, see [Docker Swarm Storage](articles/docker-swarm-storage.md). For the local bind-mount vs named-volume distinction, see [Docker Compose: Bind Mounts vs Named Volumes](articles/docker-compose-volumes-vs-bind-mounts.md).

## Why Local Volumes Aren't Enough

A swarm scheduler can move a task to any eligible node — on failure, drain, or update. A local volume stays on the original node, so the rescheduled task starts with empty (or missing) data. Options to deal with this:

1. **Pin the task** to one node with a placement constraint and use local storage — simple, but no HA and the data is stranded if that node dies.
2. **Use shared/distributed storage** every node can reach — the task can run anywhere and see the same data.

Most of this guide is about option 2.

## Network File System Solutions

### NFS

The most common and straightforward shared-storage choice. Use an existing NFS server (a NAS, TrueNAS, or a Linux box) and Docker's built-in `local` driver with NFS options.

- **Pros:** simple, well understood, works with the built-in driver, good enough for most homelab and small-production stacks.
- **Cons:** the NFS server is a single point of failure unless you build an HA NFS pair; network latency and a single server can bottleneck heavy workloads.

Best for straightforward setups and shared config/data where extreme performance is not required.

### SSHFS (vieux/sshfs plugin)

Mounts a remote directory over SSH via the `vieux/sshfs` volume plugin. Easy if you already have SSH access to a host.

- **Pros:** trivial to set up on existing SSH infrastructure.
- **Cons:** noticeably slower than NFS; SSH overhead makes it unsuitable for databases or high I/O. The `vieux/sshfs` plugin is old and effectively unmaintained, so treat it as a convenience for light, non-critical use rather than a production backbone.

## Distributed Storage Systems

### GlusterFS

An open-source scale-out filesystem that aggregates disks from multiple servers with replication for HA.

- **Pros:** replication and high availability, pools storage across nodes.
- **Cons:** operational complexity, and — importantly — **GlusterFS is end-of-life**. Upstream development wound down and it no longer receives active maintenance, so do not choose it for new deployments. Migrate existing GlusterFS setups to NFS (for simplicity) or Ceph (for distributed HA).

### Ceph / CephFS

A highly scalable distributed storage system providing object, block (RBD), and file (CephFS) storage with automatic distribution and replication.

- **Pros:** robust, self-healing, scalable, supports snapshots and fine-grained access; strong choice for serious distributed HA.
- **Cons:** the most complex option to deploy and operate; resource-hungry and needs enough nodes to be resilient. Usually run as a separate cluster that swarm nodes mount.

Best when you need genuine distributed high availability and are prepared for the operational investment.

## Commercial and Cloud-Backed Options

### Portworx

An enterprise container-storage platform that can run as a swarm global service, offering replicated block storage at near-local speeds on converged compute/storage nodes.

- **Pros:** performance plus replication, enterprise support, rich features.
- **Cons:** commercial licensing; heavier to adopt. Note its primary focus has shifted toward Kubernetes, so verify current Swarm support before committing.

### Cloud provider volumes (EBS / Azure Disk / GCP PD)

On a cloud swarm, the cleanest option is often the provider's own block storage attached to nodes. Historically the **Rex-Ray** / libStorage plugin bridged Docker to these backends, but Rex-Ray is **archived and no longer maintained** — do not build new systems on it. Prefer the cloud provider's current CSI/volume tooling, or reconsider whether a managed Kubernetes offering fits better for cloud-native block storage.

### Longhorn

A distributed block-storage system that is **designed for Kubernetes**, not Swarm. It is listed here only to set expectations: it is not a native Swarm solution, and adapting it is not a supported path. If Longhorn appeals, it is a signal that Kubernetes may suit the workload better.

## Comparison

| Solution | Model | HA | Complexity | Status / Note | Good for |
|----------|-------|----|------------|--------------|----------|
| Local + pin | Node-local | No | Lowest | Built in | Single-node stateful, dev |
| NFS | Shared file | With HA NFS only | Low | Widely used | Most small/medium stacks |
| SSHFS | Shared file over SSH | No | Low | Plugin unmaintained | Light, non-critical use |
| GlusterFS | Distributed file | Yes | High | End-of-life — avoid new use | Legacy only |
| Ceph / CephFS | Distributed object/block/file | Yes | Highest | Actively maintained | Serious distributed HA |
| Portworx | Commercial block | Yes | Medium–High | Commercial; K8s-focused | Enterprise needs |
| Cloud block (EBS/etc.) | Provider block | Provider-managed | Medium | Use current CSI tooling | Cloud swarms |

General ordering to keep in mind:

- **Performance:** local > NFS > distributed filesystems (network and replication add overhead).
- **Availability:** distributed (Ceph) > NFS with HA > single NFS server > local.
- **Complexity:** local < NFS < SSHFS < Ceph/Portworx.
- **Cost:** open source < commercial; but factor in the operational cost of running complex systems yourself.

## Choosing a Solution

- **Simple needs / homelab / small production:** start with **NFS**. It is well supported, uses the built-in driver, and is easy to reason about. Add HA NFS if the single server becomes a risk.
- **Serious distributed high availability:** use **Ceph**, accepting the setup and operational overhead, typically as a dedicated cluster the swarm mounts.
- **Enterprise with budget and support needs:** evaluate **Portworx** (confirming current Swarm support).
- **Cloud:** use the provider's current block-storage/CSI integration rather than legacy plugins.
- **Avoid for new work:** **GlusterFS** (EOL), **Rex-Ray** (archived), and **SSHFS** for anything performance- or reliability-sensitive.

If your requirements keep pointing at Kubernetes-first tools (Longhorn, modern Portworx, CSI drivers), that is a hint that Kubernetes may be the better platform for the workload — see [Docker Compose vs Docker Swarm](articles/docker-compose-vs-swarm.md) for that trade-off.

## Minimal Examples

These illustrate how each attaches; see [Docker Swarm Storage](articles/docker-swarm-storage.md) for full server setup, permissions, and stack examples.

### NFS-backed volume

```bash
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/srv/nfs/swarm/appdata \
  app_data

docker service create \
  --name app \
  --mount source=app_data,target=/app/data \
  nginx:1.27-alpine
```

Or declaratively in a stack:

```yaml
services:
  web:
    image: nginx:1.27-alpine
    volumes:
      - app_data:/usr/share/nginx/html
    ports:
      - "80:80"

volumes:
  app_data:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=192.168.1.100,rw"
      device: ":/srv/nfs/swarm/appdata"
```

### SSHFS plugin (light use only)

```bash
docker plugin install --grant-all-permissions vieux/sshfs

docker volume create -d vieux/sshfs \
  -o sshcmd=user@host:/remote/path \
  -o password=... \
  sshfs_data
```

## Key Considerations

- **Match storage to the workload.** Databases want low-latency, consistent storage (local-pinned, Ceph RBD, or cloud block); shared web assets tolerate NFS well.
- **Plan for node loss.** Local storage plus a pin is simple but strands data if the node dies; distributed storage survives node failure at the cost of complexity.
- **Watch permissions.** Shared volumes surface UID/GID mismatches quickly; pre-own directories to the service's user.
- **Back up regardless of the layer.** Replication is not backup — dump databases and archive volumes on a schedule.
- **Prefer maintained tools.** Steer clear of EOL/archived projects (GlusterFS, Rex-Ray) for anything new.

For setup procedures, backups, and troubleshooting, continue to [Docker Swarm Storage](articles/docker-swarm-storage.md). See also the [Docker Swarm Cheatsheet](articles/docker-swarm-cheatsheet.md) and [Docker Compose vs Docker Swarm](articles/docker-compose-vs-swarm.md).
