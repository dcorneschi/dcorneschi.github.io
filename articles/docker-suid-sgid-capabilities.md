# SUID, SGID, and Capabilities in Docker

SUID, SGID, and Linux capabilities are three privilege mechanisms that behave inside containers just as they do on the host — which makes them both useful and, if ignored, a security risk. This guide explains each, then shows how Docker and Compose let you drop, add, and neutralize these privileges to follow least privilege.

For the host-side detail, see [Linux Capabilities](articles/linux-capabilities.md) and the [Linux File Permissions Guide](articles/linux-file-permissions.md). For running containers as non-root, see [Set the Container User (UID/GID) in Docker Compose](articles/docker-compose-set-container-user.md) and [Docker Compose: Running Containers Without Root](articles/docker-compose-non-root.md).

## The Three Mechanisms

### SUID (Set User ID)

When the SUID bit is set on an executable, the process runs with the **file owner's** identity, not the caller's. `passwd` is the classic example: a normal user runs it, but it executes as root to edit `/etc/shadow`.

- Shown as `s` in the owner execute position: `-rwsr-xr-x`.
- Set with `chmod u+s`; find with `find / -perm -4000 -type f`.
- Risk: a vulnerability in a SUID-root binary gives an attacker the owner's full privileges (usually root).

### SGID (Set Group ID)

Like SUID, but the process runs with the file's **group** identity. On a **directory**, SGID makes newly created files inherit the directory's group — useful for shared workspaces.

- Shown as `s` in the group execute position: `-rwxr-sr-x`.
- Set with `chmod g+s`; find with `find / -perm -2000 -type f`.

### Capabilities

Capabilities split root's monolithic power into discrete units, so a process can hold only what it needs instead of full root:

- `CAP_NET_BIND_SERVICE` — bind ports below 1024.
- `CAP_NET_RAW` — use raw sockets (e.g. `ping`, `tcpdump`).
- `CAP_NET_ADMIN` — configure interfaces, routing, firewall.
- `CAP_DAC_OVERRIDE` — bypass file permission checks.
- `CAP_SYS_ADMIN` — a broad, powerful catch-all; treat as near-root.

Managed on the host with `setcap`/`getcap`. Capabilities are Linux-specific, whereas SUID/SGID are portable across Unix.

### How They Compare

| Aspect | SUID / SGID | Capabilities |
|--------|-------------|--------------|
| Granularity | All-or-nothing (full owner/group identity) | Fine-grained, per-power |
| Portability | Broad Unix support | Linux only |
| Attack surface | Whole owner privilege on compromise | Limited to the granted powers |
| Modern preference | Legacy | Preferred for privilege separation |

Capabilities are the modern approach: a compromised process holds only specific powers rather than complete ownership. `setcap 'cap_net_bind_service=+ep'` on a binary replaces the need to make it SUID root.

## SUID and SGID in Containers

SUID/SGID bits work identically inside a container. If an image ships a vulnerable SUID-root binary, an attacker who gains code execution as a normal container user can escalate to root **inside the container** — and root in a container is a meaningfully larger blast radius than a normal user, especially if the container is privileged or shares namespaces.

Inspect an image for SUID/SGID binaries:

```bash
# SUID binaries
docker run --rm myimage:1.0 find / -perm -4000 -type f 2>/dev/null

# SUID and SGID
docker run --rm myimage:1.0 find / -perm /6000 -type f 2>/dev/null
```

Remove unneeded ones at build time:

```dockerfile
FROM debian:12-slim
# ... install app ...
# Strip SUID/SGID bits from binaries the app does not need
RUN find / -xdev -perm /6000 -type f -exec chmod a-s {} + || true
```

Better still, prevent SUID escalation entirely at runtime (below) rather than relying on stripping every binary.

## Capabilities in Docker

Docker starts containers with a **restricted default capability set** (a subset of root's powers), then drops the rest. Even so, the recommended baseline is to drop everything and add back only what a service needs.

Inspect the effective capabilities of a container:

```bash
# capsh prints the current capability sets
docker run --rm ubuntu:24.04 sh -c 'capsh --print'
```

The default set includes capabilities such as `CAP_CHOWN`, `CAP_NET_RAW`, `CAP_SETUID`, `CAP_SETGID`, and `CAP_SETFCAP`, among others — not the full root set. (Do not rely on any hand-typed list; run `capsh --print` to see the actual set on your Docker version, since defaults can change.)

Drop all capabilities:

```bash
docker run --rm --cap-drop=ALL ubuntu:24.04 id
```

Add back only what is required:

```bash
# Grant only raw-socket access, nothing else
docker run --rm --cap-drop=ALL --cap-add=NET_RAW ubuntu:24.04 ping -c1 1.1.1.1
```

Docker capability flags use the name **without** the `CAP_` prefix (`NET_RAW`, not `CAP_NET_RAW`).

## no-new-privileges: Neutralizing SUID

The single most effective control against SUID/SGID escalation in a container is the `no-new-privileges` flag. It sets the kernel `NoNewPrivs` bit so a process can never gain privileges through `execve` — SUID and SGID bits are ignored, and file capabilities cannot raise privileges.

```bash
docker run --rm --security-opt no-new-privileges:true myimage:1.0
```

With this set, even a leftover SUID-root binary in the image cannot escalate. Combine it with a non-root `user` and `--cap-drop=ALL` for defense in depth.

## Compose Configuration

### Baseline: drop all, add nothing

```yaml
services:
  web:
    image: nginx:1.27-alpine
    cap_drop:
      - ALL
    ports:
      - "8080:80"    # host:container
```

Note the port mapping is `host:container`; the source example's `"80:8080"` would publish host 80 to container 8080, which nginx does not listen on by default.

### Add only what a service needs

```yaml
services:
  dns:
    image: internetsystemsconsortium/bind9:9.20
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE     # bind to port 53 (< 1024)
    ports:
      - "53:53/udp"
      - "53:53/tcp"
```

### Network capture (multiple capabilities)

```yaml
services:
  sniffer:
    image: nicolaka/netshoot:latest
    cap_drop:
      - ALL
    cap_add:
      - NET_RAW
      - NET_ADMIN
    network_mode: host
    command: ["tcpdump", "-i", "any"]
```

### Non-root user to defeat SUID

```yaml
services:
  app:
    image: myapp:1.0.0
    user: "1000:1000"
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
```

Running as an unprivileged user already means SUID-to-root inside the container is the main escalation path; `no-new-privileges` closes it, and `cap_drop: [ALL]` removes the ambient powers.

### Hardened service (all controls together)

```yaml
services:
  hardened:
    image: myapp:1.0.0
    user: "1000:1000"
    read_only: true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true
    tmpfs:
      - /tmp
      - /var/run
    volumes:
      - ./config:/app/config:ro
```

`read_only: true` makes the root filesystem immutable (with `tmpfs` for the few writable paths), so an attacker cannot drop or modify binaries; the capability and privilege controls limit what any compromised process can do.

### SGID directories via a shared group

SGID on a directory is usually handled by ownership on a mounted volume rather than a Compose flag. Run the service as a user in the shared group and pre-create the directory `chmod g+s` on the host so new files inherit the group. See the group-sharing approach in [Linux File Permissions Guide](articles/linux-file-permissions.md).

## Scanning an Image for SUID Binaries

A one-off check before deploying:

```bash
docker run --rm myimage:1.0 find / -perm -4000 -type f 2>/dev/null
```

In Compose, a throwaway service works too, but quote the shell command correctly:

```yaml
services:
  suid-scan:
    image: myimage:1.0
    entrypoint: ["sh", "-c", "find / -perm -4000 -type f 2>/dev/null"]
```

```bash
docker compose run --rm suid-scan
```

The source's `entrypoint: find` with `command: ["/ -perm -4000 2>/dev/null"]` does not work — `find` cannot interpret a shell redirection as an argument. Use `sh -c` so the shell handles `2>/dev/null`.

## Best Practices

- Set `cap_drop: [ALL]` as the baseline, then `cap_add` only the specific capabilities a service needs.
- Run as a non-root `user` so SUID-to-root is the only escalation path, then close it.
- Add `security_opt: [no-new-privileges:true]` to neutralize SUID/SGID and file-capability escalation.
- Use `read_only: true` with `tmpfs` for writable paths to prevent tampering.
- Scan images for SUID/SGID binaries and strip unneeded ones at build time.
- Prefer file capabilities (`setcap`) over SUID root when building your own images.
- Never run `--privileged` unless truly required; it disables most of these protections.

## Quick Reference

```bash
# Inspect capabilities and SUID binaries in an image
docker run --rm ubuntu:24.04 sh -c 'capsh --print'
docker run --rm myimage:1.0 find / -perm /6000 -type f 2>/dev/null

# Least-privilege run
docker run --rm \
  --user 1000:1000 \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  --read-only \
  myimage:1.0
```

```yaml
services:
  app:
    image: myapp:1.0.0
    user: "1000:1000"
    read_only: true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true
    tmpfs:
      - /tmp
```

For deeper background, see [Linux Capabilities](articles/linux-capabilities.md), the [Linux File Permissions Guide](articles/linux-file-permissions.md), [Docker Compose: Running Containers Without Root](articles/docker-compose-non-root.md), and [Set the Container User (UID/GID) in Docker Compose](articles/docker-compose-set-container-user.md).
