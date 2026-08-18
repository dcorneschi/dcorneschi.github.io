# AWS IAM Concepts Guide

IAM fundamentals — roles, policy types, evaluation logic, Identity Center (SSO), federation, temporary credentials, permission boundaries, and root account vs admin user.

## Best Practices

- Never use the root account for day-to-day tasks
- Enable MFA for all human users (especially root)
- Apply least privilege: grant only the permissions needed
- Use groups to assign permissions instead of attaching policies directly to users
- Rotate access keys regularly and remove unused credentials
- Use roles for applications and services (never embed long-term credentials in code)
- Use AWS Organizations SCPs for guardrails across accounts

## Roles

A role is an AWS identity with permission policies. Unlike users, roles don't have permanent credentials — they're assumed temporarily by:

- AWS services (EC2, Lambda, ECS)
- Users from your account or other accounts
- Federated users from external identity providers

Think of a role as a "hat" that grants specific permissions when worn. When you assume a role, you get temporary security credentials.

Common use cases:
- EC2 instance accessing S3 buckets
- Lambda function writing to DynamoDB
- Cross-account access between AWS accounts

## Policy Types

### Inline Policies

Embedded directly in a single user, group, or role. Useful for strict 1:1 relationships.

```bash
aws iam put-user-policy \
  --user-name jsmith \
  --policy-name AllowS3ReadOnly \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:ListBucket"
        ],
        "Resource": [
          "arn:aws:s3:::my-bucket",
          "arn:aws:s3:::my-bucket/*"
        ]
      }
    ]
  }'
```

### AWS Managed Policies

Created and maintained by AWS. Good starting point but can be overly broad.

### Customer Managed Policies

Created by you, reusable across multiple identities. Preferred for most use cases.

```bash
# Create the policy
aws iam create-policy \
  --policy-name AllowS3ReadOnly \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:ListBucket"
        ],
        "Resource": [
          "arn:aws:s3:::my-bucket",
          "arn:aws:s3:::my-bucket/*"
        ]
      }
    ]
  }'

# Attach it to a user (reusable — can also attach to roles or groups)
aws iam attach-user-policy \
  --user-name jsmith \
  --policy-arn arn:aws:iam::123456789012:policy/AllowS3ReadOnly
```

A managed policy is a standalone object with its own ARN, versioning, and can be attached to many identities. An inline policy lives inside a single identity and is deleted when that identity is deleted.

### Inline vs Managed — Comparison

| Aspect | Managed Policy | Inline Policy |
|--------|---------------|---------------|
| Reusability | Can attach to multiple identities | Tied to one identity |
| Lifecycle | Independent | Deleted with identity |
| Version control | Supports up to 5 versions | No versioning |
| Size limit | 6,144 characters | 2,048 characters |
| Max per identity | 10 managed policies | 10 inline policies |
| Cross-account | Can be shared across accounts | Cannot be shared |
| Best for | Common, reusable permissions | Unique, one-off permissions |

**When to use which:**
- Use **managed policies** when permissions are reusable across multiple identities
- Use **inline policies** when permissions are unique to one identity and should never be accidentally applied elsewhere

## Policy JSON Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-app-bucket"
    }
  ]
}
```

## Identity-Based vs Resource-Based Policies

| Type | Attached To | Defines |
|------|-------------|---------|
| Identity-based | Users, groups, roles | What the identity can do |
| Resource-based | Resources (S3, SQS, KMS) | Who can access the resource |

Resource-based policies support cross-account access without assuming a role.

## Trust Policies vs Permission Policies

| Policy | Attached To | Purpose |
|--------|-------------|---------|
| Trust policy | Role | Defines *who* can assume the role (the principal) |
| Permission policy | Role | Defines *what* the role can do once assumed |

## Permission Boundaries

- Sets the *maximum* permissions an identity can have
- Effective permissions = identity-based policy AND permission boundary (intersection)
- Useful for delegating user/role creation safely (letting developers create roles without escalating privileges)

## Policy Evaluation Logic

AWS evaluates policies in this order:

1. **Explicit Deny** — if any policy explicitly denies, the request is denied (always wins)
2. **SCPs (Organizations)** — if an SCP doesn't allow the action, it's denied
3. **Permission boundaries** — if a boundary is set and doesn't allow the action, it's denied
4. **Session policies** — for temporary sessions, must also allow
5. **Explicit Allow** — at least one policy must explicitly allow the action
6. **Implicit Deny** — if nothing explicitly allows it, the request is denied by default

## Conditions

Policies support conditions to restrict access further:

| Condition Key | Use Case |
|---------------|----------|
| `aws:SourceIp` | IP restriction |
| `aws:MultiFactorAuthPresent` | MFA enforcement |
| `aws:CurrentTime` | Time-based access |
| `aws:RequestTag` / `aws:ResourceTag` | Tag-based access control |
| `aws:RequestedRegion` | Region restriction |

Example:

```json
"Condition": {
  "Bool": { "aws:MultiFactorAuthPresent": "true" },
  "IpAddress": { "aws:SourceIp": "10.0.0.0/8" }
}
```

## Service-Linked Roles

- Pre-defined roles created by AWS services (e.g., `AWSServiceRoleForECS`)
- Cannot modify the permission policy — AWS manages it
- Automatically created when you enable certain service features
- Can only be deleted after removing the dependent resources

## IRSA (IAM Roles for Service Accounts)

- Allows Kubernetes pods in EKS to assume IAM roles without node-level credentials
- Uses an OIDC provider to federate trust between Kubernetes service accounts and IAM roles
- The role trust policy references the OIDC provider ARN and the service account name/namespace

## IAM Identity Center (AWS SSO)

Recommended for human users. Authenticate via a central portal and get short-lived credentials for any account/role you're granted.

### How It Works

1. Admin enables IAM Identity Center in the management account
2. Users are defined in a directory (built-in, Active Directory, or external IdP like Okta/Azure AD)
3. **Permission sets** define what users can do in each account (maps to IAM roles behind the scenes)
4. Users log in via a portal URL, pick an account + permission set, and get temporary credentials

### Key Benefits Over Access Keys

- **No long-lived credentials on disk** — credentials auto-expire (1–12 hours)
- **No access keys to rotate or leak**
- **Centralized access management** across all accounts
- **MFA built-in** — enforced at the IdP level
- **Full audit trail** in CloudTrail

### CLI Setup

```bash
# One-time configuration (interactive)
aws configure sso --profile my-sso-profile
```

Creates a profile in `~/.aws/config`:

```ini
[profile my-sso-profile]
sso_start_url = https://mycompany.awsapps.com/start
sso_region = eu-west-1
sso_account_id = 123456789012
sso_role_name = AdministratorAccess
region = eu-west-1
output = json
```

### Daily Usage

```bash
# Login (opens browser for IdP + MFA)
aws sso login --profile my-sso-profile

# Use normally
aws s3 ls --profile my-sso-profile
aws ec2 describe-instances --profile my-sso-profile

# Check identity
aws sts get-caller-identity --profile my-sso-profile
```

### Multiple Accounts / Roles

```ini
[profile dev-readonly]
sso_start_url = https://mycompany.awsapps.com/start
sso_region = eu-west-1
sso_account_id = 111111111111
sso_role_name = ReadOnlyAccess
region = eu-west-1

[profile prod-admin]
sso_start_url = https://mycompany.awsapps.com/start
sso_region = eu-west-1
sso_account_id = 222222222222
sso_role_name = AdministratorAccess
region = eu-west-1
```

A single `aws sso login` authenticates once — all profiles sharing the same `sso_start_url` use the cached session.

### Session Duration

| Type | Default | Configurable |
|------|---------|-------------|
| Portal session | 8 hours | 1–90 days |
| Permission set session | 1 hour | Up to 12 hours |

### SSO vs Access Keys

| Aspect | Access Keys + AssumeRole | IAM Identity Center (SSO) |
|--------|--------------------------|---------------------------|
| Credentials on disk | Yes (`~/.aws/credentials`) | No (cached token, auto-expires) |
| MFA | Must pass manually per assume-role | Handled at IdP login (once per session) |
| Key rotation | Manual responsibility | Not needed |
| Multi-account | One assume-role call per account | One login, switch profiles freely |
| Revocation | Deactivate key + revoke sessions | Disable user in directory (instant) |
| Best for | Automation, legacy setups | Human users, daily CLI/console access |

## GetSessionToken (MFA-Protected)

Exchange long-lived keys + MFA code for temporary credentials:

```bash
aws sts get-session-token \
  --serial-number arn:aws:iam::123456789012:mfa/jsmith \
  --token-code 123456 \
  --duration-seconds 3600
```

Useful when you can't eliminate the IAM user yet but want MFA on every action.

## OIDC Federation (AssumeRoleWithWebIdentity)

For CI/CD pipelines (GitHub Actions, GitLab CI) and Kubernetes (IRSA). No stored secrets.

```bash
aws sts assume-role-with-web-identity \
  --role-arn arn:aws:iam::123456789012:role/GitHubActionsRole \
  --role-session-name github-run \
  --web-identity-token "$OIDC_TOKEN"
```

## SAML Federation (AssumeRoleWithSAML)

For corporate SSO (Active Directory, Okta) federated to AWS. Users authenticate against the IdP, receive a SAML assertion, and exchange it for temporary credentials.

## Instance / Task / Execution Roles

For workloads on AWS (EC2, ECS, Lambda, EKS). The service automatically provides and rotates temporary credentials via the metadata service — no configuration beyond assigning the role.

## Which Temporary Credential Method to Use

| Scenario | Solution |
|----------|----------|
| Human console/CLI access | IAM Identity Center (SSO) |
| CI/CD pipelines | OIDC federation |
| Cross-account access | `sts assume-role` |
| App on EC2/ECS/Lambda | Instance/task/execution role |
| Existing IAM user + MFA | `sts get-session-token` |
| Corporate SSO | SAML federation |

## Key Points

- Temporary credentials cannot be revoked individually — revoke all sessions via IAM console (inline deny on `aws:TokenIssueTime`)
- Maximum `AssumeRole` session: 12 hours (configurable, default 1 hour)
- Maximum `GetSessionToken` session: 36 hours for IAM users
- Always prefer roles and federation over IAM users — the goal is zero long-lived credentials

## Root Account vs IAM User with AdministratorAccess

The root account is tied to the email that created the AWS account. It exists outside IAM — no policy or SCP can restrict it.

### Actions Only Root Can Perform

- Change account name, email, or root password
- Close the AWS account
- Change the AWS support plan
- Enable MFA Delete on S3 buckets
- Create a CloudFront key pair (legacy)
- Configure billing (activate IAM access, modify payment methods)
- Restore IAM permissions when all access is locked out
- Register in the Reserved Instance Marketplace
- Sign up for GovCloud
- Edit account-level S3 Block Public Access
- Configure tax settings

### Root vs Admin — Comparison

| Aspect | Root Account | IAM User + AdministratorAccess |
|--------|-------------|-------------------------------|
| Restricted by IAM policies | No | Yes |
| Restricted by SCPs | No | Yes |
| Restricted by permission boundaries | No | Yes |
| Account-level operations | Full access | Cannot perform |
| MFA enforcement | Self-managed | Enforced via policies/SCPs |
| Blast radius if compromised | Unrestricted, no guardrails | Can be limited after the fact |
| Recommended for daily use | Never | Yes (with MFA + temporary credentials) |

Root is the only identity that can recover an account where all IAM access has been removed. This is why it should never be used for daily tasks — unrestricted blast radius with no guardrails.

## Migrating Inline to Managed Policies

1. **Audit** — identify common permission patterns across inline policies
2. **Create managed policies** — develop standardized policies for common patterns
3. **Gradual migration** — replace inline policies with managed policies incrementally
4. **Test** — verify permissions still work after each change
5. **Document** — record the purpose and scope of each managed policy

## IAM Access Analyzer

Identifies resources shared with external entities and finds unused permissions. Can also generate least-privilege policies from CloudTrail activity and validate policy documents for errors.

## Credential Report

A CSV report of all IAM users showing access key status, MFA enabled, password last used/rotated, and creation time. Useful for auditing compliance.

## Last Accessed Information

Shows which AWS services each identity has accessed and when. Use it to find unused permissions and tighten policies — services never accessed are candidates for removal.

## Cross-Account Role Patterns

### External ID (Confused Deputy Prevention)

When granting a third party access to your account, use an external ID to prevent the [confused deputy problem](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html):

Trust policy on the role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::999999999999:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-id-from-third-party"
        }
      }
    }
  ]
}
```

### Cross-Account Access (Your Own Accounts)

Trust policy allowing a role from another account you own:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::222222222222:role/ProdDeployRole"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

## See Also

- [AWS IAM CLI Cheatsheet](aws-iam-cheatsheet.md) — All `aws iam` and `aws sts` commands, credential reports, Access Analyzer, policy simulation, and audit one-liners.
- [AWS STS Assume Role with MFA](aws-sts-assume-role.md) — MFA-enforced assume role, session scripts, and role chaining.
- [Assume an IAM Role via CLI (Step by Step)](aws-assume-role-cli-walkthrough.md) — Full walkthrough creating users, policies, roles, and assuming them.
