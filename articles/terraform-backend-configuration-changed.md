# Terraform: Backend Configuration Changed

## The Error

```
Error: Backend configuration changed

A change in the backend configuration has been detected, which may require migrating existing state.

If you wish to attempt automatic migration of the state, use "terraform init -migrate-state".
If you wish to store the current configuration with no changes to the state, use "terraform init -reconfigure".
```

## What This Means

When you run `terraform init` successfully, Terraform saves the backend configuration it used into an internal metadata file: `.terraform/terraform.tfstate`. All subsequent Terraform commands in this working directory use that saved configuration as "the backend configuration."

This error means Terraform found an existing `.terraform/terraform.tfstate` file and noticed it contains different settings than what's currently in your `backend` block. It's asking you how to handle the difference.

## The Two Options

### Option 1: `-migrate-state`

```bash
terraform init -migrate-state
```

This tells Terraform to:

1. Read the state snapshot using the **old** configuration (from `.terraform/terraform.tfstate`)
2. Write it to the **new** location (from your current `backend` block)

**When to use:**

- You changed where the state is stored (different S3 bucket, different key path, different storage account)
- You want to physically move the state from the old location to the new location
- The old credentials still work (Terraform needs to read from the old location)

**Example scenarios:**

- Changed `bucket` from `my-old-bucket` to `my-new-bucket`
- Changed `key` from `dev/terraform.tfstate` to `prod/terraform.tfstate`
- Moving from local backend to S3
- Moving from one Terraform Cloud workspace to another

### Option 2: `-reconfigure`

```bash
terraform init -reconfigure
```

This tells Terraform to discard the saved backend configuration and just use what's in the `backend` block — as if initializing for the first time.

**When to use:**

- You changed authentication settings (profile, role ARN, SSO, credentials)
- You changed settings that don't affect where the state is physically stored
- The old credentials are no longer valid
- You just want Terraform to use the current configuration and stop complaining

**Example scenarios:**

- Changed `profile` from `old-profile` to `new-sso-profile`
- Changed `role_arn` to a new IAM role
- Added or changed `region` without moving the bucket
- Switched from access keys to SSO
- Changed `encrypt`, `dynamodb_table`, or `workspace_key_prefix`

### Alternative: Delete .terraform

Instead of `-reconfigure`, you can achieve the same result by deleting the cached state:

```bash
# Delete just the metadata file
rm .terraform/terraform.tfstate

# Or delete the entire .terraform directory (also removes providers/modules cache)
rm -rf .terraform

# Then run init normally
terraform init
```

This forces Terraform to initialize fresh with no memory of previous configuration.

## How to Decide

```
Did the physical location of the state change?
(bucket, key, container, storage account, path)

├── YES → terraform init -migrate-state
│         (moves state from old location to new)
│
└── NO → terraform init -reconfigure
          (just use the new config, state hasn't moved)
```

### Common Backend Changes and Which Flag to Use

| What Changed | State Moved? | Use |
|---|---|---|
| `bucket` | Yes | `-migrate-state` |
| `key` | Yes | `-migrate-state` |
| `profile` | No | `-reconfigure` |
| `role_arn` | No | `-reconfigure` |
| `region` (same bucket) | No | `-reconfigure` |
| `encrypt` | No | `-reconfigure` |
| `dynamodb_table` | No | `-reconfigure` |
| `workspace_key_prefix` | Depends | `-reconfigure` (unless switching workspaces) |
| `access_key` / `secret_key` | No | `-reconfigure` |
| `shared_credentials_file` | No | `-reconfigure` |
| Local → S3 | Yes | `-migrate-state` |
| S3 → S3 (different bucket) | Yes | `-migrate-state` |
| Terraform Cloud → S3 | Yes | `-migrate-state` |

## Understanding .terraform/terraform.tfstate

This is NOT your actual infrastructure state. It's a metadata file that records:

- Which backend Terraform is configured to use
- The backend type and its settings
- A pointer to where the real state lives

```bash
# View the backend metadata
cat .terraform/terraform.tfstate | python3 -m json.tool
```

Example content:

```json
{
    "version": 3,
    "serial": 1,
    "lineage": "abc123",
    "backend": {
        "type": "s3",
        "config": {
            "bucket": "my-terraform-state",
            "key": "prod/terraform.tfstate",
            "region": "eu-west-1",
            "profile": "old-profile"
        },
        "hash": 1234567890
    }
}
```

When you change your `backend "s3" {}` block, Terraform compares the hash of the new config against the saved hash. If they differ, you get the "Backend configuration changed" error.

## Practical Example: Switching to SSO

You had:

```hcl
terraform {
  backend "s3" {
    bucket  = "my-terraform-state"
    key     = "prod/terraform.tfstate"
    region  = "eu-west-1"
    profile = "old-iam-user"
  }
}
```

You changed to:

```hcl
terraform {
  backend "s3" {
    bucket  = "my-terraform-state"
    key     = "prod/terraform.tfstate"
    region  = "eu-west-1"
    profile = "my-sso-profile"
  }
}
```

The state hasn't moved (same bucket, same key). Only the authentication method changed. Use:

```bash
terraform init -reconfigure
```

`-migrate-state` would try to read from the old location (using the old profile that no longer works) and write to the new location (which is the same place) — pointless at best, fails at worst.

## Practical Example: Moving State to a New Bucket

You had:

```hcl
terraform {
  backend "s3" {
    bucket = "old-bucket"
    key    = "terraform.tfstate"
    region = "us-east-1"
  }
}
```

You changed to:

```hcl
terraform {
  backend "s3" {
    bucket = "new-bucket"
    key    = "terraform.tfstate"
    region = "eu-west-1"
  }
}
```

The state needs to move from `old-bucket` to `new-bucket`. Use:

```bash
terraform init -migrate-state
```

Terraform reads state from `old-bucket` and writes it to `new-bucket`.

## Tips

- **When in doubt, use `-reconfigure`** if you know the state file hasn't moved
- **Always ensure you have a backup** of your state before migration
- **`-migrate-state` requires access to both old and new backends** at the time of running
- **If `-migrate-state` fails** (e.g., old credentials expired), use `-reconfigure` — the state is still at the same location, just accessed differently
- **CI/CD pipelines** should typically use `-reconfigure` (or `rm -rf .terraform`) since they initialize fresh each run
- **The `.terraform` directory should be in `.gitignore`** — it's local cache, not shared configuration
