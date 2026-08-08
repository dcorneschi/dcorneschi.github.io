<p align="center">
  <img src="/articles/images/docker-logo.svg" alt="Docker logo" width="200"/>
</p>

<h1 align="center">Docker Compose Cheatsheet</h1>

## Basic Commands

```bash
# Start services (detached)
docker compose up -d

# Start with build (rebuild images before starting)
docker compose up -d --build

# Start specific service(s)
docker compose up -d web db

# Start with forced recreate (even if config hasn't changed)
docker compose up -d --force-recreate

# Start without recreating existing containers
docker compose up -d --no-recreate

# Start and remove orphaned containers (services no longer in compose file)
docker compose up -d --remove-orphans

# Stop services (remove containers and default network)
docker compose down

# Stop and remove volumes
docker compose down -v

# Stop and remove images (all or local-only)
docker compose down --rmi all
docker compose down --rmi local

# Stop and remove everything
docker compose down -v --rmi all --remove-orphans

# Stop services without removing containers
docker compose stop

# Stop a specific service
docker compose stop web

# Start stopped services
docker compose start

# Restart services
docker compose restart
docker compose restart web

# Restart with timeout
docker compose restart -t 30 web

# Pause / unpause services
docker compose pause
docker compose unpause
```

## Service Management

```bash
# List running services
docker compose ps

# List all services (including stopped)
docker compose ps -a

# List services with specific format
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"

# List service containers (IDs only)
docker compose ps -q

# View service images
docker compose images

# View running processes in services
docker compose top

# Scale a service
docker compose up -d --scale worker=5

# Scale multiple services
docker compose up -d --scale worker=5 --scale api=3
```

## Logs

```bash
# View logs from all services
docker compose logs

# Follow logs (real-time)
docker compose logs -f

# Follow specific service(s)
docker compose logs -f web app

# With timestamps
docker compose logs -t

# Last N lines
docker compose logs --tail 100

# Last N lines and follow
docker compose logs -f --tail 50 web

# Since a time
docker compose logs --since 30m
docker compose logs --since "2024-01-01T00:00:00"

# No color (useful for piping/logging)
docker compose logs --no-color

# No log prefix (service name)
docker compose logs --no-log-prefix
```

## Execute Commands

```bash
# Execute command in a running service container
docker compose exec web bash
docker compose exec db psql -U postgres

# Execute as specific user
docker compose exec -u root web sh

# Execute without TTY (non-interactive, scripts)
docker compose exec -T web cat /etc/hosts

# Execute with environment variable
docker compose exec -e DEBUG=true web ./run.sh

# Execute with working directory
docker compose exec -w /app web ls

# Run a one-off command (creates a new container)
docker compose run --rm web bash
docker compose run --rm web npm test

# Run without starting linked services
docker compose run --no-deps --rm web bash

# Run with port mapping (override)
docker compose run --rm -p 9090:8080 web

# Run with specific service ports published
docker compose run --rm --service-ports web
```

## Build

```bash
# Build all services
docker compose build

# Build specific service
docker compose build web

# Build without cache
docker compose build --no-cache

# Build with build arguments
docker compose build --build-arg VERSION=1.0

# Build in parallel
docker compose build --parallel

# Pull images before building
docker compose build --pull

# View build output (no truncation)
docker compose build --progress=plain
```

## Pull and Push

```bash
# Pull all service images
docker compose pull

# Pull specific service
docker compose pull web

# Pull and recreate
docker compose pull && docker compose up -d

# Pull quietly (no progress output)
docker compose pull -q

# Push built images to registry
docker compose push

# Push specific service
docker compose push web
```

## Config and Validation

```bash
# View merged compose configuration
docker compose config

# Validate compose file (check for errors)
docker compose config --quiet

# View resolved service configuration
docker compose config --services

# View resolved volume names
docker compose config --volumes

# View resolved profile names
docker compose config --profiles

# View compose config as JSON
docker compose config --format json

# Use specific compose file
docker compose -f docker-compose.prod.yml up -d

# Use multiple compose files (merged left to right)
docker compose -f docker-compose.yml -f docker-compose.override.yml up -d

# Use specific project name
docker compose -p myproject up -d

# Use specific env file
docker compose --env-file .env.prod up -d
```

## Docker Compose File Reference

### Basic Structure

```yaml
version: "3.8"    # optional in modern Docker Compose

services:
  web:
    image: nginx:latest
    # or build from Dockerfile
    build: .

  db:
    image: postgres:16

volumes:
  pgdata:

networks:
  frontend:
  backend:
```

### Service Options

```yaml
services:
  app:
    # Image or build
    image: myapp:1.0
    build:
      context: .
      dockerfile: Dockerfile.prod
      args:
        - VERSION=1.0
      target: production          # multi-stage target

    # Container name (default: project_service_1)
    container_name: myapp

    # Networking
    ports:
      - "8080:80"                 # host:container
      - "127.0.0.1:9090:9090"    # bind to specific interface
      - "3000"                    # random host port
    expose:
      - "3000"                    # internal only (no host mapping)
    networks:
      - frontend
      - backend
    hostname: myapp.local
    dns:
      - 8.8.8.8
    extra_hosts:
      - "host.docker.internal:host-gateway"

    # Volumes
    volumes:
      - ./src:/app/src             # bind mount
      - pgdata:/var/lib/data       # named volume
      - /tmp/cache:/cache:ro       # read-only
    tmpfs:
      - /tmp

    # Environment
    environment:
      - NODE_ENV=production
      - DB_HOST=db
    env_file:
      - .env
      - .env.local

    # Commands
    command: ["npm", "start"]
    entrypoint: ["/docker-entrypoint.sh"]
    working_dir: /app

    # Dependencies
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

    # Health check
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

    # Restart policy
    restart: unless-stopped        # no | always | on-failure | unless-stopped

    # Resource limits
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M

    # Logging
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

    # Security
    user: "1000:1000"
    read_only: true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true

    # Labels
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`app.example.com`)"

    # Misc
    stdin_open: true               # equivalent to -i
    tty: true                      # equivalent to -t
    init: true                     # use tini as PID 1
    privileged: false
    pid: "host"                    # share host PID namespace
    platform: linux/amd64
```

### Volumes

```yaml
volumes:
  # Named volume (Docker-managed)
  pgdata:

  # Named volume with driver options
  nfs-data:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=nfs-server.example.com,rw"
      device: ":/path/to/share"

  # External volume (must exist already)
  existing-data:
    external: true
```

### Networks

```yaml
networks:
  # Default bridge network
  frontend:

  # Custom subnet
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

  # External network (must exist already)
  proxy:
    external: true
```

### Profiles

```yaml
services:
  web:
    image: nginx

  debug:
    image: busybox
    profiles:
      - debug

  test:
    image: myapp-test
    profiles:
      - test
```

```bash
# Start only default services (no profile)
docker compose up -d

# Start with specific profile
docker compose --profile debug up -d

# Start with multiple profiles
docker compose --profile debug --profile test up -d
```

### Secrets

```yaml
services:
  app:
    secrets:
      - db_password
      - api_key

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    environment: "API_KEY"        # from host env (Compose v2.23+)
```

Inside the container, secrets are mounted at `/run/secrets/<name>`.

### Extension Fields (YAML Anchors)

```yaml
x-common: &common
  restart: unless-stopped
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"

services:
  web:
    <<: *common
    image: nginx

  app:
    <<: *common
    image: myapp
```

## Multiple Compose Files

### Override Pattern

```bash
# docker-compose.yml        (base)
# docker-compose.override.yml (auto-loaded for dev)
docker compose up -d

# Explicit production override
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

### docker-compose.yml (base)

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
```

### docker-compose.override.yml (development — auto-loaded)

```yaml
services:
  app:
    volumes:
      - ./src:/app/src
    environment:
      - NODE_ENV=development
      - DEBUG=true
```

### docker-compose.prod.yml (production)

```yaml
services:
  app:
    image: registry.example.com/myapp:latest
    restart: unless-stopped
    environment:
      - NODE_ENV=production
    deploy:
      resources:
        limits:
          memory: 512M
```

## Environment Variables

### .env File (auto-loaded)

```bash
# .env (in same directory as docker-compose.yml)
POSTGRES_USER=admin
POSTGRES_PASSWORD=secret
APP_VERSION=1.0
```

### Variable Substitution in Compose

```yaml
services:
  db:
    image: postgres:${POSTGRES_VERSION:-16}    # default value
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:?error message}  # required

  app:
    image: myapp:${APP_VERSION}
```

### Precedence (highest to lowest)

1. Environment variables from shell
2. Variables from `--env-file` flag
3. Variables from `env_file` directive in compose
4. Variables from `.env` file
5. Default values in `${VAR:-default}` syntax

## Wait-for-Dependencies Patterns

### depends_on with condition

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

### Using wait-for-it / dockerize

```yaml
services:
  app:
    command: ["./wait-for-it.sh", "db:5432", "--", "npm", "start"]
    depends_on:
      - db
```

## Common Patterns

### Web App + Database + Cache

```yaml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/app
      - REDIS_URL=redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=app
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d app"]
      interval: 10s
      timeout: 5s
      retries: 5

  cache:
    image: redis:alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

### Reverse Proxy with Traefik

```yaml
services:
  traefik:
    image: traefik:v3.0
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    restart: unless-stopped

  app:
    image: myapp
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`app.example.com`)"
      - "traefik.http.routers.app.entrypoints=websecure"
      - "traefik.http.services.app.loadbalancer.server.port=3000"
    restart: unless-stopped
```

### Development with Hot Reload

```yaml
services:
  app:
    build:
      context: .
      target: development
    volumes:
      - ./src:/app/src
      - /app/node_modules          # anonymous volume (prevents overwrite)
    ports:
      - "3000:3000"
      - "9229:9229"               # debugger port
    environment:
      - NODE_ENV=development
    command: ["npm", "run", "dev"]
```

## Troubleshooting

```bash
# View merged config (check variable substitution)
docker compose config

# Check why a service won't start
docker compose logs web
docker compose ps -a

# View events
docker compose events

# Force recreate a service
docker compose up -d --force-recreate web

# Reset everything and start fresh
docker compose down -v --rmi all && docker compose up -d --build

# Check which compose file is being used
docker compose ls

# View compose project details
docker compose ls --format json

# Diff between running and config
docker compose up -d --dry-run
```

## CLI Flags Reference

| Flag | Description |
|------|-------------|
| `-f file` | Specify compose file |
| `-p name` | Specify project name |
| `--env-file file` | Specify env file |
| `--profile name` | Enable a profile |
| `--parallel N` | Max parallelism for builds |
| `--progress mode` | Progress output (auto/plain/tty/quiet) |
| `--dry-run` | Show what would happen without executing |
| `--verbose` | Show more output |
| `--ansi mode` | Control ANSI output (never/always/auto) |

### docker compose up Flags

| Flag | Description |
|------|-------------|
| `-d` | Detached mode |
| `--build` | Build images before starting |
| `--force-recreate` | Recreate containers even if unchanged |
| `--no-recreate` | Don't recreate already running containers |
| `--no-build` | Don't build images |
| `--remove-orphans` | Remove containers for undefined services |
| `--scale svc=N` | Scale a service to N instances |
| `--wait` | Wait for services to be healthy |
| `--timeout N` | Timeout in seconds for container shutdown |

### docker compose down Flags

| Flag | Description |
|------|-------------|
| `-v` / `--volumes` | Remove named volumes |
| `--rmi all` | Remove all images |
| `--rmi local` | Remove only locally-built images |
| `--remove-orphans` | Remove orphaned containers |
| `-t N` | Shutdown timeout |

## Remove and Cleanup

```bash
# Remove stopped service containers
docker compose rm

# Remove with confirmation skip
docker compose rm -f

# Remove stopped containers and their volumes
docker compose rm -v

# Remove all (containers, networks, volumes, images, orphans)
docker compose down --rmi all -v --remove-orphans
```

## Advanced Network Configuration

```yaml
networks:
  frontend:
    driver: bridge

  backend:
    driver: bridge
    internal: true            # no external connectivity

  encrypted-overlay:
    driver: overlay
    driver_opts:
      encrypted: "yes"
```

The `internal: true` option creates a network with no outbound connectivity — containers can only talk to each other, not the internet.

## Advanced Volume Configuration

```yaml
volumes:
  db_data:
    driver: local

  cache:
    driver: local
    driver_opts:
      type: tmpfs
      device: tmpfs

  nfs-share:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=nfs-server.example.com,rw"
      device: ":/path/to/share"
```

## Database with Init Scripts

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret

volumes:
  db_data:
```

Files in `/docker-entrypoint-initdb.d/` are executed on first startup (when the data volume is empty). Supports `.sql`, `.sql.gz`, and `.sh` files.

## Version Compatibility

```bash
# Docker Compose V2 (plugin, recommended)
docker compose version

# Docker Compose V1 (standalone, legacy)
docker-compose --version
```

| Version | Command | Notes |
|---------|---------|-------|
| V2 (plugin) | `docker compose` (no hyphen) | Included with Docker Desktop and modern Docker Engine |
| V1 (standalone) | `docker-compose` (with hyphen) | Legacy, requires separate install |

Most modern installations use V2. If you see `docker-compose` in old documentation, replace with `docker compose`.

## Quick Reference

| Command | Description |
|---------|-------------|
| `docker compose up -d` | Start services in background |
| `docker compose down` | Stop and remove containers |
| `docker compose down -v` | Stop and remove containers + volumes |
| `docker compose ps` | List running services |
| `docker compose logs -f` | Follow logs |
| `docker compose exec svc bash` | Shell into service |
| `docker compose run --rm svc cmd` | One-off command |
| `docker compose build --no-cache` | Rebuild without cache |
| `docker compose pull` | Pull latest images |
| `docker compose restart svc` | Restart a service |
| `docker compose config` | Validate and view resolved config |
| `docker compose top` | View running processes |
| `docker compose --profile name up` | Start with profile |
| `docker compose up --scale svc=N` | Scale a service |

## Kill and Signals

```bash
# Kill services with specific signal
docker compose kill -s SIGINT

# Send SIGTERM to a specific service
docker compose kill -s SIGTERM web

# Send custom signal to all services
docker compose kill -s SIGUSR1
```

## File Copy (docker compose cp)

```bash
# Copy files from service container to host
docker compose cp web:/app/logs ./logs

# Copy files from host to service container
docker compose cp ./config.json web:/app/config.json
```

## Port Inspection

```bash
# Show mapped port for a service
docker compose port web 80

# Show port on specific protocol
docker compose port web 80/tcp
```

## Events

```bash
# Stream real-time events
docker compose events

# Stream events in JSON format
docker compose events --json

# Filter events by service
docker compose events web
```

## Wait for Healthy Services

```bash
# Start and wait for all services to be healthy
docker compose up --wait

# Wait with timeout (seconds)
docker compose up --wait --wait-timeout 30
```

## Scale to Zero

```bash
# Stop a service without removing it (scale down)
docker compose up --scale web=0
```

## Execute in Detached Mode

```bash
# Run a background task inside a running container
docker compose exec -d web python background_task.py
```

## Advanced Filtering and Status

```bash
# Show only running containers
docker compose ps --status=running

# Show only exited containers
docker compose ps --status=exited

# Show services only (names)
docker compose ps --services

# Show all containers including stopped
docker compose ps --all

# Display resolved config with image digests
docker compose config --resolve-image-digests

# List available profiles
docker compose config --profiles
```

## Project Directory

```bash
# Set project directory explicitly
docker compose --project-directory /path/to/project up

# Run with custom port mapping (docker compose run)
docker compose run --publish 8080:80 web bash
```

## Init, Signals, and Graceful Shutdown

```yaml
services:
  app:
    image: myapp
    init: true                  # Use tini as PID 1 (proper signal handling)
    stop_signal: SIGTERM        # Signal sent on stop (default: SIGTERM)
    stop_grace_period: 30s      # Time to wait before SIGKILL
    restart: unless-stopped
```

## Resource Limits with ulimits

```yaml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000
```

## Logging Drivers

```yaml
services:
  app:
    image: myapp
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        labels: "production_status"

  syslog-app:
    image: myapp
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://192.168.0.42:514"
```

## Complex Network with Static IPs

```yaml
networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

  backend:
    driver: bridge
    internal: true

  external_network:
    external: true
    name: my_external_network

services:
  web:
    networks:
      frontend:
        ipv4_address: 172.20.0.10
      backend:

  db:
    networks:
      backend:
```

## Advanced Volume (Bind with Driver Options)

```yaml
volumes:
  host_bind:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /host/path/to/data

  shared_external:
    external: true
    name: my_shared_volume
```

## Nginx Reverse Proxy Pattern

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app

  app:
    build: .
    expose:
      - "3000"
```

## Useful Command Combinations

```bash
# Complete development reset
docker compose down -v --remove-orphans && docker compose up --build

# Update and restart specific service (no deps)
docker compose pull web && docker compose up -d --no-deps web

# Quick database access
docker compose exec db psql -U postgres -d myapp

# Run tests in isolated container
docker compose run --rm --no-deps web npm test

# Backup database before update
docker compose exec db pg_dump -U postgres myapp > backup.sql

# Health check then deploy
docker compose config --quiet && docker compose up --wait

# Resource usage monitoring
docker compose top && docker stats $(docker compose ps -q)

# Log rotation and cleanup
docker compose logs --since 24h > app.log && docker system prune -f

# Zero-downtime scale (scale up, then scale back down)
docker compose up -d --no-deps --scale web=2 web && \
    sleep 10 && \
    docker compose up -d --no-deps --scale web=1 web
```
