# Kubernetes Pod Commands

How `command` and `args` work in pod specs — their relationship to Docker's ENTRYPOINT/CMD, YAML syntax styles, and common patterns.

## YAML Syntax for Lists

Both styles are valid and produce the same result:

### Bracket Style (JSON/Flow Syntax)

```yaml
command: ['sh', '-c', 'echo hello']
args: ['--verbose', '--port=8080']
```

### Dash Style (Block Syntax)

```yaml
command:
  - sh
  - -c
  - echo hello
args:
  - --verbose
  - --port=8080
```

Identical to Kubernetes — just different YAML formatting. Pick whichever is more readable for your case.

## command vs args

| Field | Maps to Docker | Purpose |
|---|---|---|
| `command` | `ENTRYPOINT` | Overrides the image's entrypoint |
| `args` | `CMD` | Overrides the image's default arguments |

### Override Matrix

| command set? | args set? | What runs |
|:---:|:---:|---|
| No | No | Image's ENTRYPOINT + CMD (defaults) |
| No | Yes | Image's ENTRYPOINT + your args |
| Yes | No | Your command (no args) |
| Yes | Yes | Your command + your args |

## Key Rules

- If you set `command`, it **completely replaces** the image's `ENTRYPOINT`
- If you set `args`, it **replaces** the image's `CMD`
- If you set both, both are replaced
- If you set neither, the image defaults are used

## Examples

### Just Override args (Keeps Image's ENTRYPOINT)

```yaml
containers:
  - name: app
    image: myapp
    args: ["--port", "3000"]
```

The image's ENTRYPOINT runs with `--port 3000` instead of its default CMD.

### Override Both

```yaml
containers:
  - name: app
    image: myapp
    command: ["/bin/sh"]
    args: ["-c", "echo hello && sleep 3600"]
```

Ignores the image's ENTRYPOINT and CMD entirely.

### Run a Shell Script

```yaml
containers:
  - name: worker
    image: ubuntu:22.04
    command: ["/bin/bash", "-c"]
    args:
      - |
        echo "Starting..."
        apt-get update
        python3 /app/main.py --workers=4
        echo "Done"
```

The `|` (literal block) lets you write multi-line scripts cleanly.

### Override Entrypoint to Debug

```yaml
# Keep a container alive for debugging (bypass normal startup)
containers:
  - name: debug
    image: myapp:latest
    command: ["sleep"]
    args: ["infinity"]
```

### Pass Environment Variables in Commands

```yaml
containers:
  - name: app
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
      - echo "Host is $(hostname) and date is $(date)"
    env:
      - name: MY_VAR
        value: "hello"
```

Environment variable expansion works inside `args` when using `sh -c`.

### Init Container Example

```yaml
initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c']
    args:
      - |
        until nc -z db-service 5432; do
          echo "Waiting for database..."
          sleep 2
        done
        echo "Database is ready"
```

## Common Patterns

### Health Check Server

```yaml
containers:
  - name: app
    image: python:3.11-slim
    command: ["python3"]
    args: ["-m", "http.server", "8080"]
```

### Custom Entrypoint with Config File

```yaml
containers:
  - name: nginx
    image: nginx:latest
    command: ["nginx"]
    args: ["-g", "daemon off;", "-c", "/etc/nginx/custom.conf"]
```

### Chained Commands

```yaml
containers:
  - name: setup
    image: alpine
    command: ["/bin/sh", "-c"]
    args:
      - |
        apk add --no-cache curl &&
        curl -o /data/config.json https://config-server/api/config &&
        echo "Config downloaded"
```

### Using exec Form (No Shell)

```yaml
# Runs the binary directly, no shell interpretation
containers:
  - name: app
    image: myapp
    command: ["/app/server"]
    args: ["--config=/etc/app/config.yaml", "--port=8080", "--log-level=info"]
```

No shell means no variable expansion, no pipes, no redirects — but cleaner signal handling (SIGTERM goes directly to the process).

## Docker Comparison

| Dockerfile | Kubernetes | Effect |
|------------|-----------|--------|
| `ENTRYPOINT ["python3"]` | `command: ["python3"]` | Sets the executable |
| `CMD ["app.py"]` | `args: ["app.py"]` | Sets the default arguments |
| `ENTRYPOINT ["python3"]` + `CMD ["app.py"]` | No command/args | Runs `python3 app.py` |
| — | `args: ["server.py"]` | Runs `python3 server.py` (overrides CMD only) |
| — | `command: ["/bin/sh"]` + `args: ["-c", "echo hi"]` | Replaces both |

## Shell Form vs Exec Form

| Style | YAML | Behavior |
|-------|------|----------|
| Exec form | `command: ["/app/server", "--port=8080"]` | Process is PID 1, receives signals directly |
| Shell form | `command: ["/bin/sh", "-c", "/app/server --port=8080"]` | Shell is PID 1, may not forward signals |

**Prefer exec form for production** — SIGTERM is forwarded correctly for graceful shutdown.

Use shell form when you need:
- Variable expansion (`$MY_VAR`)
- Pipes (`|`)
- Redirections (`>`, `>>`)
- Command chaining (`&&`, `||`)

## Gotchas

- **command overrides ENTRYPOINT completely**: If you set `command`, the image's ENTRYPOINT is gone. There's no "append" — it's a full replacement.
- **Each list item is one argument**: `command: ["echo", "hello world"]` passes `hello world` as a single argument. `command: ["echo hello world"]` tries to execute a binary named `echo hello world`.
- **No shell by default**: `command: ["echo $HOME"]` won't expand `$HOME`. You need `command: ["/bin/sh", "-c", "echo $HOME"]`.
- **Quoting in bracket style**: Use single quotes around the list items in YAML: `['sh', '-c', 'echo hello']`. Double quotes also work but require escaping backslashes.
- **Multi-line args need `|`**: For readability with long scripts, use the YAML literal block scalar.
- **Signal handling**: If you wrap your process in a shell (`sh -c`), the shell is PID 1 and may not forward SIGTERM to your app. Use `exec` or exec form for proper graceful shutdown:
  ```yaml
  command: ["/bin/sh", "-c"]
  args: ["exec /app/server --port=8080"]  # exec replaces the shell
  ```
