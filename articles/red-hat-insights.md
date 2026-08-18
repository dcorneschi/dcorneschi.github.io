# Red Hat Insights

Red Hat Insights is a hosted service included with every RHEL subscription that continuously analyzes registered systems for configuration risks, security vulnerabilities, compliance drift, and patch status. It provides proactive recommendations and Ansible-based remediation playbooks through the [console.redhat.com](https://console.redhat.com) web interface.

## Key Services

| Service | Purpose |
|---------|---------|
| Advisor | Detects configuration issues affecting availability, stability, performance, and security |
| Vulnerability | Assesses exposure to CVEs, prioritizes by severity and existing mitigations |
| Compliance | Evaluates systems against OpenSCAP policies (PCI-DSS, CIS, HIPAA, etc.) |
| Patch | Shows applicable advisories (RHSA, RHBA, RHEA) and missing updates per system |
| Drift | Compares system configurations against baselines or other systems |
| Inventory | Central registry of all registered systems with facts, tags, and groups |
| Remediations | Generates Ansible playbooks to fix issues found by advisor, vulnerability, compliance, and patch |
| Malware Detection | Scans systems for known malware signatures using YARA rules |
| Resource Optimization | Right-sizing recommendations for cloud instances (AWS, Azure, GCP) |
| Image Builder | Build custom RHEL images for cloud, virtualization, and bare-metal deployments |

## Install

```bash
# RHEL 8+ (pre-installed — just needs registration)
# The insights-client package is included by default (except minimal install)

# RHEL 7 (package loaded but not installed)
sudo yum install insights-client

# RHEL 6.10+ (manual install — legacy package name)
sudo yum install redhat-access-insights

# Verify installed version
insights-client --version
rpm -q insights-client
```

## Register

```bash
# Prerequisites: system must be registered with subscription-manager
sudo subscription-manager register --auto-attach

# Register with Red Hat Insights
sudo insights-client --register

# Register with a custom display name (shown in the Insights UI)
sudo insights-client --register --display-name="web-prod-01.example.com"

# Register and run first data collection in one step
sudo insights-client --register

# Change display name after registration
sudo insights-client --display-name="new-name.example.com"

# Check registration status
sudo insights-client --status

# Unregister from Insights
sudo insights-client --unregister
```

## Data Collection

```bash
# Run a manual data collection and upload
sudo insights-client

# Collect data but do NOT upload (review what gets sent)
sudo insights-client --no-upload

# Check what data is collected (stored temporarily)
ls /var/tmp/insights-client/

# Force a check-in without full collection
sudo insights-client --checkin

# View the archive that would be uploaded
sudo insights-client --no-upload --output-file=/tmp/insights-archive.tar.gz
tar -tzf /tmp/insights-archive.tar.gz

# Run with verbose output for troubleshooting
sudo insights-client --verbose

# Run compliance scan (requires SCAP policy assigned in console)
sudo insights-client --compliance

# Run malware detection scan
sudo insights-client --collector malware-detection
```

## Important File Paths

| Path | Purpose |
|------|---------|
| `/etc/insights-client/insights-client.conf` | Current configuration file |
| `/etc/redhat-access-insights/redhat-access-insights.conf` | Legacy configuration file (RHEL 6/7 with old package) |
| `/var/log/insights-client/insights-client.log` | Client log file |
| `/var/tmp/insights-client/` | Temporary data collection directory |
| `/etc/insights-client/tags.yaml` | System tags for inventory grouping |
| `/etc/insights-client/file-redaction.yaml` | File/command exclusion rules |
| `/etc/insights-client/file-content-redaction.yaml` | Content pattern redaction rules |
| `/etc/insights-client/.registered` | Registration marker file |
| `/etc/insights-client/.lastupload` | Last upload timestamp |
| `/usr/share/xml/scap/ssg/content/` | OpenSCAP SCAP Security Guide profiles |

## Configuration File

The main configuration file is `/etc/insights-client/insights-client.conf`.

```ini
# /etc/insights-client/insights-client.conf

[insights-client]
# Enable or disable automatic scheduling (default: True)
auto_config=True

# Upload URL (default — rarely needs changing)
# base_url=cert-api.access.redhat.com/r/insights

# Authentication method (cert or basic)
authmethod=CERT

# Custom display name for the system
# display_name=myserver.example.com

# Proxy configuration
# proxy=http://proxy.example.com:8080

# Proxy authentication
# proxy_user=proxyuser
# proxy_pass=proxypassword

# Disable automatic updates of the insights-client egg
# auto_update=True

# Enable/disable data obfuscation for hostnames
# obfuscate=False

# Enable/disable data obfuscation for IP addresses
# obfuscate_hostname=False

# Connection timeout in seconds
# cmd_timeout=120

# Log level
# loglevel=DEBUG

# Redact specific patterns from collected data
# redaction_file=/etc/insights-client/file-redaction.yaml

# Content redaction for IP addresses
# file-content-redaction.yaml can strip IPs and hostnames from payloads
```

## Proxy Configuration

```bash
# Option 1: Set proxy in insights-client.conf
sudo sed -i 's/^#proxy=.*/proxy=http:\/\/proxy.example.com:8080/' \
  /etc/insights-client/insights-client.conf

# Option 2: Use environment variable
export HTTPS_PROXY=http://proxy.example.com:8080
sudo --preserve-env=HTTPS_PROXY insights-client --register

# Option 3: Use subscription-manager proxy (auto_config=True picks it up)
sudo subscription-manager config --server.proxy_hostname=proxy.example.com \
  --server.proxy_port=8080

# Test connectivity through the proxy
sudo insights-client --test-connection
```

## Scheduling

```bash
# The client runs automatically every 24 hours via systemd timer
systemctl status insights-client.timer
systemctl list-timers insights-client.timer

# View the timer schedule
systemctl cat insights-client.timer

# Disable automatic collection
sudo systemctl stop insights-client.timer
sudo systemctl disable insights-client.timer

# Re-enable automatic collection
sudo systemctl enable insights-client.timer
sudo systemctl start insights-client.timer

# On RHEL 7 (uses cron instead of systemd timer)
cat /etc/cron.daily/insights-client

# Override the schedule interval (edit the timer)
sudo systemctl edit insights-client.timer
# [Timer]
# OnCalendar=
# OnCalendar=*-*-* 02:00:00
```

## Data Redaction

```bash
# Create file-redaction.yaml to exclude files from collection
cat /etc/insights-client/file-redaction.yaml
```

```yaml
# /etc/insights-client/file-redaction.yaml
# Exclude specific files or commands from data collection
---
commands:
  - /bin/rpm -qa
  - /usr/sbin/dmidecode
files:
  - /etc/samba/smb.conf
patterns:
  - "password"
  - "secret_key"
keywords:
  - "SSN"
  - "credit_card"
```

```bash
# Create file-content-redaction.yaml to mask content patterns
cat /etc/insights-client/file-content-redaction.yaml
```

```yaml
# /etc/insights-client/file-content-redaction.yaml
# Redact patterns from all collected data
---
patterns:
  regex:
    - "\\d{3}-\\d{2}-\\d{4}"        # SSN pattern
    - "password\\s*=\\s*\\S+"        # password assignments
keywords:
  - "my_secret_value"
  - "internal_hostname"
```

## Tags and Groups

```bash
# Create tags file for the system (used for filtering in the UI)
cat /etc/insights-client/tags.yaml
```

```yaml
# /etc/insights-client/tags.yaml
# Tags are key/value pairs for organizing systems in Insights
---
group: web-servers
environment: production
location: datacenter-1
team: platform
application: ecommerce
patch_window: sunday-02:00
```

```bash
# Tags are uploaded on next insights-client run
sudo insights-client

# Verify tags are attached (check in console.redhat.com > Inventory)
```

## Troubleshooting

```bash
# Test connectivity to Red Hat Insights
sudo insights-client --test-connection

# Run with full debug output
sudo insights-client --verbose

# Check registration status
sudo insights-client --status

# View client logs
sudo cat /var/log/insights-client/insights-client.log
sudo journalctl -u insights-client

# Check if subscription-manager identity is valid
sudo subscription-manager identity

# Re-register if identity is broken
sudo subscription-manager clean
sudo subscription-manager register --auto-attach
sudo insights-client --register

# Verify certificate files exist
ls -la /etc/pki/consumer/
# Should contain cert.pem and key.pem

# Check insights-client package version
rpm -q insights-client

# Update insights-client to latest
sudo dnf update insights-client      # RHEL 8+
sudo yum update insights-client      # RHEL 7

# Network requirements (ports and URLs)
# HTTPS (443) to:
#   cert-api.access.redhat.com
#   api.access.redhat.com
#   cert.console.redhat.com
#   console.redhat.com

# Force re-registration
sudo insights-client --unregister
sudo insights-client --register

# Clear local cache
sudo rm -rf /var/tmp/insights-client/
sudo rm -rf /etc/insights-client/.registered
```

## Satellite Integration

```bash
# Systems registered via Satellite automatically report to Insights
# (if the Insights plugin is enabled on Satellite)

# Register to Satellite with Insights enabled
sudo subscription-manager register --org=myorg --activationkey=mykey \
  --serverurl=https://satellite.example.com/rhsm \
  --baseurl=https://satellite.example.com/pulp/repos

# Insights client auto-detects Satellite and routes through it
# The auto_config=True setting picks up Satellite connection info

# Verify connection routes through Satellite
sudo insights-client --test-connection --verbose

# On Satellite: enable Insights plugin
# Administer > Settings > RH Cloud
# Set "Synchronize Inventory" to Yes

# Satellite acts as a proxy — systems don't need direct internet access
# Insights data flows: Client -> Satellite/Capsule -> Red Hat Cloud
```

## OpenSCAP Compliance Scanning

```bash
# Install OpenSCAP scanner and SCAP Security Guide
sudo yum install openscap-scanner scap-security-guide    # RHEL 7
sudo dnf install openscap-scanner scap-security-guide    # RHEL 8+

# Verify oscap version
oscap -V

# List available profiles for RHEL 7
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel7-xccdf.xml

# List available profiles for RHEL 8
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel8-xccdf.xml

# List available profiles for RHEL 9
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-xccdf.xml

# Run a compliance scan and generate results for Insights
TZ=UTC oscap xccdf eval \
  --profile standard \
  --results /var/lib/insights/latest-compliance-report.xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel7-xccdf.xml

# Run scan with HTML report output
TZ=UTC oscap xccdf eval \
  --profile standard \
  --results /var/lib/insights/latest-compliance-report.xml \
  --report /var/www/html/compliance-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel8-xccdf.xml

# Upload compliance results to Insights
sudo insights-client --compliance
```

## CLI Reference

| Command | Description |
|---------|-------------|
| `insights-client --register` | Register system with Insights |
| `insights-client --unregister` | Remove system from Insights |
| `insights-client --status` | Show registration status |
| `insights-client` | Run data collection and upload |
| `insights-client --no-upload` | Collect data without uploading |
| `insights-client --checkin` | Lightweight check-in without full collection |
| `insights-client --display-name=NAME` | Set or change display name |
| `insights-client --group=GROUP` | Set the system group |
| `insights-client --test-connection` | Test connectivity to Insights |
| `insights-client --verbose` | Run with debug output |
| `insights-client --compliance` | Run OpenSCAP compliance scan |
| `insights-client --collector malware-detection` | Run malware detection |
| `insights-client --output-file=PATH` | Save archive to file |
| `insights-client --offline` | Collect for offline analysis |
| `insights-client --version` | Show client version |
| `insights-client --support` | Generate support bundle for troubleshooting |

## Useful One-Liners

```bash
# Register all systems in an inventory file using Ansible
ansible all -m shell -a "insights-client --register" -b

# Check which systems are NOT registered
ansible all -m shell -a "insights-client --status" -b | grep -B1 "NOT"

# Bulk update display names from hostname
ansible all -m shell -a 'insights-client --display-name="$(hostname -f)"' -b

# List all insights-related packages
rpm -qa | grep -i insights

# Check when the last upload happened
stat /etc/insights-client/.lastupload 2>/dev/null || echo "No upload recorded"

# Monitor insights-client timer
watch systemctl list-timers insights-client.timer

# Extract and inspect the collected payload
sudo insights-client --no-upload --output-file=/tmp/payload.tar.gz
mkdir /tmp/payload && tar -xzf /tmp/payload.tar.gz -C /tmp/payload
find /tmp/payload -type f | head -30

# Quick health check across fleet (Ansible)
ansible all -m shell -a "insights-client --test-connection 2>&1 | tail -1" -b
```
