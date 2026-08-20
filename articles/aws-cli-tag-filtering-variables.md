# AWS CLI Tag Filtering with Shell Variables

Patterns for using shell variables in AWS CLI tag-based filters — variable substitution, arrays, functions, loops, and validation.

## Basic Variable Substitution

```bash
# Define the tag name and value as variables
TAG_NAME="Environment"
TAG_VALUE="production"

# Use the variable in the AWS CLI command
aws ec2 describe-instances \
    --filters "Name=tag:${TAG_NAME},Values=${TAG_VALUE}"

# Or with shorthand (less safe with adjacent characters)
aws ec2 describe-instances \
    --filters "Name=tag:$TAG_NAME,Values=$TAG_VALUE"
```

## Multiple Variables

```bash
TAG_KEY="Project"
TAG_VALUE="my-project"
REGION="us-west-2"

aws ec2 describe-instances \
    --region "$REGION" \
    --filters "Name=tag:${TAG_KEY},Values=${TAG_VALUE}"
```

## Dynamic Tag Key

```bash
# When the tag key itself is dynamic
ENVIRONMENT="prod"
TAG_KEY="Environment-${ENVIRONMENT}"

aws ec2 describe-instances \
    --filters "Name=tag:${TAG_KEY},Values=active"
```

## Arrays for Multiple Values

```bash
TAG_NAME="Environment"
TAG_VALUES=("production" "staging" "development")

# Convert array to comma-separated string
VALUES=$(IFS=,; echo "${TAG_VALUES[*]}")

aws ec2 describe-instances \
    --filters "Name=tag:${TAG_NAME},Values=${VALUES}"
```

## Reading from File or Command Output

```bash
# Read tag value from file
TAG_VALUE=$(cat /path/to/tag-value.txt)

# Or from another command
TAG_VALUE=$(aws sts get-caller-identity --query 'Account' --output text)

aws ec2 describe-instances \
    --filters "Name=tag:Owner,Values=${TAG_VALUE}"
```

## JSON Format with Variables

```bash
TAG_NAME="Application"
TAG_VALUE="web-server"

# Using JSON format (useful for complex filters)
aws ec2 describe-instances \
    --filters "[{\"Name\":\"tag:${TAG_NAME}\",\"Values\":[\"${TAG_VALUE}\"]}]"
```

## Reusable Function

```bash
find_by_tag() {
    local tag_name=$1
    local tag_value=$2
    local service=${3:-ec2}

    case $service in
        ec2)
            aws ec2 describe-instances \
                --filters "Name=tag:${tag_name},Values=${tag_value}"
            ;;
        s3)
            aws s3api list-buckets \
                --query "Buckets[?contains(Tags[?Key=='${tag_name}'].Value, '${tag_value}')]"
            ;;
    esac
}

# Usage
find_by_tag "Environment" "production"
find_by_tag "Project" "my-app" "s3"
```

## Multiple Tag Filters

```bash
PROJECT="my-project"
ENV="production"
TEAM="backend"

aws ec2 describe-instances \
    --filters \
        "Name=tag:Project,Values=${PROJECT}" \
        "Name=tag:Environment,Values=${ENV}" \
        "Name=tag:Team,Values=${TEAM}"
```

## With JMESPath Query

```bash
TAG_NAME="Name"
TAG_VALUE="web-server"

aws ec2 describe-instances \
    --filters "Name=tag:${TAG_NAME},Values=${TAG_VALUE}" \
    --query "Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key=='${TAG_NAME}'].Value|[0],State:State.Name}" \
    --output table
```

## Practical Examples

### Find Instances by Environment

```bash
CLUSTER_ENV="production"
aws ec2 describe-instances \
    --filters "Name=tag:Environment,Values=${CLUSTER_ENV}" \
    --query 'Reservations[].Instances[].[InstanceId,Tags[?Key==`Name`].Value|[0]]' \
    --output table
```

### Find Subnets by Tag

```bash
VPC_NAME="production-vpc"
aws ec2 describe-subnets \
    --filters "Name=tag:VPC,Values=${VPC_NAME}"
```

### Find Security Groups by Tag

```bash
APPLICATION="web-app"
aws ec2 describe-security-groups \
    --filters "Name=tag:Application,Values=${APPLICATION}"
```

### Find RDS Instances by Tag

```bash
DATABASE_TYPE="mysql"
aws rds describe-db-instances \
    --query "DBInstances[?TagList[?Key=='Type' && Value=='${DATABASE_TYPE}']]"
```

### Find Load Balancers by Tag

```bash
ENVIRONMENT="staging"
aws elbv2 describe-tags \
    --query "TagDescriptions[?Tags[?Key=='Environment' && Value=='${ENVIRONMENT}']]"
```

## Advanced Patterns

### Environment-Based Configuration

```bash
#!/bin/bash
ENV=${1:-development}
REGION=${2:-us-east-1}

aws ec2 describe-instances \
    --region "$REGION" \
    --filters "Name=tag:Environment,Values=${ENV}" \
    --query 'Reservations[].Instances[].[InstanceId,Tags[?Key==`Name`].Value|[0]]' \
    --output table
```

### Loop Through Multiple Tags

```bash
ENVIRONMENTS=("development" "staging" "production")

for env in "${ENVIRONMENTS[@]}"; do
    echo "=== Environment: $env ==="
    aws ec2 describe-instances \
        --filters "Name=tag:Environment,Values=${env}" \
        --query 'Reservations[].Instances[].InstanceId' \
        --output text
done
```

### Conditional Tag Filtering

```bash
if [ "$ENV" == "production" ]; then
    TAG_FILTER="Name=tag:CriticalService,Values=true"
else
    TAG_FILTER="Name=tag:Environment,Values=${ENV}"
fi

aws ec2 describe-instances --filters "$TAG_FILTER"
```

### Interactive Tag Selection

```bash
echo "Available environments:"
aws ec2 describe-instances \
    --query 'Reservations[].Instances[].Tags[?Key==`Environment`].Value' \
    --output text | tr '\t' '\n' | sort -u

read -p "Enter environment: " SELECTED_ENV

aws ec2 describe-instances \
    --filters "Name=tag:Environment,Values=${SELECTED_ENV}"
```

## Variable Validation

### Require a Variable

```bash
TAG_VALUE=${1:?"Usage: $0 <tag-value>"}

aws ec2 describe-instances \
    --filters "Name=tag:Project,Values=${TAG_VALUE}"
```

### Default Values

```bash
TAG_VALUE=${TAG_VALUE:-"default-project"}
REGION=${AWS_DEFAULT_REGION:-"us-east-1"}

aws ec2 describe-instances \
    --region "$REGION" \
    --filters "Name=tag:Project,Values=${TAG_VALUE}"
```

### Validate Tag Returns Results

```bash
TAG_NAME="Project"
TAG_VALUE="my-app"

RESULT=$(aws ec2 describe-instances \
    --filters "Name=tag:${TAG_NAME},Values=${TAG_VALUE}" \
    --query 'Reservations[].Instances[].InstanceId' \
    --output text)

if [ -z "$RESULT" ]; then
    echo "No instances found with tag ${TAG_NAME}=${TAG_VALUE}"
    exit 1
fi

echo "Found instances: $RESULT"
```

## Debugging

```bash
# Enable debug mode to see variable expansion
set -x

TAG_NAME="Environment"
TAG_VALUE="production"

aws ec2 describe-instances \
    --filters "Name=tag:${TAG_NAME},Values=${TAG_VALUE}"

set +x
```

## Tips and Common Pitfalls

| Tip | Why |
|-----|-----|
| Always quote filter values | Spaces or special characters break unquoted variables |
| Use `${VAR}` over `$VAR` | Better isolation from adjacent characters |
| Validate inputs before running | Empty variables produce malformed filters |
| Use arrays for multiple values | More maintainable than hardcoded comma-separated strings |
| Tag names and values are case-sensitive | `Environment` != `environment` |
| Use functions for repeated queries | DRY principle, easier to maintain |
| Check exit status in scripts | `$?` tells you if the command succeeded |
| Test with `--dry-run` when available | Validates the command without executing |
