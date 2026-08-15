# AWS EBS Cheatsheet

## EBS Volume Types

| Type | Name | Use Case | Max IOPS | Max Throughput | Size | Durability | Multi-Attach |
|------|------|----------|----------|----------------|------|------------|--------------|
| `gp3` | General Purpose SSD | Most workloads (default) | 16,000 | 1,000 MB/s | 1 GiB–16 TiB | 99.8–99.9% | No |
| `gp2` | General Purpose SSD | Legacy default | 16,000 (burst) | 250 MB/s | 1 GiB–16 TiB | 99.8–99.9% | No |
| `io2` | Provisioned IOPS SSD | Databases, latency-sensitive | 64,000 | 1,000 MB/s | 4 GiB–16 TiB | 99.999% | Yes |
| `io2 Block Express` | Provisioned IOPS SSD | Largest databases | 256,000 | 4,000 MB/s | 4 GiB–64 TiB | 99.999% | Yes |
| `io1` | Provisioned IOPS SSD | Legacy provisioned IOPS | 64,000 | 1,000 MB/s | 4 GiB–16 TiB | 99.8–99.9% | Yes |
| `st1` | Throughput Optimized HDD | Big data, log processing | 500 | 500 MB/s | 125 GiB–16 TiB | 99.8–99.9% | No |
| `sc1` | Cold HDD | Infrequent access | 250 | 250 MB/s | 125 GiB–16 TiB | 99.8–99.9% | No |

> `gp3` is the recommended default. It's cheaper than `gp2` and allows you to configure IOPS and throughput independently.

## Volume Management

### Create a Volume

```sh
# Create a gp3 volume
aws ec2 create-volume \
  --volume-type gp3 \
  --size 100 \
  --availability-zone us-east-1a \
  --iops 3000 \
  --throughput 125 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=data-vol}]'

# Create an io2 volume
aws ec2 create-volume \
  --volume-type io2 \
  --size 500 \
  --iops 10000 \
  --availability-zone us-east-1a

# Create from a snapshot
aws ec2 create-volume \
  --snapshot-id snap-0abc123 \
  --volume-type gp3 \
  --availability-zone us-east-1a
```

### List Volumes

```sh
# List all volumes
aws ec2 describe-volumes --output table

# List with key details
aws ec2 describe-volumes \
  --query "Volumes[].{ID:VolumeId, Size:Size, Type:VolumeType, State:State, AZ:AvailabilityZone, Attached:Attachments[0].InstanceId}" \
  --output table

# Find unattached volumes
aws ec2 describe-volumes \
  --filters "Name=status,Values=available" \
  --query "Volumes[].{ID:VolumeId, Size:Size, Type:VolumeType}" \
  --output table

# Find volumes by tag
aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=data-vol" \
  --output table

# Find volumes attached to a specific instance
aws ec2 describe-volumes \
  --filters "Name=attachment.instance-id,Values=i-0abc123" \
  --query "Volumes[].{ID:VolumeId, Device:Attachments[0].Device, Size:Size}" \
  --output table
```

### Attach a Volume

```sh
# Attach to an instance
aws ec2 attach-volume \
  --volume-id vol-0abc123 \
  --instance-id i-0abc123 \
  --device /dev/xvdf

# Wait for attachment
aws ec2 wait volume-in-use --volume-ids vol-0abc123
```

### Detach a Volume

```sh
# Graceful detach
aws ec2 detach-volume --volume-id vol-0abc123

# Force detach (if instance is unresponsive)
aws ec2 detach-volume --volume-id vol-0abc123 --force

# Wait for detachment
aws ec2 wait volume-available --volume-ids vol-0abc123
```

### Delete a Volume

```sh
# Delete a single volume
aws ec2 delete-volume --volume-id vol-0abc123

# Delete all unattached volumes (careful!)
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "Volumes[].VolumeId" --output text | \
  xargs -n1 aws ec2 delete-volume --volume-id
```

## Modify a Volume (Online Resize)

EBS volumes can be resized, change type, or adjust IOPS/throughput without downtime:

```sh
# Increase size
aws ec2 modify-volume --volume-id vol-0abc123 --size 200

# Change type (gp2 → gp3)
aws ec2 modify-volume --volume-id vol-0abc123 --volume-type gp3

# Adjust IOPS and throughput (gp3)
aws ec2 modify-volume --volume-id vol-0abc123 \
  --volume-type gp3 \
  --iops 6000 \
  --throughput 500

# Check modification progress
aws ec2 describe-volumes-modifications --volume-ids vol-0abc123 \
  --query "VolumesModifications[0].{State:ModificationState, Progress:Progress, Original:OriginalSize, Target:TargetSize}" \
  --output table
```

After resizing, grow the filesystem inside the instance:

```sh
# XFS
sudo xfs_growfs /mount-point

# ext4
sudo resize2fs /dev/xvdf

# If partition-based, grow the partition first
sudo growpart /dev/xvdf 1
sudo resize2fs /dev/xvdf1
```

> You can only modify a volume once every 6 hours. If modification state shows `optimizing`, the volume is usable but still completing in the background.

## Snapshots

### Create a Snapshot

```sh
# Snapshot a volume
aws ec2 create-snapshot \
  --volume-id vol-0abc123 \
  --description "Backup 2024-01-15" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=daily-backup}]'

# Snapshot without stopping instance (crash-consistent)
# Best practice: flush filesystem first
ssh ec2-user@instance "sudo sync && sudo fsfreeze -f /data"
aws ec2 create-snapshot --volume-id vol-0abc123 --description "Consistent backup"
ssh ec2-user@instance "sudo fsfreeze -u /data"
```

### List Snapshots

```sh
# List your snapshots
aws ec2 describe-snapshots --owner-ids self \
  --query "Snapshots[].{ID:SnapshotId, Volume:VolumeId, Size:VolumeSize, State:State, Date:StartTime}" \
  --output table

# Find snapshots for a specific volume
aws ec2 describe-snapshots --owner-ids self \
  --filters "Name=volume-id,Values=vol-0abc123" \
  --output table

# Find snapshots older than 30 days
aws ec2 describe-snapshots --owner-ids self \
  --query "Snapshots[?StartTime<='$(date -u -d '30 days ago' '+%Y-%m-%dT%H:%M:%S')'].{ID:SnapshotId, Date:StartTime, Size:VolumeSize}" \
  --output table
```

### Copy a Snapshot (Cross-Region)

```sh
# Copy to another region (run from the destination region)
aws ec2 copy-snapshot \
  --region eu-west-1 \
  --source-region us-east-1 \
  --source-snapshot-id snap-0abc123 \
  --description "DR copy"
```

### Delete Snapshots

```sh
# Delete a snapshot
aws ec2 delete-snapshot --snapshot-id snap-0abc123

# Delete all snapshots for a volume
aws ec2 describe-snapshots --owner-ids self \
  --filters "Name=volume-id,Values=vol-0abc123" \
  --query "Snapshots[].SnapshotId" --output text | \
  xargs -n1 aws ec2 delete-snapshot --snapshot-id
```

### Snapshot Lifecycle with DLM

```sh
# Create a lifecycle policy (daily snapshots, retain 7)
aws dlm create-lifecycle-policy \
  --description "Daily EBS snapshots" \
  --state ENABLED \
  --execution-role-arn arn:aws:iam::123456789012:role/AWSDataLifecycleManagerDefaultRole \
  --policy-details '{
    "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
    "ResourceTypes": ["VOLUME"],
    "TargetTags": [{"Key": "Backup", "Value": "true"}],
    "Schedules": [{
      "Name": "DailySnapshot",
      "CreateRule": {"Interval": 24, "IntervalUnit": "HOURS", "Times": ["03:00"]},
      "RetainRule": {"Count": 7},
      "CopyTags": true
    }]
  }'

# List lifecycle policies
aws dlm get-lifecycle-policies --output table
```

### Snapshot Lock

Lock snapshots to prevent deletion for compliance and data retention. No extra charge.

| Mode | Behavior |
|------|----------|
| Governance | Locked against deletion by all users. IAM users with proper permissions can extend/shorten duration, delete lock, or change to Compliance mode. |
| Compliance | Locked against deletion by root and all IAM users. After a cooling-off period (up to 72 hours), the lock cannot be removed until it expires. Duration can be extended but not shortened. |

```bash
# Lock a snapshot in governance mode (1 year)
aws ec2 lock-snapshot --snapshot-id snap-0abc123 \
  --lock-mode governance --lock-duration 365

# Lock a snapshot in compliance mode (5 years, 24h cooling-off)
aws ec2 lock-snapshot --snapshot-id snap-0abc123 \
  --lock-mode compliance --lock-duration 1825 \
  --cool-off-period 24

# Unlock a governance-mode snapshot
aws ec2 unlock-snapshot --snapshot-id snap-0abc123

# Describe locked snapshots
aws ec2 describe-locked-snapshots

# Describe lock status for a specific snapshot
aws ec2 describe-locked-snapshots --snapshot-ids snap-0abc123
```

Notes:
- Locked snapshots can still be shared, copied, or archived
- If using customer-managed KMS keys, ensure the key remains valid for the lock duration
- AWS Backup independently manages retention — locking Backup-created snapshots is not recommended

### Multi-Region Snapshot and Volume Queries

```bash
# Find snapshots older than 1 month across all regions
for REGION in $(aws ec2 describe-regions --output text --query 'Regions[].[RegionName]'); do
  echo "$REGION"
  aws ec2 describe-snapshots --owner-ids self --region $REGION \
    --query "Snapshots[?(StartTime<='$(date --date='-1 month' '+%Y-%m-%d')')].{ID:SnapshotId,Time:StartTime,Details:Description}" \
    --output table
done

# Find publicly shared snapshots across all regions
for REGION in $(aws ec2 describe-regions --output text --query 'Regions[].[RegionName]'); do
  echo "$REGION:"
  for snap in $(aws ec2 describe-snapshots --owner self --output text --region $REGION --query 'Snapshots[*].SnapshotId'); do
    aws ec2 describe-snapshot-attribute --snapshot-id $snap --region $REGION \
      --output text --attribute createVolumePermission \
      --query '[SnapshotId,CreateVolumePermissions[?Group == `all`]]'
  done
done

# Find unattached volumes across all regions
for REGION in $(aws ec2 describe-regions --output text --query 'Regions[].[RegionName]'); do
  echo "$REGION"
  aws ec2 describe-volumes --filter "Name=status,Values=available" --region $REGION \
    --query 'Volumes[*].{VolumeID:VolumeId,Size:Size,Type:VolumeType,AZ:AvailabilityZone}' \
    --output table
done

# Find volumes in error state across all regions
for REGION in $(aws ec2 describe-regions --output text --query 'Regions[].[RegionName]'); do
  echo "$REGION"
  aws ec2 describe-volumes --filter "Name=status,Values=error" --region $REGION \
    --query 'Volumes[*].{VolumeID:VolumeId,Size:Size,Type:VolumeType,AZ:AvailabilityZone}' \
    --output table
done

# Find volumes currently being modified (optimizing) across all regions
for REGION in $(aws ec2 describe-regions --output text --query 'Regions[].[RegionName]'); do
  echo "$REGION"
  aws ec2 describe-volumes-modifications --region $REGION \
    --filter 'Name=modification-state,Values=optimizing' \
    --query 'VolumesModifications[].{VolumeID:VolumeId,TargetSize:TargetSize,OriginalSize:OriginalSize,Progress:Progress}' \
    --output table
done
```

## Encryption

```sh
# Create an encrypted volume
aws ec2 create-volume \
  --volume-type gp3 \
  --size 100 \
  --availability-zone us-east-1a \
  --encrypted

# Create with a custom KMS key
aws ec2 create-volume \
  --volume-type gp3 \
  --size 100 \
  --availability-zone us-east-1a \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abcd-1234

# Enable encryption by default for the account
aws ec2 enable-ebs-encryption-by-default

# Check if encryption by default is enabled
aws ec2 get-ebs-encryption-by-default
```

> You cannot encrypt an existing unencrypted volume in-place. Create an encrypted snapshot copy, then create a new volume from it.

### Encrypt an Existing Volume

```sh
# 1. Snapshot the unencrypted volume
aws ec2 create-snapshot --volume-id vol-0abc123 --description "Pre-encryption"

# 2. Copy the snapshot with encryption
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-0abc123 \
  --encrypted \
  --description "Encrypted copy"

# 3. Create a new volume from the encrypted snapshot
aws ec2 create-volume \
  --snapshot-id snap-encrypted123 \
  --volume-type gp3 \
  --availability-zone us-east-1a

# 4. Swap volumes (stop instance, detach old, attach new)
```

## Multi-Attach (io1/io2 Only)

Share a volume across up to 16 Nitro instances in the same AZ:

```sh
# Create a multi-attach volume
aws ec2 create-volume \
  --volume-type io2 \
  --size 100 \
  --iops 10000 \
  --availability-zone us-east-1a \
  --multi-attach-enabled

# Attach to multiple instances
aws ec2 attach-volume --volume-id vol-0abc123 --instance-id i-instance1 --device /dev/xvdf
aws ec2 attach-volume --volume-id vol-0abc123 --instance-id i-instance2 --device /dev/xvdf
```

> Multi-Attach requires a cluster-aware filesystem (GFS2, OCFS2) or application-level locking. ext4/XFS will corrupt data.

## Tagging

```sh
# Tag a volume
aws ec2 create-tags --resources vol-0abc123 \
  --tags Key=Name,Value=data-vol Key=env,Value=prod Key=Backup,Value=true

# Tag a snapshot
aws ec2 create-tags --resources snap-0abc123 \
  --tags Key=Name,Value=daily-backup

# Find volumes missing a tag
aws ec2 describe-volumes \
  --query "Volumes[?!Tags || !contains(Tags[].Key, 'Backup')].{ID:VolumeId, Size:Size}" \
  --output table
```

## Performance

### gp3 Baseline vs Provisioned

| Parameter | gp3 Baseline (free) | gp3 Max (extra cost) |
|-----------|---------------------|----------------------|
| IOPS | 3,000 | 16,000 |
| Throughput | 125 MB/s | 1,000 MB/s |

IOPS and throughput are independent of volume size on gp3.

### gp2 Burst Behavior

| Volume Size | Baseline IOPS | Burst IOPS | Burst Duration |
|-------------|---------------|------------|----------------|
| 1–33 GiB | 100 | 3,000 | 30 min (at 1 GiB) |
| 100 GiB | 300 | 3,000 | ~60 min |
| 1,000 GiB | 3,000 | N/A (baseline = burst) | N/A |
| 5,334+ GiB | 16,000 | N/A (max) | N/A |

gp2 formula: 3 IOPS per GiB (minimum 100, max 16,000).

### Monitor Volume Performance

```sh
# Check volume metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeReadOps \
  --dimensions Name=VolumeId,Value=vol-0abc123 \
  --start-time $(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ') \
  --end-time $(date -u '+%Y-%m-%dT%H:%M:%SZ') \
  --period 300 \
  --statistics Sum

# Key metrics to monitor
# VolumeReadOps / VolumeWriteOps     — IOPS consumed
# VolumeReadBytes / VolumeWriteBytes — Throughput consumed
# VolumeQueueLength                  — Outstanding I/O (>1 = saturated)
# VolumeThroughputPercentage         — % of provisioned throughput used (io1/io2)
# VolumeConsumedReadWriteOps         — % of provisioned IOPS used (io1/io2)
# BurstBalance                       — gp2 burst credits remaining (%)
```

### Check Burst Balance (gp2)

```sh
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name BurstBalance \
  --dimensions Name=VolumeId,Value=vol-0abc123 \
  --start-time $(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ') \
  --end-time $(date -u '+%Y-%m-%dT%H:%M:%SZ') \
  --period 300 \
  --statistics Average
```

If burst balance drops to 0%, you're throttled to baseline IOPS. Migrate to gp3 or increase volume size.

## One-Liners

```sh
# Total EBS storage used (in GiB)
aws ec2 describe-volumes --query "sum(Volumes[].Size)" --output text

# Count volumes by type
aws ec2 describe-volumes --query "Volumes[].VolumeType" --output text | tr '\t' '\n' | sort | uniq -c

# Find largest volumes
aws ec2 describe-volumes \
  --query "sort_by(Volumes, &Size)[-5:].{ID:VolumeId, Size:Size, Type:VolumeType}" --output table

# Find volumes not tagged for backup
aws ec2 describe-volumes \
  --query "Volumes[?!contains(Tags[].Key || [''], 'Backup')].{ID:VolumeId, Size:Size, Instance:Attachments[0].InstanceId}" \
  --output table

# Cost estimate: find all unattached volumes and their sizes
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query "Volumes[].{ID:VolumeId, SizeGiB:Size, Type:VolumeType, Created:CreateTime}" \
  --output table

# Find all unencrypted volumes
aws ec2 describe-volumes --filters Name=encrypted,Values=false \
  | jq '.Volumes[] | .VolumeId, .Encrypted'

# Snapshot all volumes tagged for backup
aws ec2 describe-volumes --filters "Name=tag:Backup,Values=true" \
  --query "Volumes[].VolumeId" --output text | \
  xargs -n1 -I {} aws ec2 create-snapshot --volume-id {} --description "Automated backup $(date +%F)"

# List all instances with their volume IDs and names
aws ec2 describe-instances \
  --query "Reservations[*].Instances[*].[Tags[?Key=='Name'].Value|[0],InstanceId,BlockDeviceMappings[*].Ebs.VolumeId]" \
  --output text

# List volumes attached to the current instance (from inside the instance)
aws ec2 describe-volumes \
  --filters Name=attachment.instance-id,Values=$(curl -s http://169.254.169.254/latest/meta-data/instance-id) \
  --output table
```

## Self-Provisioning a Volume from Inside an Instance

Create and attach a volume using instance metadata — useful in userdata scripts:

```sh
# Get instance identity from metadata
REGION=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)

# Create a volume in the same AZ
VOLUME_ID=$(aws ec2 create-volume \
  --volume-type gp3 \
  --size 50 \
  --region $REGION \
  --availability-zone $AZ \
  --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=data-$INSTANCE_ID}]" \
  | jq -r .VolumeId)

# Wait for volume to be available
aws ec2 wait volume-available --volume-ids $VOLUME_ID --region $REGION

# Attach to this instance
aws ec2 attach-volume \
  --device /dev/xvdf \
  --volume-id $VOLUME_ID \
  --instance-id $INSTANCE_ID \
  --region $REGION

# Wait for attachment
aws ec2 wait volume-in-use --volume-ids $VOLUME_ID --region $REGION

# Format and mount (inside the instance)
sleep 5  # wait for device to appear
sudo mkfs.xfs /dev/xvdf
sudo mkdir -p /data
sudo mount /dev/xvdf /data
```

## Gotchas

- **AZ-locked**: EBS volumes exist in a single AZ. You cannot attach a volume in `us-east-1a` to an instance in `us-east-1b`. Migrate via snapshot.
- **One attachment at a time** (unless multi-attach io1/io2). Detach before attaching elsewhere.
- **Modification cooldown**: After modifying a volume (resize, type change), you must wait 6 hours before modifying again.
- **gp2 burst depletion**: Small gp2 volumes (< 1 TiB) can run out of burst credits under sustained load. Migrate to gp3.
- **DeleteOnTermination**: Root volumes default to `DeleteOnTermination=true`. Additional volumes default to `false`. Verify before terminating instances.
- **Snapshot is incremental**: Only changed blocks are stored after the first snapshot. Deleting a snapshot doesn't lose data — it redistributes blocks to remaining snapshots.
- **Encryption is free**: No performance penalty or extra cost for encryption. Enable by default.
- **Nitro required for io2 Block Express**: Only Nitro-based instances support io2 Block Express (256K IOPS).
- **Device names on Nitro**: Nitro instances use NVMe. `/dev/xvdf` appears as `/dev/nvme1n1`. Use `lsblk` or `nvme list` to find the mapping.
