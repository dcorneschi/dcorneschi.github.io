<p align="center">
  <img src="/articles/images/docker-logo.svg" alt="Docker logo" width="200"/>
</p>

<h1 align="center">Docker Compose: ports vs expose</h1>

## Quick Summary

| | `ports` | `expose` |
|--|---------|----------|
| Accessible from host | Yes | No |
| Accessible from other containers | Yes | Yes (same network) |
| Published to host interface | Yes | No |
| Visible in `docker ps` | Yes (as port mapping) | No |
| Use case | External access (browsers, clients) | Internal service communication |
| Equivalent `docker run` flag | `-p` | `--expose` |

## ports

Maps a container port to a port on the host machine. The service becomes reachable from outside Docker — browsers, external clients, other machines on the network.

```yaml
services:
  web:
    image: nginx
    ports:
      - "8080:80"        # host:container
```

```
Host machine (any client)
    │
    ├─── localhost:8080 ──▶ container:80
    │
Other machines on network
    │
    └─── host-ip:8080 ──▶ container:80
```

### Syntax Variants

```yaml
ports:
  # Short syntax
  - "8080:80"                  # host 8080 → container 80
  - "443:443"                  # same port on both
  - "3000"                     # random host port → container 3000
  - "127.0.0.1:8080:80"       # bind to localhost only
  - "0.0.0.0:8080:80"         # bind to all interfaces (explicit)
  - "8080:80/udp"             # UDP protocol
  - "8080:80/tcp"             # TCP protocol (default)

  # Long syntax (more explicit)
  - target: 80                 # container port
    published: 8080            # host port
    protocol: tcp
    mode: host                 # host or ingress (Swarm)
```

### Binding to Specific Interfaces

```yaml
services:
  web:
    ports:
      # Only accessible from the host itself (not from the network)
      - "127.0.0.1:8080:80"

  db:
    ports:
      # Only accessible from a specific network interface
      - "192.168.1.10:5432:5432"
```

This is important for security — binding to `127.0.0.1` prevents other machines on the network from reaching the port.

## expose

Declares that the container listens on a port, but does **not** publish it to the host. The port is only reachable from other containers on the same Docker network.

```yaml
services:
  app:
    image: myapp
    expose:
      - "3000"
```

```
Host machine
    │
    └─── localhost:3000 ──✗ NOT REACHABLE

Other containers (same network)
    │
    └─── app:3000 ──▶ container:3000 ✓ REACHABLE
```

### Syntax

```yaml
expose:
  - "3000"
  - "8080"
  - "3000/udp"        # specific protocol
```

`expose` is primarily documentation — it signals to other developers and tools which ports the service uses. Containers on the same network can communicate on any port regardless of `expose`, but declaring it makes intent explicit.

## How Containers Communicate

Even without `ports` or `expose`, containers on the same Docker network can reach each other on any port using the service name as hostname:

```yaml
services:
  web:
    image: nginx
    ports:
      - "80:80"      # host needs to reach this

  app:
    image: myapp
    expose:
      - "3000"       # documents the port, web can reach it

  db:
    image: postgres
    # No ports, no expose — but app can still reach db:5432
    environment:
      POSTGRES_PASSWORD: secret
```

```
Internet/Host ──▶ web:80 (published)
                    │
                    └──▶ app:3000 (internal only)
                            │
                            └──▶ db:5432 (internal only, not even exposed)
```

All three containers can talk to each other by service name because they're on the same default Compose network.

## When to Use Which

### Use `ports` when:

- The service needs to be accessed from outside Docker (browsers, API clients, external tools)
- You're running a web server, API gateway, or reverse proxy that faces users
- You're developing locally and need to hit the service from your machine
- The service is the entry point to your application

### Use `expose` when:

- The service is internal — only other containers need to reach it
- You're running a database, cache, message queue, or internal microservice
- You want to document which port the service uses without making it public
- Security requires the service to be unreachable from the host network

### Use neither when:

- The container is a worker/consumer that only makes outbound connections
- The service connects to other containers but doesn't accept inbound traffic

## Common Patterns

### Reverse Proxy (Only Proxy is Published)

```yaml
services:
  nginx:
    image: nginx
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - app

  app:
    image: myapp
    expose:
      - "3000"
    # NOT published — only nginx reaches it

  db:
    image: postgres
    # No ports, no expose — only app reaches it
```

### Development (Everything Published for Debugging)

```yaml
services:
  app:
    image: myapp
    ports:
      - "3000:3000"     # access app directly
      - "9229:9229"     # debugger port

  db:
    image: postgres
    ports:
      - "5432:5432"     # access db from host tools (pgAdmin, DBeaver)

  redis:
    image: redis
    ports:
      - "6379:6379"     # access redis from host (redis-cli)
```

### Production (Minimal Exposure)

```yaml
services:
  traefik:
    image: traefik:v3.0
    ports:
      - "80:80"
      - "443:443"
    # Only traefik faces the outside

  app:
    image: myapp
    expose:
      - "3000"
    # Internal only — traefik routes to it

  db:
    image: postgres
    # Nothing exposed — only app connects

  redis:
    image: redis
    # Nothing exposed — only app connects
```

### Multiple Protocols

```yaml
services:
  dns:
    image: coredns
    ports:
      - "53:53/tcp"
      - "53:53/udp"

  streaming:
    image: mystream
    ports:
      - "8080:8080/tcp"   # HTTP control
    expose:
      - "5004/udp"        # RTP stream (internal only)
```

## Security Implications

### ports Risks

- Published ports bypass Docker network isolation
- By default, `ports` binds to `0.0.0.0` — accessible from any network interface
- On Linux, Docker manipulates iptables directly, potentially **bypassing UFW/firewalld rules**
- Always bind to `127.0.0.1` if the service should only be reachable locally

```yaml
# DANGEROUS — accessible from the entire network
ports:
  - "5432:5432"

# SAFER — only accessible from the host itself
ports:
  - "127.0.0.1:5432:5432"
```

### Docker and Firewall (Linux)

Docker inserts iptables rules in the `DOCKER` chain, which is processed **before** `INPUT`. This means:

- `ufw deny 5432` does NOT block a Docker-published port
- The port is still reachable from the network

Mitigations:
- Don't publish ports you don't need — use `expose` instead
- Bind to `127.0.0.1` for host-only access
- Use `DOCKER_IPTABLES=false` in daemon.json (but then you must manage rules manually)
- Use an external firewall / security group (cloud environments)

### Best Practices

- Publish only entry-point services (reverse proxy, load balancer)
- Internal services (databases, caches, workers) should use `expose` or nothing
- Always specify the bind address for published ports in production
- Use Docker networks for isolation instead of relying on host firewall

## Relationship to Dockerfile EXPOSE

The `EXPOSE` instruction in a Dockerfile is purely documentation:

```dockerfile
# Dockerfile
EXPOSE 3000
```

It does **not** publish the port. It tells users and tools that the container listens on 3000. It's the Dockerfile equivalent of `expose` in Compose.

| Layer | Directive | Publishes to host? | Documents port? |
|-------|-----------|-------------------|-----------------|
| Dockerfile | `EXPOSE 3000` | No | Yes |
| Compose | `expose: ["3000"]` | No | Yes |
| Compose | `ports: ["8080:3000"]` | Yes | Yes |
| CLI | `docker run -p 8080:3000` | Yes | Yes |
| CLI | `docker run --expose 3000` | No | Yes |
| CLI | `docker run -P` | Yes (random ports for all EXPOSE'd) | Yes |

The `-P` flag (uppercase) publishes **all** ports declared with `EXPOSE` to random host ports:

```bash
docker run -d -P nginx
docker port <container>
# 80/tcp -> 0.0.0.0:32768
```

## Inspecting Ports

```bash
# Show port mappings for a running container
docker port myapp
# 3000/tcp -> 0.0.0.0:3000

# Show all containers with port info
docker compose ps
# NAME    IMAGE    COMMAND    SERVICE    PORTS
# web     nginx    ...        web        0.0.0.0:80->80/tcp

# Check which port compose mapped for a service
docker compose port web 80
# 0.0.0.0:80

# Check from inside the container which ports are listening
docker exec myapp ss -tlnp
docker exec myapp netstat -tlnp
```

## Troubleshooting

### Port conflict ("address already in use")

```bash
# Find what's using the port on the host
sudo ss -tlnp | grep :8080
sudo lsof -i :8080

# Use a different host port
ports:
  - "8081:80"    # changed host port
```

### Container reachable from network when it shouldn't be

```yaml
# Problem: bound to all interfaces by default
ports:
  - "5432:5432"

# Fix: bind to localhost only
ports:
  - "127.0.0.1:5432:5432"
```

### Container-to-container communication not working

- Ensure both services are on the same Docker network
- Use the **service name** as hostname (not `localhost`, not container IP)
- Check that the target container is actually listening (`docker exec target ss -tlnp`)
- `expose` is not required for container-to-container communication — it's just documentation
