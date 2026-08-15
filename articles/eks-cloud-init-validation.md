# Validating cloud-init Completion on EKS Nodes

When an EKS node launches, cloud-init runs the userdata (bootstrap script) that configures kubelet and joins the cluster. If cloud-init fails or completes partially, the node won't function correctly. This guide covers how to verify cloud-init ran successfully and troubleshoot failures.

## How cloud-init Works on EKS Nodes

```
EC2 Instance Launches
    │
    ▼
cloud-init starts (systemd early boot)
    │
    ├── Retrieves userdata from IMDS (169.254.169.254)
    ├── Runs through boot stages:
    │   1. local     — network-independent setup
    │   2. network   — after networking is up
    │   3. config    — module configuration
    │   4. final     — userdata scripts (runcmd, scripts-user)
    │
    ▼
/etc/eks/bootstrap.sh runs (in the "final" stage)
    │
    ├── Configures kubelet (endpoint, CA, cluster name)
    ├── Starts kubelet service
    └── Node joins cluster
```

On EKS-optimized AMIs, the critical action happens in the `final` stage — that's when `/etc/eks/bootstrap.sh` executes.

## Quick Status Check

### From Inside the Node (SSH/SSM)

```sh
# One-liner: is cloud-init done?
cloud-init status
# Expected: "status: done"
# Bad: "status: error" or "status: running"

# Detailed status
cloud-init status --long

# Example output (success):
# status: done
# time: Thu, 15 Jan 2024 10:30:42 +0000
# detail: DataSourceEc2Local

# Example output (failure):
# status: error
# time: Thu, 15 Jan 2024 10:30:42 +0000
# detail: DataSourceEc2Local
# errors:
#   - "Failed to run module scripts-user (module failed with exit code 1)"
```

### Wait for cloud-init (In Userdata Scripts)

If you have custom userdata that depends on cloud-init finishing:

```sh
# Block until cloud-init completes all stages
cloud-init status --wait

# Returns exit code:
# 0 = success
# 1 = error
# 2 = recoverable error
```

### From Outside (SSM RunCommand)

```sh
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["cloud-init status --long"]'

# Get result
aws ssm get-command-invocation \
  --command-id <cmd-id> \
  --instance-id <instance-id> \
  --query "StandardOutputContent" --output text
```

## Checking cloud-init Logs

### Key Log Files

| File | Contains |
|------|----------|
| `/var/log/cloud-init.log` | Detailed debug log (all stages) |
| `/var/log/cloud-init-output.log` | stdout/stderr of userdata scripts |
| `/run/cloud-init/result.json` | Final status (success/error) |
| `/run/cloud-init/status.json` | Per-stage status |
| `/var/lib/cloud/instance/user-data.txt` | The actual userdata that ran |
| `/var/lib/cloud/instance/scripts/` | Extracted scripts |

### Check result.json

```sh
cat /run/cloud-init/result.json
```

```json
// Success:
{
  "v1": {
    "errors": []
  }
}

// Failure:
{
  "v1": {
    "errors": [
      "Failed running /var/lib/cloud/instance/scripts/part-001"
    ]
  }
}
```

### Check Per-Stage Status

```sh
cat /run/cloud-init/status.json | jq
```

```json
{
  "v1": {
    "init": { "start": 1705312200.0, "finished": 1705312201.5, "errors": [] },
    "init-local": { "start": 1705312198.0, "finished": 1705312199.0, "errors": [] },
    "modules-config": { "start": 1705312202.0, "finished": 1705312203.0, "errors": [] },
    "modules-final": { "start": 1705312204.0, "finished": null, "errors": ["exit code 1"] }
  }
}
```

If `modules-final` has errors or `finished: null`, the bootstrap script failed.

### View Userdata Output

```sh
# What the bootstrap script printed
cat /var/log/cloud-init-output.log

# Search for errors
grep -i "error\|fail\|fatal" /var/log/cloud-init-output.log

# Search for the bootstrap script specifically
grep -A 20 "bootstrap.sh" /var/log/cloud-init-output.log
```

### View the Actual Userdata

```sh
# What userdata was provided to the instance
cat /var/lib/cloud/instance/user-data.txt

# Or from IMDS
curl -s http://169.254.169.254/latest/user-data
```

## Validating EKS Bootstrap Specifically

### Check if bootstrap.sh Ran

```sh
# Was bootstrap.sh executed?
grep "bootstrap.sh" /var/log/cloud-init-output.log

# Did kubelet start?
systemctl status kubelet

# Is kubelet configured correctly?
cat /var/lib/kubelet/kubeconfig | grep server
```

### Verify kubelet Configuration

```sh
# API server endpoint
grep server /var/lib/kubelet/kubeconfig

# Cluster CA
grep certificate-authority-data /var/lib/kubelet/kubeconfig

# kubelet extra args (labels, taints, max-pods)
cat /etc/kubernetes/kubelet/kubelet-config.json | jq '.clusterDNS, .maxPods'

# Or on older AMIs
ps aux | grep kubelet | grep -o '\-\-[a-z-]*'
```

### Check Bootstrap Exit Code

```sh
# cloud-init logs the exit code of each script
grep "exit code" /var/log/cloud-init.log | tail -5

# Or check the specific script result
grep "scripts-user" /var/log/cloud-init.log | grep -i "result\|exit\|fail"
```

## Common cloud-init Failures on EKS

### Bootstrap Script Not Found

```
/bin/bash: /etc/eks/bootstrap.sh: No such file or directory
```

**Cause**: Wrong AMI (not EKS-optimized)

**Fix**: Use an EKS-optimized AMI:
```sh
aws ssm get-parameter --name /aws/service/eks/optimized-ami/1.30/amazon-linux-2023/x86_64/standard/recommended/image_id \
  --query "Parameter.Value" --output text
```

### API Server Unreachable

```
Unable to connect to the server: dial tcp <ip>:443: i/o timeout
```

**Cause**: Network issue (SG, route table, DNS)

**Verify**:
```sh
# From the node
ENDPOINT=$(grep server /var/lib/kubelet/kubeconfig | awk '{print $2}' | sed 's|https://||')
timeout 5 bash -c "echo > /dev/tcp/$ENDPOINT/443" && echo "Reachable" || echo "BLOCKED"
```

### Wrong Cluster Name or CA

```
certificate signed by unknown authority
```

**Cause**: Mismatched `--b64-cluster-ca` or `--apiserver-endpoint` in userdata

**Verify**:
```sh
# Compare what's in kubeconfig vs what the cluster actually uses
echo "In kubeconfig:"
grep certificate-authority-data /var/lib/kubelet/kubeconfig | awk '{print $2}' | head -c 40
echo ""
echo "From cluster API:"
aws eks describe-cluster --name <cluster> --query "cluster.certificateAuthority.data" --output text | head -c 40
```

### Userdata Syntax Error

```
/var/lib/cloud/instance/scripts/part-001: line 5: syntax error near unexpected token
```

**Cause**: Shell syntax error in the userdata script

**Fix**: Check the raw userdata:
```sh
cat /var/lib/cloud/instance/user-data.txt
# Look for encoding issues, missing quotes, or broken heredocs
```

### Timeout Waiting for cloud-init

If cloud-init hangs (never completes):

```sh
# Check what stage it's stuck on
cloud-init status --long

# Check if it's waiting for something
journalctl -u cloud-init --no-pager | tail -30

# Common hang: waiting for network (metadata service unreachable)
curl -s -o /dev/null -w "%{http_code}" http://169.254.169.254/latest/meta-data/
```

### Permission Denied

```
/etc/eks/bootstrap.sh: Permission denied
```

**Cause**: Userdata script not marked executable, or SELinux blocking

**Fix**: Ensure shebang line is present:
```sh
#!/bin/bash
/etc/eks/bootstrap.sh my-cluster
```

### Proxy Misconfiguration

```
Unable to connect to the server: proxyconnect tcp: dial tcp <proxy-ip>:3128: i/o timeout
```

**Cause**: Node is configured with HTTP_PROXY/HTTPS_PROXY but the proxy can't reach the EKS API endpoint, or the `NO_PROXY` list is missing critical entries.

**Fix**: Ensure `NO_PROXY` includes the Kubernetes service CIDR, pod CIDR, and the EKS API endpoint:

```sh
# In userdata, before bootstrap.sh:
export HTTP_PROXY=http://proxy.internal:3128
export HTTPS_PROXY=http://proxy.internal:3128
export NO_PROXY=169.254.169.254,10.0.0.0/8,172.20.0.0/16,.eks.amazonaws.com,.amazonaws.com

/etc/eks/bootstrap.sh my-cluster
```

## Monitoring cloud-init Across Nodes

### SSM RunCommand for Fleet Checks

```sh
# Check cloud-init status on all nodes in a node group
INSTANCE_IDS=$(aws ec2 describe-instances \
  --filters "Name=tag:eks:nodegroup-name,Values=<ng>" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].InstanceId" --output text)

aws ssm send-command \
  --instance-ids $INSTANCE_IDS \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["cloud-init status --long && echo --- && systemctl is-active kubelet"]'
```

### CloudWatch Logs for cloud-init

Configure cloud-init logs to ship to CloudWatch:

```yaml
# In userdata, before bootstrap.sh:
#!/bin/bash

# Install and configure CloudWatch agent for cloud-init logs
cat > /opt/aws/amazon-cloudwatch-agent/etc/config.json << 'EOF'
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/cloud-init-output.log",
            "log_group_name": "/eks/nodes/cloud-init",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
EOF
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a start

# Then run bootstrap
/etc/eks/bootstrap.sh my-cluster
```

### EC2 Console: Get System Log

Without SSH or SSM, you can see early boot output:

```sh
# Get console output (includes cloud-init boot messages)
aws ec2 get-console-output --instance-id <id> --output text | grep -i "cloud-init\|bootstrap\|kubelet"
```

## Pre-Validating Userdata Before Launch

### Dry Run with cloud-init

Test userdata syntax locally before launching instances:

```sh
# Validate cloud-init syntax (on any Linux machine with cloud-init installed)
cloud-init devel schema --config-type user-data --doc userdata.txt

# Or test MIME multipart encoding
cloud-init devel make-mime --attach userdata.sh:text/x-shellscript > combined.txt
```

### Shell Syntax Check

```sh
# Check bash syntax without executing
bash -n userdata.sh

# Shellcheck for common issues
shellcheck userdata.sh
```

### Validate Base64 Encoding

Launch templates store userdata as base64. Verify it decodes correctly:

```sh
# Encode
base64 userdata.sh > userdata-b64.txt

# Verify decode
base64 -d userdata-b64.txt | head -5
# Should show: #!/bin/bash
```

## cloud-init Stages and EKS Timing

| Stage | When | What Runs | EKS Relevance |
|-------|------|-----------|---------------|
| `init-local` | Before network | Seed network config | Rarely relevant |
| `init` | After network | Metadata fetch, SSH keys | IMDS available |
| `modules-config` | After init | Package installs, disk setup | Custom pre-bootstrap |
| `modules-final` | Last | Userdata scripts (runcmd, scripts-user) | **bootstrap.sh runs here** |

If your custom userdata needs to run BEFORE bootstrap.sh, use `bootcmd` (runs in `init`) instead of `runcmd` (runs in `modules-final`).

## Gotchas

- **cloud-init runs only on first boot**: If you stop/start an instance, cloud-init does NOT re-run userdata. Only a newly launched instance runs it.
- **MIME multipart**: If your userdata has both a cloud-config section and a shell script, they must be MIME-encoded. A raw shell script with `#cloud-config` in it will confuse cloud-init.
- **Exit code matters**: If your pre-bootstrap script exits non-zero, cloud-init marks the stage as failed — but the instance keeps running. kubelet may or may not start depending on where the failure occurred.
- **cloud-init timeout**: cloud-init has a 120-second default timeout for metadata retrieval. If IMDS is slow (hop limit issue, network delay), cloud-init can fail before userdata even runs.
- **Bottlerocket doesn't use cloud-init**: Bottlerocket uses a TOML-based settings API. cloud-init commands don't apply.
- **`/var/log/cloud-init-output.log` may be empty on managed nodes**: EKS managed node groups use a minimal bootstrap — the log may only show the bootstrap.sh output, not verbose cloud-init stages.
- **Encoding issues**: If userdata contains non-ASCII characters or Windows line endings (`\r\n`), the shell script may fail with cryptic errors.
