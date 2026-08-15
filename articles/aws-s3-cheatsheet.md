# AWS S3 Cheatsheet

AWS S3 (Simple Storage Service) CLI commands for managing buckets, objects, permissions, lifecycle policies, and data transfers.

## S3 Concepts

### Buckets

- Buckets are containers for objects (files), defined at the region level
- Bucket names must be **globally unique** across all AWS accounts
- Naming rules:
  - 3–63 characters long
  - No uppercase, no underscores
  - Must start with a lowercase letter or number
  - Cannot be an IP address format

### Objects

- Objects have a **key** — the full path within the bucket:

```
s3://my-bucket/my_file.txt
s3://my-bucket/my_folder1/another_folder/my_file.txt
```

- The key is composed of **prefix** + **object name** — there's no concept of directories (just keys with slashes)
- Object values are the content of the body:
  - Max object size: **5 TB**
  - If uploading more than 5 GB, must use **multipart upload**
- Metadata: list of text key/value pairs (system or user metadata)
- Tags: Unicode key/value pairs (up to 10) — useful for lifecycle rules and access policies
- Version ID: present if versioning is enabled

---

## Quick Reference

| Action | Command |
|--------|---------|
| List buckets | `aws s3 ls` |
| List objects in bucket | `aws s3 ls s3://bucket-name/` |
| Copy file to S3 | `aws s3 cp file.txt s3://bucket-name/` |
| Copy file from S3 | `aws s3 cp s3://bucket-name/file.txt .` |
| Sync local to S3 | `aws s3 sync . s3://bucket-name/` |
| Remove object | `aws s3 rm s3://bucket-name/file.txt` |
| Create bucket | `aws s3 mb s3://bucket-name` |
| Delete bucket | `aws s3 rb s3://bucket-name` |
| Move object | `aws s3 mv s3://bucket/old.txt s3://bucket/new.txt` |
| Presigned URL | `aws s3 presign s3://bucket-name/file.txt` |

---

## Bucket Operations

### Create and Delete Buckets

```bash
# Create a bucket
aws s3 mb s3://my-bucket

# Create a bucket in a specific region
aws s3 mb s3://my-bucket --region eu-west-1

# Delete an empty bucket
aws s3 rb s3://my-bucket

# Delete a bucket and all its contents
aws s3 rb s3://my-bucket --force
```

### List Buckets

```bash
# List all buckets
aws s3 ls

# List with creation dates (default output)
aws s3 ls

# List using s3api (JSON output)
aws s3api list-buckets --query 'Buckets[].Name' --output text
```

---

## Object Operations

### List Objects

```bash
# List objects in a bucket
aws s3 ls s3://my-bucket/

# List objects recursively
aws s3 ls s3://my-bucket/ --recursive

# List with human-readable sizes
aws s3 ls s3://my-bucket/ --recursive --human-readable

# List with summary (total size and count)
aws s3 ls s3://my-bucket/ --recursive --human-readable --summarize

# List objects in a prefix (folder)
aws s3 ls s3://my-bucket/logs/2024/
```

### Copy Objects

```bash
# Upload a file to S3
aws s3 cp file.txt s3://my-bucket/

# Upload to a specific prefix (folder)
aws s3 cp file.txt s3://my-bucket/backups/

# Download a file from S3
aws s3 cp s3://my-bucket/file.txt .

# Download to a specific local path
aws s3 cp s3://my-bucket/file.txt /tmp/downloaded.txt

# Copy between buckets
aws s3 cp s3://source-bucket/file.txt s3://dest-bucket/

# Copy an entire folder recursively
aws s3 cp /local/dir/ s3://my-bucket/dir/ --recursive

# Copy with a specific storage class
aws s3 cp file.txt s3://my-bucket/ --storage-class STANDARD_IA

# Copy with server-side encryption
aws s3 cp file.txt s3://my-bucket/ --sse AES256
aws s3 cp file.txt s3://my-bucket/ --sse aws:kms --sse-kms-key-id key-id

# Copy with metadata
aws s3 cp file.txt s3://my-bucket/ --metadata '{"env":"prod","team":"ops"}'

# Copy with content type
aws s3 cp index.html s3://my-bucket/ --content-type "text/html"

# Dry run (show what would be copied)
aws s3 cp /local/dir/ s3://my-bucket/dir/ --recursive --dryrun
```

### Move Objects

```bash
# Move (rename) an object within a bucket
aws s3 mv s3://my-bucket/old-name.txt s3://my-bucket/new-name.txt

# Move from local to S3
aws s3 mv file.txt s3://my-bucket/

# Move from S3 to local
aws s3 mv s3://my-bucket/file.txt .

# Move between buckets
aws s3 mv s3://source-bucket/file.txt s3://dest-bucket/

# Move an entire folder
aws s3 mv s3://my-bucket/old-prefix/ s3://my-bucket/new-prefix/ --recursive
```

### Remove Objects

```bash
# Delete a single object
aws s3 rm s3://my-bucket/file.txt

# Delete all objects in a prefix (folder)
aws s3 rm s3://my-bucket/logs/ --recursive

# Delete all objects in a bucket
aws s3 rm s3://my-bucket/ --recursive

# Dry run (show what would be deleted)
aws s3 rm s3://my-bucket/logs/ --recursive --dryrun

# Delete with include/exclude filters
aws s3 rm s3://my-bucket/ --recursive --exclude "*" --include "*.log"
```

---

## Sync

`sync` compares source and destination, only transferring new or modified files.

```bash
# Sync local directory to S3
aws s3 sync . s3://my-bucket/

# Sync S3 to local directory
aws s3 sync s3://my-bucket/ .

# Sync between buckets
aws s3 sync s3://source-bucket/ s3://dest-bucket/

# Sync and delete files that don't exist in source
aws s3 sync . s3://my-bucket/ --delete

# Sync with exclude patterns
aws s3 sync . s3://my-bucket/ --exclude "*.tmp" --exclude ".git/*"

# Sync with include patterns (exclude everything else first)
aws s3 sync . s3://my-bucket/ --exclude "*" --include "*.log"

# Sync only files larger than 1MB
aws s3 sync . s3://my-bucket/ --exclude "*" --include "*" --size-only

# Dry run
aws s3 sync . s3://my-bucket/ --dryrun

# Sync with specific storage class
aws s3 sync . s3://my-bucket/ --storage-class GLACIER

# Sync with ACL
aws s3 sync . s3://my-bucket/ --acl public-read
```

### Sync Behavior

| Flag | Description |
|------|-------------|
| `--delete` | Delete destination files not in source |
| `--exact-timestamps` | Compare timestamps exactly (default: size + timestamp) |
| `--size-only` | Only compare file size, ignore timestamps |
| `--exclude` | Exclude files matching pattern |
| `--include` | Include files matching pattern (applied after exclude) |
| `--dryrun` | Show what would happen without doing it |

---

## Include/Exclude Filters

Filters are applied in order. `--exclude` first, then `--include`:

```bash
# Upload only .log files
aws s3 cp /var/log/ s3://my-bucket/logs/ --recursive --exclude "*" --include "*.log"

# Upload everything except .tmp and .git
aws s3 sync . s3://my-bucket/ --exclude "*.tmp" --exclude ".git/*"

# Upload only files in a specific subfolder
aws s3 sync . s3://my-bucket/ --exclude "*" --include "data/*"

# Delete only .log files from S3
aws s3 rm s3://my-bucket/ --recursive --exclude "*" --include "*.log"
```

---

## Presigned URLs

Generate temporary access URLs for private objects:

```bash
# Generate a presigned URL (default: 1 hour expiry)
aws s3 presign s3://my-bucket/file.txt

# Generate with custom expiry (seconds)
aws s3 presign s3://my-bucket/file.txt --expires-in 3600

# 7-day expiry (maximum)
aws s3 presign s3://my-bucket/file.txt --expires-in 604800
```

---

## s3api Commands

Lower-level API commands for advanced operations:

### Bucket Configuration

```bash
# Get bucket location (region)
aws s3api get-bucket-location --bucket my-bucket

# Get bucket versioning status
aws s3api get-bucket-versioning --bucket my-bucket

# Enable versioning
aws s3api put-bucket-versioning --bucket my-bucket \
  --versioning-configuration Status=Enabled

# Suspend versioning
aws s3api put-bucket-versioning --bucket my-bucket \
  --versioning-configuration Status=Suspended

Versioning notes:
- Same key overwrite increments the version: 1, 2, 3…
- Protects against unintended deletes (can restore a previous version)
- Files not versioned prior to enabling versioning will have version `null`
- Suspending versioning does not delete previous versions

# Get bucket encryption
aws s3api get-bucket-encryption --bucket my-bucket

# Enable default encryption (SSE-S3)
aws s3api put-bucket-encryption --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'

# Enable default encryption (SSE-KMS)
aws s3api put-bucket-encryption --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "aws:kms", "KMSMasterKeyID": "key-id"}}]
  }'

Encryption methods:

| Method | Description |
|--------|-------------|
| SSE-S3 | Keys managed by AWS (AES-256) — default |
| SSE-KMS | Keys managed by AWS KMS — audit trail via CloudTrail |
| SSE-C | Customer-provided keys — you manage the key, AWS performs encryption |
| Client-Side | Encrypt before uploading — AWS never sees plaintext |

# Get bucket tagging
aws s3api get-bucket-tagging --bucket my-bucket

# Set bucket tags
aws s3api put-bucket-tagging --bucket my-bucket \
  --tagging 'TagSet=[{Key=Environment,Value=prod},{Key=Team,Value=ops}]'
```

### Object Operations

```bash
# Get object metadata (head)
aws s3api head-object --bucket my-bucket --key file.txt

# Get object with specific version
aws s3api get-object --bucket my-bucket --key file.txt --version-id version-id output.txt

# List object versions
aws s3api list-object-versions --bucket my-bucket --prefix file.txt

# Delete a specific version
aws s3api delete-object --bucket my-bucket --key file.txt --version-id version-id

# Restore object from Glacier
aws s3api restore-object --bucket my-bucket --key file.txt \
  --restore-request '{"Days":7,"GlacierJobParameters":{"Tier":"Standard"}}'

# Put object with tagging
aws s3api put-object --bucket my-bucket --key file.txt --body file.txt \
  --tagging "env=prod&team=ops"

# Get object tags
aws s3api get-object-tagging --bucket my-bucket --key file.txt

# Copy object and change storage class
aws s3api copy-object --bucket my-bucket --key file.txt \
  --copy-source my-bucket/file.txt --storage-class GLACIER
```

### Access Control

```bash
# Get bucket ACL
aws s3api get-bucket-acl --bucket my-bucket

# Get bucket policy
aws s3api get-bucket-policy --bucket my-bucket

# Put bucket policy from file
aws s3api put-bucket-policy --bucket my-bucket --policy file://policy.json

# Delete bucket policy
aws s3api delete-bucket-policy --bucket my-bucket

# Block all public access
aws s3api put-public-access-block --bucket my-bucket \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Get public access block settings
aws s3api get-public-access-block --bucket my-bucket
```

### Restrict a Bucket to a Specific User

IAM roles cannot be assigned directly to S3 buckets. To restrict access, use an IAM policy on the user, a bucket policy on the bucket, or both.

**IAM Policy on the User** — grant access only to a specific bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-private-bucket",
        "arn:aws:s3:::my-private-bucket/*"
      ]
    }
  ]
}
```

**Bucket Policy (Deny Everyone Else)** — deny all principals except a specific user:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-private-bucket",
        "arn:aws:s3:::my-private-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": "arn:aws:iam::123456789012:user/specific-user"
        }
      }
    }
  ]
}
```

| Approach | Controls | Use when |
|----------|----------|----------|
| IAM policy | What the user can do | You want to grant access to the user |
| Bucket policy | Who can touch the bucket | You want to lock down the bucket itself |

Best practices:
- Replace `s3:*` with specific actions (`s3:GetObject`, `s3:PutObject`) for least-privilege
- An explicit `Deny` overrides any other `Allow`, even from other policies
- Always include both the bucket ARN and the objects ARN (`/*`) in the Resource field
- For strongest isolation, use both approaches together

---

## Lifecycle Rules

```bash
# Get lifecycle configuration
aws s3api get-bucket-lifecycle-configuration --bucket my-bucket

# Set lifecycle rules from file
aws s3api put-bucket-lifecycle-configuration --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json

# Delete lifecycle configuration
aws s3api delete-bucket-lifecycle --bucket my-bucket
```

### Example lifecycle.json

```json
{
  "Rules": [
    {
      "ID": "Move to IA after 30 days",
      "Status": "Enabled",
      "Filter": {"Prefix": "logs/"},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER"}
      ],
      "Expiration": {"Days": 365}
    },
    {
      "ID": "Delete incomplete multipart uploads",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }
  ]
}
```

---

## Storage Classes

| Class | Use Case | Retrieval |
|-------|----------|-----------|
| `STANDARD` | Frequently accessed data | Instant |
| `STANDARD_IA` | Infrequent access, still needs fast retrieval | Instant (per-GB retrieval fee) |
| `ONEZONE_IA` | Infrequent, single AZ (non-critical) | Instant |
| `INTELLIGENT_TIERING` | Unknown or changing access patterns | Instant |
| `REDUCED_REDUNDANCY` | Non-critical, reproducible data (deprecated) | Instant |
| `GLACIER_IR` | Archive with instant retrieval | Instant (per-GB fee) |
| `GLACIER` | Long-term archive | Minutes to hours |
| `DEEP_ARCHIVE` | Rarely accessed archive | 12–48 hours |

- **Durability** — how likely data is to be lost due to hardware failure (99.999999999% for all classes except RRS)
- **Availability** — the data is there to retrieve when you request it (varies by class)

```bash
# Upload with specific storage class
aws s3 cp file.txt s3://my-bucket/ --storage-class STANDARD_IA

# Change storage class of existing object (copy in place)
aws s3 cp s3://my-bucket/file.txt s3://my-bucket/file.txt --storage-class GLACIER

# Check storage class of an object
aws s3api head-object --bucket my-bucket --key file.txt --query 'StorageClass'
```

---

## Multipart Uploads

For files larger than 5 GB (automatically used by `aws s3 cp` for files > 8 MB):

```bash
# Configure multipart threshold and chunk size
aws configure set s3.multipart_threshold 64MB
aws configure set s3.multipart_chunksize 16MB

# List incomplete multipart uploads
aws s3api list-multipart-uploads --bucket my-bucket

# Abort a specific multipart upload
aws s3api abort-multipart-upload --bucket my-bucket --key file.txt --upload-id upload-id

# Abort all incomplete multipart uploads (using lifecycle or manually)
aws s3api list-multipart-uploads --bucket my-bucket \
  --query 'Uploads[].{Key:Key,UploadId:UploadId}' --output text | \
  while read key upload_id; do
    aws s3api abort-multipart-upload --bucket my-bucket --key "$key" --upload-id "$upload_id"
  done
```

---

## Transfer Acceleration

```bash
# Enable transfer acceleration on a bucket
aws s3api put-bucket-accelerate-configuration --bucket my-bucket \
  --accelerate-configuration Status=Enabled

# Use accelerated endpoint
aws s3 cp file.txt s3://my-bucket/ --endpoint-url https://s3-accelerate.amazonaws.com

# Check acceleration status
aws s3api get-bucket-accelerate-configuration --bucket my-bucket
```

---

## Static Website Hosting

```bash
# Enable static website hosting
aws s3 website s3://my-bucket/ --index-document index.html --error-document error.html

# Get website configuration
aws s3api get-bucket-website --bucket my-bucket

# Delete website configuration
aws s3api delete-bucket-website --bucket my-bucket

# Upload website files with correct content types
aws s3 sync ./site/ s3://my-bucket/ --delete \
  --exclude "*" --include "*.html" --content-type "text/html"
aws s3 sync ./site/ s3://my-bucket/ --delete \
  --exclude "*" --include "*.css" --content-type "text/css"
aws s3 sync ./site/ s3://my-bucket/ --delete \
  --exclude "*" --include "*.js" --content-type "application/javascript"
```

Website endpoint: `http://bucket-name.s3-website-region.amazonaws.com`

---

## Bucket Notifications

```bash
# Get notification configuration
aws s3api get-bucket-notification-configuration --bucket my-bucket

# Set notification (sends to SNS on object creation)
aws s3api put-bucket-notification-configuration --bucket my-bucket \
  --notification-configuration file://notification.json
```

---

## Logging and Metrics

```bash
# Enable server access logging
aws s3api put-bucket-logging --bucket my-bucket \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "my-logs-bucket",
      "TargetPrefix": "s3-access-logs/"
    }
  }'

# Get logging configuration
aws s3api get-bucket-logging --bucket my-bucket

# Get bucket metrics
aws s3api list-bucket-metrics-configurations --bucket my-bucket
```

---

## CORS Configuration

```bash
# Get CORS rules
aws s3api get-bucket-cors --bucket my-bucket

# Set CORS rules
aws s3api put-bucket-cors --bucket my-bucket --cors-configuration file://cors.json

# Delete CORS configuration
aws s3api delete-bucket-cors --bucket my-bucket
```

### Example cors.json

```json
{
  "CORSRules": [
    {
      "AllowedHeaders": ["*"],
      "AllowedMethods": ["GET", "PUT", "POST"],
      "AllowedOrigins": ["https://example.com"],
      "ExposeHeaders": ["ETag"],
      "MaxAgeSeconds": 3600
    }
  ]
}
```

---

## Replication

```bash
# Get replication configuration
aws s3api get-bucket-replication --bucket my-bucket

# Set replication (requires versioning enabled on both buckets)
aws s3api put-bucket-replication --bucket my-bucket \
  --replication-configuration file://replication.json

# Delete replication configuration
aws s3api delete-bucket-replication --bucket my-bucket
```

---

## Performance and Transfer Configuration

Configure in `~/.aws/config`:

```ini
[default]
s3 =
  max_concurrent_requests = 20
  max_queue_size = 10000
  multipart_threshold = 64MB
  multipart_chunksize = 16MB
  max_bandwidth = 50MB/s
  use_accelerate_endpoint = false
  addressing_style = path
```

| Setting | Default | Description |
|---------|---------|-------------|
| `max_concurrent_requests` | 10 | Parallel transfer threads |
| `multipart_threshold` | 8MB | File size to trigger multipart upload |
| `multipart_chunksize` | 8MB | Size of each multipart chunk |
| `max_bandwidth` | None | Bandwidth throttle (e.g., `50MB/s`) |
| `max_queue_size` | 1000 | Maximum number of tasks in the queue |
| `use_accelerate_endpoint` | false | Use transfer acceleration |

---

## Useful One-Liners

```bash
# Total size of a bucket
aws s3 ls s3://my-bucket/ --recursive --summarize | tail -2

# Count objects in a bucket
aws s3 ls s3://my-bucket/ --recursive | wc -l

# Find largest objects
aws s3 ls s3://my-bucket/ --recursive --human-readable | sort -k3 -rh | head -20

# Find objects older than 90 days
aws s3api list-objects-v2 --bucket my-bucket --query \
  "Contents[?LastModified<='2024-01-01'].{Key:Key,Size:Size,Modified:LastModified}"

# Empty a bucket (delete all objects)
aws s3 rm s3://my-bucket/ --recursive

# Empty a versioned bucket (delete all versions and markers)
aws s3api list-object-versions --bucket my-bucket --output json \
  --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' | \
  aws s3api delete-objects --bucket my-bucket --delete file:///dev/stdin

# Copy all objects between accounts (with source bucket policy allowing access)
aws s3 sync s3://source-bucket/ s3://dest-bucket/ --source-region us-east-1 --region eu-west-1

# Upload with progress
aws s3 cp large-file.tar.gz s3://my-bucket/ --no-progress  # suppress progress
# (progress shown by default for large files)

# Find all public objects (using bucket ACL)
aws s3api get-bucket-acl --bucket my-bucket

# List all buckets with their region
aws s3api list-buckets --query 'Buckets[].Name' --output text | \
  xargs -I{} sh -c 'echo "{}: $(aws s3api get-bucket-location --bucket {} --query LocationConstraint --output text)"'

# Calculate cost estimate (get total size per storage class)
aws s3api list-objects-v2 --bucket my-bucket --query \
  "Contents[].{Key:Key,Size:Size,StorageClass:StorageClass}" --output table

# Generate inventory of all objects as CSV
aws s3 ls s3://my-bucket/ --recursive | awk '{print $1","$2","$3","$4}' > inventory.csv

# Batch delete objects matching a pattern
aws s3 rm s3://my-bucket/ --recursive --exclude "*" --include "*.tmp"

# Mirror a bucket (sync both directions)
aws s3 sync s3://bucket-a/ s3://bucket-b/
aws s3 sync s3://bucket-b/ s3://bucket-a/
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `AccessDenied` | Missing IAM permissions | Check bucket policy and IAM role |
| `NoSuchBucket` | Bucket doesn't exist or wrong region | Verify bucket name and `--region` |
| `BucketAlreadyExists` | Bucket name globally taken | Choose a different name |
| `InvalidBucketName` | Name doesn't meet requirements | Use lowercase, 3–63 chars, no underscores |
| `SlowTransfer` | Default concurrency too low | Increase `max_concurrent_requests` |
| `EntityTooLarge` | Single PUT > 5 GB | Use multipart upload (automatic for `aws s3 cp`) |
| `403 on presigned URL` | URL expired or wrong credentials | Check `--expires-in` and signing credentials |
| `PermanentRedirect` | Wrong region for bucket | Add `--region` matching bucket location |
| Sync not detecting changes | Timestamps match, size same | Use `--exact-timestamps` or `--size-only` |

### Debug Commands

```bash
# Enable debug output
aws s3 ls s3://my-bucket/ --debug

# Check your identity
aws sts get-caller-identity

# Test access to a bucket
aws s3 ls s3://my-bucket/ 2>&1

# Check bucket policy
aws s3api get-bucket-policy --bucket my-bucket --output text | python3 -m json.tool
```

---

## Useful Links

- What is AWS S3: https://digitalcloud.training/what-is-aws-s3
- AWS S3 Documentation: https://docs.aws.amazon.com/s3/
