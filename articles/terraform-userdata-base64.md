# Terraform UserData Base64 Encoding and Decoding

How user_data gets encoded, when to use `base64encode()` vs `user_data_base64`, how to decode and inspect userdata from state and running instances, and debugging techniques.

## How UserData Works

AWS EC2 user_data is always stored as base64-encoded content. Terraform handles encoding depending on which argument you use:

| Argument | Terraform Encodes? | You Provide |
|----------|-------------------|-------------|
| `user_data` | Yes (auto-encodes) | Plain text script |
| `user_data_base64` | No (pass-through) | Already base64-encoded content |

```hcl
# Option 1: user_data — Terraform auto-encodes to base64
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"

  user_data = <<-EOF
    #!/bin/bash
    echo "Hello World"
  EOF
}

# Option 2: user_data_base64 — You handle encoding
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"

  user_data_base64 = base64encode(<<-EOF
    #!/bin/bash
    echo "Hello World"
  EOF
  )
}

# Option 3: user_data_base64 with templatefile
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"

  user_data_base64 = base64encode(templatefile("${path.module}/scripts/init.sh.tpl", {
    environment = var.environment
    app_version = var.app_version
  }))
}
```

## When to Use user_data_base64

Use `user_data_base64` instead of `user_data` when:

1. **You want explicit control** over encoding
2. **The content is already base64** (from another source)
3. **Launch templates** — `user_data` in `aws_launch_template` requires base64
4. **Preventing plan diff noise** — `user_data_base64` avoids some hash-related diffs
5. **Multi-part cloud-init** — `cloudinit_config` produces base64 output

```hcl
# Launch template REQUIRES base64 encoding
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = var.ami
  instance_type = var.instance_type

  user_data = base64encode(<<-EOF
    #!/bin/bash
    echo "Launch templates need base64encode()"
  EOF
  )
}

# cloudinit_config produces base64 output
data "cloudinit_config" "app" {
  gzip          = true
  base64_encode = true

  part {
    content_type = "text/x-shellscript"
    content      = file("${path.module}/scripts/setup.sh")
  }
}

resource "aws_instance" "app" {
  ami              = var.ami
  instance_type    = "t3.micro"
  user_data_base64 = data.cloudinit_config.app.rendered
}
```

## Decoding UserData from Terraform State

### From terraform output

```bash
# If you output the user_data
terraform output -raw instance_user_data | base64 -d
```

### From terraform show

```bash
# Show the resource and decode user_data
terraform state show aws_instance.web | grep user_data

# Full JSON extraction and decode
terraform show -json | jq -r '.values.root_module.resources[] | select(.address == "aws_instance.web") | .values.user_data' | base64 -d
```

### From terraform console

```bash
terraform console

# Decode user_data from a resource
> base64decode(aws_instance.web.user_data)

# For launch templates
> base64decode(aws_launch_template.app.user_data)
```

### From State File Directly

```bash
# Extract and decode from terraform.tfstate
jq -r '.resources[] | select(.type == "aws_instance" and .name == "web") | .instances[0].attributes.user_data' terraform.tfstate | base64 -d

# For all instances
jq -r '.resources[] | select(.type == "aws_instance") | .instances[0] | "\(.attributes.tags.Name):\n" + (.attributes.user_data | @base64d)' terraform.tfstate
```

## Decoding UserData from Running Instances

### From the Instance (IMDS)

```bash
# From inside the instance — Instance Metadata Service
curl -s http://169.254.169.254/latest/user-data

# IMDSv2 (token required)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/user-data
```

Note: The metadata service returns user_data already decoded — no base64 decoding needed.

### From AWS CLI (External)

```bash
# Get user_data for an instance (returns base64)
aws ec2 describe-instance-attribute \
  --instance-id i-0abc123def456789 \
  --attribute userData \
  --query 'UserData.Value' \
  --output text | base64 -d

# Using instance from Terraform output
aws ec2 describe-instance-attribute \
  --instance-id $(terraform output -raw instance_id) \
  --attribute userData \
  --query 'UserData.Value' \
  --output text | base64 -d

# For launch template versions
aws ec2 describe-launch-template-versions \
  --launch-template-id lt-abc123 \
  --versions '$Latest' \
  --query 'LaunchTemplateVersions[0].LaunchTemplateData.UserData' \
  --output text | base64 -d
```

### Gzipped UserData (cloudinit_config with gzip=true)

```bash
# If user_data was gzipped before base64 encoding
aws ec2 describe-instance-attribute \
  --instance-id i-0abc123def456789 \
  --attribute userData \
  --query 'UserData.Value' \
  --output text | base64 -d | gunzip

# Or using zcat
aws ec2 describe-instance-attribute \
  --instance-id i-0abc123def456789 \
  --attribute userData \
  --query 'UserData.Value' \
  --output text | base64 -d | zcat
```

## Inspecting UserData Before Apply

### Preview with terraform console

```bash
terraform console

# Preview heredoc content
> templatefile("scripts/init.sh.tpl", { environment = "prod", app_version = "2.1.0" })

# Preview base64-encoded result
> base64encode(templatefile("scripts/init.sh.tpl", { environment = "prod" }))

# Decode to verify
> base64decode(base64encode("test content"))
```

### Preview with terraform plan

```bash
# Plan shows user_data as a hash (SHA256)
terraform plan

# To see the actual content, use:
terraform plan -out=tfplan
terraform show -json tfplan | jq -r '.resource_changes[] | select(.address == "aws_instance.web") | .change.after.user_data' | base64 -d
```

### Render to File for Testing

```bash
# Render the template to a local file
terraform console <<< 'templatefile("scripts/init.sh.tpl", {environment="dev", app="myapp"})' > /tmp/rendered-userdata.sh

# Check syntax
bash -n /tmp/rendered-userdata.sh

# Lint with shellcheck
shellcheck /tmp/rendered-userdata.sh
```

## Comparing UserData (State vs Running Instance)

```bash
#!/bin/bash
# compare-userdata.sh — Compare Terraform state with actual instance

INSTANCE_ID=$(terraform output -raw instance_id)

# Get userdata from Terraform state
terraform show -json | jq -r '.values.root_module.resources[] | select(.type == "aws_instance") | .values.user_data' | base64 -d > /tmp/state-userdata.sh

# Get userdata from running instance
aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute userData \
  --query 'UserData.Value' \
  --output text | base64 -d > /tmp/instance-userdata.sh

# Compare
diff /tmp/state-userdata.sh /tmp/instance-userdata.sh

if [ $? -eq 0 ]; then
    echo "✅ UserData matches between state and instance"
else
    echo "⚠️  UserData differs — instance may have been modified"
fi
```

## UserData Hash and Drift

Terraform tracks `user_data` changes via SHA256 hash. If the content changes, Terraform proposes a replacement (user_data change forces new instance by default).

### Prevent Replacement on UserData Change

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"
  user_data     = templatefile("scripts/init.sh.tpl", { version = var.app_version })

  lifecycle {
    ignore_changes = [user_data]
  }
}
# Instance won't be replaced when user_data changes
# Useful when: userdata runs only at first boot anyway
```

### Force Replacement on UserData Change

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"
  user_data     = templatefile("scripts/init.sh.tpl", { version = var.app_version })

  # Default behavior — instance is replaced when user_data changes
  # user_data_replace_on_change = true  (default since AWS provider 4.x)
}
```

### user_data_replace_on_change (AWS Provider 4.x+)

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t3.micro"
  user_data     = file("scripts/init.sh")

  # Explicitly control whether user_data changes force replacement
  user_data_replace_on_change = true   # Replace instance (default)
  # user_data_replace_on_change = false  # Update in-place (won't re-run)
}
```

## Multipart MIME UserData

For complex scenarios combining cloud-config YAML and shell scripts:

```hcl
data "cloudinit_config" "web" {
  gzip          = true
  base64_encode = true

  part {
    content_type = "text/cloud-config"
    content = yamlencode({
      package_update = true
      packages       = ["nginx", "curl", "jq"]
    })
  }

  part {
    content_type = "text/x-shellscript"
    content = templatefile("${path.module}/scripts/setup.sh.tpl", {
      environment = var.environment
      app_name    = var.app_name
    })
  }

  part {
    content_type = "text/x-shellscript"
    content      = file("${path.module}/scripts/monitoring.sh")
  }
}

resource "aws_instance" "web" {
  ami              = var.ami
  instance_type    = "t3.micro"
  user_data_base64 = data.cloudinit_config.web.rendered
}
```

### Decode Multipart UserData

```bash
# Decode base64 + gunzip multipart content
aws ec2 describe-instance-attribute \
  --instance-id i-0abc123def456789 \
  --attribute userData \
  --query 'UserData.Value' \
  --output text | base64 -d | gunzip

# If gzip=false, just decode base64
aws ec2 describe-instance-attribute \
  --instance-id i-0abc123def456789 \
  --attribute userData \
  --query 'UserData.Value' \
  --output text | base64 -d

# Split multipart content by boundary
aws ec2 describe-instance-attribute \
  --instance-id i-0abc123def456789 \
  --attribute userData \
  --query 'UserData.Value' \
  --output text | base64 -d | gunzip | csplit - '/^--//' '{*}'
```

## Debugging UserData Failures

### Check Cloud-Init Logs

```bash
# Main cloud-init log (execution details)
cat /var/log/cloud-init-output.log

# Cloud-init status and errors
cat /var/log/cloud-init.log

# Check if cloud-init completed successfully
cloud-init status
cloud-init status --long

# Re-run cloud-init (for testing)
sudo cloud-init clean
sudo cloud-init init
sudo cloud-init modules --mode=config
sudo cloud-init modules --mode=final
```

### Common Failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| Script doesn't run | Missing `#!/bin/bash` shebang | Add shebang as first line |
| Partial execution | Script error halts execution | Add `set -e` or check logs |
| Wrong content | Terraform interpolation ate `$` | Escape with `$$` |
| Empty userdata | base64 encoding issue | Check with `terraform console` |
| Instance loops rebooting | Userdata error + restart-on-fail | Check cloud-init-output.log |
| Gzipped content garbled | Wrong decode method | Use `gunzip` after base64 decode |

### Output UserData for an Existing Output

```hcl
output "instance_user_data_decoded" {
  value     = base64decode(aws_instance.web.user_data)
  sensitive = true
}
```

```bash
# View it
terraform output -raw instance_user_data_decoded
```

## Decoding UserData Diffs from terraform plan

When `terraform plan` shows userdata changes as base64, it's unreadable. These techniques decode the base64 inline.

### One-Liners

```bash
# Decode userdata from plan output (simple)
terraform plan -no-color | grep -E "[\+\-].*user_data" | sed -E 's/.*user_data.*= "(.*)".*/\1/' | head -1 | base64 -d

# Decode and show both old and new versions
terraform plan -no-color | grep -E "[\+\-].*user_data" | sed -E 's/.*user_data.*= "(.*)".*/\1/' | while read -r line; do echo "$line" | base64 -d; echo "---"; done

# Save plan output first, then decode (avoids running plan twice)
terraform plan -no-color > /tmp/plan.txt
grep -E "[\+\-].*user_data" /tmp/plan.txt | sed -E 's/.*user_data.*= "(.*)".*/\1/' | head -1 | base64 -d
```

### Proper Diff Between Old and New UserData

```bash
# Extract old and new, decode, then diff
terraform plan -no-color | grep -E "[\+\-].*user_data" | sed -E 's/.*user_data.*= "(.*)".*/\1/' | \
  awk 'NR==1{print $0 | "base64 -d > /tmp/old_userdata"} NR==2{print $0 | "base64 -d > /tmp/new_userdata"}' && \
  diff -u /tmp/old_userdata /tmp/new_userdata
```

### Using JSON Plan (Most Reliable)

```bash
# Generate JSON plan and extract with jq
terraform plan -out=tfplan
terraform show -json tfplan > plan.json

# Decode the planned userdata
jq -r '.resource_changes[] | select(.change.after.user_data != null) | .change.after.user_data' plan.json | base64 -d

# Show before and after
jq -r '.resource_changes[] | select(.change.before.user_data != null) | .change.before.user_data' plan.json | base64 -d > /tmp/ud_before.sh
jq -r '.resource_changes[] | select(.change.after.user_data != null) | .change.after.user_data' plan.json | base64 -d > /tmp/ud_after.sh
diff -u /tmp/ud_before.sh /tmp/ud_after.sh
```

### Shell Function (Add to ~/.bashrc)

```bash
terraform_userdata_diff() {
    terraform plan -no-color | grep -E "[\+\-].*user_data" | sed -E 's/.*user_data.*= "(.*)".*/\1/' | {
        read -r old_data
        read -r new_data
        if [ -n "$old_data" ] && [ -n "$new_data" ]; then
            echo "$old_data" | base64 -d > /tmp/old_userdata
            echo "$new_data" | base64 -d > /tmp/new_userdata
            echo "=== UserData Diff ==="
            diff -u /tmp/old_userdata /tmp/new_userdata
            rm -f /tmp/old_userdata /tmp/new_userdata
        else
            echo "Could not extract userdata changes from terraform plan"
        fi
    }
}

# Usage: just run
terraform_userdata_diff
```

### Python Decoder (Handles Edge Cases)

```bash
terraform plan -no-color | grep -E "[\+\-].*user_data" | sed -E 's/.*user_data.*= "(.*)".*/\1/' | python3 -c "
import sys, base64
for line in sys.stdin:
    try:
        decoded = base64.b64decode(line.strip()).decode('utf-8')
        print(decoded)
        print('---')
    except Exception as e:
        print(f'Failed to decode: {e}')
"
```

### Tips

- Always use `terraform plan -no-color` to avoid ANSI escape sequences breaking parsing
- Save plan output to a file first if you need to analyze it multiple times
- For large userdata, the JSON plan approach (`terraform show -json`) is the most reliable
- Multiline base64 values in plan output can break simple grep patterns — use the JSON method

### First-AZ Only (Multi-AZ Deployments)

When you have multiple AZs but only need to see changes for the first one:

```bash
# Decode first userdata change only (skip other AZs)
terraform plan -no-color | grep "user_data" | head -1 | \
  sed -E 's/.*"([^"]*)" -> "([^"]*)".*/\1\n\2/' | \
  while read -r line; do echo "$line" | base64 -d; echo "---"; done

# First AZ with labeled diff
terraform plan -no-color | grep "user_data" | head -1 | \
  sed -E 's/.*"([^"]*)" -> "([^"]*)".*/\1|\2/' | {
    IFS='|' read -r old new
    diff -u --label="OLD" --label="NEW" <(echo "$old" | base64 -d) <(echo "$new" | base64 -d)
  }

# Save both versions to files
terraform plan -no-color | grep "user_data" | head -1 | \
  sed -E 's/.*"([^"]*)" -> "([^"]*)".*/\1|\2/' | {
    IFS='|' read -r old new
    echo "$old" | base64 -d > userdata_old.sh
    echo "$new" | base64 -d > userdata_new.sh
    echo "Saved: userdata_old.sh and userdata_new.sh"
  }
```

### Enhanced Shell Function (with diff and export)

```bash
tf_userdata_diff() {
    local mode=${1:-"diff"}  # diff, old, new, export

    local output=$(terraform plan -no-color | grep "user_data" | head -1 | \
      sed -E 's/.*"([^"]*)" -> "([^"]*)".*/\1|\2/')

    if [ -z "$output" ]; then
        echo "No userdata changes found in plan"
        return 1
    fi

    IFS='|' read -r old new <<< "$output"

    case $mode in
        old)  echo "$old" | base64 -d ;;
        new)  echo "$new" | base64 -d ;;
        export)
            echo "$old" | base64 -d > userdata_old.sh
            echo "$new" | base64 -d > userdata_new.sh
            chmod +x userdata_old.sh userdata_new.sh
            echo "Exported: userdata_old.sh, userdata_new.sh"
            diff -u userdata_old.sh userdata_new.sh
            ;;
        *)
            echo "=== OLD ==="; echo "$old" | base64 -d; echo ""
            echo "=== NEW ==="; echo "$new" | base64 -d; echo ""
            echo "=== DIFF ==="
            diff -u --label="OLD" --label="NEW" \
              <(echo "$old" | base64 -d) <(echo "$new" | base64 -d)
            ;;
    esac
}

# Usage:
# tf_userdata_diff          # Show old, new, and diff
# tf_userdata_diff old      # Show only old decoded content
# tf_userdata_diff new      # Show only new decoded content
# tf_userdata_diff export   # Save to files and show diff
```

### Plan Format Notes

Different Terraform versions format inline changes differently. Adjust the sed regex if needed:

```bash
# Standard format: "old_base64" -> "new_base64"
sed -E 's/.*"([^"]*)" -> "([^"]*)".*/\1|\2/'

# Some versions use =>
sed -E 's/.*"([^"]*)" => "([^"]*)".*/\1|\2/'

# Test with sample data
echo '      ~ user_data = "IyEvYmluL2Jhc2gKZWNobyAiT2xkIg==" -> "IyEvYmluL2Jhc2gKZWNobyAiTmV3Ig=="' | \
  sed -E 's/.*"([^"]*)" -> "([^"]*)".*/\1\n\2/' | \
  while read -r line; do echo "$line" | base64 -d; echo "---"; done
```

## Quick Reference

```bash
# Encode
echo "#!/bin/bash" | base64                          # Shell
base64encode("content")                              # Terraform

# Decode from state
terraform show -json | jq -r '...user_data' | base64 -d

# Decode from instance (AWS CLI)
aws ec2 describe-instance-attribute --instance-id ID --attribute userData --query 'UserData.Value' --output text | base64 -d

# Decode from inside instance
curl -s http://169.254.169.254/latest/user-data

# Decode gzipped multipart
... | base64 -d | gunzip

# Preview before apply
terraform console <<< 'templatefile("script.tpl", {var1 = "val1"})'

# Compare state vs reality
diff <(terraform show -json | jq -r '..user_data' | base64 -d) <(aws ec2 describe-instance-attribute ...)
```

| Function | Purpose |
|----------|---------|
| `base64encode()` | Encode string to base64 in Terraform |
| `base64decode()` | Decode base64 to string in Terraform |
| `base64 -d` | Decode base64 in shell (Linux) |
| `base64 --decode` | Decode base64 in shell (macOS) |
| `base64gzip()` | Gzip then base64 encode (Terraform) |
| `templatefile()` | Render template with variables |
| `file()` | Read file content as string |
