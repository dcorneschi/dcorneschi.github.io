# JMESPath Query Language for AWS CLI

JMESPath is a query language for JSON used by AWS CLI to filter and format output. Use the `--query` parameter to apply JMESPath expressions.

```bash
aws <service> <command> --query '<expression>' --output <format>
```

## Basic Syntax

### Basic Structure

```bash
aws <service> <command> --query '<expression>' --output <format>
```

### Common Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `.` | Access nested field | `State.Name` |
| `[]` | Array indexing/slicing | `Items[0]`, `Items[:5]` |
| `[*]` | Wildcard (all elements, preserves structure) | `Instances[*].InstanceId` |
| `[]` | Flatten projection | `Reservations[].Instances[]` |
| `\|` | Pipe (pass result to next expression) | `Items[] \| [0]` |
| `@` | Current element (in filter context) | `max_by(@, &Size)` |
| `?` | Filter expression | `[?State==\`running\`]` |
| `&&` | Logical AND | `[?A==\`x\` && B==\`y\`]` |
| `\|\|` | Logical OR | `[?A==\`x\` \|\| A==\`y\`]` |
| `!` | Logical NOT | `[?!(State==\`terminated\`)]` |
| `` ` ` `` | Literal value | `` `running` ``, `` `100` `` |

```bash
# Get a single top-level field
aws ec2 describe-vpcs --query 'Vpcs[0].VpcId'
# "vpc-0123456789abcdef0"

# Nested field
aws ec2 describe-instances --query 'Reservations[0].Instances[0].State.Name'
# "running"
```

### Select Multiple Fields

```bash
# Array of specific fields (multi-select list)
aws ec2 describe-instances \
    --query 'Reservations[0].Instances[0].[InstanceId, InstanceType, State.Name]'
# ["i-0123456789abcdef0", "t3.micro", "running"]

# Named fields (multi-select hash)
aws ec2 describe-instances \
    --query 'Reservations[0].Instances[0].{ID:InstanceId, Type:InstanceType, State:State.Name}'
# {"ID": "i-0123456789abcdef0", "Type": "t3.micro", "State": "running"}
```

### Index Access

```bash
# First element
--query 'Items[0]'

# Last element
--query 'Items[-1]'

# Specific index
--query 'Items[2]'
```

### Array Slicing

```bash
# First 5 elements
--query 'Buckets[:5].Name'

# Skip first 2, get next 3
--query 'Buckets[2:5].Name'

# Last 3 elements
--query 'Items[-3:]'

# Every other element
--query 'Items[::2]'
```

## Flatten and Iterate

### Flatten Nested Arrays

AWS often returns nested structures (`Reservations[].Instances[]`). Use `[]` to flatten:

```bash
# Without flatten — nested arrays
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[*].InstanceId'
# [["i-aaa"], ["i-bbb", "i-ccc"]]

# With flatten — flat list
aws ec2 describe-instances \
    --query 'Reservations[].Instances[].InstanceId'
# ["i-aaa", "i-bbb", "i-ccc"]
```

`[*]` preserves structure, `[]` flattens one level.

### Iterate and Select Fields

```bash
# All instances with specific fields
aws ec2 describe-instances \
    --query 'Reservations[].Instances[].[InstanceId, InstanceType, PublicIpAddress]' \
    --output table

# Named fields across all instances
aws ec2 describe-instances \
    --query 'Reservations[].Instances[].{ID:InstanceId, Type:InstanceType, IP:PublicIpAddress}' \
    --output table
```

## Filtering

### Filter by Value

Use `?` inside brackets for conditional filtering:

```bash
# Instances in running state
aws ec2 describe-instances \
    --query 'Reservations[].Instances[?State.Name==`running`].[InstanceId, InstanceType]' \
    --output table

# Volumes larger than 100 GB
aws ec2 describe-volumes \
    --query 'Volumes[?Size>`100`].[VolumeId, Size, VolumeType]' \
    --output table

# Security groups in a specific VPC
aws ec2 describe-security-groups \
    --query 'SecurityGroups[?VpcId==`vpc-0123456789abcdef0`].GroupName'
```

### Comparison Operators

| Operator | Meaning |
|----------|---------|
| `==` | Equal |
| `!=` | Not equal |
| `<` | Less than |
| `<=` | Less than or equal |
| `>` | Greater than |
| `>=` | Greater than or equal |

```bash
# Instances NOT in terminated state
aws ec2 describe-instances \
    --query 'Reservations[].Instances[?State.Name!=`terminated`].[InstanceId, State.Name]'

# EBS volumes smaller than 50 GB
aws ec2 describe-volumes \
    --query 'Volumes[?Size<`50`].[VolumeId, Size]'
```

### Logical Operators

| Operator | Syntax |
|----------|--------|
| AND | `&&` |
| OR | `\|\|` |
| NOT | `!` |

```bash
# Running AND t3.micro
aws ec2 describe-instances \
    --query 'Reservations[].Instances[?State.Name==`running` && InstanceType==`t3.micro`].[InstanceId]'

# Volumes that are available OR have errors
aws ec2 describe-volumes \
    --query 'Volumes[?State==`available` || State==`error`].[VolumeId, State]'

# Instances that are NOT terminated
aws ec2 describe-instances \
    --query 'Reservations[].Instances[?!(State.Name==`terminated`)].[InstanceId, State.Name]'
```

### Filter by Tag

Tags are arrays of `{Key, Value}` objects, which requires a nested query:

```bash
# Get Name tag value
aws ec2 describe-instances \
    --query 'Reservations[].Instances[].[InstanceId, Tags[?Key==`Name`].Value | [0]]' \
    --output table

# Filter instances by tag value
aws ec2 describe-instances \
    --query 'Reservations[].Instances[?Tags[?Key==`Environment` && Value==`production`]].[InstanceId, InstanceType]'

# Get multiple tag values
aws ec2 describe-instances \
    --query 'Reservations[].Instances[].{
        ID: InstanceId,
        Name: Tags[?Key==`Name`].Value | [0],
        Env: Tags[?Key==`Environment`].Value | [0]
    }' --output table
```

### Filter with contains()

```bash
# Instance types containing "large"
aws ec2 describe-instances \
    --query 'Reservations[].Instances[?contains(InstanceType, `large`)].[InstanceId, InstanceType]'

# Security groups with "web" in the name
aws ec2 describe-security-groups \
    --query 'SecurityGroups[?contains(GroupName, `web`)].[GroupId, GroupName]'

# Tags containing a substring
aws ec2 describe-instances \
    --query 'Reservations[].Instances[?Tags[?Key==`Name` && contains(Value, `prod`)]].[InstanceId]'
```

### Filter with starts_with() and ends_with()

```bash
# Instance IDs starting with specific prefix (in tags)
aws ec2 describe-instances \
    --query 'Reservations[].Instances[?Tags[?Key==`Name` && starts_with(Value, `web-`)]].[InstanceId]'

# AMIs ending with specific suffix
aws ec2 describe-images --owners self \
    --query 'Images[?ends_with(Name, `-prod`)].[ImageId, Name]'
```

## Sorting

### sort_by()

```bash
# Sort instances by launch time
aws ec2 describe-instances \
    --query 'sort_by(Reservations[].Instances[], &LaunchTime)[].[InstanceId, LaunchTime]' \
    --output table

# Sort volumes by size (descending — use reverse())
aws ec2 describe-volumes \
    --query 'reverse(sort_by(Volumes, &Size))[].[VolumeId, Size, VolumeType]' \
    --output table

# Sort AMIs by creation date (get the newest)
aws ec2 describe-images --owners self \
    --query 'sort_by(Images, &CreationDate)[-1].[ImageId, Name, CreationDate]' \
    --output text
```

## Functions

### Common Functions

| Function | Description | Example |
|----------|-------------|---------|
| `length()` | Count elements | `length(Reservations[].Instances[])` |
| `sort_by()` | Sort by field | `sort_by(Items, &Field)` |
| `reverse()` | Reverse order | `reverse(sort_by(Items, &Date))` |
| `max_by()` | Element with max value | `max_by(Items, &Size)` |
| `min_by()` | Element with min value | `min_by(Items, &Size)` |
| `sum()` | Sum numeric values | `sum(Contents[].Size)` |
| `contains()` | Check if string/array contains value | `contains(Name, \`prod\`)` |
| `starts_with()` | Check string prefix | `starts_with(Name, \`web\`)` |
| `ends_with()` | Check string suffix | `ends_with(Name, \`.json\`)` |
| `not_null()` | First non-null argument | `not_null(PublicIp, PrivateIp)` |
| `to_string()` | Convert to string | `to_string(Size)` |
| `to_number()` | Convert to number | `to_number(Port)` |
| `type()` | Get value type | `type(Value)` |
| `keys()` | Get object keys | `keys(Tags)` |
| `values()` | Get object values | `values(Tags)` |
| `join()` | Join array to string | `join(\`,\`, Items)` |

### Function Examples

```bash
# Count running instances
aws ec2 describe-instances \
    --filters "Name=instance-state-name,Values=running" \
    --query 'length(Reservations[].Instances[])'

# Get the largest volume
aws ec2 describe-volumes \
    --query 'max_by(Volumes, &Size).[VolumeId, Size]'

# Get the newest AMI
aws ec2 describe-images --owners self \
    --query 'max_by(Images, &CreationDate).[ImageId, Name, CreationDate]' \
    --output text

# First non-null IP (public or private)
aws ec2 describe-instances \
    --query 'Reservations[].Instances[].{ID:InstanceId, IP:not_null(PublicIpAddress, PrivateIpAddress)}'

# Join instance IDs into comma-separated string
aws ec2 describe-instances \
    --filters "Name=instance-state-name,Values=running" \
    --query 'join(`,`, Reservations[].Instances[].InstanceId)'
```

## Pipe Expressions

Use `|` to pipe the result of one expression to another:

```bash
# Flatten then get first element
aws ec2 describe-instances \
    --query 'Reservations[].Instances[].InstanceId | [0]'

# Sort then get the last 5
aws ec2 describe-images --owners self \
    --query 'sort_by(Images, &CreationDate) | [-5:].[ImageId, Name]' \
    --output table

# Filter then count
aws ec2 describe-instances \
    --query 'Reservations[].Instances[?State.Name==`running`] | length([])'

# Flatten nested tag query
aws ec2 describe-instances \
    --query 'Reservations[].Instances[].[Tags[?Key==`Name`].Value | [0], InstanceId]'
```

## Wildcards and Projections

### Wildcard `*`

```bash
# All values from all keys in a map
--query 'SecurityGroup.*'

# All elements in an array (explicit)
--query 'Instances[*].InstanceId'
```

### Object Projection

```bash
# Get all values from a tag map
aws ec2 describe-instances \
    --query 'Reservations[].Instances[].Tags[*].Value'
```

## Output Formats

The `--output` flag changes how JMESPath results are displayed:

| Format | Best For |
|--------|----------|
| `json` | Programmatic parsing, piping to `jq` |
| `text` | Shell scripting, piping to `awk`/`cut`/`xargs` |
| `table` | Human-readable display |
| `yaml` | Readable structured output |

```bash
# Table with headers (multi-select hash gives nice headers)
aws ec2 describe-instances \
    --query 'Reservations[].Instances[].{Name:Tags[?Key==`Name`].Value|[0], ID:InstanceId, Type:InstanceType, State:State.Name}' \
    --output table

# Text for scripting (tab-separated)
aws ec2 describe-instances \
    --query 'Reservations[].Instances[].[InstanceId, PublicIpAddress]' \
    --output text

# JSON for jq processing
aws ec2 describe-instances \
    --query 'Reservations[].Instances[]' \
    --output json | jq '.[].InstanceId'
```

## Real-World Examples

### EC2

```bash
# All running instances with Name, ID, type, IPs
aws ec2 describe-instances \
    --filters "Name=instance-state-name,Values=running" \
    --query 'Reservations[].Instances[].{
        Name: Tags[?Key==`Name`].Value | [0],
        ID: InstanceId,
        Type: InstanceType,
        PublicIP: PublicIpAddress,
        PrivateIP: PrivateIpAddress,
        AZ: Placement.AvailabilityZone
    }' --output table

# Instances launched in the last 24 hours
aws ec2 describe-instances \
    --query 'Reservations[].Instances[?LaunchTime>=`2024-01-01`].[InstanceId, LaunchTime, InstanceType]' \
    --output table

# Get instance Name tag from instance ID
aws ec2 describe-instances \
    --instance-ids i-0123456789abcdef0 \
    --query 'Reservations[].Instances[].Tags[?Key==`Name`].Value | []' \
    --output text

# Get security group IDs for running instances
aws ec2 describe-instances \
    --query 'Reservations[].Instances[?State.Name==`running`].SecurityGroups[].GroupId'

# Get instance with most recent launch time
aws ec2 describe-instances \
    --query 'Reservations[].Instances[] | max_by(@, &LaunchTime).{ID:InstanceId, Launched:LaunchTime}'

# Get instances by tag value
aws ec2 describe-instances \
    --query 'Reservations[].Instances[?Tags[?Key==`Name`].Value | [0]==`MyServer`].InstanceId'
```

### S3

```bash
# List bucket names only
aws s3api list-buckets --query 'Buckets[].Name'

# List buckets with creation date
aws s3api list-buckets \
    --query 'Buckets[].{Name:Name, Created:CreationDate}' \
    --output table

# Buckets with "prod" in the name
aws s3api list-buckets \
    --query 'Buckets[?contains(Name, `prod`)].[Name]' \
    --output text

# Count objects in a bucket (first 1000)
aws s3api list-objects-v2 --bucket my-bucket \
    --query 'length(Contents)'

# Get total size of all objects
aws s3api list-objects-v2 --bucket my-bucket \
    --query 'sum(Contents[].Size)'

# List objects with specific extension
aws s3api list-objects-v2 --bucket my-bucket \
    --query 'Contents[?ends_with(Key, `.jpg`)].Key'

# List objects larger than 1MB
aws s3api list-objects-v2 --bucket my-bucket \
    --query 'Contents[?Size > `1048576`].[Key, Size]'

# Objects starting with prefix
aws s3api list-objects-v2 --bucket my-bucket \
    --query 'Contents[?starts_with(Key, `images/`)].Key'

# Get most recently modified object
aws s3api list-objects-v2 --bucket my-bucket \
    --query 'Contents[] | max_by(@, &LastModified).{Key:Key, Modified:LastModified}'
```

### IAM

```bash
# List user names only
aws iam list-users --query 'Users[].UserName'

# List users with creation date
aws iam list-users \
    --query 'Users[].{Name:UserName, Created:CreateDate}' \
    --output table

# Users who haven't logged in (no password last used)
aws iam list-users \
    --query 'Users[?PasswordLastUsed==null].[UserName, CreateDate]' \
    --output table

# Users created after a specific date
aws iam list-users \
    --query 'Users[?CreateDate >= `2024-01-01T00:00:00Z`].[UserName, CreateDate]'

# Users with tags
aws iam list-users \
    --query 'Users[?length(Tags) > `0`].[UserName]'

# List attached policy ARNs for a user
aws iam list-attached-user-policies --user-name admin \
    --query 'AttachedPolicies[].PolicyArn'

# List policies attached to a user (names only)
aws iam list-attached-user-policies --user-name admin \
    --query 'AttachedPolicies[].PolicyName'

# Get policy version details
aws iam get-policy \
    --policy-arn arn:aws:iam::123456789012:policy/MyPolicy \
    --query 'Policy.[PolicyName, Arn, DefaultVersionId]'
```

### ELB / ALB

```bash
# List load balancers with DNS names
aws elbv2 describe-load-balancers \
    --query 'LoadBalancers[].{Name:LoadBalancerName, DNS:DNSName, Type:Type}' \
    --output table

# Target groups with health status
aws elbv2 describe-target-health --target-group-arn <arn> \
    --query 'TargetHealthDescriptions[].{ID:Target.Id, Port:Target.Port, Health:TargetHealth.State}'
```

### RDS

```bash
# List database instance identifiers
aws rds describe-db-instances \
    --query 'DBInstances[].DBInstanceIdentifier'

# List DB instances with status
aws rds describe-db-instances \
    --query 'DBInstances[].{ID:DBInstanceIdentifier, Engine:Engine, Status:DBInstanceStatus, Class:DBInstanceClass}' \
    --output table

# Get available databases
aws rds describe-db-instances \
    --query 'DBInstances[?DBInstanceStatus==`available`].[DBInstanceIdentifier, Engine]'

# Filter by engine type
aws rds describe-db-instances \
    --query 'DBInstances[?Engine==`mysql`].DBInstanceIdentifier'

# Get database endpoints
aws rds describe-db-instances \
    --query 'DBInstances[].[DBInstanceIdentifier, Endpoint.Address, Endpoint.Port]' \
    --output table

# Get endpoint for a specific DB
aws rds describe-db-instances \
    --db-instance-identifier my-db \
    --query 'DBInstances[0].Endpoint.[Address, Port]' \
    --output text
```

### Lambda

```bash
# List function names
aws lambda list-functions \
    --query 'Functions[].FunctionName'

# List functions with runtime and memory
aws lambda list-functions \
    --query 'Functions[].{Name:FunctionName, Runtime:Runtime, Memory:MemorySize, Timeout:Timeout}' \
    --output table

# Functions using Python
aws lambda list-functions \
    --query 'Functions[?starts_with(Runtime, `python`)].[FunctionName, Runtime]' \
    --output table

# Get functions with specific runtime
aws lambda list-functions \
    --query 'Functions[?Runtime==`python3.9`].FunctionName'

# Get function ARNs and memory size
aws lambda list-functions \
    --query 'Functions[].[FunctionName, FunctionArn, MemorySize]' \
    --output table
```

### CloudFormation

```bash
# List stack names
aws cloudformation list-stacks \
    --query 'StackSummaries[].StackName'

# List stacks with status (exclude deleted)
aws cloudformation list-stacks \
    --query 'StackSummaries[?StackStatus!=`DELETE_COMPLETE`].{Name:StackName, Status:StackStatus, Updated:LastUpdatedTime}' \
    --output table

# Get stacks by status
aws cloudformation list-stacks \
    --query 'StackSummaries[?StackStatus==`CREATE_COMPLETE`].StackName'

# Get stack outputs
aws cloudformation describe-stacks --stack-name my-stack \
    --query 'Stacks[0].Outputs[].[OutputKey, OutputValue]' \
    --output table

# Get a specific output value
aws cloudformation describe-stacks --stack-name my-stack \
    --query 'Stacks[0].Outputs[?OutputKey==`VpcId`].OutputValue | [0]' \
    --output text
```

## Gotchas and Tips

### Backticks for Literal Values

JMESPath uses backticks for literal values, not quotes:

```bash
# Correct — backticks for literal
--query 'Instances[?State==`running`]'

# Wrong — quotes are for JSON field names
--query 'Instances[?State=="running"]'
```

### Shell Quoting

The `--query` value should be in single quotes to prevent shell interpretation:

```bash
# Correct — single quotes protect backticks from shell
aws ec2 describe-instances --query 'Reservations[].Instances[?State.Name==`running`]'

# If you must use double quotes, escape backticks
aws ec2 describe-instances --query "Reservations[].Instances[?State.Name==\`running\`]"
```

### Null Values

Fields that don't exist return `null`. Use `not_null()` or filter them out:

```bash
# Skip instances without public IP
aws ec2 describe-instances \
    --query 'Reservations[].Instances[?PublicIpAddress!=null].[InstanceId, PublicIpAddress]'

# Provide fallback value
aws ec2 describe-instances \
    --query 'Reservations[].Instances[].{ID:InstanceId, IP:PublicIpAddress || `N/A`}'
```

### `| [0]` for Single Values from Nested Filters

Tag queries return arrays. Use `| [0]` to get the first (usually only) value:

```bash
# Without | [0] — returns an array
Tags[?Key==`Name`].Value
# ["my-instance"]

# With | [0] — returns the value directly
Tags[?Key==`Name`].Value | [0]
# "my-instance"
```

### Flatten `[]` to Remove Empty Arrays

When filtering at a parent level, non-matching parents return empty arrays. Append `[]` to clean up:

```bash
# Without flatten — contains empty arrays for non-matching reservations
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[?Tags[?Key==`Name` && Value==`my-app`]].InstanceId'
# [[], ["i-0123456789abcdef0"], []]

# With flatten — clean result
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[?Tags[?Key==`Name` && Value==`my-app`]].InstanceId[]'
# ["i-0123456789abcdef0"]
```

### Filter at Parent Level, Access Child Field

To find a parent object based on a nested value, apply the filter at the parent level:

```bash
# "Get the Instance ID for instances named 'my-app'"
# Filter at Instances level (not Tags level), then access InstanceId
aws ec2 describe-instances \
    --query "Reservations[*].Instances[? Tags[? Key=='Name' && Value=='my-app'] ].InstanceId[]"
```

This pattern: filter Instances by their Tags, then select InstanceId from the matching Instances.

### Using JMESPath in Python (boto3)

```python
import boto3, jmespath, json

ec2 = boto3.client('ec2')
response = ec2.describe_instances()

# Apply JMESPath filter to the response
expression = "Reservations[*].Instances[? Tags[? Key == 'Name' && Value == 'my-app'] ].InstanceId[]"
result = jmespath.search(expression, response)
print(json.dumps(result, indent=2))
```

Install: `pip install jmespath` (also bundled with boto3).

### Combining with --filter

`--filter` (server-side) + `--query` (client-side) is the most efficient pattern:

```bash
# Filter narrows results server-side, query formats client-side
aws ec2 describe-instances \
    --filters "Name=instance-state-name,Values=running" "Name=instance-type,Values=t3.*" \
    --query 'Reservations[].Instances[].{ID:InstanceId, Name:Tags[?Key==`Name`].Value|[0]}' \
    --output table
```

## Testing JMESPath Expressions

### jp CLI Tool

```bash
# Install jp (JMESPath terminal)
go install github.com/jmespath/jp/cmd/jp@latest
# or
pip install jmespath-terminal

# Test expressions against JSON
echo '{"foo": {"bar": "baz"}}' | jp 'foo.bar'
# "baz"

# Pipe AWS output to jp for testing
aws ec2 describe-instances --output json | jp 'Reservations[].Instances[].InstanceId'
```

### Online Playground

Test expressions at [jmespath.org](https://jmespath.org/) with sample JSON data before using them in AWS CLI commands.

## EKS-Specific Examples

### Find EKS Subnets

```bash
cluster="my-cluster"

# Get subnet IDs tagged for EKS cluster
aws ec2 describe-subnets \
    --query "Subnets[?Tags[?Key=='kubernetes.io/cluster/${cluster}']].SubnetId" \
    --output text

# With additional details
aws ec2 describe-subnets \
    --query "Subnets[?Tags[?Key=='kubernetes.io/cluster/${cluster}']].{ID:SubnetId, VPC:VpcId, AZ:AvailabilityZone, CIDR:CidrBlock}" \
    --output table
```

### Find Public EKS Subnets (for ALB)

```bash
cluster="my-cluster"

aws ec2 describe-subnets \
    --query "Subnets[?Tags[?Key=='kubernetes.io/cluster/${cluster}'] && Tags[?Key=='kubernetes.io/role/elb']].SubnetId" \
    --output text
```

### Find Private EKS Subnets (for internal LB)

```bash
cluster="my-cluster"

aws ec2 describe-subnets \
    --query "Subnets[?Tags[?Key=='kubernetes.io/cluster/${cluster}'] && Tags[?Key=='kubernetes.io/role/internal-elb']].SubnetId" \
    --output text
```

### EKS with Multiple Conditions

```bash
cluster="my-cluster"
environment="production"

aws ec2 describe-subnets \
    --query "Subnets[?Tags[?Key=='kubernetes.io/cluster/${cluster}'] && Tags[?Key=='Environment' && Value=='${environment}']].SubnetId"
```

### Use --filters Instead of Complex JMESPath (More Efficient)

```bash
cluster="my-cluster"

# Filters do server-side matching — faster for tag queries
aws ec2 describe-subnets \
    --filters "Name=tag:kubernetes.io/cluster/${cluster},Values=shared,owned" \
    --query "Subnets[].{ID:SubnetId, AZ:AvailabilityZone, CIDR:CidrBlock}" \
    --output table
```

## Shell Variable Quoting

When using shell variables in JMESPath, you need double quotes around `--query` so the shell expands variables, and single quotes inside for JMESPath string literals:

```bash
cluster="my-cluster"

# Correct: double quotes for shell expansion, single quotes for JMESPath strings
aws ec2 describe-subnets \
    --query "Subnets[?Tags[?Key=='kubernetes.io/cluster/${cluster}']]"

# Also correct: escaped double quotes inside
aws ec2 describe-subnets \
    --query "Subnets[?Tags[?Key==\"kubernetes.io/cluster/${cluster}\"]]"

# Wrong: single quotes prevent shell variable expansion
aws ec2 describe-subnets \
    --query 'Subnets[?Tags[?Key==`kubernetes.io/cluster/${cluster}`]]'
# ${cluster} is NOT expanded!
```

### Debug Query Expansion

```bash
# Use set -x to see the actual query sent
set -x
cluster="my-cluster"
aws ec2 describe-subnets \
    --query "Subnets[?Tags[?Key=='kubernetes.io/cluster/${cluster}']].SubnetId"
set +x
```

## Common Syntax Errors and Fixes

| Error | Wrong | Correct |
|-------|-------|---------|
| Backticks instead of quotes (with variables) | `` `kubernetes.io/${cluster}` `` | `'kubernetes.io/${cluster}'` |
| Missing `?` in filter | `[Tags[Key=='Name']]` | `[?Tags[?Key=='Name']]` |
| Single `=` for comparison | `[?Key='Name']` | `[?Key=='Name']` |
| Extra closing bracket | `[?Key=='Name']]]` | `[?Key=='Name']]` |
| Unescaped backticks in double-quoted query | `` --query "..`value`" `` | `--query "..\`value\`"` |

## Reusable Shell Functions

### EKS Subnet Discovery

```bash
get_eks_subnets() {
    local cluster_name=$1
    local output_format=${2:-text}

    aws ec2 describe-subnets \
        --query "Subnets[?Tags[?Key=='kubernetes.io/cluster/${cluster_name}']].SubnetId" \
        --output "$output_format"
}

# Usage
SUBNETS=$(get_eks_subnets my-cluster)
echo "EKS Subnets: $SUBNETS"
```

### Generic Tag Query

```bash
query_by_tag() {
    local resource_type=$1
    local tag_key=$2
    local tag_value=$3

    case $resource_type in
        subnets)
            aws ec2 describe-subnets \
                --query "Subnets[?Tags[?Key=='${tag_key}' && Value=='${tag_value}']].SubnetId" \
                --output text
            ;;
        instances)
            aws ec2 describe-instances \
                --query "Reservations[].Instances[?Tags[?Key=='${tag_key}' && Value=='${tag_value}']].InstanceId" \
                --output text
            ;;
        volumes)
            aws ec2 describe-volumes \
                --query "Volumes[?Tags[?Key=='${tag_key}' && Value=='${tag_value}']].VolumeId" \
                --output text
            ;;
    esac
}

# Usage
query_by_tag instances Environment production
query_by_tag subnets "kubernetes.io/cluster/my-cluster" shared
```

## Common Errors Table

| Error | Likely Cause | Fix |
|-------|-------------|-----|
| "Invalid JMESPath expression" | Syntax error in query | Check brackets, quotes, operators |
| "Function not found" | Typo in function name | Verify: `length`, `sort_by`, `max_by`, `contains` |
| "Type error" | Wrong type for comparison | Use backticks for numbers: `` `100` `` not `"100"` |
| Empty results `[]` | Filter too restrictive | Remove filters one by one to debug |
| "Unknown output type" | Bad `--output` value | Use: `json`, `text`, `table`, `yaml` |
| Shell variable not expanded | Single quotes around `--query` | Use double quotes when query contains `$variable` |

## Tips and Best Practices

1. **Use backticks for literal values** when the outer query is in single quotes: `` `running` ``
2. **Use single quotes inside double-quoted queries** when you need shell variable expansion: `'value'`
3. **Test queries incrementally** — build complex queries step by step
4. **Use `--filters` for server-side filtering** — faster than `--query` for large datasets
5. **Use `--output text`** for scripting — tab-separated, easy to pipe to `awk`/`cut`
6. **Use `|| 'N/A'`** for default values when fields might be null
7. **Use `| [0]`** after tag filters to unwrap the single-element array
8. **Combine `--filters` + `--query`** — filter narrows, query formats
