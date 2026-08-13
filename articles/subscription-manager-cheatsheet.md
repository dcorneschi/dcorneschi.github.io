# subscription-manager Cheatsheet

`subscription-manager` is the tool for managing Red Hat subscriptions on RHEL systems. It registers hosts with Red Hat Customer Portal (or Satellite/Foreman), attaches subscriptions, enables repositories, and manages system entitlements.

## Register

```bash
# Register with username/password (interactive prompt for password)
sudo subscription-manager register --username=user@example.com

# Register with password inline
sudo subscription-manager register --username=user@example.com --password='MyPass123'

# Register with an activation key (preferred for automation — no password needed)
sudo subscription-manager register --activationkey=mykey --org=myorg

# Register with multiple activation keys
sudo subscription-manager register --activationkey=base-key,extras-key --org=myorg

# Register and auto-attach a subscription
sudo subscription-manager register --username=user@example.com --auto-attach

# Register to Satellite / Foreman (custom server)
sudo subscription-manager register --org=myorg --activationkey=mykey \
  --serverurl=https://satellite.example.com/rhsm \
  --baseurl=https://satellite.example.com/pulp/repos

# Register with a specific environment (Satellite)
sudo subscription-manager register --org=myorg --activationkey=mykey \
  --environment=Production/RHEL9

# Register and set system purpose
sudo subscription-manager register --username=user@example.com --auto-attach \
  --role="Red Hat Enterprise Linux Server" \
  --usage="Production" \
  --service-level="Premium"

# Force re-registration (re-use the same system UUID)
sudo subscription-manager register --force --username=user@example.com

# Register with a specific name (overrides hostname)
sudo subscription-manager register --username=user@example.com --name=myserver.example.com
```

## Unregister

```bash
# Unregister and remove all subscriptions
sudo subscription-manager unregister

# Remove system from RHSM but keep repos enabled (rare use case)
sudo subscription-manager remove --all
```

## Attach Subscriptions

```bash
# Auto-attach best matching subscription
sudo subscription-manager attach --auto

# Attach a specific subscription by pool ID
sudo subscription-manager attach --pool=8a85f98c60c2c2b40160c324e5c94e5a

# List available pools (subscriptions you can attach)
sudo subscription-manager list --available

# List all available pools (including already consumed)
sudo subscription-manager list --available --all

# List available pools with filters
sudo subscription-manager list --available --matches="Red Hat Enterprise Linux"
sudo subscription-manager list --available --all    # Include consumed

# List currently attached subscriptions
sudo subscription-manager list --consumed

# Remove a specific subscription
sudo subscription-manager remove --serial=1234567890

# Remove all attached subscriptions
sudo subscription-manager remove --all
```

## Repositories

```bash
# List all available repositories
sudo subscription-manager repos --list

# List only enabled repositories
sudo subscription-manager repos --list-enabled

# List only disabled repositories
sudo subscription-manager repos --list-disabled

# Enable a repository
sudo subscription-manager repos --enable=rhel-9-for-x86_64-baseos-rpms
sudo subscription-manager repos --enable=rhel-9-for-x86_64-appstream-rpms

# Enable multiple repositories
sudo subscription-manager repos \
  --enable=rhel-9-for-x86_64-baseos-rpms \
  --enable=rhel-9-for-x86_64-appstream-rpms \
  --enable=rhel-9-for-x86_64-supplementary-rpms

# Disable a repository
sudo subscription-manager repos --disable=rhel-9-for-x86_64-supplementary-rpms

# Disable all repos, then enable only what you need
sudo subscription-manager repos --disable="*"
sudo subscription-manager repos \
  --enable=rhel-9-for-x86_64-baseos-rpms \
  --enable=rhel-9-for-x86_64-appstream-rpms

# Search for repositories matching a pattern
sudo subscription-manager repos --list | grep -i satellite
sudo subscription-manager repos --list | grep -i codeready
```

### Common Repository Names

| Repository | RHEL Version | Content |
|-----------|-------------|---------|
| `rhel-9-for-x86_64-baseos-rpms` | 9 | Base OS packages |
| `rhel-9-for-x86_64-appstream-rpms` | 9 | Application Stream |
| `rhel-9-for-x86_64-supplementary-rpms` | 9 | Supplementary packages |
| `codeready-builder-for-rhel-9-x86_64-rpms` | 9 | Build dependencies (like EPEL needs) |
| `rhel-8-for-x86_64-baseos-rpms` | 8 | Base OS packages |
| `rhel-8-for-x86_64-appstream-rpms` | 8 | Application Stream |
| `codeready-builder-for-rhel-8-x86_64-rpms` | 8 | Build dependencies |
| `rhel-7-server-rpms` | 7 | Base server packages |
| `rhel-7-server-optional-rpms` | 7 | Optional packages |
| `rhel-7-server-extras-rpms` | 7 | Extras (Docker, etc.) |
| `rhel-7-server-supplementary-rpms` | 7 | Supplementary |

## System Identity & Status

```bash
# Show registration status
sudo subscription-manager status

# Show system identity (UUID, name, org)
sudo subscription-manager identity

# Show current subscription details
sudo subscription-manager list --installed

# Show all attached subscriptions with details
sudo subscription-manager list --consumed

# Show system facts
sudo subscription-manager facts

# Show specific fact
sudo subscription-manager facts --list | grep cpu
sudo subscription-manager facts --list | grep memory
sudo subscription-manager facts --list | grep virt

# Update facts (after hardware change)
sudo subscription-manager facts --update

# Show system purpose status
sudo subscription-manager syspurpose show
```

## System Purpose (RHEL 8+)

System purpose helps Red Hat match the best subscription to the system:

```bash
# Set role
sudo subscription-manager syspurpose set role "Red Hat Enterprise Linux Server"
sudo subscription-manager syspurpose set role "Red Hat Enterprise Linux Workstation"

# Set usage
sudo subscription-manager syspurpose set usage "Production"
sudo subscription-manager syspurpose set usage "Development/Test"
sudo subscription-manager syspurpose set usage "Disaster Recovery"

# Set service level
sudo subscription-manager syspurpose set service_level_agreement "Premium"
sudo subscription-manager syspurpose set service_level_agreement "Standard"
sudo subscription-manager syspurpose set service_level_agreement "Self-Support"

# Add-ons
sudo subscription-manager syspurpose add-ons --add "Extended Update Support"

# Show current purpose
sudo subscription-manager syspurpose show

# Clear purpose
sudo subscription-manager syspurpose unset role
sudo subscription-manager syspurpose unset usage
sudo subscription-manager syspurpose unset service_level_agreement
```

## Release Lock

Pin the system to a specific RHEL minor release (prevents upgrading beyond it):

```bash
# Show current release lock
sudo subscription-manager release --show

# List available releases
sudo subscription-manager release --list

# Lock to a specific release
sudo subscription-manager release --set=9.2
sudo subscription-manager release --set=8.8

# Remove release lock (get latest)
sudo subscription-manager release --unset
```

### Test a Release Without Locking

```bash
# Check what updates are available for a specific release (without setting it)
sudo yum --releasever=8.7 check-update
sudo dnf --releasever=9.2 check-update
```

> **Use case:** Production systems that must stay on a specific minor release for certification or support reasons. EUS (Extended Update Support) subscriptions provide patches for locked releases.

## Service Level (Legacy — RHEL 7)

On older systems without syspurpose, use the `service-level` subcommand:

```bash
# Show available service levels
sudo subscription-manager service-level --list

# Set service level
sudo subscription-manager service-level --set=Premium

# Show current service level
sudo subscription-manager service-level --show

# Unset
sudo subscription-manager service-level --unset
```

## Configuration

```bash
# Show all configuration
sudo subscription-manager config --list

# Show specific section
sudo subscription-manager config --list --section=server
sudo subscription-manager config --list --section=rhsm

# Set server hostname (for Satellite)
sudo subscription-manager config --server.hostname=satellite.example.com
sudo subscription-manager config --server.port=443
sudo subscription-manager config --server.prefix=/rhsm
sudo subscription-manager config --rhsm.baseurl=https://satellite.example.com/pulp/repos

# Set proxy
sudo subscription-manager config --server.proxy_hostname=proxy.example.com
sudo subscription-manager config --server.proxy_port=3128
sudo subscription-manager config --server.proxy_user=proxyuser
sudo subscription-manager config --server.proxy_password=proxypass

# Remove proxy
sudo subscription-manager config --remove=server.proxy_hostname
sudo subscription-manager config --remove=server.proxy_port
```

### Configuration File

```bash
# Main config
cat /etc/rhsm/rhsm.conf

# Key settings:
# [server]
# hostname = subscription.rhsm.redhat.com    (or satellite.example.com)
# port = 443
# prefix = /subscription                      (or /rhsm for Satellite)
#
# [rhsm]
# baseurl = https://cdn.redhat.com            (or satellite pulp URL)
# manage_repos = 1                            (let sub-manager manage .repo files)
# full_refresh_on_yum = 0

# Disable repository management entirely
# (sets manage_repos=0 and removes /etc/yum.repos.d/redhat.repo)
sudo subscription-manager config --rhsm.manage_repos=0

# Re-enable repository management
sudo subscription-manager config --rhsm.manage_repos=1
sudo subscription-manager refresh
```

### rhsm Service and rhsmcertd Daemon

```bash
# The rhsmcertd daemon runs in the background and:
# - Checks for subscription/entitlement updates every 4 hours (default)
# - Auto-refreshes /etc/yum.repos.d/redhat.repo
# - Triggers auto-attach if configured

# Check rhsmcertd status
sudo systemctl status rhsmcertd

# Start / enable rhsmcertd
sudo systemctl start rhsmcertd
sudo systemctl enable rhsmcertd

# Restart after config changes
sudo systemctl restart rhsmcertd

# Configuration: /etc/rhsm/rhsm.conf
# [rhsmcertd]
# certCheckInterval = 240    (minutes — default 4 hours)
# autoAttachInterval = 1440  (minutes — default 24 hours)

# Force immediate refresh instead of waiting for rhsmcertd
sudo subscription-manager refresh
```

## Refresh & Clean

```bash
# Refresh subscription data from the server
sudo subscription-manager refresh

# Redeem a subscription (hardware vendor pre-installed systems)
sudo subscription-manager redeem --email=admin@example.com --locale=en_US

# Clean all local subscription data (useful for re-registration)
sudo subscription-manager clean

# Remove all subscription data and certs
sudo subscription-manager remove --all
sudo subscription-manager unregister
sudo subscription-manager clean
```

## Satellite / Foreman Registration

### Using subscription-manager Directly

```bash
# Install Satellite CA certificate
sudo dnf install http://satellite.example.com/pub/katello-ca-consumer-latest.noarch.rpm

# Or manually:
sudo curl -o /etc/pki/rpm-gpg/katello-server-ca.pem \
  https://satellite.example.com/pub/katello-server-ca.pem

# Register with activation key
sudo subscription-manager register --org="MyOrg" --activationkey="RHEL9-Production"

# Verify
sudo subscription-manager identity
sudo subscription-manager repos --list-enabled
```

### Using rhc (Remote Host Configuration — RHEL 8.8+/9.2+)

```bash
# Connect to Red Hat Insights + register in one step
sudo rhc connect --activation-key=mykey --organization=myorg

# Disconnect
sudo rhc disconnect

# Status
sudo rhc status
```

### Using curl-based Registration (Global Registration Template)

```bash
# From Satellite's host registration page:
curl -sS --insecure 'https://satellite.example.com/register?activation_keys=mykey&organization_id=1' \
  -H 'Authorization: Bearer <token>' | bash
```

## Simple Content Access (SCA)

RHEL systems with Simple Content Access enabled (default since 2022) don't need to attach specific subscriptions. Registration alone provides access:

```bash
# Register — that's it. No attach needed with SCA.
sudo subscription-manager register --org=myorg --activationkey=mykey

# Verify SCA is active
sudo subscription-manager status
# Look for: "Content Access Mode is set to Simple Content Access"

# Repos are available immediately
sudo subscription-manager repos --list-enabled
```

## Auto-Attach vs Manual Attach

| Method | Command | Use Case |
|--------|---------|----------|
| Auto-attach | `subscription-manager attach --auto` | Let RHSM pick the best subscription |
| Pool ID | `subscription-manager attach --pool=<id>` | Specific subscription control |
| Activation key | `register --activationkey=key` | Automation, pre-defined subscriptions |
| SCA | Just register | Modern accounts (no attach needed) |

## One-Liners

```bash
# Full registration in one command (automation)
sudo subscription-manager register --org=myorg --activationkey=prod-rhel9 --force

# Show subscription expiration dates
sudo subscription-manager list --consumed | grep -E "Subscription Name|Ends"

# Show which repos are enabled
sudo subscription-manager repos --list-enabled | grep "Repo ID"

# Count available subscriptions
sudo subscription-manager list --available | grep -c "Pool ID"

# Find pools matching a product name
sudo subscription-manager list --available --matches="*Extended*Update*"

# Show system UUID
sudo subscription-manager identity | grep "system identity"

# Check if registered
sudo subscription-manager identity &>/dev/null && echo "Registered" || echo "NOT registered"

# List all facts as key=value
sudo subscription-manager facts --list

# Show virtual/physical status
sudo subscription-manager facts --list | grep virt.is_guest

# Quick status check
sudo subscription-manager status | head -5

# Enable CodeReady Builder (needed for EPEL and many dev packages)
sudo subscription-manager repos --enable codeready-builder-for-rhel-9-x86_64-rpms    # RHEL 9
sudo subscription-manager repos --enable codeready-builder-for-rhel-8-x86_64-rpms    # RHEL 8

# Export repo list for documentation
sudo subscription-manager repos --list-enabled --format="%{id}" 2>/dev/null | sort

# Force re-registration preserving the same UUID
UUID=$(sudo subscription-manager identity | grep "system identity" | awk '{print $NF}')
sudo subscription-manager register --force --consumerid=$UUID --org=myorg --activationkey=mykey
```

## Troubleshooting

### "Unable to verify server's identity"

```bash
# Missing CA certificate — install it
sudo dnf install http://satellite.example.com/pub/katello-ca-consumer-latest.noarch.rpm

# Or disable certificate verification (testing only!)
sudo subscription-manager config --server.insecure=1
```

### "This system is not registered"

```bash
# Check if registered
sudo subscription-manager identity

# If not registered, register
sudo subscription-manager register --username=user@example.com --auto-attach
```

### "Unable to find available subscriptions"

```bash
# Refresh data
sudo subscription-manager refresh

# Check available pools
sudo subscription-manager list --available --all

# Auto-attach
sudo subscription-manager attach --auto

# Check if SCA is enabled (no attach needed)
sudo subscription-manager status
```

### "Network error" or "Connection refused"

```bash
# Check connectivity to RHSM
curl -v https://subscription.rhsm.redhat.com/subscription/

# Check proxy settings
sudo subscription-manager config --list | grep proxy

# Test with proxy
curl -v --proxy http://proxy:3128 https://subscription.rhsm.redhat.com/subscription/

# Check DNS
nslookup subscription.rhsm.redhat.com
```

### "Insufficient entitlements"

```bash
# Show what's consumed vs available
sudo subscription-manager list --consumed
sudo subscription-manager list --available

# Try auto-attach
sudo subscription-manager attach --auto

# Or manually attach the right pool
sudo subscription-manager list --available --matches="*RHEL*"
sudo subscription-manager attach --pool=<pool-id>
```

### Repos Not Showing After Registration

```bash
# Ensure manage_repos is enabled
sudo subscription-manager config --rhsm.manage_repos=1

# Refresh
sudo subscription-manager refresh

# Check repo files are generated
ls /etc/yum.repos.d/redhat.repo

# If missing, try re-attaching
sudo subscription-manager attach --auto
```

### Clean Re-Registration

```bash
# Nuclear option — full cleanup and fresh start
sudo subscription-manager remove --all
sudo subscription-manager unregister
sudo subscription-manager clean
sudo dnf clean all
sudo rm -f /etc/pki/consumer/*
sudo rm -f /etc/pki/entitlement/*

# Re-register
sudo subscription-manager register --org=myorg --activationkey=mykey
```

## Environment Management (Satellite/Katello)

```bash
# List available environments
sudo subscription-manager environments --list --org=myorg

# Register to a specific environment
sudo subscription-manager register --org=myorg --activationkey=mykey \
  --environment=Production/RHEL9
```

## Content Overrides (repo-override)

Override repository properties without editing repo files directly. Useful for enabling/disabling GPG checks or changing baseurls on specific repos:

```bash
# List all content overrides
sudo subscription-manager repo-override --list

# Add an override (e.g., disable GPG check for a repo)
sudo subscription-manager repo-override --repo=rhel-9-for-x86_64-baseos-rpms \
  --add=gpgcheck:0

# Add multiple overrides
sudo subscription-manager repo-override --repo=myrepo \
  --add=enabled:1 --add=gpgcheck:0

# Remove a specific override
sudo subscription-manager repo-override --repo=rhel-9-for-x86_64-baseos-rpms \
  --remove=gpgcheck

# Remove all overrides for a repo
sudo subscription-manager repo-override --repo=rhel-9-for-x86_64-baseos-rpms \
  --remove-all
```

## Plugin Management

```bash
# List installed subscription-manager plugins
sudo subscription-manager plugins --list

# List plugin hooks (shows which events each plugin hooks into)
sudo subscription-manager plugins --listhooks
```

## Import Certificates

Manually import subscription certificates (offline/disconnected systems):

```bash
# Import an entitlement certificate (system must NOT be registered)
sudo subscription-manager import --certificate=/path/to/cert.pem

# Import multiple certificates
sudo subscription-manager import \
  --certificate=/path/to/cert1.pem \
  --certificate=/path/to/cert2.pem

# Verify import
sudo subscription-manager list --consumed
```

> **Use case:** Disconnected systems that can't reach the RHSM server. Export certificates from the Customer Portal and import them manually.

> **Note:** You cannot import certificates into a system that is already registered. Unregister first with `subscription-manager unregister` if needed. With SCA-enabled accounts, this workflow is deprecated — use offline registration instead.

## Per-Command Options

These options work with most `subscription-manager` subcommands:

| Option | Description |
|--------|-------------|
| `--help` | Show help for any command |
| `--proxy=<url>` | Use specific proxy for this command |
| `--proxyuser=<user>` | Proxy username |
| `--proxypassword=<pass>` | Proxy password |
| `--no-proxy` | Don't use proxy for this command |
| `--force` | Force the operation (re-register, overwrite) |
| `--org=<org_id>` | Specify organization ID |
| `--serverurl=<url>` | Override server URL for this command |

## Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | General error |
| `64` | Command line usage error |
| `69` | Service unavailable (network error, server unreachable) |
| `70` | Registration required (system not registered) |

```bash
# Use in scripts to check registration status
subscription-manager identity &>/dev/null
case $? in
  0) echo "Registered" ;;
  70) echo "Not registered" ;;
  69) echo "Cannot reach server" ;;
  *) echo "Error: $?" ;;
esac
```

## Important Files

| File | Purpose |
|------|---------|
| `/etc/rhsm/rhsm.conf` | Main configuration (server URL, proxy, repo management) |
| `/etc/pki/consumer/cert.pem` | Consumer identity certificate |
| `/etc/pki/consumer/key.pem` | Consumer identity key |
| `/etc/pki/entitlement/*.pem` | Subscription entitlement certificates |
| `/etc/pki/product/*.pem` | Installed product certificates |
| `/etc/yum.repos.d/redhat.repo` | Auto-generated repo file from subscriptions |
| `/var/lib/rhsm/facts/` | System facts (custom facts go here) |
| `/var/lib/rhsm/cache/` | Cached subscription data |
| `/var/log/rhsm/rhsm.log` | subscription-manager log file |
| `/etc/rhsm/ca/` | CA certificates for RHSM/Satellite |

## Quick Reference

```bash
# Registration
subscription-manager register --username=USER --auto-attach
subscription-manager register --org=ORG --activationkey=KEY
subscription-manager unregister

# Status
subscription-manager status
subscription-manager identity
subscription-manager list --installed
subscription-manager list --consumed

# Subscriptions
subscription-manager attach --auto
subscription-manager attach --pool=POOLID
subscription-manager list --available
subscription-manager remove --all

# Repositories
subscription-manager repos --list-enabled
subscription-manager repos --enable=REPO
subscription-manager repos --disable=REPO

# Release
subscription-manager release --set=9.2
subscription-manager release --unset

# Maintenance
subscription-manager refresh
subscription-manager clean
subscription-manager facts --update
```
