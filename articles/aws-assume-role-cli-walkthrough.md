# Assume an IAM Role via CLI (Step by Step)

## Step 1 — Create an IAM User

```bash
aws iam create-user --user-name Bob
```

## Step 2 — Create a Policy That Allows Assuming a Role

Save as `example-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "iam:ListRoles",
        "sts:AssumeRole"
      ],
      "Resource": "*"
    }
  ]
}
```

Create and attach it:

```bash
aws iam create-policy --policy-name example-policy --policy-document file://example-policy.json
# Note the ARN in the output, e.g. arn:aws:iam::123456789012:policy/example-policy

aws iam attach-user-policy --user-name Bob --policy-arn "arn:aws:iam::123456789012:policy/example-policy"
```

Verify:

```bash
aws iam list-attached-user-policies --user-name Bob
```

## Step 3 — Create a Trust Policy for the Role

Save as `example-role-trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": {
      "AWS": "123456789012"
    },
    "Action": "sts:AssumeRole"
  }
}
```

This allows all IAM identities in account `123456789012` to assume the role. To restrict to a specific user:

```json
"Principal": {
  "AWS": "arn:aws:iam::123456789012:user/Bob"
}
```

## Step 4 — Create the Role and Attach Permissions

```bash
# Create the role with the trust policy
aws iam create-role --role-name example-role --assume-role-policy-document file://example-role-trust-policy.json

# Attach a permission policy (e.g., RDS read-only)
aws iam attach-role-policy --role-name example-role --policy-arn "arn:aws:iam::aws:policy/AmazonRDSReadOnlyAccess"

# Verify
aws iam list-attached-role-policies --role-name example-role
```

## Step 5 — Create Access Keys and Configure the CLI

```bash
aws iam create-access-key --user-name Bob
# Note the AccessKeyId and SecretAccessKey from the output

aws configure
# AWS Access Key ID: <paste AccessKeyId>
# AWS Secret Access Key: <paste SecretAccessKey>
# Default region name: eu-west-1
# Default output format: json
```

## Step 6 — Verify Identity and Confirm No Direct Access

```bash
# Confirm you're Bob
aws sts get-caller-identity

# This should fail with "Access Denied" (Bob doesn't have RDS access directly)
aws rds describe-db-instances --query "DBInstances[*].[DBInstanceIdentifier, DBName, DBInstanceStatus, AvailabilityZone, DBInstanceClass]"
```

## Step 7 — Assume the Role

```bash
# Find the role ARN
aws iam list-roles --query "Roles[?RoleName == 'example-role'].[RoleName, Arn]"

# Assume it
aws sts assume-role --role-arn "arn:aws:iam::123456789012:role/example-role" --role-session-name AWSCLI-Session
```

Note the `AccessKeyId`, `SecretAccessKey`, and `SessionToken` from the output.

## Step 8 — Export the Temporary Credentials

### Linux / macOS

```bash
export AWS_ACCESS_KEY_ID=RoleAccessKeyID
export AWS_SECRET_ACCESS_KEY=RoleSecretKey
export AWS_SESSION_TOKEN=RoleSessionToken
```

### Windows

```cmd
SET AWS_ACCESS_KEY_ID=RoleAccessKeyID
SET AWS_SECRET_ACCESS_KEY=RoleSecretKey
SET AWS_SESSION_TOKEN=RoleSessionToken
```

## Step 9 — Verify You're Using the Role

```bash
aws sts get-caller-identity
# Should show: arn:aws:sts::123456789012:assumed-role/example-role/AWSCLI-Session

# Now this works (role has RDS read-only access)
aws rds describe-db-instances --query "DBInstances[*].[DBInstanceIdentifier, DBName, DBInstanceStatus, AvailabilityZone, DBInstanceClass]"
```

## Step 10 — Return to the Original User

### Linux / macOS

```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN

# Confirm you're back as Bob
aws sts get-caller-identity
```

### Windows

```cmd
SET AWS_ACCESS_KEY_ID=
SET AWS_SECRET_ACCESS_KEY=
SET AWS_SESSION_TOKEN=
```

## Alternative — Use a Named Profile (Recommended)

Instead of manually exporting variables, add to `~/.aws/config`:

```ini
[profile example-role]
role_arn = arn:aws:iam::123456789012:role/example-role
source_profile = default
```

Then use:

```bash
aws rds describe-db-instances --profile example-role
```

The CLI handles assume-role automatically — no manual credential export needed.

## See Also

- [AWS STS Assume Role with MFA](aws-sts-assume-role.md) — MFA-enforced assume role for daily use, session scripts, and role chaining.
