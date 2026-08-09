# Installing AWS CLI

The AWS CLI (version 2) is the official command-line tool for interacting with AWS services. This guide covers installation on RHEL, Ubuntu, macOS, and configuration.

## Install on RHEL / CentOS / Rocky / AlmaLinux

### Method 1: Official Installer (Recommended)

```bash
# Download
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Install unzip if needed
sudo yum install -y unzip    # RHEL 7
sudo dnf install -y unzip    # RHEL 8+

# Unzip and install
unzip awscliv2.zip
sudo ./aws/install

# Or with explicit install paths
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin

# Verify
aws --version
# aws-cli/2.x.x Python/3.x.x Linux/x86_64

# Clean up
rm -rf aws awscliv2.zip
```

### ARM64 (Graviton)

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
rm -rf aws awscliv2.zip
```

### Update Existing Installation

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install --update
aws --version
rm -rf aws awscliv2.zip
```

### Custom Install Location

```bash
# Install to custom path (useful if no root access)
./aws/install -i ~/aws-cli -b ~/bin

# Add to PATH
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## Install on Ubuntu / Debian

### Method 1: Official Installer (Recommended)

```bash
# Download
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Install unzip if needed
sudo apt install -y unzip

# Unzip and install
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version

# Clean up
rm -rf aws awscliv2.zip
```

### ARM64 (Graviton)

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
rm -rf aws awscliv2.zip
```

### Method 2: Snap

```bash
sudo snap install aws-cli --classic
aws --version
```

### Method 3: pip (Not Recommended for v2)

```bash
# AWS CLI v1 only (v2 is not available via pip)
pip install awscli
```

> **Note:** pip installs CLI v1 which is end-of-support. Use the official installer for v2.

## Install on macOS

### Homebrew

```bash
brew install awscli
aws --version
```

### Official Installer (GUI)

```bash
# Download the pkg
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"

# Install
sudo installer -pkg AWSCLIV2.pkg -target /

# Verify
aws --version

# Clean up
rm AWSCLIV2.pkg
```

### macOS (Command Line, No GUI)

```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg ./AWSCLIV2.pkg -target /
aws --version
```

## Configure

### Initial Configuration

```bash
aws configure
# AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name [None]: us-east-1
# Default output format [None]: json
```

### Named Profiles

```bash
# Configure a named profile
aws configure --profile production

# Use a profile
aws ec2 describe-instances --profile production

# Set default profile for session
export AWS_PROFILE=production
```

### Configuration Files

```bash
# Credentials file
cat ~/.aws/credentials
# [default]
# aws_access_key_id = AKIAIOSFODNN7EXAMPLE
# aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# Config file
cat ~/.aws/config
# [default]
# region = us-east-1
# output = json
#
# [profile production]
# region = eu-west-1
# output = table
```

### Environment Variables

```bash
# Override config with environment variables
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_DEFAULT_REGION=us-east-1
export AWS_DEFAULT_OUTPUT=json

# Use a specific profile
export AWS_PROFILE=production

# Disable pager
export AWS_PAGER=""
```

### Credential Precedence (Highest to Lowest)

1. Command-line options (`--region`, `--profile`)
2. Environment variables (`AWS_ACCESS_KEY_ID`, etc.)
3. CLI credential file (`~/.aws/credentials`)
4. CLI config file (`~/.aws/config`)
5. Instance profile (IAM role on EC2)

## Verify Installation

```bash
# Version
aws --version

# Check identity
aws sts get-caller-identity

# Check account number only
aws sts get-caller-identity --query Account --output text

# List S3 buckets (quick connectivity test)
aws s3 ls

# Check configured region
aws configure get region

# Check configured profile
aws configure list
```

## Auto-Completion

### Bash

```bash
# Add to ~/.bashrc
echo 'complete -C aws_completer aws' >> ~/.bashrc
source ~/.bashrc
```

### Zsh

```bash
# Add to ~/.zshrc
echo 'autoload bashcompinit && bashcompinit' >> ~/.zshrc
echo 'complete -C aws_completer aws' >> ~/.zshrc
source ~/.zshrc
```

### Verify

```bash
# Type 'aws s3' then press Tab — should show sub-commands
aws s3 <Tab>
```

## Uninstall

### Linux (Official Installer)

```bash
# Find install location
which aws
ls -la $(which aws)

# Default locations
sudo rm -rf /usr/local/aws-cli
sudo rm /usr/local/bin/aws
sudo rm /usr/local/bin/aws_completer
```

### macOS (Homebrew)

```bash
brew uninstall awscli
```

### macOS (Official pkg)

```bash
sudo rm -rf /usr/local/aws-cli
sudo rm /usr/local/bin/aws
sudo rm /usr/local/bin/aws_completer
```

### Ubuntu (Snap)

```bash
sudo snap remove aws-cli
```

## Troubleshooting

### "aws: command not found"

```bash
# Check if installed
which aws
ls /usr/local/bin/aws

# If installed to custom location, check PATH
echo $PATH

# Add to PATH
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### "Unable to locate credentials"

```bash
# Check if credentials exist
cat ~/.aws/credentials

# Check current identity
aws sts get-caller-identity

# Re-configure
aws configure

# If on EC2, check instance profile
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

### "An error occurred (ExpiredToken)"

```bash
# Session token expired — re-authenticate
aws configure

# If using temporary credentials (STS)
aws sts get-session-token
```

### SSL Certificate Errors

```bash
# Skip SSL verification (testing only)
aws s3 ls --no-verify-ssl

# Or set CA bundle
export AWS_CA_BUNDLE=/path/to/ca-bundle.crt
```

### Proxy Configuration

```bash
export HTTP_PROXY=http://proxy.example.com:8080
export HTTPS_PROXY=http://proxy.example.com:8080
export NO_PROXY=169.254.169.254    # EC2 metadata endpoint
```

## Install via User Data (EC2 Bootstrap)

```bash
#!/bin/bash
# Install AWS CLI v2 at instance launch
cd /tmp
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install
rm -rf aws awscliv2.zip
```

## Docker

```bash
# Run AWS CLI from Docker (no local install)
docker run --rm -it amazon/aws-cli --version

# With credentials
docker run --rm -it \
    -v ~/.aws:/root/.aws \
    amazon/aws-cli s3 ls

# Alias for convenience
alias aws='docker run --rm -it -v ~/.aws:/root/.aws -v $(pwd):/aws amazon/aws-cli'
```

## Version Check and Upgrade Path

```bash
# Check current version
aws --version

# Check for v1 (legacy)
# aws-cli/1.x.x Python/... → upgrade to v2

# Upgrade v2 (Linux)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install --update
rm -rf aws awscliv2.zip

# Upgrade v2 (macOS Homebrew)
brew upgrade awscli
```
