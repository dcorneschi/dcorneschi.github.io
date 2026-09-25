# UID vs PUID/PGID in Docker: Getting Volume Permissions Right

Mounted-volume permission errors are one of the most common Docker frustrations: a container writes files your host user can't touch, or can't read files you own. The fix is making the container's process run as the **same UID/GID** as the host user that owns the files. There are two mechanisms for that — the native `user:` directive and the `PUID`/`PGID` environment-variable convention — and knowing which an image supports is the whole game.

For the related mechanics, see [Set the Container User (UID/GID) in Docker Compose](articles/docker-compose-set-container-user.md), [Running Containers Without Root](articles/docker-compose-non-root.md), and [Fixing Docker Bind-Mount Permission Errors](articles/docker-bind-mount-permissions.md).

## UID and GID — the System Concept

A **UID** is the numeric identifier the kernel uses for a user; a **GID** is the same for a group. File ownership, process ownership, and permission checks all work on these numbers — usernames are just a convenience mapping in `/etc/passwd` and `/etc/group`.

```sh
id                       # your uid, gid, and groups
id -u                    # just your UID (e.g. 1000)
id -g                    # just your GID
getent passwd 1000       # who owns UID 1000
```

`root` is always UID 0; the first regular user is typically UID 1000. **The kernel checks numbers, not names** — this is the key to container permissions. A file owned by UID 1000 on the host is owned by "whatever user is UID 1000" inside a container, regardless of what that user is named there (or whether it has a name at all).

## Why Volumes Break

A bind mount shares the *same inodes* between host and container — same owner UID/GID on both sides. If the container's process runs as UID 0 (root, the default) or some baked-in UID like 911, files it creates are owned by that UID on the host too, and your host user can't manage them. Conversely, files you own as UID 1000 may be unreadable to the container's user.

The cure: make the container process run as **your host UID/GID**. Two ways to do that.

## Mechanism 1: `user:` (native Docker)

Docker's own `--user` flag (compose `user:`) tells the runtime to start the container's main process as a given UID/GID. It's built into the container runtime and works on **any** image.

```sh
docker run --user 1000:1000 myimage
```

```yaml
services:
  myapp:
    image: myimage
    user: "1000:1000"     # UID:GID
```

- Works with any image — it's an OCI/runtime feature, not something the image opts into.
- The process starts *directly* as that UID; there's no root phase inside the container.
- Downside: if the image's entrypoint needs root first (to `chown` a data dir, drop privileges, install at runtime), forcing `user:` can break it. The UID may also have no matching entry in the container's `/etc/passwd`, which some apps dislike.

## Mechanism 2: `PUID`/`PGID` (an image convention)

`PUID`/`PGID` are **not** Docker features — Docker knows nothing about them. They're environment variables that *some images' entrypoint scripts* read to adjust an internal user's UID/GID at startup. This pattern is the hallmark of **[linuxserver.io](https://www.linuxserver.io/)** images (Sonarr, Radarr, Plex, etc.), which use `s6-overlay`.

```yaml
services:
  sonarr:
    image: linuxserver/sonarr
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - /path/on/host/config:/config
      - /path/on/host/media:/data
```

How it works: the container **starts as root**, its entrypoint remaps the internal `abc` user to `PUID:PGID` (and `chown`s what it needs), then drops privileges to run the app as that UID. That root-first phase is exactly why these images can fix ownership on the volume before starting — something the `user:` directive can't do.

> `PUID`/`PGID` only work if the **image is written to honor them**. Setting `PUID` on an image that doesn't read it does nothing — the variable is simply ignored. It is not a substitute for `user:` on arbitrary images.

## Which to Use

| Situation | Use |
|-----------|-----|
| linuxserver.io image (Sonarr/Radarr/Plex/…) | `PUID`/`PGID` env vars |
| Image documents `PUID`/`PGID` support | `PUID`/`PGID` env vars |
| Official image (nginx, postgres, redis, …) | `user:` directive |
| Custom / your own image | `user:` (or build in your own PUID handling) |
| Entrypoint needs root before dropping privileges | `PUID`/`PGID` (if supported) — `user:` would break it |

Rule of thumb: **`PUID`/`PGID` when the image supports it; `user:` otherwise.** Don't set both — they solve the same problem two different ways and can conflict.

## Match the Container to Your Host UID

Whichever mechanism, the goal is the same: use *your* host UID/GID so mounted files line up. Parameterize it rather than hardcoding:

```yaml
services:
  sonarr:
    image: linuxserver/sonarr
    environment:
      - PUID=${PUID:-1000}      # default 1000 if unset
      - PGID=${PGID:-1000}
    volumes:
      - ./config:/config
      - /srv/media:/data
```

```sh
# run with your actual host UID/GID
PUID=$(id -u) PGID=$(id -g) docker compose up -d
```

For the native mechanism, interpolate into `user:`:

```yaml
services:
  myapp:
    image: myimage
    user: "${UID:-1000}:${GID:-1000}"
```

```sh
UID=$(id -u) GID=$(id -g) docker compose up -d
```

See [Defining Variables in Docker Compose](articles/docker-compose-variables-env.md) for how `${VAR:-default}` interpolation and `.env` files work.

## Does an Image Support PUID/PGID?

Since it's a per-image convention, you have to check. In order of reliability:

**1. Read the docs.** The image's Docker Hub / GitHub page states it explicitly. Any `linuxserver/*` image supports `PUID`/`PGID`.

**2. Inspect the baked-in environment:**

```sh
docker image inspect IMAGE --format '{{.Config.Env}}' | tr ' ' '\n' | grep -i -E 'puid|pgid|uid|gid'
```

**3. Check the running environment:**

```sh
docker run --rm IMAGE env | grep -iE 'puid|pgid'
```

**4. Look for the linuxserver.io signature** (s6-overlay + the `abc` user):

```sh
docker run --rm IMAGE cat /etc/passwd | grep abc      # linuxserver images have user 'abc'
docker run --rm --entrypoint sh IMAGE -c 'ls /etc/s6-overlay 2>/dev/null && echo s6-based'
```

Quick heuristics:
- **linuxserver.io** → always `PUID`/`PGID`.
- **Official images** (nginx, postgres, …) → no `PUID`; use `user:`.
- **Custom images** → check the docs / Dockerfile.

## Key Takeaways

- The kernel enforces permissions by **UID/GID number**, not username — matching numbers across host and container is what fixes volume permissions.
- **`user:` / `--user`** is native Docker: works on any image, starts the process directly as that UID, but can break entrypoints that need root first.
- **`PUID`/`PGID`** is an *image convention* (linuxserver.io / s6-overlay): the container starts as root, remaps its user, `chown`s volumes, then drops privileges. It does nothing on images that don't implement it.
- Set the value to your host UID/GID with `PUID=$(id -u) PGID=$(id -g)` (or `user: "${UID}:${GID}"`); don't use both mechanisms at once.
- Verify support from the image's docs, `docker image inspect ... Config.Env`, or the presence of s6-overlay / an `abc` user.

## Quick Reference

```sh
id -u ; id -g                                   # your host UID / GID

# linuxserver.io-style image (PUID/PGID convention)
docker run -e PUID=$(id -u) -e PGID=$(id -g) -v /host:/data linuxserver/sonarr

# any image (native user directive)
docker run --user "$(id -u):$(id -g)" -v /host:/data myimage

# does this image support PUID/PGID?
docker image inspect IMAGE --format '{{.Config.Env}}' | tr ' ' '\n' | grep -i puid
```

```yaml
# compose: PUID/PGID (image must support it)
environment:
  - PUID=${PUID:-1000}
  - PGID=${PGID:-1000}

# compose: native user directive (any image)
user: "${UID:-1000}:${GID:-1000}"
```

For related material, see [Set the Container User (UID/GID) in Docker Compose](articles/docker-compose-set-container-user.md), [Running Containers Without Root](articles/docker-compose-non-root.md), [Fixing Docker Bind-Mount Permission Errors](articles/docker-bind-mount-permissions.md), and [Defining Variables in Docker Compose](articles/docker-compose-variables-env.md).
