# terraform.tfstate vs .terraform/terraform.tfstate

## Overview

These two files serve completely different purposes in a Terraform workflow, and confusing them is a common source of frustration.

## terraform.tfstate (Root State File)

**Location:** `./terraform.tfstate` (project root, or remote backend)

This is the **actual state file** — the source of truth for what Terraform has provisioned. It maps your declared resources to real-world infrastructure objects (IDs, attributes, dependencies).

- Created after `terraform apply`
- Tracks every resource Terraform manages
- Used by `plan` and `apply` to calculate diffs
- Can live locally (default) or in a remote backend (S3, GCS, Terraform Cloud, etc.)
- Contains sensitive data (resource IDs, outputs, sometimes secrets)

When you configure a remote backend (e.g., S3 + DynamoDB), this file lives remotely and is never stored on disk in the project root.

## .terraform/terraform.tfstate (Backend State Cache)

**Location:** `.terraform/terraform.tfstate`

This is **not** your infrastructure state. It's a small metadata file that records **which backend Terraform is currently configured to use**.

Contents look something like:

```json
{
  "version": 3,
  "serial": 1,
  "backend": {
    "type": "s3",
    "config": {
      "bucket": "my-tf-state",
      "key": "prod/terraform.tfstate",
      "region": "eu-central-1"
    },
    "hash": 284639827463
  }
}
```

- Created/updated during `terraform init`
- Tells Terraform where to find the real state
- Lives inside `.terraform/` which is a local working directory (like `node_modules`)
- The entire `.terraform/` directory should be in `.gitignore` — it contains downloaded providers, modules, and backend metadata (including potentially cached credentials)

## The Confusion

| Aspect | `terraform.tfstate` | `.terraform/terraform.tfstate` |
|--------|---------------------|-------------------------------|
| Purpose | Infrastructure state | Backend configuration cache |
| Created by | `terraform apply` | `terraform init` |
| Contains | Resource mappings, outputs | Backend type and config |
| Sensitive | Yes | Not really |
| In version control | No (if using remote backend) | No (entire `.terraform/` dir is gitignored) |

---

## Why terraform init -reconfigure is required

### The Problem

When you change your backend configuration (switch from local to S3, change the S3 bucket/key, switch between workspaces, etc.), Terraform detects a mismatch between:

1. What your `.tf` files declare as the backend
2. What `.terraform/terraform.tfstate` has cached from the last `init`

Running a plain `terraform init` in this situation produces:

```
Error: Backend configuration changed

A change in the backend configuration has been detected, which may require
migrating existing state.
```

Terraform won't proceed because it doesn't know if you want to **migrate** state from the old backend to the new one, or simply **forget** the old config and point to the new one.

### What -reconfigure does

```bash
terraform init -reconfigure
```

This tells Terraform:

> "I know the backend changed. Don't try to migrate state. Just update `.terraform/terraform.tfstate` to reflect the new backend configuration and move on."

It re-initializes the backend from scratch without attempting any state migration.

### When you need it

- **Switching environments** — You have separate state files per environment and change the backend key (e.g., `dev/terraform.tfstate` → `prod/terraform.tfstate`)
- **CI/CD pipelines** — The `.terraform/` directory may be cached from a previous run with a different backend config
- **After cloning or moving a project** — Stale `.terraform/` metadata from another machine
- **Switching from local to remote backend** (when you don't need to migrate existing state)
- **Backend config is parameterized** via `-backend-config` flags and the values changed between runs
- **After module upgrades that affect backend configuration** (see below)

### Module upgrades and -reconfigure

Yes — module upgrades can trigger the need for `-reconfigure`, though indirectly. Here's how:

**1. Modules that wrap backend configuration**

Some teams use wrapper scripts or generate backend config from module outputs. If upgrading a module changes the expected backend key, bucket, or structure, the cached `.terraform/terraform.tfstate` becomes stale.

**2. Provider/module hash mismatches in `.terraform.lock.hcl`**

When you upgrade modules (via `terraform init -upgrade`), Terraform re-downloads them into `.terraform/modules/`. If the `.terraform/` directory is partially cached (e.g., in CI) and you upgrade modules in your `.tf` files, running `terraform init` may complain about inconsistencies. A `-reconfigure` forces a clean re-initialization.

**3. Module upgrades that change provider requirements**

A new module version might require a different provider version or introduce new providers. If the old `.terraform/` directory has cached provider binaries and lock state from the previous module version, Terraform can get confused. Running:

```bash
terraform init -reconfigure -upgrade
```

clears the backend cache AND pulls fresh modules and providers.

**4. Terragrunt / automation wrappers**

If you use Terragrunt or similar tools that auto-generate backend blocks from module-level config, upgrading the module source can change the generated `backend {}` block. The cached backend metadata then mismatches the new generated config, requiring `-reconfigure`.

### The typical scenario

```
1. You upgrade a module version in your .tf files
2. The module (or your automation) changes backend config slightly
3. You run `terraform init`
4. Error: "Backend configuration changed"
5. Fix: `terraform init -reconfigure` (if state already exists at the new location)
   Or:  `terraform init -migrate-state` (if you need to move state)
```

So module upgrades don't directly change `.terraform/terraform.tfstate`, but they often change the inputs or structure that **generates** the backend configuration, which causes the mismatch.

### -reconfigure vs -migrate-state

| Flag | Behavior |
|------|----------|
| `-reconfigure` | Drops old backend cache, re-initializes fresh. No state migration. |
| `-migrate-state` | Copies state from old backend to new backend, then updates cache. |

Use `-migrate-state` when you're moving an existing project's state to a new location and need to preserve it. Use `-reconfigure` when you're pointing to an already-existing state or starting fresh.

### Practical Example (CI/CD)

```bash
# Backend key is dynamic per environment
terraform init \
  -reconfigure \
  -backend-config="bucket=my-tf-state" \
  -backend-config="key=${ENV}/terraform.tfstate" \
  -backend-config="region=eu-central-1"
```

Without `-reconfigure`, this would fail on the second run if `ENV` changed between pipelines, because the cached backend config no longer matches.

---

## What's Inside the .terraform/ Directory

The `.terraform/` directory is Terraform's local working cache, regenerated by `terraform init`. It contains:

```
.terraform/
├── terraform.tfstate          # Backend metadata (discussed above)
├── providers/                 # Downloaded provider binaries
│   └── registry.terraform.io/
│       └── hashicorp/
│           └── aws/
│               └── 5.x.x/
│                   └── darwin_arm64/
│                       └── terraform-provider-aws_v5.x.x
├── modules/                   # Downloaded module source code
│   ├── modules.json           # Module manifest (maps module calls to paths)
│   └── vpc/                   # Example: downloaded vpc module
└── environment                # Current workspace name (if using workspaces)
```

Key points:
- **providers/** — Cached provider plugin binaries. Can be large (hundreds of MB for AWS provider)
- **modules/** — Source code for external modules fetched from registries or git
- **environment** — A single-line file with the active workspace name (default: `default`)
- Everything here is reproducible via `terraform init` — it's safe to delete the whole directory and re-init

---

## .terraform.lock.hcl (The Dependency Lock File)

Not to be confused with `.terraform/terraform.tfstate`, this file lives in the **project root** (not inside `.terraform/`).

```
.terraform.lock.hcl    ← commit this to version control
.terraform/            ← do NOT commit this
```

- Created on first `terraform init`, records exact provider versions and cryptographic checksums
- Ensures every team member and CI runner uses identical provider versions
- Similar in purpose to `package-lock.json` or `Pipfile.lock`
- **Should be committed to version control** — unlike the `.terraform/` directory
- Updated by `terraform init -upgrade` when you want to pull newer provider versions within your constraints

---

## State Locking (Preventing Concurrent Corruption)

When using a remote backend, Terraform supports **state locking** to prevent two processes from writing to the same state simultaneously.

Without locking, concurrent `terraform apply` runs can corrupt state — both read the current state, compute a diff, and race to write back, with one overwriting the other.

### Common Locking Mechanisms

| Backend | Locking Method |
|---------|---------------|
| S3 | DynamoDB table (or S3 native locking with newer versions) |
| GCS | Built-in object locking |
| Azure Blob | Blob leases |
| Terraform Cloud / HCP | Managed automatically |
| Consul | Session-based locks |

### Example: S3 Backend with DynamoDB Locking

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "eu-central-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

If a lock is already held (e.g., crashed `apply`), you'll see:

```
Error: Error acquiring the state lock
```

Force-unlock with caution:
```bash
terraform force-unlock LOCK_ID
```

---

## State Recovery and Disaster Scenarios

### Recovery Approaches

1. **S3 versioning** — If your state bucket has versioning enabled, restore a previous version
2. **terraform.tfstate.backup** — Terraform creates a local backup before each write (local backend only)
3. **terraform import** — Re-import resources one by one into a fresh state (last resort)
4. **terraform refresh** (deprecated) / `terraform apply -refresh-only` — Sync state with actual infra without making changes

### Prevention

- Always enable versioning on your state bucket
- Always enable state locking
- Use separate state files per environment/component (state isolation)
- Run `terraform plan` in CI before `apply` to catch drift early

---

## Recommended .gitignore for Terraform Projects

```gitignore
# Local .terraform directory (providers, modules, backend cache)
**/.terraform/*

# State files (should be in remote backend, not in repo)
*.tfstate
*.tfstate.*

# Crash log files
crash.log
crash.*.log

# Sensitive variable files (may contain passwords/secrets)
*.tfvars
*.tfvars.json

# Override files (local developer overrides)
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# CLI configuration file
.terraformrc
terraform.rc
```

**Do commit:**
- `.terraform.lock.hcl` (dependency lock file)
- `*.tf` files (your actual configuration)
- `terraform.tfvars.example` (template without real values)

---

## TL;DR

- `terraform.tfstate` = your actual infrastructure state (the important one)
- `.terraform/terraform.tfstate` = a cache of which backend to use (plumbing)
- `.terraform.lock.hcl` = provider version lock file (commit this!)
- `.terraform/` directory = local cache of providers + modules (gitignore this!)
- `terraform init -reconfigure` = "reset the backend cache without migrating state" — required when backend config changes and you don't need to carry old state forward
- Always use remote backends with state locking for team environments
- Enable S3 bucket versioning for disaster recovery
