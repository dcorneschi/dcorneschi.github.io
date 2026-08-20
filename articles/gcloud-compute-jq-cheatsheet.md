# GCP Compute Engine with jq Cheatsheet

Commands for managing Google Cloud Compute Engine instances and processing their JSON output with jq.

## Basic Instance Commands

### List All Instances

```bash
# Basic list
gcloud compute instances list

# JSON output for jq processing
gcloud compute instances list --format=json
```

### Create Instance

```bash
# Create basic instance
gcloud compute instances create my-instance \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --image-family=debian-11 \
    --image-project=debian-cloud

# Create with JSON output
gcloud compute instances create my-instance \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --format=json
```

### Instance Management

```bash
# Start instance
gcloud compute instances start INSTANCE_NAME --zone=ZONE

# Stop instance
gcloud compute instances stop INSTANCE_NAME --zone=ZONE

# Reset instance
gcloud compute instances reset INSTANCE_NAME --zone=ZONE

# Delete instance
gcloud compute instances delete INSTANCE_NAME --zone=ZONE
```

## Instance Listing with jq

### Basic Instance Information

```bash
# Get instance names only
gcloud compute instances list --format=json | jq -r '.[].name'

# Get instance names and zones
gcloud compute instances list --format=json | jq -r '.[] | "\(.name) - \(.zone | split("/")[-1])"'

# Get instance names, zones, and status
gcloud compute instances list --format=json | jq -r '.[] | "\(.name) | \(.zone | split("/")[-1]) | \(.status)"'

# Create a table format
gcloud compute instances list --format=json | jq -r '["NAME", "ZONE", "MACHINE_TYPE", "STATUS"], (.[] | [.name, (.zone | split("/")[-1]), (.machineType | split("/")[-1]), .status]) | @tsv'
```

### Instance Counts

```bash
# Total number of instances
gcloud compute instances list --format=json | jq 'length'

# Count by status
gcloud compute instances list --format=json | jq 'group_by(.status) | map({status: .[0].status, count: length})'

# Count by zone
gcloud compute instances list --format=json | jq 'group_by(.zone) | map({zone: (.[0].zone | split("/")[-1]), count: length})'

# Count by machine type
gcloud compute instances list --format=json | jq 'group_by(.machineType) | map({machine_type: (.[0].machineType | split("/")[-1]), count: length})'
```

## Instance Details with jq

### Get Specific Instance Details

```bash
# Get single instance details
gcloud compute instances describe INSTANCE_NAME --zone=ZONE --format=json

# Extract specific fields
gcloud compute instances describe INSTANCE_NAME --zone=ZONE --format=json | jq '{name, status, machineType, creationTimestamp}'

# Get creation date in readable format
gcloud compute instances describe INSTANCE_NAME --zone=ZONE --format=json | jq -r '.creationTimestamp | strptime("%Y-%m-%dT%H:%M:%S") | strftime("%Y-%m-%d %H:%M:%S")'
```

### Instance IPs and Network Info

```bash
# Get internal IP
gcloud compute instances describe INSTANCE_NAME --zone=ZONE --format=json | jq -r '.networkInterfaces[0].networkIP'

# Get external IP
gcloud compute instances describe INSTANCE_NAME --zone=ZONE --format=json | jq -r '.networkInterfaces[0].accessConfigs[0].natIP // "No external IP"'

# Get all network interfaces
gcloud compute instances describe INSTANCE_NAME --zone=ZONE --format=json | jq '.networkInterfaces[] | {name, networkIP, network: (.network | split("/")[-1])}'

# Get network and subnet info
gcloud compute instances list --format=json | jq -r '.[] | "\(.name): \(.networkInterfaces[0].networkIP) (\(.networkInterfaces[0].network | split("/")[-1]))"'
```

## Instance Filtering

### Filter by Status

```bash
# Get running instances
gcloud compute instances list --format=json | jq '.[] | select(.status == "RUNNING")'

# Get stopped instances
gcloud compute instances list --format=json | jq '.[] | select(.status == "TERMINATED")'

# Get instances that are not running
gcloud compute instances list --format=json | jq '.[] | select(.status != "RUNNING") | {name, status, zone: (.zone | split("/")[-1])}'
```

### Filter by Zone/Region

```bash
# Get instances in specific zone
gcloud compute instances list --format=json | jq '.[] | select(.zone | contains("us-central1-a"))'

# Get instances in specific region
gcloud compute instances list --format=json | jq '.[] | select(.zone | contains("us-central1"))'

# Group instances by region
gcloud compute instances list --format=json | jq 'group_by(.zone | split("/")[-1] | split("-")[0:2] | join("-")) | map({region: (.[0].zone | split("/")[-1] | split("-")[0:2] | join("-")), instances: map(.name)})'
```

### Filter by Machine Type

```bash
# Get instances with specific machine type
gcloud compute instances list --format=json | jq '.[] | select(.machineType | contains("e2-medium"))'

# Get instances with CPU count > 2
gcloud compute instances list --format=json | jq '.[] | select(.machineType | contains("e2-standard")) | select(.machineType | split("-")[-1] | tonumber > 2)'

# Group by machine family
gcloud compute instances list --format=json | jq 'group_by(.machineType | split("/")[-1] | split("-")[0]) | map({family: (.[0].machineType | split("/")[-1] | split("-")[0]), count: length})'
```

### Filter by Tags and Labels

```bash
# Get instances with specific tag
gcloud compute instances list --format=json | jq '.[] | select(.tags.items[]? == "web-server")'

# Get instances with specific label
gcloud compute instances list --format=json | jq '.[] | select(.labels.env == "production")'

# List all unique tags
gcloud compute instances list --format=json | jq '[.[].tags.items[]?] | unique'

# List all unique labels
gcloud compute instances list --format=json | jq '[.[].labels | to_entries[] | .key] | unique'
```

## Instance Status and Monitoring

### Instance Uptime and Age

```bash
# Calculate instance age in days
gcloud compute instances list --format=json | jq -r '.[] | "\(.name): \((now - (.creationTimestamp | strptime("%Y-%m-%dT%H:%M:%S%Z") | mktime)) / 86400 | floor) days old"'

# Get instances older than 30 days
gcloud compute instances list --format=json | jq '.[] | select((now - (.creationTimestamp | strptime("%Y-%m-%dT%H:%M:%S%Z") | mktime)) / 86400 > 30) | {name, age_days: ((now - (.creationTimestamp | strptime("%Y-%m-%dT%H:%M:%S%Z") | mktime)) / 86400 | floor)}'

# Get last start time
gcloud compute instances describe INSTANCE_NAME --zone=ZONE --format=json | jq -r '.lastStartTimestamp | strptime("%Y-%m-%dT%H:%M:%S") | strftime("%Y-%m-%d %H:%M:%S")'
```

### Instance Resources

```bash
# Get CPU and memory info
gcloud compute instances list --format=json | jq '.[] | {name, machine_type: (.machineType | split("/")[-1]), cpu_platform: .cpuPlatform}'

# Extract CPU count from machine type
gcloud compute instances list --format=json | jq '.[] | {name, cpus: (if (.machineType | split("/")[-1] | contains("custom")) then (.machineType | split("/")[-1] | split("-")[1] | tonumber) else (.machineType | split("/")[-1] | split("-")[-1] | tonumber) end)}'
```

## Instance Networking

### Network Analysis

```bash
# Get all internal IPs
gcloud compute instances list --format=json | jq -r '.[] | "\(.name): \(.networkInterfaces[0].networkIP)"'

# Get instances with external IPs
gcloud compute instances list --format=json | jq '.[] | select(.networkInterfaces[0].accessConfigs[0].natIP) | {name, external_ip: .networkInterfaces[0].accessConfigs[0].natIP}'

# Get instances without external IPs
gcloud compute instances list --format=json | jq '.[] | select(.networkInterfaces[0].accessConfigs[0].natIP == null) | .name'

# Network interface details
gcloud compute instances list --format=json | jq '.[] | {name, network_info: [.networkInterfaces[] | {network: (.network | split("/")[-1]), subnet: (.subnetwork | split("/")[-1]), internal_ip: .networkIP, external_ip: (.accessConfigs[0].natIP // "none")}]}'
```

### Security and Firewall

```bash
# Get instances with specific network tags
gcloud compute instances list --format=json | jq '.[] | select(.tags.items | index("http-server")) | {name, tags: .tags.items}'

# Check service account assignments
gcloud compute instances list --format=json | jq '.[] | {name, service_account: (.serviceAccounts[0].email // "default")}'

# Get instances allowing HTTP traffic
gcloud compute instances list --format=json | jq '.[] | select(.tags.items | index("http-server")) | .name'
```

## Instance Disks and Storage

### Boot Disk Information

```bash
# Get boot disk details
gcloud compute instances list --format=json | jq '.[] | {name, boot_disk: .disks[] | select(.boot == true) | {disk_name: (.source | split("/")[-1]), size_gb: .diskSizeGb, type: (.type | split("/")[-1])}}'

# Get disk sizes
gcloud compute instances list --format=json | jq '.[] | {name, total_disk_gb: ([.disks[].diskSizeGb | tonumber] | add)}'

# Get instances with SSD disks
gcloud compute instances list --format=json | jq '.[] | select(.disks[] | .type | contains("pd-ssd")) | {name, ssd_disks: [.disks[] | select(.type | contains("pd-ssd")) | .source | split("/")[-1]]}'
```

### Disk Analysis

```bash
# Count disks per instance
gcloud compute instances list --format=json | jq '.[] | {name, disk_count: (.disks | length)}'

# Get persistent disk info
gcloud compute instances describe INSTANCE_NAME --zone=ZONE --format=json | jq '.disks[] | {device_name: .deviceName, source: (.source | split("/")[-1]), auto_delete: .autoDelete, boot: .boot}'
```

## Instance Metadata

### Metadata Extraction

```bash
# Get all metadata
gcloud compute instances describe INSTANCE_NAME --zone=ZONE --format=json | jq '.metadata'

# Get specific metadata key
gcloud compute instances describe INSTANCE_NAME --zone=ZONE --format=json | jq -r '.metadata.items[] | select(.key == "startup-script") | .value'

# List all metadata keys
gcloud compute instances describe INSTANCE_NAME --zone=ZONE --format=json | jq -r '.metadata.items[]?.key'

# Get SSH keys
gcloud compute instances describe INSTANCE_NAME --zone=ZONE --format=json | jq -r '.metadata.items[] | select(.key == "ssh-keys") | .value'
```

### Custom Metadata

```bash
# Find instances with specific metadata
gcloud compute instances list --format=json | jq '.[] | select(.metadata.items[]?.key == "environment")'

# Extract custom labels
gcloud compute instances list --format=json | jq '.[] | {name, labels: (.labels // {})}'
```

## Bulk Operations

### Batch Information Gathering

```bash
# Get summary of all instances
gcloud compute instances list --format=json | jq '{
  total_instances: length,
  running: [.[] | select(.status == "RUNNING")] | length,
  stopped: [.[] | select(.status == "TERMINATED")] | length,
  zones: [.[].zone | split("/")[-1]] | unique,
  machine_types: [.[].machineType | split("/")[-1]] | unique
}'

# Generate CSV report
gcloud compute instances list --format=json | jq -r '["Name", "Zone", "Status", "Machine Type", "Internal IP", "External IP"], (.[] | [.name, (.zone | split("/")[-1]), .status, (.machineType | split("/")[-1]), .networkInterfaces[0].networkIP, (.networkInterfaces[0].accessConfigs[0].natIP // "")]) | @csv'

# Cost analysis preparation
gcloud compute instances list --format=json | jq '.[] | {name, zone: (.zone | split("/")[-1]), machine_type: (.machineType | split("/")[-1]), status, preemptible: (.scheduling.preemptible // false)}'
```

### Instance Health Check

```bash
# Check for instances without external IP
gcloud compute instances list --format=json | jq '.[] | select(.networkInterfaces[0].accessConfigs == null or (.networkInterfaces[0].accessConfigs | length) == 0) | .name'

# Find potentially problematic instances
gcloud compute instances list --format=json | jq '.[] | select(.status != "RUNNING" and .status != "TERMINATED") | {name, status, zone: (.zone | split("/")[-1])}'

# Check for old instances
gcloud compute instances list --format=json | jq '.[] | select((now - (.creationTimestamp | strptime("%Y-%m-%dT%H:%M:%S%Z") | mktime)) / 86400 > 90) | {name, age_days: ((now - (.creationTimestamp | strptime("%Y-%m-%dT%H:%M:%S%Z") | mktime)) / 86400 | floor), status}'
```

## Advanced jq Queries

### Complex Filtering

```bash
# Get instances matching multiple criteria
gcloud compute instances list --format=json | jq '.[] | select(.status == "RUNNING" and (.machineType | contains("e2-")) and (.networkInterfaces[0].accessConfigs[0].natIP != null))'

# Find instances with custom machine types
gcloud compute instances list --format=json | jq '.[] | select(.machineType | contains("custom")) | {name, custom_machine_type: (.machineType | split("/")[-1])}'

# Get instances by network
gcloud compute instances list --format=json | jq 'group_by(.networkInterfaces[0].network) | map({network: (.[0].networkInterfaces[0].network | split("/")[-1]), instances: [.[].name]})'
```

### Data Transformation

```bash
# Create instance inventory
gcloud compute instances list --format=json | jq 'map({
  name,
  zone: (.zone | split("/")[-1]),
  region: (.zone | split("/")[-1] | split("-")[0:2] | join("-")),
  status,
  machine_type: (.machineType | split("/")[-1]),
  internal_ip: .networkInterfaces[0].networkIP,
  external_ip: (.networkInterfaces[0].accessConfigs[0].natIP // "none"),
  created: .creationTimestamp,
  tags: (.tags.items // []),
  labels: (.labels // {}),
  preemptible: (.scheduling.preemptible // false)
})'

# Generate monitoring dashboard data
gcloud compute instances list --format=json | jq '{
  summary: {
    total: length,
    by_status: group_by(.status) | map({status: .[0].status, count: length}),
    by_zone: group_by(.zone) | map({zone: (.[0].zone | split("/")[-1]), count: length}),
    by_machine_type: group_by(.machineType) | map({type: (.[0].machineType | split("/")[-1]), count: length})
  },
  instances: map({
    name,
    zone: (.zone | split("/")[-1]),
    status,
    uptime_days: ((now - (.creationTimestamp | strptime("%Y-%m-%dT%H:%M:%S%Z") | mktime)) / 86400 | floor)
  })
}'
```

### Reporting Queries

```bash
# Security audit report
gcloud compute instances list --format=json | jq 'map({
  name,
  zone: (.zone | split("/")[-1]),
  has_external_ip: (.networkInterfaces[0].accessConfigs[0].natIP != null),
  service_account: (.serviceAccounts[0].email // "default"),
  tags: (.tags.items // []),
  preemptible: (.scheduling.preemptible // false),
  auto_restart: .scheduling.automaticRestart,
  on_host_maintenance: .scheduling.onHostMaintenance
})'

# Resource utilization report
gcloud compute instances list --format=json | jq '{
  resource_summary: {
    total_instances: length,
    total_vcpus: [.[] | if (.machineType | split("/")[-1] | contains("custom")) then (.machineType | split("/")[-1] | split("-")[1] | tonumber) else (.machineType | split("/")[-1] | split("-")[-1] | tonumber) end] | add,
    total_disk_gb: [.[].disks[] | .diskSizeGb | tonumber] | add
  },
  by_machine_family: group_by(.machineType | split("/")[-1] | split("-")[0]) | map({
    family: (.[0].machineType | split("/")[-1] | split("-")[0]),
    count: length,
    instances: [.[].name]
  })
}'
```

## Useful jq Patterns for GCP

### One-Liners for Common Tasks

```bash
# Quick instance status check
gcloud compute instances list --format=json | jq -r '.[] | "\(.name): \(.status) (\(.zone | split("/")[-1]))"' | sort

# Find all unique machine types in use
gcloud compute instances list --format=json | jq -r '.[].machineType | split("/")[-1]' | sort | uniq

# Get instances created today
gcloud compute instances list --format=json | jq --arg today "$(date +%Y-%m-%d)" '.[] | select(.creationTimestamp | startswith($today)) | .name'

# Count instances per project (if using multiple projects)
gcloud compute instances list --format=json | jq 'group_by(.selfLink | split("/")[6]) | map({project: (.[0].selfLink | split("/")[6]), count: length})'
```

## Tips and Best Practices

### Performance Tips

- Use `--filter` with gcloud commands when possible to reduce data transfer
- Pipe large outputs through `jq -c` for compact JSON when processing
- Use `gcloud compute instances list --zones=ZONE` to limit scope
- Cache frequently used instance data in local files

### Common Patterns

```bash
# Combine gcloud filter with jq for efficiency
gcloud compute instances list --filter="status:RUNNING" --format=json | jq '.[] | .name'

# Use jq's @base64 for encoding data
echo "script content" | jq -Rs '@base64'

# Pretty print with colors
gcloud compute instances list --format=json | jq -C '.'

# Stream processing for large datasets
gcloud compute instances list --format=json | jq -c '.[]' | while read instance; do
    echo "$instance" | jq '.name'
done
```
