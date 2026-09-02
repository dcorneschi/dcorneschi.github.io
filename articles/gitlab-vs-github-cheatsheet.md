# GitLab vs GitHub Cheatsheet

Side-by-side reference of GitLab and GitHub — terminology, CI/CD, permissions, APIs, CLI tools, and the platform features that map (and the ones that don't). Aimed at people who know one platform and need to work in the other.

## Terminology Map

| Concept | GitHub | GitLab |
|---------|--------|--------|
| Change proposal | Pull Request (PR) | Merge Request (MR) |
| Grouping of repos | Organization | Group (supports nested subgroups) |
| Repo collection owner | Org / User | Group / User / Namespace |
| CI/CD config file | `.github/workflows/*.yml` | `.gitlab-ci.yml` |
| CI/CD unit of reuse | Action (Marketplace) | Component (CI/CD Catalog) |
| Issue board | Projects (boards) | Issue Boards |
| Wiki | Wiki | Wiki |
| Package hosting | GitHub Packages | GitLab Package Registry |
| Container registry | GitHub Container Registry (GHCR) | GitLab Container Registry |
| Static site hosting | GitHub Pages | GitLab Pages |
| Code search / review | Code scanning, CodeQL | SAST/Secret Detection (built-in) |
| Serverless runners | GitHub-hosted runners | GitLab SaaS runners / self-managed runners |
| Protected refs | Branch protection rules | Protected branches / tags / environments |

## Hosting and Editions

| Aspect | GitHub | GitLab |
|--------|--------|--------|
| SaaS | github.com | gitlab.com |
| Self-hosted | GitHub Enterprise Server | GitLab self-managed (Community + Enterprise editions) |
| Open source core | No (proprietary) | Yes (Community Edition is open source) |
| Free private repos | Yes | Yes |
| Tiers | Free, Team, Enterprise | Free, Premium, Ultimate |

## CI/CD Comparison

The biggest conceptual gap between the two platforms.

| Feature | GitHub Actions | GitLab CI/CD |
|---------|----------------|--------------|
| Config location | `.github/workflows/` (multiple files) | `.gitlab-ci.yml` (one entry file, can `include:` more) |
| Execution unit | Steps composed from Marketplace actions | Jobs = container image + shell `script:` |
| Checkout | `actions/checkout@v4` step | Automatic before `script:` |
| Tool setup | `actions/setup-*` actions | Choose an `image:` with the tool baked in |
| Reuse across repos | Reusable workflows (`workflow_call`) | `include:` + `extends:`, CI/CD Components |
| Prebuilt building blocks | Marketplace actions | CI/CD Catalog components |
| Stages/ordering | `needs:` (DAG) | `stages:` + `needs:` (DAG) |
| Manual trigger | `workflow_dispatch` | `when: manual` + Run pipeline UI |
| Matrix builds | `strategy.matrix` | `parallel:matrix` |
| Caching | `actions/cache` | `cache:` keyword |
| Artifacts | `actions/upload/download-artifact` | `artifacts:` keyword |
| Secrets | Repo/org/environment secrets | CI/CD variables (masked/protected) |
| OIDC to cloud | `id-token: write` + provider action | `id_tokens:` keyword |
| Environments | `environment:` | `environment:` (with protected environments) |

### Why There's No `actions/checkout@v4` in GitLab

On GitHub, almost every job starts with reusable **action blocks** — `uses: actions/checkout@v4`, `uses: actions/setup-node@v4`, and so on. These are versioned, third-party (or first-party) units you pull from the Marketplace with `uses:`. GitLab has **no `uses:` keyword and no equivalent block** you drop in. This is a deliberate design choice, not a missing feature.

```yaml
# GitHub: you compose the job from action blocks
steps:
  - uses: actions/checkout@v4          # clone the repo
  - uses: actions/setup-node@v4        # install Node
    with: { node-version: 20 }
  - run: npm ci

# GitLab: no blocks — the runner clones for you, and the image provides the tools
job:
  image: node:20                        # "setup-node" is just picking the image
  script:
    - npm ci                            # checkout already happened before this ran
```

Why the block model doesn't exist in GitLab:

- **The runner already does the common blocks.** `actions/checkout` exists on GitHub because a job starts empty and must explicitly fetch the code. A GitLab runner **clones (or fetches) the repository automatically** before `script:` runs, so there's nothing to add — the single most-used action block is unnecessary by design. You tune it with the `GIT_STRATEGY`, `GIT_DEPTH`, and `GIT_SUBMODULE_STRATEGY` variables instead of a step.
- **The unit of reuse is the image, not a step.** `actions/setup-node@v4` installs a toolchain into a blank runner. In GitLab you instead pick an `image:` that already contains the toolchain (`node:20`, `python:3.12`, `hashicorp/terraform`). The "block" is baked into the container, so there's no setup action to call.
- **Scripts over black boxes.** An action like `checkout@v4` is opaque JavaScript/Docker you trust to run in your job's context — a real supply-chain surface. GitLab keeps every step as shell you can read in `script:`, so there's less need to import third-party blocks for routine work.
- **When you *do* want reusable blocks, they're `include:`/`extends:`/Components — not `uses:`.** GitLab's answer to "share a building block" is pulling in YAML via `include:`, inheriting job definitions with `extends:`, or adding a versioned **CI/CD Component** from the Catalog (`include: - component: .../build@1.0.0` with `inputs:`). That's the closest analog to a versioned action, but it composes *pipeline config*, not opaque steps.

| GitHub action block | GitLab equivalent |
|---------------------|-------------------|
| `actions/checkout@v4` | Nothing — repo cloned automatically (tune with `GIT_STRATEGY`, `GIT_DEPTH`) |
| `actions/setup-node@v4` (and other `setup-*`) | Set `image:` to a versioned language image |
| A Marketplace action with `with:` inputs | A CI/CD Component: `include: - component: .../name@ver` with `inputs:` |
| Reusable workflow (`uses: org/repo/.github/workflows/x.yml@v1`) | `include:` + `extends:` |

### Minimal Pipeline Side by Side

GitHub Actions:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm test
```

GitLab CI/CD:

```yaml
# .gitlab-ci.yml
test:
  image: node:20
  script:
    - npm ci
    - npm test
```

### The "Action Button": Manual Dispatch with plan/apply/destroy

A common GitHub pattern is a `workflow_dispatch` **dropdown button** that picks which Terraform action to run, then gating each step with `if:`:

```yaml
# GitHub Actions — the "button" is workflow_dispatch inputs
on:
  workflow_dispatch:
    inputs:
      action:
        description: "Terraform action"
        type: choice
        options: [plan, apply, destroy]
        default: plan

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      # ... checkout, setup-terraform, init ...

      - name: Terraform Plan
        if: ${{ github.event.inputs.action == 'plan' }}
        run: terraform plan -input=false

      - name: Terraform Apply
        if: ${{ github.event.inputs.action == 'apply' }}
        run: terraform apply -auto-approve -input=false

      - name: Terraform Destroy
        if: ${{ github.event.inputs.action == 'destroy' }}
        run: terraform destroy -auto-approve -input=false
```

GitLab has **no such dropdown button**. There's no `workflow_dispatch` UI widget that hands you `github.event.inputs.action`. Instead you drive the action through a **pipeline variable** and separate manual jobs. The variable *is* the button:

```yaml
# GitLab CI/CD — the "button" is a pipeline variable + manual jobs
variables:
  # Rendered as a dropdown on the "Run pipeline" screen
  ACTION:
    value: "plan"
    options: ["plan", "apply", "destroy"]
    description: "Terraform action to run"

.tf:
  image: hashicorp/terraform:latest
  before_script:
    - terraform init -input=false

plan:
  extends: .tf
  script:
    - terraform plan -input=false
  rules:
    - if: '$ACTION == "plan"'

apply:
  extends: .tf
  script:
    - terraform apply -auto-approve -input=false
  rules:
    - if: '$ACTION == "apply"'
      when: manual   # requires a deliberate click in the pipeline view

destroy:
  extends: .tf
  script:
    - terraform destroy -auto-approve -input=false
  rules:
    - if: '$ACTION == "destroy"'
      when: manual
```

**Why GitLab doesn't have these buttons — and why the variable approach is the equivalent:**

- **No hidden UI-only state.** On GitHub, `github.event.inputs.action` exists only because a dropdown fed it in; the value isn't part of your config, it's injected by the event. GitLab keeps the choice in a declared CI/CD **variable**, so it's visible in the pipeline's variable list, in job logs, and reproducible on re-run. The action is data you can audit, not a transient button press.
- **Script-plus-image philosophy.** GitLab jobs are just an `image:` plus shell `script:`. Control flow ("which action?") is expressed the same way as everything else — a variable read by `rules:` — rather than a platform-specific dispatch widget.
- **`rules:` replace the `if:` step gates.** Where GitHub attaches `if:` to each step inside one job, GitLab makes each action its own job selected by `rules: - if: '$ACTION == "..."'`. Same outcome, but each action shows up as a distinct, individually retryable job.
- **`when: manual` is the real safety gate.** The destructive actions (`apply`, `destroy`) are manual jobs, so even after the pipeline starts you must explicitly click **play** on that job. Combined with a **protected environment**, this is how GitLab enforces "are you sure?" without an approval button.
- **`options:` gives you the dropdown feel.** Declaring `value`/`options`/`description` on the variable renders an actual dropdown on the **Run pipeline** screen — the closest thing to GitHub's `type: choice` input — while staying a plain variable underneath.

The one-line translation: **`workflow_dispatch` input + `if:` on GitHub → CI/CD variable (`ACTION`) + `rules:`/`when: manual` on GitLab.**

## Permissions and Roles

| GitHub role | Rough GitLab equivalent |
|-------------|-------------------------|
| Read | Guest / Reporter |
| Triage | Reporter |
| Write | Developer |
| Maintain | Maintainer |
| Admin | Owner |

GitLab roles are numeric-tiered (Guest → Reporter → Developer → Maintainer → Owner) and apply at the group and project level, with group roles inherited by subgroups and projects. GitHub uses per-repo roles plus org-level roles and teams.

## Git Remotes and Auth

```bash
# HTTPS clone
git clone https://github.com/org/repo.git
git clone https://gitlab.com/group/subgroup/repo.git

# SSH clone
git clone git@github.com:org/repo.git
git clone git@gitlab.com:group/repo.git

# GitHub: personal access token (classic/fine-grained) as HTTPS password
# GitLab: personal access token, project/group access token, or deploy token
```

| Auth method | GitHub | GitLab |
|-------------|--------|--------|
| Personal token | Personal Access Token (PAT) | Personal Access Token (PAT) |
| Scoped repo token | Fine-grained PAT / deploy key | Project / Group Access Token, Deploy Token |
| CI job token | `GITHUB_TOKEN` (auto) | `CI_JOB_TOKEN` (auto) |
| SSH keys | Per-user | Per-user + per-project deploy keys |

## CLI Tools

| Task | GitHub (`gh`) | GitLab (`glab`) |
|------|---------------|-----------------|
| Auth login | `gh auth login` | `glab auth login` |
| Clone | `gh repo clone org/repo` | `glab repo clone group/repo` |
| Create PR/MR | `gh pr create` | `glab mr create` |
| List PR/MR | `gh pr list` | `glab mr list` |
| Check out PR/MR | `gh pr checkout 123` | `glab mr checkout 123` |
| View pipeline/CI | `gh run list` | `glab ci list` |
| View CI status | `gh run view <id>` | `glab ci view` |
| Create issue | `gh issue create` | `glab issue create` |
| Create release | `gh release create v1.0.0` | `glab release create v1.0.0` |

## REST API Quick Reference

```bash
# --- List repos/projects ---
# GitHub
curl -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/user/repos

# GitLab (projects are referenced by numeric ID or URL-encoded path)
curl -H "PRIVATE-TOKEN: $GL_TOKEN" \
  https://gitlab.com/api/v4/projects

# --- Get a single project/repo ---
# GitHub
curl -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/org/repo

# GitLab (URL-encode the full path group%2Fsubgroup%2Frepo)
curl -H "PRIVATE-TOKEN: $GL_TOKEN" \
  https://gitlab.com/api/v4/projects/group%2Frepo

# --- Create a PR / MR ---
# GitHub
curl -X POST -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/org/repo/pulls \
  -d '{"title":"My PR","head":"feature","base":"main"}'

# GitLab
curl -X POST -H "PRIVATE-TOKEN: $GL_TOKEN" \
  "https://gitlab.com/api/v4/projects/123/merge_requests" \
  -d "source_branch=feature&target_branch=main&title=My MR"
```

| API detail | GitHub | GitLab |
|------------|--------|--------|
| Base path | `https://api.github.com` | `https://gitlab.com/api/v4` |
| Auth header | `Authorization: Bearer <token>` | `PRIVATE-TOKEN: <token>` |
| Project reference | `org/repo` in path | numeric ID or URL-encoded `group%2Frepo` |
| GraphQL | `https://api.github.com/graphql` | `https://gitlab.com/api/graphql` |
| Pagination | `Link` header, `per_page` | `Link` header, `per_page`, `X-Total` |

## Webhooks and Automation

| Feature | GitHub | GitLab |
|---------|--------|--------|
| Webhooks | Repo/org webhooks | Project/group/system hooks |
| Event bots | GitHub Apps | GitLab Apps (OAuth) / integrations |
| Scheduled jobs | Scheduled workflows (cron) | Pipeline schedules |
| Merge automation | Auto-merge, merge queue | Merge trains, auto-merge |
| Code owners | `CODEOWNERS` file | `CODEOWNERS` file (Premium+ enforcement) |

## Features That Don't Map Cleanly

- **Marketplace vs Catalog** — GitHub's Marketplace has a large third-party action ecosystem; GitLab's CI/CD Catalog is smaller and favors your own components plus built-in templates. GitLab leans on container images and built-in platform features instead of third-party steps.
- **Merge trains** (GitLab) — queues MRs and tests them against the projected post-merge state; GitHub's merge queue is the analog but behaves differently.
- **Built-in security scanning** (GitLab) — SAST, DAST, dependency scanning, container scanning, and secret detection ship as one-line includes; on GitHub these are separate features (code scanning/CodeQL, Dependabot, secret scanning).
- **Nested subgroups** (GitLab) — arbitrary depth of groups; GitHub has orgs + teams (teams can nest, orgs cannot).
- **Single-product breadth** (GitLab) — repo, CI/CD, registry, security, and deployment in one app vs GitHub's more modular set of features and Actions ecosystem.

## When to Reach for Which

- Both are excellent for hosting Git repos, PR/MR review, and CI/CD.
- GitHub tends to win on **community/open-source reach** and the breadth of its **Actions Marketplace**.
- GitLab tends to win on **integrated DevOps in one product**, **self-managed/open-source core**, and **transparent script-based pipelines**.

## References

- [GitLab documentation](https://docs.gitlab.com/) — official docs
- [GitHub documentation](https://docs.github.com/) — official docs
- [GitLab REST API](https://docs.gitlab.com/ee/api/rest/) — official docs
- [GitHub REST API](https://docs.github.com/en/rest) — official docs
- [Migrating from GitHub to GitLab](https://docs.gitlab.com/ee/user/project/import/github.html) — official docs

*Content was rephrased for compliance with licensing restrictions.*
