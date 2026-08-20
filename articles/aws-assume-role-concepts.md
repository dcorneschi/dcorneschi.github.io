# AWS AssumeRole Concepts

AssumeRole is an AWS STS operation that allows an IAM user, role, or federated user to temporarily assume another role and receive temporary security credentials.

## How It Works

1. An entity (user, role, or service) calls `sts:AssumeRole`
2. AWS validates the trust policy on the target role
3. If allowed, AWS returns temporary credentials
4. The caller uses those credentials to access resources with the role's permissions
5. Credentials expire automatically after the session duration

## Temporary Credentials

When you assume a role, you receive:

| Credential | Description |
|-----------|-------------|
| Access Key ID | Starts with `ASIA` (temporary, unlike `AKIA` for permanent keys) |
| Secret Access Key | Paired with the access key |
| Session Token | Required for all API calls with temporary credentials |
| Expiration | When credentials become invalid (1–12 hours) |

## Common Use Cases

| Use Case | Description |
|----------|-------------|
| Cross-account access | Access resources in different AWS accounts |
| Privilege escalation | Temporarily gain elevated permissions for specific tasks |
| Service-to-service auth | Applications assuming roles to access other AWS services |
| Federated access | External identity providers granting AWS access |
| CI/CD pipelines | Deployment processes assuming roles with specific permissions |
| Break-glass access | Emergency admin access with additional approval |

## Trust Policy

The role being assumed must have a trust policy that allows the caller:

### Same-Account Trust

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/ExampleUser"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Cross-Account Trust

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "UniqueSecret"
        }
      }
    }
  ]
}
```

### Trust an AWS Service

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Trust with MFA Required

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/Admin"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

## CLI Usage

### Assume a Role

```bash
# Assume a role and get temporary credentials
aws sts assume-role \
    --role-arn "arn:aws:iam::123456789012:role/MyRole" \
    --role-session-name "MySession"

# With MFA
aws sts assume-role \
    --role-arn "arn:aws:iam::123456789012:role/MyRole" \
    --role-session-name "MySession" \
    --serial-number "arn:aws:iam::123456789012:mfa/user" \
    --token-code 123456

# With external ID (cross-account)
aws sts assume-role \
    --role-arn "arn:aws:iam::123456789012:role/CrossAccountRole" \
    --role-session-name "CrossSession" \
    --external-id "UniqueSecret"

# With custom duration (seconds, max depends on role config)
aws sts assume-role \
    --role-arn "arn:aws:iam::123456789012:role/MyRole" \
    --role-session-name "MySession" \
    --duration-seconds 3600
```

### Export Credentials

```bash
# Parse and export (bash)
CREDS=$(aws sts assume-role \
    --role-arn "arn:aws:iam::123456789012:role/MyRole" \
    --role-session-name "MySession" \
    --output json)

export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r '.Credentials.SessionToken')

# Verify
aws sts get-caller-identity
```

### One-Liner Script

```bash
#!/bin/bash
# assume-role.sh <role-arn>
eval $(aws sts assume-role \
    --role-arn "$1" \
    --role-session-name "$(whoami)-$(date +%s)" \
    --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
    --output text | \
    awk '{printf "export AWS_ACCESS_KEY_ID=%s\nexport AWS_SECRET_ACCESS_KEY=%s\nexport AWS_SESSION_TOKEN=%s\n", $1, $2, $3}')
```

## AWS CLI Profile Configuration

Configure profiles to automatically assume roles:

```ini
# ~/.aws/config

# Source profile with access keys
[profile default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
region = us-east-1

# Auto-assume role when using this profile
[profile production]
role_arn = arn:aws:iam::123456789012:role/ProductionRole
source_profile = default
region = us-east-1

# With MFA required
[profile admin]
role_arn = arn:aws:iam::123456789012:role/AdminRole
source_profile = default
mfa_serial = arn:aws:iam::123456789012:mfa/user
region = us-east-1

# Cross-account with external ID
[profile partner-account]
role_arn = arn:aws:iam::999988887777:role/PartnerRole
source_profile = default
external_id = UniqueSecret
region = eu-west-1
```

```bash
# Use a profile
aws s3 ls --profile production
aws ec2 describe-instances --profile admin
```

## Role Chaining

Assume a role, then assume another role from that session:

```bash
# First hop: assume RoleA
CREDS_A=$(aws sts assume-role \
    --role-arn "arn:aws:iam::111111111111:role/RoleA" \
    --role-session-name "hop1")

# Export RoleA credentials
export AWS_ACCESS_KEY_ID=$(echo $CREDS_A | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS_A | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo $CREDS_A | jq -r '.Credentials.SessionToken')

# Second hop: assume RoleB from RoleA's session
CREDS_B=$(aws sts assume-role \
    --role-arn "arn:aws:iam::222222222222:role/RoleB" \
    --role-session-name "hop2")
```

> **Note:** Role chaining limits session duration to 1 hour maximum, regardless of the role's `MaxSessionDuration` setting.

## Session Duration

| Scenario | Default | Maximum |
|----------|:-------:|:-------:|
| IAM user assumes role | 1 hour | 12 hours (role setting) |
| Role assumes role (chaining) | 1 hour | 1 hour (hard limit) |
| SAML/OIDC federation | 1 hour | 12 hours |
| EC2 instance profile | 6 hours | 6 hours |

```bash
# Set max session duration on a role (seconds)
aws iam update-role \
    --role-name MyRole \
    --max-session-duration 43200  # 12 hours

# Request specific duration when assuming
aws sts assume-role \
    --role-arn "arn:aws:iam::123456789012:role/MyRole" \
    --role-session-name "LongSession" \
    --duration-seconds 43200
```

## CloudTrail Logging

Every `AssumeRole` call is logged in CloudTrail:

```bash
# Find AssumeRole events
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole \
    --max-results 10

# Filter by role
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=ResourceName,AttributeValue=arn:aws:iam::123456789012:role/MyRole \
    --max-results 10
```

## Best Practices

| Practice | Why |
|----------|-----|
| Principle of least privilege | Only grant necessary permissions to the role |
| Use external IDs | Prevents confused deputy problem for cross-account |
| Require MFA for sensitive roles | Extra security layer for admin/production access |
| Set appropriate session duration | Balance security vs. usability |
| Use meaningful session names | Makes CloudTrail auditing easier |
| Monitor with CloudTrail | Track who assumes what and when |
| Prefer role assumption over long-lived keys | Temporary credentials auto-expire |
| Use conditions in trust policies | Restrict by source IP, time, or organization |

## Security Benefits

- **Temporary access** — credentials automatically expire
- **Audit trail** — all AssumeRole calls are logged in CloudTrail
- **Granular permissions** — different roles for different purposes
- **No long-term credentials** — reduces risk of credential compromise
- **Revocable sessions** — can revoke active sessions via IAM policy
- **Cross-account isolation** — maintain separate accounts while enabling access

## See Also

- [AWS STS Assume Role with MFA](articles/aws-sts-assume-role.md) — MFA-enforced role assumption with session scripts
- [Assume an IAM Role via CLI (Step by Step)](articles/aws-assume-role-cli-walkthrough.md) — Full walkthrough creating users, policies, and roles
- [AWS IAM Concepts Guide](articles/aws-iam-concepts-guide.md) — IAM fundamentals and Identity Center
