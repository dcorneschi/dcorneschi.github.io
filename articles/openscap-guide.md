# OpenSCAP Security Compliance Guide

OpenSCAP is the open-source implementation of SCAP (Security Content Automation Protocol) on Red Hat Enterprise Linux. It provides automated security compliance scanning, vulnerability assessment, and remediation using standardized NIST-certified security policies. This guide covers installation, scanning, remediation, profile selection, and integration with Satellite, Ansible, and Insights across RHEL 7–10.

---

## Concepts

| Term | Meaning |
|------|---------|
| **SCAP** | Security Content Automation Protocol — NIST standard for automated compliance |
| **XCCDF** | eXtensible Configuration Checklist Description Format — defines security benchmarks |
| **OVAL** | Open Vulnerability and Assessment Language — defines system checks |
| **SSG** | SCAP Security Guide — Red Hat's collection of SCAP profiles and rules |
| **Profile** | A named set of rules targeting a specific compliance standard (CIS, STIG, PCI-DSS, etc.) |
| **Data stream** | A single XML file containing all definitions, benchmarks, profiles, and rules |
| **Remediation** | Automatically fixing non-compliant configuration items |
| **Tailoring** | Customizing a profile by enabling/disabling specific rules |

---

## Installation

```bash
# RHEL 7
yum install openscap-scanner scap-security-guide

# RHEL 8/9/10
dnf install openscap-scanner scap-security-guide

# Optional: GUI tool
dnf install scap-workbench

# Optional: utilities for content development
dnf install openscap-utils

# Verify installation
oscap --version
rpm -q scap-security-guide
```

### Content Location

The SCAP Security Guide data stream files are installed at:

```bash
# RHEL 7
/usr/share/xml/scap/ssg/content/ssg-rhel7-ds.xml

# RHEL 8
/usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml

# RHEL 9
/usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# RHEL 10
/usr/share/xml/scap/ssg/content/ssg-rhel10-ds.xml

# List all available content
ls /usr/share/xml/scap/ssg/content/
```

---

## Profiles

### List Available Profiles

```bash
# List all profiles in the data stream
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Show only profile IDs and titles
oscap info --profiles /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

### Common Profiles (RHEL 8/9/10)

| Profile ID | Standard | Description |
|-----------|----------|-------------|
| `xccdf_org.ssgproject.content_profile_cis` | CIS Benchmark Level 2 | Center for Internet Security — full hardening |
| `xccdf_org.ssgproject.content_profile_cis_server_l1` | CIS Level 1 Server | CIS baseline for servers (less restrictive) |
| `xccdf_org.ssgproject.content_profile_cis_workstation_l1` | CIS Level 1 Workstation | CIS baseline for workstations |
| `xccdf_org.ssgproject.content_profile_stig` | DISA STIG | Defense Information Systems Agency hardening |
| `xccdf_org.ssgproject.content_profile_ospp` | OSPP | Protection Profile for General Purpose OS |
| `xccdf_org.ssgproject.content_profile_pci-dss` | PCI-DSS | Payment Card Industry Data Security Standard |
| `xccdf_org.ssgproject.content_profile_hipaa` | HIPAA | Health Insurance Portability and Accountability Act |
| `xccdf_org.ssgproject.content_profile_e8` | Australian Essential Eight | Australian Signals Directorate Essential Eight |
| `xccdf_org.ssgproject.content_profile_anssi_bp28_high` | ANSSI High | French National Agency for Security (high level) |

### Profile Details

```bash
# Show detailed info for a specific profile
oscap info --profile xccdf_org.ssgproject.content_profile_cis \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

---

## Scanning

### Verify Installation

```bash
# Check OpenSCAP version and supported specifications
oscap -V

# Find installed SCAP content files
rpm -ql scap-security-guide | grep content
```

### Basic Compliance Scan

```bash
# Scan against CIS Level 1 Server profile (using --results-arf for data streams)
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --results-arf /tmp/arf.xml \
  --report /tmp/scan-report.html \
  --oval-results \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

> **Tip:** Use `--results-arf` instead of `--results` when scanning with data stream files. The ARF (Asset Reporting Format) file is more complete than plain XCCDF results — it bundles the benchmark, system info, and check results into one file.

### Scan Options

```bash
# Full scan with all output formats
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --results /tmp/results-stig.xml \
  --report /tmp/report-stig.html \
  --oval-results \
  --fetch-remote-resources \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Scan and return exit code (useful for CI/CD)
# Exit 0 = all pass, Exit 2 = some fail
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_pci-dss \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
echo $?
```

### Scan Remote System via SSH

```bash
# Scan a remote host
oscap-ssh user@remote-host 22 xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --results /tmp/remote-results.xml \
  --report /tmp/remote-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

### Scan a Container Image

```bash
# Scan a container image (RHEL 8+)
oscap-podman <image-id> xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --report /tmp/container-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Scan a Docker container (older RHEL 7)
oscap-docker <container-id> xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  /usr/share/xml/scap/ssg/content/ssg-rhel7-ds.xml
```

### Vulnerability Scan (OVAL)

```bash
# Download latest RHEL OVAL definitions
wget https://access.redhat.com/security/data/oval/v2/RHEL9/rhel-9.oval.xml.bz2
bunzip2 rhel-9.oval.xml.bz2

# Scan for known vulnerabilities
oscap oval eval \
  --results /tmp/oval-results.xml \
  --report /tmp/oval-report.html \
  rhel-9.oval.xml
```

---

## Remediation

### Online Remediation (During Scan)

```bash
# Scan AND fix non-compliant rules in one pass
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --remediate \
  --results /tmp/remediation-results.xml \
  --report /tmp/remediation-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

> **Warning:** `--remediate` modifies your system. Always test in a non-production environment first and review the profile rules before applying.

### Generate Bash Remediation Script

```bash
# Generate a bash script with all fixes for a profile
oscap xccdf generate fix \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --fix-type bash \
  --output /tmp/remediate-cis.sh \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Review before running
less /tmp/remediate-cis.sh

# Apply
chmod +x /tmp/remediate-cis.sh
/tmp/remediate-cis.sh
```

### Generate Targeted Fix (Only Failed Rules)

```bash
# First, scan and save results
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --results /tmp/stig-results.xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Generate fix only for rules that failed
oscap xccdf generate fix \
  --fix-type bash \
  --result-id "" \
  --output /tmp/fix-failed-only.sh \
  /tmp/stig-results.xml
```

### Generate Ansible Remediation Playbook

```bash
# Generate full Ansible playbook for a profile
oscap xccdf generate fix \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --fix-type ansible \
  --output /tmp/remediate-cis.yml \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Generate playbook for only failed rules
oscap xccdf generate fix \
  --fix-type ansible \
  --result-id "" \
  --output /tmp/fix-failed.yml \
  /tmp/scan-results.xml

# Dry-run first: preview changes without applying (recommended)
ansible-playbook -i localhost, -c local --check --diff /tmp/remediate-cis.yml

# Once reviewed and satisfied, apply for real
ansible-playbook -i localhost, -c local /tmp/remediate-cis.yml
```

### Pre-Built Ansible Remediation Roles

The `scap-security-guide` package includes ready-made Ansible roles:

```bash
# Location of Ansible content
ls /usr/share/scap-security-guide/ansible/

# Example: apply CIS profile
ansible-playbook -i localhost, -c local \
  /usr/share/scap-security-guide/ansible/rhel9-playbook-cis_server_l1.yml
```

---

## Tailoring (Customizing Profiles)

### Create a Tailoring File

Tailoring allows you to customize an existing profile without modifying the original content:

```bash
# Using scap-workbench (GUI)
scap-workbench

# Using autotailor (CLI, RHEL 8+)
# Disable a specific rule from a profile
autotailor \
  --new-profile-id my_custom_cis \
  --unselect xccdf_org.ssgproject.content_rule_disable_host_auth \
  --unselect xccdf_org.ssgproject.content_rule_require_emergency_target_auth \
  --output /tmp/my-tailoring.xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml \
  xccdf_org.ssgproject.content_profile_cis_server_l1
```

### Scan with Tailoring

```bash
# Use a tailoring file during scan
oscap xccdf eval \
  --profile my_custom_cis \
  --tailoring-file /tmp/my-tailoring.xml \
  --results /tmp/tailored-results.xml \
  --report /tmp/tailored-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

---

## Reports and Results

### Generate HTML Report from Results

```bash
# If you already have results XML, generate a new report
oscap xccdf generate report /tmp/scan-results.xml > /tmp/report.html
```

### DISA STIG Viewer Output

For DoD environments that use the DISA STIG Viewer tool, generate results with aligned rule IDs:

```bash
# Generate results compatible with STIG Viewer
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --stig-viewer /tmp/stig-viewer-results.xml \
  --results-arf /tmp/arf.xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Import stig-viewer-results.xml into DISA STIG Viewer:
# File → Import → XCCDF Results File → select stig-viewer-results.xml
```

> **Note:** `--stig-viewer` can be used together with `--results-arf` and `--results` to produce both standard and STIG Viewer results in a single scan. The STIG Viewer maps results as: Open = fail, Not a Finding = pass, Not Reviewed = notchecked, Not Applicable = notapplicable.

### Interpret Results

| Result | Meaning |
|--------|---------|
| `pass` | Rule is satisfied |
| `fail` | Rule is not satisfied — system is non-compliant |
| `notapplicable` | Rule does not apply to this system |
| `notchecked` | Rule could not be evaluated (missing data or dependencies) |
| `error` | An error occurred during evaluation |
| `fixed` | Rule was non-compliant but remediation was applied successfully |

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | All rules passed (or not applicable) |
| 1 | Error during scanning |
| 2 | At least one rule failed |

---

## Installation-Time Hardening

### Kickstart Integration

Apply a security profile during RHEL installation:

```bash
# In kickstart file (%addon section)
%addon org_fedora_oscap
  content-type = scap-security-guide
  profile = xccdf_org.ssgproject.content_profile_cis_server_l1
%end
```

### Image Builder Integration (RHEL 8+)

Build pre-hardened images with compliance baked in:

```bash
# Create a blueprint with OpenSCAP profile
cat > hardened-blueprint.toml << 'EOF'
name = "hardened-rhel9"
description = "CIS Level 1 hardened RHEL 9"
version = "1.0.0"

[customizations]
[customizations.openscap]
profile_id = "xccdf_org.ssgproject.content_profile_cis_server_l1"
datastream = "/usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml"
EOF

# Push and build
composer-cli blueprints push hardened-blueprint.toml
composer-cli compose start hardened-rhel9 qcow2
```

---

## Integration with Satellite and Insights

### Red Hat Satellite

```bash
# On Satellite, import SCAP content
foreman-rake foreman_openscap:bulk_upload:default

# Or upload custom content
hammer scap-content create \
  --title "RHEL9 SSG" \
  --scap-file /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Create a compliance policy
hammer policy create \
  --name "CIS Level 1" \
  --scap-content-id 1 \
  --scap-content-profile-id 2 \
  --period weekly \
  --weekday monday

# Assign policy to host group
hammer policy update --id 1 --hostgroup-ids 5,6

# View compliance reports
hammer policy compliance-report --id 1
```

### Red Hat Insights

```bash
# Register system with Insights
insights-client --register

# Insights compliance service uses the same SSG profiles
# Configure compliance policies in the Insights web UI:
# https://console.redhat.com/insights/compliance

# Manual scan upload to Insights
insights-client --compliance
```

---

## Scheduling Scans

### systemd Timer

```bash
# Create a service unit
cat > /etc/systemd/system/openscap-scan.service << 'EOF'
[Unit]
Description=OpenSCAP CIS Compliance Scan

[Service]
Type=oneshot
ExecStart=/usr/bin/oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --results /var/log/openscap/results-%i.xml \
  --report /var/log/openscap/report-%i.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
EOF

# Create a timer
cat > /etc/systemd/system/openscap-scan.timer << 'EOF'
[Unit]
Description=Weekly OpenSCAP Scan

[Timer]
OnCalendar=Sun 02:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

# Enable
mkdir -p /var/log/openscap
systemctl daemon-reload
systemctl enable --now openscap-scan.timer
```

### Cron Alternative

```bash
# /etc/cron.weekly/openscap-scan
#!/bin/bash
DATE=$(date +%Y%m%d)
RESULTS_DIR="/var/log/openscap"
mkdir -p "$RESULTS_DIR"

oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --results "$RESULTS_DIR/results-$DATE.xml" \
  --report "$RESULTS_DIR/report-$DATE.html" \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Optional: send email on failures
if [ $? -eq 2 ]; then
  mail -s "OpenSCAP: compliance failures on $(hostname)" admin@example.com \
    < "$RESULTS_DIR/report-$DATE.html"
fi
```

---

## Ansible Integration

### Using the compliance-as-code Ansible Roles

```bash
# Install the Ansible roles (from scap-security-guide)
dnf install scap-security-guide

# List available playbooks
ls /usr/share/scap-security-guide/ansible/

# Available per-profile playbooks:
# rhel9-playbook-cis_server_l1.yml
# rhel9-playbook-stig.yml
# rhel9-playbook-pci-dss.yml
# rhel9-playbook-ospp.yml
# ... etc
```

### Example: Apply CIS and Verify

```yaml
# apply-and-scan.yml
---
- name: Apply CIS Level 1 and verify compliance
  hosts: all
  become: true
  tasks:
    - name: Apply CIS remediation
      ansible.builtin.include_role:
        name: /usr/share/scap-security-guide/ansible/rhel9-role-cis_server_l1

    - name: Run compliance scan
      ansible.builtin.command:
        cmd: >
          oscap xccdf eval
          --profile xccdf_org.ssgproject.content_profile_cis_server_l1
          --results /tmp/post-remediation-results.xml
          --report /tmp/post-remediation-report.html
          /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
      register: scan_result
      failed_when: scan_result.rc == 1
      changed_when: false

    - name: Fetch report
      ansible.builtin.fetch:
        src: /tmp/post-remediation-report.html
        dest: ./reports/{{ inventory_hostname }}/
        flat: true
```

---

## Troubleshooting

### Common Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| `oscap: command not found` | Package not installed | `dnf install openscap-scanner` |
| No profiles listed | Missing SSG content | `dnf install scap-security-guide` |
| `File not found: ssg-rhel9-ds.xml` | Wrong RHEL version content | Check with `rpm -ql scap-security-guide` |
| Scan hangs on `--fetch-remote-resources` | Network issues | Remove flag or pre-download OVAL content |
| Rules showing `notchecked` | Missing dependencies | Install required packages (e.g., `audit`, `aide`) |
| Remediation broke services | Aggressive profile rules | Use tailoring to exclude problematic rules |

### Useful Debugging

```bash
# Validate content file
oscap ds sds-validate /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Check which rules are in a profile
oscap info --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Verbose scan output
oscap xccdf eval --verbose DEVEL \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml 2>/tmp/debug.log

# Check SSG package version
rpm -q scap-security-guide
```

---

## Quick Reference

```bash
# === Installation ===
dnf install openscap-scanner scap-security-guide

# === List profiles ===
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# === Scan ===
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --results /tmp/results.xml \
  --report /tmp/report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# === Scan + Remediate ===
oscap xccdf eval --remediate \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --results /tmp/results.xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# === Generate bash fix script ===
oscap xccdf generate fix \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --fix-type bash \
  --output /tmp/fix.sh \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# === Generate Ansible playbook ===
oscap xccdf generate fix \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --fix-type ansible \
  --output /tmp/fix.yml \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# === Generate fix for failed rules only ===
oscap xccdf generate fix \
  --fix-type bash \
  --result-id "" \
  --output /tmp/fix-failed.sh \
  /tmp/results.xml

# === Scan remote host ===
oscap-ssh user@host 22 xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --report /tmp/remote-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# === Scan container ===
oscap-podman <image-id> xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# === OVAL vulnerability scan ===
oscap oval eval --report /tmp/vuln.html rhel-9.oval.xml

# === Validate content ===
oscap ds sds-validate /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# === Report from existing results ===
oscap xccdf generate report /tmp/results.xml > /tmp/report.html
```
