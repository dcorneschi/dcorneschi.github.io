# Docker Overlay2 Storage Driver

How Docker uses the overlay2 filesystem to layer images and containers — LowerDir, UpperDir, WorkDir, MergedDir, and practical inspection commands.

## How Overlay2 Works

Overlay2 is Docker's default storage driver. It creates a union filesystem by stacking read-only image layers with a read-write container layer:

```
┌─────────────────────────────────────────┐
│           Container Process             │
│         (sees MergedDir as /)           │
├─────────────────────────────────────────┤
│  MergedDir (unified view)               │
│  ┌───────────────────────────────────┐  │
│  │  UpperDir (container layer, r/w)  │  │
│  ├───────────────────────────────────┤  │
│  │  LowerDir (image layers, r/o)     │  │
│  │    layer N (topmost)              │  │
│  │    layer N-1                      │  │
│  │    ...                            │  │
│  │    layer 1 (base)                 │  │
│  └───────────────────────────────────┘  │
│  WorkDir (overlay internal use)         │
└─────────────────────────────────────────┘
```

## Directory Roles

| Directory | Read/Write | Purpose |
|-----------|:----------:|---------|
| LowerDir | Read-only | Image layers stacked in order — the base filesystem |
| UpperDir | Read-write | Container-specific changes (new/modified/deleted files) |
| WorkDir | Internal | Required empty directory for overlay atomic operations |
| MergedDir | Unified view | Combined view of all layers — what the container sees as `/` |

### LowerDir (Image Layers)

- Read-only layers from the Docker image
- Each image layer is a separate directory
- Assembled in order (base layer first, topmost layer last)
- Shared across all containers using the same image
- Never modified by running containers

### UpperDir (Container Layer)

- Read-write layer unique to each container
- All changes made inside the container are written here:
  - New files created
  - Existing files modified (copy-on-write)
  - Deleted files (stored as whiteout files)
- This is what makes each container's filesystem unique
- Lost when the container is removed (unless using volumes)

### WorkDir

- Required by the Linux overlay filesystem
- Must be an empty directory on the same filesystem as UpperDir
- Used internally for atomic operations (e.g., rename)
- Should never be manually modified

### MergedDir

- The unified view of LowerDir + UpperDir
- Docker effectively `chroot`s into this directory when running the container
- What processes inside the container see as their root filesystem (`/`)
- Files from UpperDir override files in LowerDir with the same path

## Inspecting Overlay2 Layers

### Find a Container's Layers

```bash
# Get container's mount info
docker inspect <container> --format '{{.GraphDriver.Data.MergedDir}}'
docker inspect <container> --format '{{.GraphDriver.Data.UpperDir}}'
docker inspect <container> --format '{{.GraphDriver.Data.LowerDir}}'
docker inspect <container> --format '{{.GraphDriver.Data.WorkDir}}'

# Full overlay mount info
docker inspect <container> --format '{{json .GraphDriver.Data}}' | jq .

# See the actual overlay mount
mount | grep overlay
```

### Example Output

```bash
$ docker inspect mycontainer --format '{{json .GraphDriver.Data}}' | jq .
{
  "LowerDir": "/var/lib/docker/overlay2/abc123-init/diff:/var/lib/docker/overlay2/def456/diff:/var/lib/docker/overlay2/ghi789/diff",
  "MergedDir": "/var/lib/docker/overlay2/abc123/merged",
  "UpperDir": "/var/lib/docker/overlay2/abc123/diff",
  "WorkDir": "/var/lib/docker/overlay2/abc123/work"
}
```

### Browse the Layers

```bash
# See what's in the container's writable layer (changes only)
ls /var/lib/docker/overlay2/<container-layer-id>/diff/

# See the merged view (what the container sees)
ls /var/lib/docker/overlay2/<container-layer-id>/merged/

# See the image layers
docker inspect <image> --format '{{json .RootFS.Layers}}' | jq .
```

### Check Storage Driver

```bash
# Confirm overlay2 is in use
docker info | grep "Storage Driver"
# Storage Driver: overlay2

# Show backing filesystem
docker info | grep "Backing Filesystem"
# Backing Filesystem: xfs (or ext4)
```

## Copy-on-Write (CoW)

When a container modifies a file from a lower (image) layer:

1. The file is copied from LowerDir to UpperDir (copy-up)
2. The modification is applied to the copy in UpperDir
3. The container sees the modified version (UpperDir takes priority)
4. The original in LowerDir is untouched

```bash
# See what a container has changed (files in UpperDir)
docker diff <container>
# A = Added
# C = Changed
# D = Deleted

# Example output:
# C /etc
# A /etc/nginx/custom.conf
# C /var/log/nginx/access.log
```

### Whiteout Files

When a container deletes a file from a lower layer, a "whiteout" file is created in UpperDir:

```bash
# Whiteout files are character devices with 0/0 major/minor
# They hide the corresponding file in LowerDir
ls -la /var/lib/docker/overlay2/<id>/diff/
# .wh.deleted-file = whiteout for "deleted-file"
```

## Map Containers and Images to Overlay2 Hashes

Script to show which overlay2 directory belongs to which container/image:

```bash
echo "=== CONTAINERS ==="
echo "NAME                 IMAGE                     STATUS                         OVERLAY_HASH"
echo "==================== ========================= ============================== ================================"
docker ps -aq | while read container; do
  name=$(docker inspect "$container" --format "{{.Name}}" 2>/dev/null | sed "s/^\///")
  image=$(docker inspect "$container" --format "{{.Config.Image}}" 2>/dev/null)
  status=$(docker ps -a --format "{{.Status}}" --filter "id=$container" 2>/dev/null)
  merged=$(docker inspect "$container" --format "{{.GraphDriver.Data.MergedDir}}" 2>/dev/null)
  overlay_hash=$(basename "$(dirname "$merged")" 2>/dev/null | cut -c1-32)
  printf "%-20s %-25s %-30s %-32s\n" "${name:0:18}" "${image:0:23}" "${status:0:28}" "$overlay_hash"
done

echo -e "\n=== IMAGES ==="
echo "REPOSITORY:TAG                  SIZE       OVERLAY_HASH"
echo "=============================== ========== ================================"
docker images -q | while read image; do
  tags=$(docker inspect "$image" --format "{{if .RepoTags}}{{index .RepoTags 0}}{{else}}<none>{{end}}" 2>/dev/null)
  size=$(docker inspect "$image" --format "{{.Size}}" 2>/dev/null)
  overlays=$(docker inspect "$image" --format "{{.GraphDriver.Data.UpperDir}}" 2>/dev/null)
  overlay_hash=$(basename "$(dirname "$overlays")" 2>/dev/null | cut -c1-32)
  human_size="$size"
  [ "$size" -gt 1000000000 ] && human_size="$((size/1000000000))GB"
  [ "$size" -gt 1000000 ] && [ "$size" -lt 1000000000 ] && human_size="$((size/1000000))MB"
  printf "%-31s %-10s %-32s\n" "${tags:0:29}" "$human_size" "$overlay_hash"
done

echo -e "\n=== VOLUMES ==="
echo "NAME                 DRIVER     MOUNTPOINT"
echo "==================== ========== =================================="
docker volume ls -q | while read volume; do
  driver=$(docker inspect "$volume" --format "{{.Driver}}" 2>/dev/null)
  mountpoint=$(docker inspect "$volume" --format "{{.Mountpoint}}" 2>/dev/null)
  printf "%-20s %-10s %-34s\n" "${volume:0:18}" "$driver" "${mountpoint:0:32}"
done
```

This helps identify which overlay2 directories in `/var/lib/docker/overlay2/` belong to which containers and images — useful for debugging disk usage or investigating specific layers.

## Disk Usage

### Check Overlay2 Disk Usage

```bash
# Overall Docker disk usage
docker system df
docker system df -v    # Verbose (per-container/image)

# Overlay2 directory size
du -sh /var/lib/docker/overlay2/

# Per-layer sizes
du -sh /var/lib/docker/overlay2/*/diff | sort -rh | head -20

# Find large container layers (writable)
for id in $(docker ps -q); do
    size=$(du -sh "$(docker inspect $id --format '{{.GraphDriver.Data.UpperDir}}')" 2>/dev/null | cut -f1)
    name=$(docker inspect $id --format '{{.Name}}')
    echo "$size $name"
done | sort -rh
```

### Clean Up

```bash
# Remove unused data (stopped containers, dangling images, unused networks)
docker system prune

# Remove all unused images (not just dangling)
docker system prune -a

# Remove unused volumes too
docker system prune -a --volumes

# Remove specific container's layer
docker rm <container>
```

## Image Layer Inspection

```bash
# Show image layers (diff IDs)
docker inspect <image> --format '{{json .RootFS.Layers}}' | jq .

# Show image history (commands that created each layer)
docker history <image>

# Show image history without truncation
docker history --no-trunc <image>

# Show layer sizes
docker history --format "{{.Size}}\t{{.CreatedBy}}" <image>

# Find which layer a file belongs to
docker inspect <image> --format '{{.GraphDriver.Data.LowerDir}}'
# Then search each layer:
find /var/lib/docker/overlay2/<layer-id>/diff -name "filename"
```

## Configuration

### Docker Daemon Storage Options

```json
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true",
    "overlay2.size=20G"
  ]
}
```

```bash
# Location: /etc/docker/daemon.json
# Restart Docker after changes
sudo systemctl restart docker
```

### Change Storage Location

```bash
# Move Docker data directory
sudo systemctl stop docker
sudo mv /var/lib/docker /new/path/docker

# Update daemon.json
{
  "data-root": "/new/path/docker"
}

sudo systemctl start docker
```

## Overlay2 vs Other Storage Drivers

| Driver | Status | Use Case |
|--------|--------|----------|
| overlay2 | Default, recommended | Production on all modern kernels |
| overlay | Legacy | Older kernels without multiple lowerdir support |
| devicemapper | Deprecated | Was used on RHEL/CentOS 7 |
| aufs | Deprecated | Was default on older Ubuntu |
| btrfs | Supported | Btrfs filesystems |
| zfs | Supported | ZFS filesystems |
| vfs | Supported | Testing only (no CoW, full copy per layer) |

## Kernel Requirements

```bash
# Check kernel supports overlay
cat /proc/filesystems | grep overlay

# Check overlay module
lsmod | grep overlay

# Load overlay module
sudo modprobe overlay

# Minimum kernel versions:
# overlay2: Linux 4.0+ (RHEL 7.2+, Ubuntu 16.04+)
# Multiple lower dirs: Linux 4.0+
```

## Troubleshooting

### "No space left on device"

```bash
# Check overlay2 usage
du -sh /var/lib/docker/overlay2/
df -h /var/lib/docker/

# Find largest layers
du -sh /var/lib/docker/overlay2/*/diff | sort -rh | head -10

# Clean up
docker system prune -a --volumes
```

### Overlay Mount Errors

```bash
# Check if overlay module is loaded
lsmod | grep overlay
sudo modprobe overlay

# Check filesystem supports overlay (needs d_type)
xfs_info /var/lib/docker | grep ftype
# ftype=1 is required for XFS

# If ftype=0, reformat with ftype=1 or use ext4
mkfs.xfs -f -n ftype=1 /dev/sdX
```

### Performance Issues

```bash
# Many layers = slower file lookups
# Reduce layers in Dockerfile:
# Bad (many layers):
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget

# Good (single layer):
RUN apt-get update && apt-get install -y curl wget && rm -rf /var/lib/apt/lists/*

# Check layer count
docker inspect <image> --format '{{len .RootFS.Layers}}'
```

### Inode Exhaustion

```bash
# Check inodes
df -i /var/lib/docker/

# Many small files in overlay2 can exhaust inodes
# Solution: use a filesystem with more inodes or clean up
```

## Quick Reference

| Action | Command |
|--------|---------|
| Check storage driver | `docker info \| grep "Storage Driver"` |
| Container's merged dir | `docker inspect <c> --format '{{.GraphDriver.Data.MergedDir}}'` |
| Container's upper dir | `docker inspect <c> --format '{{.GraphDriver.Data.UpperDir}}'` |
| Container's changes | `docker diff <container>` |
| Image layers | `docker inspect <img> --format '{{json .RootFS.Layers}}'` |
| Image history | `docker history <image>` |
| Disk usage | `docker system df -v` |
| Overlay2 total size | `du -sh /var/lib/docker/overlay2/` |
| Clean up | `docker system prune -a --volumes` |
| Verify overlay support | `cat /proc/filesystems \| grep overlay` |
