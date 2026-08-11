# DNS Commands Cheatsheet

## dig

`dig` (Domain Information Groper) is the most powerful DNS lookup tool. It queries DNS servers and displays detailed responses.

### Basic Queries

```bash
# Query A record (default)
dig example.com

# Query specific record type
dig example.com A
dig example.com AAAA
dig example.com MX
dig example.com NS
dig example.com TXT
dig example.com SOA
dig example.com CNAME
dig example.com SRV
dig example.com PTR
dig example.com CAA

# Query ANY (all records) — may not work on all servers
dig example.com ANY
```

### Short Output

```bash
# Short answer only (just the result)
dig +short example.com
dig +short example.com MX
dig +short example.com NS

# Short with record type shown
dig +short +answer example.com ANY
```

### Query a Specific DNS Server

```bash
# Use a specific nameserver
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com
dig @ns1.example.com example.com

# Query the authoritative nameserver directly
dig +trace example.com
```

### Reverse DNS Lookup

```bash
# PTR record (IP → hostname)
dig -x 8.8.8.8
dig -x 192.168.1.1

# Short reverse
dig +short -x 8.8.8.8
```

### Trace (Follow Delegation Chain)

```bash
# Show the full resolution path from root servers
dig +trace example.com

# Trace with short output
dig +trace +short example.com
```

### Control Output Sections

```bash
# Show only the answer section
dig +noall +answer example.com

# Show answer + authority
dig +noall +answer +authority example.com

# Show all sections with comments
dig +all example.com

# No comments (cleaner output)
dig +nocomments example.com

# Show statistics
dig +stats example.com

# Show query time only
dig +noall +stats example.com
```

### TTL and Caching

```bash
# Show TTL values
dig +noall +answer +ttlid example.com

# Check if a record is cached (compare TTL with authoritative)
dig example.com          # TTL from resolver cache
dig @ns1.example.com example.com  # TTL from authoritative
```

### DNSSEC

```bash
# Query with DNSSEC validation
dig +dnssec example.com

# Check DNSSEC chain
dig +dnssec +trace example.com

# Query DNSKEY records
dig example.com DNSKEY

# Query DS records
dig example.com DS
```

### Zone Transfer (AXFR)

```bash
# Attempt a full zone transfer (must be allowed by the server)
dig @ns1.example.com example.com AXFR

# Incremental zone transfer
dig @ns1.example.com example.com IXFR=2024010100
```

### Batch Queries

```bash
# Query multiple domains from a file
dig -f domains.txt +short

# Query multiple record types
dig example.com A example.com MX example.com NS
```

### TCP Mode

```bash
# Force TCP (useful for large responses or debugging)
dig +tcp example.com

# Specify port
dig -p 5353 example.com

# Retry and timeout settings
dig example.com A +time=2 +tries=1
```

### Recursion Control

```bash
# Disable recursion (query like an authoritative server)
dig example.com A +norecurse

# Show AD (Authenticated Data) bit from validating resolver
dig example.com A +adflag

# Disable EDNS (debug broken middleboxes/firewalls)
dig example.com A +noedns
```

### EDNS Client Subnet (CDN Testing)

```bash
# Test what IP a CDN returns for a specific client location
dig @1.1.1.1 example.com A +subnet=8.8.8.8/32

# Useful for verifying geo-DNS or CDN routing behavior
dig @8.8.8.8 cdn.example.com A +subnet=203.0.113.1/24
```

### Output Formatting

```bash
# Machine-readable (no extra formatting)
dig +noall +answer +nocmd +noquestion example.com

# BIND format (for zone files)
dig +noall +answer example.com | awk '{print $1, $2, $3, $4, $5}'

# Multiline SOA (human-readable)
dig +multiline example.com SOA

# Display SOA from all authoritative nameservers
dig example.com +nssearch

# Use search list from /etc/resolv.conf
dig ftp +search

# Set default dig options (applied to every query)
echo "+noall +answer" > ~/.digrc
```

## host

`host` is a simpler DNS lookup tool — quick lookups with less verbose output.

### Basic Usage

```bash
# A record lookup
host example.com

# Reverse lookup
host 8.8.8.8

# Query specific record type
host -t MX example.com
host -t NS example.com
host -t TXT example.com
host -t SOA example.com
host -t AAAA example.com
host -t SRV _ldap._tcp.example.com
host -t CNAME www.example.com

# All records
host -a example.com
```

### Using a Specific Server

```bash
# Query a specific DNS server
host example.com 8.8.8.8
host example.com ns1.example.com
```

### Verbose Output

```bash
# Verbose mode (similar to dig)
host -v example.com

# Debug mode
host -d example.com

# Force TCP
host -T example.com
```

### Zone Transfer

```bash
# List all records in a zone (if allowed)
host -l example.com ns1.example.com

# Perform a zone transfer
host -a -l example.com

# Display SOA records from all authoritative nameservers
host -C example.com
```

## nslookup

`nslookup` is an older DNS lookup tool. It works in interactive and non-interactive modes.

### Non-Interactive Mode

```bash
# Basic lookup
nslookup example.com

# Reverse lookup
nslookup 8.8.8.8

# Query specific record type
nslookup -type=MX example.com
nslookup -type=NS example.com
nslookup -type=TXT example.com
nslookup -type=SOA example.com
nslookup -type=SRV _ldap._tcp.example.com

# Use a specific server
nslookup example.com 8.8.8.8
```

### Interactive Mode

```bash
$ nslookup
> server 8.8.8.8
> set type=MX
> example.com
> set type=NS
> example.com
> exit
```

### Debug Mode

```bash
nslookup -debug example.com

# Custom port
nslookup -port=5353 example.com 127.0.0.1
```

> **Note:** nslookup behavior differs across platforms. Prefer `dig` for scripting.

## getent

`getent` queries the system's name service switch (NSS) — respects `/etc/hosts`, `/etc/nsswitch.conf`, and configured resolvers.

```bash
# Resolve hostname (uses NSS, not just DNS)
getent hosts example.com
getent hosts server01

# Resolve by IP
getent hosts 192.168.1.1

# Show all hosts from /etc/hosts
getent hosts

# Query specific database
getent ahosts example.com     # all address records
getent ahostsv4 example.com   # IPv4 only
getent ahostsv6 example.com   # IPv6 only
```

## resolvectl / systemd-resolve

On systemd-based systems, `resolvectl` (formerly `systemd-resolve`) manages DNS resolution.

```bash
# Query a hostname
resolvectl query example.com

# Show DNS configuration
resolvectl status

# Show statistics
resolvectl statistics

# Flush DNS cache
resolvectl flush-caches

# Show cache statistics
resolvectl statistics | grep -i cache

# Query specific interface DNS
resolvectl dns eth0
```

For older systems:

```bash
systemd-resolve --status
systemd-resolve example.com
systemd-resolve --flush-caches
```

## DNS Configuration Files

### /etc/resolv.conf

```bash
# View current DNS config
cat /etc/resolv.conf

# Typical content:
# nameserver 8.8.8.8
# nameserver 8.8.4.4
# search example.com
# domain example.com
# options timeout:2 attempts:3
```

| Directive | Description |
|-----------|-------------|
| `nameserver` | DNS server IP (up to 3) |
| `search` | Search domains (appended to short hostnames) |
| `domain` | Default domain |
| `options` | Resolver options (timeout, attempts, rotate, ndots) |

### /etc/nsswitch.conf

Controls the order of name resolution:

```bash
# Typical entry
hosts: files dns myhostname

# files = /etc/hosts first
# dns = DNS query
# myhostname = resolve own hostname
```

### /etc/hosts

```bash
# Static hostname mappings (checked before DNS if 'files' is first in nsswitch)
127.0.0.1   localhost
192.168.1.10 server01.example.com server01
```

## DNS Record Types

| Type | Description | Example |
|------|-------------|---------|
| `A` | IPv4 address | `93.184.216.34` |
| `AAAA` | IPv6 address | `2606:2800:220:1:248:1893:25c8:1946` |
| `CNAME` | Canonical name (alias) | `www → example.com` |
| `MX` | Mail exchange (with priority) | `10 mail.example.com` |
| `NS` | Nameserver | `ns1.example.com` |
| `TXT` | Text (SPF, DKIM, verification) | `"v=spf1 include:..."` |
| `SOA` | Start of Authority | Serial, refresh, retry, expire |
| `PTR` | Pointer (reverse DNS) | `8.8.8.8 → dns.google` |
| `SRV` | Service locator | `_ldap._tcp 0 100 389 ldap.example.com` |
| `CAA` | Certificate Authority Authorization | `0 issue "letsencrypt.org"` |
| `DNSKEY` | DNSSEC public key | Used for DNSSEC validation |
| `DS` | Delegation Signer | Links parent to child DNSSEC |
| `NAPTR` | Naming Authority Pointer | Used in SIP/ENUM |

## Practical Examples

### Check if DNS is Working

```bash
# Quick test
dig +short google.com
host google.com
ping -c 1 google.com
```

### Find Mail Servers for a Domain

```bash
dig +short MX example.com
host -t MX example.com
```

### Find Nameservers for a Domain

```bash
dig +short NS example.com
host -t NS example.com
```

### Check SPF Record

```bash
dig +short TXT example.com | grep spf
```

### Check DKIM Record

```bash
dig +short TXT selector1._domainkey.example.com
```

### Check DMARC Record

```bash
dig +short TXT _dmarc.example.com
```

### Find IP of a Hostname

```bash
dig +short example.com
host example.com
getent hosts example.com
```

### Find Hostname of an IP

```bash
dig -x 8.8.8.8 +short
host 8.8.8.8
```

### Check DNS Propagation

```bash
# Query multiple public DNS servers
for ns in 8.8.8.8 1.1.1.1 9.9.9.9 208.67.222.222; do
  echo "$ns: $(dig +short @$ns example.com)"
done
```

### Measure DNS Response Time

```bash
# dig shows query time in the output
dig example.com | grep "Query time"

# Multiple queries to measure average
for i in {1..10}; do
  dig example.com | grep "Query time"
done
```

### Check SOA Serial (Zone Version)

```bash
dig +short SOA example.com
# ns1.example.com. admin.example.com. 2024010101 3600 900 604800 86400
# The 3rd field (2024010101) is the serial number
```

### Flush Local DNS Cache

```bash
# systemd-resolved
resolvectl flush-caches
# or
systemd-resolve --flush-caches

# nscd (if used)
nscd -i hosts

# dnsmasq (if used)
systemctl restart dnsmasq

# macOS
sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder
```

### Test DNS from a Container/Pod

```bash
# Kubernetes
kubectl run dns-test --rm -it --image=busybox -- nslookup kubernetes.default

# Docker
docker run --rm busybox nslookup google.com
```

## Troubleshooting

### Common DNS Issues

| Symptom | Check | Fix |
|---------|-------|-----|
| `NXDOMAIN` | Record doesn't exist | Verify domain/subdomain spelling |
| `SERVFAIL` | Server can't resolve | Check DNSSEC, upstream server |
| `REFUSED` | Server won't answer | Check ACLs, firewall |
| `connection timed out` | No response from server | Check network, firewall (port 53) |
| Resolves externally but not internally | Split DNS | Check `/etc/resolv.conf`, internal DNS |
| Stale/old records | Caching | Flush cache, check TTL |

### Debug DNS Resolution Order

```bash
# What resolver is being used?
cat /etc/resolv.conf

# What's the NSS order?
grep hosts /etc/nsswitch.conf

# Is it in /etc/hosts?
grep example.com /etc/hosts

# Is systemd-resolved managing DNS?
resolvectl status
ls -la /etc/resolv.conf   # is it a symlink?
```

### Test with Different Tools

```bash
# If dig works but ping doesn't → /etc/hosts or nsswitch issue
dig +short example.com     # queries DNS directly
getent hosts example.com   # uses NSS (hosts file + DNS)
ping -c 1 example.com     # uses NSS
```

## Installation

```bash
# dig, host, nslookup (bind-utils)
yum install bind-utils    # RHEL 6/7
dnf install bind-utils    # RHEL 8/9/10
apt install dnsutils      # Ubuntu/Debian

# Verify
which dig host nslookup
```

## drill (ldns)

`drill` is a DNS query tool from the ldns library, similar to dig but with better DNSSEC support:

```bash
# Basic lookup
drill example.com A

# DNSSEC details (chase the chain of trust)
drill -S example.com

# Query specific server
drill @1.1.1.1 example.com A

# Force TCP with custom port
drill -T -p 5353 @127.0.0.1 example.com A
```

Install: `dnf install ldns-utils` (RHEL) or `apt install ldnsutils` (Ubuntu)

## kdig (Knot DNS)

`kdig` is a dig-like tool from the Knot DNS project with modern DNS transport support:

```bash
# Basic lookup with DNSSEC
kdig @1.1.1.1 example.com A +dnssec

# Trace delegation
kdig +trace example.com

# DNS-over-TLS (DoT)
kdig +tls @1.1.1.1 +tls-host=cloudflare-dns.com example.com A

# TCP with custom port
kdig +tcp -p 5353 @127.0.0.1 example.com A
```

Install: `dnf install knot-utils` (RHEL) or `apt install knot-dnsutils` (Ubuntu)

## Windows DNS Commands

### PowerShell: Resolve-DnsName

```powershell
# Basic lookup
Resolve-DnsName example.com -Type A

# Use specific server
Resolve-DnsName example.com -Type MX -Server 1.1.1.1

# Reverse lookup
Resolve-DnsName 8.8.8.8 -Type PTR

# TCP only
Resolve-DnsName example.com -Type A -TcpOnly

# DNSSEC OK bit
Resolve-DnsName example.com -DnssecOk

# Bypass hosts file / LLMNR / NetBIOS
Resolve-DnsName example.com -DnsOnly -NoHostsFile

# Show configured DNS servers
Get-DnsClientServerAddress
```

### Classic Windows Commands

```cmd
:: Flush DNS cache
ipconfig /flushdns

:: Display DNS cache
ipconfig /displaydns

:: Legacy lookup
nslookup example.com 1.1.1.1
```

## macOS DNS

```bash
# Show all DNS configuration (resolvers, search domains)
scutil --dns

# Flush DNS cache
sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder

# Query (dig and host work the same as Linux)
dig +short example.com
host example.com
```

## NetworkManager DNS

```bash
# Show DNS servers configured by NetworkManager
nmcli dev show | grep -i dns

# Show connection details
nmcli connection show "eth0" | grep -i dns

# Set DNS for a connection
nmcli connection modify "eth0" ipv4.dns "8.8.8.8 8.8.4.4"
nmcli connection up "eth0"

# Docker container resolver
docker exec -it <container> cat /etc/resolv.conf
```

## Packet Capture (DNS Traffic)

When responses look wrong or to debug network-level issues:

```bash
# Capture DNS traffic (UDP/TCP port 53)
sudo tcpdump -ni any port 53 -vvv -s0

# Capture DNS-over-TLS (port 853)
sudo tcpdump -ni any port 853 -vvv -s0

# Capture only queries to a specific server
sudo tcpdump -ni any host 8.8.8.8 and port 53

# Write to pcap file for Wireshark analysis
sudo tcpdump -ni any port 53 -w /tmp/dns_capture.pcap
```

Wireshark display filters:

| Filter | Protocol |
|--------|----------|
| `dns` | Standard DNS (port 53) |
| `tcp.port == 853` | DNS-over-TLS (DoT) |
| `dns.flags.response == 0` | DNS queries only |
| `dns.flags.response == 1` | DNS responses only |
| `dns.flags.rcode != 0` | Error responses |

> **Tip:** If responses get truncated (TC bit set), the client should retry over TCP: `dig +tcp ...`

## Delegation and Authority Checks

```bash
# End-to-end delegation path
dig +trace example.com

# Check NS records at parent (TLD servers)
dig ns example.com @a.gtld-servers.net

# Check NS records at child (authoritative)
dig ns example.com @ns1.example.com

# Verify glue records (A/AAAA for nameservers)
dig ns example.com +short | xargs -I{} dig +short {} A
dig ns example.com +short | xargs -I{} dig +short {} AAAA

# Compare SOA serial across resolvers (propagation check)
for ns in $(dig +short NS example.com); do
  echo "$ns: $(dig +short SOA example.com @$ns | awk '{print $3}')"
done
```

## Performance and Latency

```bash
# Measure DNS resolution time
time dig @1.1.1.1 example.com A +tries=1 +time=1 +nocmd +noquestion +stats

# Check resolver caching (compare TTLs)
dig example.com A +noall +answer    # from resolver (decreasing TTL = cached)
dig @ns1.example.com example.com A +noall +answer   # from authoritative (full TTL)

# Force query only authoritative servers (no recursion)
dig +norecurse example.com A
```

## Common Troubleshooting Patterns

### 1. Name resolves externally but not internally

- Check split-horizon DNS: query internal and external resolvers separately
- Inspect `search` domains and `ndots` in `/etc/resolv.conf`
- Verify `/etc/nsswitch.conf` order

### 2. Intermittent SERVFAIL

```bash
# Try disabling EDNS (broken middleboxes/firewalls)
dig +noedns example.com A

# Try TCP
dig +tcp example.com A

# Check DNSSEC chain
dig example.com DS @a.gtld-servers.net +dnssec
```

### 3. Wrong IP returned via CDN

```bash
# Test with different client subnets
dig @1.1.1.1 example.com A +subnet=203.0.113.1/24
dig @8.8.8.8 example.com A +subnet=198.51.100.1/24

# Query from different regional resolvers
dig @1.1.1.1 example.com A
dig @8.8.8.8 example.com A
dig @9.9.9.9 example.com A
```

### 4. PTR mismatch warnings

```bash
# Ensure reverse zone matches forward
dig -x <ip> +short                    # should return hostname
dig $(dig -x <ip> +short) +short     # should return original IP
```

### 5. Zone transfer open (security issue)

```bash
# Test if AXFR is allowed (should be restricted)
dig @ns1.example.com example.com AXFR
```

### 6. MX delivery issues

```bash
# Verify MX targets have A/AAAA records
dig +short MX example.com
dig +short $(dig +short MX example.com | awk '{print $2}') A

# Check SPF/DKIM/DMARC
dig +short TXT example.com | grep spf
dig +short TXT default._domainkey.example.com
dig +short TXT _dmarc.example.com
```

## BIND/named Server Administration

Commands for managing a BIND DNS server:

```bash
# Check named service status
service named status
systemctl status named

# Check zone file syntax and integrity
named-checkzone example.com /var/named/db.example.com

# Check configuration file syntax
named-checkconf
named-checkconf -t /var/named/chroot    # if running in chroot

# Report BIND version and build options
named -V

# Check if named is running under chroot
ps -ef | grep named

# Reload configuration file and zones
rndc reload

# Reload a specific zone
rndc reload example.com

# Toggle query logging (check /var/log/messages; same command to disable)
rndc querylog

# Flush the server cache
rndc flush

# Show server status
rndc status

# Print the domain part of the FQDN
dnsdomainname
```

## Notes

- Some corporate networks intercept or filter DNS/DoT/DoH traffic
- `ANY` queries are often minimized or blocked by resolvers — always request the specific type
- For privacy, prefer trusted resolvers and be mindful of EDNS Client Subnet (`+subnet`) usage
- When comparing resolvers, differences may be due to caching, geo-DNS, or split-horizon

## Quick Reference

| Task | Command |
|------|---------|
| A record | `dig +short example.com` |
| MX record | `dig +short MX example.com` |
| NS record | `dig +short NS example.com` |
| TXT record | `dig +short TXT example.com` |
| Reverse lookup | `dig -x 8.8.8.8 +short` |
| Use specific server | `dig @8.8.8.8 example.com` |
| Trace resolution | `dig +trace example.com` |
| Check TTLs | `dig +noall +answer example.com A` |
| Test with TCP | `dig +tcp example.com A` |
| Disable EDNS | `dig +noedns example.com A` |
| DNSSEC signal | `dig +dnssec example.com A` |
| Disable recursion | `dig +norecurse example.com A` |
| CDN testing (ECS) | `dig @1.1.1.1 example.com A +subnet=8.8.8.8/32` |
| Zone transfer | `dig @ns1.example.com example.com AXFR` |
| Simple lookup | `host example.com` |
| Flush cache (Linux) | `resolvectl flush-caches` |
| Flush cache (macOS) | `sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder` |
| Flush cache (Windows) | `ipconfig /flushdns` |
| Check resolver | `cat /etc/resolv.conf` |
| Windows lookup | `Resolve-DnsName example.com -Type A -Server 1.1.1.1` |
| Capture DNS traffic | `sudo tcpdump -ni any port 53 -vvv` |
