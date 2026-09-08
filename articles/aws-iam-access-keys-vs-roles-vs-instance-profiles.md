# AWS IAM: Access Keys vs Roles vs Instance Profiles

These three terms get mixed up constantly because they all deal with "how
something authenticates to AWS." But they answer different questions:

- **Access keys** — *long-lived* static credentials for an IAM user.
- **Roles** — an *identity with permissions* that anyone/anything trusted can
  assume to get *temporary* credentials.
- **Instance profiles** — the *delivery mechanism* that hands a role to an EC2
  instance.

## Quick Comparison

| | Access keys | IAM role | Instance profile |
|---|---|---|---|
| What it is | Static credential pair for an IAM user | An identity with a permission policy + a trust policy | A container that binds one role to an EC2 instance |
| Credential lifetime | Long-lived until rotated/deleted | Temporary (minutes to hours) | Delivers the role's temporary creds |
| Credential type | `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` | Access key + secret + **session token** | Same temporary creds, fetched via IMDS |
| Who/what uses it | Users, scripts, CI, legacy apps | EC2, Lambda, ECS, EKS, users, cross-account | EC2 instances specifically |
| Rotation | Manual (a security burden) | Automatic (AWS rotates behind the scenes) | Automatic |
| Stored on disk? | Usually yes (`~/.aws/credentials`) | Ideally never | Never (served from instance metadata) |
| Main risk | Leakage of permanent secrets | Overly permissive trust policy | Attaching an over-privileged role |

## Access Keys

An access key is a static `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` pair tied
to an IAM **user**. They don't expire on their own — they work until you rotate
or delete them.

```bash
# Create access keys for a user
aws iam create-access-key --user-name jsmith

# Configure them locally
aws configure --profile jsmith
# (prompts for the key ID and secret)

# Use them
aws s3 ls --profile jsmith
```

Because they're long-lived and often land in files, env vars, CI config, or
(worst case) source control, they're the most common source of AWS credential
leaks. Prefer them only where nothing else works — and rotate regularly.

**When they're still reasonable:** a legacy on-prem app or a third-party tool
that has no way to assume a role.

## IAM Roles

A role is an identity you *assume* rather than log in as. It has two policies:

- A **permissions policy** — what the role can do.
- A **trust policy** — who is allowed to assume it (a service, a user, another
  account).

Assuming a role returns **temporary** credentials that include a session token
and expire automatically, so there's nothing long-lived to leak or rotate.

```bash
# Assume a role and capture temporary credentials
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/ReadOnlyRole \
  --role-session-name demo

# Confirm which identity you're now using
aws sts get-caller-identity
```

Roles are the recommended way to grant access to AWS services (EC2, Lambda,
ECS/EKS), for cross-account access, and for humans via IAM Identity Center (SSO).

See [AWS AssumeRole Concepts](articles/aws-assume-role-concepts.md) and the
[Assume an IAM Role via CLI walkthrough](articles/aws-assume-role-cli-walkthrough.md)
for the trust-policy details.

## Instance Profiles

An instance profile is *not* a separate permission mechanism — it's the wrapper
that lets an **EC2 instance** use a role. An instance profile contains exactly
one role; when you "attach a role to an EC2 instance," you're really attaching an
instance profile.

The instance then fetches temporary credentials from the Instance Metadata
Service (IMDS) — no keys on disk.

```bash
# Create a role EC2 can assume (trust policy allows the EC2 service)
aws iam create-role \
  --role-name AppServerRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }]
  }'

# Create the instance profile and put the role in it
aws iam create-instance-profile --instance-profile-name AppServerProfile
aws iam add-role-to-instance-profile \
  --instance-profile-name AppServerProfile \
  --role-name AppServerRole

# Attach it to a running instance
aws ec2 associate-iam-instance-profile \
  --instance-id i-0abc123 \
  --iam-instance-profile Name=AppServerProfile
```

From inside the instance, credentials come from IMDS:

```bash
# IMDSv2 (token-based) — the secure default
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/

# The SDK/CLI do this automatically — you don't set any keys
aws sts get-caller-identity
```

> The AWS SDKs and CLI use instance-profile credentials automatically when no
> other credentials are configured, so applications on EC2 need zero credential
> setup.

## How They Relate

- A **role** is the actual identity and set of permissions.
- An **instance profile** is how a role reaches an **EC2 instance** (1 profile → 1 role).
- Other services skip instance profiles entirely: Lambda uses an *execution
  role*, ECS tasks use a *task role*, and EKS pods use *IRSA* or *Pod Identity* —
  all of which are just roles delivered by that service's own mechanism.
- **Access keys** are the odd one out: static user credentials, used only when
  assuming a role isn't an option.

## Choosing Between Them

- **Workload on AWS (EC2/Lambda/ECS/EKS):** use a role (via instance profile /
  execution role / task role / IRSA). Never bake access keys into an AMI or
  container image.
- **Human access:** use IAM Identity Center (SSO) with roles, not per-user
  access keys.
- **Cross-account access:** use a role with a trust policy for the other account.
- **Legacy / external system that can't assume a role:** access keys, rotated
  regularly and scoped to least privilege.

## Best Practices

1. Prefer temporary credentials (roles) over long-lived access keys everywhere
   you can.
2. If you must use access keys, rotate them on a schedule and never commit them
   to source control.
3. Scope both permission policies and trust policies to least privilege.
4. Enforce IMDSv2 on EC2 to reduce SSRF-based credential theft.
5. Audit unused access keys and roles regularly.

## Related

- [AWS IAM Concepts Guide](articles/aws-iam-concepts-guide.md)
- [AWS AssumeRole Concepts](articles/aws-assume-role-concepts.md)
- [AWS STS Assume Role with MFA](articles/aws-sts-assume-role.md)
- [AWS IAM CLI Cheatsheet](articles/aws-iam-cheatsheet.md)

## Skills Practiced

- Distinguishing static access keys from temporary role credentials
- Understanding that an instance profile is a role delivery wrapper for EC2
- Creating a role + instance profile and attaching it to an instance
- Choosing the right credential mechanism per workload and access pattern
