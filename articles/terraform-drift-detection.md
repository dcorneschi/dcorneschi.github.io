# Terraform Config Drift Detection

Configuration drift occurs when the real-world state of infrastructure diverges from what Terraform expects. This happens when resources are modified outside of Terraform — via the console, CLI, automation scripts, or other tools. Detecting and resolving drift is essential for maintaining infrastructure consistency.

## What is Drift?

Drift is the difference between:
- **Desired state** — what your `.tf` files declare
- **Actual state** — what exists in the cloud right now
- **Known state** — what Terraform's state file records

| Scenario | Example |
|----------|---------|
| Manual console change | Someone resizes an instance via AWS Console |
| External automation | A Lambda function modifies security group rules |
| Auto-scaling | ASG changes desired_capacity |
| Tagging policies | AWS Organizations adds compliance tags |
| Emergency hotfix | Ops team changes a route table directly |
| Terraform code change | You update `.tf` but haven't applied yet |

## Detecting Drift

### terraform plan (Standard Drift Check)

Every `terraform plan` detects drift by refreshing state against reality:

```bash
# Standard plan — refreshes state and shows drift + desired changes
terraform plan

# Output shows:
# ~ resource "aws_instance" "web" {
#     ~ instance_type = "t3.micro" -> "t3.large"  (drift detected)
#   }
```

### terraform plan -refresh-only (Drift Only, No Changes)

Shows what changed in the real world without proposing any fixes:

```bash
# Detect drift without planning any changes
terraform plan -refresh-only

# Shows only real-world changes, not config differences
# Useful for: "what happened since last apply?"
```

### terraform apply -refresh-only (Accept Drift)

Updates the state file to match reality without modifying infrastructure:

```bash
# Accept the current real-world state into Terraform state
terraform apply -refresh-only

# Use when:
# - An approved manual change was made
# - Auto-scaling adjusted capacity
# - You want state to reflect reality before making new changes
```

### terraform plan -detailed-exitcode (CI/CD Automation)

Use exit codes to detect drift programmatically:

```bash
terraform plan -detailed-exitcode
# Exit 0 = no changes (no drift, config matches)
# Exit 1 = error
# Exit 2 = changes detected (drift or pending config changes)
```

```bash
# CI/CD script example
terraform plan -detailed-exitcode -out=tfplan
EXIT_CODE=$?

case $EXIT_CODE in
  0) echo "No drift detected" ;;
  1) echo "Error running plan"; exit 1 ;;
  2) echo "DRIFT DETECTED — review changes"; exit 1 ;;
esac
```

## Understanding Plan Output

### Drift vs Desired Changes

```bash
# Drift indicator: ~ (tilde) with "has changed" note
# Terraform 1.4+ explicitly labels drift:

# aws_instance.web has changed
# ~ resource "aws_instance" "web" {
#       id            = "i-abc123"
#     ~ instance_type = "t3.micro" -> "t3.large"    # DRIFT (external change)
#       tags          = { ... }
#   }

# Desired change indicator: ~ (tilde) from your config update
# ~ resource "aws_instance" "web" {
#     ~ instance_type = "t3.micro" -> "t3.medium"   # YOUR change in .tf
#   }
```

### Reading the Symbols

| Symbol | Meaning |
|--------|---------|
| `+` | Create (new resource) |
| `-` | Destroy (remove resource) |
| `~` | Update in-place |
| `-/+` | Destroy and recreate (forced replacement) |
| `<=` | Read (data source) |

## Automated Drift Detection

### Scheduled Drift Check (Single Directory)

```bash
#!/bin/bash
# drift-check.sh — Run on a schedule (e.g., every hour via cron or CI)

set -euo pipefail

WORKDIR="/path/to/terraform"
SLACK_WEBHOOK="${SLACK_WEBHOOK_URL}"

cd "$WORKDIR"
terraform init -input=false -no-color

# Plan with detailed exit code
terraform plan -detailed-exitcode -input=false -no-color -out=drift.tfplan 2>&1 | tee drift-output.txt
EXIT_CODE=${PIPESTATUS[0]}

if [ "$EXIT_CODE" -eq 2 ]; then
    # Drift detected — notify
    DRIFT_SUMMARY=$(terraform show -no-color drift.tfplan | grep -E "^[[:space:]]*(~|\+|-)" | head -20)
    
    curl -X POST "$SLACK_WEBHOOK" \
      -H "Content-Type: application/json" \
      -d "{\"text\": \"⚠️ Terraform drift detected in $(basename $WORKDIR):\n\`\`\`${DRIFT_SUMMARY}\`\`\`\"}"
    
    exit 2
fi

echo "No drift detected"
rm -f drift.tfplan drift-output.txt
```

### GitHub Actions Drift Detection

```yaml
name: Terraform Drift Detection
on:
  schedule:
    - cron: '0 */6 * * *'  # Every 6 hours
  workflow_dispatch:

jobs:
  drift-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.0

      - name: Terraform Init
        run: terraform init -input=false

      - name: Check for Drift
        id: plan
        run: |
          terraform plan -detailed-exitcode -input=false -no-color -out=tfplan 2>&1 | tee plan-output.txt
          echo "exitcode=$?" >> $GITHUB_OUTPUT
        continue-on-error: true

      - name: Report Drift
        if: steps.plan.outputs.exitcode == '2'
        run: |
          echo "## ⚠️ Drift Detected" >> $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY
          terraform show -no-color tfplan >> $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY

      - name: Fail on Drift
        if: steps.plan.outputs.exitcode == '2'
        run: exit 1
```

### GitLab CI Drift Detection

```yaml
drift-detection:
  stage: monitor
  image: hashicorp/terraform:1.7
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script:
    - terraform init -input=false
    - terraform plan -detailed-exitcode -input=false -no-color
  allow_failure:
    exit_codes:
      - 2  # Drift detected but don't block pipeline
```

## Multi-Cluster Drift Detection

### Simple Script (Standard Directory Structure)

```bash
#!/bin/bash
# simple-drift-detection.sh
CLUSTERS=("cluster-1" "cluster-2" "cluster-3")

for cluster in "${CLUSTERS[@]}"; do
    echo "Checking drift for $cluster..."
    cd "terraform/$cluster"

    terraform plan -detailed-exitcode -out=plan.out
    exit_code=$?

    if [ $exit_code -eq 2 ]; then
        echo "⚠️  DRIFT DETECTED in $cluster"
        terraform show plan.out
    elif [ $exit_code -eq 0 ]; then
        echo "✅ No drift in $cluster"
    else
        echo "❌ Error checking $cluster"
    fi

    cd ../..
done
```

### Flexible Script (Custom Directory Paths)

```bash
#!/bin/bash
# drift-detection.sh

declare -A CLUSTERS=(
    ["prod-us-east"]="infrastructure/aws/us-east-1"
    ["prod-eu-west"]="infrastructure/aws/eu-west-1"
    ["staging-cluster"]="environments/staging/terraform"
    ["dev-cluster"]="dev/k8s-terraform"
)

ORIGINAL_DIR=$(pwd)
DRIFT_SUMMARY=()

for cluster_name in "${!CLUSTERS[@]}"; do
    cluster_path="${CLUSTERS[$cluster_name]}"
    echo "Checking drift for $cluster_name at $cluster_path..."

    if [ ! -d "$cluster_path" ]; then
        echo "❌ Directory $cluster_path does not exist"
        DRIFT_SUMMARY+=("$cluster_name: DIRECTORY_NOT_FOUND")
        continue
    fi

    cd "$cluster_path" || continue

    if [ ! -d ".terraform" ]; then
        terraform init -input=false >/dev/null 2>&1
    fi

    terraform plan -detailed-exitcode >/dev/null 2>&1
    exit_code=$?

    case $exit_code in
        0) DRIFT_SUMMARY+=("$cluster_name: NO_DRIFT") ;;
        2) DRIFT_SUMMARY+=("$cluster_name: DRIFT_DETECTED") ;;
        *) DRIFT_SUMMARY+=("$cluster_name: ERROR") ;;
    esac

    cd "$ORIGINAL_DIR"
done

echo ""
echo "=========================================="
echo "DRIFT DETECTION SUMMARY"
echo "=========================================="
for summary in "${DRIFT_SUMMARY[@]}"; do
    echo "$summary"
done

# Exit with error if any drift detected
for summary in "${DRIFT_SUMMARY[@]}"; do
    if [[ "$summary" == *"DRIFT_DETECTED"* ]]; then
        exit 1
    fi
done
exit 0
```

### JSON Configuration Approach

**clusters.json:**

```json
{
  "clusters": [
    {
      "name": "prod-us-east",
      "path": "infrastructure/aws/us-east-1",
      "backend_config": "backend-prod.hcl"
    },
    {
      "name": "prod-eu-west",
      "path": "infrastructure/aws/eu-west-1",
      "backend_config": "backend-prod.hcl"
    },
    {
      "name": "staging",
      "path": "environments/staging/terraform"
    }
  ]
}
```

```bash
#!/bin/bash
# drift-detection-config.sh

CONFIG_FILE="clusters.json"
ORIGINAL_DIR=$(pwd)

clusters=$(jq -r '.clusters[] | @base64' "$CONFIG_FILE")

for cluster_data in $clusters; do
    cluster=$(echo "$cluster_data" | base64 --decode)
    name=$(echo "$cluster" | jq -r '.name')
    path=$(echo "$cluster" | jq -r '.path')
    backend_config=$(echo "$cluster" | jq -r '.backend_config // empty')

    echo "Checking drift for $name..."
    cd "$path" || continue

    if [ -n "$backend_config" ]; then
        terraform init -backend-config="$backend_config" -input=false >/dev/null 2>&1
    else
        terraform init -input=false >/dev/null 2>&1
    fi

    terraform plan -detailed-exitcode -out=plan.out
    exit_code=$?

    case $exit_code in
        0) echo "✅ No drift in $name" ;;
        2) echo "⚠️  DRIFT DETECTED in $name"; terraform show plan.out ;;
        *) echo "❌ Error checking $name" ;;
    esac

    cd "$ORIGINAL_DIR"
    echo ""
done
```

### Auto-Discovery of Terraform Directories

```bash
#!/bin/bash
# auto-discover-drift.sh — Find all Terraform directories and check drift

ORIGINAL_DIR=$(pwd)
drift_found=false

echo "Auto-discovering Terraform directories..."
terraform_dirs=$(find . -type f -name "*.tf" -exec dirname {} \; | sort -u)

for dir in $terraform_dirs; do
    if [ -z "$(find "$dir" -maxdepth 1 -name "*.tf" -type f)" ]; then
        continue
    fi

    cluster_name=$(basename "$dir")
    echo "Checking drift for $cluster_name at $dir..."

    cd "$dir" || continue

    if [ ! -d ".terraform" ]; then
        terraform init -input=false >/dev/null 2>&1
        if [ $? -ne 0 ]; then
            echo "❌ Failed to initialize $cluster_name"
            cd "$ORIGINAL_DIR"
            continue
        fi
    fi

    terraform plan -detailed-exitcode >/dev/null 2>&1
    exit_code=$?

    case $exit_code in
        0) echo "✅ No drift in $cluster_name" ;;
        2) echo "⚠️  DRIFT DETECTED in $cluster_name"; drift_found=true ;;
        *) echo "❌ Error checking $cluster_name" ;;
    esac

    cd "$ORIGINAL_DIR"
done

if [ "$drift_found" = true ]; then
    echo "🚨 Drift detected in one or more directories!"
    exit 1
fi
```

### Makefile Integration

```makefile
.PHONY: drift-check drift-check-all drift-fix

drift-check-all:
	@echo "Checking drift for all clusters..."
	@./scripts/drift-detection.sh

drift-check:
	@if [ -z "$(CLUSTER)" ]; then \
		echo "Usage: make drift-check CLUSTER=cluster-name"; \
		exit 1; \
	fi
	@./scripts/drift-detection.sh $(CLUSTER)

drift-fix:
	@if [ -z "$(CLUSTER)" ]; then \
		echo "Usage: make drift-fix CLUSTER=cluster-name"; \
		exit 1; \
	fi
	cd $$(jq -r ".clusters[] | select(.name==\"$(CLUSTER)\") | .path" clusters.json) && terraform apply
```

### Docker-Based Drift Detection

```bash
#!/bin/bash
# docker-drift-detection.sh — Consistent environment via Docker

declare -A CLUSTERS=(
    ["prod-us-east"]="infrastructure/aws/us-east-1:1.7"
    ["staging"]="environments/staging/terraform:1.6"
)

for cluster_name in "${!CLUSTERS[@]}"; do
    IFS=':' read -r cluster_path tf_version <<< "${CLUSTERS[$cluster_name]}"

    echo "Checking $cluster_name (Terraform $tf_version)..."

    docker run --rm \
        -v "$PWD/$cluster_path:/workspace" \
        -v "$HOME/.aws:/root/.aws:ro" \
        -w /workspace \
        "hashicorp/terraform:$tf_version" \
        sh -c '
            terraform init -input=false >/dev/null 2>&1
            terraform plan -detailed-exitcode >/dev/null 2>&1
            case $? in
                0) echo "✅ No drift" ;;
                2) echo "⚠️  DRIFT DETECTED" ;;
                *) echo "❌ Error" ;;
            esac
        '
done
```

## Handling Plugin and Version Issues

### Validate and Auto-Fix Before Drift Check

```bash
validate_and_fix() {
    local cluster_name="$1"

    terraform validate >/dev/null 2>&1
    if [ $? -ne 0 ]; then
        error_output=$(terraform validate 2>&1)

        if echo "$error_output" | grep -qi "plugin\|provider.*not installed\|registry.terraform.io"; then
            echo "⚠️  Plugin issue in $cluster_name, reinitializing..."
            terraform init -input=false >/dev/null 2>&1
        elif echo "$error_output" | grep -qi "version constraint"; then
            echo "⚠️  Version constraint issue in $cluster_name, upgrading..."
            terraform init -upgrade -input=false >/dev/null 2>&1
        else
            echo "❌ Validation error in $cluster_name"
            return 1
        fi
    fi
    return 0
}
```

### Version-Aware Detection with tfenv

```bash
#!/bin/bash
# Requires tfenv installed

declare -A CLUSTERS=(
    ["prod"]="infrastructure/prod|1.7.0"
    ["staging"]="infrastructure/staging|1.6.0"
)

for cluster_name in "${!CLUSTERS[@]}"; do
    IFS='|' read -r cluster_path tf_version <<< "${CLUSTERS[$cluster_name]}"

    echo "Checking $cluster_name (requires Terraform $tf_version)..."
    tfenv use "$tf_version" 2>/dev/null || tfenv install "$tf_version" && tfenv use "$tf_version"

    cd "$cluster_path" || continue
    terraform init -input=false >/dev/null 2>&1
    terraform plan -detailed-exitcode >/dev/null 2>&1

    case $? in
        0) echo "✅ No drift" ;;
        2) echo "⚠️  DRIFT DETECTED" ;;
        *) echo "❌ Error" ;;
    esac

    cd "$ORIGINAL_DIR"
done
```

### Common Plugin Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Plugin not installed | Version change or missing init | `terraform init` |
| Version constraint mismatch | Lock file conflicts | `terraform init -upgrade` |
| Corrupted plugin cache | Interrupted download | Remove `.terraform/providers`, reinit |
| Architecture mismatch | Plugins for wrong OS/arch | Remove `.terraform`, reinit on correct platform |
| Registry unreachable | Network/firewall issue | Use mirror or local plugin cache |

## Resolving Drift

### Option 1: Revert Drift (Apply Your Config)

Force infrastructure back to what your `.tf` declares:

```bash
# Standard apply — reverts drift AND applies any new changes
terraform apply
```

### Option 2: Accept Drift (Update State)

Accept the real-world state when the external change is intentional:

```bash
# Update state to match reality
terraform apply -refresh-only

# Then update your .tf files to match
# (so next plan shows no changes)
```

### Option 3: Ignore Drift (lifecycle)

For attributes managed by external tools, prevent Terraform from detecting drift:

```hcl
resource "aws_autoscaling_group" "app" {
  desired_capacity = 3
  min_size         = 2
  max_size         = 10

  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
# Auto-scaling changes desired_capacity — Terraform won't report drift
```

### Option 4: Remove from State

If a resource should no longer be managed by Terraform:

```bash
# Remove from Terraform management (resource stays in cloud)
terraform state rm aws_instance.manually_managed

# Resource still exists but Terraform forgets about it
```

### Option 5: Import the Current State

If the resource was recreated or modified significantly:

```bash
# Remove old state and reimport
terraform state rm aws_instance.web
terraform import aws_instance.web i-0abc123def456789

# Then adjust .tf to match current attributes
terraform plan  # Should show no changes if .tf matches reality
```

## Drift Detection with JSON Output

### Parse Drift Programmatically

```bash
# Generate plan as JSON
terraform plan -out=tfplan
terraform show -json tfplan > plan.json

# Find drifted resources (action_reason = "read_because_config_unknown" or changes outside TF)
jq '.resource_changes[] | select(.change.actions != ["no-op"])' plan.json

# List only drifted resource addresses
jq -r '.resource_changes[] | select(.change.actions != ["no-op"]) | .address' plan.json

# Show before/after for drifted resources
jq '.resource_changes[] | select(.change.actions != ["no-op"]) | {address, before: .change.before, after: .change.after}' plan.json

# Count drift by resource type
jq -r '[.resource_changes[] | select(.change.actions != ["no-op"]) | .type] | group_by(.) | map({type: .[0], count: length})' plan.json
```

### Drift Report Script

```bash
#!/bin/bash
# drift-report.sh — Generate a drift report

terraform plan -out=tfplan -input=false >/dev/null 2>&1
terraform show -json tfplan > plan.json

echo "=== Terraform Drift Report ==="
echo "Date: $(date)"
echo ""

# Count changes
TOTAL=$(jq '[.resource_changes[] | select(.change.actions != ["no-op"])] | length' plan.json)
CREATES=$(jq '[.resource_changes[] | select(.change.actions == ["create"])] | length' plan.json)
UPDATES=$(jq '[.resource_changes[] | select(.change.actions == ["update"])] | length' plan.json)
DELETES=$(jq '[.resource_changes[] | select(.change.actions == ["delete"])] | length' plan.json)
REPLACES=$(jq '[.resource_changes[] | select(.change.actions | contains(["delete","create"]))] | length' plan.json)

echo "Total changes: $TOTAL"
echo "  Creates:  $CREATES"
echo "  Updates:  $UPDATES"
echo "  Deletes:  $DELETES"
echo "  Replaces: $REPLACES"
echo ""

if [ "$TOTAL" -gt 0 ]; then
    echo "Changed resources:"
    jq -r '.resource_changes[] | select(.change.actions != ["no-op"]) | "  \(.change.actions | join(",")): \(.address)"' plan.json
fi

rm -f tfplan plan.json
```

## Preventing Drift

### 1. Restrict Console/CLI Access

```bash
# Use IAM policies to prevent manual changes
# Allow read-only console access, require Terraform for modifications
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "ec2:ModifyInstanceAttribute",
        "ec2:AuthorizeSecurityGroupIngress",
        "rds:ModifyDBInstance"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": "terraform-automation"
        }
      }
    }
  ]
}
```

### 2. Tag Resources as Terraform-Managed

```hcl
# Default tags on all resources
provider "aws" {
  region = "eu-west-1"

  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Repository  = "infra-repo"
      Workspace   = terraform.workspace
    }
  }
}
```

### 3. Use ignore_changes Strategically

```hcl
# Only ignore attributes that SHOULD change externally
lifecycle {
  ignore_changes = [
    desired_capacity,  # ASG auto-scaling
    task_definition,   # CI/CD deployments
    tags["LastScan"],  # Security scanning tools
  ]
}
# Don't use ignore_changes = all unless fully externally managed
```

### 4. Lock Down State with Sentinel/OPA

```hcl
# Sentinel policy example (Terraform Cloud/Enterprise)
# Deny plans with drift on critical resources
import "tfplan/v2" as tfplan

critical_types = ["aws_db_instance", "aws_kms_key", "aws_vpc"]

drift_on_critical = filter tfplan.resource_changes as _, rc {
    rc.type in critical_types and
    rc.change.actions contains "update"
}

main = rule {
    length(drift_on_critical) == 0
}
```

### 5. Scheduled Reconciliation

```bash
# Cron job: detect and auto-fix drift on non-critical resources
# /etc/cron.d/terraform-reconcile

0 2 * * * root /opt/terraform/reconcile.sh >> /var/log/terraform-reconcile.log 2>&1
```

```bash
#!/bin/bash
# reconcile.sh — Auto-apply if only non-critical drift exists

cd /path/to/terraform

terraform plan -detailed-exitcode -out=tfplan -input=false
if [ $? -eq 2 ]; then
    # Check if changes are only to non-critical resources
    CRITICAL=$(terraform show -json tfplan | jq '[.resource_changes[] | select(.change.actions != ["no-op"]) | select(.type | test("aws_db_instance|aws_kms_key|aws_vpc"))] | length')
    
    if [ "$CRITICAL" -eq 0 ]; then
        echo "Auto-fixing non-critical drift..."
        terraform apply -auto-approve tfplan
    else
        echo "CRITICAL drift detected — manual review required"
        # Send alert
    fi
fi
```

## Drift in Terraform Cloud/Enterprise

Terraform Cloud has built-in drift detection:

```hcl
# In workspace settings, enable "Health > Drift Detection"
# TFC automatically runs refresh-only plans on a schedule

# Notifications are sent via:
# - Email
# - Slack
# - Webhooks
# - Microsoft Teams
```

### Workspace Configuration

| Setting | Recommendation |
|---------|---------------|
| Drift detection | Enabled |
| Check interval | Every 24 hours (or per policy) |
| Auto-apply refresh | Disabled (review first) |
| Notifications | Slack + email for production |

## Common Drift Scenarios

| Scenario | Detection | Resolution |
|----------|-----------|------------|
| Instance resized in console | `plan` shows `instance_type` change | Apply to revert, or accept with `-refresh-only` |
| SG rule added manually | `plan` shows ingress rule diff | Apply to remove, or import and update .tf |
| ASG capacity changed by scaling | `plan` shows `desired_capacity` change | Use `ignore_changes` — this is expected |
| Tags added by AWS Config | `plan` shows tag additions | Use `ignore_changes = [tags]` |
| Resource deleted externally | `plan` shows recreation needed | Apply to recreate, or `state rm` to forget |
| DNS record changed manually | `plan` shows record value change | Apply to revert, or update .tf |

## Best Practices

1. **Run drift detection on a schedule** — catch changes before they compound
2. **Use `-refresh-only`** to understand drift without making decisions yet
3. **Tag everything as Terraform-managed** — makes it obvious what shouldn't be touched manually
4. **Use `ignore_changes`** sparingly and only for genuinely external attributes
5. **Don't auto-apply drift fixes** on critical resources — always review first
6. **Keep state locked** during applies to prevent concurrent modifications
7. **Use `-detailed-exitcode`** in CI/CD to fail pipelines on unexpected drift
8. **Document approved manual changes** — update `.tf` files to match accepted changes
9. **Separate environments** — drift in dev is less critical than drift in prod
10. **Review drift before applying new changes** — run `plan -refresh-only` first to understand the baseline
