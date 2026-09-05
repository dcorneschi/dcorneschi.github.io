# How to Use the AWS Free Tier Effectively

The AWS Free Tier is the cheapest way to learn AWS, prototype, and run small side
projects — but it's also the source of a lot of surprise bills. The rules changed
meaningfully in mid-2025, so the first thing to know is **which Free Tier you're on**, and
the second is how to stay inside it on purpose rather than by luck.

> **The single most important habit:** set up a billing alarm (or a zero-spend budget) on
> day one. The Free Tier does not hard-stop most spending on legacy accounts, and even on
> new credit-based accounts you want to see usage before it becomes a charge.

---

## First: which Free Tier are you on?

AWS restructured the Free Tier for accounts created **on or after July 15, 2025**. The
model is completely different from the older one, so identify your account's regime before
anything else.

| | Account created **before** July 15, 2025 (legacy) | Account created **on/after** July 15, 2025 (new) |
|---|---|---|
| **Model** | Usage-limit based | Credit + plan based |
| **Duration** | 12 months from sign-up (plus always-free offers) | 6 months, **or** until credits are exhausted, whichever comes first |
| **What you get** | Monthly free usage allowances per service | **Up to $100 sign-up credit + up to $100 more earned = $200** in credits |
| **Overage behavior** | Exceed a limit → billed pay-as-you-go | On the **Free plan you can't exceed limits**; on the **Paid plan** you pay normally |

The rest of this guide calls out where the two differ. If you're unsure, check the Free
Tier / billing widget on the AWS console home page — new accounts show a plan, credit
balance, and expiration date.

## The new (2025+) model: Free plan vs Paid plan

New accounts choose a plan at sign-up (and can switch later):

- **Free plan** — explore services for up to **6 months** with **no charges and no
  commitment**. You **cannot exceed** the Free Tier limits; when you'd go over, the action
  is blocked rather than billed. Great for pure learning with zero bill risk.
- **Paid plan** — normal pay-as-you-go, but your **credits are applied first**. Choose this
  when you need services or capacity beyond the free limits.

**Credits:** up to **$100 at sign-up**, plus up to **$100 more** earned by completing
guided activities (exploring specific services), for **$200 total**. Credits apply to
eligible services on either plan. Your Free Tier ends at **6 months or when credits run
out**, whichever is first — then you upgrade to the Paid plan to keep running.

### Earning the extra $100

The additional credits come from completing guided **activities** found in the **Explore
AWS** widget on your Console Home dashboard. There are five. Below is the quickest way to
finish each — do them in this order (Budgets first, since it's also your safety net).

> The activities are meant to be done through the **Explore AWS** widget in the console so
> they register as complete. The commands below are the fast, minimal equivalent of each
> task — handy if you prefer the CLI or want to understand what the guided flow does.

**1. Set up a cost budget with AWS Budgets** (free, do this first)

```bash
aws budgets create-budget \
  --account-id "$(aws sts get-caller-identity --query Account --output text)" \
  --budget '{"BudgetName":"MyFirstBudget","BudgetLimit":{"Amount":"5","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}'
```

**2. Launch (and terminate) an Amazon EC2 instance**

```bash
# Grab the current Amazon Linux 2023 AMI, launch a free-tier t3.micro
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text)
aws ec2 run-instances --image-id "$AMI" --instance-type t3.micro --count 1

# ...then clean up (use the InstanceId from the output above)
aws ec2 terminate-instances --instance-ids i-0123456789abcdef0
```

**3. Try a foundation model in the Amazon Bedrock playground**

Easiest in the console: **Bedrock → Playgrounds → Chat**, pick a model (request access if
prompted), type a prompt, run it. CLI equivalent once model access is granted:

```bash
aws bedrock-runtime invoke-model \
  --model-id anthropic.claude-3-haiku-20240307-v1:0 \
  --body '{"anthropic_version":"bedrock-2023-05-31","max_tokens":100,"messages":[{"role":"user","content":"Hello in one sentence"}]}' \
  --cli-binary-format raw-in-base64-out out.json && cat out.json
```

**4. Create a web app with AWS Lambda (function URL)**

```bash
# Minimal handler
echo 'def handler(e,c): return {"statusCode":200,"body":"Hello from Lambda"}' > lambda_function.py
zip fn.zip lambda_function.py

# Create the function (needs an execution role ARN) and give it a public URL
aws lambda create-function --function-name hello-web \
  --runtime python3.12 --handler lambda_function.handler \
  --role arn:aws:iam::<acct-id>:role/<lambda-basic-exec-role> \
  --zip-file fileb://fn.zip
aws lambda create-function-url-config --function-name hello-web --auth-type NONE
# curl the returned FunctionUrl to see "Hello from Lambda"
```

**5. Create an Amazon RDS database** (single-AZ, no backups — cheapest)

```bash
aws rds create-db-instance \
  --db-instance-identifier explore-db --db-instance-class db.t3.micro \
  --engine mysql --master-username admin --master-user-password 'ChangeMe123!' \
  --allocated-storage 20 --no-multi-az --backup-retention-period 0

# Clean up when the activity is done
aws rds delete-db-instance --db-instance-identifier explore-db --skip-final-snapshot
```

> Each activity except Budgets **spends a few cents of credit** while it runs the service —
> that's expected. **Clean up afterward** (terminate the EC2 instance, delete the Lambda
> function URL, delete the RDS database) so nothing keeps drawing down your credits.

> Practical implication: on the new model, treat the $200 as a budget to *spend
> deliberately* on learning, not a background allowance. Turn resources off when you're not
> using them so credits last the full 6 months.

## The legacy (pre-2025) model: three kinds of "free"

Older accounts (and the mental model most tutorials still use) have three distinct offer
types. Knowing which bucket a service falls into is what prevents surprise charges.

- **12-month free** — free for the first year from sign-up, up to a monthly allowance.
  Classic examples: **750 hours/month of a free-tier-eligible EC2 instance**, 5 GB of S3
  standard storage, 750 hours of RDS db.t-class. After 12 months (or over the limit), you
  pay standard rates.
- **Always free** — free indefinitely within limits, for everyone. Examples: **1M AWS
  Lambda requests/month**, DynamoDB 25 GB storage, CloudWatch basic metrics/alarms.
- **Trials** — short-term free trials that start when you activate a specific service
  (e.g., a number of days or a one-time allotment), then bill normally.

The trap: a tutorial says "this is free," but it's only free if (a) you're inside the
12-month window, (b) you stay under the monthly allowance, and (c) you picked the
free-tier-eligible size/type.

## Free-tier-eligible EC2: the size matters

The eligible instance types differ by account age — using the wrong size means you pay
from the first hour:

- **Legacy accounts:** `t2.micro`, `t3.micro`.
- **New accounts:** `t3.micro`, `t3.small`, `t4g.micro`, `t4g.small`, `c7i-flex.large`,
  `m7i-flex.large`.

Don't guess — ask the API which types and AMIs are actually free-tier-eligible for your
account:

```bash
# Free-tier-eligible instance types
aws ec2 describe-instance-types \
  --filters Name=free-tier-eligible,Values=true \
  --query "InstanceTypes[*].[InstanceType]" --output text | sort

# Free-tier-eligible AMIs
aws ec2 describe-images \
  --filters Name=free-tier-eligible,Values=true \
  --query "Images[*].[ImageId]" --output text | sort
```

EBS: `gp3`, `gp2`, `st1`, `sc1`, and `standard` volumes are free-tier-eligible — but only
up to the monthly storage allowance on legacy accounts. A big root volume or extra data
volumes will still cost you.

## Set up cost guardrails on day one

This is the part most people skip and later regret.

1. **Create a zero-spend or low-threshold budget** in AWS Budgets. A budget alerting at
   `$0.01` (or a few dollars) tells you the instant *anything* starts costing money.
2. **Enable Free Tier usage alerts** in Billing preferences so AWS emails you as you
   approach a limit (legacy accounts especially).
3. **Add a CloudWatch billing alarm** on the `EstimatedCharges` metric as a second line of
   defense.
4. **Turn on Cost Explorer** to see what's actually accruing, broken down by service.
5. **Use the console's Free Tier page / widget** to watch remaining credits (new) or
   usage vs. limits (legacy).

```bash
# Create a budget that alerts as soon as monthly cost crosses ~$0.01
aws budgets create-budget \
  --account-id "$(aws sts get-caller-identity --query Account --output text)" \
  --budget '{"BudgetName":"FreeTierMonitor","BudgetLimit":{"Amount":"1","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}' \
  --notifications-with-subscribers '[{"Notification":{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":0.01,"ThresholdType":"ABSOLUTE_VALUE"},"Subscribers":[{"SubscriptionType":"EMAIL","Address":"you@example.com"}]}]'

# See current Free Tier usage programmatically
aws freetier get-free-tier-usage

# Quick check of month-to-date spend from the CLI
aws ce get-cost-and-usage \
  --time-period Start=$(date -u +%Y-%m-01),End=$(date -u +%Y-%m-%d) \
  --granularity MONTHLY --metrics "UnblendedCost"
```

## Service-by-service traps (and how to avoid them)

The free-tier gotchas are service-specific. The biggest ones, with the fix:

- **EC2 — orphaned root volume.** Set `DeleteOnTermination: true` on the root EBS volume so
  it's cleaned up when you terminate the instance. Otherwise the volume lingers and keeps
  billing. Also remember **750 hours/month is shared across all instances** — two instances
  running 24×7 = 1,500 hours, so you pay for 750. The legacy EBS allowance is **30 GB
  total**, so a single 30 GB volume is fine but two 20 GB volumes (40 GB) bills the extra
  10 GB.

  ```bash
  # Launch a free-tier-eligible instance with a small, auto-deleting root volume
  aws ec2 run-instances \
    --image-id ami-0a1b2c3d4e5f67890 \
    --instance-type t3.micro \
    --key-name my-key \
    --security-group-ids sg-0a1b2c3d4e5f67890 \
    --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":8,"VolumeType":"gp3","DeleteOnTermination":true}}]' \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=free-tier-instance}]'
  ```

- **RDS — the Multi-AZ trap.** Enabling Multi-AZ adds a second DB instance, doubling usage
  and blowing the free allowance (or burning credits). Create with `--no-multi-az`, set
  `--backup-retention-period 0` so automated-backup storage doesn't accrue, and disable
  `--no-auto-minor-version-upgrade` for a stable dev database.

  ```bash
  # Free-tier-eligible single-AZ RDS instance (no Multi-AZ, no backups)
  aws rds create-db-instance \
    --db-instance-identifier free-tier-db \
    --db-instance-class db.t3.micro \
    --engine mysql \
    --master-username admin \
    --master-user-password 'your-secure-password' \
    --allocated-storage 20 \
    --no-multi-az \
    --backup-retention-period 0 \
    --no-auto-minor-version-upgrade \
    --storage-type gp2
  ```

- **Lambda — memory changes the math.** The always-free allowance is **1M requests/month**
  and **400,000 GB-seconds** of compute. At **128 MB** that's ~3.2M seconds of runtime
  (~37 days of continuous execution); bump the function to **1024 MB** and you get only
  ~400K seconds (~4.6 days, ~8× less) for the same allowance. Keep learning functions small.
- **DynamoDB — use provisioned mode.** The always-free 25 GB + 25 RCU/25 WCU (≈200M
  requests/month) applies to **provisioned** capacity. **On-demand mode does not qualify**
  for the free capacity units, so a table created on-demand can bill from the first request.

  ```bash
  # Provisioned-mode table that stays within free-tier capacity
  aws dynamodb create-table \
    --table-name MyApp \
    --attribute-definitions AttributeName=PK,AttributeType=S \
    --key-schema AttributeName=PK,KeyType=HASH \
    --billing-mode PROVISIONED \
    --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
  ```

- **S3 — request limits, not just storage.** The legacy allowance includes ~5 GB plus only
  **20,000 GET and 2,000 PUT** requests/month. A modestly trafficked static site can burn
  20,000 GETs in a day. Put **CloudFront in front** — its free allowance (≈1 TB transfer,
  ~10M requests/month) is far larger and it caches S3.
- **CloudWatch Logs — ingestion after the free allowance** is commonly ~$0.50/GB (US
  regions). Chatty Lambda/app logging adds up quietly; set log retention and log less in dev.

## Where the surprise bills actually come from

Most "the Free Tier charged me" stories trace to a handful of causes. Watch these:

- **NAT Gateway.** Not free, billed hourly **plus** per-GB. A NAT Gateway left running is
  one of the most common surprise charges. Use a NAT instance or VPC endpoints for
  learning, or delete it when done.
- **Elastic IPs.** An **unattached** (or idle) Elastic IP is billed. Release EIPs you're
  not using. (AWS now also charges for public IPv4 addresses generally.)
- **EBS volumes and snapshots left behind** after you terminate an instance — they keep
  costing money. Delete orphaned volumes/snapshots.
- **Load balancers (ALB/NLB).** Not free-tier; they bill hourly even with no traffic.
- **Data transfer out** to the internet and **cross-AZ** traffic. The allowance is small;
  large egress adds up.
- **Wrong region or wrong instance size.** Free-tier eligibility is size-specific; a
  `t3.medium` is never free.
- **Multiple instances / running 24×7.** The 750 hours/month legacy allowance covers
  *one* instance continuously — two instances burn it in ~15 days.
- **Public IPv4 addresses**, provisioned-but-idle resources (RDS, OpenSearch,
  ElastiCache), and forgotten test stacks.

## Practical habits to make it last

- **Stop or terminate when you're done.** Stopped EC2 instances don't bill for compute
  (but their EBS volumes still do). Terminate what you won't reuse.
- **Tag everything** (e.g. `project=learning`) so you can find and clean up resources, and
  filter them in Cost Explorer.
- **Prefer serverless for learning** — Lambda, DynamoDB, and S3 have generous always-free
  or long allowances and cost nothing when idle. No idle-capacity billing.
- **Use Infrastructure as Code** (CloudFormation/Terraform) so you can spin an environment
  up, experiment, and `destroy` it completely — no orphaned resources.
- **One region.** Resources scatter across regions easily; pick one and check the others
  occasionally for stragglers.
- **Do a monthly sweep:** unattached EIPs, orphaned EBS volumes/snapshots, idle load
  balancers and NAT gateways, stopped-but-not-terminated instances, and old RDS instances.
- **On the new model,** spend credits intentionally: run experiments in short bursts and
  shut down between sessions so the 6-month window and $200 aren't consumed by idle
  resources.

### A monthly-sweep script

Run these to catch the resources people forget about:

```bash
# Running instances you may have left on
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,Launched:LaunchTime}" \
  --output table

# Orphaned (unattached) EBS volumes
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "Volumes[].{ID:VolumeId,Size:Size,Type:VolumeType}" --output table

# Unattached Elastic IPs (billed even when idle) — then release them
aws ec2 describe-addresses \
  --query "Addresses[?AssociationId==null].{IP:PublicIp,AllocID:AllocationId}" --output table
# aws ec2 release-address --allocation-id <eipalloc-...>

# Running RDS instances
aws rds describe-db-instances \
  --query "DBInstances[].{ID:DBInstanceIdentifier,Class:DBInstanceClass,Status:DBInstanceStatus}" \
  --output table
```

## Quick reference

| Goal | Do this |
|------|---------|
| Know your regime | Check account creation date vs. July 15, 2025; view the console Free Tier widget |
| Avoid any bill (new accounts) | Choose the **Free plan** (can't exceed limits) |
| Catch spend instantly | AWS Budgets zero/low-threshold alert + CloudWatch billing alarm |
| Pick a free EC2 size | `aws ec2 describe-instance-types --filters Name=free-tier-eligible,Values=true` |
| Avoid classic traps | Delete idle NAT gateways, unattached EIPs, orphaned EBS, idle LBs |
| Learn cheaply | Prefer Lambda/DynamoDB/S3; use IaC and tear down after each session |

---

### Sources

- [AWS Free Tier](https://aws.amazon.com/free/)
- [Using the AWS Free Tier (Billing User Guide)](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/billing-free-tier.html)
- [Free Tier account plan comparison table (Free plan vs Paid plan)](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-plans.html)
- [Earning additional Free Tier credits](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier.html)
- [EC2 Free Tier: benefits before and after July 15, 2025](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html)
- [Creating a zero spend budget (AWS Budgets)](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-create.html)
