# Set the Container User (UID/GID) in Docker Compose

By default a Docker container runs as root (UID 0). To run a service as a specific user, set its UID and GID. This is the usual fix for bind-mount permission errors and a basic security improvement. This guide shows four ways to do it and the caveats for each.

Docker permission checks use **numeric** UID and GID, not names. A UID inside the container is the same UID on the host, so matching the container user to the owner of your mounted files is what makes writes succeed.

> For a deeper treatment of non-root images, entrypoint behavior, capabilities, and read-only containers, see [Docker Compose: Running Containers Without Root](articles/docker-compose-non-root.md).

## Option 1: Set the user in docker-compose.yml

Hardcode the UID and GID with the `user` directive. This overrides any `USER` set in the image.

```yaml
services:
  your-service:
    image: myapp:1.0.0
    user: "1000:1000"   # UID:GID
```

You can also give just a UID (`user: "1000"`), in which case the primary group comes from the image's user database. Prefer the explicit `UID:GID` form so behavior does not depend on the image.

This is the simplest option and works well when the target UID is stable across every host that runs the stack.

## Option 2: Use environment variables

Parameterize the user so it can differ per host or per developer:

```yaml
services:
  your-service:
    image: myapp:1.0.0
    user: "${UID}:${GID}"
```

Then supply the values when starting the stack:

```bash
UID=$(id -u) GID=$(id -g) docker compose up
```

> **bash caveat:** `UID` is a read-only variable in bash, and `GID` is often unset, so `export UID=...` fails and `${UID}`/`${GID}` may not resolve the way you expect. This inline form works because the assignments are passed to the `docker compose` process, but the more portable approach is to use your own variable names.

A more reliable pattern uses distinct names and a default value:

```yaml
services:
  your-service:
    image: myapp:1.0.0
    user: "${DOCKER_UID:-1000}:${DOCKER_GID:-1000}"
```

```bash
export DOCKER_UID=$(id -u)
export DOCKER_GID=$(id -g)
docker compose up -d
```

Or record the values once in a `.env` file next to the Compose file, which Compose loads automatically:

```env
DOCKER_UID=1000
DOCKER_GID=1000
```

## Option 3: Override at runtime

For a one-off run without editing the Compose file, pass `--user`:

```bash
docker compose run --user 1000:1000 your-service
```

`docker compose run` starts a new one-off container for the service, which is handy for a shell or a maintenance command:

```bash
docker compose run --user "$(id -u):$(id -g)" your-service sh
```

The equivalent flag exists on the Docker CLI as well:

```bash
docker run --user 1000:1000 myapp:1.0.0
```

Note that `run` does not affect the long-running service started by `docker compose up`; use Option 1 or 2 for that.

## Option 4: Find your current UID and GID

Look up the IDs to use:

```bash
id -u    # your user ID (UID)
id -g    # your primary group ID (GID)
id       # full identity, including supplementary groups
```

Matching the container user to the owner of your bind-mounted files is the common goal:

```bash
# Who owns the data directory on the host?
stat -c '%u:%g' ./data
```

If that prints `1000:1000`, set the service `user` to `1000:1000` so the process can read and write those files.

## Verify the Running User

Confirm the container actually runs as the intended user:

```bash
# Inspect the configured user
docker inspect --format '{{ .Config.User }}' "$(docker compose ps -q your-service)"

# Check the effective user inside a running container
docker compose exec your-service id
```

The `id` output should show the UID and GID you set.

## Which Option to Use

| Situation | Recommended option |
|-----------|--------------------|
| Fixed UID across all hosts | Option 1: hardcoded `user:` |
| UID varies per developer or host | Option 2: environment variables or `.env` |
| One-off command or debugging shell | Option 3: `docker compose run --user` |
| Unsure which UID to use | Option 4: `id -u` / `id -g`, then Option 1 or 2 |

## Caveats

- Setting `user` does not change file ownership; the mounted files must already be owned by, or accessible to, that UID/GID. See the bind-mount fixes in the [non-root guide](articles/docker-compose-non-root.md).
- Some images must start as root to run an entrypoint that fixes permissions and then drops privileges. Forcing `user` on those images can break startup; such images often expose `PUID`/`PGID` variables instead.
- A non-root user cannot bind to ports below 1024 without `NET_BIND_SERVICE` or an unprivileged-port sysctl.
- Named volumes inherit ownership from the image at first creation, while bind mounts always reflect host ownership. This difference is covered in [Docker Compose: Bind Mounts vs Named Volumes](articles/docker-compose-volumes-vs-bind-mounts.md).
- The user you specify may not exist in the image's `/etc/passwd`; the process still runs with that numeric UID, though some programs warn about an unknown user.

## Quick Reference

```yaml
# Hardcoded
services:
  app:
    user: "1000:1000"

# Parameterized with safe defaults
services:
  app:
    user: "${DOCKER_UID:-1000}:${DOCKER_GID:-1000}"
```

```bash
# Find your IDs
id -u; id -g

# Parameterized up
export DOCKER_UID=$(id -u) DOCKER_GID=$(id -g)
docker compose up -d

# One-off override
docker compose run --user "$(id -u):$(id -g)" app sh

# Verify
docker compose exec app id
```

See also the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md) and [Docker Compose: Running Containers Without Root](articles/docker-compose-non-root.md).
