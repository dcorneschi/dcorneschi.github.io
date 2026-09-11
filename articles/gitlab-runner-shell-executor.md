# Switching a GitLab Runner to the Shell Executor

If your Docker-executor GitLab Runner keeps failing on image pulls — usually Docker Hub rate limits — switching to the **shell executor** sidesteps the problem entirely. The shell executor runs job scripts directly on the host, with no container and no image pull. This guide covers converting an existing runner, registering a new shell runner alongside a Docker one, testing it, and the security tradeoffs to weigh.

## Why Switch

Docker Hub enforces anonymous pull rate limits, so a busy Docker-executor runner can start failing jobs with `toomanyrequests` errors. The shell executor removes the dependency on pulling images:

- **No Docker Hub rate limits** — nothing is pulled; scripts run on the host.
- **Faster startup** — no container creation overhead.
- **Direct host access** — full use of tools already installed on the machine.
- **Simpler debugging** — no container layer between you and the job.

The tradeoff is weaker isolation — covered in Security below.

## Option A: Convert an Existing Runner

### 1. Stop the runner

```bash
sudo gitlab-runner stop
```

### 2. Edit the configuration

```bash
sudo nano /etc/gitlab-runner/config.toml
```

### 3. Change the executor to shell

Find the runner block and switch its executor, removing the Docker-specific section:

```toml
# Before
[[runners]]
  name = "your-runner-name"
  url = "https://your-gitlab-url"
  token = "your-token"
  executor = "docker"
  [runners.docker]
    image = "alpine:latest"
    # ... other docker settings
```

```toml
# After
[[runners]]
  name = "your-runner-name"
  url = "https://your-gitlab-url"
  token = "your-token"
  executor = "shell"
  # the [runners.docker] section is removed
```

### 4. Start the runner

```bash
sudo gitlab-runner start
```

### 5. Verify

```bash
sudo gitlab-runner status
sudo gitlab-runner list
```

## Option B: Register a New Shell Runner

To keep your Docker runner and add a shell one, register a new runner non-interactively:

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "https://your-gitlab-url" \
  --registration-token "your-token" \
  --executor "shell" \
  --description "shell-runner" \
  --tag-list "shell"
```

Without `--non-interactive` you'll be prompted for the URL, registration token (GitLab → Admin Area → CI/CD → Runners, or a project's settings), description, tags, and executor.

> Note: recent GitLab versions favor **authentication tokens** created in the UI over the older `--registration-token` flow. If your instance uses the newer model, create the runner in the UI first and register with `--token` instead of `--registration-token`.

## Match Tags So Jobs Land on the Right Runner

A runner only picks up jobs whose `tags:` match its own (unless it's set to run untagged jobs).

1. In GitLab: project → **Settings** → **CI/CD** → **Runners**.
2. Edit the runner and ensure it has a `shell` tag (remove `docker` if you converted it).
3. Tag the jobs in `.gitlab-ci.yml` with `shell` so they route to this runner.

## Test the Configuration

```yaml
stages:
  - test

hello-runner:
  stage: test
  tags:
    - shell
  script:
    - echo "Hello from GitLab Runner!"
    - date
    - hostname
    - whoami

system-info:
  stage: test
  tags:
    - shell
  script:
    - uname -a
    - df -h
    - free -h || echo "Memory info not available"
```

Because scripts run on the host, any command you use (git, curl, language toolchains) must already be installed on that machine — there's no image to provide them.

## Troubleshooting

### Runner isn't picking up jobs

```bash
sudo gitlab-runner status
sudo gitlab-runner --debug run     # run in foreground with verbose output
sudo gitlab-runner restart
```

Most often this is a **tag mismatch** — the job's tags don't match the runner's, or the job is untagged and the runner isn't configured to run untagged jobs.

### Verify registration and config

```bash
sudo gitlab-runner verify
sudo cat /etc/gitlab-runner/config.toml
```

### Permission issues

The shell executor runs jobs as the `gitlab-runner` user. If jobs need extra privileges:

```bash
# Allow specific commands via sudo (prefer a scoped sudoers rule over blanket sudo)
sudo usermod -aG docker gitlab-runner   # only if jobs must run docker commands

sudo gitlab-runner restart
```

Adding the runner user to `docker` (or `sudo`) grants broad host access — do it deliberately, not by default.

## Security Considerations

The shell executor trades isolation for simplicity:

- Jobs run **directly on the host** as the `gitlab-runner` user, with access to everything that user can reach.
- There's no container sandbox — a malicious or buggy job can affect the host and other jobs.
- Anyone who can push a pipeline can run commands on the runner host.

Guidance:

- Prefer the shell executor for **trusted, internal** repos, testing, and dev — not for untrusted or public projects.
- For production or multi-tenant CI, keep the Docker executor (and solve rate limits another way — see below).
- Grant the runner user the minimum privileges its jobs actually need.

## Alternatives to Fixing Docker Hub Rate Limits

If you'd rather keep Docker isolation, you can address the rate limit without switching executors:

- Authenticate the runner to Docker Hub (a logged-in account has higher limits) via `DOCKER_AUTH_CONFIG`.
- Pull images through a **pull-through registry mirror** so repeated builds hit a local cache.
- Host base images in GitLab's own container registry.

## Complete Example config.toml

```toml
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "shell-runner"
  url = "https://your-gitlab-url"
  token = "your-runner-token"
  executor = "shell"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
```

With this in place the runner executes jobs directly on the host — no image pulls, and Docker Hub rate limits are out of the picture entirely.
