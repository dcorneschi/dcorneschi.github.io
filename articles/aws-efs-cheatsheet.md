# AWS EFS Cheatsheet

Amazon Elastic File System (EFS) is a fully managed, scalable NFS file system for use with AWS services and on-premises resources. It grows and shrinks automatically as you add/remove files — no provisioning required.

## Key Features

| Feature | Detail |
|---------|--------|
| Protocol | NFSv4.1 |
| Scaling | Automatic, petabyte-scale |
| Availability | Multi-AZ (Regional) or Single-AZ (One Zone) |
| Access | EC2, ECS, EKS, Lambda, on-premises (via Direct Connect/VPN) |
| Encryption | At rest (KMS) and in transit (TLS) |
| Performance modes | General Purpose, Max I/O |
| Throughput modes | Bursting, Provisioned, Elastic |

## Create File System

### CLI

```bash
# Create regional file system (multi-AZ)
aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=Name,Value=my-efs

# Create with creation token (idempotent creation)
aws efs create-file-system \
  --creation-token my-efs-token \
  --performance-mode generalPurpose \
  --encrypted

# Create One Zone file system (single-AZ, cheaper)
aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --availability-zone-name eu-west-1a \
  --tags Key=Name,Value=my-efs-onezone

# List file systems
aws efs describe-file-systems --query "FileSystems[].[FileSystemId,Name,SizeInBytes.Value,LifeCycleState]" --output table

# Get file system details
aws efs describe-file-systems --file-system-id fs-12345678

# Get file system by creation token
aws efs describe-file-systems --creation-token my-efs-token
```

### Mount Targets

Each AZ needs a mount target for instances in that AZ to access EFS:

```bash
# Create mount target in a subnet
aws efs create-mount-target \
  --file-system-id fs-12345678 \
  --subnet-id subnet-aaa \
  --security-groups sg-123

# Create mount targets in all AZs
for SUBNET in subnet-aaa subnet-bbb subnet-ccc; do
  aws efs create-mount-target \
    --file-system-id fs-12345678 \
    --subnet-id $SUBNET \
    --security-groups sg-123
done

# List mount targets
aws efs describe-mount-targets --file-system-id fs-12345678

# Delete mount target
aws efs delete-mount-target --mount-target-id fsmt-12345678
```

## Mounting on EC2

### Prerequisites

```bash
# Install EFS utilities (Amazon Linux / RHEL)
sudo yum install -y amazon-efs-utils

# Install EFS utilities (Ubuntu — from package if available)
sudo apt install -y amazon-efs-utils

# Install EFS utilities (Debian/Ubuntu — build from source)
sudo apt-get update
sudo apt-get -y install git binutils
git clone https://github.com/aws/efs-utils
cd efs-utils
./build-deb.sh
sudo apt-get -y install ./build/amazon-efs-utils*deb

# Or install NFS client only
sudo yum install -y nfs-utils        # RHEL
sudo apt install -y nfs-common       # Ubuntu
```

### Mount with EFS Helper (Recommended)

```bash
# Mount with encryption in transit (TLS)
sudo mount -t efs -o tls fs-12345678:/ /mnt/efs

# Mount specific access point
sudo mount -t efs -o tls,accesspoint=fsap-12345678 fs-12345678:/ /mnt/efs

# Mount with IAM authorization
sudo mount -t efs -o tls,iam fs-12345678:/ /mnt/efs
```

### Mount with NFS (Without EFS Helper)

```bash
# Standard NFS mount
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport \
  fs-12345678.efs.eu-west-1.amazonaws.com:/ /mnt/efs

# With AZ-specific DNS name
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport \
  availability-zone.fs-12345678.efs.eu-west-1.amazonaws.com:/ /mnt/efs

# Mount by IP address (mount target IP)
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport \
  172.31.21.52:/ /mnt/efs
```

### Persistent Mount (fstab)

```bash
# Using EFS helper
echo "fs-12345678:/ /mnt/efs efs _netdev,tls 0 0" | sudo tee -a /etc/fstab

# Using NFS
echo "fs-12345678.efs.eu-west-1.amazonaws.com:/ /mnt/efs nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,_netdev 0 0" | sudo tee -a /etc/fstab

# Mount all fstab entries
sudo mount -a
```

## Access Points

Access points provide application-specific entry points with enforced user identity and root directory:

```bash
# Create access point
aws efs create-access-point \
  --file-system-id fs-12345678 \
  --posix-user "Uid=1000,Gid=1000" \
  --root-directory "Path=/app-data,CreationInfo={OwnerUid=1000,OwnerGid=1000,Permissions=755}" \
  --tags Key=Name,Value=app-access-point

# List access points
aws efs describe-access-points --file-system-id fs-12345678

# Delete access point
aws efs delete-access-point --access-point-id fsap-12345678
```

### Use Cases for Access Points

- Per-application isolation (each app gets its own subdirectory)
- Enforce POSIX user/group without trusting the client
- ECS/EKS workloads that need specific UID/GID mapping
- Lambda function file system access

## Security

### Security Groups

EFS mount targets need a security group allowing NFS traffic (port 2049):

```bash
# Create security group for EFS
aws ec2 create-security-group \
  --group-name efs-sg \
  --description "Allow NFS from ECS tasks" \
  --vpc-id vpc-123

# Allow inbound NFS from ECS task security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-efs \
  --protocol tcp \
  --port 2049 \
  --source-group sg-ecs-tasks
```

### File System Policy (Resource-Based)

```bash
aws efs put-file-system-policy \
  --file-system-id fs-12345678 \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::123456789012:role/ecsTaskRole"},
        "Action": [
          "elasticfilesystem:ClientMount",
          "elasticfilesystem:ClientWrite"
        ],
        "Condition": {
          "Bool": {"elasticfilesystem:AccessedViaMountTarget": "true"}
        }
      },
      {
        "Effect": "Deny",
        "Principal": {"AWS": "*"},
        "Action": "*",
        "Condition": {
          "Bool": {"aws:SecureTransport": "false"}
        }
      }
    ]
  }'
```

### Encryption

```bash
# Create encrypted file system (at rest, using default KMS key)
aws efs create-file-system --encrypted

# Create with custom KMS key
aws efs create-file-system --encrypted --kms-key-id arn:aws:kms:eu-west-1:123456789012:key/key-id

# Encryption in transit is handled at mount time with -o tls
```

### Root Squash

Enforce root squash via a file system policy that denies `ClientRootAccess`:

```bash
aws efs put-file-system-policy \
  --file-system-id fs-12345678 \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {"AWS": "*"},
        "Action": [
          "elasticfilesystem:ClientMount",
          "elasticfilesystem:ClientWrite"
        ],
        "Condition": {
          "Bool": {"elasticfilesystem:AccessedViaMountTarget": "true"}
        }
      },
      {
        "Effect": "Deny",
        "Principal": {"AWS": "*"},
        "Action": "elasticfilesystem:ClientRootAccess",
        "Condition": {
          "Bool": {"elasticfilesystem:AccessedViaMountTarget": "true"}
        }
      }
    ]
  }'
```

### File System Policy

```bash
# Get current policy
aws efs describe-file-system-policy --file-system-id fs-12345678
```

### Mount Options Reference

| Option | Value | Purpose |
|--------|-------|---------|
| `nfsvers` | 4.1 | NFS version (4.1 recommended) |
| `rsize` | 1048576 | Read buffer size (1MB recommended) |
| `wsize` | 1048576 | Write buffer size (1MB recommended) |
| `hard` | - | Retry on timeout (recommended) |
| `timeo` | 600 | Timeout in deciseconds (60s) |
| `retrans` | 2 | Number of retries |
| `nconnect` | 16 | Multiple TCP connections (throughput optimization) |
| `noresvport` | - | Use non-privileged source port (recommended for EFS) |

### Optimize for Throughput

```bash
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,tcp,nconnect=16 \
  fs-12345678.efs.eu-west-1.amazonaws.com:/ /mnt/efs
```

## Performance and Throughput

### Performance Modes

| Mode | Use Case | IOPS |
|------|----------|------|
| General Purpose (default) | Latency-sensitive workloads (web, CMS, dev) | Up to 35,000 read / 7,000 write |
| Max I/O | Highly parallelized workloads (big data, media) | Unlimited (higher latency) |

### Throughput Modes

| Mode | Behavior |
|------|----------|
| Bursting (default) | Throughput scales with file system size, burst credits |
| Provisioned | Fixed throughput regardless of size |
| Elastic | Automatically scales throughput up/down based on workload |

```bash
# Change to provisioned throughput
aws efs update-file-system \
  --file-system-id fs-12345678 \
  --throughput-mode provisioned \
  --provisioned-throughput-in-mibps 100

# Change to elastic throughput
aws efs update-file-system \
  --file-system-id fs-12345678 \
  --throughput-mode elastic

# Check burst credit balance
aws cloudwatch get-metric-statistics \
  --namespace AWS/EFS \
  --metric-name BurstCreditBalance \
  --dimensions Name=FileSystemId,Value=fs-12345678 \
  --start-time $(date -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average
```

## Lifecycle Management

Move infrequently accessed files to cheaper storage classes:

```bash
# Enable lifecycle policy (move to IA after 30 days)
aws efs put-lifecycle-configuration \
  --file-system-id fs-12345678 \
  --lifecycle-policies \
    "TransitionToIA=AFTER_30_DAYS" \
    "TransitionToPrimaryStorageClass=AFTER_1_ACCESS"

# Full Intelligent-Tiering (IA + Archive)
aws efs put-lifecycle-configuration \
  --file-system-id fs-12345678 \
  --lifecycle-policies \
    "TransitionToIA=AFTER_7_DAYS" \
    "TransitionToArchive=AFTER_90_DAYS" \
    "TransitionToPrimaryStorageClass=AFTER_1_ACCESS"

# Check current lifecycle policy
aws efs describe-lifecycle-configuration --file-system-id fs-12345678
```

### Storage Classes

| Class | Cost | Access Latency |
|-------|------|----------------|
| Standard | Higher | Low (sub-millisecond) |
| Infrequent Access (IA) | ~92% cheaper | Higher (first-byte latency) |
| Archive | ~95% cheaper | Higher |

## EFS with ECS

### Task Definition Volume

```json
{
  "volumes": [
    {
      "name": "efs-data",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-12345678",
        "transitEncryption": "ENABLED",
        "authorizationConfig": {
          "accessPointId": "fsap-12345678",
          "iam": "ENABLED"
        }
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "app",
      "mountPoints": [
        {
          "sourceVolume": "efs-data",
          "containerPath": "/data",
          "readOnly": false
        }
      ]
    }
  ]
}
```

## EFS with EKS

### StorageClass and PVC

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-12345678
  directoryPerms: "755"
  uid: "1000"
  gid: "1000"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

## EFS with Lambda

```bash
aws lambda update-function-configuration \
  --function-name my-function \
  --file-system-configs "Arn=arn:aws:elasticfilesystem:eu-west-1:123456789012:access-point/fsap-12345678,LocalMountPath=/mnt/efs"
```

## Monitoring

```bash
# File system size
aws efs describe-file-systems --file-system-id fs-12345678 \
  --query "FileSystems[].SizeInBytes.Value"

# CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EFS \
  --metric-name ClientConnections \
  --dimensions Name=FileSystemId,Value=fs-12345678 \
  --start-time $(date -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum
```

### Key Metrics

| Metric | Description |
|--------|-------------|
| `ClientConnections` | Number of active connections |
| `DataReadIOBytes` | Bytes read |
| `DataWriteIOBytes` | Bytes written |
| `BurstCreditBalance` | Remaining burst credits (bursting mode) |
| `PercentIOLimit` | How close to General Purpose IOPS limit |
| `TotalIOBytes` | Total I/O throughput |

## Backup

```bash
# Enable automatic backups (uses AWS Backup)
aws efs put-backup-policy \
  --file-system-id fs-12345678 \
  --backup-policy Status=ENABLED

# Check backup policy
aws efs describe-backup-policy --file-system-id fs-12345678

# Create backup vault
aws backup create-backup-vault --backup-vault-name my-efs-vault

# Restore from backup
aws backup start-restore-job \
  --recovery-point-arn arn:aws:backup:eu-west-1:123456789012:recovery-point:recovery-point-id \
  --iam-role-arn arn:aws:iam::123456789012:role/service-role/AWSBackupServiceRole
```

## Delete File System

```bash
# Must delete mount targets first
aws efs describe-mount-targets --file-system-id fs-12345678 \
  --query "MountTargets[].MountTargetId" --output text | \
  xargs -I{} aws efs delete-mount-target --mount-target-id {}

# Wait for mount targets to be deleted, then:
aws efs delete-file-system --file-system-id fs-12345678
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Mount hangs | Check security group allows port 2049 from client |
| Permission denied | Check POSIX permissions, access point UID/GID |
| Slow performance | Check burst credits, consider provisioned/elastic throughput |
| DNS resolution fails | Ensure VPC DNS resolution is enabled |
| Mount target not found | Create mount target in the same AZ as the instance |
| TLS mount fails | Install `amazon-efs-utils`, check `stunnel` is installed |
| Lambda timeout on mount | Ensure Lambda is in same VPC with route to EFS |
| Stale NFS file handle | Restart NFS client or remount the file system |
| NFS server not responding | Verify instance can reach mount target AZ, check NACLs |

### Connectivity Tests

```bash
# Test DNS resolution
nslookup fs-12345678.efs.eu-west-1.amazonaws.com

# Test NFS port connectivity
ping fs-12345678.efs.eu-west-1.amazonaws.com

# Check mount status
mount | grep efs
df -h /mnt/efs
```

### Unmount

```bash
sudo umount /mnt/efs
```

### NFS Statistics

```bash
# Client statistics
nfsstat -c

# Server statistics (if running NFS server)
nfsstat -s
```

## EFS Utils File Paths

| File | Purpose |
|------|---------|
| `/etc/amazon/efs/efs-utils.conf` | EFS mount helper configuration |
| `/var/log/amazon/efs/mount.log` | EFS mount helper log (troubleshooting) |

## Cost Optimization

### Storage Class Pricing

| Class | Relative Cost | Use Case |
|-------|--------------|----------|
| Standard | Highest | Frequently accessed files |
| Infrequent Access (IA) | ~92% cheaper | Occasional access |
| Archive | ~95% cheaper | Long-term storage, rarely accessed |

### Tips

- Use **One Zone** storage class for dev/test (47% cheaper than Regional)
- Enable **lifecycle policies** to move cold data to IA/Archive
- Use **Elastic throughput** instead of Provisioned when workload is spiky
- Monitor **BurstCreditBalance** — if consistently dropping, switch to Provisioned
- Use **access points** to organize data and apply policies per application
- Delete unused mount targets to avoid charges

## EKS CSI Driver Installation

### Install with eksctl and Helm

```bash
# Create IAM policy for EFS CSI driver
curl -o iam-policy-efs.json https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json

aws iam create-policy \
  --policy-name AmazonEKS_EFS_CSI_Driver_Policy \
  --policy-document file://iam-policy-efs.json

# Create service account with IAM role
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace kube-system \
  --name efs-csi-controller-sa \
  --attach-policy-arn arn:aws:iam::123456789012:policy/AmazonEKS_EFS_CSI_Driver_Policy \
  --approve

# Install EFS CSI driver via Helm
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=efs-csi-controller-sa
```

### Dynamic Provisioning StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-12345678
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic_provisioning"
```

Dynamic provisioning creates an access point per PVC, providing isolation between applications.

### Static PV and PVC

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-12345678
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

### Pod Using EFS

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: efs-storage
      mountPath: /data
  volumes:
  - name: efs-storage
    persistentVolumeClaim:
      claimName: efs-pvc
```

```bash
# Verify mount inside pod
kubectl exec app-pod -- df -h /data
kubectl exec app-pod -- touch /data/test.txt
```

### Multiple Pods Sharing EFS (ReadWriteMany)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shared-storage-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shared-app
  template:
    metadata:
      labels:
        app: shared-app
    spec:
      containers:
      - name: app
        image: nginx
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: efs-pvc
```

All 3 replicas share the same EFS volume — ideal for shared content or configuration.

### EKS Troubleshooting

```bash
# Check CSI driver pods
kubectl get pods -n kube-system | grep efs
kubectl get pods -n kube-system -l app=efs-csi-controller

# CSI controller logs
kubectl logs -n kube-system -l app=efs-csi-controller -c efs-plugin

# Check PVC status
kubectl describe pvc efs-pvc
```

## Backup Plan (AWS Backup)

### Create Backup Plan with Schedule

```bash
cat > backup-plan.json << 'EOF'
{
  "BackupPlanName": "efs-daily-backup",
  "Rules": [{
    "RuleName": "DailyBackup",
    "TargetBackupVaultName": "Default",
    "ScheduleExpression": "cron(0 5 ? * * *)",
    "StartWindowMinutes": 60,
    "CompletionWindowMinutes": 120,
    "Lifecycle": {
      "DeleteAfterDays": 30
    }
  }]
}
EOF

aws backup create-backup-plan --backup-plan file://backup-plan.json
```

### On-Demand Backup

```bash
aws backup start-backup-job \
  --backup-vault-name Default \
  --resource-arn arn:aws:elasticfilesystem:eu-west-1:123456789012:file-system/fs-12345678 \
  --iam-role-arn arn:aws:iam::123456789012:role/service-role/AWSBackupDefaultServiceRole
```

### Disable Backups

```bash
aws efs put-backup-policy --file-system-id fs-12345678 --backup-policy Status=DISABLED
```

## Performance Testing

```bash
# Test write performance
dd if=/dev/zero of=/mnt/efs/test-file bs=1M count=1000

# Test read performance
dd if=/mnt/efs/test-file of=/dev/null bs=1M

# Cleanup
rm /mnt/efs/test-file
```

## Verify Installation

```bash
# Check EFS utils installed (RHEL/Amazon Linux)
rpm -qa | grep amazon-efs-utils

# Check EFS utils installed (Debian/Ubuntu)
dpkg -l | grep amazon-efs-utils
```

## Connectivity Testing

```bash
# Test NFS port connectivity
telnet fs-12345678.efs.eu-west-1.amazonaws.com 2049

# DNS resolution
nslookup fs-12345678.efs.eu-west-1.amazonaws.com
```

## Storage Metrics

```bash
# Get storage bytes by class (Standard, IA, Archive)
aws cloudwatch get-metric-statistics \
  --namespace AWS/EFS \
  --metric-name StorageBytes \
  --dimensions Name=FileSystemId,Value=fs-12345678 Name=StorageClass,Value=Total \
  --start-time $(date -d '1 day ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date +%Y-%m-%dT%H:%M:%S) \
  --period 86400 \
  --statistics Average
```

## Best Practices

1. Use one mount target per AZ for high availability
2. Enable encryption at rest and in transit (`-o tls`)
3. Use access points for application isolation
4. Enable lifecycle management to reduce costs
5. Use provisioned/elastic throughput for consistent performance
6. Monitor burst credits in bursting mode
7. Use security groups to restrict access (port 2049 only from allowed sources)
8. Enable automatic backups via AWS Backup
9. Use IAM authentication (`-o tls,iam`) for enhanced security
10. Test failover between AZs
