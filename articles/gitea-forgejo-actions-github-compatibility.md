# Gitea / Forgejo Actions vs GitHub Actions Compatibility

Gitea Actions (and Forgejo Actions, which shares its lineage since Forgejo forked Gitea) is a self-hosted CI system deliberately designed to be compatible with GitHub Actions — same YAML workflow format, same `uses:`/`run:` steps, and it can pull actions like `actions/checkout` straight from GitHub. That compatibility is high but not total. This guide maps what carries over, what's extra, and what's missing or behaves differently, so you know what to expect when moving workflows to a self-hosted forge.

> Sources: Gitea's official [Compared to GitHub Actions](https://docs.gitea.com/1.26/usage/actions/comparison) and [Actions FAQ](https://docs.gitea.com/1.25/usage/actions/faq/). Details reflect Gitea 1.25/1.26; Forgejo tracks the same design with its own runner. Content was rephrased for compliance with licensing restrictions.

## The Big Picture

- **Workflow format is the same.** `.gitea/workflows/*.yaml` (or `.github/workflows/*.yaml`) uses the GitHub Actions YAML syntax — jobs, steps, `uses`, `run`, `on:` triggers, matrix builds.
- **A separate runner executes jobs.** GitHub runs jobs on GitHub-hosted or self-hosted runners; Gitea/Forgejo require you to install and register a runner daemon yourself — `act_runner` for Gitea, **Forgejo Runner** for Forgejo. It's built on [nektos/act](https://github.com/nektos/act), which runs jobs in Docker containers.
- **Actions are reusable.** By default a non-qualified `uses: actions/checkout@v4` is fetched from github.com, so most Marketplace-style actions work as-is.

## Runner Setup (the main operational difference)

Unlike GitHub's hosted runners, there's no managed compute — you provide it:

```bash
# Gitea: register the act_runner against your instance
act_runner register --instance https://gitea.example.com --token <REGISTRATION_TOKEN>

# Then run it (typically as a service)
act_runner daemon
```

The runner introduces itself to the instance and reports **labels** describing what it can run (e.g. `ubuntu-latest:docker://...`). Your workflow's `runs-on:` must match a registered label, or the job stays queued with nothing to pick it up.

## Where Gitea/Forgejo Adds Features GitHub Lacks

- **Absolute action URLs.** You can reference an action by full URL from any Git host, e.g. `uses: https://gitea.com/actions/checkout@v4` or `uses: http://your-instance/owner/repo@branch` — not just `owner/repo`.
- **Actions written in Go.** In addition to JavaScript and container actions, Gitea supports authoring actions in Go.
- **Extra cron shorthands.** `schedule` accepts non-standard entries like `@yearly`, `@monthly`, `@weekly`, `@daily`, `@hourly`, which GitHub Actions does not.

## Configuring Where Actions Are Downloaded From

The `[actions].DEFAULT_ACTIONS_URL` setting controls where non-qualified actions resolve. Since Gitea 1.21 it accepts two values:

- `github` (the default) — fetch `actions/checkout@v4` from github.com.
- `self` — fetch only from your own instance, useful for air-gapped or restricted networks.

Absolute URLs in `uses:` always work regardless of this setting.

## Unsupported Workflow Syntax

These GitHub keys are parsed but currently **ignored** (they won't error, but they do nothing):

| Syntax | Status in Gitea Actions |
|--------|-------------------------|
| `jobs.<id>.timeout-minutes` | Ignored |
| `jobs.<id>.continue-on-error` | Ignored |
| `jobs.<id>.environment` | Ignored |
| Complex `runs-on` expressions | Only `runs-on: xyz` or `runs-on: [xyz]` supported |
| Problem Matchers | Ignored |
| Error annotations (`::error::`) | Ignored |

For step/job conditionals, only the `always()` expression function is guaranteed supported.

## Different Behavior to Watch For

- **Token and permissions model.** The auto-injected token is `GITEA_TOKEN` (vs `GITHUB_TOKEN`). Gitea honors `permissions:` / `jobs.<id>.permissions`, but the scopes differ: GitHub-only scopes like `statuses`, `checks`, `deployments`, `id-token`, `security-events`, and `pages` are **not** supported, while Gitea adds its own (`code`, `releases`, `wiki`, `projects`). Effective permissions are clamped by repo/owner settings and further restricted for fork PRs.
- **Publishing packages.** The built-in job token can't yet publish to the instance's package registry (e.g. push OCI images); the workaround is a Personal Access Token.
- **Context availability isn't enforced.** Gitea doesn't check context availability, so you can reference the `env` context in more places than GitHub allows — convenient, but it means a workflow that works on Gitea might fail context checks on GitHub.
- **Secrets variable naming.** Use `GITEA_*` conventions where the FAQ recommends, to avoid mixing GitHub-specific secrets into workflows you run on a forge.

## Missing UI Features

The execution is compatible; the web UI is leaner. Notably, **Pre and Post steps** and **Services** don't get their own collapsible sections in the job log view — their output is still there, just not separated out the way GitHub's UI does.

## Writing a Portable Workflow

To keep a workflow working on both GitHub and a self-hosted forge:

```yaml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest          # must match a registered runner label
    steps:
      - uses: actions/checkout@v4    # resolves from github.com by default
      - run: make test
```

- Stick to widely supported syntax; avoid `timeout-minutes`, `continue-on-error`, and `environment` if you need identical behavior on both.
- Don't rely on Problem Matchers or annotations for gating.
- Keep `runs-on` to a simple label.
- If you need package publishing on Gitea/Forgejo, plan for a PAT rather than the job token.

## Summary

- Gitea and Forgejo Actions reuse GitHub Actions' workflow format and can pull actions from GitHub, so most workflows port with little change.
- You must run your own runner (`act_runner` / Forgejo Runner) and match `runs-on` labels — there are no hosted runners.
- Extras: absolute action URLs, Go actions, and extra cron shorthands.
- Gaps: `timeout-minutes`, `continue-on-error`, `environment`, Problem Matchers, and annotations are ignored; only `always()` is guaranteed among expressions.
- Differences: `GITEA_TOKEN` with a different permission scope set, no job-token package publishing (use a PAT), unchecked context availability, and a leaner job-log UI.
