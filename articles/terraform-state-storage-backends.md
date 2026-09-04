# Where Terraform Can Store State: A Guide to Backends

Terraform tracks everything it manages in a **state file** (`terraform.tfstate`). This file maps your configuration to real-world resources, stores metadata, and caches attribute values. Because it can contain sensitive data (passwords, keys, IPs) and is the source of truth for your infrastructure, *where* and *how* you store it matters a lot.

This article walks through the main options: the cloud providers (AWS, Azure, GCP), self-hosted/homelab setups, and a few others worth knowing.

---

## Why State Storage Matters

Before diving into backends, here's what a good remote state setup gives you:

- **Collaboration** – Multiple people/CI pipelines share one authoritative state.
- **Locking** – Prevents two `apply` runs from corrupting state at the same time.
- **Durability** – State survives a laptop wipe or a deleted CI runner.
- **Security** – Encryption at rest and access control for sensitive values.
- **Versioning** – Roll back to a previous state if something goes wrong.

The default is a **local backend** (a file on disk). That's fine for learning and one-person experiments, but for anything shared or production-grade you want a **remote backend**.

---

## 1. Local Backend (Default)

State lives as a plain file on your machine.

```hcl
terraform {
  backend "local" {
    path = "relative/path/to/terraform.tfstate"
  }
}
```

**Pros**
- Zero setup, works out of the box.
- Fast, no network dependency.

**Cons**
- No locking across machines.
- No built-in encryption (it's plaintext JSON).
- Easy to lose; hard to share.
- Sensitive data sits unprotected on disk.

**Use it for:** learning, throwaway experiments, and quick local testing only.

---

## 2. AWS – S3 Backend (+ DynamoDB / native locking)

The S3 backend is the most common choice in AWS shops. State is stored as an object in an S3 bucket.

```hcl
terraform {
  backend "s3" {
    bucket       = "my-terraform-state"
    key          = "prod/network/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true          # native S3 locking (Terraform 1.10+)
    # dynamodb_table = "terraform-locks"   # legacy locking approach
  }
}
```

**State locking options**
- **S3 native lockfile** (`use_lockfile = true`) – Available in Terraform 1.10+; no separate table needed.
- **DynamoDB table** – The long-standing approach; a table with a `LockID` partition key holds the lock. Still widely used and required on older versions.

**Recommended bucket settings**
- Enable **versioning** so you can recover previous state.
- Enable **server-side encryption** (SSE-S3 or SSE-KMS).
- Block all public access.
- Use IAM policies to tightly scope who can read/write the state key.

**Pros**
- Highly durable and available.
- Fine-grained IAM access control and KMS encryption.
- Versioning for rollback.

**Cons**
- Requires provisioning the bucket (and optionally DynamoDB) first — a chicken-and-egg bootstrap step.

---

## 3. Azure – azurerm Backend (Blob Storage)

Azure uses a Storage Account container to hold the state blob. Locking is handled automatically via blob leases (no extra service needed).

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "tfstatestore1234"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

**Authentication options**
- Azure CLI (`az login`) for local dev.
- Service Principal with client ID/secret.
- Managed Identity (great for CI running in Azure).
- SAS token or storage access key.

**Recommended settings**
- Enable **blob versioning** and soft delete for recovery.
- Encryption at rest is on by default (Microsoft-managed keys), and you can bring your own key with Key Vault.
- Restrict access with RBAC and network rules/private endpoints.

**Pros**
- Built-in state locking via blob leases (no separate lock table).
- Encryption on by default.
- Integrates cleanly with Azure AD / Managed Identity.

**Cons**
- Storage account naming rules are strict (globally unique, lowercase, no dashes).

---

## 4. GCP – gcs Backend (Cloud Storage)

Google Cloud Storage buckets store the state object, with automatic locking.

```hcl
terraform {
  backend "gcs" {
    bucket = "my-tf-state-bucket"
    prefix = "prod/network"
  }
}
```

**Authentication options**
- Application Default Credentials (`gcloud auth application-default login`).
- Service account key file via `GOOGLE_APPLICATION_CREDENTIALS`.
- Workload Identity for GKE / CI.

**Recommended settings**
- Enable **object versioning** for rollback.
- Use **customer-managed encryption keys (CMEK)** via Cloud KMS if you need control beyond Google-managed encryption (which is on by default).
- Lock down with uniform bucket-level IAM.

**Pros**
- Automatic locking built in.
- Strong consistency and default encryption.
- Simple `prefix`-based layout for multiple states.

**Cons**
- Bucket names are globally unique.

---

## 5. Self-Hosted / Homelab Options

If you're running a homelab or on-prem environment (Proxmox, Kubernetes, NAS, bare metal), you have several solid choices. Most of them work by mimicking cloud APIs or using HTTP.

### 5a. MinIO (S3-compatible object storage)

MinIO speaks the S3 API, so you can reuse the **s3 backend** by pointing it at your MinIO endpoint.

```hcl
terraform {
  backend "s3" {
    bucket                      = "terraform-state"
    key                         = "homelab/terraform.tfstate"
    region                      = "us-east-1"   # arbitrary, but required
    endpoints                   = { s3 = "https://minio.homelab.local:9000" }
    use_path_style              = true
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    use_lockfile                = true          # native locking, Terraform 1.10+
  }
}
```

**Great for:** homelabbers who already run MinIO for object storage. You get S3-style workflows, versioning, and encryption on your own hardware.

> Note: exact skip/endpoint options vary slightly by Terraform version. On older versions, use `endpoint` (singular) and provide a DynamoDB-compatible lock or rely on the newer `use_lockfile`.

### 5b. HTTP Backend (custom or self-hosted state server)

The HTTP backend stores state via REST calls to any server that implements the simple GET/POST/LOCK/UNLOCK contract.

```hcl
terraform {
  backend "http" {
    address        = "https://state.homelab.local/states/prod"
    lock_address   = "https://state.homelab.local/states/prod/lock"
    unlock_address = "https://state.homelab.local/states/prod/lock"
    lock_method    = "PUT"
    unlock_method  = "DELETE"
    username       = "tfuser"
    # password via TF_HTTP_PASSWORD env var
  }
}
```

**Servers that implement this:**
- **GitLab-managed Terraform state** (even self-hosted GitLab CE/EE) – arguably the easiest turnkey option if you already run GitLab in your homelab.
- **Atlantis** and various small open-source state servers.
- Roll-your-own with a lightweight app.

### 5c. Kubernetes Backend

If your homelab runs Kubernetes, state can live in a Kubernetes **Secret**.

```hcl
terraform {
  backend "kubernetes" {
    secret_suffix = "prod"
    namespace     = "terraform"
    # in-cluster config used automatically when running inside the cluster
  }
}
```

**Pros:** no extra storage service; leverages etcd. **Cons:** not ideal for very large state; etcd size limits apply; back up etcd.

### 5d. PostgreSQL Backend

A self-hosted Postgres instance can store state, with locking via Postgres advisory locks.

```hcl
terraform {
  backend "pg" {
    conn_str = "postgres://tfuser:password@db.homelab.local/terraform_state?sslmode=require"
  }
}
```

**Great for:** homelabs already running Postgres. Built-in locking and easy backups with standard DB tooling.

### 5e. Plain NFS / Local File on a NAS

You *can* put the local backend file on an NFS mount or Samba share on your NAS. It's simple but risky: NFS locking is unreliable and there's no encryption. Prefer MinIO or Postgres over this if you can.

---

## 6. Other Notable Backends

### HCP Terraform / Terraform Enterprise (`cloud` block)

HashiCorp's managed (or self-hosted Enterprise) offering. Handles state, locking, remote runs, policy, and a UI.

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "prod-network"
    }
  }
}
```

**Pros:** state + remote execution + RBAC + audit + secrets, all managed. **Cons:** cost at scale; free tier is limited.

### OpenTofu note

[OpenTofu](https://opentofu.org/) (the open-source Terraform fork) supports the same set of backends with the same configuration syntax, plus **state encryption** as a first-class feature — worth knowing if encryption-at-rest by the tool itself is a requirement.

---

## Quick Comparison

| Backend | Platform | Locking | Encryption at Rest | Best For |
|---|---|---|---|---|
| local | Any | No | No | Learning, throwaway |
| s3 | AWS | Native lockfile or DynamoDB | Yes (SSE/KMS) | AWS production |
| azurerm | Azure | Blob lease (built-in) | Yes (default) | Azure production |
| gcs | GCP | Built-in | Yes (default/CMEK) | GCP production |
| s3 → MinIO | Homelab | Native lockfile | Yes (configurable) | Self-hosted S3 workflow |
| http (GitLab) | Homelab/On-prem | Yes | Depends on server | GitLab-based teams |
| kubernetes | Homelab/K8s | Yes | etcd encryption | Small state in a cluster |
| pg | Homelab/Any | Advisory locks | DB-level | Postgres-based homelabs |
| cloud (HCP/TFE) | Managed/Enterprise | Yes | Yes | Teams wanting turnkey |

---

## Best Practices (Regardless of Backend)

1. **Always use a remote backend for anything shared.** Local files don't scale past one person.
2. **Enable locking.** Corrupted state from concurrent runs is painful to fix.
3. **Enable versioning** on the storage so you can roll back a bad state.
4. **Encrypt at rest** and restrict access — state contains secrets in plaintext.
5. **Separate state per environment** (dev/staging/prod) using different keys/prefixes/workspaces.
6. **Never commit `terraform.tfstate` to Git.** Add it to `.gitignore`.
7. **Bootstrap carefully.** The bucket/table/DB that holds state often has to be created before the backend can use it — create it manually or in a separate "bootstrap" configuration.
8. **Back up self-hosted state** (etcd, Postgres, MinIO) just like any other critical data.

---

## Summary

- **Just learning?** Local backend is fine.
- **On AWS/Azure/GCP?** Use the native object-storage backend (s3, azurerm, gcs) with versioning, encryption, and locking.
- **Homelab?** MinIO (S3-compatible), self-hosted GitLab state (http), Kubernetes Secrets, or PostgreSQL are all solid — pick the one that matches services you already run.
- **Want a turnkey managed experience?** HCP Terraform / Terraform Enterprise.

Whatever you choose, the golden rules are the same: remote, locked, versioned, encrypted, and access-controlled.
