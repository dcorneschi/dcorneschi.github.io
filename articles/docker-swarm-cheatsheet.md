# Docker Swarm Cheatsheet

Docker Swarm is Docker's native clustering and orchestration solution. It creates and manages a cluster of Docker nodes, deploying services across them with built-in load balancing, rolling updates, secrets management, and service discovery.

## Swarm Setup

### Initialize Swarm

```bash
# Initialize swarm (on manager node)
docker swarm init

# Initialize with specific advertise address
docker swarm init --advertise-addr 192.168.1.100

# Initialize with specific listen address
docker swarm init --listen-addr 192.168.1.100:2377
```

### Join Nodes

```bash
# Get join token for workers
docker swarm join-token worker

# Get join token for managers
docker swarm join-token manager

# Join as worker (run on worker node)
docker swarm join --token SWMTKN-1-... 192.168.1.100:2377

# Join as manager (run on manager node)
docker swarm join --token SWMTKN-1-... 192.168.1.100:2377
```

### Leave Swarm

```bash
# Leave swarm (on worker node)
docker swarm leave

# Leave swarm (on manager node, must force)
docker swarm leave --force

# Remove node from swarm (run on manager)
docker node rm node-id
```

### Rotate Join Tokens

```bash
# Rotate worker join token (invalidates old token)
docker swarm join-token --rotate worker

# Rotate manager join token
docker swarm join-token --rotate manager
```

### Lock the Swarm (Autolock)

Encrypts Raft logs at rest — managers require an unlock key after restart:

```bash
# Enable autolock
docker swarm update --autolock=true

# View current unlock key
docker swarm unlock-key

# Rotate unlock key
docker swarm unlock-key --rotate

# Unlock a restarted manager
docker swarm unlock

# Disable autolock
docker swarm update --autolock=false
```

## Node Management

### List and Inspect Nodes

```bash
# List all nodes
docker node ls

# Inspect node details
docker node inspect node-id
docker node inspect self
docker node inspect --pretty node-id

# List nodes with custom format
docker node ls --format "table {{.Hostname}}\t{{.Status}}\t{{.ManagerStatus}}"
```

### Node Labels

```bash
# Add labels to nodes
docker node update --label-add type=database node-id
docker node update --label-add zone=us-west node-id
docker node update --label-add environment=production node-id

# Remove labels
docker node update --label-rm type node-id
```

### Node Availability

```bash
# Set node availability
docker node update --availability active node-id
docker node update --availability pause node-id
docker node update --availability drain node-id

# Promote worker to manager
docker node promote node-id

# Demote manager to worker
docker node demote node-id
```

## Service Management

### Create Services

```bash
# Create simple service
docker service create --name web nginx

# Create service with replicas
docker service create --replicas 3 --name web nginx

# Create service with resource constraints
docker service create \
  --name web \
  --replicas 3 \
  --limit-cpu 0.5 \
  --limit-memory 512M \
  --reserve-cpu 0.25 \
  --reserve-memory 256M \
  nginx

# Create service with placement constraints
docker service create \
  --name db \
  --constraint node.labels.type==database \
  --constraint node.role==worker \
  postgres:12

# Create service with placement preferences (spread across zones)
docker service create \
  --name web \
  --placement-pref spread=node.labels.zone \
  nginx
```

### Publishing Ports

```bash
# Publish port (ingress mode — routing mesh, load balanced across all nodes)
docker service create --name web --publish 80:80 nginx

# Publish with explicit ingress mode
docker service create --name web --publish published=80,target=80,mode=ingress nginx

# Publish in host mode (no routing mesh, direct to node running the task)
docker service create --name web --publish published=80,target=80,mode=host nginx

# Publish multiple ports
docker service create --name app \
  --publish 80:80 \
  --publish 443:443 \
  nginx
```

| Mode | Behavior |
|------|----------|
| `ingress` (default) | Routing mesh — any node in the swarm can accept connections, load balanced to tasks |
| `host` | Binds directly to the node's port — only nodes running a task accept connections |

### Global Mode Services

Global services run one task on every node (useful for monitoring agents, log collectors):

```bash
# Create global service (one instance per node)
docker service create --mode global --name node-exporter prom/node-exporter

# Global service with constraints (only on workers)
docker service create \
  --mode global \
  --constraint node.role==worker \
  --name log-agent \
  fluentd:latest
```

### Placement Constraint Expressions

| Constraint | Description |
|------------|-------------|
| `node.id==<id>` | Match specific node ID |
| `node.hostname==<name>` | Match node hostname |
| `node.role==manager` | Only on manager nodes |
| `node.role==worker` | Only on worker nodes |
| `node.platform.os==linux` | Match operating system |
| `node.platform.arch==amd64` | Match architecture |
| `node.labels.<key>==<value>` | Match custom node label |
| `engine.labels.<key>==<value>` | Match Docker engine label |

```bash
# Combine multiple constraints (AND logic)
docker service create \
  --name db \
  --constraint node.role==worker \
  --constraint node.labels.type==database \
  --constraint node.platform.os==linux \
  postgres:15
```

### List and Inspect Services

```bash
# List services
docker service ls

# Inspect service
docker service inspect service-name
docker service inspect --pretty service-name

# List service tasks (containers)
docker service ps service-name

# Show service logs
docker service logs service-name
docker service logs -f service-name
docker service logs -f --tail 100 service-name
```

### Update Services

```bash
# Scale service
docker service scale web=5
docker service scale web=3 db=2

# Update service image
docker service update --image nginx:1.20 web

# Update with rolling update configuration
docker service update \
  --update-parallelism 2 \
  --update-delay 10s \
  --update-failure-action rollback \
  --image nginx:1.20 \
  web

# Add environment variables
docker service update --env-add NODE_ENV=production web

# Update resource limits
docker service update --limit-cpu 1.0 --limit-memory 1G web
```

### Rollback Services

```bash
# Rollback service to previous version
docker service rollback web

# Configure rollback behavior
docker service update \
  --rollback-parallelism 1 \
  --rollback-delay 5s \
  web
```

### Remove Services

```bash
# Remove single service
docker service rm web

# Remove multiple services
docker service rm web db cache
```

## Network Management

### Create Networks

```bash
# Create overlay network
docker network create --driver overlay my-network

# Create overlay network with subnet
docker network create \
  --driver overlay \
  --subnet 10.0.1.0/24 \
  --gateway 10.0.1.1 \
  my-network

# Create attachable overlay network (allows standalone containers)
docker network create --driver overlay --attachable my-network

# Create encrypted overlay network
docker network create --driver overlay --opt encrypted my-network
```

### List and Inspect Networks

```bash
# List networks
docker network ls

# List only overlay networks
docker network ls --filter driver=overlay

# Inspect network
docker network inspect my-network
```

### Connect Services to Networks

```bash
# Create service with specific network
docker service create --network my-network --name web nginx

# Connect existing service to network
docker service update --network-add my-network web

# Disconnect service from network
docker service update --network-rm my-network web
```

## Volume Management

### Create and Use Volumes

```bash
# Create named volume
docker volume create my-data

# Create NFS volume
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/data/nfs \
  nfs-data
```

### Mount Volumes in Services

```bash
# Named volume
docker service create \
  --name db \
  --mount source=db-data,target=/var/lib/postgresql/data \
  postgres:12

# Bind mount
docker service create \
  --name web \
  --mount type=bind,source=/host/path,target=/container/path \
  nginx

# tmpfs mount
docker service create \
  --name app \
  --mount type=tmpfs,target=/tmp,tmpfs-size=1G \
  myapp:latest
```

## Secrets Management

### Create and Manage Secrets

```bash
# Create secret from stdin
echo "my-secret-password" | docker secret create db-password -

# Create secret from file
docker secret create ssl-cert /path/to/cert.pem

# List secrets
docker secret ls

# Inspect secret (metadata only, value not shown)
docker secret inspect db-password

# Remove secret
docker secret rm db-password
```

### Use Secrets in Services

```bash
# Attach secret to service
docker service create \
  --name db \
  --secret db-password \
  --secret ssl-cert \
  postgres:12

# Secret with custom target and permissions
docker service create \
  --name app \
  --secret source=api-key,target=/run/secrets/api-key,uid=1000,gid=1000,mode=0400 \
  myapp:latest
```

## Config Objects

### Create and Manage Configs

```bash
# Create config from file
docker config create nginx.conf /path/to/nginx.conf

# Create config from stdin
docker config create app-config - <<EOF
server {
    listen 80;
    server_name example.com;
}
EOF

# List configs
docker config ls

# Inspect config
docker config inspect nginx.conf

# Remove config
docker config rm nginx.conf
```

### Use Configs in Services

```bash
docker service create \
  --name web \
  --config source=nginx.conf,target=/etc/nginx/nginx.conf \
  nginx
```

## Stack Management

### Deploy Stacks

```bash
# Deploy stack from compose file
docker stack deploy -c docker-compose.yml myapp

# Deploy with multiple compose files
docker stack deploy -c docker-compose.yml -c docker-compose.prod.yml myapp
```

### Manage Stacks

```bash
# List stacks
docker stack ls

# List stack services
docker stack services myapp

# List stack tasks
docker stack ps myapp

# Remove stack
docker stack rm myapp
```

## Monitoring and Troubleshooting

### Service Status

```bash
# Quick cluster status
docker node ls && docker service ls && docker stack ls

# Check tasks on specific node
docker node ps node-id

# Service tasks with no truncation
docker service ps service-name --no-trunc

# Service events
docker events --filter service=service-name
```

### Debugging

```bash
# Exec into a running service container
docker exec -it $(docker ps -q -f label=com.docker.swarm.service.name=service-name) /bin/bash

# Get container ID for a specific service task
TASK_ID=$(docker service ps service-name --format "{{.ID}}" -f desired-state=running | head -1)
CONTAINER_ID=$(docker inspect $TASK_ID --format "{{.Status.ContainerStatus.ContainerID}}")
docker exec -it $CONTAINER_ID /bin/bash
```

### Resource Monitoring

```bash
# Disk usage
docker system df

# Live container stats
docker stats

# Node resource info
docker node inspect node-id --format "{{json .Description.Resources}}"

# Cleanup unused resources
docker system prune -f
```

## Production Setup

### High Availability (3 Managers)

```bash
# Manager 1 (primary)
docker swarm init --advertise-addr 192.168.1.10

# Manager 2
docker swarm join --token MANAGER-TOKEN 192.168.1.10:2377

# Manager 3
docker swarm join --token MANAGER-TOKEN 192.168.1.10:2377

# Workers
docker swarm join --token WORKER-TOKEN 192.168.1.10:2377
```

### Node Labels for Production

```bash
# By role
docker node update --label-add role=web node1
docker node update --label-add role=database node2
docker node update --label-add role=cache node3

# By availability zone
docker node update --label-add zone=us-west-1a node1
docker node update --label-add zone=us-west-1b node2
docker node update --label-add zone=us-west-1c node3
```

### Node Maintenance

```bash
# Drain node for maintenance (migrates tasks away)
docker node update --availability drain node-id

# Return node to active
docker node update --availability active node-id
```

### Backup and Restore

```bash
# Backup swarm state (on manager)
sudo tar -czf swarm-backup.tar.gz -C /var/lib/docker/swarm .

# Backup volumes
docker run --rm -v volume_name:/data -v $(pwd):/backup ubuntu tar czf /backup/backup.tar.gz /data

# Restore swarm state
sudo systemctl stop docker
sudo rm -rf /var/lib/docker/swarm/*
sudo tar -xzf swarm-backup.tar.gz -C /var/lib/docker/swarm
sudo systemctl start docker
```

## Compose File for Swarm (deploy key)

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    deploy:
      replicas: 3
      placement:
        preferences:
          - spread: node.labels.zone
        max_replicas_per_node: 1
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 5s
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - frontend
    secrets:
      - source: api_key
        target: /run/secrets/api_key
        uid: "1000"
        gid: "1000"
        mode: 0400

  db:
    image: postgres:15
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.role == database
    volumes:
      - db-data:/var/lib/postgresql/data
    secrets:
      - db-password
    networks:
      - backend

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
    internal: true

volumes:
  db-data:

secrets:
  api_key:
    external: true
  db-password:
    external: true
```

## Best Practices

| Area | Recommendation |
|------|----------------|
| Managers | Use odd number (3 or 5) for HA quorum |
| Services | Design stateless when possible |
| Health checks | Always implement for automatic recovery |
| Resources | Set CPU/memory limits and reservations |
| Secrets | Never put sensitive data in images or env vars |
| Networks | Use overlay networks to isolate services |
| Updates | Configure rolling updates with rollback on failure |
| Placement | Spread replicas across zones/nodes |
| Security | Run containers as non-root when possible |
| Backups | Regularly backup swarm state and volumes |
