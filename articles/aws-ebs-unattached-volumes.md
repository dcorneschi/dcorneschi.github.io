# Unattached EBS Volumes: Detection and Monitoring

Unattached EBS volumes are volumes in the `available` state — they exist but are not attached to any instance. They still incur storage charges and are a common source of cloud waste.

## Why Unattached Volumes Exist

- Instance terminated with `DeleteOnTermination=false` on data volumes
- Volume manually detached and forgotten
- Failed automation that created volumes but didn't clean up
- Snapshots restored to volumes that were never attached
- Testing/debugging volumes left behind

## Detection

### Find All Unattached Volumes

```sh
# Simple list
aws ec2 describe-volumes --filters "Name=status,Values=available" --output table

# With cost-relevant details
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "Volumes[].{ID:VolumeId, Size:Size, Type:VolumeType, AZ:AvailabilityZone, Created:CreateTime}" \
  --output table

# Total wasted storage (GiB)
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "sum(Volumes[].Size)" --output text

# Count by volume type
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "Volumes[].VolumeType" --output text | tr '\t' '\n' | sort | uniq -c

# Find unattached volumes with no Name tag (likely orphaned)
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "Volumes[?!Tags || !contains(Tags[].Key, 'Name')].{ID:VolumeId, Size:Size, Created:CreateTime}" \
  --output table
```

### Find Volumes Unattached for a Long Time

```sh
# Volumes with no recent attachment (check detach time via CloudTrail or creation date)
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "Volumes[?CreateTime<='$(date -u -d '30 days ago' '+%Y-%m-%dT%H:%M:%S')'].{ID:VolumeId, Size:Size, Created:CreateTime, Type:VolumeType}" \
  --output table
```

### Find Unattached Volumes Across All Regions

```sh
#!/bin/bash
for REGION in $(aws ec2 describe-regions --query "Regions[].RegionName" --output text); do
  COUNT=$(aws ec2 describe-volumes --region $REGION --filters "Name=status,Values=available" \
    --query "length(Volumes)" --output text)
  if [ "$COUNT" -gt 0 ]; then
    SIZE=$(aws ec2 describe-volumes --region $REGION --filters "Name=status,Values=available" \
      --query "sum(Volumes[].Size)" --output text)
    echo "$REGION: $COUNT volumes, ${SIZE} GiB"
  fi
done
```

### Find Unencrypted Unattached Volumes

```sh
aws ec2 describe-volumes \
  --filters "Name=status,Values=available" "Name=encrypted,Values=false" \
  --query "Volumes[].{ID:VolumeId, Size:Size, AZ:AvailabilityZone}" \
  --output table
```

## Cost Estimation

### Calculate Monthly Cost

EBS pricing varies by type and region. Approximate monthly costs (us-east-1):

| Type | $/GiB/month |
|------|-------------|
| gp3 | $0.08 |
| gp2 | $0.10 |
| io2 | $0.125 + $0.065/provisioned IOPS |
| io1 | $0.125 + $0.065/provisioned IOPS |
| st1 | $0.045 |
| sc1 | $0.015 |

```sh
# Estimate wasted cost (gp3 volumes only, simplified)
aws ec2 describe-volumes --filters "Name=status,Values=available" "Name=volume-type,Values=gp3" \
  --query "sum(Volumes[].Size)" --output text | awk '{printf "Estimated waste: $%.2f/month\n", $1 * 0.08}'

# Detailed cost breakdown by type
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "Volumes[].{Type:VolumeType, Size:Size}" --output json | \
  jq 'group_by(.Type) | map({
    type: .[0].Type,
    count: length,
    total_gib: (map(.Size) | add),
    est_monthly: (
      if .[0].Type == "gp3" then (map(.Size) | add) * 0.08
      elif .[0].Type == "gp2" then (map(.Size) | add) * 0.10
      elif .[0].Type == "io1" or .[0].Type == "io2" then (map(.Size) | add) * 0.125
      elif .[0].Type == "st1" then (map(.Size) | add) * 0.045
      elif .[0].Type == "sc1" then (map(.Size) | add) * 0.015
      else 0 end
    )
  })'
```

## Safe Cleanup Process

Never delete volumes blindly. Follow this workflow:

### Step 1: Identify Candidates

```sh
# List unattached volumes older than 30 days
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "Volumes[?CreateTime<='$(date -u -d '30 days ago' '+%Y-%m-%dT%H:%M:%S')'].{ID:VolumeId, Size:Size, Type:VolumeType, Tags:Tags[?Key=='Name'].Value|[0], Created:CreateTime}" \
  --output table
```

### Step 2: Snapshot Before Deletion (Safety Net)

```sh
# Snapshot each candidate before deleting
for VOL in $(aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "Volumes[?CreateTime<='$(date -u -d '30 days ago' '+%Y-%m-%dT%H:%M:%S')'].VolumeId" --output text); do
  echo "Snapshotting $VOL..."
  aws ec2 create-snapshot --volume-id $VOL --description "Pre-delete backup $(date +%F)" \
    --tag-specifications "ResourceType=snapshot,Tags=[{Key=Name,Value=pre-delete-$VOL},{Key=DeleteAfter,Value=$(date -u -d '90 days' '+%Y-%m-%d')}]"
done
```

### Step 3: Delete the Volumes

```sh
# Delete after confirming snapshots completed
for VOL in $(aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "Volumes[?CreateTime<='$(date -u -d '30 days ago' '+%Y-%m-%dT%H:%M:%S')'].VolumeId" --output text); do
  echo "Deleting $VOL..."
  aws ec2 delete-volume --volume-id $VOL
done
```

### Step 4: Clean Up Old Safety Snapshots

```sh
# Delete safety snapshots after 90 days
aws ec2 describe-snapshots --owner-ids self \
  --filters "Name=tag:DeleteAfter,Values=$(date +%Y-%m-%d)" \
  --query "Snapshots[].SnapshotId" --output text | \
  xargs -n1 aws ec2 delete-snapshot --snapshot-id
```

## Monitoring with CloudWatch

### CloudWatch Metric: Volume Idle Time

Attached volumes with zero I/O may also be wasted:

```sh
# Check if an attached volume has had any I/O in the last 7 days
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeReadOps \
  --dimensions Name=VolumeId,Value=vol-0abc123 \
  --start-time $(date -u -d '7 days ago' '+%Y-%m-%dT%H:%M:%SZ') \
  --end-time $(date -u '+%Y-%m-%dT%H:%M:%SZ') \
  --period 604800 \
  --statistics Sum \
  --query "Datapoints[0].Sum" --output text
```

If the sum of `VolumeReadOps` + `VolumeWriteOps` is 0 over 7 days, the volume is idle.

### Find Idle Attached Volumes

```sh
#!/bin/bash
# Check all attached volumes for zero I/O in the last 7 days
for VOL in $(aws ec2 describe-volumes --filters "Name=status,Values=in-use" \
  --query "Volumes[].VolumeId" --output text); do

  READS=$(aws cloudwatch get-metric-statistics \
    --namespace AWS/EBS \
    --metric-name VolumeReadOps \
    --dimensions Name=VolumeId,Value=$VOL \
    --start-time $(date -u -d '7 days ago' '+%Y-%m-%dT%H:%M:%SZ') \
    --end-time $(date -u '+%Y-%m-%dT%H:%M:%SZ') \
    --period 604800 --statistics Sum \
    --query "Datapoints[0].Sum" --output text 2>/dev/null)

  WRITES=$(aws cloudwatch get-metric-statistics \
    --namespace AWS/EBS \
    --metric-name VolumeWriteOps \
    --dimensions Name=VolumeId,Value=$VOL \
    --start-time $(date -u -d '7 days ago' '+%Y-%m-%dT%H:%M:%SZ') \
    --end-time $(date -u '+%Y-%m-%dT%H:%M:%SZ') \
    --period 604800 --statistics Sum \
    --query "Datapoints[0].Sum" --output text 2>/dev/null)

  if [ "${READS:-0}" = "0" ] && [ "${WRITES:-0}" = "0" ]; then
    SIZE=$(aws ec2 describe-volumes --volume-ids $VOL --query "Volumes[0].Size" --output text)
    echo "IDLE: $VOL (${SIZE} GiB)"
  fi
done
```

## Automated Monitoring with AWS Config

AWS Config can detect unattached volumes automatically:

```sh
# Use the managed rule: ec2-volume-inuse-check
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "ec2-volume-inuse-check",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "EC2_VOLUME_INUSE_CHECK"
  },
  "Scope": {
    "ComplianceResourceTypes": ["AWS::EC2::Volume"]
  }
}'

# Check compliance results
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name ec2-volume-inuse-check \
  --compliance-types NON_COMPLIANT \
  --query "EvaluationResults[].{Resource:EvaluationResultIdentifier.EvaluationResultQualifier.ResourceId}" \
  --output table
```

## Automated Cleanup with Lambda

Schedule a Lambda to report or clean up unattached volumes:

```python
import boto3
from datetime import datetime, timezone, timedelta

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    
    # Find unattached volumes older than 30 days
    response = ec2.describe_volumes(Filters=[{'Name': 'status', 'Values': ['available']}])
    
    threshold = datetime.now(timezone.utc) - timedelta(days=30)
    orphaned = []
    
    for vol in response['Volumes']:
        if vol['CreateTime'] < threshold:
            orphaned.append({
                'VolumeId': vol['VolumeId'],
                'Size': vol['Size'],
                'Type': vol['VolumeType'],
                'Created': vol['CreateTime'].isoformat()
            })
    
    # Log findings (or send to SNS, Slack, etc.)
    print(f"Found {len(orphaned)} unattached volumes older than 30 days")
    for vol in orphaned:
        print(f"  {vol['VolumeId']}: {vol['Size']} GiB ({vol['Type']}), created {vol['Created']}")
    
    # Optional: auto-snapshot and delete
    # for vol in orphaned:
    #     ec2.create_snapshot(VolumeId=vol['VolumeId'], Description='Auto pre-delete')
    #     ec2.delete_volume(VolumeId=vol['VolumeId'])
    
    return {'orphaned_count': len(orphaned), 'volumes': orphaned}
```

Schedule with EventBridge (weekly):

```sh
aws events put-rule \
  --name "check-orphaned-ebs" \
  --schedule-expression "rate(7 days)"

aws lambda add-permission \
  --function-name check-orphaned-ebs \
  --statement-id eventbridge-invoke \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:us-east-1:123456789012:rule/check-orphaned-ebs

aws events put-targets \
  --rule check-orphaned-ebs \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-east-1:123456789012:function:check-orphaned-ebs"
```

## Prevention

### Set DeleteOnTermination for Data Volumes

```sh
# When launching an instance
aws ec2 run-instances ... \
  --block-device-mappings '[{"DeviceName":"/dev/xvdf","Ebs":{"VolumeSize":100,"DeleteOnTermination":true}}]'

# Modify an existing attachment
aws ec2 modify-instance-attribute --instance-id i-0abc123 \
  --block-device-mappings '[{"DeviceName":"/dev/xvdf","Ebs":{"DeleteOnTermination":true}}]'
```

### Terraform: Ensure DeleteOnTermination

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abc123"
  instance_type = "t3.medium"

  root_block_device {
    volume_size           = 20
    delete_on_termination = true
  }

  ebs_block_device {
    device_name           = "/dev/xvdf"
    volume_size           = 100
    volume_type           = "gp3"
    delete_on_termination = true  # Prevents orphaned volumes
  }
}
```

### Tag Volumes for Ownership

```sh
# Tag at creation
aws ec2 create-volume --volume-type gp3 --size 50 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Owner,Value=team-platform},{Key=Purpose,Value=app-data},{Key=CreatedBy,Value=automation}]'
```

Volumes without `Owner` or `Purpose` tags are likely orphaned.

## Gotchas

- **Snapshots don't cost the same as volumes**: A snapshot is incremental and compressed — much cheaper than keeping the volume running. Snapshot before deleting.
- **DeleteOnTermination default is false for additional volumes**: Only root volumes default to `true`. Every data volume you attach manually stays behind on termination unless you explicitly set this.
- **Volumes in "available" state still cost money**: There's no "stopped" state for EBS. If it exists, you pay.
- **You can't attach a volume cross-AZ**: An unattached volume in `us-east-1a` can only be attached to instances in `us-east-1a`. To move it, snapshot → restore in the target AZ.
- **AWS Config charges per rule evaluation**: The `ec2-volume-inuse-check` rule is useful but adds to your Config bill if you have many volumes.


## Multi-Account / Multi-Region Detection

### Quick One-Liner

Scan across multiple AWS profiles and regions in one pass:

```sh
for profile in dev uat mgmt secure prod pci; do
  for region in us-east-1 ap-southeast-1 us-west-2 us-west-1 eu-west-2 eu-central-1; do
    echo "=== $profile / $region ===" && \
    aws ec2 describe-volumes --profile "$profile" --region "$region" \
      --filters Name=status,Values=available \
      --query 'Volumes[].{Name:Tags[?Key==`Name`]|[0].Value,ID:VolumeId,Size:Size,Type:VolumeType,AZ:AvailabilityZone,Created:CreateTime}' \
      --output table
  done
done
```

### CSV Report Script

Generate a timestamped CSV report across all accounts and regions:

```sh
#!/bin/bash
# list-unattached-ebs.sh — Generates a CSV of unattached EBS volumes across accounts/regions

PROFILES="dev uat mgmt secure prod pci"
REGIONS="us-east-1 ap-southeast-1 us-west-2 us-west-1 eu-west-2 eu-central-1"
OUTPUT="unattached-ebs-$(date +%Y%m%d-%H%M%S).csv"

echo "Account,Region,Name,VolumeId,Size(GiB),Type,AZ,Created" > "$OUTPUT"

for profile in $PROFILES; do
  for region in $REGIONS; do
    aws ec2 describe-volumes --profile "$profile" --region "$region" \
      --filters Name=status,Values=available \
      --query 'Volumes[].{Name:Tags[?Key==`Name`]|[0].Value,ID:VolumeId,Size:Size,Type:VolumeType,AZ:AvailabilityZone,Created:CreateTime}' \
      --output json 2>/dev/null | \
    jq -r --arg acct "$profile" --arg reg "$region" \
      '.[] | [$acct, $reg, (.Name // "untagged"), .ID, (.Size|tostring), .Type, .AZ, .Created] | @csv' >> "$OUTPUT"
  done
done

echo "Report: $OUTPUT"
wc -l "$OUTPUT"
```

Usage:

```sh
chmod +x list-unattached-ebs.sh
./list-unattached-ebs.sh
```

Output columns: Account, Region, Name, VolumeId, Size(GiB), Type, AZ, Created.

### Prerequisites

- AWS CLI v2 configured with named profiles (`~/.aws/config`)
- `jq` installed
- IAM permissions: `ec2:DescribeVolumes`, `ec2:DescribeRegions`

## Datadog Monitoring

Datadog doesn't have a native "unattached EBS" metric, but you can detect them using zero I/O activity.

### Dashboard Formula

In the graph editor:

1. Query **a**: `sum:aws.ebs.volume_read_ops{*} by {volumeid}`
2. Query **b**: `sum:aws.ebs.volume_write_ops{*} by {volumeid}`
3. Formula: `a + b`

Use `top(a + b, 10, 'last', 'asc')` to surface the lowest-activity volumes first.

### Monitor: Alert on Zero I/O for 24h

```json
{
  "name": "EBS Volume - Zero I/O (Likely Unattached)",
  "type": "query alert",
  "query": "sum(last_1d):sum:aws.ebs.volume_read_ops{*} by {volumeid} + sum:aws.ebs.volume_write_ops{*} by {volumeid} <= 0",
  "message": "EBS volume {{volumeid.name}} has had zero I/O ops over the last 24h.\nThis volume is likely unattached or unused.\nReview and delete if no longer needed.\n\n@slack-your-channel @team-infra",
  "options": {
    "thresholds": { "critical": 0 },
    "notify_no_data": false,
    "renotify_interval": 1440,
    "evaluation_delay": 900,
    "include_tags": true
  },
  "tags": ["team:infra", "cost:optimization", "service:ebs"],
  "priority": 3
}
```

Key settings:

| Setting | Value | Reason |
|---------|-------|--------|
| `renotify_interval` | 1440 min (24h) | Re-alerts daily, not spammy |
| `evaluation_delay` | 900 sec (15 min) | Buffer for CloudWatch metric lag |
| `notify_no_data` | false | No false alerts if volume gets deleted |
| `priority` | P3 | Cost optimization, not an outage |

### Tips for Datadog EBS Monitoring

- Add `account` and `region` to the `by {}` grouping for per-account breakdown
- Pipe the CSV script output to Datadog as a custom metric for a single pane of glass
- Schedule the script via cron for weekly reports
- Datadog AWS integration must have EBS metrics enabled
