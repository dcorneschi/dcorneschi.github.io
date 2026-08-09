# Installing AWS SSM Agent

AWS Systems Manager Agent (SSM Agent) enables Session Manager, Run Command, Patch Manager, and other Systems Manager features. It lets you manage EC2 instances without SSH, open ports, or bastion hosts.

## Pre-Installed Instances

SSM Agent comes pre-installed on:
- Amazon Linux 1 and 2
- Amazon Linux 2023
- Ubuntu 16.04+ (Snap-based)
- Windows Server 2016+
- Some macOS AMIs

For RHEL, CentOS, Rocky, AlmaLinux, Debian, and custom AMIs, install it manually.

## Quick Setup Steps

1. Create IAM instance profile with `AmazonSSMManagedInstanceCore` (not enabled by default)
2. Attach the role as your instance profile
3. Install SSM Agent on the instance

## Prerequisites

The instance needs:
1. **IAM role** with `AmazonSSMManagedInstanceCore` policy attached
2. **Network access** to Systems Manager endpoints (via internet, NAT, or VPC endpoints)
3. **SSM Agent** installed and running

### IAM Role

```bash
# Create IAM role for EC2 with SSM permissions
aws iam create-role \
    --role-name SSMRole \
    --assume-role-policy-document '{
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Principal": {"Service": "ec2.amazonaws.com"},
            "Action": "sts:AssumeRole"
        }]
    }'

# Attach the SSM managed policy
aws iam attach-role-policy \
    --role-name SSMRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Create instance profile
aws iam create-instance-profile --instance-profile-name SSMProfile
aws iam add-role-to-instance-profile \
    --instance-profile-name SSMProfile \
    --role-name SSMRole

# Attach to existing instance
aws ec2 associate-iam-instance-profile \
    --instance-id i-0123456789abcdef0 \
    --iam-instance-profile Name=SSMProfile
```

## Install on RHEL / CentOS / Rocky / AlmaLinux

### RHEL 7 / CentOS 7

```bash
sudo yum install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm

sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
sudo systemctl status amazon-ssm-agent
```

### RHEL 8 / 9 / 10 / Rocky / AlmaLinux

```bash
sudo dnf install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm

sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
sudo systemctl status amazon-ssm-agent
```

> **Note:** RHEL 8/9/10 and CentOS 8+ require Python 3 for SSM Agent. If not present, install it first: `sudo dnf install -y python3`

### ARM64 (Graviton instances)

```bash
# RHEL / CentOS / Rocky (ARM64)
sudo dnf install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_arm64/amazon-ssm-agent.rpm

sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

## Install on Ubuntu / Debian

### Ubuntu 16.04+ (Snap — Usually Pre-Installed)

```bash
# Check if already installed
sudo snap list amazon-ssm-agent

# If not installed
sudo snap install amazon-ssm-agent --classic

# Start/enable
sudo snap start amazon-ssm-agent

# Check status
sudo snap services amazon-ssm-agent
```

### Ubuntu (APT / systemd — Alternative to Snap)

```bash
# Download and install
wget https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/debian_amd64/amazon-ssm-agent.deb
sudo dpkg -i amazon-ssm-agent.deb

# Enable and start
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
sudo systemctl status amazon-ssm-agent

# Clean up
rm amazon-ssm-agent.deb
```

### Ubuntu ARM64 (Graviton)

```bash
wget https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/debian_arm64/amazon-ssm-agent.deb
sudo dpkg -i amazon-ssm-agent.deb

sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

### Debian 9+

```bash
wget https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/debian_amd64/amazon-ssm-agent.deb
sudo dpkg -i amazon-ssm-agent.deb

sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
sudo systemctl status amazon-ssm-agent
```

### SUSE Linux Enterprise Server (SLES) 12 / 15

```bash
wget https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm
sudo rpm --install amazon-ssm-agent.rpm

sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

### Ubuntu 14.04 (Upstart)

```bash
mkdir /tmp/ssm && cd /tmp/ssm
wget https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/debian_amd64/amazon-ssm-agent.deb
sudo dpkg -i amazon-ssm-agent.deb
sudo start amazon-ssm-agent
```

## Verify Installation

```bash
# Check service status
sudo systemctl status amazon-ssm-agent

# Check agent version
amazon-ssm-agent --version

# View agent log
less /var/log/amazon/ssm/amazon-ssm-agent.log

# Check if registered with Systems Manager
aws ssm describe-instance-information \
    --filters "Key=InstanceIds,Values=i-0123456789abcdef0" \
    --query 'InstanceInformationList[*].[InstanceId,PingStatus,AgentVersion]' \
    --output table

# List all managed instances
aws ssm describe-instance-information \
    --query 'InstanceInformationList[*].[InstanceId,PingStatus,PlatformType,AgentVersion]' \
    --output table
```

## Use Session Manager

Once SSM Agent is running and the IAM role is attached:

```bash
# Start a session (no SSH key needed, no open ports)
aws ssm start-session --target i-0123456789abcdef0

# Start session with a named profile
aws --profile production ssm start-session --target i-0123456789abcdef0

# Once connected, switch to ec2-user (SSM starts as ssm-user by default)
sudo su - ec2-user
```

> **Note:** SSM Session Manager does not require security groups allowing inbound SSH, nor does the instance need a direct routable network path from your machine. All traffic goes through the SSM service over HTTPS (443).

```bash
# Port forwarding through SSM
aws ssm start-session \
    --target i-0123456789abcdef0 \
    --document-name AWS-StartPortForwardingSession \
    --parameters '{"portNumber":["8080"],"localPortNumber":["8080"]}'

# Port forward to a remote host through the instance
aws ssm start-session \
    --target i-0123456789abcdef0 \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters '{"host":["rds-endpoint.region.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["5432"]}'

# Run a command on the instance
aws ssm send-command \
    --instance-ids i-0123456789abcdef0 \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=["uptime","df -h"]'

# Run a script from a public GitHub repository
aws ssm send-command \
    --instance-ids i-0123456789abcdef0 \
    --document-name "AWS-RunRemoteScript" \
    --parameters '{
        "sourceType":["GitHub"],
        "sourceInfo":["{\"owner\":\"my-org\",\"repository\":\"my-repo\",\"path\":\"scripts/setup.sh\"}"],
        "commandLine":["bash setup.sh"]
    }'

# Run a script from a private GitHub repository
# First, store your GitHub token in Parameter Store:
aws ssm put-parameter --name "github-token" --value "ghp_xxxx" --type SecureString

# Then reference it with tokenInfo:
aws ssm send-command \
    --instance-ids i-0123456789abcdef0 \
    --document-name "AWS-RunRemoteScript" \
    --parameters '{
        "sourceType":["GitHub"],
        "sourceInfo":["{\"owner\":\"my-org\",\"repository\":\"private-repo\",\"path\":\"scripts/\",\"tokenInfo\":\"{{ssm-secure:github-token}}\"}"],
        "commandLine":["bash deploy.sh"]
    }'

# Run an Ansible playbook from GitHub
aws ssm send-command \
    --instance-ids i-0123456789abcdef0 \
    --document-name "AWS-RunRemoteScript" \
    --parameters '{
        "sourceType":["GitHub"],
        "sourceInfo":["{\"owner\":\"my-org\",\"repository\":\"ansible-repo\",\"path\":\"playbooks/\"}"],
        "commandLine":["ansible-playbook -i \"localhost,\" -c local site.yml"]
    }'

# Check command output
aws ssm get-command-invocation \
    --command-id "command-id-from-above" \
    --instance-id i-0123456789abcdef0
```

## Update SSM Agent

```bash
# Check current version
amazon-ssm-agent --version

# Update on RHEL/CentOS/Rocky
sudo yum update -y amazon-ssm-agent
# or download latest:
sudo yum install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm

# Update on Ubuntu (snap)
sudo snap refresh amazon-ssm-agent

# Update on Ubuntu/Debian (deb)
wget https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/debian_amd64/amazon-ssm-agent.deb
sudo dpkg -i amazon-ssm-agent.deb

# Update via Systems Manager (all instances at once)
aws ssm send-command \
    --targets "Key=tag:Environment,Values=production" \
    --document-name "AWS-UpdateSSMAgent"
```

## Uninstall

```bash
# RHEL/CentOS/Rocky
sudo systemctl stop amazon-ssm-agent
sudo yum remove -y amazon-ssm-agent

# Ubuntu (snap)
sudo snap stop amazon-ssm-agent
sudo snap remove amazon-ssm-agent

# Ubuntu/Debian (deb)
sudo systemctl stop amazon-ssm-agent
sudo dpkg -r amazon-ssm-agent
```

## VPC Endpoints (Private Subnets)

For instances without internet access, create VPC endpoints for SSM:

```bash
# Required endpoints (Interface type)
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-0123456789abcdef0 \
    --service-name com.amazonaws.us-east-1.ssm \
    --vpc-endpoint-type Interface \
    --subnet-ids subnet-0123456789abcdef0 \
    --security-group-ids sg-0123456789abcdef0

aws ec2 create-vpc-endpoint \
    --vpc-id vpc-0123456789abcdef0 \
    --service-name com.amazonaws.us-east-1.ssmmessages \
    --vpc-endpoint-type Interface \
    --subnet-ids subnet-0123456789abcdef0 \
    --security-group-ids sg-0123456789abcdef0

aws ec2 create-vpc-endpoint \
    --vpc-id vpc-0123456789abcdef0 \
    --service-name com.amazonaws.us-east-1.ec2messages \
    --vpc-endpoint-type Interface \
    --subnet-ids subnet-0123456789abcdef0 \
    --security-group-ids sg-0123456789abcdef0

# Optional: S3 endpoint (Gateway type, for downloading packages)
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-0123456789abcdef0 \
    --service-name com.amazonaws.us-east-1.s3 \
    --vpc-endpoint-type Gateway \
    --route-table-ids rtb-0123456789abcdef0
```

Required endpoints:
| Endpoint | Purpose |
|----------|---------|
| `com.amazonaws.<region>.ssm` | SSM API calls |
| `com.amazonaws.<region>.ssmmessages` | Session Manager connections |
| `com.amazonaws.<region>.ec2messages` | Run Command messages |
| `com.amazonaws.<region>.s3` (optional) | Download agent updates, patches |

## Install via User Data (At Launch)

### RHEL 7 / Amazon Linux 2 (x86_64)

```bash
#!/bin/bash
cd /tmp
sudo yum install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

### RHEL 8/9/10 / AL2023 (x86_64)

```bash
#!/bin/bash
cd /tmp
sudo dnf install -y python3
sudo dnf install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

### ARM64 (Graviton) — RHEL / Amazon Linux

```bash
#!/bin/bash
cd /tmp
sudo dnf install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_arm64/amazon-ssm-agent.rpm
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

### Ubuntu

```bash
#!/bin/bash
snap install amazon-ssm-agent --classic
snap start amazon-ssm-agent
```

### Ubuntu / Debian (deb method)

```bash
#!/bin/bash
mkdir /tmp/ssm && cd /tmp/ssm
wget https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/debian_amd64/amazon-ssm-agent.deb
sudo dpkg -i amazon-ssm-agent.deb
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

> **Tip:** Enable automatic updates for SSM Agent by running the `AWS-UpdateSSMAgent` document on a schedule via State Manager, so the agent stays current without manual intervention.

## Hybrid Activation (On-Premises or Other Clouds)

Register non-EC2 machines (on-premises, other clouds) with Systems Manager:

```bash
# Create a hybrid activation
aws ssm create-activation \
    --default-instance-name "on-prem-server" \
    --iam-role "SSMServiceRole" \
    --registration-limit 10

# Output: ActivationId and ActivationCode

# On the target machine (after installing agent):
sudo amazon-ssm-agent -register \
    -code "activation-code" \
    -id "activation-id" \
    -region "us-east-1"

sudo systemctl restart amazon-ssm-agent
```

## Troubleshooting

### Agent Not Showing as "Online"

```bash
# Check agent is running
sudo systemctl status amazon-ssm-agent

# Check agent logs
sudo cat /var/log/amazon/ssm/amazon-ssm-agent.log | tail -50
sudo cat /var/log/amazon/ssm/errors.log | tail -20

# Check IAM role is attached
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/
# Should return the role name

# Check connectivity to SSM endpoints
curl -s https://ssm.us-east-1.amazonaws.com
# Should return XML (not timeout)

# Check region is correct
cat /etc/amazon/ssm/seelog.xml 2>/dev/null
cat /etc/amazon/ssm/amazon-ssm-agent.json 2>/dev/null
```

### "AccessDeniedException" or "Not Authorized"

- Verify the IAM role has `AmazonSSMManagedInstanceCore` policy
- Check the instance profile is attached: `aws ec2 describe-instances --instance-ids i-xxx --query 'Reservations[0].Instances[0].IamInstanceProfile'`
- If using VPC endpoints, ensure security groups allow HTTPS (443) inbound from the instance

### Agent Crashes or Restarts

```bash
# Check for errors
journalctl -u amazon-ssm-agent --since "1 hour ago"

# Reinstall
sudo systemctl stop amazon-ssm-agent
sudo yum remove -y amazon-ssm-agent
sudo yum install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm
sudo systemctl start amazon-ssm-agent
```

### Session Manager Plugin (Local Machine)

To use `aws ssm start-session` from your local machine:

```bash
# macOS (Homebrew)
brew install --cask session-manager-plugin

# macOS (Manual)
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/mac/sessionmanager-bundle.zip" -o "sessionmanager-bundle.zip"
unzip sessionmanager-bundle.zip
sudo ./sessionmanager-bundle/install -i /usr/local/sessionmanagerplugin -b /usr/local/bin/session-manager-plugin
rm -rf sessionmanager-bundle sessionmanager-bundle.zip

# Linux (64-bit)
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o "session-manager-plugin.deb"
sudo dpkg -i session-manager-plugin.deb

# Verify
session-manager-plugin --version
```

## SSH Over SSM (No Open Ports)

### Bash Function: Connect by Instance Name

```bash
# Add to ~/.bashrc or ~/.zshrc
aws-ssh() {
    local instance_name=${1}
    local instance_id=$(aws ec2 describe-instances \
        --filter "Name=tag:Name,Values=${instance_name}" \
        --query "Reservations[].Instances[?State.Name == 'running'].InstanceId[]" \
        --output text)
    aws ssm start-session --target ${instance_id}
}

# With named profile
aws-ssh() {
    local instance_name=${1}
    local profile=${2:-default}
    local instance_id=$(aws --profile ${profile} ec2 describe-instances \
        --filter "Name=tag:Name,Values=${instance_name}" \
        --query "Reservations[].Instances[?State.Name == 'running'].InstanceId[]" \
        --output text)
    aws --profile ${profile} ssm start-session --target ${instance_id}
}

# Usage:
aws-ssh my-server-name
aws-ssh my-server-name production
```

### SSH Config (ProxyCommand)

Use SSM as the transport for SSH (for `scp`, `rsync`, ProxyCommand):

```bash
# ~/.ssh/config
Host i-* mi-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
    User ec2-user
    IdentityFile ~/.ssh/my-key.pem
```

Then connect normally:

```bash
ssh i-0123456789abcdef0
scp file.txt i-0123456789abcdef0:/tmp/
```

## Download URLs Reference

| Platform | Architecture | URL |
|----------|-------------|-----|
| RHEL/CentOS/Rocky (RPM) | x86_64 | `https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm` |
| RHEL/CentOS/Rocky (RPM) | ARM64 | `https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_arm64/amazon-ssm-agent.rpm` |
| Ubuntu/Debian (DEB) | x86_64 | `https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/debian_amd64/amazon-ssm-agent.deb` |
| Ubuntu/Debian (DEB) | ARM64 | `https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/debian_arm64/amazon-ssm-agent.deb` |
