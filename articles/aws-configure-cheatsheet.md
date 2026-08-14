# aws configure Cheatsheet

## Basic Commands

```bash
# Configure default profile (interactive)
aws configure

# Configure a named profile
aws configure --profile myprofile

# List all profiles
aws configure list-profiles

# Show current profile config
aws configure list

# Show specific profile config
aws configure list --profile myprofile
```

---

## Config Files

| File | Purpose |
|------|---------|
| `~/.aws/credentials` | Access keys (key ID, secret, session token) |
| `~/.aws/config` | Region, output format, role assumptions |

### Why Two Files?

AWS separates credentials from configuration intentionally:

- **`~/.aws/credentials`** — contains secrets (access keys, session tokens). Should have tight permissions (`chmod 600`), excluded from dotfile repos and backups.
- **`~/.aws/config`** — contains non-sensitive settings (region, output format, role ARNs, SSO URLs). Safe to commit to a dotfile repo or share with teammates.

This separation means you can:
- Back up `~/.aws/config` without risking credential exposure
- Use different permission models for each file
- Share config patterns across a team without sharing secrets

```bash
# Recommended permissions
chmod 600 ~/.aws/credentials
chmod 644 ~/.aws/config
```

---

## ~/.aws/credentials

```ini
[default]
aws_access_key_id     = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[staging]
aws_access_key_id     = AKIAI44QH8DHBEXAMPLE
aws_secret_access_key = je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY
```

---

## ~/.aws/config

```ini
# Basic profile
[default]
region = us-east-1
output = json

# Named profile
[profile staging]
region = eu-west-1
output = yaml

# Profile that assumes a role
[profile admin]
role_arn       = arn:aws:iam::123456789012:role/AdminRole
source_profile = default
region         = us-east-1

# Profile that assumes a role with MFA
[profile admin-mfa]
role_arn           = arn:aws:iam::123456789012:role/AdminRole
source_profile     = default
mfa_serial         = arn:aws:iam::123456789012:mfa/myuser
region             = us-east-1

# SSO profile
[profile sso-admin]
sso_start_url  = https://mycompany.awsapps.com/start
sso_account_id = 123456789012
sso_role_name  = AdministratorAccess
region         = us-east-1
output         = json
```

---

## Get/Set Individual Values

```bash
# Get a value
aws configure get region
aws configure get region --profile myprofile

# Set a value
aws configure set region us-west-2
aws configure set region eu-west-1 --profile myprofile
aws configure set output table --profile myprofile

# Set credentials
aws configure set aws_access_key_id AKIAIOSFODNN7EXAMPLE --profile myprofile
aws configure set aws_secret_access_key wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY --profile myprofile
```

---

## Environment Variables

Environment variables override profile settings entirely.

```bash
export AWS_PROFILE=myprofile                  # Use a named profile
export AWS_DEFAULT_REGION=us-east-1           # Override region
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_SESSION_TOKEN=AQoXnyc...           # Required for temporary credentials

# Unset to go back to profile-based auth
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN
unset AWS_PROFILE
```

### Precedence (highest to lowest)

```
1. CLI flags           (--region, --profile)
2. Environment vars    (AWS_ACCESS_KEY_ID, AWS_DEFAULT_REGION, ...)
3. ~/.aws/credentials
4. ~/.aws/config
5. Instance metadata   (EC2/ECS/Lambda IAM role)
```

---

## Assume Role

### Via config (automatic)

```ini
# ~/.aws/config
[profile admin]
role_arn       = arn:aws:iam::123456789012:role/AdminRole
source_profile = default
region         = us-east-1
```

```bash
aws sts get-caller-identity --profile admin
# AWS automatically assumes the role
```

### Via CLI (manual, exports to env)

```bash
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/AdminRole \
  --role-session-name mysession \
  --output json)

export AWS_ACCESS_KEY_ID=$(echo "$CREDS" | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo "$CREDS" | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo "$CREDS" | jq -r '.Credentials.SessionToken')
```

### With MFA

```bash
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/AdminRole \
  --role-session-name mysession \
  --serial-number arn:aws:iam::123456789012:mfa/myuser \
  --token-code 123456 \
  --output json)
```

### With duration_seconds

The default session duration is **1 hour** (3600 seconds). You can extend it up to the role's maximum session duration (configurable up to 12 hours):

```ini
# ~/.aws/config — extend to 4 hours
[profile long-session]
role_arn         = arn:aws:iam::123456789012:role/CIRole
source_profile   = default
duration_seconds = 14400
region           = us-east-1
```

```bash
# Via CLI
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/CIRole \
  --role-session-name ci-build \
  --duration-seconds 14400 \
  --output json)
```

**Note:** The role itself must allow the requested duration. Check with:

```bash
aws iam get-role --role-name CIRole --query 'Role.MaxSessionDuration'
# Default is 3600, max configurable is 43200 (12 hours)
```

### With external_id (cross-account)

Common in multi-account setups where the trust policy requires an external ID to prevent the [confused deputy problem](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html):

```ini
# ~/.aws/config
[profile vendor-account]
role_arn    = arn:aws:iam::999888777666:role/VendorAccessRole
external_id = UniqueExternalId123
source_profile = default
region      = us-east-1
```

```bash
# Via CLI
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::999888777666:role/VendorAccessRole \
  --role-session-name cross-account \
  --external-id UniqueExternalId123 \
  --output json)
```

The trust policy on the target role requires the external ID:

```json
{
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "UniqueExternalId123"
    }
  }
}
```

---

## Credential Process

For teams using external credential managers, `credential_process` allows any executable to provide credentials dynamically:

```ini
# ~/.aws/config

# Using aws-vault
[profile vault-prod]
credential_process = aws-vault exec prod --json

# Using saml2aws
[profile saml-prod]
credential_process = saml2aws login --skip-prompt --quiet --credential-process --role arn:aws:iam::123456789012:role/ProdRole

# Using a custom script
[profile custom]
credential_process = /usr/local/bin/fetch-creds.sh --account prod
```

The script/tool must output JSON in this format:

```json
{
  "Version": 1,
  "AccessKeyId": "AKIAIOSFODNN7EXAMPLE",
  "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  "SessionToken": "AQoXnyc...",
  "Expiration": "2024-01-15T12:00:00Z"
}
```

Common tools that support `credential_process`:

| Tool | Use Case |
|------|----------|
| `aws-vault` | Encrypted local credential storage + role assumption |
| `saml2aws` | SAML-based SSO (Okta, ADFS, Ping) |
| `granted` | Fast profile switching with credential_process |
| `aws-sso-util` | Enhanced AWS SSO credential management |
| Custom scripts | Internal IdP integrations, HashiCorp Vault |

---

## Credential Source and Caching

### credential_source (for EC2/ECS roles)

When running on AWS infrastructure and you need to chain role assumptions from the instance role:

```ini
# ~/.aws/config — assume a role using the EC2 instance role as source
[profile cross-account-from-ec2]
role_arn          = arn:aws:iam::999888777666:role/CrossAccountRole
credential_source = Ec2InstanceMetadata
region            = us-east-1

# From ECS task role
[profile cross-account-from-ecs]
role_arn          = arn:aws:iam::999888777666:role/CrossAccountRole
credential_source = EcsContainer
region            = us-east-1

# From environment variables
[profile cross-account-from-env]
role_arn          = arn:aws:iam::999888777666:role/CrossAccountRole
credential_source = Environment
region            = us-east-1
```

| Value | Source |
|-------|--------|
| `Ec2InstanceMetadata` | EC2 instance profile / IMDSv2 |
| `EcsContainer` | ECS task IAM role |
| `Environment` | `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` env vars |

**Note:** `credential_source` and `source_profile` are mutually exclusive — use one or the other.

### Credential Caching

When you use `source_profile` + `role_arn`, the AWS CLI caches assumed-role credentials in `~/.aws/cli/cache/`:

```bash
# View cached credentials
ls ~/.aws/cli/cache/
# Files are named by hash of the profile config

# Clear cache (force re-assumption)
rm -f ~/.aws/cli/cache/*.json
```

Caching behavior:
- Cached credentials are reused until they expire
- `duration_seconds` controls how long they're valid
- MFA-protected roles prompt for a token only when the cache expires
- SDKs (boto3, etc.) maintain their own in-memory cache per process

---

## SSO

```bash
# Login
aws sso login --profile sso-admin

# Use profile after login
aws s3 ls --profile sso-admin

# Logout
aws sso logout
```

---

## Verify Identity

```bash
# Check who you are authenticated as
aws sts get-caller-identity

# Check with a specific profile
aws sts get-caller-identity --profile admin

# Example output:
# {
#     "UserId": "AROA2OHJBKACYGPBL7CL3:session-name",
#     "Account": "123456789012",
#     "Arn": "arn:aws:sts::123456789012:assumed-role/AdminRole/session-name"
# }
```

---

## Output Formats

```bash
aws configure set output json    # Default structured format
aws configure set output yaml    # YAML format
aws configure set output table   # Human-readable table
aws configure set output text    # Tab-separated, good for scripting
```

---

## Tips

```bash
# Use a profile for a single command without changing default
aws s3 ls --profile staging

# Override region for a single command
aws ec2 describe-instances --region eu-west-1

# Check if env vars are active (overriding profile)
env | grep AWS_

# Delete a profile (manual - no CLI command)
# Remove the relevant [profile name] block from ~/.aws/config
# and [name] block from ~/.aws/credentials
```

---

## CLI Behavior Settings

### retry_mode and max_attempts

Controls how the CLI/SDK handles throttling and transient errors:

```ini
# ~/.aws/config
[default]
retry_mode   = standard
max_attempts = 5
```

| retry_mode | Behavior |
|------------|----------|
| `legacy` | Default pre-2020. Limited retries, no backoff standardization |
| `standard` | Exponential backoff with jitter. Retries on throttling, timeouts, and transient errors |
| `adaptive` | Like standard, but also adjusts request rate based on throttling responses (experimental) |

```bash
# Set via environment variable (useful in CI)
export AWS_RETRY_MODE=standard
export AWS_MAX_ATTEMPTS=10
```

When to change:
- Scripts making many API calls in loops → increase `max_attempts`
- Hitting `ThrottlingException` or `TooManyRequestsException` → use `standard` or `adaptive`
- Lambda or short-lived processes → keep `max_attempts` low to fail fast

### ca_bundle

For corporate environments with TLS inspection proxies or custom CA certificates:

```ini
# ~/.aws/config
[default]
ca_bundle = /etc/pki/tls/certs/corporate-ca-bundle.crt
```

```bash
# Via environment variable
export AWS_CA_BUNDLE=/etc/pki/tls/certs/corporate-ca-bundle.crt
```

Common scenarios:
- Corporate proxy that re-signs TLS traffic (Zscaler, Netskope, Bluecoat)
- Private AWS endpoints using internal CAs (PrivateLink with custom certs)
- Self-signed certs in dev environments

Without this, you'll see:
```
SSL validation failed for https://sts.amazonaws.com/ [Errno 1] _ssl.c:510: 
error:14090086:SSL routines:SSL3_GET_SERVER_CERTIFICATE:certificate verify failed
```

### cli_pager

By default, the AWS CLI pipes long output through a pager (`less`). This breaks scripts and CI pipelines that expect output on stdout:

```ini
# ~/.aws/config — disable pager globally
[default]
cli_pager =
```

```bash
# Via environment variable
export AWS_PAGER=""

# Disable for a single command
aws ec2 describe-instances --no-cli-pager
```

| Setting | Behavior |
|---------|----------|
| Not set (default) | Uses `less` as pager |
| `cli_pager =` (empty) | No pager, output goes straight to stdout |
| `cli_pager = cat` | Same as disabling (cat just passes through) |
| `cli_pager = bat` | Use `bat` for syntax-highlighted paged output |

**Recommendation:** Always set `cli_pager =` in CI/CD environments and scripts. Interactively, the default pager is fine.
