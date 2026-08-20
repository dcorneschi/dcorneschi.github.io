# AWS Login: Simplified Developer Access to AWS

The `aws login` command (available in AWS CLI v2.32.0+) provides a secure way to get temporary credentials for local development without creating long-term access keys. It uses the same sign-in method you already use for the AWS Management Console.

## Overview

- No long-term access keys required
- Uses your existing AWS Console sign-in method
- Credentials are automatically rotated every 15 minutes
- Session valid up to the IAM principal's session duration (max 12 hours)
- Works with AWS CLI, AWS SDKs, and AWS Tools for PowerShell
- Available in all AWS commercial Regions (excluding China and GovCloud)
- Uses OAuth 2.0 authorization code flow with PKCE for security

## Prerequisites

```bash
# AWS CLI version 2.32.0 or later required
aws --version

# Update if needed (macOS)
brew upgrade awscli

# Update if needed (Linux)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install --update
rm -rf aws awscliv2.zip
```

## Basic Usage (IAM User or Root)

```bash
# Start the login flow
aws login

# If no default Region is set, you'll be prompted:
# > Enter the AWS Region [None]: us-east-1

# The CLI opens your default browser
# Sign in with your root or IAM user credentials
# Return to the terminal once complete

# Verify your identity
aws sts get-caller-identity
```

## Federated Sign-In

For users authenticating through an organization's identity provider:

```bash
# Start login — opens browser
aws login

# In the browser:
# - Select your active IAM role session from federated sign-in
# - Or sign in through your IdP in another tab, then refresh
# - Supports up to 5 active sessions with multi-session support

# Return to CLI after successful login
aws sts get-caller-identity
```

## Using Profiles

```bash
# Configure a named profile
aws login --profile production

# Run commands with the profile
aws s3 ls --profile production
aws sts get-caller-identity --profile production

# Switch between accounts/roles using different profiles
aws login --profile dev
aws login --profile staging
```

## Remote Development (Headless Servers)

```bash
# On the remote server (no browser available)
aws login --remote

# This provides instructions to complete auth from a device with browser access
# Credentials are delivered to the remote server
```

## AWS Tools for PowerShell

```powershell
# Equivalent command in PowerShell
Invoke-AWSLogin
```

## Using with AWS SDKs

Credentials from `aws login` are automatically available to AWS SDKs through the standard credential provider chain. No additional configuration needed.

```python
# Python (boto3) — credentials are picked up automatically
import boto3
client = boto3.client('s3')
response = client.list_buckets()
```

```javascript
// Node.js — credentials are picked up automatically
const { S3Client, ListBucketsCommand } = require("@aws-sdk/client-s3");
const client = new S3Client({});
const response = await client.send(new ListBucketsCommand({}));
```

## Legacy SDK Support (credential_process)

For older AWS SDKs that don't support the new console credentials provider, use the `credential_process` configuration:

```ini
# ~/.aws/config
[profile legacy-app]
credential_process = aws configure export-credentials --profile default
```

## Session Duration and Expiration

### Default Session Durations

| Principal Type | Default Duration | Maximum |
|----------------|-----------------|---------|
| IAM users | 12 hours | 12 hours |
| IAM roles | 1 hour | 12 hours (configurable via `MaxSessionDuration`) |
| Federated roles | Depends on role config | 12 hours |

Within the session window, short-term credentials rotate automatically every 15 minutes (transparent to you).

### Check Remaining Session Time

```bash
# Show credential expiration (15-min rotation window)
aws configure export-credentials
# Look for the "Expiration" field

# Environment variable format
aws configure export-credentials --format env
# Check AWS_CREDENTIAL_EXPIRATION value

# Verify you're still authenticated
aws sts get-caller-identity
```

### When the Session Expires

Once the overall session duration limit is reached, the next AWS CLI command will prompt you to log in again:

```bash
# Simply re-authenticate
aws login
```

> **Note:** There is no built-in countdown timer. You'll get an authentication error or re-login prompt when the session ends.

## IAM Policy Control

### Required IAM Actions

The `aws login` feature requires two IAM actions:

- `signin:AuthorizeOAuth2Access`
- `signin:CreateOAuth2Token`

### Allow Access (Managed Policy)

```bash
# Attach the managed policy to a user or role
aws iam attach-user-policy \
    --user-name developer \
    --policy-arn arn:aws:iam::aws:policy/SignInLocalDevelopmentAccess

# Or attach to a role
aws iam attach-role-policy \
    --role-name DeveloperRole \
    --policy-arn arn:aws:iam::aws:policy/SignInLocalDevelopmentAccess
```

### Custom IAM Policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "signin:AuthorizeOAuth2Access",
                "signin:CreateOAuth2Token"
            ],
            "Resource": "*"
        }
    ]
}
```

### Deny via SCP (AWS Organizations)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Action": [
                "signin:AuthorizeOAuth2Access",
                "signin:CreateOAuth2Token"
            ],
            "Resource": "*"
        }
    ]
}
```

## CloudTrail Logging

The `aws login` command generates two CloudTrail event types:

| Event Name | Description |
|------------|-------------|
| `AuthorizeOAuth2Access` | Browser-side authorization grant |
| `CreateOAuth2Token` | CLI-side token exchange |

Both events are logged in the Region where the user signs in.

### Sample CloudTrail Event (AuthorizeOAuth2Access)

```json
{
    "eventSource": "signin.amazonaws.com",
    "eventName": "AuthorizeOAuth2Access",
    "awsRegion": "us-east-1",
    "requestParameters": {
        "scope": "openid",
        "redirect_uri": "http://127.0.0.1:53037/oauth/callback",
        "code_challenge_method": "SHA-256",
        "client_id": "arn:aws:signin:::devtools/same-device"
    }
}
```

### Sample CloudTrail Event (CreateOAuth2Token)

```json
{
    "eventSource": "signin.amazonaws.com",
    "eventName": "CreateOAuth2Token",
    "awsRegion": "us-east-1",
    "requestParameters": {
        "client_id": "arn:aws:signin:::devtools/same-device"
    }
}
```

## Security Notes

- Uses OAuth 2.0 with PKCE to prevent authorization code interception
- No long-term credentials stored locally
- Credentials are short-lived and automatically rotated
- Complements centralized root access management in AWS Organizations
- Recommended as an alternative to long-term IAM access keys for development

## Centralized Root Access Management

For organizations using AWS Organizations:

```bash
# After enabling centralized root management and deleting root credentials
# on member accounts, root login is denied on those accounts
# This also prevents programmatic access with root credentials via aws login

# Developers should use IAM users or federated roles instead
aws login --profile my-role
```

## Logout and Session Invalidation

```bash
# End the current session early
aws login logout

# Remove cached credentials for a specific profile
aws login logout --profile production

# Verify you're no longer authenticated
aws sts get-caller-identity
# Should return an error: "Unable to locate credentials"
```

> **Note:** Logging out invalidates the local session. The IAM credentials themselves remain valid on the AWS side until they naturally expire.

## Comparison with Other Auth Methods

| Method | Credentials | Setup Complexity | Best For |
|--------|-------------|-----------------|----------|
| `aws login` | Temporary (auto-rotated) | Minimal — just run it | Quick start, individual devs |
| `aws sso login` | Temporary (auto-rotated) | Medium — requires IAM Identity Center config | Organizations with SSO/IdP |
| Access keys (`aws configure`) | Long-term (static) | Minimal | CI/CD, service accounts (not recommended for humans) |
| `aws sts assume-role` | Temporary (manual refresh) | Medium — requires role trust policy | Cross-account access, elevated privileges |
| Instance profile (EC2/ECS) | Temporary (auto-rotated) | None on the instance | Workloads running on AWS |

### Key Differences: `aws login` vs `aws sso login`

| Feature | `aws login` | `aws sso login` |
|---------|-------------|-----------------|
| Requires IAM Identity Center | No | Yes |
| Works with IAM users | Yes | No |
| Works with federated roles | Yes | Yes (via IdC) |
| Pre-configuration needed | None | `aws configure sso` required |
| Multi-account switching | Via profiles | Built-in account/role picker |
| Ideal for | Solo devs, quick start | Enterprise teams with centralized identity |

## Troubleshooting

### Browser Doesn't Open

```bash
# Manually open the URL printed in the terminal
# Or set a specific browser
export BROWSER=/usr/bin/firefox
aws login
```

### Port Conflict (Callback URL)

```bash
# aws login uses a local HTTP server for the OAuth callback
# If you see "Address already in use":

# Find what's using the port
lsof -i :53037

# Kill it or wait for it to release, then retry
aws login
```

### "Access Denied" or "Not Authorized"

```bash
# Verify the SignInLocalDevelopmentAccess policy is attached
aws iam list-attached-user-policies --user-name your-username

# Or check for the required actions
aws iam simulate-principal-policy \
    --policy-source-arn arn:aws:iam::123456789012:user/your-username \
    --action-names signin:AuthorizeOAuth2Access signin:CreateOAuth2Token
```

### Region Mismatch

```bash
# If commands fail with region errors after login, check your config
aws configure get region

# Set a default region
aws configure set region us-east-1

# Or specify per-command
aws s3 ls --region eu-west-1
```

### "Token has expired" After Working Fine

```bash
# Your overall session duration has been reached
# Simply re-authenticate
aws login

# To check your role's max session duration
aws iam get-role --role-name YourRole --query 'Role.MaxSessionDuration'
# Default: 3600 (1 hour) — can be set up to 43200 (12 hours)
```

### Federated Session Not Appearing

```bash
# If your federated session doesn't show in the browser:
# 1. Sign into AWS Console via your IdP in another tab
# 2. Return to the aws login tab
# 3. Click "Refresh" to detect the new session
```

### Credentials Not Picked Up by SDK

```bash
# Verify credentials are available
aws configure export-credentials

# Check the credential provider chain order:
# 1. Environment variables (AWS_ACCESS_KEY_ID)
# 2. Shared credentials file (~/.aws/credentials)
# 3. AWS config file (~/.aws/config)

# If env vars override aws login credentials, unset them
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN
```

## Quick Reference

| Action | Command |
|--------|---------|
| Login (default profile) | `aws login` |
| Login (named profile) | `aws login --profile <name>` |
| Login (remote/headless) | `aws login --remote` |
| Verify identity | `aws sts get-caller-identity` |
| PowerShell login | `Invoke-AWSLogin` |

## See Also

- [AWS CLI Installation](aws-cli-install.md) — Install AWS CLI v2
- [AWS IAM CLI Cheatsheet](aws-iam-cheatsheet.md) — IAM user and role management
- [AWS STS Assume Role](aws-sts-assume-role.md) — Temporary credentials via AssumeRole
- [AWS IAM Concepts Guide](aws-iam-concepts-guide.md) — IAM fundamentals and Identity Center
