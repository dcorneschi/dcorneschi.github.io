# AWS API Rate Limits, Quotas and Throttling

A practical reference for understanding and dealing with AWS API throttling, service quotas, and rate limits — especially in the context of Terraform and automation.

## How AWS API Throttling Works

AWS uses a token bucket algorithm per account, per region, per API action:

- **Bucket size** (burst capacity): The maximum number of requests you can make instantly
- **Refill rate** (sustained rate): Tokens added per second once the bucket is partially or fully empty
- When the bucket is empty: `RequestLimitExceeded` or `Throttling` error
- Requests are rejected until tokens refill

```
┌─────────────────────────────────┐
│  Token Bucket (e.g. 100 max)    │
│  ████████████████░░░░░░░░░░░░░  │  ← 60 tokens remaining
│                                 │
│  Refill: +20 tokens/sec         │
│  Each API call: -1 token        │
└─────────────────────────────────┘
```

## Types of AWS Throttling

| Type | Scope | Example |
|------|-------|---------|
| Request rate limiting | Per API action | 100 DescribeLaunchTemplates calls burst, 20/sec sustained |
| Resource rate limiting | Per resource affected | RunInstances: 1000 instances burst, 2/sec sustained |
| Service quotas | Per resource type | Max 5000 security groups per VPC |
| Account-level limits | Cross-service | Max concurrent Lambda invocations |

## EC2 API Rate Limits

### By Category

| Category | Actions | Bucket Size | Refill Rate |
|----------|---------|:-----------:|:-----------:|
| Non-mutating | Describe*, List*, Get* (general) | 100 | 20/sec |
| Unfiltered non-mutating | DescribeInstances, DescribeVolumes, DescribeSecurityGroups, DescribeSnapshots, DescribeNetworkInterfaces (without pagination or filters) | 50 | 10/sec |
| Mutating | Create*, Modify*, Delete* (general) | 50 | 5/sec |
| Resource-intensive | AuthorizeSecurityGroupIngress, RequestSpotInstances, AcceptVpcPeeringConnection | 50 | 5/sec |
| Console non-mutating | Describe/List/Get calls from the AWS console | 100 | 10/sec |

### Notable Individual Limits

| API Action | Bucket Size | Refill Rate |
|-----------|:-----------:|:-----------:|
| `RunInstances` | 5 (requests) / 1000 (resources) | 2/sec |
| `TerminateInstances` | 100 (requests) / 1000 (resources) | 5/sec (requests) / 20/sec (resources) |
| `StartInstances` | 5 (requests) / 1000 (resources) | 2/sec |
| `CreateVpcEndpoint` | 4 | 0.3/sec |
| `CopyImage` | 100 | 1/sec |
| `CreateNatGateway` | 10 | 1/sec |
| `DescribeInstanceTopology` | 1 | 1/sec |

## Other AWS Services — Common Rate Limits

| Service | Typical Limit | Notes |
|---------|--------------|-------|
| IAM | ~10-20 req/sec per action | Very low, shared across account globally (not per-region) |
| STS | ~100 req/sec for AssumeRole | Shared bucket across AssumeRole variants |
| S3 | 5,500 GET/sec and 3,500 PUT/sec per prefix | Per-prefix, not per-bucket |
| Lambda | 10x concurrent invocations for invoke rate | Burst up to 500-3000 depending on region |
| ELB | Similar token bucket to EC2 | ~20-50 req/sec for Describe calls |
| CloudFormation | ~1 req/sec for stack operations | Very aggressive throttling |
| Route53 | 5 req/sec per API action | Very low |
| EKS | ~10-20 req/sec for Describe calls | Lower than EC2 |

## Error Responses

Different services use different error codes:

| Error Code | Service(s) | Meaning |
|-----------|-----------|---------|
| `RequestLimitExceeded` | EC2 | Token bucket empty |
| `Throttling` | Most AWS services | General throttle |
| `ThrottlingException` | IAM, STS, newer services | General throttle |
| `TooManyRequestsException` | Lambda, API Gateway | Rate exceeded |
| `Rate exceeded` | Various | Generic rate limit |
| `SlowDown` | S3 | Too many requests to a prefix |

## Terraform-Specific Symptoms

### What It Looks Like

```bash
# Terraform retries with backoff (up to 25 attempts)
retrying request EC2/DescribeLaunchTemplates, attempt 5
RequestLimitExceeded: Request limit exceeded.
```

Terraform plan hangs or takes much longer than expected.

### Quick Fixes

```bash
# Reduce parallelism (default is 10)
terraform plan -parallelism=2

# Skip refresh (uses cached state)
terraform plan -refresh=false

# Target specific modules
terraform plan -target=module.specific_module
```

### Long-Term Terraform Fixes

| Fix | Effect |
|-----|--------|
| Reduce parallelism | `terraform plan -parallelism=2` — fewer concurrent API calls |
| Skip refresh | `terraform plan -refresh=false` — no Describe calls |
| Target modules | `terraform plan -target=module.x` — refresh subset of resources |
| Split state | Separate root modules — each plan refreshes fewer resources |
| Use data source sparingly | Each data source triggers API calls during refresh |
| Import vs data source | Importing reduces ongoing Describe calls |

## Viewing Your Limits

### Service Quotas Console

1. Go to https://console.aws.amazon.com/servicequotas/home/services/ec2/quotas/
2. Search for the API action name (e.g., `DescribeLaunchTemplates`)
3. Two entries per action:
   - `{API} request bucket maximum capacity` → burst
   - `{API} request bucket refill rate` → sustained rate
4. "Applied account-level quota" = your effective limit
5. "AWS default quota" = the baseline

### CLI

```bash
# List all EC2 quotas
aws service-quotas list-service-quotas --service-code ec2

# Get a specific quota
aws service-quotas get-service-quota \
    --service-code ec2 \
    --quota-code L-0B52B4C4

# List quotas that have been overridden
aws service-quotas list-service-quotas --service-code ec2 \
    --query "Quotas[?Value != DefaultValue]"
```

### CloudWatch Usage Metrics

```bash
# If Usage metrics are published for the action
# Namespace: AWS/Usage
# MetricName: CallCount
# Dimensions:
#   Service: EC2
#   Type: API
#   Resource: DescribeLaunchTemplates

# Not all API actions publish CallCount metrics
```

## Detecting Throttling

### CloudTrail — Event History

```bash
# All throttled calls in the last hour
aws cloudtrail lookup-events \
    --max-results 50 \
    --start-time "$(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ)" | \
    jq '.Events[].CloudTrailEvent | fromjson | select(.errorCode == "RequestLimitExceeded" or .errorCode == "Throttling") | {
      eventTime,
      eventName,
      userAgent: .userAgent,
      sourceIP: .sourceIPAddress,
      errorCode
    }'
```

### Find Who's Making the Calls

```bash
# Lookup specific API action
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=DescribeLaunchTemplates \
    --max-results 20 | jq '.Events[].CloudTrailEvent | fromjson | {
      eventTime,
      userIdentity: .userIdentity.arn,
      sourceIP: .sourceIPAddress,
      userAgent: .userAgent,
      errorCode
    }'
```

### Count Calls by User Agent

```bash
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=DescribeLaunchTemplates \
    --max-results 50 | \
    jq '[.Events[].CloudTrailEvent | fromjson | .userAgent] | group_by(.) | map({agent: .[0], count: length}) | sort_by(-.count)'
```

### Common Culprits

| User Agent | Source |
|-----------|--------|
| `HashiCorp/1.0 Terraform/...` | Terraform runs |
| `autoscaling.amazonaws.com` | ASG polling launch templates |
| `aws-sdk-go/...` | Custom tooling or CI scripts |
| `Boto3/...` | Python scripts/Lambda |
| `console.ec2.amazonaws.com` | Someone in the AWS console |
| `datadog-agent/...` | Monitoring tools |

### In Datadog (If AWS Integration Active)

```bash
# CloudTrail logs
# source:cloudtrail @error.kind:RequestLimitExceeded

# By API action
# source:cloudtrail @error.kind:RequestLimitExceeded | group by @evt.name

# By caller
# source:cloudtrail @error.kind:RequestLimitExceeded | group by @userIdentity.arn

# Usage metrics
# sum:aws.usage.call_count{service:ec2} by {resource}
```

### In AWS Console

1. **CloudTrail** → Event history → Filter: Error code = `RequestLimitExceeded`
2. **CloudWatch** → Metrics → AWS/Usage → CallCount (if published)
3. **Service Quotas** → EC2 → search the API action name

## Requesting a Limit Increase

### Via Service Quotas (Preferred)

```bash
# Find the quota code
aws service-quotas list-service-quotas --service-code ec2 \
    --query "Quotas[?contains(QuotaName, 'DescribeLaunchTemplates')]"

# Request increase
aws service-quotas request-service-quota-increase \
    --service-code ec2 \
    --quota-code L-0B52B4C4 \
    --desired-value 200
```

### Via Support Case

1. AWS Console → Support → Create case
2. Category: Service limit increase
3. Service: EC2 API
4. Include: region, API action, current error rate, desired limit, business justification

Allow 24-48 hours for approval. Changes take up to 24 hours to reflect in Service Quotas.

## Mitigations and Best Practices

### General Best Practices

| Practice | Why |
|----------|-----|
| Implement exponential backoff with jitter | Spreads retries, avoids thundering herd |
| Use pagination and filters on Describe calls | Avoids the smaller "unfiltered" bucket (50 instead of 100) |
| Cache API responses | Avoid redundant calls within short windows |
| Stagger automation schedules | Don't run all CI pipelines at :00 |
| Use EventBridge instead of polling | React to changes vs. constant Describe calls |
| Consolidate tooling | One source of truth instead of 5 tools all calling DescribeInstances |

### Retry Strategy (SDK Default)

```bash
# Exponential backoff with jitter:
# Attempt 1: immediate
# Attempt 2: wait ~1 sec
# Attempt 3: wait ~2 sec
# Attempt 4: wait ~4 sec
# ...
# Max: 25 attempts (Terraform AWS provider default)

# Add jitter (random 0-1 sec) to avoid synchronized retries
# across parallel workers
```

## Key Domains

| Domain | Service | Region |
|--------|---------|--------|
| `ec2.us-east-1.amazonaws.com` | EC2 API | us-east-1 |
| `compute-1.amazonaws.com` | EC2 public DNS hostnames | us-east-1 |
| `ec2.<region>.amazonaws.com` | EC2 API | Other regions |
| `<region>.compute.amazonaws.com` | EC2 public DNS hostnames | Other regions |

## AWS by the Numbers

| Metric | Count |
|--------|-------|
| Total AWS services | 240+ |
| API actions across all services | 15,000+ (estimated) |
| AWS Regions | 39 |
| Availability Zones | 120+ |

## API Protocol Types

| Protocol | Services | Characteristics |
|----------|----------|----------------|
| REST (JSON) | S3, API Gateway, Lambda, EKS, DynamoDB | Standard HTTP verbs, JSON payloads |
| Query (XML) | EC2, IAM, STS, SQS, CloudFormation | URL-encoded parameters, XML responses |
| JSON-RPC | DynamoDB (legacy), Step Functions, SWF | JSON payloads with target header |

All AWS APIs use SigV4 (Signature Version 4) for authentication, regardless of protocol.

## Additional Service Rate Limits

### IAM and STS

| Action | Rate Limit | Scope |
|--------|:---------:|-------|
| Most IAM actions | ~10-20 req/sec | Global (not per-region) |
| AssumeRole | ~100 req/sec | Global |
| GetCallerIdentity | ~100 req/sec | Global |
| CreateRole / DeleteRole | ~5 req/sec | Global |

IAM and STS are global services — the rate limit is shared across ALL regions. Cannot be increased via Service Quotas.

### S3

| Operation Type | Rate Limit | Scope |
|---------------|:---------:|-------|
| GET/HEAD | 5,500 req/sec | Per prefix |
| PUT/COPY/POST/DELETE | 3,500 req/sec | Per prefix |
| LIST | 5,500 req/sec | Per prefix |
| Control plane (CreateBucket, etc.) | ~100 req/sec | Per account, per region |

S3 data plane limits are per prefix, not per bucket. Distribute across prefixes for virtually unlimited throughput.

### Lambda

| Quota | Default Limit | Notes |
|-------|:---:|-------|
| Concurrent executions | 1,000 (soft) | Per account, per region |
| Burst concurrency | 500-3,000 | Depends on region |
| Invocation rate | 10x concurrent limit | Per function |
| Control plane (CreateFunction) | ~10-15 req/sec | Per account |

### DynamoDB

| Quota | Default Limit | Notes |
|-------|:---:|-------|
| Read Capacity Units (RCU) | 40,000 per table | Adjustable |
| Write Capacity Units (WCU) | 40,000 per table | Adjustable |
| Control plane (CreateTable) | ~10 req/sec | Per account |

Throttled requests return `ProvisionedThroughputExceededException`.

### ECS

| Category | Bucket Size | Refill Rate |
|----------|:-----------:|:-----------:|
| Mutating (RunTask, etc.) | 100 | 20/sec |
| Non-mutating (DescribeTask, etc.) | 100 | 40/sec |
| Fargate task launches | 100 | 20/sec |

### CloudFormation

| Action | Rate Limit | Notes |
|--------|:---------:|-------|
| Stack operations (Create/Update/Delete) | ~1-3 req/sec | Extremely aggressive |
| Describe operations | ~5-10 req/sec | Per account |
| Change sets | ~1-3 req/sec | Per account |

### Route 53

| Action | Rate Limit | Notes |
|--------|:---------:|-------|
| All API actions | 5 req/sec | Per account — very low |
| DNS queries | No practical limit | Handled by resolvers globally |

Route 53 API limits are among the lowest in AWS. Cannot be increased.

### API Gateway (Your APIs)

| Quota | Default |
|-------|:-------:|
| Account-level RPS | 10,000 req/sec |
| Burst capacity | 5,000 requests |
| WebSocket connections | 500 new/sec |

### CloudWatch

| Category | Rate Limit |
|----------|:---------:|
| PutMetricData | 500 TPS |
| GetMetricData | 50 TPS |
| PutLogEvents | 5,000 req/sec per log group |
| DescribeAlarms | 9 TPS |
| FilterLogEvents | 5 TPS per log group |

### SQS

| Category | Rate Limit |
|----------|:---------:|
| Standard queues | Virtually unlimited |
| FIFO queues | 3,000 msg/sec (batching) / 300 without |
| Control plane (CreateQueue) | ~30 req/sec |

## Most Aggressively Throttled Services

| Rank | Service | Effective Limit | Impact |
|:----:|---------|:--------------:|--------|
| 1 | Route 53 | 5 req/sec | Terraform with many DNS records |
| 2 | CloudFormation | ~1-3 req/sec | Large stack deployments |
| 3 | IAM/STS | ~10-20 req/sec (global) | Shared across ALL regions, no increase |
| 4 | EKS | ~10-20 req/sec | Autoscaler + CI pipelines |
| 5 | CloudWatch (read) | 5-50 TPS | Monitoring tools |
| 6 | EC2 (mutating) | 50 burst / 5/sec | Scale events + Terraform |
| 7 | Lambda (control plane) | ~10-15 req/sec | Many-function deployments |
| 8 | DynamoDB (control plane) | ~10 req/sec | Many-table deployments |

## Service Quotas — Common Service Codes

| Service | Code |
|---------|------|
| EC2 | `ec2` |
| Lambda | `lambda` |
| ECS | `ecs` |
| EKS | `eks` |
| S3 | `s3` |
| DynamoDB | `dynamodb` |
| RDS | `rds` |
| IAM | `iam` |
| CloudFormation | `cloudformation` |
| Route 53 | `route53` |
| ELB | `elasticloadbalancing` |
| API Gateway | `apigateway` |

```bash
# List adjustable quotas for any service
aws service-quotas list-service-quotas --service-code ec2 \
    --query "Quotas[?Adjustable==\`true\`].{Name:QuotaName,Value:Value,Code:QuotaCode}" \
    --output table
```

## Non-Increasable Limits

These limits cannot be raised via Service Quotas or support cases:

- IAM/STS rate limits (global, hard-coded)
- Route 53 API rate (5 req/sec)
- Some CloudFormation limits
- S3 per-prefix limits (but distributing across prefixes works around this)

## Additional Best Practices

### Use Pagination and Filters

```bash
# Bad — uses the smaller "unfiltered" bucket (50 tokens)
aws ec2 describe-instances

# Good — uses the larger "filtered" bucket (100 tokens)
aws ec2 describe-instances --filters "Name=tag:Environment,Values=prod"
```

### Consolidate API Calls

```bash
# Bad — 100 separate calls
for id in $instance_ids; do
    aws ec2 describe-instances --instance-ids "$id"
done

# Good — 1 call with multiple IDs
aws ec2 describe-instances --instance-ids $instance_ids
```

### Use EventBridge Instead of Polling

Replace periodic Describe calls with event-driven architecture:

- EC2 state changes → EventBridge rule → Lambda
- ECS task state changes → EventBridge rule → SNS
- S3 object created → EventBridge rule → Step Functions

### Stagger Automation

| Instead of | Do this |
|-----------|---------|
| All CI at :00 | Random offset per pipeline |
| Terraform plan every 5 min | Trigger on git push only |
| Cron every minute for monitoring | Use CloudWatch native metrics |
| Multiple tools calling same APIs | Consolidate into single source |

### Cache API Responses

```bash
# Terraform: skip refresh when possible
terraform plan -refresh=false

# Scripts: cache with TTL
aws ec2 describe-vpcs > /tmp/vpcs-cache.json
# Re-fetch only if older than 5 minutes
```

## AWS SDK Retry Configuration

### Python (boto3)

```python
import boto3
from botocore.config import Config

# Standard retry mode (default)
config = Config(
    retries={
        'max_attempts': 10,
        'mode': 'standard'  # exponential backoff
    }
)

# Adaptive retry mode (adjusts to throttling patterns)
config = Config(
    retries={
        'max_attempts': 10,
        'mode': 'adaptive'  # tracks throttle rate, slows preemptively
    }
)

client = boto3.client('ec2', config=config)
```

### Go (aws-sdk-go-v2)

```go
import (
    "github.com/aws/aws-sdk-go-v2/aws/retry"
    "github.com/aws/aws-sdk-go-v2/config"
)

cfg, _ := config.LoadDefaultConfig(context.TODO(),
    config.WithRetryer(func() aws.Retryer {
        return retry.AddWithMaxAttempts(retry.NewStandard(), 10)
    }),
)
```

### Node.js (aws-sdk v3)

```javascript
const { EC2Client } = require("@aws-sdk/client-ec2");

const client = new EC2Client({
    maxAttempts: 10,
    retryMode: "adaptive", // or "standard"
});
```

### Terraform AWS Provider

```hcl
provider "aws" {
  region = "us-east-1"

  # Default max_retries is 25
  max_retries = 10

  # Retry only throttling errors (not all errors)
  retry_mode = "adaptive"
}
```

### Retry Modes Explained

| Mode | Behavior |
|------|----------|
| `legacy` | Basic retry with fixed backoff (older SDKs) |
| `standard` | Exponential backoff with jitter, retries on throttling + transient errors |
| `adaptive` | Like standard, but tracks throttle rate and preemptively slows requests |

## Cross-Account Throttling Notes

- Each AWS account has its own independent token buckets per region
- Organizations SCPs do not affect rate limits — they only control permissions
- Cross-account role assumption (AssumeRole) consumes tokens from the calling account's STS bucket
- If Account A assumes a role in Account B and calls EC2, the EC2 rate limit consumed is Account B's (where the API call lands)
- Delegated administrator accounts share no rate limit buckets with the management account
- AWS Support limit increases are per-account — you must request separately for each account

```bash
# Scenario: Account A assumes role in Account B, calls EC2
# STS AssumeRole → consumes Account A's STS bucket
# EC2 DescribeInstances → consumes Account B's EC2 bucket

# Verify which account you're operating in
aws sts get-caller-identity
```

## API Endpoint Patterns

| Pattern | Example | Service |
|---------|---------|---------|
| `<service>.<region>.amazonaws.com` | `ec2.us-east-1.amazonaws.com` | Most services |
| `<service>.amazonaws.com` | `iam.amazonaws.com` | Global (IAM, Route53, CloudFront) |
| `s3.<region>.amazonaws.com` | `s3.us-east-1.amazonaws.com` | S3 (path-style) |
| `<bucket>.s3.<region>.amazonaws.com` | `my-bucket.s3.us-east-1.amazonaws.com` | S3 (virtual-hosted) |
| `<id>.execute-api.<region>.amazonaws.com` | | API Gateway |
| `<cluster>.eks.<region>.amazonaws.com` | | EKS API server |

## Quick Reference

| Scenario | Action |
|----------|--------|
| Terraform plan is slow | `terraform plan -parallelism=2` |
| Skip API calls entirely | `terraform plan -refresh=false` |
| Find who's throttled | CloudTrail → filter `RequestLimitExceeded` |
| Check your limits | Service Quotas → EC2 → search API name |
| Request increase | `aws service-quotas request-service-quota-increase` |
| Monitor ongoing | CloudWatch AWS/Usage → CallCount metric |
| Investigate caller | CloudTrail → filter by EventName → group by userAgent |
