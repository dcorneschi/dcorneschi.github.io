# Setting Up a Gitea Actions Runner (act_runner)

Gitea Actions runs CI jobs on a separate runner daemon called **act_runner**, which you install, register against your instance, and keep running. This guide walks through enabling Actions, installing the runner (binary or Docker), registering it, running it as a service, configuring labels, and the security options worth knowing — including ephemeral runners.

> For how Gitea Actions compares to GitHub Actions (workflow compatibility, `runs-on` labels, `GITEA_TOKEN`), see [Gitea / Forgejo Actions vs GitHub Actions Compatibility](articles/gitea-forgejo-actions-github-compatibility.md).

## Prerequisites

- **Docker** (recommended) installed with the daemon running — act_runner uses it to execute jobs in containers.
- A **Gitea instance** at version 1.19.0+ with Actions enabled.

## Step 1: Enable Actions in Gitea

Actions are on by default in Gitea 1.21.0+. On earlier versions, enable them in `app.ini` and restart Gitea:

```ini
[actions]
ENABLED = true
```

## Step 2: Install act_runner

### Option A — binary

```bash
# Download the binary for your platform
wget https://gitea.com/gitea/act_runner/releases/latest/download/act_runner-linux-amd64
chmod +x act_runner-linux-amd64
sudo mv act_runner-linux-amd64 /usr/local/bin/act_runner

# Verify
act_runner --version
```

### Option B — Docker image

```bash
docker pull gitea/act_runner:latest      # latest stable
docker pull gitea/act_runner:nightly     # newest features
```

## Step 3: Get a Registration Token

Grab a token from your instance at the level you want the runner scoped to:

- **Instance-wide:** `your-gitea/-/admin/actions/runners`
- **Organization:** `your-gitea/org/<org>/settings/actions/runners`
- **Repository:** `your-gitea/<owner>/<repo>/settings/actions/runners`

The token looks like `D0gvfu2iHfUjNqCYVljVyRV14fISpJ…`. Choose the narrowest scope that fits — a repo-level runner is safer than an instance-wide one.

## Step 4: Generate a Config (optional)

```bash
act_runner generate-config > config.yaml
```

The defaults work out of the box; edit this only to customize labels, cache, or container options.

## Step 5: Register the Runner

### Interactive

```bash
act_runner register --instance https://your-gitea-instance.com --token YOUR_TOKEN
```

### Non-interactive

```bash
act_runner register \
  --no-interactive \
  --instance https://your-gitea-instance.com \
  --token YOUR_TOKEN \
  --name my-runner \
  --labels ubuntu-latest:docker://node:20-bookworm
```

Registration writes a `.runner` file holding the runner's identity — keep it with the working directory the daemon uses.

## Step 6: Start the Runner

### Command line

```bash
act_runner daemon
# or with an explicit config
act_runner daemon --config config.yaml
```

### Docker

```bash
docker run -d \
  --name gitea-runner \
  -e GITEA_INSTANCE_URL=https://your-gitea-instance.com \
  -e GITEA_RUNNER_REGISTRATION_TOKEN=YOUR_TOKEN \
  -e GITEA_RUNNER_NAME=my-docker-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp/runner-data:/data \
  gitea/act_runner:latest
```

### Docker Compose

```yaml
services:
  gitea-runner:
    image: gitea/act_runner:latest
    environment:
      - GITEA_INSTANCE_URL=https://your-gitea-instance.com
      - GITEA_RUNNER_REGISTRATION_TOKEN=YOUR_TOKEN
      - GITEA_RUNNER_NAME=my-runner
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./runner-data:/data
    restart: unless-stopped
```

Mounting `docker.sock` gives the runner (and thus every job) control of the host's Docker — see Security below.

## Step 7: Run as a systemd Service

Create `/etc/systemd/system/act_runner.service`:

```ini
[Unit]
Description=Gitea Actions runner
After=docker.service

[Service]
Type=simple
User=act_runner
ExecStart=/usr/local/bin/act_runner daemon --config /etc/act_runner/config.yaml
WorkingDirectory=/var/lib/act_runner
Restart=always
RestartSec=15

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now act_runner
sudo systemctl status act_runner
```

## Step 8: Configure Labels

Labels advertise what a runner can run, and a workflow's `runs-on:` must match one of them. Each label maps a name to an execution environment:

```yaml
runner:
  labels:
    - "ubuntu-latest:docker://node:20-bookworm"   # run in this container image
    - "ubuntu-22.04:docker://node:20-bookworm"
    - "my-custom:host"                              # run directly on the host
```

The `:docker://<image>` form runs jobs in that image; `:host` runs them directly on the runner machine (no isolation — use sparingly).

## Step 9: Test It

Add `.gitea/workflows/test.yaml` to a repository:

```yaml
name: Test Workflow
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run test
        run: echo "Hello from Gitea Actions!"
```

Push it, then watch the run in the repo's **Actions** tab. If the job stays queued, the `runs-on` label doesn't match any registered runner (see Troubleshooting).

## Ephemeral Runners (safer for untrusted jobs)

An ephemeral runner handles a single job and then retires, which limits the blast radius of a compromised job:

```bash
# Binary
act_runner register \
  --instance https://your-gitea-instance.com \
  --token YOUR_TOKEN \
  --ephemeral

# Docker
docker run -d \
  --name gitea-runner-ephemeral \
  -e GITEA_INSTANCE_URL=https://your-gitea-instance.com \
  -e GITEA_RUNNER_REGISTRATION_TOKEN=YOUR_TOKEN \
  -e GITEA_RUNNER_EPHEMERAL=1 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitea/act_runner:latest
```

## Security Considerations

- **Isolate the host.** Run runners on a machine separate from the Gitea instance so a rogue job can't reach Gitea's data.
- **The `docker.sock` mount is powerful.** A job with access to the host Docker socket can effectively control the host. Prefer ephemeral runners, avoid `:host` labels, and consider rootless Docker or a socket proxy.
- **Scope tokens narrowly.** Register at the repo or org level rather than instance-wide when you can.
- **Use ephemeral runners** for public or untrusted repositories.

## Troubleshooting

| Symptom | Likely cause / fix |
|---------|--------------------|
| Job stuck "queued" | No runner label matches `runs-on` — check registered labels vs the workflow |
| Runner offline in the UI | Daemon not running, or lost network to Gitea; check `systemctl status`/logs |
| "docker: command not found" in jobs | Docker not installed/running on the runner host |
| Registration fails | Token expired or wrong instance URL |

Check logs with `journalctl -u act_runner -f` (service) or `docker logs gitea-runner` (container), and confirm the runner shows up under **Actions → Runners** in the Gitea UI.

## Summary

- Enable Actions (default in 1.21.0+), install `act_runner`, and register it with a scoped token.
- Run it via CLI, Docker, or a systemd service; keep it alive with `Restart=always`.
- Labels bind `runs-on` to a container image or the host — a mismatch is the usual reason jobs don't start.
- Isolate the runner host, be deliberate about the `docker.sock` mount, and use ephemeral runners for untrusted work.
