# Move the Docker Data Directory (data-root)

Docker stores images, containers, volumes, and build data under a single root directory, `/var/lib/docker` by default. When that filesystem runs low on space, you can relocate the data to another path by copying the existing data and setting `data-root` in the daemon configuration.

This guide follows a copy-first approach: the original data stays in place until the new location is confirmed working, so you can roll back cleanly. It targets Linux hosts running Docker Engine under systemd.

> **Warning:** This procedure requires downtime. All containers stop while Docker is stopped, and moving the data root affects every image, container, and local volume on the host. Take a maintenance window, and do not delete the old directory until the new location is verified.

## Before You Start

Confirm the current data root and how much space it uses:

```sh
# Current data-root (default is /var/lib/docker)
docker info -f '{{ .DockerRootDir }}'

# Space used by the current data directory
sudo du -sh /var/lib/docker

# Free space on the destination filesystem
df -h /apps
```

The destination must have enough free space for the full contents of the current data root, plus room to grow. Prefer a destination on a separate disk or volume dedicated to Docker data.

> Note the storage driver before you begin with `docker info -f '{{ .Driver }}'`. The `overlay2` driver requires a backing filesystem with `d_type` support (ext4, or XFS formatted with `ftype=1`). Moving data onto an unsupported filesystem can break Docker. See the [Docker Overlay2 Storage Driver](articles/docker-overlay2-storage.md) guide for details.

## Step 1: Stop the Docker Service

Stop Docker so the data directory is not being written during the copy. Also stop the socket so the daemon is not reactivated on demand.

```sh
sudo systemctl stop docker docker.socket
```

Confirm nothing is still running:

```sh
sudo systemctl status docker --no-pager
docker info 2>/dev/null || echo "Docker daemon is stopped"
```

## Step 2: Copy the Data

Create the destination directory and copy the data, preserving permissions, ownership, and extended attributes:

```sh
sudo mkdir -p /apps
sudo rsync -aqxP /var/lib/docker/ /apps/docker/
```

The flags matter:

- `-a` preserves permissions, ownership, timestamps, symlinks, and device files.
- `-x` stays on one filesystem, so it does not descend into other mounts under `/var/lib/docker`.
- `-q` suppresses non-error output; `-P` shows progress and allows a resumed transfer.

The **trailing slashes** are important. `/var/lib/docker/` copies the contents of the directory into `/apps/docker/`. Omitting the source trailing slash would create `/apps/docker/docker/` instead.

Verify the copy looks complete before continuing:

```sh
sudo du -sh /var/lib/docker /apps/docker
sudo ls /apps/docker
```

The sizes should be close. You should see subdirectories such as `containers`, `image`, `overlay2`, and `volumes` in `/apps/docker`.

## Step 3: Create the Daemon Configuration

Set `data-root` in `/etc/docker/daemon.json`:

```sh
sudo vi /etc/docker/daemon.json
```

If the file does not exist, create it with just this setting:

```json
{
    "data-root": "/apps/docker"
}
```

If the file already contains other settings, add `data-root` as an additional key rather than overwriting the file. For example:

```json
{
    "log-driver": "json-file",
    "data-root": "/apps/docker"
}
```

Validate the JSON before restarting; an invalid `daemon.json` prevents the daemon from starting:

```sh
# Fails with a parse error if the JSON is malformed
python3 -m json.tool /etc/docker/daemon.json
```

## Step 4: Start the Docker Service

```sh
sudo systemctl start docker
```

If you stopped `docker.socket`, starting `docker` reactivates it. Confirm the service is healthy:

```sh
sudo systemctl status docker --no-pager
```

## Step 5: Verify the New Data Root

Confirm Docker now reports the new location and that your images and containers are intact:

```sh
# Should print /apps/docker
docker info -f '{{ .DockerRootDir }}'

# Existing images and containers should still be listed
docker images
docker ps -a

# Local volumes should still be present
docker volume ls
```

Start a container to confirm the runtime works end to end:

```sh
docker run --rm hello-world
```

If everything checks out, the migration is complete.

## Step 6: Reclaim the Old Directory

Only after you have verified the new data root, remove the old data to free space. This step is destructive and cannot be undone, so do it deliberately.

```sh
# Optional: keep the old copy until you are fully confident
sudo mv /var/lib/docker /var/lib/docker.old

# Later, once you are sure, delete it
sudo rm -rf /var/lib/docker.old
```

If you renamed the old directory, keep it only as long as you have space; it still occupies the original filesystem.

## Making the Destination a Dedicated Mount

For a lasting fix, back the new data root with its own disk or logical volume so Docker can never fill the root filesystem. Mount it at the data-root path and persist it in `/etc/fstab`.

```sh
# Example: mount a dedicated volume at /apps/docker
sudo mount /dev/vg_docker/lv_docker /apps/docker

# Persist across reboots (adjust device, path, and filesystem)
echo "/dev/vg_docker/lv_docker /apps/docker xfs defaults 0 2" | sudo tee -a /etc/fstab
```

Migrate the data (Step 2) onto the mounted volume so the files live on the new device rather than on the directory's original filesystem. See the [LVM Cheatsheet](articles/lvm-cheatsheet.md) and the [/etc/fstab Guide](articles/linux-fstab-guide.md) for creating and persisting volumes.

## SELinux Notes

On RHEL, Rocky Linux, AlmaLinux, and Fedora, SELinux labels the default data root. A relocated directory may lack the correct context, which can prevent containers from starting.

```sh
# Apply the Docker data-root file context to the new path
sudo semanage fcontext -a -e /var/lib/docker /apps/docker
sudo restorecon -R -v /apps/docker
```

`semanage` is provided by the `policycoreutils-python-utils` package. Reserve `setenforce 0` for short-lived diagnosis only; do not run production hosts with SELinux disabled to work around labeling.

## Rollback

If Docker fails to start or the data looks wrong, revert to the original location while you investigate. The original data is untouched because Step 2 copied rather than moved it.

```sh
sudo systemctl stop docker docker.socket

# Remove or revert the data-root setting in daemon.json
sudo vi /etc/docker/daemon.json

sudo systemctl start docker

# Confirm the original root is active again
docker info -f '{{ .DockerRootDir }}'
```

## Troubleshooting

| Symptom | Likely cause | Check |
|---------|--------------|-------|
| Daemon fails to start after the change | Invalid `daemon.json` | `journalctl -u docker --no-pager`; validate JSON |
| Images or containers missing | Copy incomplete or wrong trailing slash | Compare `du -sh` of both paths; look for a nested `docker/docker` |
| Containers fail to start on RHEL-family hosts | Missing SELinux context on the new path | `restorecon -R -v /apps/docker` |
| `data-root` ignored, old path still used | Setting overridden elsewhere | Check for `--data-root` in the unit's `ExecStart` or a systemd drop-in |
| Errors about `d_type` or overlay support | Destination filesystem unsupported | Verify ext4, or XFS with `ftype=1` |

Inspect daemon logs when the service will not start:

```sh
sudo journalctl -u docker --no-pager -n 50
```

If both `daemon.json` and a systemd `ExecStart` flag set the data root, Docker refuses to start due to the conflict. Configure it in only one place; `daemon.json` is preferred.

## Quick Reference

```sh
# 1. Stop Docker
sudo systemctl stop docker docker.socket

# 2. Copy data to the new location
sudo mkdir -p /apps
sudo rsync -aqxP /var/lib/docker/ /apps/docker/

# 3. Set data-root in /etc/docker/daemon.json
#    { "data-root": "/apps/docker" }

# 4. Start Docker
sudo systemctl start docker

# 5. Verify
docker info -f '{{ .DockerRootDir }}'
docker images && docker ps -a

# 6. Reclaim old space once verified
sudo rm -rf /var/lib/docker.old
```

For daemon configuration keys and the storage-driver background, see the [Docker Cheatsheet](articles/docker-cheatsheet.md) and [Docker Overlay2 Storage Driver](articles/docker-overlay2-storage.md).
