# AWS CloudFormation CLI Cheatsheet

AWS CloudFormation CLI commands for managing stacks, change sets, drift detection, and template operations.

## Stack Operations

### Create Stack

```bash
# Create stack from local template
aws cloudformation create-stack \
    --stack-name my-stack \
    --template-body file://template.yaml

# Create stack from S3 template
aws cloudformation create-stack \
    --stack-name my-stack \
    --template-url https://s3.amazonaws.com/my-bucket/template.yaml

# Create with parameters
aws cloudformation create-stack \
    --stack-name my-stack \
    --template-body file://template.yaml \
    --parameters ParameterKey=Env,ParameterValue=prod ParameterKey=InstanceType,ParameterValue=t3.medium

# Create with parameters from file
aws cloudformation create-stack \
    --stack-name my-stack \
    --template-body file://template.yaml \
    --parameters file://params.json

# Create with IAM capabilities (required for IAM resources)
aws cloudformation create-stack \
    --stack-name my-stack \
    --template-body file://template.yaml \
    --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM

# Create with all capabilities (IAM + transforms like SAM)
aws cloudformation create-stack \
    --stack-name my-stack \
    --template-body file://template.yaml \
    --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND

# Create with tags
aws cloudformation create-stack \
    --stack-name my-stack \
    --template-body file://template.yaml \
    --tags Key=Environment,Value=production Key=Team,Value=platform

# Create with rollback disabled (useful for debugging)
aws cloudformation create-stack \
    --stack-name my-stack \
    --template-body file://template.yaml \
    --disable-rollback

# Create with termination protection
aws cloudformation create-stack \
    --stack-name my-stack \
    --template-body file://template.yaml \
    --enable-termination-protection

# Create with notification ARN
aws cloudformation create-stack \
    --stack-name my-stack \
    --template-body file://template.yaml \
    --notification-arns arn:aws:sns:us-east-1:123456789012:stack-notifications
```

### Update Stack

```bash
# Update stack with new template
aws cloudformation update-stack \
    --stack-name my-stack \
    --template-body file://template.yaml

# Update with parameters
aws cloudformation update-stack \
    --stack-name my-stack \
    --template-body file://template.yaml \
    --parameters ParameterKey=Env,ParameterValue=staging

# Update keeping previous parameter values
aws cloudformation update-stack \
    --stack-name my-stack \
    --template-body file://template.yaml \
    --parameters ParameterKey=Env,UsePreviousValue=true ParameterKey=InstanceType,ParameterValue=t3.large

# Update with capabilities
aws cloudformation update-stack \
    --stack-name my-stack \
    --template-body file://template.yaml \
    --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
```

### Delete Stack

```bash
# Delete stack
aws cloudformation delete-stack --stack-name my-stack

# Delete stack and retain specific resources
aws cloudformation delete-stack \
    --stack-name my-stack \
    --retain-resources LogGroup S3Bucket

# Disable termination protection before deleting
aws cloudformation update-termination-protection \
    --no-enable-termination-protection \
    --stack-name my-stack

aws cloudformation delete-stack --stack-name my-stack
```

### Wait for Stack Operations

```bash
# Wait for stack creation to complete
aws cloudformation wait stack-create-complete --stack-name my-stack

# Wait for stack update to complete
aws cloudformation wait stack-update-complete --stack-name my-stack

# Wait for stack deletion to complete
aws cloudformation wait stack-delete-complete --stack-name my-stack

# Wait for stack to exist
aws cloudformation wait stack-exists --stack-name my-stack
```

## Describe and List

### List Stacks

```bash
# List all active stacks
aws cloudformation list-stacks \
    --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE

# List all stacks (including deleted)
aws cloudformation list-stacks

# List stacks with specific status
aws cloudformation list-stacks \
    --stack-status-filter CREATE_FAILED ROLLBACK_COMPLETE DELETE_FAILED

# List stack names only
aws cloudformation list-stacks \
    --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
    --query 'StackSummaries[].StackName' --output text
```

### Describe Stack

```bash
# Describe a stack
aws cloudformation describe-stacks --stack-name my-stack

# Get stack status
aws cloudformation describe-stacks --stack-name my-stack \
    --query 'Stacks[0].StackStatus' --output text

# Get stack outputs
aws cloudformation describe-stacks --stack-name my-stack \
    --query 'Stacks[0].Outputs' --output table

# Get specific output value
aws cloudformation describe-stacks --stack-name my-stack \
    --query 'Stacks[0].Outputs[?OutputKey==`VpcId`].OutputValue' --output text

# Get stack parameters
aws cloudformation describe-stacks --stack-name my-stack \
    --query 'Stacks[0].Parameters' --output table
```

### Stack Resources

```bash
# List all resources in a stack
aws cloudformation list-stack-resources --stack-name my-stack

# Describe a specific resource
aws cloudformation describe-stack-resource \
    --stack-name my-stack \
    --logical-resource-id MyEC2Instance

# Get physical resource ID
aws cloudformation describe-stack-resource \
    --stack-name my-stack \
    --logical-resource-id MyEC2Instance \
    --query 'StackResourceDetail.PhysicalResourceId' --output text
```

### Stack Events

```bash
# List stack events (most recent first)
aws cloudformation describe-stack-events --stack-name my-stack

# Get events for failed resources
aws cloudformation describe-stack-events --stack-name my-stack \
    --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`].[LogicalResourceId,ResourceStatusReason]' \
    --output table

# Watch stack events in real time (poll every 5 seconds)
while true; do
    aws cloudformation describe-stack-events --stack-name my-stack \
        --query 'StackEvents[0].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]' \
        --output text
    sleep 5
done
```

## Change Sets

### Create Change Set

```bash
# Create change set for an update
aws cloudformation create-change-set \
    --stack-name my-stack \
    --change-set-name my-changes \
    --template-body file://template.yaml \
    --capabilities CAPABILITY_IAM

# Create change set with parameters
aws cloudformation create-change-set \
    --stack-name my-stack \
    --change-set-name my-changes \
    --template-body file://template.yaml \
    --parameters ParameterKey=Env,ParameterValue=prod

# Create change set for a new stack (CREATE type)
aws cloudformation create-change-set \
    --stack-name new-stack \
    --change-set-name initial-create \
    --change-set-type CREATE \
    --template-body file://template.yaml
```

### Describe and Execute Change Set

```bash
# Describe change set (see what will change)
aws cloudformation describe-change-set \
    --stack-name my-stack \
    --change-set-name my-changes

# List changes only
aws cloudformation describe-change-set \
    --stack-name my-stack \
    --change-set-name my-changes \
    --query 'Changes[].ResourceChange.{Action:Action,Resource:LogicalResourceId,Type:ResourceType}' \
    --output table

# Execute change set (apply changes)
aws cloudformation execute-change-set \
    --stack-name my-stack \
    --change-set-name my-changes

# Delete change set without executing
aws cloudformation delete-change-set \
    --stack-name my-stack \
    --change-set-name my-changes

# List all change sets for a stack
aws cloudformation list-change-sets --stack-name my-stack
```

## Template Operations

### Validate Template

```bash
# Validate local template
aws cloudformation validate-template --template-body file://template.yaml

# Validate S3 template
aws cloudformation validate-template --template-url https://s3.amazonaws.com/bucket/template.yaml
```

### Get Template

```bash
# Get template from existing stack
aws cloudformation get-template --stack-name my-stack

# Get template body only
aws cloudformation get-template --stack-name my-stack \
    --query 'TemplateBody' --output text > recovered-template.yaml

# Get processed template (after transforms)
aws cloudformation get-template --stack-name my-stack --template-stage Processed

# Get template summary (parameters, resources, etc.)
aws cloudformation get-template-summary --template-body file://template.yaml

# Get template summary from a stack
aws cloudformation get-template-summary --stack-name my-stack
```

### Package Template (for nested stacks / SAM)

```bash
# Package local artifacts to S3 (uploads Lambda code, nested templates, etc.)
aws cloudformation package \
    --template-file template.yaml \
    --s3-bucket my-deployment-bucket \
    --output-template-file packaged.yaml

# Deploy packaged template
aws cloudformation deploy \
    --template-file packaged.yaml \
    --stack-name my-stack \
    --capabilities CAPABILITY_IAM
```

## Deploy (High-Level Command)

```bash
# Deploy (creates or updates stack)
aws cloudformation deploy \
    --template-file template.yaml \
    --stack-name my-stack

# Deploy with parameters
aws cloudformation deploy \
    --template-file template.yaml \
    --stack-name my-stack \
    --parameter-overrides Env=prod InstanceType=t3.medium

# Deploy with capabilities
aws cloudformation deploy \
    --template-file template.yaml \
    --stack-name my-stack \
    --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM

# Deploy with tags
aws cloudformation deploy \
    --template-file template.yaml \
    --stack-name my-stack \
    --tags Environment=production Team=platform

# Deploy with no-execute (creates change set only)
aws cloudformation deploy \
    --template-file template.yaml \
    --stack-name my-stack \
    --no-execute-changeset

# Deploy with rollback configuration
aws cloudformation deploy \
    --template-file template.yaml \
    --stack-name my-stack \
    --rollback-configuration "RollbackTriggers=[{Arn=arn:aws:cloudwatch:us-east-1:123456789012:alarm:HighErrors,Type=AWS::CloudWatch::Alarm}]"
```

## Drift Detection

```bash
# Detect drift on a stack
aws cloudformation detect-stack-drift --stack-name my-stack

# Check drift detection status
aws cloudformation describe-stack-drift-detection-status \
    --stack-drift-detection-id <detection-id>

# Describe drifted resources
aws cloudformation describe-stack-resource-drifts \
    --stack-name my-stack \
    --stack-resource-drift-status-filters MODIFIED DELETED

# Detect drift on a specific resource
aws cloudformation detect-stack-resource-drift \
    --stack-name my-stack \
    --logical-resource-id MySecurityGroup
```

## Stack Sets (Multi-Account/Region)

```bash
# Create stack set
aws cloudformation create-stack-set \
    --stack-set-name my-stack-set \
    --template-body file://template.yaml \
    --capabilities CAPABILITY_IAM

# Add stack instances (deploy to accounts/regions)
aws cloudformation create-stack-instances \
    --stack-set-name my-stack-set \
    --accounts 111111111111 222222222222 \
    --regions us-east-1 eu-west-1

# Update stack set
aws cloudformation update-stack-set \
    --stack-set-name my-stack-set \
    --template-body file://template.yaml

# List stack sets
aws cloudformation list-stack-sets

# Describe stack set
aws cloudformation describe-stack-set --stack-set-name my-stack-set

# List stack instances
aws cloudformation list-stack-instances --stack-set-name my-stack-set

# Delete stack instances
aws cloudformation delete-stack-instances \
    --stack-set-name my-stack-set \
    --accounts 111111111111 \
    --regions us-east-1 \
    --no-retain-stacks

# Delete stack set (must delete all instances first)
aws cloudformation delete-stack-set --stack-set-name my-stack-set
```

## Import Existing Resources

```bash
# Create change set for import
aws cloudformation create-change-set \
    --stack-name my-stack \
    --change-set-name import-resources \
    --change-set-type IMPORT \
    --template-body file://template.yaml \
    --resources-to-import "[{\"ResourceType\":\"AWS::S3::Bucket\",\"LogicalResourceId\":\"MyBucket\",\"ResourceIdentifier\":{\"BucketName\":\"my-existing-bucket\"}}]"

# Execute the import
aws cloudformation execute-change-set \
    --stack-name my-stack \
    --change-set-name import-resources
```

## Stack Policies

```bash
# Set stack policy (protect resources from updates)
aws cloudformation set-stack-policy \
    --stack-name my-stack \
    --stack-policy-body file://stack-policy.json

# Get current stack policy
aws cloudformation get-stack-policy --stack-name my-stack

# Override policy during update (temporary)
aws cloudformation update-stack \
    --stack-name my-stack \
    --template-body file://template.yaml \
    --stack-policy-during-update-body file://override-policy.json
```

### Example Stack Policy

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "Update:*",
      "Principal": "*",
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": "Update:Replace",
      "Principal": "*",
      "Resource": "LogicalResourceId/ProductionDatabase"
    }
  ]
}
```

## Parameters File Format

```json
[
  {
    "ParameterKey": "Environment",
    "ParameterValue": "production"
  },
  {
    "ParameterKey": "InstanceType",
    "ParameterValue": "t3.medium"
  },
  {
    "ParameterKey": "VpcId",
    "ParameterValue": "vpc-0123456789abcdef0"
  }
]
```

## Troubleshooting

### Stack Stuck in DELETE_FAILED

```bash
# Find resources that failed to delete
aws cloudformation describe-stack-events --stack-name my-stack \
    --query 'StackEvents[?ResourceStatus==`DELETE_FAILED`].[LogicalResourceId,ResourceStatusReason]' \
    --output table

# Retry delete, skipping problematic resources
aws cloudformation delete-stack \
    --stack-name my-stack \
    --retain-resources FailedResource1 FailedResource2
```

### Stack Stuck in UPDATE_ROLLBACK_FAILED

```bash
# Continue rollback, skipping problematic resources
aws cloudformation continue-update-rollback \
    --stack-name my-stack \
    --resources-to-skip LogicalResourceId1 LogicalResourceId2
```

### Find Why a Resource Failed

```bash
# Get failure reason from events
aws cloudformation describe-stack-events --stack-name my-stack \
    --query 'StackEvents[?ResourceStatus==`CREATE_FAILED` || ResourceStatus==`UPDATE_FAILED`].{Resource:LogicalResourceId,Reason:ResourceStatusReason,Time:Timestamp}' \
    --output table
```

### Check Resource Limits

```bash
# Describe account limits
aws cloudformation describe-account-limits
```

## Quick Reference

| Action | Command |
|--------|---------|
| Create stack | `aws cloudformation create-stack --stack-name X --template-body file://t.yaml` |
| Update stack | `aws cloudformation update-stack --stack-name X --template-body file://t.yaml` |
| Delete stack | `aws cloudformation delete-stack --stack-name X` |
| Deploy (create/update) | `aws cloudformation deploy --stack-name X --template-file t.yaml` |
| Describe stack | `aws cloudformation describe-stacks --stack-name X` |
| List stacks | `aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE` |
| Stack events | `aws cloudformation describe-stack-events --stack-name X` |
| Stack outputs | `aws cloudformation describe-stacks --stack-name X --query 'Stacks[0].Outputs'` |
| Validate template | `aws cloudformation validate-template --template-body file://t.yaml` |
| Create change set | `aws cloudformation create-change-set --stack-name X --change-set-name Y` |
| Detect drift | `aws cloudformation detect-stack-drift --stack-name X` |
| Wait for completion | `aws cloudformation wait stack-create-complete --stack-name X` |
| Get template | `aws cloudformation get-template --stack-name X` |
| Package (SAM/nested) | `aws cloudformation package --template-file t.yaml --s3-bucket bucket` |
