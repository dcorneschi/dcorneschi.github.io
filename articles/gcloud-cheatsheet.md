# gcloud CLI Cheatsheet

Google Cloud CLI commands for managing GCP resources — authentication, compute, storage, functions, Cloud Run, GKE, SQL, IAM, networking, and monitoring.

## Authentication and Configuration

### Initial Setup

```bash
# Install gcloud CLI
curl https://sdk.cloud.google.com | bash

# Initialize gcloud
gcloud init

# Authenticate with Google Cloud
gcloud auth login

# Set application default credentials
gcloud auth application-default login

# List authenticated accounts
gcloud auth list

# Set active account
gcloud config set account [ACCOUNT]

# Revoke credentials
gcloud auth revoke [ACCOUNT]
```

### Configuration Management

```bash
# List configurations
gcloud config configurations list

# Create new configuration
gcloud config configurations create [CONFIG_NAME]

# Activate configuration
gcloud config configurations activate [CONFIG_NAME]

# Set default project
gcloud config set project [PROJECT_ID]

# Set default region
gcloud config set compute/region [REGION]

# Set default zone
gcloud config set compute/zone [ZONE]

# View current configuration
gcloud config list
```

## Projects

```bash
# List projects
gcloud projects list

# Create project
gcloud projects create [PROJECT_ID] --name="[PROJECT_NAME]"

# Delete project
gcloud projects delete [PROJECT_ID]

# Set current project
gcloud config set project [PROJECT_ID]

# Get project info
gcloud projects describe [PROJECT_ID]

# Enable API
gcloud services enable [API_NAME]

# List enabled APIs
gcloud services list --enabled

# Disable API
gcloud services disable [API_NAME]
```

## Compute Engine

### Instances

```bash
# List instances
gcloud compute instances list

# Create instance
gcloud compute instances create [INSTANCE_NAME] \
    --zone=[ZONE] \
    --machine-type=[MACHINE_TYPE] \
    --image-family=[IMAGE_FAMILY] \
    --image-project=[IMAGE_PROJECT]

# Delete instance
gcloud compute instances delete [INSTANCE_NAME] --zone=[ZONE]

# Start instance
gcloud compute instances start [INSTANCE_NAME] --zone=[ZONE]

# Stop instance
gcloud compute instances stop [INSTANCE_NAME] --zone=[ZONE]

# Reset instance
gcloud compute instances reset [INSTANCE_NAME] --zone=[ZONE]

# SSH into instance
gcloud compute ssh [INSTANCE_NAME] --zone=[ZONE]

# Copy files to instance
gcloud compute scp [LOCAL_FILE] [INSTANCE_NAME]:[REMOTE_PATH] --zone=[ZONE]

# Copy files from instance
gcloud compute scp [INSTANCE_NAME]:[REMOTE_PATH] [LOCAL_FILE] --zone=[ZONE]

# Get instance details
gcloud compute instances describe [INSTANCE_NAME] --zone=[ZONE]
```

### Machine Types

```bash
# List machine types
gcloud compute machine-types list --zones=[ZONE]

# List machine types in region
gcloud compute machine-types list --filter="zone:([REGION])"
```

### Images

```bash
# List images
gcloud compute images list

# Create image from instance
gcloud compute images create [IMAGE_NAME] \
    --source-disk=[DISK_NAME] \
    --source-disk-zone=[ZONE]

# Delete image
gcloud compute images delete [IMAGE_NAME]
```

### Disks

```bash
# List disks
gcloud compute disks list

# Create disk
gcloud compute disks create [DISK_NAME] \
    --size=[SIZE_GB] \
    --zone=[ZONE] \
    --type=[DISK_TYPE]

# Delete disk
gcloud compute disks delete [DISK_NAME] --zone=[ZONE]

# Snapshot disk
gcloud compute disks snapshot [DISK_NAME] \
    --snapshot-names=[SNAPSHOT_NAME] \
    --zone=[ZONE]
```

### Snapshots

```bash
# List snapshots
gcloud compute snapshots list

# Delete snapshot
gcloud compute snapshots delete [SNAPSHOT_NAME]
```

## App Engine

```bash
# Deploy application
gcloud app deploy

# Deploy specific version
gcloud app deploy --version=[VERSION]

# View application
gcloud app browse

# Stream logs
gcloud app logs tail

# List versions
gcloud app versions list

# Delete version
gcloud app versions delete [VERSION]

# Set traffic allocation
gcloud app services set-traffic [SERVICE] --splits=[VERSION]=1.0

# Scale service
gcloud app versions migrate [VERSION]
```

## Cloud Storage

### Buckets

```bash
# List buckets
gsutil ls

# Create bucket
gsutil mb gs://[BUCKET_NAME]

# Delete bucket
gsutil rb gs://[BUCKET_NAME]

# Delete bucket with contents
gsutil rm -r gs://[BUCKET_NAME]
```

### Objects

```bash
# List objects in bucket
gsutil ls gs://[BUCKET_NAME]

# Copy file to bucket
gsutil cp [LOCAL_FILE] gs://[BUCKET_NAME]/

# Copy file from bucket
gsutil cp gs://[BUCKET_NAME]/[OBJECT] [LOCAL_PATH]

# Copy directory recursively
gsutil cp -r [LOCAL_DIR] gs://[BUCKET_NAME]/

# Sync directory with bucket
gsutil rsync -r [LOCAL_DIR] gs://[BUCKET_NAME]/

# Delete object
gsutil rm gs://[BUCKET_NAME]/[OBJECT]

# Delete all objects in bucket
gsutil rm gs://[BUCKET_NAME]/**

# Make object public
gsutil acl ch -u AllUsers:R gs://[BUCKET_NAME]/[OBJECT]

# Set bucket permissions
gsutil iam ch user:[EMAIL]:objectViewer gs://[BUCKET_NAME]
```

### Object Lifecycle

```bash
# Set lifecycle policy
gsutil lifecycle set [LIFECYCLE_CONFIG] gs://[BUCKET_NAME]

# Get lifecycle policy
gsutil lifecycle get gs://[BUCKET_NAME]
```

## Cloud Functions

```bash
# Deploy function
gcloud functions deploy [FUNCTION_NAME] \
    --runtime=[RUNTIME] \
    --trigger-http \
    --entry-point=[ENTRY_POINT] \
    --source=[SOURCE_PATH]

# Deploy with trigger
gcloud functions deploy [FUNCTION_NAME] \
    --runtime=[RUNTIME] \
    --trigger-topic=[TOPIC_NAME] \
    --entry-point=[ENTRY_POINT]

# List functions
gcloud functions list

# Describe function
gcloud functions describe [FUNCTION_NAME]

# Delete function
gcloud functions delete [FUNCTION_NAME]

# Call function
gcloud functions call [FUNCTION_NAME] --data='{"key":"value"}'

# View logs
gcloud functions logs read [FUNCTION_NAME]
```

## Cloud Run

```bash
# Deploy service
gcloud run deploy [SERVICE_NAME] \
    --image=[IMAGE_URL] \
    --platform=managed \
    --region=[REGION] \
    --allow-unauthenticated

# List services
gcloud run services list

# Describe service
gcloud run services describe [SERVICE_NAME] --region=[REGION]

# Delete service
gcloud run services delete [SERVICE_NAME] --region=[REGION]

# Update service
gcloud run services update [SERVICE_NAME] \
    --image=[NEW_IMAGE_URL] \
    --region=[REGION]

# Set traffic allocation
gcloud run services update-traffic [SERVICE_NAME] \
    --to-revisions=[REVISION]=100 \
    --region=[REGION]
```

## Kubernetes Engine (GKE)

### Clusters

```bash
# Create cluster
gcloud container clusters create [CLUSTER_NAME] \
    --zone=[ZONE] \
    --num-nodes=[NUM_NODES]

# List clusters
gcloud container clusters list

# Get cluster credentials
gcloud container clusters get-credentials [CLUSTER_NAME] --zone=[ZONE]

# Delete cluster
gcloud container clusters delete [CLUSTER_NAME] --zone=[ZONE]

# Resize cluster
gcloud container clusters resize [CLUSTER_NAME] \
    --num-nodes=[NUM_NODES] \
    --zone=[ZONE]
```

### Node Pools

```bash
# List node pools
gcloud container node-pools list --cluster=[CLUSTER_NAME] --zone=[ZONE]

# Create node pool
gcloud container node-pools create [POOL_NAME] \
    --cluster=[CLUSTER_NAME] \
    --zone=[ZONE] \
    --num-nodes=[NUM_NODES]

# Delete node pool
gcloud container node-pools delete [POOL_NAME] \
    --cluster=[CLUSTER_NAME] \
    --zone=[ZONE]
```

## Cloud SQL

### Instances

```bash
# List instances
gcloud sql instances list

# Create instance
gcloud sql instances create [INSTANCE_NAME] \
    --database-version=[VERSION] \
    --tier=[TIER] \
    --region=[REGION]

# Delete instance
gcloud sql instances delete [INSTANCE_NAME]

# Connect to instance
gcloud sql connect [INSTANCE_NAME] --user=[USER]

# Create database
gcloud sql databases create [DATABASE_NAME] --instance=[INSTANCE_NAME]

# Create user
gcloud sql users create [USERNAME] \
    --instance=[INSTANCE_NAME] \
    --password=[PASSWORD]
```

### Backups

```bash
# Create backup
gcloud sql backups create --instance=[INSTANCE_NAME]

# List backups
gcloud sql backups list --instance=[INSTANCE_NAME]

# Restore from backup
gcloud sql backups restore [BACKUP_ID] --restore-instance=[INSTANCE_NAME]
```

## Identity and Access Management (IAM)

### Service Accounts

```bash
# List service accounts
gcloud iam service-accounts list

# Create service account
gcloud iam service-accounts create [SA_NAME] \
    --display-name="[DISPLAY_NAME]"

# Delete service account
gcloud iam service-accounts delete [SA_EMAIL]

# Create key for service account
gcloud iam service-accounts keys create [KEY_FILE] \
    --iam-account=[SA_EMAIL]

# List keys for service account
gcloud iam service-accounts keys list --iam-account=[SA_EMAIL]
```

### IAM Policies

```bash
# Get IAM policy
gcloud projects get-iam-policy [PROJECT_ID]

# Add IAM binding
gcloud projects add-iam-policy-binding [PROJECT_ID] \
    --member="user:[EMAIL]" \
    --role="[ROLE]"

# Remove IAM binding
gcloud projects remove-iam-policy-binding [PROJECT_ID] \
    --member="user:[EMAIL]" \
    --role="[ROLE]"

# List available roles
gcloud iam roles list

# Describe role
gcloud iam roles describe [ROLE]
```

## Networking

### VPC Networks

```bash
# List networks
gcloud compute networks list

# Create network
gcloud compute networks create [NETWORK_NAME] --subnet-mode=[MODE]

# Delete network
gcloud compute networks delete [NETWORK_NAME]

# List subnets
gcloud compute networks subnets list

# Create subnet
gcloud compute networks subnets create [SUBNET_NAME] \
    --network=[NETWORK_NAME] \
    --range=[IP_RANGE] \
    --region=[REGION]
```

### Firewall Rules

```bash
# List firewall rules
gcloud compute firewall-rules list

# Create firewall rule
gcloud compute firewall-rules create [RULE_NAME] \
    --allow=[PROTOCOL]:[PORT] \
    --source-ranges=[IP_RANGE] \
    --target-tags=[TAG]

# Delete firewall rule
gcloud compute firewall-rules delete [RULE_NAME]
```

### Load Balancers

```bash
# Create HTTP load balancer
gcloud compute url-maps create [MAP_NAME] \
    --default-service=[BACKEND_SERVICE]

# Create backend service
gcloud compute backend-services create [SERVICE_NAME] \
    --protocol=[PROTOCOL] \
    --global
```

## Monitoring and Logging

### Cloud Logging

```bash
# Read logs
gcloud logging read "[FILTER]" --limit=[LIMIT] --format="[FORMAT]"

# Stream logs (requires beta)
gcloud beta logging tail "[FILTER]"

# Write log entry
gcloud logging write [LOG_NAME] "[MESSAGE]" --severity=[SEVERITY]

# List logs
gcloud logging logs list
```

### Cloud Monitoring

```bash
# Create alert policy
gcloud alpha monitoring policies create --policy-from-file=[POLICY_FILE]
```

## Deployment Manager

```bash
# Deploy template
gcloud deployment-manager deployments create [DEPLOYMENT_NAME] \
    --config=[CONFIG_FILE]

# List deployments
gcloud deployment-manager deployments list

# Update deployment
gcloud deployment-manager deployments update [DEPLOYMENT_NAME] \
    --config=[CONFIG_FILE]

# Delete deployment
gcloud deployment-manager deployments delete [DEPLOYMENT_NAME]

# Preview deployment
gcloud deployment-manager deployments create [DEPLOYMENT_NAME] \
    --config=[CONFIG_FILE] \
    --preview
```

## Billing

```bash
# List billing accounts
gcloud beta billing accounts list

# Link project to billing account
gcloud beta billing projects link [PROJECT_ID] \
    --billing-account=[ACCOUNT_ID]

# Get project billing info
gcloud beta billing projects describe [PROJECT_ID]
```

## Configuration and Info

### General Info

```bash
# Get gcloud info
gcloud info

# Check component status
gcloud components list

# Update components
gcloud components update

# Install component
gcloud components install [COMPONENT]

# List regions
gcloud compute regions list

# List zones
gcloud compute zones list

# List available APIs
gcloud services list --available
```

### Quotas

```bash
# List quotas
gcloud compute project-info describe --project=[PROJECT_ID]

# Request quota increase
gcloud alpha compute project-info describe --project=[PROJECT_ID]
```

## Common Environment Variables

```bash
export GOOGLE_APPLICATION_CREDENTIALS="path/to/service-account-key.json"
export GOOGLE_CLOUD_PROJECT="your-project-id"
export CLOUDSDK_CORE_PROJECT="your-project-id"
export CLOUDSDK_COMPUTE_REGION="us-central1"
export CLOUDSDK_COMPUTE_ZONE="us-central1-a"
```

## Useful Flags

### Global Flags

| Flag | Purpose |
|------|---------|
| `--project=[PROJECT_ID]` | Specify project |
| `--zone=[ZONE]` | Specify zone |
| `--region=[REGION]` | Specify region |
| `--format=[FORMAT]` | Output format (json, yaml, table, csv) |
| `--filter=[EXPRESSION]` | Filter resources |
| `--sort-by=[FIELD]` | Sort output |
| `--limit=[NUMBER]` | Limit number of results |
| `--quiet` | Disable interactive prompts |
| `--verbosity=[LEVEL]` | Set verbosity (debug, info, warning, error, critical) |

### Common Formats

```bash
# Custom table format
gcloud compute instances list --format="table(name,status,zone)"

# Extract specific field
gcloud compute instances list --format="value(name)"

# JSON output
gcloud compute instances list --format="json"

# YAML output
gcloud compute instances list --format="yaml"
```

## Common Use Cases

### Quick Instance Creation

```bash
# Create a simple VM
gcloud compute instances create my-vm \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --boot-disk-size=20GB \
    --tags=http-server,https-server
```

### Deploy Static Website to Cloud Storage

```bash
# Create bucket
gsutil mb gs://my-static-website

# Upload files
gsutil cp -r ./website/* gs://my-static-website/

# Make bucket public
gsutil iam ch allUsers:objectViewer gs://my-static-website

# Enable website configuration
gsutil web set -m index.html -e 404.html gs://my-static-website
```

### Quick Function Deployment

```bash
# Deploy HTTP function
gcloud functions deploy hello-world \
    --runtime=python39 \
    --trigger-http \
    --allow-unauthenticated \
    --entry-point=hello_world \
    --source=.
```

## Troubleshooting

```bash
# Check authentication
gcloud auth list

# Check current project
gcloud config get-value project

# Check quotas
gcloud compute project-info describe --project=[PROJECT_ID]

# Check enabled APIs
gcloud services list --enabled

# Debug with verbose output
gcloud [COMMAND] --verbosity=debug

# Clear credentials
gcloud auth revoke --all

# Reset configuration
gcloud config configurations delete default
gcloud init
```

## Tips

- Use `gcloud interactive` for interactive shell with auto-completion
- Use `gcloud alpha` and `gcloud beta` for experimental features
- Use `--dry-run` flag when available to preview changes
- Use `--async` flag for long-running operations
- Set up shell completion: `source "$(gcloud info --format='value(installation.sdk_root)')/completion.bash.inc"`
