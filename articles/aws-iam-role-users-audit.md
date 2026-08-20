# AWS IAM Role Users Audit

Methods to list users associated with a specific AWS role — trust policies, active sessions, policy scanning, LDAP/SAML federation, and IAM Identity Center.

## List Who Can Assume a Role

### Check the Trust Policy

The trust policy is the definitive source for who can assume a role:

```bash
# Get the trust policy
aws iam get-role --role-name MyRole --query 'Role.AssumeRolePolicyDocument'

# Table format
aws iam get-role --role-name MyRole --query 'Role.AssumeRolePolicyDocument' --output table

# Full role details
aws iam get-role --role-name MyRole
```

### Check Role Permissions (What It Can Do)

```bash
# List attached managed policies
aws iam list-attached-role-policies --role-name MyRole

# List inline policies
aws iam list-role-policies --role-name MyRole

# Get inline policy document
aws iam get-role-policy --role-name MyRole --policy-name PolicyName
```

## Find Users with AssumeRole Permissions

### Scan User Policies

```bash
# List all users
aws iam list-users --query 'Users[].UserName' --output text

# Check a user's attached policies
aws iam list-attached-user-policies --user-name username

# Check a user's inline policies
aws iam list-user-policies --user-name username

# Get inline policy document
aws iam get-user-policy --user-name username --policy-name PolicyName
```

### Scan Group Policies

```bash
# List all groups
aws iam list-groups --query 'Groups[].GroupName' --output text

# Check group attached policies
aws iam list-attached-group-policies --group-name GroupName

# List group members
aws iam get-group --group-name GroupName --query 'Users[].UserName'

# Check all groups for role references
aws iam list-groups --query 'Groups[].GroupName' --output text | \
    xargs -I {} aws iam list-attached-group-policies --group-name {}
```

## List Active Role Sessions (CloudTrail)

```bash
# Find recent AssumeRole events for a specific role
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole \
    --max-results 20 | \
    jq '.Events[].CloudTrailEvent | fromjson | select(.requestParameters.roleArn | contains("MyRole")) | {
        time: .eventTime,
        caller: .userIdentity.arn,
        sourceIP: .sourceIPAddress
    }'

# From CloudWatch Logs (if CloudTrail logs are forwarded)
aws logs filter-log-events \
    --log-group-name CloudTrail/DefaultLogGroup \
    --start-time $(date -d '24 hours ago' +%s)000 \
    --filter-pattern '{ $.eventName = "AssumeRole" && $.requestParameters.roleArn = "*MyRole*" }'
```

## Advanced Queries

```bash
# List all roles with their trust policies
aws iam list-roles --query 'Roles[].[RoleName,AssumeRolePolicyDocument]' --output table

# Find roles that trust a specific user
aws iam list-roles --query 'Roles[?contains(to_string(AssumeRolePolicyDocument), `arn:aws:iam::123456789012:user/bob`)].[RoleName]' --output text

# Find roles that trust a specific account
aws iam list-roles --query 'Roles[?contains(to_string(AssumeRolePolicyDocument), `111122223333`)].[RoleName]' --output text
```

## Comprehensive Audit Script

```bash
#!/bin/bash
# audit-role-access.sh — Find all entities that can assume a given role
ROLE_NAME="${1:?Usage: $0 <role-name>}"

echo "=== Trust Policy for $ROLE_NAME ==="
aws iam get-role --role-name "$ROLE_NAME" --query 'Role.AssumeRolePolicyDocument' --output json

echo -e "\n=== Role Permissions ==="
aws iam list-attached-role-policies --role-name "$ROLE_NAME" --output table
aws iam list-role-policies --role-name "$ROLE_NAME" --output table

echo -e "\n=== Users with inline policies referencing this role ==="
for user in $(aws iam list-users --query 'Users[].UserName' --output text); do
    for policy in $(aws iam list-user-policies --user-name "$user" --query 'PolicyNames[]' --output text); do
        if aws iam get-user-policy --user-name "$user" --policy-name "$policy" \
            --query 'PolicyDocument' --output text 2>/dev/null | grep -q "$ROLE_NAME"; then
            echo "  User: $user (inline policy: $policy)"
        fi
    done
done

echo -e "\n=== Groups with policies referencing this role ==="
for group in $(aws iam list-groups --query 'Groups[].GroupName' --output text); do
    for policy in $(aws iam list-group-policies --group-name "$group" --query 'PolicyNames[]' --output text); do
        if aws iam get-group-policy --group-name "$group" --policy-name "$policy" \
            --query 'PolicyDocument' --output text 2>/dev/null | grep -q "$ROLE_NAME"; then
            members=$(aws iam get-group --group-name "$group" --query 'Users[].UserName' --output text)
            echo "  Group: $group (members: $members)"
        fi
    done
done
```

## SAML/LDAP Federation

### List SAML Providers

```bash
# List all SAML providers
aws iam list-saml-providers --query 'SAMLProviderList[].Arn'

# Get SAML provider metadata
aws iam get-saml-provider \
    --saml-provider-arn arn:aws:iam::123456789012:saml-provider/MyLDAPProvider

# Find roles that trust SAML providers
aws iam list-roles --query 'Roles[?contains(to_string(AssumeRolePolicyDocument), `saml-provider`)].[RoleName]' --output text
```

### Monitor SAML Assume Events

```bash
# Find AssumeRoleWithSAML events
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithSAML \
    --max-results 20 | \
    jq '.Events[].CloudTrailEvent | fromjson | {
        time: .eventTime,
        role: .requestParameters.roleArn,
        principal: .requestParameters.principalArn,
        sourceIP: .sourceIPAddress
    }'
```

### Query LDAP Directly

```bash
# Find users in an LDAP group that maps to an AWS role
ldapsearch -x -H ldap://ldap.example.com \
    -D "cn=admin,dc=example,dc=com" -W \
    -b "dc=example,dc=com" \
    "(memberOf=cn=aws-admin-role,ou=groups,dc=example,dc=com)"

# List all LDAP groups that map to AWS roles
ldapsearch -x -H ldap://ldap.example.com \
    -D "cn=admin,dc=example,dc=com" -W \
    -b "ou=groups,dc=example,dc=com" "(cn=aws-*)"
```

## IAM Identity Center (SSO)

### List Permission Sets and Assignments

```bash
# List Identity Center instances
aws sso-admin list-instances

# List permission sets
INSTANCE_ARN=$(aws sso-admin list-instances --query 'Instances[0].InstanceArn' --output text)
aws sso-admin list-permission-sets --instance-arn "$INSTANCE_ARN"

# List account assignments for a permission set
aws sso-admin list-account-assignments \
    --instance-arn "$INSTANCE_ARN" \
    --account-id 123456789012 \
    --permission-set-arn arn:aws:sso:::permissionSet/ssoins-xxx/ps-xxx
```

### Query Identity Store (Users and Groups)

```bash
# Get identity store ID
IDENTITY_STORE_ID=$(aws sso-admin list-instances --query 'Instances[0].IdentityStoreId' --output text)

# List users in identity store
aws identitystore list-users --identity-store-id "$IDENTITY_STORE_ID"

# List groups
aws identitystore list-groups --identity-store-id "$IDENTITY_STORE_ID"

# Get group memberships
aws identitystore list-group-memberships \
    --identity-store-id "$IDENTITY_STORE_ID" \
    --group-id "group-id-here"

# Find a group by name
aws identitystore get-group-id \
    --identity-store-id "$IDENTITY_STORE_ID" \
    --alternate-identifier '{"UniqueAttribute":{"AttributePath":"displayName","AttributeValue":"DevOps"}}'

# Describe a specific user
aws identitystore describe-user \
    --identity-store-id "$IDENTITY_STORE_ID" \
    --user-id "user-id-here"
```

## AWS Managed Microsoft AD

```bash
# List directories
aws ds describe-directories

# Get directory details
aws ds describe-directories --directory-id d-xxxxxxxxxx

# List trusts
aws ds describe-trusts --directory-id d-xxxxxxxxxx
```

## OIDC Providers

```bash
# List OIDC providers
aws iam list-open-id-connect-providers

# Get OIDC provider details
aws iam get-open-id-connect-provider \
    --open-id-connect-provider-arn arn:aws:iam::123456789012:oidc-provider/provider.example.com

# Find roles that trust OIDC providers
aws iam list-roles --query 'Roles[?contains(to_string(AssumeRolePolicyDocument), `oidc-provider`)].[RoleName]' --output text

# Monitor AssumeRoleWithWebIdentity events
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity \
    --max-results 10
```

## Key CloudTrail Events

| Event | Trigger |
|-------|---------|
| `AssumeRole` | IAM user/role assumes another role |
| `AssumeRoleWithSAML` | SAML-federated user assumes a role |
| `AssumeRoleWithWebIdentity` | OIDC/web identity assumes a role |
| `GetFederationToken` | User gets federated session |
| `CreateLoginProfile` | Console password created for user |

```bash
# Monitor all federation events
aws cloudtrail lookup-events \
    --start-time "$(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%SZ)" \
    --lookup-attributes AttributeKey=EventSource,AttributeValue=sts.amazonaws.com \
    --max-results 50
```

## Quick Reference

| Goal | Command |
|------|---------|
| Trust policy | `aws iam get-role --role-name X --query 'Role.AssumeRolePolicyDocument'` |
| Role permissions | `aws iam list-attached-role-policies --role-name X` |
| User's policies | `aws iam list-attached-user-policies --user-name X` |
| Group members | `aws iam get-group --group-name X` |
| SAML providers | `aws iam list-saml-providers` |
| OIDC providers | `aws iam list-open-id-connect-providers` |
| SSO assignments | `aws sso-admin list-account-assignments --instance-arn X --account-id Y --permission-set-arn Z` |
| Identity store users | `aws identitystore list-users --identity-store-id X` |
| AssumeRole events | `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole` |
