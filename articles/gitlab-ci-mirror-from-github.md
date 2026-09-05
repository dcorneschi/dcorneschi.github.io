# Continuously Mirror a GitHub Repository into GitLab (CI/CD)

A focused guide to two related operations:

1. **Cloning** a repository from GitHub.
2. **Continuously mirroring** it into GitLab using a scheduled CI/CD pipeline
   (a pull-based sync that works on GitLab Free, without the premium pull-mirror
   feature).

---

## Part 1: Clone from GitHub

### Normal working clone

```bash
git clone https://github.com/USER/REPO.git
cd REPO
```

For a private repo, use a personal access token:

```bash
git clone https://TOKEN@github.com/USER/REPO.git
```

### Mirror clone (all branches, tags, and refs)

Use a bare `--mirror` clone when the goal is to copy the entire repository
faithfully (this is what the sync pipeline below uses):

```bash
git clone --mirror https://github.com/USER/REPO.git repo.git
cd repo.git
```

To refresh a mirror clone with the latest from GitHub:

```bash
git remote update --prune
```

### Import via the GitLab web UI

GitLab can pull the repository server-side, without using the CLI at all. There
are two web import methods:

**Repository by URL (recommended for public repos, no token):**

1. **New project -> Import project -> Repository by URL**
2. Git repository URL: `https://github.com/USER/REPO.git`
3. Leave username/password blank for a public repo (for a private repo, provide
   a GitHub token as the password).
4. Set the project name/namespace and click **Create project**.

**GitHub importer (requires a token, even for public repos):**

1. **New project -> Import project -> GitHub**
2. Provide a GitHub personal access token — the importer always needs one to
   list your repositories via the GitHub API, even for public repos
   (use no scopes or `public_repo` for public; `repo` for private).
3. Select the repository to import.

If you see **"No import options available"**, an administrator must enable the
import sources under **Admin Area -> Settings -> General -> Import and export
settings**.

> Note: web import is a **one-time copy**. To keep GitLab updated from GitHub
> afterward, use the scheduled CI/CD mirror in Part 2.

---

## Part 2: Scheduled CI/CD mirror (GitHub -> GitLab)

This runs entirely inside GitLab. A scheduled pipeline clones from GitHub and
pushes the refs into the current GitLab project. It works on GitLab Free, where
the built-in **Pull** mirroring option is greyed out.

### Prerequisites

- The destination GitLab project (see the important note about creating it
  **empty** in Step 4).
- A **project access token** (Settings -> Access Tokens) with the
  `write_repository` scope and the **Maintainer** role.

### Step 1: Add the token as a CI/CD variable

**Settings -> CI/CD -> Variables -> Add variable**

- Key: `GITLAB_SYNC_TOKEN`
- Value: the project access token
- Flags: **Masked** (and **Protected** only if you run on protected branches)

For a **private** GitHub source, also add a `GITHUB_TOKEN` variable and use
`https://${GITHUB_TOKEN}@github.com/USER/REPO.git` in the clone URL.

### Step 2: Add the pipeline job

Commit this `.gitlab-ci.yml` to the **default branch** of the GitLab project:

```yaml
stages:
  - sync

sync-from-github:
  stage: sync                                   # a normal stage, not .pre/.post
  image: alpine:latest
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'   # automated schedule
    - if: '$CI_PIPELINE_SOURCE == "web"'        # manual "Run pipeline" button
  variables:
    GIT_STRATEGY: none                          # we manage git ourselves
  before_script:
    - apk add --no-cache git
  script:
    - git clone --mirror https://github.com/USER/REPO.git repo.git
    - cd repo.git
    - git push --mirror "https://oauth2:${GITLAB_SYNC_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git"
```

Notes on the rules:

- Only `schedule` and `web` sources run the job. `push` is intentionally
  excluded so the mirror push does not trigger another pipeline (no loop).
- The **CI Lint / pipeline simulation** simulates a `push` event, so it will
  always report an **empty pipeline** for this config. That is expected — test
  with the schedule or the **Run pipeline** button, not the simulator.

### Step 3: Create a pipeline schedule

**Build -> Pipeline schedules -> New schedule**

- Interval pattern (cron), e.g. `*/30 * * * *` for every 30 minutes.
- Target branch: the default branch (where `.gitlab-ci.yml` lives).
- Save, then use the **play (>)** button to run it on demand.

### Step 4: Make the destination repo empty (important)

`git push --mirror` force-pushes. If the GitLab repo already has commits with
different history than GitHub (for example an initial README commit, or a prior
import), the push is non-fast-forward and GitLab rejects it on a protected
branch:

```
remote: GitLab: You are not allowed to force push code to a protected branch on this project.
 ! [remote rejected] main -> main (pre-receive hook declined)
```

**Recommended fix:** make the GitLab destination an **empty** repository so the
first mirror push has nothing to conflict with.

1. Delete the GitLab project (if it has no independent commits worth keeping).
2. Create a new **blank** project — do **not** add a README, `.gitignore`, or
   license.
3. Re-run the sync pipeline. With an empty target, `--mirror` succeeds and no
   force push is needed. Subsequent syncs stay fast-forward while GitHub history
   is linear.

Alternatives if you cannot recreate the repo empty:

- Enable **Settings -> Repository -> Protected branches -> `main` -> Allowed to
  force push**.
- Or replace the mirror push with a non-force push of branches and tags (does
  not prune deleted branches, fails on rewritten history):

  ```yaml
  - git push "https://oauth2:${GITLAB_SYNC_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git" "refs/heads/*:refs/heads/*" "refs/tags/*:refs/tags/*"
  ```

### Step 5: Do not store the pipeline config in the mirrored repo (important)

`git push --mirror` makes the GitLab repo an **exact** copy of GitHub. Any file
that exists only in the GitLab repo — including `.gitlab-ci.yml` — is
**deleted** on the next sync, because GitHub does not have it. If you put the
pipeline config in the same project you are mirroring into, it disappears the
first time the pipeline runs.

Choose one of these layouts so the config survives:

**Option A — Run the sync from a separate "mirror controller" project (recommended)**

Keep the pipeline in a small, dedicated GitLab project that is **not** itself
mirrored. Its job pushes into the target project by full URL:

```yaml
stages:
  - sync

sync-from-github:
  stage: sync
  image: alpine:latest
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
    - if: '$CI_PIPELINE_SOURCE == "web"'
  variables:
    GIT_STRATEGY: none
  before_script:
    - apk add --no-cache git
  script:
    - git clone --mirror https://github.com/USER/REPO.git repo.git
    - cd repo.git
    - git push --mirror "https://oauth2:${GITLAB_SYNC_TOKEN}@gitlab.example.com/USER/REPO.git"
```

The schedule lives on the controller project; the target repo is never touched
except by the mirror push. One controller can sync many repositories.

**Option B — Put `.gitlab-ci.yml` in the GitHub source repo**

If the config lives in GitHub, the mirror preserves it (it becomes part of the
mirrored content). Simple, but it adds GitLab-specific config to the GitHub repo.

**Option C — Point to an external CI config file**

In the target project: **Settings -> CI/CD -> General pipelines -> CI/CD
configuration file** -> set a path in another project, e.g.
`.gitlab-ci.yml@USER/repo-mirror`. The config is then not stored in the mirrored
repo at all.

---

## Troubleshooting checklist

| Symptom | Cause | Fix |
|---|---|---|
| "The resulting pipeline would have been empty" in CI Lint | Simulator tests a `push` event, which the rules exclude | Expected — trigger via schedule or Run pipeline instead |
| Pipeline runs but has no jobs | Job was in the special `.pre`/`.post` stage | Use a normal stage (e.g. `sync`) |
| Manual "Run pipeline" produces nothing | Missing the `web` rule, or `.gitlab-ci.yml` not on the target branch | Add `if: '$CI_PIPELINE_SOURCE == "web"'`; commit the file to the target branch |
| `not allowed to force push code to a protected branch` | Divergent history; `--mirror` force-push rejected | Recreate the GitLab repo empty, or allow force push, or use a non-force push |
| `authentication failed` | `GITLAB_SYNC_TOKEN` missing/incorrect scope | Use a token with `write_repository` and Maintainer role |
| `could not read Username for github.com` | Private GitHub source without credentials | Add `GITHUB_TOKEN` and use it in the clone URL |

## Notes

- `--mirror` keeps GitLab identical to GitHub, including pruning branches
  removed upstream. Use it only when GitHub is the single source of truth.
- This is schedule-based, so it is not instant on every GitHub commit.
- Tokens belong in masked CI/CD variables, never committed to the repository.
