# AWS IAM CLI Cheatsheet

All `aws iam` and `aws sts` commands for managing users, groups, roles, policies, access keys, MFA, and temporary credentials.

## Users

```bash
# List all users
aws iam list-users
aws iam list-users --query 'Users[].UserName' --output table
aws iam list-users --query "Users[].[UserName, Arn, CreateDate]" --output table

# Create user
aws iam create-user --user-name jsmith

# Get user details (creation date, ARN)
aws iam get-user --user-name jsmith

# Delete user (must remove keys, policies, groups first)
aws iam delete-user --user-name jsmith

# Add user to group
aws iam add-user-to-group --user-name jsmith --group-name Developers

# Remove user from group
aws iam remove-user-from-group --user-name jsmith --group-name Developers

# List groups a user belongs to
aws iam list-groups-for-user --user-name jsmith

# Tag a user
aws iam tag-user --user-name jsmith --tags Key=Team,Value=Platform

# List user tags
aws iam list-user-tags --user-name jsmith
```

## Groups

```bash
# List all groups
aws iam list-groups

# Create group
aws iam create-group --group-name Developers

# Delete group
aws iam delete-group --group-name Developers

# List group members
aws iam get-group --group-name Developers

# Attach managed policy to group
aws iam attach-group-policy --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess

# Detach managed policy from group
aws iam detach-group-policy --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess

# List managed policies on a group
aws iam list-attached-group-policies --group-name Developers

# List inline policies on a group
aws iam list-group-policies --group-name Developers

# Add inline policy to group
aws iam put-group-policy --group-name Developers \
  --policy-name AllowS3 --policy-document file://policy.json

# Delete inline policy from group
aws iam delete-group-policy --group-name Developers --policy-name AllowS3
```

## Roles

```bash
# List all roles
aws iam list-roles
aws iam list-roles --query 'Roles[].RoleName' --output table
aws iam list-roles --query "Roles[].[RoleName, Arn]"                            # Name + ARN only
aws iam list-roles --query "Roles[?RoleName == 'MyRole'].[RoleName, Arn]"       # Find specific role by name

# Create role
aws iam create-role --role-name MyRole \
  --assume-role-policy-document file://trust-policy.json

# Get role details
aws iam get-role --role-name MyRole

# Get trust policy (who can assume)
aws iam get-role --role-name MyRole --query 'Role.AssumeRolePolicyDocument'

# Update trust policy
aws iam update-assume-role-policy --role-name MyRole \
  --policy-document file://trust-policy.json

# Delete role
aws iam delete-role --role-name MyRole

# Attach managed policy to role
aws iam attach-role-policy --role-name MyRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Detach managed policy from role
aws iam detach-role-policy --role-name MyRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# List managed policies on a role
aws iam list-attached-role-policies --role-name MyRole

# List inline policies on a role
aws iam list-role-policies --role-name MyRole

# Add inline policy to role
aws iam put-role-policy --role-name MyRole \
  --policy-name AllowDynamo --policy-document file://policy.json

# Get inline policy on role
aws iam get-role-policy --role-name MyRole --policy-name AllowDynamo

# Delete inline policy from role
aws iam delete-role-policy --role-name MyRole --policy-name AllowDynamo

# Set max session duration (default: 3600, max: 43200)
aws iam update-role --role-name MyRole --max-session-duration 14400

# Tag a role
aws iam tag-role --role-name MyRole --tags Key=Environment,Value=Production
```

## Policies

### Managed Policies

```bash
# List customer managed policies
aws iam list-policies --scope Local
aws iam list-policies --scope Local --query 'Policies[].PolicyName' --output table

# List AWS managed policies
aws iam list-policies --scope AWS --query 'Policies[?starts_with(PolicyName, `AmazonS3`)].[PolicyName, Arn]' --output table
aws iam list-policies --scope AWS --query "Policies[?PolicyName == 'AdministratorAccess']"  # Find specific AWS managed policy

# Create managed policy
aws iam create-policy --policy-name MyPolicy --policy-document file://policy.json

# Get policy metadata (ARN, version, attachment count)
aws iam get-policy --policy-arn arn:aws:iam::123456789012:policy/MyPolicy
aws iam get-policy --policy-arn arn:aws:iam::aws:policy/AdministratorAccess     # AWS managed policy

# Get policy document (specific version)
aws iam get-policy-version \
  --policy-arn arn:aws:iam::123456789012:policy/MyPolicy \
  --version-id v1

# Get default (active) version
VERSION=$(aws iam get-policy --policy-arn arn:aws:iam::123456789012:policy/MyPolicy --query 'Policy.DefaultVersionId' --output text)
aws iam get-policy-version --policy-arn arn:aws:iam::123456789012:policy/MyPolicy --version-id "$VERSION"

# Create a new version (becomes default)
aws iam create-policy-version \
  --policy-arn arn:aws:iam::123456789012:policy/MyPolicy \
  --policy-document file://policy-v2.json \
  --set-as-default

# List policy versions
aws iam list-policy-versions --policy-arn arn:aws:iam::123456789012:policy/MyPolicy

# Delete a non-default version
aws iam delete-policy-version \
  --policy-arn arn:aws:iam::123456789012:policy/MyPolicy \
  --version-id v1

# Delete policy (must detach from all entities first)
aws iam delete-policy --policy-arn arn:aws:iam::123456789012:policy/MyPolicy

# List who uses a policy
aws iam list-entities-for-policy --policy-arn arn:aws:iam::123456789012:policy/MyPolicy
```

### Inline Policies

```bash
# Add inline policy to user
aws iam put-user-policy --user-name jsmith \
  --policy-name MyInlinePolicy --policy-document file://policy.json

# Get inline policy from user
aws iam get-user-policy --user-name jsmith --policy-name MyInlinePolicy

# List inline policies on user
aws iam list-user-policies --user-name jsmith

# Delete inline policy from user
aws iam delete-user-policy --user-name jsmith --policy-name MyInlinePolicy
```

### Attaching / Detaching Managed Policies

```bash
# Attach to user
aws iam attach-user-policy --user-name jsmith \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Detach from user
aws iam detach-user-policy --user-name jsmith \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# List managed policies on user
aws iam list-attached-user-policies --user-name jsmith
```

## Access Keys

```bash
# List access keys for a user
aws iam list-access-keys --user-name jsmith

# Create access key
aws iam create-access-key --user-name jsmith

# Get last used info
aws iam get-access-key-last-used --access-key-id AKIAXXXXXXXXX

# Deactivate access key
aws iam update-access-key --user-name jsmith \
  --access-key-id AKIAXXXXXXXXX --status Inactive

# Reactivate access key
aws iam update-access-key --user-name jsmith \
  --access-key-id AKIAXXXXXXXXX --status Active

# Delete access key
aws iam delete-access-key --user-name jsmith --access-key-id AKIAXXXXXXXXX
```

## MFA

```bash
# List MFA devices for a user
aws iam list-mfa-devices --user-name jsmith

# List all virtual MFA devices
aws iam list-virtual-mfa-devices

# Create virtual MFA device
aws iam create-virtual-mfa-device \
  --virtual-mfa-device-name jsmith-mfa \
  --outfile /tmp/qrcode.png \
  --bootstrap-method QRCodePNG

# Enable MFA (requires two consecutive codes from the device)
aws iam enable-mfa-device --user-name jsmith \
  --serial-number arn:aws:iam::123456789012:mfa/jsmith-mfa \
  --authentication-code1 123456 \
  --authentication-code2 789012

# Deactivate MFA device
aws iam deactivate-mfa-device --user-name jsmith \
  --serial-number arn:aws:iam::123456789012:mfa/jsmith-mfa

# Delete virtual MFA device
aws iam delete-virtual-mfa-device \
  --serial-number arn:aws:iam::123456789012:mfa/jsmith-mfa
```

## Passwords

```bash
# Create login profile (console password)
aws iam create-login-profile --user-name jsmith \
  --password 'TempP@ssw0rd!' --password-reset-required

# Update password
aws iam update-login-profile --user-name jsmith \
  --password 'NewP@ssw0rd!' --password-reset-required

# Delete login profile (disable console access)
aws iam delete-login-profile --user-name jsmith

# Get password policy
aws iam get-account-password-policy

# Generate YAML skeleton for password policy update
aws iam update-account-password-policy --generate-cli-skeleton yaml-input

# Update password policy
aws iam update-account-password-policy \
  --minimum-password-length 14 \
  --require-symbols \
  --require-numbers \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --max-password-age 90 \
  --password-reuse-prevention 12
```

## STS (Security Token Service)

```bash
# Get caller identity (who am I?)
aws sts get-caller-identity
aws sts get-caller-identity --query Arn --output text    # ARN only

# Assume a role
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyRole \
  --role-session-name my-session

# Assume a role with MFA
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyRole \
  --role-session-name my-session \
  --serial-number arn:aws:iam::123456789012:mfa/jsmith \
  --token-code 123456

# Assume role with duration
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyRole \
  --role-session-name my-session \
  --duration-seconds 14400

# Assume role with external ID (cross-account)
aws sts assume-role \
  --role-arn arn:aws:iam::999999999999:role/ThirdPartyRole \
  --role-session-name cross-account \
  --external-id "unique-external-id"

# Get session token (MFA-protected)
aws sts get-session-token \
  --serial-number arn:aws:iam::123456789012:mfa/jsmith \
  --token-code 123456 \
  --duration-seconds 3600

# Assume role with web identity (OIDC)
aws sts assume-role-with-web-identity \
  --role-arn arn:aws:iam::123456789012:role/OIDCRole \
  --role-session-name oidc-session \
  --web-identity-token "$OIDC_TOKEN"

# Decode authorization error message
aws sts decode-authorization-message --encoded-message "$ENCODED_MSG"
```

## Policy Simulation

```bash
# Simulate if a user can perform an action
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/jsmith \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/file.txt

# Simulate multiple actions
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/MyRole \
  --action-names s3:GetObject s3:PutObject s3:DeleteObject \
  --resource-arns "arn:aws:s3:::my-bucket/*"

# Simulate a custom policy document
aws iam simulate-custom-policy \
  --policy-input-list file://policy.json \
  --action-names ec2:DescribeInstances ec2:StartInstances
```

## Credential Report

```bash
# Generate credential report
aws iam generate-credential-report

# Download (base64 decode)
aws iam get-credential-report --output text --query Content | base64 -d > credential-report.csv

# Find users without MFA
aws iam get-credential-report --output text --query Content | base64 -d | \
  awk -F, 'NR>1 && $4=="true" && $8=="false" {print $1, "- no MFA"}'

# Find inactive access keys (not used in 90 days)
aws iam get-credential-report --output text --query Content | base64 -d | \
  awk -F, 'NR>1 && $9=="true" {print $1, $11}'
```

## Access Analyzer

```bash
# Create analyzer
aws accessanalyzer create-analyzer --analyzer-name my-analyzer --type ACCOUNT

# List findings
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:eu-west-1:123456789012:analyzer/my-analyzer

# Validate a policy
aws accessanalyzer validate-policy \
  --policy-document file://policy.json \
  --policy-type IDENTITY_POLICY
```

## Service Last Accessed

```bash
# Generate report
JOB_ID=$(aws iam generate-service-last-accessed-details \
  --arn arn:aws:iam::123456789012:user/jsmith \
  --query 'JobId' --output text)

# Get results (services accessed)
aws iam get-service-last-accessed-details --job-id "$JOB_ID" \
  --query 'ServicesLastAccessed[?LastAuthenticated!=`null`].[ServiceName,LastAuthenticated]' \
  --output table

# Services never accessed (candidates for removal)
aws iam get-service-last-accessed-details --job-id "$JOB_ID" \
  --query 'ServicesLastAccessed[?LastAuthenticated==`null`].ServiceName' \
  --output table
```

## Instance Profiles

```bash
# List instance profiles
aws iam list-instance-profiles

# Create instance profile
aws iam create-instance-profile --instance-profile-name MyProfile

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name MyProfile --role-name MyRole

# Remove role from instance profile
aws iam remove-role-from-instance-profile \
  --instance-profile-name MyProfile --role-name MyRole

# Delete instance profile
aws iam delete-instance-profile --instance-profile-name MyProfile
```

## Account Settings

```bash
# Get account summary (limits and usage)
aws iam get-account-summary

# Get account alias
aws iam list-account-aliases

# Set account alias (used in console login URL)
aws iam create-account-alias --account-alias my-company

# Delete account alias
aws iam delete-account-alias --account-alias my-company
```

## OIDC Providers

```bash
# List OIDC providers
aws iam list-open-id-connect-providers

# Create OIDC provider (e.g., for EKS IRSA or GitHub Actions)
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list "6938fd4d98bab03faadb97b34396831e3780aea1"

# Get OIDC provider details
aws iam get-open-id-connect-provider \
  --open-id-connect-provider-arn arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com

# Delete OIDC provider
aws iam delete-open-id-connect-provider \
  --open-id-connect-provider-arn arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com
```

## Useful One-Liners

```bash
# List all users with their access key status
aws iam list-users --query 'Users[].UserName' --output text | \
  xargs -I{} bash -c 'echo "=== {} ==="; aws iam list-access-keys --user-name {} --output table'

# Find all roles assumable by a specific account
aws iam list-roles --query "Roles[?contains(AssumeRolePolicyDocument.Statement[].Principal.AWS, '999999999999')].RoleName" --output table

# List all policies attached to a role (inline + managed)
ROLE="MyRole"
echo "--- Managed ---"
aws iam list-attached-role-policies --role-name "$ROLE" --query 'AttachedPolicies[].PolicyName' --output table
echo "--- Inline ---"
aws iam list-role-policies --role-name "$ROLE" --output table

# Export all managed policy documents
aws iam list-policies --scope Local --query 'Policies[].Arn' --output text | \
  xargs -I{} bash -c 'echo "=== {} ==="; VER=$(aws iam get-policy --policy-arn {} --query Policy.DefaultVersionId --output text); aws iam get-policy-version --policy-arn {} --version-id $VER --query PolicyVersion.Document'

# Count resources
aws iam get-account-summary --query 'SummaryMap.{Users:Users,Roles:Roles,Groups:Groups,Policies:Policies}'
```

## See Also

- [AWS IAM Concepts Guide](aws-iam-concepts-guide.md) — Roles, policy types, evaluation logic, Identity Center, federation, and root vs admin.
- [AWS STS Assume Role with MFA](aws-sts-assume-role.md) — MFA-enforced assume role, session scripts, and role chaining.
- [Assume an IAM Role via CLI (Step by Step)](aws-assume-role-cli-walkthrough.md) — Full walkthrough creating users, policies, roles, and assuming them.
