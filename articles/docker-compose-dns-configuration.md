# Configuring DNS in Docker Compose

Docker Compose lets you control how a container resolves names: which upstream DNS servers it uses, which search domains apply to short hostnames, and how it reaches other services by name. This guide covers the valid options, how Docker's built-in service discovery works, and a common misconception about "network-level DNS".

## Two Kinds of DNS in Compose

There are two separate concerns:

1. **Service discovery between containers** — resolving another service by its name (for example `db` or `cache`). This is handled automatically by Docker's embedded DNS on user-defined networks; you do not configure servers for it.
2. **Upstream/external resolution** — resolving names outside the Compose project (for example `api.example.com`). This is where the `dns` and `dns_search` options apply.

## Service-Level DNS

Set custom resolvers and search domains on a service. These map to the container's `/etc/resolv.conf`.

```yaml
services:
  myapp:
    image: nginx:1.27-alpine
    dns:
      - 8.8.8.8
      - 8.8.4.4
    dns_search:
      - example.com
    dns_opt:
      - ndots:2
      - timeout:3
```

- `dns` is a list of resolver IP addresses the container uses for external lookups.
- `dns_search` sets search domains, so a short name like `web` can be tried as `web.example.com`.
- `dns_opt` sets resolver options such as `ndots` and `timeout` (the same options you would put in `resolv.conf`).

DNS configuration is per service; each service can have its own resolvers.

## Service Discovery on User-Defined Networks

When services share a user-defined network, Docker runs an embedded DNS server at `127.0.0.11` inside each container and resolves service names to their container IPs automatically. You do not set this up manually.

```yaml
services:
  web:
    image: nginx:1.27-alpine
    networks:
      - appnet

  db:
    image: postgres:16.4
    networks:
      - appnet

networks:
  appnet:
```

Here `web` can reach the database at the hostname `db` with no extra DNS configuration. Compose creates a default network even if you declare none, so name-based discovery usually works out of the box.

### Network aliases

Give a service additional resolvable names on a network with `aliases`:

```yaml
services:
  db:
    image: postgres:16.4
    networks:
      appnet:
        aliases:
          - database
          - primary-db

networks:
  appnet:
```

Other containers on `appnet` can now reach it as `db`, `database`, or `primary-db`.

## There Is No Network-Level dns Key

A common mistake is trying to set DNS on the network itself:

```yaml
# INVALID — the networks element has no dns attribute
networks:
  mynetwork:
    driver: bridge
    dns:
      - 8.8.8.8
```

The top-level `networks` element supports `driver`, `driver_opts`, `ipam`, `internal`, `attachable`, `labels`, `external`, and a few others — but **not** `dns`. Compose will reject or ignore a `dns` key placed there. To give every service the same resolvers, set `dns` on each service (optionally via a shared YAML anchor), not on the network.

Some low-level resolver behavior can be influenced through `driver_opts` for specific network drivers, but that is driver-dependent and is not a general `dns` setting. For most cases, configure DNS on the service.

### Apply the same DNS to many services

Use an extension field and a YAML anchor to avoid repeating the resolver list:

```yaml
x-dns: &dns
  dns:
    - 8.8.8.8
    - 1.1.1.1
  dns_search:
    - example.com

services:
  web:
    image: nginx:1.27-alpine
    <<: *dns

  api:
    image: myapi:1.0.0
    <<: *dns
```

This is the correct way to share DNS settings "across the network" — by applying them to each service consistently.

## A Valid Custom Network with IPAM

The source example mixed a valid custom bridge with an invalid `dns` key. Here is the valid part, with DNS moved to the service where it belongs:

```yaml
services:
  app:
    image: myapp:1.0.0
    networks:
      - mynetwork
    dns:
      - 8.8.8.8
      - 1.1.1.1

networks:
  mynetwork:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-mynetwork
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

## Adding Individual Host Entries

To add specific name-to-IP mappings without a full resolver, use `extra_hosts`, which writes to `/etc/hosts`:

```yaml
services:
  app:
    image: myapp:1.0.0
    extra_hosts:
      - "internal-api:10.0.0.5"
      - "host.docker.internal:host-gateway"
```

`host.docker.internal:host-gateway` is the common way to let a container reach a service on the Docker host.

## Verifying DNS Inside a Container

```bash
# Inspect the container's resolver configuration
docker compose exec app cat /etc/resolv.conf

# Resolve another service by name (embedded DNS)
docker compose exec app getent hosts db

# Resolve an external name
docker compose exec app getent hosts example.com

# If nslookup/dig is available in the image
docker compose exec app nslookup db
docker compose exec app nslookup example.com
```

On a user-defined network, `/etc/resolv.conf` typically shows `nameserver 127.0.0.11` — Docker's embedded DNS, which forwards external queries to the configured or host resolvers.

## Troubleshooting

| Symptom | Likely cause | Check |
|---------|--------------|-------|
| A service cannot resolve another by name | Services are not on a shared user-defined network | `docker compose exec app getent hosts <service>`; verify `networks` |
| External names fail but service names work | Upstream resolver unreachable | Set `dns:` on the service; test `getent hosts example.com` |
| Short names not resolving | Missing search domain | Add `dns_search:` |
| `dns` under `networks:` seems ignored | It is not a valid network attribute | Move `dns` to each service |
| Slow lookups | High `ndots` causing extra search attempts | Tune `dns_opt: ["ndots:1"]` |

## Notes

- `dns` and `dns_search` are set per service and control external resolution.
- Container-to-container discovery is automatic on user-defined networks via embedded DNS at `127.0.0.11`.
- The `networks` element has no `dns` key; share DNS across services with a YAML anchor instead.
- Use `aliases` for extra service names and `extra_hosts` for static entries.

## Quick Reference

```yaml
services:
  app:
    image: myapp:1.0.0
    networks:
      - appnet
    dns:
      - 8.8.8.8
      - 1.1.1.1
    dns_search:
      - example.com
    dns_opt:
      - ndots:1
    extra_hosts:
      - "legacy-host:10.0.0.9"

networks:
  appnet:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

```bash
docker compose exec app cat /etc/resolv.conf
docker compose exec app getent hosts appnet-service-name
docker compose exec app getent hosts example.com
```

For broader Compose reference, see the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md) and [Defining Variables in Docker Compose](articles/docker-compose-variables-env.md). For general name-resolution concepts, see the [DNS Cheatsheet](articles/dns-cheatsheet.md).
