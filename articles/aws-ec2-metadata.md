# EC2 Instance Metadata Service (IMDS) Cheatsheet

The Instance Metadata Service (IMDS) provides information about a running EC2 instance — accessible only from within the instance via a link-local address. No AWS credentials required.

## Endpoints

| Protocol | Address |
|----------|---------|
| IPv4 | `http://169.254.169.254/latest/meta-data/` |
| IPv6 | `http://[fd00:ec2::254]/latest/meta-data/` |
| User data | `http://169.254.169.254/latest/user-data/` |
| Dynamic data | `http://169.254.169.254/latest/dynamic/` |

## IMDSv1 vs IMDSv2

IMDSv2 uses a session token (PUT request) and is more secure against SSRF attacks. IMDSv1 is a simple GET request.

### IMDSv2 (Recommended)

```sh
# Get a token (valid for 6 hours by default)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Use the token in requests
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
```

### IMDSv1 (Legacy)

```sh
# Simple GET (no token)
curl -s http://169.254.169.254/latest/meta-data/instance-id
```

> If IMDSv1 returns no response, your instance likely requires IMDSv2. Check with `aws ec2 describe-instances --instance-id <id> --query "Reservations[0].Instances[0].MetadataOptions"`.

## Common Metadata Queries

### Helper Function

```sh
# Reusable function for IMDSv2 calls
imds() {
  local token
  token=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
  curl -s -H "X-aws-ec2-metadata-token: $token" \
    "http://169.254.169.254/latest/meta-data/$1"
}

# Usage
imds instance-id
imds instance-type
imds local-ipv4
```

### Instance Identity

```sh
# Instance ID
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
# i-0abc123def456

# Instance type
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-type
# m5.xlarge

# AMI ID
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/ami-id
# ami-0abcdef1234567890

# Hostname
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/hostname
# ip-10-0-1-42.ec2.internal

# Local hostname
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-hostname
```

### Networking

```sh
# Private IP
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4
# 10.0.1.42

# Public IP (empty if no public IP assigned)
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4
# 54.123.45.67

# Public hostname
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-hostname

# MAC address
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/mac

# Security groups
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/security-groups

# VPC ID (via network interface)
MAC=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/mac)
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/network/interfaces/macs/$MAC/vpc-id

# Subnet ID
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/network/interfaces/macs/$MAC/subnet-id
```

### Placement and Region

```sh
# Availability Zone
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone
# us-east-1a

# Region
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/region
# us-east-1
```

### IAM Role Credentials

```sh
# Get the IAM role name attached to the instance
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Get temporary credentials for the role
ROLE=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/)
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE
```

The credentials response includes `AccessKeyId`, `SecretAccessKey`, `Token`, and `Expiration`.

### Instance Identity Document

```sh
# Full identity document (JSON)
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/dynamic/instance-identity/document
```

Returns:

```json
{
  "accountId": "123456789012",
  "architecture": "x86_64",
  "availabilityZone": "us-east-1a",
  "billingProducts": null,
  "imageId": "ami-0abcdef1234567890",
  "instanceId": "i-0abc123def456",
  "instanceType": "m5.xlarge",
  "kernelId": null,
  "pendingTime": "2024-01-15T10:30:00Z",
  "privateIp": "10.0.1.42",
  "region": "us-east-1",
  "version": "2017-09-30"
}
```

### User Data

```sh
# Get user data (base64-decoded automatically)
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/user-data
```

### Tags (If Enabled)

Instance tags are available in metadata only if you enabled "Allow tags in instance metadata":

```sh
# List available tags
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/tags/instance/

# Get a specific tag value
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/tags/instance/Name
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/tags/instance/Environment
```

Enable with:

```sh
aws ec2 modify-instance-metadata-options \
  --instance-id <id> \
  --instance-metadata-tags enabled
```

## The ec2-metadata Script

Amazon Linux includes an `ec2-metadata` helper script that wraps IMDS calls:

```sh
# All metadata
ec2-metadata --all

# Specific fields
ec2-metadata -i    # instance-id
ec2-metadata -t    # instance-type
ec2-metadata -o    # local-ipv4
ec2-metadata -v    # public-ipv4
ec2-metadata -z    # availability-zone
ec2-metadata -a    # ami-id
ec2-metadata -s    # security-groups
ec2-metadata -h    # local-hostname
ec2-metadata -p    # public-hostname
ec2-metadata -e    # public-keys
```

### ec2-metadata Options

| Flag | Field |
|------|-------|
| `-a` | AMI ID |
| `-b` | Block device mapping |
| `-d` | User data |
| `-e` | Public keys |
| `-h` | Local hostname |
| `-i` | Instance ID |
| `-k` | Kernel ID |
| `-l` | AMI launch index |
| `-m` | AMI manifest path |
| `-n` | Ancestor AMI IDs |
| `-o` | Local (private) IPv4 |
| `-p` | Public hostname |
| `-r` | Reservation ID |
| `-s` | Security groups |
| `-t` | Instance type |
| `-u` | Instance identity document (dynamic) |
| `-v` | Public IPv4 |
| `-z` | Availability zone |

### Installing ec2-metadata on Non-Amazon Linux

```sh
# Download the script
wget http://s3.amazonaws.com/ec2metadata/ec2-metadata
chmod +x ec2-metadata
./ec2-metadata --all

# Or install to a system path
wget http://s3.amazonaws.com/ec2metadata/ec2-metadata -O /usr/local/bin/ec2-metadata
chmod +x /usr/local/bin/ec2-metadata
ec2-metadata --all
```

> **Note:** The `ec2-metadata` script uses IMDSv1. For IMDSv2-only instances, use `curl` with tokens or the AWS CLI `ec2 describe-*` commands instead.

## ec2-instance-connect (Alternative)

On newer Amazon Linux 2 and AL2023, the `ec2-metadata` command is available directly as a package:

```sh
# Amazon Linux 2023
dnf install ec2-instance-connect

# Verify
ec2-metadata --help
```

## One-Liners and Recipes

```sh
# Get region without trailing AZ letter
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/region

# Set AWS_DEFAULT_REGION from metadata
export AWS_DEFAULT_REGION=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/region)

# Get account ID from identity document
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.accountId'

# Tag-based hostname (if tags in metadata enabled)
hostnamectl set-hostname $(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/tags/instance/Name)

# Wait for metadata service to be available (useful in userdata scripts)
until curl -s -o /dev/null -w "%{http_code}" http://169.254.169.254/latest/meta-data/ | grep -q 200; do
  sleep 1
done

# Get all network interface IPs
MAC=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/mac)
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/network/interfaces/macs/$MAC/local-ipv4s

# Self-terminating instance (for spot/batch jobs)
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
aws ec2 terminate-instances --instance-ids $INSTANCE_ID

# Check if instance is a spot instance
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-life-cycle
# on-demand or spot

# Get spot termination notice (2 min warning)
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/spot/termination-time
```

## Configuring IMDS

```sh
# Require IMDSv2 (disable IMDSv1)
aws ec2 modify-instance-metadata-options \
  --instance-id <id> \
  --http-tokens required \
  --http-endpoint enabled

# Set hop limit (increase for containers)
aws ec2 modify-instance-metadata-options \
  --instance-id <id> \
  --http-put-response-hop-limit 2

# Disable IMDS entirely
aws ec2 modify-instance-metadata-options \
  --instance-id <id> \
  --http-endpoint disabled

# Check current IMDS settings
aws ec2 describe-instances --instance-id <id> \
  --query "Reservations[0].Instances[0].MetadataOptions" --output json
```

## Gotchas

- **Not protected by authentication**: Anyone with shell access on the instance can read all metadata, including IAM credentials. Never store secrets in user data.
- **IMDSv1 is vulnerable to SSRF**: If your application is exposed to the internet, enforce IMDSv2 to prevent server-side request forgery attacks.
- **Hop limit**: Default is 1. Docker/ECS containers add a hop, so IMDS calls fail from containers unless you increase the hop limit to 2.
- **1024 PPS limit**: IMDS shares the link-local rate limit with DNS (Route 53 Resolver), NTP, and Windows licensing — all capped at 1024 packets/second combined.
- **Tags not available by default**: Instance tags are NOT in metadata unless explicitly enabled with `--instance-metadata-tags enabled`.
- **ec2-metadata script uses IMDSv1**: The bundled helper script doesn't support IMDSv2. If you've enforced IMDSv2, use `curl` with tokens instead.
- **IPv6 requires Nitro**: The `[fd00:ec2::254]` endpoint only works on Nitro-based instances in IPv6-enabled subnets.
- **User data is limited to 16 KB** (raw, before base64 encoding).
