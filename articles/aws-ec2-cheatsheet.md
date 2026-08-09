# AWS EC2 Cheatsheet

## Instances

### List and Describe

```bash
# List all instances
aws ec2 describe-instances

# List with specific fields (table format)
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' \
    --output table

# List running instances only
aws ec2 describe-instances \
    --filters "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,PublicIpAddress]' \
    --output table

# Get instance by ID
aws ec2 describe-instances --instance-ids i-0123456789abcdef0

# Get instance by name tag
aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=my-server"

# Get instance by multiple filters
aws ec2 describe-instances \
    --filters \
        "Name=instance-type,Values=t3.micro" \
        "Name=instance-state-name,Values=running"

# Get only instance IDs
aws ec2 describe-instances \
    --filters "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].InstanceId' \
    --output text

# Get public IP of an instance
aws ec2 describe-instances \
    --instance-ids i-0123456789abcdef0 \
    --query 'Reservations[0].Instances[0].PublicIpAddress' \
    --output text

# Get private IP
aws ec2 describe-instances \
    --instance-ids i-0123456789abcdef0 \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' \
    --output text
```

### Start, Stop, Reboot, Terminate

```bash
# Start an instance
aws ec2 start-instances --instance-ids i-0123456789abcdef0

# Start multiple instances
aws ec2 start-instances --instance-ids i-0123456789abcdef0 i-0123456789abcdef1

# Stop an instance
aws ec2 stop-instances --instance-ids i-0123456789abcdef0

# Force stop (like pulling the power cord)
aws ec2 stop-instances --instance-ids i-0123456789abcdef0 --force

# Reboot an instance
aws ec2 reboot-instances --instance-ids i-0123456789abcdef0

# Terminate an instance (permanent deletion)
aws ec2 terminate-instances --instance-ids i-0123456789abcdef0

# Terminate with text output (useful in scripts)
aws ec2 terminate-instances --instance-ids i-0123456789abcdef0 --output text

# Enable termination protection
aws ec2 modify-instance-attribute \
    --instance-id i-0123456789abcdef0 \
    --disable-api-termination

# Disable termination protection
aws ec2 modify-instance-attribute \
    --instance-id i-0123456789abcdef0 \
    --no-disable-api-termination
```

### Launch Instances

```bash
# Launch a basic instance
aws ec2 run-instances \
    --image-id ami-0123456789abcdef0 \
    --instance-type t3.micro \
    --key-name my-key-pair \
    --security-group-ids sg-0123456789abcdef0 \
    --subnet-id subnet-0123456789abcdef0

# Launch with name tag
aws ec2 run-instances \
    --image-id ami-0123456789abcdef0 \
    --instance-type t3.micro \
    --key-name my-key-pair \
    --security-group-ids sg-0123456789abcdef0 \
    --subnet-id subnet-0123456789abcdef0 \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=my-server}]'

# Launch with user data (startup script)
aws ec2 run-instances \
    --image-id ami-0123456789abcdef0 \
    --instance-type t3.micro \
    --key-name my-key-pair \
    --user-data file://startup-script.sh

# Launch with IAM instance profile
aws ec2 run-instances \
    --image-id ami-0123456789abcdef0 \
    --instance-type t3.medium \
    --key-name my-key-pair \
    --iam-instance-profile Name=my-instance-role

# Launch with EBS volume configuration
aws ec2 run-instances \
    --image-id ami-0123456789abcdef0 \
    --instance-type t3.micro \
    --key-name my-key-pair \
    --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":50,"VolumeType":"gp3"}}]'

# Launch spot instance
aws ec2 run-instances \
    --image-id ami-0123456789abcdef0 \
    --instance-type t3.micro \
    --instance-market-options '{"MarketType":"spot","SpotOptions":{"MaxPrice":"0.01"}}'

# Dry run (check permissions without launching)
aws ec2 run-instances \
    --image-id ami-0123456789abcdef0 \
    --instance-type t3.micro \
    --key-name my-key-pair \
    --dry-run
```

### Modify Instances

```bash
# Change instance type (instance must be stopped)
aws ec2 modify-instance-attribute \
    --instance-id i-0123456789abcdef0 \
    --instance-type '{"Value":"t3.large"}'

# Enable detailed monitoring
aws ec2 monitor-instances --instance-ids i-0123456789abcdef0

# Disable detailed monitoring
aws ec2 unmonitor-instances --instance-ids i-0123456789abcdef0

# Change security groups
aws ec2 modify-instance-attribute \
    --instance-id i-0123456789abcdef0 \
    --groups sg-0123456789abcdef0 sg-0123456789abcdef1

# Change user data (instance must be stopped)
aws ec2 modify-instance-attribute \
    --instance-id i-0123456789abcdef0 \
    --user-data file://new-user-data.sh

# Enable/disable source-dest check (for NAT instances)
aws ec2 modify-instance-attribute \
    --instance-id i-0123456789abcdef0 \
    --no-source-dest-check
```

### Tags

```bash
# Add tags to instance
aws ec2 create-tags \
    --resources i-0123456789abcdef0 \
    --tags Key=Environment,Value=production Key=Team,Value=backend

# Remove tags
aws ec2 delete-tags \
    --resources i-0123456789abcdef0 \
    --tags Key=Environment

# List instances by tag
aws ec2 describe-instances \
    --filters "Name=tag:Environment,Values=production" \
    --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value|[0]]' \
    --output table
```

### Connect

```bash
# SSH to instance
ssh -i ~/.ssh/my-key.pem ec2-user@<public-ip>

# SSH to Amazon Linux 2
ssh -i ~/.ssh/my-key.pem ec2-user@<public-ip>

# SSH to Ubuntu
ssh -i ~/.ssh/my-key.pem ubuntu@<public-ip>

# SSH to Debian
ssh -i ~/.ssh/my-key.pem admin@<public-ip>

# SSH to RHEL
ssh -i ~/.ssh/my-key.pem ec2-user@<public-ip>

# EC2 Instance Connect (browser-based or CLI)
aws ec2-instance-connect send-ssh-public-key \
    --instance-id i-0123456789abcdef0 \
    --instance-os-user ec2-user \
    --ssh-public-key file://~/.ssh/id_ed25519.pub

# SSM Session Manager (no SSH key needed, no open ports)
aws ssm start-session --target i-0123456789abcdef0

# SSM port forwarding
aws ssm start-session \
    --target i-0123456789abcdef0 \
    --document-name AWS-StartPortForwardingSession \
    --parameters '{"portNumber":["8080"],"localPortNumber":["8080"]}'
```

## AMIs (Amazon Machine Images)

```bash
# List your own AMIs
aws ec2 describe-images --owners self

# Describe a specific AMI in a specific region
aws ec2 describe-images --region eu-west-2 --image-ids ami-0123456789abcdef0

# Find Amazon Linux 2023 AMIs
aws ec2 describe-images \
    --owners amazon \
    --filters "Name=name,Values=al2023-ami-*-x86_64" \
    --query 'Images | sort_by(@, &CreationDate) | [-1].[ImageId,Name]' \
    --output text

# Find Ubuntu AMIs
aws ec2 describe-images \
    --owners 099720109477 \
    --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-*-24.04-amd64*" \
    --query 'Images | sort_by(@, &CreationDate) | [-1].[ImageId,Name]' \
    --output text

# Create AMI from an instance
aws ec2 create-image \
    --instance-id i-0123456789abcdef0 \
    --name "my-server-backup-$(date +%Y%m%d)" \
    --no-reboot

# Create AMI with reboot (cleaner snapshot)
aws ec2 create-image \
    --instance-id i-0123456789abcdef0 \
    --name "my-server-backup-$(date +%Y%m%d)"

# Deregister (delete) an AMI
aws ec2 deregister-image --image-id ami-0123456789abcdef0

# Copy AMI to another region
aws ec2 copy-image \
    --source-image-id ami-0123456789abcdef0 \
    --source-region us-east-1 \
    --name "my-ami-copy" \
    --region eu-west-1

# Share AMI with another account
aws ec2 modify-image-attribute \
    --image-id ami-0123456789abcdef0 \
    --launch-permission "Add=[{UserId=123456789012}]"
```

## Security Groups

```bash
# List security groups
aws ec2 describe-security-groups

# List with name filter
aws ec2 describe-security-groups \
    --filters "Name=group-name,Values=my-sg"

# Create a security group
aws ec2 create-security-group \
    --group-name my-sg \
    --description "My security group" \
    --vpc-id vpc-0123456789abcdef0

# Add inbound rule (SSH from specific IP)
aws ec2 authorize-security-group-ingress \
    --group-id sg-0123456789abcdef0 \
    --protocol tcp \
    --port 22 \
    --cidr 203.0.113.0/32

# Add inbound rule (HTTP from anywhere)
aws ec2 authorize-security-group-ingress \
    --group-id sg-0123456789abcdef0 \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0

# Add inbound rule (port range)
aws ec2 authorize-security-group-ingress \
    --group-id sg-0123456789abcdef0 \
    --protocol tcp \
    --port 8000-9000 \
    --cidr 10.0.0.0/16

# Add inbound rule (from another security group)
aws ec2 authorize-security-group-ingress \
    --group-id sg-0123456789abcdef0 \
    --protocol tcp \
    --port 5432 \
    --source-group sg-0987654321fedcba0

# Remove inbound rule
aws ec2 revoke-security-group-ingress \
    --group-id sg-0123456789abcdef0 \
    --protocol tcp \
    --port 22 \
    --cidr 203.0.113.0/32

# Add outbound rule
aws ec2 authorize-security-group-egress \
    --group-id sg-0123456789abcdef0 \
    --protocol tcp \
    --port 443 \
    --cidr 0.0.0.0/0

# Delete security group
aws ec2 delete-security-group --group-id sg-0123456789abcdef0
```

## Key Pairs

```bash
# List key pairs
aws ec2 describe-key-pairs

# Create a new key pair (save the private key!)
aws ec2 create-key-pair \
    --key-name my-key-pair \
    --query 'KeyMaterial' \
    --output text > ~/.ssh/my-key-pair.pem
chmod 600 ~/.ssh/my-key-pair.pem

# Import existing public key
aws ec2 import-key-pair \
    --key-name my-key-pair \
    --public-key-material fileb://~/.ssh/id_ed25519.pub

# Delete a key pair
aws ec2 delete-key-pair --key-name my-key-pair
```

## EBS Volumes

```bash
# List volumes
aws ec2 describe-volumes

# Pretty-print with jq
aws ec2 describe-volumes | jq .

# List all volume IDs (JMESPath)
aws ec2 describe-volumes --query 'Volumes[*].VolumeId'

# List volume IDs and types (JMESPath)
aws ec2 describe-volumes --query 'Volumes[].[VolumeId, VolumeType]'

# List all volume IDs (jq)
aws ec2 describe-volumes --output json | jq -r '.Volumes[].VolumeId'

# List volume IDs and types (jq)
aws ec2 describe-volumes --output json | jq -r '.Volumes[] | .VolumeId, .VolumeType'

# Find available (unattached) volumes (jq)
aws ec2 describe-volumes | jq '.Volumes[] | select(.State=="available") | .VolumeId'

# Available volumes without quotes
aws ec2 describe-volumes | jq '.Volumes[] | select(.State=="available") | .VolumeId' | tr -d \"

# List volumes attached to an instance
aws ec2 describe-volumes \
    --filters "Name=attachment.instance-id,Values=i-0123456789abcdef0"

# Create a volume
aws ec2 create-volume \
    --availability-zone us-east-1a \
    --size 100 \
    --volume-type gp3

# Create encrypted volume
aws ec2 create-volume \
    --availability-zone us-east-1a \
    --size 100 \
    --volume-type gp3 \
    --encrypted

# Attach volume to instance
aws ec2 attach-volume \
    --volume-id vol-0123456789abcdef0 \
    --instance-id i-0123456789abcdef0 \
    --device /dev/xvdf

# Detach volume
aws ec2 detach-volume --volume-id vol-0123456789abcdef0

# Force detach
aws ec2 detach-volume --volume-id vol-0123456789abcdef0 --force

# Delete volume
aws ec2 delete-volume --volume-id vol-0123456789abcdef0

# Modify volume (resize, change type — no downtime)
aws ec2 modify-volume \
    --volume-id vol-0123456789abcdef0 \
    --size 200 \
    --volume-type gp3

# Create snapshot
aws ec2 create-snapshot \
    --volume-id vol-0123456789abcdef0 \
    --description "Backup $(date +%Y%m%d)"

# List snapshots
aws ec2 describe-snapshots --owner-ids self

# Describe a specific snapshot
aws ec2 describe-snapshots --snapshot-ids snap-0123456789abcdef0

# Delete snapshot
aws ec2 delete-snapshot --snapshot-id snap-0123456789abcdef0

# Create volume from snapshot
aws ec2 create-volume \
    --snapshot-id snap-0123456789abcdef0 \
    --availability-zone us-east-1a \
    --volume-type gp3
```

## Elastic IPs

```bash
# Allocate a new Elastic IP
aws ec2 allocate-address --domain vpc

# List Elastic IPs
aws ec2 describe-addresses

# Associate Elastic IP with instance
aws ec2 associate-address \
    --allocation-id eipalloc-0123456789abcdef0 \
    --instance-id i-0123456789abcdef0

# Disassociate Elastic IP
aws ec2 disassociate-address --association-id eipassoc-0123456789abcdef0

# Release (delete) Elastic IP
aws ec2 release-address --allocation-id eipalloc-0123456789abcdef0
```

## VPC and Networking

```bash
# List VPCs
aws ec2 describe-vpcs

# List subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-0123456789abcdef0"

# List route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-0123456789abcdef0"

# List internet gateways
aws ec2 describe-internet-gateways

# List NAT gateways
aws ec2 describe-nat-gateways

# List network interfaces
aws ec2 describe-network-interfaces \
    --filters "Name=attachment.instance-id,Values=i-0123456789abcdef0"

# List all IPs in a subnet
aws ec2 describe-network-interfaces \
    --filters Name=subnet-id,Values=subnet-0123456789abcdef0 \
    | jq -r '.NetworkInterfaces[].PrivateIpAddresses[].PrivateIpAddress' | sort

# Get public IP of NAT gateway
aws ec2 describe-nat-gateways \
    --query 'NatGateways[*].[NatGatewayId,NatGatewayAddresses[0].PublicIp]' \
    --output table
```

## Instance Metadata (From Inside the Instance)

```bash
# IMDSv2 — get token first
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Instance ID
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/instance-id

# Public IP
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/public-ipv4

# Private IP
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/local-ipv4

# Instance type
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/instance-type

# Availability zone
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/placement/availability-zone

# Region
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/placement/region

# IAM role credentials
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Security groups
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/security-groups

# User data
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/user-data

# All available metadata categories
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/
```

## CloudWatch Monitoring

```bash
# Get CPU utilization for an instance (last hour)
aws cloudwatch get-metric-statistics \
    --namespace AWS/EC2 \
    --metric-name CPUUtilization \
    --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average

# Get status check failures
aws cloudwatch get-metric-statistics \
    --namespace AWS/EC2 \
    --metric-name StatusCheckFailed \
    --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Maximum

# List available metrics for EC2
aws cloudwatch list-metrics --namespace AWS/EC2
```

## Instance Status and Events

```bash
# Check instance status
aws ec2 describe-instance-status --instance-ids i-0123456789abcdef0

# Check all instances with impaired status
aws ec2 describe-instance-status \
    --filters "Name=instance-status.status,Values=impaired"

# Get console output (boot logs)
aws ec2 get-console-output --instance-id i-0123456789abcdef0

# Get screenshot (for Windows or GUI debugging)
aws ec2 get-console-screenshot --instance-id i-0123456789abcdef0
```

## Useful Patterns

### Stop All Running Instances (by tag)

```bash
aws ec2 describe-instances \
    --filters "Name=tag:Environment,Values=dev" "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].InstanceId' \
    --output text | xargs aws ec2 stop-instances --instance-ids
```

### Find Untagged Instances

```bash
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[?!not_null(Tags[?Key==`Name`].Value|[0])].[InstanceId,InstanceType,State.Name]' \
    --output table
```

### List Instances with Costs

```bash
aws ec2 describe-instances \
    --filters "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,Tags[?Key==`Name`].Value|[0],Placement.AvailabilityZone]' \
    --output table
```

### Find Old Snapshots (Older than 30 Days)

```bash
aws ec2 describe-snapshots \
    --owner-ids self \
    --query "Snapshots[?StartTime<='$(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%S)'].[SnapshotId,VolumeId,StartTime,VolumeSize]" \
    --output table
```

### Find Unused Elastic IPs

```bash
aws ec2 describe-addresses \
    --query 'Addresses[?!InstanceId].[PublicIp,AllocationId]' \
    --output table
```

### Find Unused EBS Volumes

```bash
aws ec2 describe-volumes \
    --filters "Name=status,Values=available" \
    --query 'Volumes[*].[VolumeId,Size,VolumeType,CreateTime]' \
    --output table
```

### Change Instance Type for All Stopped Instances

```bash
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[*].[State.Name, InstanceId]' \
    --output text |
grep stopped |
awk '{print $2}' |
while read line; do
    aws ec2 modify-instance-attribute --instance-id $line --instance-type '{"Value": "t3.medium"}'
done
```

### Wait for Instance State

```bash
# Wait until instance is running
aws ec2 wait instance-running --instance-ids i-0123456789abcdef0

# Wait until instance is stopped
aws ec2 wait instance-stopped --instance-ids i-0123456789abcdef0

# Wait until instance is terminated
aws ec2 wait instance-terminated --instance-ids i-0123456789abcdef0

# Wait until status checks pass
aws ec2 wait instance-status-ok --instance-ids i-0123456789abcdef0
```

## AWS CLI Configuration

```bash
# Configure default profile
aws configure

# Configure named profile
aws configure --profile production

# Use a specific profile
aws ec2 describe-instances --profile production

# Set region per command
aws ec2 describe-instances --region eu-west-1

# List all regions
aws ec2 describe-regions --output table

# Output formats: json (default), text, table, yaml
aws ec2 describe-instances --output table

# Enable CLI auto-prompt
aws --cli-auto-prompt
```

## Common JMESPath Queries

```bash
# Get instance ID and name
--query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value|[0]]'

# Get instances in a specific state
--query 'Reservations[*].Instances[?State.Name==`running`]'

# Sort by launch time
--query 'Reservations[*].Instances[*] | sort_by(@, &LaunchTime)'

# Get first result only
--query 'Reservations[0].Instances[0].InstanceId'

# Filter by instance type
--query 'Reservations[*].Instances[?InstanceType==`t3.micro`]'

# Count instances
--query 'Reservations[*].Instances[*] | length(@)'
```

## Install SSM Agent

```bash
# Amazon Linux / RHEL / CentOS
sudo dnf install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
sudo systemctl status amazon-ssm-agent

# Ubuntu / Debian
sudo snap install amazon-ssm-agent --classic
sudo systemctl status snap.amazon-ssm-agent.amazon-ssm-agent.service
```

## NVMe Tools

EC2 instances with NVMe-based EBS volumes (Nitro instances) require `nvme-cli` to identify volumes:

```bash
# Install
sudo yum install -y nvme-cli    # RHEL/Amazon Linux
sudo apt install -y nvme-cli    # Ubuntu/Debian

# List NVMe devices
nvme list

# Get serial number (maps to EBS volume ID)
nvme id-ctrl -v /dev/nvme1n1 | grep sn

# The serial number matches the EBS volume ID (without the vol- prefix and dashes)
```

## User Data

EC2 user data runs on first boot and accepts two formats:

- **Shell scripts** — must start with `#!/bin/bash` (or another shebang)
- **cloud-init directives** — must start with `#cloud-config`

```bash
# View user data of a running instance (from inside)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/user-data

# Check user data logs
sudo cat /var/log/cloud-init-output.log

# Launch with user data from file
aws ec2 run-instances \
    --image-id ami-0123456789abcdef0 \
    --instance-type t3.micro \
    --user-data file://startup-script.sh
```

## IMDSv2 (Instance Metadata Service v2)

IMDSv2 adds token-based security to the metadata endpoint. Always use IMDSv2 — it prevents SSRF attacks.

### Check IMDSv2 Status

```bash
aws ec2 describe-instances \
    --instance-id i-0123456789abcdef0 \
    --query "Reservations[0].Instances[0].MetadataOptions"
```

### Enforce IMDSv2 on Existing Instance

```bash
aws ec2 modify-instance-metadata-options \
    --instance-id i-0123456789abcdef0 \
    --http-tokens required \
    --http-endpoint enabled
```

### Enforce IMDSv2 with Terraform

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0123456789abcdef0"
  instance_type = "t3.micro"

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"    # Enforces IMDSv2
  }
}
```

### Using IMDSv2 (Token-Based Requests)

```bash
# Step 1: Get a token
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Step 2: Use the token in metadata requests
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/instance-id
```

## Launch Templates

```bash
# Create a launch template
aws ec2 create-launch-template \
    --launch-template-name my-template \
    --launch-template-data '{
        "ImageId":"ami-0123456789abcdef0",
        "InstanceType":"t3.micro",
        "KeyName":"my-key-pair",
        "SecurityGroupIds":["sg-0123456789abcdef0"],
        "TagSpecifications":[{
            "ResourceType":"instance",
            "Tags":[{"Key":"Name","Value":"my-instance"}]
        }]
    }'

# List launch templates
aws ec2 describe-launch-templates

# Describe a launch template (all versions)
aws ec2 describe-launch-template-versions \
    --launch-template-name my-template

# Get latest version
aws ec2 describe-launch-template-versions \
    --launch-template-name my-template \
    --versions '$Latest'

# Launch instance from template
aws ec2 run-instances \
    --launch-template LaunchTemplateName=my-template,Version='$Latest'

# Launch with template overrides
aws ec2 run-instances \
    --launch-template LaunchTemplateName=my-template,Version='$Latest' \
    --instance-type t3.large

# Delete launch template
aws ec2 delete-launch-template --launch-template-name my-template
```

## Instance Types and Availability Zones

```bash
# List all available instance types in current region
aws ec2 describe-instance-types \
    --query 'InstanceTypes[*].InstanceType' \
    --output text

# List instance types with details (vCPUs, memory)
aws ec2 describe-instance-types \
    --query 'InstanceTypes[*].[InstanceType,VCpuInfo.DefaultVCpus,MemoryInfo.SizeInMiB]' \
    --output table

# Filter by instance family
aws ec2 describe-instance-types \
    --filters "Name=instance-type,Values=t3.*" \
    --query 'InstanceTypes[*].[InstanceType,VCpuInfo.DefaultVCpus,MemoryInfo.SizeInMiB]' \
    --output table

# List availability zones
aws ec2 describe-availability-zones --output table

# Check which instance types are available in a specific AZ
aws ec2 describe-instance-type-offerings \
    --location-type availability-zone \
    --filters "Name=location,Values=us-east-1a" \
    --query 'InstanceTypeOfferings[*].InstanceType' \
    --output text
```

## Placement Groups

```bash
# Create a placement group (cluster — low latency)
aws ec2 create-placement-group \
    --group-name my-cluster \
    --strategy cluster

# Create spread placement group (max availability)
aws ec2 create-placement-group \
    --group-name my-spread \
    --strategy spread

# Create partition placement group
aws ec2 create-placement-group \
    --group-name my-partition \
    --strategy partition \
    --partition-count 3

# List placement groups
aws ec2 describe-placement-groups

# Launch into a placement group
aws ec2 run-instances \
    --image-id ami-0123456789abcdef0 \
    --instance-type c5.xlarge \
    --placement "GroupName=my-cluster"

# Delete placement group
aws ec2 delete-placement-group --group-name my-cluster
```

## CLI Tips and Tricks

### --filter vs --query

`--filter` is processed **server-side** (reduces data transfer and improves performance for large datasets). `--query` is processed **client-side** (formats the output after receiving it).

Always prefer `--filter` for narrowing results, then use `--query` for formatting:

```bash
# Good: filter server-side, then format client-side
aws ec2 describe-instances \
    --filters "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value|[0]]' \
    --output table

# Less efficient: query does all the work client-side
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[?State.Name==`running`].[InstanceId]'
```

### Filter by Tag Key Existence

```bash
# Find instances that have a specific tag (regardless of value)
aws ec2 describe-instances \
    --filters "Name=tag-key,Values=Environment" \
    --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Environment`].Value|[0]]' \
    --output table
```

### Disable Pager

```bash
# Per-command
aws ec2 describe-instances --no-cli-pager

# Per-session
export AWS_PAGER=""

# In ~/.aws/config
# [default]
# cli_pager =
```

### Shell Variables in Queries

Shell variables are automatically substituted even inside single quotes in `--query`:

```bash
CLUSTER="production"
aws ec2 describe-instances \
    --filters "Name=tag:Cluster,Values=$CLUSTER" \
    --query 'Reservations[*].Instances[*].InstanceId' \
    --output text
```

### Nest Command Output

Use `$()` to pass output from one command to another:

```bash
# Get IDs then describe them
aws ec2 describe-instances \
    --instance-ids $(aws ec2 describe-instances \
        --filters "Name=tag:Environment,Values=dev" \
        --query 'Reservations[*].Instances[*].InstanceId' \
        --output text)
```

### Convert Output to Comma-Separated List

Some AWS commands expect comma-separated input:

```bash
# Convert newline-separated IDs to comma-separated
aws ec2 describe-instances \
    --filters "Name=tag:Cluster,Values=prod" \
    --query 'Reservations[*].Instances[*].InstanceId' \
    --output text | tr '[:space:]' ',' | sed 's/,$//'
```

### Pagination

```bash
# Disable pagination (get all results at once)
aws ec2 describe-instances --no-paginate

# Set max items per page
aws ec2 describe-instances --max-items 100

# Use a starting token for manual pagination
aws ec2 describe-instances --starting-token <token-from-previous-call>
```

### Query Security Group by Name

```bash
aws ec2 describe-security-groups \
    --query 'SecurityGroups[?GroupName==`my-sg-name`].[GroupId,VpcId]' \
    --output text
```

### Block Device Mappings (Advanced)

```bash
# Additional EBS volume from snapshot
aws ec2 run-instances \
    --image-id ami-0123456789abcdef0 \
    --instance-type t3.micro \
    --block-device-mappings '[
        {"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":30,"VolumeType":"gp3"}},
        {"DeviceName":"/dev/xvdf","Ebs":{"SnapshotId":"snap-0123456789abcdef0"}},
        {"DeviceName":"/dev/xvdg","Ebs":{"VolumeSize":100,"VolumeType":"gp3","Encrypted":true}}
    ]'

# Ephemeral (instance store) volumes
aws ec2 run-instances \
    --image-id ami-0123456789abcdef0 \
    --instance-type m5d.large \
    --block-device-mappings '[
        {"DeviceName":"/dev/xvdb","VirtualName":"ephemeral0"},
        {"DeviceName":"/dev/xvdc","VirtualName":"ephemeral1"}
    ]'
```
