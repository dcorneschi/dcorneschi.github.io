# Docker Compose vs Docker Swarm

Docker Compose and Docker Swarm both use a Compose-format YAML file, which makes them look interchangeable — but they solve different problems. Compose runs a multi-container app on a **single host**; Swarm runs **services across a cluster** of nodes with scheduling, rolling updates, and built-in load balancing. This guide compares the commands and explains where each fits.

For command depth on each side, see the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md) and the [Docker Swarm Cheatsheet](articles/docker-swarm-cheatsheet.md).

## The Core Difference

- **Compose** operates on **containers** on one Docker host. `docker compose up` reads a Compose file and creates containers directly.
- **Swarm** operates on **services** (declarative desired state) scheduled as **tasks** across cluster nodes. `docker stack deploy` submits the desired state, and the swarm managers place and maintain it.

Both consume the same file format, but Swarm reads the `deploy:` section (replicas, placement, update policy, restart policy) that plain Compose largely ignores, while Compose reads keys like `build:` and `depends_on:` that Swarm ignores.

## Command Comparison

Swarm service names are prefixed with the stack name, so a `web` service in stack `mystack` becomes `mystack_web`.

| Task | Docker Compose (single host) | Docker Swarm (cluster) |
|------|------------------------------|------------------------|
| Deploy | `docker compose up -d` | `docker stack deploy -c compose.yml mystack` |
| Remove | `docker compose down` | `docker stack rm mystack` |
| List services | `docker compose ps` | `docker stack services mystack` |
| List tasks/replicas | `docker compose ps` | `docker stack ps mystack` |
| Service logs | `docker compose logs -f web` | `docker service logs -f mystack_web` |
| Scale | `docker compose up -d --scale web=3` | `docker service scale mystack_web=3` |
| Exec into a container | `docker compose exec web bash` | `docker exec -it $(docker ps -q -f name=mystack_web) bash` |
| Build image | `docker compose build` | Build/push separately; Swarm does not build |
| Force recreate/restart | `docker compose up -d --force-recreate` | `docker service update --force mystack_web` |
| Update image | `docker compose pull && docker compose up -d` | `docker service update --image myapp:1.2.0 mystack_web` |
| Validate/render config | `docker compose config` | `docker stack config -c compose.yml` |
| Update a service's settings | `docker compose up -d web` | `docker service update [flags] mystack_web` |

A few corrections to common comparison tables:

- `docker compose config` validates and renders the merged file. Its Swarm counterpart is `docker stack config`, **not** `docker stack ps` (which lists running tasks). Do not equate the two.
- **Swarm does not build images.** `docker stack deploy` ignores `build:`; you must build and push an image to a registry the nodes can reach, then reference it by tag. Compose builds locally.
- `docker service update` covers restart, force-recreate, image change, and scaling — there is no separate `restart` verb for a swarm service.

## Exec Is Different

Compose has a first-class `docker compose exec`. Swarm has no direct equivalent because a service's task can run on any node. `docker exec` only works on the node where the task happens to be scheduled:

```bash
# On the node running the task
docker exec -it "$(docker ps -q -f name=mystack_web)" sh
```

For a task on another node, connect to that node first, or find its placement with `docker stack ps mystack_web`.

## The deploy Key

The `deploy:` block is where Swarm differs most. Plain `docker compose up` ignores most of it (Compose honors `deploy.replicas` and `deploy.resources` to a degree, but not placement or update policy); Swarm uses it fully.

```yaml
services:
  web:
    image: myapp:1.2.0
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.role == worker
```

Under Swarm, this rolls out the image one replica at a time with a delay, starting new tasks before stopping old ones (`start-first`) for reduced downtime. Under plain Compose it is largely inert.

Conversely, `build:`, `depends_on:` conditions, and profiles are Compose features Swarm does not act on.

## When to Use Which

| Factor | Compose | Swarm |
|--------|---------|-------|
| Hosts | Single | Multiple (cluster) |
| Unit of management | Containers | Services and stacks |
| Typical use | Local dev, CI, single-server apps | Multi-node production, HA |
| Setup | None beyond Docker | `docker swarm init` + join nodes |
| Rolling updates | Manual (scale up/down) | Built in via `update_config` |
| Load balancing | Single-host networking | Routing mesh across nodes |
| Secrets/configs | `secrets` (file/env) | Cluster-managed secrets and configs |
| Builds images | Yes | No (use a registry) |

Use **Compose** for development, testing, CI, and single-host deployments where its simplicity and `build`/`exec` ergonomics shine. Use **Swarm** when you need multiple nodes, automated rolling updates, cluster-wide load balancing, and self-healing service replicas.

> Swarm is stable and simple to run, but much of the industry has moved multi-node orchestration to Kubernetes. Choose Swarm when you want built-in clustering with minimal moving parts; consider Kubernetes when you need its broader ecosystem and features.

## Workflow Examples

### Development with Compose

```bash
docker compose up -d
docker compose logs -f
docker compose up -d --scale api=2
docker compose exec api sh
docker compose down
```

### Production with Swarm

```bash
# One-time: initialize the swarm on a manager
docker swarm init --advertise-addr 192.168.1.10

# Build and push first — Swarm pulls by tag, it does not build
docker build -t registry.example.com/myapp:1.2.0 .
docker push registry.example.com/myapp:1.2.0

# Deploy the stack (image tag must be reachable by all nodes)
docker stack deploy -c docker-compose.yml myapp

# Scale a service
docker service scale myapp_api=5

# Rolling update to a new version
docker service update --image registry.example.com/myapp:1.3.0 myapp_api

# Inspect placement and health of tasks
docker stack services myapp
docker stack ps myapp

# Tear down
docker stack rm myapp
```

Note the image is pinned to a specific version and pushed to a registry the nodes share; a locally built image or a moving `latest` tag will not deploy reliably across a cluster. See [Pin Docker Image Versions Instead of latest](articles/docker-image-version-pinning.md).

## Quick Reference

```bash
# Compose (single host)
docker compose up -d
docker compose ps
docker compose logs -f web
docker compose up -d --scale web=3
docker compose exec web sh
docker compose down

# Swarm (cluster)
docker swarm init
docker stack deploy -c docker-compose.yml mystack
docker stack services mystack
docker stack ps mystack
docker service logs -f mystack_web
docker service scale mystack_web=3
docker service update --image myapp:1.3.0 mystack_web
docker stack rm mystack
```

For deeper reference, see the [Docker Compose Cheatsheet](articles/docker-compose-cheatsheet.md), the [Docker Swarm Cheatsheet](articles/docker-swarm-cheatsheet.md), and [Updating Docker Compose Containers](articles/docker-compose-updating-containers.md).
