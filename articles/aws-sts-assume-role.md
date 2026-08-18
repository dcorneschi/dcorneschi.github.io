# AWS STS Assume Role with MFA

Using short-lived temporary credentials instead of long-lived access keys. By restricting your IAM user to only `sts:AssumeRole` with MFA required, a leaked key becomes useless without the physical MFA device.

## Why Temporary Credentials

| Setup | Key leaks — attacker gets |
|-------|--------------------------|
| Access key with admin policy | Full admin access, immediately |
| Access key with only `sts:AssumeRole` + MFA required | Nothing useful (can't pass MFA) |

The access key becomes a low-privilege ticket that only works when combined with MFA:

- Your access key only has permission to call `sts:AssumeRole`
- To get admin privileges, you must also provide an MFA code
- Even if the key is leaked, it's useless without the MFA device

To eliminate access keys entirely, use **IAM Identity Center (SSO)** — browser login, no keys stored on disk.

## Setup

User `jsmith` has read-only access by default. To get admin privileges temporarily, they assume a role with the `AdministratorAccess` policy attached.

### Prerequisites

A role (e.g., `AdminRole`) must exist with:
- The managed policy `arn:aws:iam::aws:policy/AdministratorAccess` attached
- A trust policy allowing `jsmith` to assume it

Verify in the console: **IAM** → **Roles** → `AdminRole` → **Permissions** tab shows `AdministratorAccess`, **Trust relationships** tab lists `jsmith` as a trusted principal.

### Step 1 — Allow the User to Assume the Role

1. Go to **IAM** → **Users** → select `jsmith`
2. Click **Permissions** → **Add permissions** → **Add inline policy**
3. Switch to **JSON** and paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::123456789012:role/AdminRole"
    }
  ]
}
```

4. Name it `AllowAssumeAdminRole` and create

Alternatively, attach the policy at the **group** level so all members can assume the role.

### Step 2 — Assume the Role

Save as `assume-admin.sh`:

```bash
#!/bin/bash
# IMPORTANT: This script must be sourced, not executed directly.
# Usage: source ./assume-admin.sh

if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
  echo "Error: This script must be sourced to export credentials to your shell."
  echo "Usage: source ./assume-admin.sh"
  exit 1
fi

ROLE_ARN="arn:aws:iam::123456789012:role/AdminRole"
MFA_ARN="arn:aws:iam::123456789012:mfa/jsmith"

read -p "MFA code: " MFA_CODE

CREDS=$(aws sts assume-role \
  --role-arn "$ROLE_ARN" \
  --role-session-name jsmith-admin \
  --serial-number "$MFA_ARN" \
  --token-code "$MFA_CODE" \
  --output json)

if [ $? -ne 0 ]; then
  echo "Failed to assume role"
  return 1
fi

export AWS_ACCESS_KEY_ID=$(echo "$CREDS" | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo "$CREDS" | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo "$CREDS" | jq -r '.Credentials.SessionToken')

echo "Admin session active. Expires: $(echo "$CREDS" | jq -r '.Credentials.Expiration')"
aws sts get-caller-identity
```

Usage:

```bash
source ./assume-admin.sh
```

### Create an Alias

Add to `~/.bashrc` or `~/.zshrc`:

```bash
alias assume-admin='source /path/to/assume-admin.sh'
```

Then just run:

```bash
assume-admin
```

### Or Use a Named Profile (Simpler)

In `~/.aws/config`:

```ini
[profile admin]
role_arn = arn:aws:iam::123456789012:role/AdminRole
source_profile = default
mfa_serial = arn:aws:iam::123456789012:mfa/jsmith
```

Then:

```bash
aws s3 ls --profile admin
# CLI prompts for MFA code automatically
```

## Session Duration

By default sessions last **1 hour**. To increase, raise the role's `MaxSessionDuration` first:

```bash
# Check current max session duration
aws iam get-role --role-name AdminRole --query 'Role.MaxSessionDuration'

# Increase to 4 hours (14400 seconds), max is 12 hours (43200)
aws iam update-role --role-name AdminRole --max-session-duration 14400
```

Then pass `--duration-seconds` when assuming:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/AdminRole \
  --role-session-name jsmith-admin \
  --duration-seconds 14400 \
  --serial-number arn:aws:iam::123456789012:mfa/jsmith \
  --token-code 123456
```

| Parameter | Default | Maximum |
|-----------|---------|---------|
| `--duration-seconds` | 3600 (1h) | 43200 (12h) — set via `MaxSessionDuration` on the role |
| Role chaining duration | 3600 (1h) | 3600 (1h) — cannot be increased |
| External ID | optional | Required for third-party cross-account access (confused deputy prevention) |

## Role Chaining

Assume role A, then from role A assume role B. Each hop is limited to **1 hour maximum** (AWS-imposed limit on chained sessions).

```bash
# First hop
eval $(aws sts assume-role --role-arn arn:aws:iam::222222222222:role/RoleA \
  --role-session-name hop1 --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
  --output text | awk '{printf "export AWS_ACCESS_KEY_ID=%s AWS_SECRET_ACCESS_KEY=%s AWS_SESSION_TOKEN=%s", $1, $2, $3}')

# Second hop (from RoleA's session)
eval $(aws sts assume-role --role-arn arn:aws:iam::333333333333:role/RoleB \
  --role-session-name hop2 --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
  --output text | awk '{printf "export AWS_ACCESS_KEY_ID=%s AWS_SECRET_ACCESS_KEY=%s AWS_SESSION_TOKEN=%s", $1, $2, $3}')
```

## Clear Assumed Role Credentials

To return to your default profile:

```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
aws sts get-caller-identity
```
