# Testing a GitLab Runner with a Standalone Pipeline

When you register a new GitLab Runner (or inherit one and aren't sure it works), the fastest sanity check is a small, standalone pipeline that does nothing risky — no cloud credentials, no Terraform, no deploys. It just proves the runner picks up jobs, the executor runs shell, and the network reaches both the internet and your GitLab instance.

This article walks through a self-contained test pipeline, explains each job, shows how to run a CI file that *isn't* the default `.gitlab-ci.yml`, and covers how to read the results and troubleshoot.

## The Full Test Pipeline

Save this as `.gitlab-ci.runner-test.yml` (or drop it into `.gitlab-ci.yml` — see [running a non-default file](#running-a-non-default-ci-file) below):

```yaml
# Sample pipeline to test the GitLab runner with basic commands.
# Standalone test config — it does NOT touch AWS or Terraform.

stages:
  - info
  - test

# Print runner and environment details
runner-info:
  stage: info
  image: alpine:latest
  script:
    - 'echo "=== Runner is alive ==="'
    - 'echo "Job:          $CI_JOB_NAME"'
    - 'echo "Pipeline:     $CI_PIPELINE_ID"'
    - 'echo "Pipeline src: $CI_PIPELINE_SOURCE"'
    - 'echo "Branch/ref:   $CI_COMMIT_REF_NAME"'
    - 'echo "Commit:       $CI_COMMIT_SHORT_SHA"'
    - 'echo "Runner:       $CI_RUNNER_DESCRIPTION ($CI_RUNNER_TAGS)"'
    - 'echo "Project:      $CI_PROJECT_PATH"'
    - 'echo "GitLab URL:   $CI_SERVER_URL"'

# Basic shell commands to confirm the executor works
basic-commands:
  stage: test
  image: alpine:latest
  script:
    - echo "Hello from the GitLab runner"
    - date
    - whoami
    - uname -a
    - pwd
    - ls -la
    - echo "Kernel and shell OK"

# Confirm the runner has network/internet access (busybox wget in alpine)
network-check:
  stage: test
  image: alpine:latest
  script:
    - echo "Checking outbound network..."
    - 'wget -q -O- https://example.com >/dev/null && echo "Internet OK" || echo "Internet FAILED"'

# Confirm the runner can reach the GitLab instance itself
gitlab-reachability:
  stage: test
  image: alpine:latest
  script:
    - apk add --no-cache curl >/dev/null
    - echo "Checking GitLab at $CI_SERVER_URL ..."
    - 'curl -sSf -o /dev/null "$CI_SERVER_URL" && echo "GitLab reachable OK" || echo "GitLab reachable FAILED"'
```

## What Each Job Proves

The pipeline uses two stages — `info` runs first, then `test` runs its three jobs in parallel — so each job isolates one thing that can go wrong.

| Job | Stage | What it confirms |
|-----|-------|------------------|
| `runner-info` | info | A runner picked up the job and predefined CI variables are populated |
| `basic-commands` | test | The executor can run shell commands inside the container |
| `network-check` | test | The runner has outbound internet access |
| `gitlab-reachability` | test | The runner can reach the GitLab instance (needed to clone, report status, upload artifacts) |

### `runner-info` — is a runner even picking this up?

If this job runs at all, a runner is connected and matched your pipeline. The `echo` lines print [predefined CI/CD variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html) so you can confirm *which* runner and *which* context:

- `CI_RUNNER_DESCRIPTION` / `CI_RUNNER_TAGS` — identifies the exact runner and its tags. If you expected a specific self-managed runner, check these match.
- `CI_PIPELINE_SOURCE` — how the pipeline was triggered (`push`, `web`, `schedule`, `merge_request_event`).
- `CI_PROJECT_PATH` / `CI_SERVER_URL` — the project and GitLab instance the job belongs to.

The single quotes around each `echo` keep YAML happy — colons and `$(...)` inside an unquoted scalar can otherwise break parsing.

### `basic-commands` — does the executor run shell?

`date`, `whoami`, `uname -a`, `pwd`, `ls -la` are deliberately boring. They confirm the executor (Docker, shell, Kubernetes, etc.) actually starts the container and runs commands. `whoami` and `uname -a` also tell you the user and host kernel — useful when debugging permission or platform issues later.

### `network-check` — outbound internet

Uses BusyBox `wget` (built into `alpine`) to fetch a well-known URL. Prints `Internet OK` or `Internet FAILED` without failing the job, so you get a clear signal even in a locked-down network. If this fails, jobs that pull dependencies (`npm ci`, `pip install`, `apk add`) will also fail.

### `gitlab-reachability` — can the runner talk back to GitLab?

The runner must reach the GitLab instance to clone the repo, report job status, and upload artifacts/logs. This job installs `curl` and hits `$CI_SERVER_URL`. A failure here often explains "stuck" or "hung" jobs even when the internet works — common with self-managed runners behind a firewall or split-horizon DNS.

## Running a Non-Default CI File

By default GitLab only reads `.gitlab-ci.yml` at the repo root. To run a differently named file like `.gitlab-ci.runner-test.yml`, you have two options.

### Option A — Point the project at the file

In the GitLab project UI:

**Settings → CI/CD → General pipelines → CI/CD configuration file**, set it to:

```
.gitlab-ci.runner-test.yml
```

You can also reference a file in another project or a remote URL here, for example:

```
.gitlab-ci.runner-test.yml@my-group/my-project:main
```

Remember to set it back to `.gitlab-ci.yml` when you're done testing.

### Option B — Temporarily use the default name

Copy the contents into `.gitlab-ci.yml`, push, then revert. Quick, but it clobbers any real pipeline config on that branch, so prefer Option A on shared repos.

### Then trigger it

- **Push** a commit to the branch, or
- Go to **CI/CD → Pipelines → Run pipeline**, pick the branch, and start it manually.

## Reading the Results

A healthy run shows all four jobs green, with the `test` stage jobs running together after `runner-info`:

```
info   ✔ runner-info
test   ✔ basic-commands
       ✔ network-check
       ✔ gitlab-reachability
```

Open each job's log to see the output. Because `network-check` and `gitlab-reachability` print `OK`/`FAILED` instead of exiting non-zero, always read the log text — a green checkmark alone doesn't guarantee connectivity passed. To make connectivity a hard failure instead, drop the `|| echo ... FAILED` fallback so the job exits non-zero on error:

```yaml
    - wget -q -O- https://example.com >/dev/null && echo "Internet OK"
```

## Common Failure Modes

| Symptom | Likely cause | What to check |
|---------|--------------|---------------|
| Pipeline stays "pending", no job starts | No runner available or tag mismatch | Runner is online (Settings → CI/CD → Runners); job tags match runner tags |
| `runner-info` never runs | Pipeline didn't get created | YAML errors — validate in **CI/CD → Editor**; confirm the config file path |
| `basic-commands` fails immediately | Executor/image problem | Docker daemon on the runner host; image pullable; disk space |
| `network-check` prints FAILED | No outbound internet | Runner egress firewall, proxy settings, DNS on the runner host |
| `gitlab-reachability` prints FAILED | Runner can't reach GitLab | `CI_SERVER_URL` DNS/routing, TLS/cert trust, self-managed firewall |
| `apk add` hangs or fails | Package mirror unreachable | Same as network-check — outbound access to Alpine mirrors |

## Validate the YAML Before Pushing

Catch syntax errors without burning a pipeline run:

```bash
# Lint locally with the GitLab CLI (glab)
glab ci lint .gitlab-ci.runner-test.yml
```

Or use the in-app **CI/CD → Editor**, which validates the pipeline and visualizes stages before you commit.

## When to Use This

- Right after registering a new self-managed runner, to confirm it's healthy.
- When jobs mysteriously hang or fail and you want to isolate runner vs. network vs. GitLab connectivity.
- As a smoke test in a new project before wiring up real build/deploy pipelines.

Because it never touches cloud credentials or infrastructure, it's safe to run anywhere — including on production runners — without side effects.

## References

- [GitLab CI/CD YAML syntax reference](https://docs.gitlab.com/ee/ci/yaml/) — official docs
- [Predefined CI/CD variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html) — official docs
- [Custom CI/CD configuration file path](https://docs.gitlab.com/ee/ci/pipelines/settings.html#specify-a-custom-cicd-configuration-file) — official docs
- [Register a runner](https://docs.gitlab.com/runner/register/) — official docs

*Content was rephrased for compliance with licensing restrictions.*
