# resolvectl Cheatsheet

`resolvectl` is the CLI for `systemd-resolved`, the systemd DNS resolver service. It manages DNS resolution, caching, mDNS, and LLMNR on modern Linux systems. It replaces the older `systemd-resolve` command.

## Overview

`systemd-resolved` provides:

- DNS stub listener on `127.0.0.53:53`
- Per-link DNS configuration
- DNSSEC validation
- DNS-over-TLS (DoT)
- mDNS and LLMNR
- DNS caching

## Basic Commands

### Query a Hostname

```bash
# Resolve a hostname
resolvectl query example.com

# Resolve IPv4 only
resolvectl query -4 example.com

# Resolve IPv6 only
resolvectl query -6 example.com

# Reverse lookup (IP → hostname)
resolvectl query 8.8.8.8

# Query a specific record type
resolvectl query --type=MX example.com
resolvectl query --type=TXT example.com
resolvectl query --type=NS example.com
resolvectl query --type=SOA example.com
resolvectl query --type=SRV _ldap._tcp.example.com
resolvectl query --type=AAAA example.com
```

### Advanced Query Options

```bash
# Query with specific protocol
resolvectl query --protocol=dns example.com
resolvectl query --protocol=llmnr hostname.local
resolvectl query --protocol=mdns device.local

# Query using a specific interface
resolvectl query --interface=eth0 example.com

# Query with different DNS class (systemd 252+)
resolvectl query --class=IN example.com
resolvectl query --class=CH version.bind    # query BIND version
```

### Show DNS Status

```bash
# Show complete DNS configuration (all interfaces)
resolvectl status

# Show status for a specific interface
resolvectl status eth0

# Machine-readable (no pager)
resolvectl status --no-pager
```

Example output:

```
Global
       Protocols: +LLMNR +mDNS -DNSOverTLS DNSSEC=no/unsupported
resolv.conf mode: stub
     DNS Servers: 8.8.8.8 8.8.4.4
      DNS Domain: ~.

Link 2 (eth0)
    Current Scopes: DNS LLMNR/IPv4 LLMNR/IPv6
         Protocols: +DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 192.168.50.10
       DNS Servers: 192.168.50.10 8.8.8.8
        DNS Domain: homelab.local
```

## Cache Management

```bash
# Show cache statistics
resolvectl statistics

# Show cache contents (systemd 254+)
resolvectl show-cache

# Flush all caches
resolvectl flush-caches

# Reset statistics counters
resolvectl reset-statistics
```

Example statistics output:

```
DNSSEC supported by current servers: no

Transactions
Current Transactions: 0
  Total Transactions: 1234

Cache
  Current Cache Size: 42
          Cache Hits: 890
        Cache Misses: 344

DNSSEC Verdicts
              Secure: 0
            Insecure: 1234
               Bogus: 0
       Indeterminate: 0
```

## DNS Server Configuration

### View Current DNS Servers

```bash
# Per-interface DNS servers
resolvectl dns

# Show for a specific interface
resolvectl dns eth0
```

### Set DNS Servers

```bash
# Set DNS servers for an interface
resolvectl dns eth0 192.168.50.10 8.8.8.8

# Set global fallback DNS
resolvectl dns 8.8.8.8 1.1.1.1

# Set DNS server with custom port (systemd 246+)
resolvectl dns eth0 127.0.0.1:5353

# Clear DNS servers for an interface
resolvectl dns eth0 ""

# Revert to DHCP-provided DNS
resolvectl revert eth0
```

### Set Search Domains

```bash
# View current search domains
resolvectl domain

# Set search domain for an interface
resolvectl domain eth0 homelab.local

# Set routing domain (queries for this domain go to this link's DNS)
resolvectl domain eth0 ~homelab.local

# Set default route (all queries go through this link)
resolvectl domain eth0 ~.

# Multiple domains
resolvectl domain eth0 homelab.local ~internal.corp

# Clear domains for an interface
resolvectl domain eth0 ""
```

### The `~` Prefix (Routing Domains)

| Setting | Meaning |
|---------|---------|
| `homelab.local` | Search domain — short names get `.homelab.local` appended |
| `~homelab.local` | Routing domain — queries for `*.homelab.local` use this link's DNS |
| `~.` | Default route — all DNS queries go through this link |

## DNSSEC

```bash
# View DNSSEC status (all interfaces)
resolvectl dnssec

# Check DNSSEC status for a specific interface
resolvectl dnssec eth0
resolvectl status | grep DNSSEC

# Set DNSSEC mode for an interface
resolvectl dnssec eth0 yes        # validate
resolvectl dnssec eth0 no         # don't validate
resolvectl dnssec eth0 allow-downgrade  # try DNSSEC, fallback to insecure

# Query with DNSSEC info shown
resolvectl query --type=A example.com
```

## DNS-over-TLS (DoT)

```bash
# View DoT status (all interfaces)
resolvectl dnsovertls

# Check DoT status for a specific interface
resolvectl dnsovertls eth0
resolvectl status | grep DNSOverTLS

# Enable DNS-over-TLS for an interface
resolvectl dnsovertls eth0 yes         # strict (fail if DoT unavailable)
resolvectl dnsovertls eth0 opportunistic  # try DoT, fallback to plain
resolvectl dnsovertls eth0 no          # disable
```

For permanent configuration, edit `/etc/systemd/resolved.conf`:

```ini
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com 8.8.8.8#dns.google
DNSOverTLS=opportunistic
```

## mDNS and LLMNR

```bash
# Check mDNS/LLMNR status
resolvectl status | grep -E 'mDNS|LLMNR'

# Enable/disable mDNS for an interface
resolvectl mdns eth0 yes
resolvectl mdns eth0 no
resolvectl mdns eth0 resolve    # resolve only (don't respond)

# Enable/disable LLMNR for an interface
resolvectl llmnr eth0 yes
resolvectl llmnr eth0 no
resolvectl llmnr eth0 resolve
```

## Monitor DNS Queries

```bash
# Live monitor of DNS queries (useful for debugging)
resolvectl monitor
```

This shows real-time DNS queries and responses passing through systemd-resolved.

## systemd-resolved Configuration File

### /etc/systemd/resolved.conf

```ini
[Resolve]
# DNS servers (space-separated, optional #server-name for DoT)
DNS=192.168.50.10 8.8.8.8#dns.google

# Fallback DNS (used when per-link DNS is unavailable)
FallbackDNS=1.1.1.1 9.9.9.9

# Search domains
Domains=homelab.local

# DNSSEC validation
DNSSEC=allow-downgrade

# DNS-over-TLS
DNSOverTLS=opportunistic

# mDNS support (yes, no, resolve)
MulticastDNS=yes

# LLMNR support (yes, no, resolve)
LLMNR=yes

# Cache size (0 to disable)
Cache=yes
CacheFromLocalhost=no

# DNS stub listener (127.0.0.53)
DNSStubListener=yes
DNSStubListenerExtra=

# Read /etc/hosts
ReadEtcHosts=yes
```

After editing:

```bash
systemctl restart systemd-resolved
resolvectl status
```

## How /etc/resolv.conf Works with systemd-resolved

systemd-resolved can manage `/etc/resolv.conf` in several modes:

```bash
# Check what mode is active
ls -la /etc/resolv.conf
resolvectl status | grep "resolv.conf mode"
```

| Mode | resolv.conf points to | Behavior |
|------|----------------------|----------|
| stub | `/run/systemd/resolve/stub-resolv.conf` | `nameserver 127.0.0.53` — all queries go through systemd-resolved (recommended) |
| static | `/run/systemd/resolve/resolv.conf` | Lists actual upstream servers — bypasses caching |
| foreign | manually managed | systemd-resolved doesn't touch it |

### Set Up Stub Resolver (Recommended)

```bash
# Symlink resolv.conf to the stub
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf

# Restart
systemctl restart systemd-resolved
```

### Bypass systemd-resolved (Use Direct DNS)

```bash
# Point to actual resolvers (skips caching)
ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf

# Or manage manually
rm /etc/resolv.conf
cat > /etc/resolv.conf << EOF
nameserver 8.8.8.8
nameserver 1.1.1.1
search homelab.local
EOF
```

## Integration with NetworkManager

NetworkManager and systemd-resolved work together:

```bash
# Check if NM is pushing DNS to resolved
nmcli dev show | grep DNS

# Set DNS via NetworkManager (automatically configures resolved)
nmcli connection modify "eth0" ipv4.dns "192.168.50.10 8.8.8.8"
nmcli connection modify "eth0" ipv4.dns-search "homelab.local"
nmcli connection up "eth0"

# Verify resolved picked it up
resolvectl status eth0
```

## Legacy Command: systemd-resolve

On older systems (before systemd 239), use `systemd-resolve`:

```bash
# Same as resolvectl query
systemd-resolve example.com

# Same as resolvectl status
systemd-resolve --status

# Same as resolvectl flush-caches
systemd-resolve --flush-caches

# Same as resolvectl statistics
systemd-resolve --statistics

# Same as resolvectl query --type=MX
systemd-resolve --type=MX example.com
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `resolv.conf` has wrong nameserver | Symlink broken or NM override | `ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf` |
| Queries timeout | systemd-resolved not running | `systemctl start systemd-resolved` |
| Can't resolve internal names | Search domain not set | `resolvectl domain eth0 homelab.local` |
| DoT not working | Server doesn't support it or firewall blocks 853 | Try `opportunistic` mode, check port 853 |
| Cache not working | `Cache=no` in config or stub not used | Check resolved.conf and resolv.conf symlink |
| Conflicts with dnsmasq/bind | Port 53 already in use | Disable `DNSStubListener` or stop conflicting service |

### Debug Steps

```bash
# Check if resolved is running
systemctl status systemd-resolved

# Check what's listening on port 53
ss -tulnp | grep :53

# Check resolv.conf state
ls -la /etc/resolv.conf
cat /etc/resolv.conf

# Check per-interface DNS
resolvectl dns
resolvectl domain

# Test resolution through resolved
resolvectl query example.com

# Test resolution bypassing resolved (direct DNS)
dig @8.8.8.8 example.com +short

# Monitor queries in real time
resolvectl monitor

# Check logs
journalctl -u systemd-resolved -f
```

### Disable systemd-resolved (If Not Wanted)

```bash
systemctl disable --now systemd-resolved
rm /etc/resolv.conf
# Create a static resolv.conf
cat > /etc/resolv.conf << EOF
nameserver 8.8.8.8
nameserver 1.1.1.1
search homelab.local
EOF
```

## Common DNS Provider Configurations

```bash
# Cloudflare DNS
resolvectl dns eth0 1.1.1.1 1.0.0.1

# Google DNS
resolvectl dns eth0 8.8.8.8 8.8.4.4

# Quad9 DNS
resolvectl dns eth0 9.9.9.9 149.112.112.112

# Local development
resolvectl dns eth0 127.0.0.1
resolvectl domain eth0 ~local.dev
```

## Service Management

```bash
# Check systemd-resolved status
systemctl status systemd-resolved

# Restart DNS resolver (required after config changes)
systemctl restart systemd-resolved

# Enable at boot
systemctl enable systemd-resolved
```

## Per-Interface Network Configuration

For persistent per-interface DNS settings using systemd-networkd:

```bash
# View network files
ls /etc/systemd/network/

# Example: /etc/systemd/network/20-wired.network
# [Network]
# DNS=192.168.50.10 8.8.8.8
# Domains=homelab.local ~.
# DNSSEC=allow-downgrade
# DNSOverTLS=opportunistic
```

## Journal Monitoring

```bash
# Monitor systemd-resolved logs in real time
journalctl -u systemd-resolved -f

# View recent DNS logs
journalctl -u systemd-resolved --since "1 hour ago"

# Show DNS query logs (if debug enabled)
journalctl -u systemd-resolved | grep -i "question\|answer"

# Backup current DNS settings
resolvectl status > dns-backup-$(date +%Y%m%d).txt
```

## Useful One-Liners

```bash
# Quick DNS test
resolvectl query google.com && echo "DNS working"

# Show all DNS servers across interfaces
resolvectl status | grep "DNS Servers"

# Check if DNSSEC validation works (should fail)
resolvectl query --verbose sigfail.verteiltesysteme.net

# Show all DNS settings at once
resolvectl dns && resolvectl domain && resolvectl dnssec
```

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Generic failure |
| 2 | Invalid arguments |
| 3 | DNS resolution failed |
| 4 | Network error |

## Notes

- Changes made with `resolvectl` are **temporary** and reset after reboot
- For persistent changes, modify `/etc/systemd/resolved.conf` or use NetworkManager
- Some distributions use `systemd-resolve` instead of `resolvectl` (older systemd versions)
- DNS settings can be overridden by NetworkManager or other network management tools
- `resolvectl revert <interface>` restores DHCP-provided settings

## Quick Reference

| Task | Command |
|------|---------|
| Query hostname | `resolvectl query example.com` |
| Show DNS config | `resolvectl status` |
| Show cache stats | `resolvectl statistics` |
| Flush DNS cache | `resolvectl flush-caches` |
| Set DNS server | `resolvectl dns eth0 8.8.8.8` |
| Set search domain | `resolvectl domain eth0 homelab.local` |
| Set routing domain | `resolvectl domain eth0 ~homelab.local` |
| Enable DoT | `resolvectl dnsovertls eth0 opportunistic` |
| Enable DNSSEC | `resolvectl dnssec eth0 yes` |
| Monitor queries | `resolvectl monitor` |
| Revert to DHCP DNS | `resolvectl revert eth0` |
| Check resolv.conf | `ls -la /etc/resolv.conf` |
