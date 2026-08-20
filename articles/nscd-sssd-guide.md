# NSCD and SSSD Guide

NSCD (Name Service Cache Daemon) and SSSD (System Security Services Daemon) both cache name service lookups, but serve different purposes. This guide covers installation, configuration, and when to use each on RHEL and Ubuntu.

## Overview

| Feature | NSCD | SSSD |
|---------|------|------|
| Purpose | Simple caching for NSS lookups | Identity/auth provider with caching |
| Backends | Caches whatever NSS provides | LDAP, AD, Kerberos, IPA, local |
| Authentication | No (caching only) | Yes (PAM integration) |
| Offline mode | No | Yes (cached credentials) |
| Complexity | Simple | Complex but powerful |
| Use case | DNS/passwd caching on standalone hosts | Domain-joined systems, AD/LDAP integration |
| Conflict | Conflicts with SSSD | Replaces NSCD |

## NSCD (Name Service Cache Daemon)

NSCD caches results from `passwd`, `group`, `hosts`, `services`, and `netgroup` databases. It reduces load on NIS, LDAP, or DNS servers by serving repeated lookups from memory.

### Install on RHEL

```bash
sudo dnf install -y nscd
sudo systemctl enable --now nscd
```

### Install on Ubuntu

```bash
sudo apt install -y nscd
sudo systemctl enable --now nscd
```

### Configuration

```bash
sudo vi /etc/nscd.conf
```

Key configuration options:

```bash
# /etc/nscd.conf

# Enable/disable caching per database
enable-cache            passwd      yes
enable-cache            group       yes
enable-cache            hosts       yes
enable-cache            services    yes
enable-cache            netgroup    yes

# Positive cache TTL (seconds)
positive-time-to-live   passwd      600
positive-time-to-live   group       3600
positive-time-to-live   hosts       3600
positive-time-to-live   services    28800
positive-time-to-live   netgroup    28800

# Negative cache TTL (how long to cache "not found")
negative-time-to-live   passwd      20
negative-time-to-live   group       60
negative-time-to-live   hosts       20
negative-time-to-live   services    20
negative-time-to-live   netgroup    20

# Maximum entries in cache
suggested-size          passwd      211
suggested-size          group       211
suggested-size          hosts       211

# Paranoia mode — restart nscd periodically to avoid stale data
paranoia                yes
restart-interval        3600

# Check files for changes (e.g., /etc/passwd, /etc/hosts)
check-files             passwd      yes
check-files             group       yes
check-files             hosts       yes
```

### NSCD Management Commands

```bash
# Check status
sudo systemctl status nscd

# Restart
sudo systemctl restart nscd

# View statistics
sudo nscd -g

# Invalidate specific cache
sudo nscd -i passwd
sudo nscd -i group
sudo nscd -i hosts
sudo nscd -i services
sudo nscd -i netgroup

# Invalidate all caches (flush everything)
sudo nscd -i passwd && sudo nscd -i group && sudo nscd -i hosts

# Or restart the service
sudo systemctl restart nscd
```

### NSCD Troubleshooting

```bash
# Check if nscd is running
systemctl status nscd
ps aux | grep nscd

# View cache statistics
sudo nscd -g
# Look for: positive/negative hits, misses, cache size

# Debug mode (foreground with verbose output)
sudo nscd -d

# Check socket
ls -la /var/run/nscd/socket

# If nscd causes stale data
sudo nscd -i hosts
# Or disable hosts caching if using systemd-resolved:
# enable-cache hosts no
```

### When to Use NSCD

- Standalone servers without domain membership
- Reducing DNS lookup latency
- Caching NIS/NIS+ lookups
- Simple environments where SSSD is overkill
- Systems where you only need NSS caching (not authentication)

### When NOT to Use NSCD

- Systems running SSSD (they conflict)
- Active Directory domain-joined hosts (use SSSD instead)
- Systems using `systemd-resolved` for DNS (disable hosts cache)

## SSSD (System Security Services Daemon)

SSSD provides access to identity and authentication providers (LDAP, Active Directory, Kerberos, FreeIPA) with local caching. It handles NSS lookups AND PAM authentication, including offline login with cached credentials.

### Install on RHEL

```bash
# Basic SSSD
sudo dnf install -y sssd sssd-tools

# For Active Directory integration
sudo dnf install -y sssd sssd-ad sssd-tools realmd oddjob oddjob-mkhomedir \
    adcli samba-common-tools krb5-workstation

# For LDAP only
sudo dnf install -y sssd sssd-ldap sssd-tools

# Enable
sudo systemctl enable --now sssd
```

### Install on Ubuntu

```bash
# Basic SSSD
sudo apt install -y sssd sssd-tools

# For Active Directory integration
sudo apt install -y sssd sssd-ad sssd-tools realmd adcli krb5-user \
    samba-common-bin oddjob oddjob-mkhomedir

# For LDAP only
sudo apt install -y sssd sssd-ldap sssd-tools

# Enable
sudo systemctl enable --now sssd
```

### Join Active Directory Domain

```bash
# Discover the domain
realm discover EXAMPLE.COM

# Join the domain
sudo realm join -U admin@EXAMPLE.COM EXAMPLE.COM

# Verify
realm list
id user@example.com

# Allow all domain users to login
sudo realm permit --all

# Or allow specific groups
sudo realm permit -g "Domain Admins" "Linux Admins"
```

### SSSD Configuration (sssd.conf)

```bash
sudo vi /etc/sssd/sssd.conf
sudo chmod 600 /etc/sssd/sssd.conf
```

#### Active Directory Example

```ini
[sssd]
domains = example.com
config_file_version = 2
services = nss, pam

[domain/example.com]
id_provider = ad
access_provider = ad
auth_provider = ad
chpass_provider = ad

# Cache settings
cache_credentials = True
entry_cache_timeout = 300
enum_cache_timeout = 120

# UID/GID mapping (auto-generate from SID)
ldap_id_mapping = True

# Home directory and shell
fallback_homedir = /home/%u@%d
default_shell = /bin/bash
override_homedir = /home/%u

# Use short names (user instead of user@domain)
use_fully_qualified_names = False

# AD site (optional, for large environments)
# ad_site = MySite

# GPO-based access control
ad_gpo_access_control = enforcing
```

#### LDAP Example

```ini
[sssd]
domains = ldap.example.com
config_file_version = 2
services = nss, pam

[domain/ldap.example.com]
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap

ldap_uri = ldaps://ldap.example.com
ldap_search_base = dc=example,dc=com
ldap_default_bind_dn = cn=readonly,dc=example,dc=com
ldap_default_authtok = your_bind_password

# TLS settings
ldap_tls_reqcert = demand
ldap_tls_cacert = /etc/pki/tls/certs/ca-bundle.crt

# Cache
cache_credentials = True
entry_cache_timeout = 600

# Home directory
fallback_homedir = /home/%u
default_shell = /bin/bash

# Enumerate users (set False for large directories)
enumerate = False
```

#### FreeIPA Example

```ini
[sssd]
domains = ipa.example.com
config_file_version = 2
services = nss, pam, sudo

[domain/ipa.example.com]
id_provider = ipa
auth_provider = ipa
access_provider = ipa
chpass_provider = ipa

ipa_domain = ipa.example.com
ipa_server = ipa01.ipa.example.com
ipa_hostname = client.ipa.example.com

cache_credentials = True
krb5_store_password_if_offline = True
```

### SSSD Management Commands

```bash
# Status
sudo systemctl status sssd

# Restart (needed after config changes)
sudo systemctl restart sssd

# Clear all caches
sudo sss_cache -E

# Clear specific caches
sudo sss_cache -u username      # User cache
sudo sss_cache -g groupname     # Group cache
sudo sss_cache -U               # All users
sudo sss_cache -G               # All groups

# List cached users
sudo sssctl user-show username

# Check domain status
sudo sssctl domain-status example.com

# Check online/offline status
sudo sssctl domain-status example.com --online

# Verify configuration
sudo sssctl config-check

# Debug connection issues
sudo sssctl domain-status example.com --active-server
```

### NSS and PAM Configuration

#### /etc/nsswitch.conf

```bash
# With SSSD
passwd:     files sss
shadow:     files sss
group:      files sss
services:   files sss
netgroup:   files sss
automount:  files sss
sudoers:    files sss
```

#### PAM Configuration

On RHEL, `authselect` manages PAM:

```bash
# Enable SSSD profile
sudo authselect select sssd with-mkhomedir --force

# On Ubuntu, PAM is configured during sssd installation
# Or manually:
sudo pam-auth-update --enable sss
```

#### Auto-Create Home Directories

```bash
# RHEL
sudo authselect select sssd with-mkhomedir --force
sudo systemctl enable --now oddjobd

# Ubuntu
sudo pam-auth-update --enable mkhomedir
```

### SSSD Cache and Offline Login

```bash
# Enable offline authentication (in sssd.conf)
# cache_credentials = True
# krb5_store_password_if_offline = True

# Check if system is online or offline
sudo sssctl domain-status example.com

# Force offline mode (testing)
sudo sss_override user-add testuser --shell /bin/bash

# View cached credentials
sudo ldbsearch -H /var/lib/sss/db/cache_example.com.ldb
```

### SSSD Debug Logging

```bash
# Increase debug level in sssd.conf
# Add to [domain/example.com] section:
debug_level = 6

# Or set temporarily
sudo sssctl debug-level 6

# View logs
sudo tail -f /var/log/sssd/sssd_example.com.log
sudo tail -f /var/log/sssd/sssd_nss.log
sudo tail -f /var/log/sssd/sssd_pam.log

# Reset debug level
sudo sssctl debug-level 0
```

### SSSD Troubleshooting

```bash
# "No such user" after joining domain
sudo sss_cache -E
sudo systemctl restart sssd
id user@example.com

# Check connectivity to AD
sudo adcli info example.com

# Test Kerberos
kinit admin@EXAMPLE.COM
klist

# Verify LDAP connectivity
ldapsearch -H ldaps://ldap.example.com -D "cn=readonly,dc=example,dc=com" -W -b "dc=example,dc=com" "(uid=testuser)"

# Permission errors
ls -la /etc/sssd/sssd.conf
# Must be 0600 owned by root:root

# SELinux issues (RHEL)
sudo ausearch -m avc -ts recent | grep sssd
sudo setsebool -P authlogin_nsswitch_use_ldap on
```

## NSCD vs SSSD — When to Use Which

| Scenario | Recommendation |
|----------|---------------|
| Simple DNS/passwd caching | NSCD |
| Active Directory domain member | SSSD |
| LDAP authentication | SSSD |
| FreeIPA/IdM client | SSSD |
| NIS caching | NSCD |
| Offline login needed | SSSD |
| Only need host/DNS caching | NSCD (or systemd-resolved) |
| Multi-factor authentication | SSSD |
| sudo rules from LDAP/AD | SSSD |
| Standalone server, no domain | NSCD (or neither) |

## Disable NSCD When Using SSSD

NSCD and SSSD should not run simultaneously — they cache the same data and can produce stale results.

```bash
# Stop and disable NSCD
sudo systemctl stop nscd
sudo systemctl disable nscd
sudo systemctl mask nscd    # Prevent accidental re-enable

# Verify only SSSD is running
systemctl status sssd
systemctl status nscd
```

If you must keep NSCD running (rare edge case), disable passwd/group caching:

```bash
# /etc/nscd.conf — disable overlap with SSSD
enable-cache    passwd      no
enable-cache    group       no
enable-cache    netgroup    no
# Keep only hosts if needed
enable-cache    hosts       yes
```

## Quick Reference

| Action | Command |
|--------|---------|
| Install NSCD (RHEL) | `sudo dnf install nscd` |
| Install NSCD (Ubuntu) | `sudo apt install nscd` |
| Install SSSD (RHEL) | `sudo dnf install sssd sssd-tools` |
| Install SSSD (Ubuntu) | `sudo apt install sssd sssd-tools` |
| Flush NSCD cache | `sudo nscd -i passwd && sudo nscd -i group && sudo nscd -i hosts` |
| Flush SSSD cache | `sudo sss_cache -E` |
| NSCD stats | `sudo nscd -g` |
| SSSD domain status | `sudo sssctl domain-status example.com` |
| SSSD config check | `sudo sssctl config-check` |
| Join AD domain | `sudo realm join EXAMPLE.COM` |
| List domain | `realm list` |
| Test user lookup | `id user@example.com` or `getent passwd username` |
| Debug SSSD | `sudo sssctl debug-level 6` |
| SSSD logs | `/var/log/sssd/` |
