# Canceling GitLab Runner Jobs and Cleaning Up

When a runner is jammed with stuck jobs, filling its disk with build artifacts, or needs a fresh start, there are a few levers: cancel jobs (via the runner or the GitLab API), clear the build/cache directories, prune Docker leftovers, or fully unregister the runner. Several of these are destructive, so this guide flags the blast radius of each.

> Related: [Switching a GitLab Runner to the Shell Executor](articles/gitlab-runner-shell-executor.md) for executor configuration.

## 1. Stop Jobs on the Runner

```bash
# List the runners configured on this host
gitlab-runner list

# Gracefully stop the runner service (lets running jobs wind down)
gitlab-runner stop

# Forcefully kill the runner and its in-flight jobs
gitlab-runner kill
```

`stop` is the polite option — it stops accepting new jobs and shuts down. `kill` is the hard stop for a runner that won't respond. Restart afterward with `gitlab-runner start`.

## 2. Cancel Jobs via the GitLab API

To cancel jobs from the GitLab side (without touching the runner host), use the Jobs API:

```bash
# List running jobs for a project
curl --header "PRIVATE-TOKEN: <your-token>" \
  "https://gitlab.example.com/api/v4/projects/<project-id>/jobs?scope[]=running"

# Cancel a specific job
curl --request POST --header "PRIVATE-TOKEN: <your-token>" \
  "https://gitlab.example.com/api/v4/projects/<project-id>/jobs/<job-id>/cancel"
```

Cancel every running job for a project in one loop:

```bash
TOKEN=<your-token>
BASE="https://gitlab.example.com/api/v4/projects/<project-id>"

curl -s --header "PRIVATE-TOKEN: $TOKEN" "$BASE/jobs?scope[]=running" \
  | jq -r '.[].id' \
  | while read -r id; do
      curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" "$BASE/jobs/$id/cancel" >/dev/null
      echo "cancelled job $id"
    done
```

This is the cleaner approach when you only want to stop specific pipelines rather than take the whole runner down.

## 3. Clear Build and Cache Directories

Build workspaces and caches accumulate and can fill the disk. Clear them **only while the runner is stopped** so you don't yank files out from under an active job:

```bash
gitlab-runner stop

# Remove build workspaces and local cache (destructive)
sudo rm -rf /var/lib/gitlab-runner/builds/*
sudo rm -rf /var/lib/gitlab-runner/cache/*

gitlab-runner start
```

> `rm -rf` is irreversible. Double-check the paths, and confirm no job is mid-run — the runner recreates these directories on the next job.

## 4. Clean Up Docker Leftovers (Docker executor)

A Docker-executor runner leaves stopped containers, dangling volumes, and old images behind:

```bash
# Remove stopped containers
docker container prune -f

# Remove unused volumes
docker volume prune -f

# Remove dangling images (safer)
docker image prune -f

# Remove ALL unused images (aggressive)
docker image prune -a -f
```

> `docker prune` affects **everything on the host**, not just GitLab's containers. On a shared machine this can delete images and volumes other workloads rely on. Prefer the non-`-a` image prune unless you specifically want to reclaim all unused images. `docker system prune -a --volumes` is the most aggressive form — use with care.

## 5. Reset the Runner Completely

To wipe a runner's registration and start fresh:

```bash
# Unregister ONE runner (preferred — targeted)
gitlab-runner unregister --url <gitlab-url> --token <runner-token>

# Unregister EVERY runner on this host (destructive)
gitlab-runner unregister --all-runners
```

> `--all-runners` removes every runner configured on the host from GitLab and from `config.toml`. Only use it when you truly want a clean slate; otherwise unregister by URL + token to avoid taking out unrelated runners.

After unregistering, re-register as needed and restart the service.

## Choosing the Right Action

| Goal | Do this |
|------|---------|
| Stop a runaway runner | `gitlab-runner stop` (or `kill` if unresponsive) |
| Cancel specific pipelines only | GitLab Jobs API `cancel` |
| Reclaim disk from builds/cache | Stop runner, clear `builds/`+`cache/`, start |
| Reclaim disk from Docker | `docker container/volume/image prune` |
| Fully reset a runner | `gitlab-runner unregister` (by token, not `--all`) |

## Cautions

- **Stop the runner before deleting its directories** — clearing `builds/` under a live job corrupts it.
- **`rm -rf` and `docker prune` are irreversible** and, for Docker, host-wide. Verify paths and shared usage first.
- **Prefer targeted over blanket.** Unregister by token rather than `--all-runners`; prune dangling images rather than all.
- **Check permissions** — the cleanup paths need `sudo`; run those deliberately.
- **Replace placeholders** (`<gitlab-url>`, `<runner-token>`, `<your-token>`, `<project-id>`, `<job-id>`) with real values.

## Summary

- Cancel work with `gitlab-runner stop`/`kill` (host side) or the Jobs API `cancel` endpoint (GitLab side).
- Reclaim disk by clearing `builds/`+`cache/` (runner stopped) and pruning Docker leftovers.
- Fully reset by unregistering — targeted by token, or `--all-runners` for a clean slate.
- Treat `rm -rf`, `docker prune`, and `--all-runners` as high-risk; scope them narrowly.
