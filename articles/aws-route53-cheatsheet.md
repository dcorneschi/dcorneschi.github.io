# AWS Route 53 Cheatsheet

Amazon Route 53 — hosted zones, record types, routing policies, health checks, failover, and domain registration.

## Hosted Zones

```bash
# Create public hosted zone
aws route53 create-hosted-zone --name example.com --caller-reference "$(date +%s)"

# Create private hosted zone (VPC-only)
aws route53 create-hosted-zone \
  --name internal.example.com \
  --caller-reference "$(date +%s)" \
  --vpc VPCRegion=eu-west-1,VPCId=vpc-123

# List hosted zones
aws route53 list-hosted-zones
aws route53 list-hosted-zones --query 'HostedZones[].{Name:Name,ID:Id,Records:ResourceRecordSetCount}' --output table

# Get hosted zone details
aws route53 get-hosted-zone --id Z1234567890ABC

# Delete hosted zone (must delete all non-default records first)
aws route53 delete-hosted-zone --id Z1234567890ABC
```

## DNS Records

### Create/Update Records

```bash
# Create A record
aws route53 change-resource-record-sets --hosted-zone-id Z1234567890ABC --change-batch '{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "www.example.com",
      "Type": "A",
      "TTL": 300,
      "ResourceRecords": [{"Value": "1.2.3.4"}]
    }
  }]
}'

# Create CNAME record
aws route53 change-resource-record-sets --hosted-zone-id Z1234567890ABC --change-batch '{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "app.example.com",
      "Type": "CNAME",
      "TTL": 300,
      "ResourceRecords": [{"Value": "my-alb-123.eu-west-1.elb.amazonaws.com"}]
    }
  }]
}'

# Create Alias record (ALB, CloudFront, S3 — no TTL, no charge for queries)
aws route53 change-resource-record-sets --hosted-zone-id Z1234567890ABC --change-batch '{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "example.com",
      "Type": "A",
      "AliasTarget": {
        "HostedZoneId": "Z32O12XQLNTSW2",
        "DNSName": "my-alb-123.eu-west-1.elb.amazonaws.com",
        "EvaluateTargetHealth": true
      }
    }
  }]
}'

# Create MX record
aws route53 change-resource-record-sets --hosted-zone-id Z1234567890ABC --change-batch '{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "example.com",
      "Type": "MX",
      "TTL": 3600,
      "ResourceRecords": [
        {"Value": "10 mail1.example.com"},
        {"Value": "20 mail2.example.com"}
      ]
    }
  }]
}'

# Create TXT record
aws route53 change-resource-record-sets --hosted-zone-id Z1234567890ABC --change-batch '{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "example.com",
      "Type": "TXT",
      "TTL": 300,
      "ResourceRecords": [{"Value": "\"v=spf1 include:_spf.google.com ~all\""}]
    }
  }]
}'
```

### List and Delete Records

```bash
# List all records in a zone
aws route53 list-resource-record-sets --hosted-zone-id Z1234567890ABC

# List with filter
aws route53 list-resource-record-sets --hosted-zone-id Z1234567890ABC \
  --query "ResourceRecordSets[?Type=='A'].[Name,Type,ResourceRecords[0].Value]" --output table

# Delete record
aws route53 change-resource-record-sets --hosted-zone-id Z1234567890ABC --change-batch '{
  "Changes": [{
    "Action": "DELETE",
    "ResourceRecordSet": {
      "Name": "old.example.com",
      "Type": "A",
      "TTL": 300,
      "ResourceRecords": [{"Value": "1.2.3.4"}]
    }
  }]
}'
```

## Record Types

| Type | Use |
|------|-----|
| A | IPv4 address |
| AAAA | IPv6 address |
| CNAME | Alias to another domain (cannot be used at zone apex) |
| Alias | AWS-native alias to ALB/CloudFront/S3 (free, works at apex) |
| MX | Mail server |
| TXT | Text (SPF, DKIM, domain verification) |
| NS | Name servers |
| SOA | Start of authority |
| SRV | Service locator |
| CAA | Certificate authority authorization |

## Routing Policies

| Policy | Use Case |
|--------|----------|
| Simple | Single resource, no health check logic |
| Weighted | Split traffic by percentage (canary, blue/green) |
| Latency | Route to lowest-latency region |
| Failover | Active/passive — switch to standby on failure |
| Geolocation | Route based on user's geographic location |
| Geoproximity | Route based on distance (with bias) |
| Multivalue Answer | Return multiple healthy IPs (simple load balancing) |

### Weighted Routing

```bash
# 80% to primary, 20% to canary
aws route53 change-resource-record-sets --hosted-zone-id Z123 --change-batch '{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "primary",
        "Weight": 80,
        "TTL": 60,
        "ResourceRecords": [{"Value": "1.1.1.1"}]
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "canary",
        "Weight": 20,
        "TTL": 60,
        "ResourceRecords": [{"Value": "2.2.2.2"}]
      }
    }
  ]
}'
```

### Failover Routing

```bash
aws route53 change-resource-record-sets --hosted-zone-id Z123 --change-batch '{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "primary",
        "Failover": "PRIMARY",
        "TTL": 60,
        "ResourceRecords": [{"Value": "1.1.1.1"}],
        "HealthCheckId": "health-check-id"
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "secondary",
        "Failover": "SECONDARY",
        "TTL": 60,
        "ResourceRecords": [{"Value": "2.2.2.2"}]
      }
    }
  ]
}'
```

## Health Checks

```bash
# Create HTTP health check
aws route53 create-health-check --caller-reference "$(date +%s)" --health-check-config '{
  "IPAddress": "1.2.3.4",
  "Port": 80,
  "Type": "HTTP",
  "ResourcePath": "/health",
  "RequestInterval": 30,
  "FailureThreshold": 3
}'

# Create HTTPS health check
aws route53 create-health-check --caller-reference "$(date +%s)" --health-check-config '{
  "FullyQualifiedDomainName": "app.example.com",
  "Port": 443,
  "Type": "HTTPS",
  "ResourcePath": "/health",
  "RequestInterval": 10,
  "FailureThreshold": 2
}'

# Create TCP health check
aws route53 create-health-check --caller-reference "$(date +%s)" --health-check-config '{
  "IPAddress": "1.2.3.4",
  "Port": 3306,
  "Type": "TCP",
  "RequestInterval": 30,
  "FailureThreshold": 3
}'

# List health checks
aws route53 list-health-checks

# Get health check status
aws route53 get-health-check-status --health-check-id hc-123

# Delete health check
aws route53 delete-health-check --health-check-id hc-123
```

## Domain Registration

```bash
# Check domain availability
aws route53domains check-domain-availability --domain-name example.com

# Register domain
aws route53domains register-domain \
  --domain-name example.com \
  --duration-in-years 1 \
  --admin-contact file://contact.json \
  --registrant-contact file://contact.json \
  --tech-contact file://contact.json

# List registered domains
aws route53domains list-domains

# Get domain details
aws route53domains get-domain-detail --domain-name example.com

# Enable auto-renew
aws route53domains enable-domain-auto-renew --domain-name example.com

# Transfer lock
aws route53domains enable-domain-transfer-lock --domain-name example.com
```

## DNSSEC

```bash
# Enable DNSSEC signing
aws route53 enable-hosted-zone-dnssec --hosted-zone-id Z123

# Get DNSSEC status
aws route53 get-dnssec --hosted-zone-id Z123

# Disable DNSSEC
aws route53 disable-hosted-zone-dnssec --hosted-zone-id Z123
```

## Query Logging

```bash
# Create query log config
aws route53 create-query-logging-config \
  --hosted-zone-id Z123 \
  --cloud-watch-logs-log-group-arn arn:aws:logs:us-east-1:123456789012:log-group:/aws/route53/example.com

# List query log configs
aws route53 list-query-logging-configs --hosted-zone-id Z123

# Delete query log config
aws route53 delete-query-logging-config --id query-log-id
```

## Resolver (Hybrid DNS)

```bash
# Create inbound endpoint (on-prem → AWS)
aws route53resolver create-resolver-endpoint \
  --creator-request-id "$(date +%s)" \
  --name "inbound" \
  --security-group-ids sg-123 \
  --direction INBOUND \
  --ip-addresses SubnetId=subnet-aaa SubnetId=subnet-bbb

# Create outbound endpoint (AWS → on-prem)
aws route53resolver create-resolver-endpoint \
  --creator-request-id "$(date +%s)" \
  --name "outbound" \
  --security-group-ids sg-123 \
  --direction OUTBOUND \
  --ip-addresses SubnetId=subnet-aaa SubnetId=subnet-bbb

# Create forwarding rule
aws route53resolver create-resolver-rule \
  --creator-request-id "$(date +%s)" \
  --name "forward-onprem" \
  --rule-type FORWARD \
  --domain-name "corp.internal" \
  --target-ips Ip=10.0.0.53,Port=53 Ip=10.0.1.53,Port=53 \
  --resolver-endpoint-id rslvr-out-123
```

## Troubleshooting

| Issue | Check |
|-------|-------|
| DNS not resolving | Verify NS records at registrar match Route 53 hosted zone NS |
| Alias record not working | Check target resource exists and hosted zone ID is correct |
| Health check failing | Verify security group/NACL allows Route 53 health checker IPs |
| Propagation delay | TTL governs cache; lower TTL before making changes |
| Private zone not resolving | VPC must be associated with the private hosted zone |

```bash
# Test DNS resolution
dig +short example.com
dig +short example.com @ns-123.awsdns-45.com
nslookup example.com

# Check Route 53 name servers
aws route53 get-hosted-zone --id Z123 --query 'DelegationSet.NameServers'
```

## Best Practices

- Use Alias records for AWS resources (free, supports zone apex)
- Lower TTL to 60s before making changes, raise after
- Always associate health checks with failover records
- Use private hosted zones for internal service discovery
- Enable query logging for security auditing
- Use DNSSEC for domain integrity
- Register domains in Route 53 for seamless NS management
