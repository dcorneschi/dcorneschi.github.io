# Nmap Cheatsheet

Nmap (Network Mapper) is an open-source tool for network discovery, port scanning, service detection, and security auditing.

## Installation

```bash
# RHEL/CentOS/Fedora
sudo dnf install -y nmap

# Ubuntu/Debian
sudo apt install -y nmap

# macOS
brew install nmap

# Verify
nmap --version
```

## Basic Scanning

```bash
# Scan a single host
nmap 192.168.1.1

# Scan a hostname
nmap example.com

# Scan multiple hosts
nmap 192.168.1.1 192.168.1.2 192.168.1.3

# Scan a range
nmap 192.168.1.1-50

# Scan a subnet
nmap 192.168.1.0/24

# Scan from a file
nmap -iL hosts.txt

# Exclude hosts
nmap 192.168.1.0/24 --exclude 192.168.1.1,192.168.1.2

# Exclude from file
nmap 192.168.1.0/24 --excludefile exclude.txt
```

## Host Discovery (Ping Scanning)

```bash
# Ping scan only (no port scan)
nmap -sn 192.168.1.0/24

# Skip host discovery (scan even if host appears down)
nmap -Pn 192.168.1.1

# TCP SYN ping on port 443
nmap -PS443 192.168.1.1

# TCP ACK ping on port 80
nmap -PA80 192.168.1.1

# UDP ping
nmap -PU53 192.168.1.1

# ICMP echo ping
nmap -PE 192.168.1.0/24

# ICMP timestamp discovery
nmap -PP 192.168.1.1

# ICMP netmask discovery
nmap -PM 192.168.1.1

# ARP ping (local network only, very fast)
nmap -PR 192.168.1.0/24

# Combination ping
nmap -PS22,80,443 -PA80 -PE 192.168.1.0/24

# List targets without scanning
nmap -sL 192.168.1.0/24
```

## Port Scanning Techniques

### Scan Types

```bash
# TCP SYN scan (default, requires root, stealthy)
sudo nmap -sS 192.168.1.1

# TCP connect scan (no root required, full handshake)
nmap -sT 192.168.1.1

# UDP scan (slow)
sudo nmap -sU 192.168.1.1

# TCP ACK scan (detect firewall rules, not open ports)
sudo nmap -sA 192.168.1.1

# TCP FIN scan (stealthy, bypasses some firewalls)
sudo nmap -sF 192.168.1.1

# TCP Xmas scan (FIN+PSH+URG flags)
sudo nmap -sX 192.168.1.1

# TCP NULL scan (no flags set)
sudo nmap -sN 192.168.1.1

# Window scan (like ACK but detects open ports on some systems)
sudo nmap -sW 192.168.1.1

# Maimon scan (FIN/ACK)
sudo nmap -sM 192.168.1.1

# IP protocol scan (find supported IP protocols)
sudo nmap -sO 192.168.1.1

# Combined TCP + UDP
sudo nmap -sS -sU 192.168.1.1
```

### Port Specification

```bash
# Specific ports
nmap -p 22,80,443 192.168.1.1

# Port range
nmap -p 1-1000 192.168.1.1

# All ports (1-65535)
nmap -p- 192.168.1.1

# Top N ports (by popularity)
nmap --top-ports 100 192.168.1.1

# Specific TCP and UDP ports
nmap -p T:22,80,443,U:53,161 192.168.1.1

# Fast scan (top 100 ports)
nmap -F 192.168.1.1

# Scan ports by service name
nmap -p http,https,ssh 192.168.1.1
```

## Service and Version Detection

```bash
# Service version detection
nmap -sV 192.168.1.1

# Aggressive version detection
nmap -sV --version-intensity 5 192.168.1.1

# Light version detection (faster)
nmap -sV --version-light 192.168.1.1

# OS detection
sudo nmap -O 192.168.1.1

# Aggressive OS detection
sudo nmap -O --osscan-guess 192.168.1.1

# Combined service + OS detection
sudo nmap -sV -O 192.168.1.1

# Aggressive scan (OS + version + scripts + traceroute)
nmap -A 192.168.1.1
```

## Nmap Scripting Engine (NSE)

```bash
# Run default scripts
nmap -sC 192.168.1.1

# Run specific script
nmap --script=http-title 192.168.1.1

# Run script category
nmap --script=vuln 192.168.1.1
nmap --script=safe 192.168.1.1
nmap --script=discovery 192.168.1.1

# Run multiple scripts
nmap --script=http-title,http-headers 192.168.1.1

# Run scripts with arguments
nmap --script=http-brute --script-args='userdb=users.txt,passdb=pass.txt' 192.168.1.1

# List available scripts
ls /usr/share/nmap/scripts/
nmap --script-help=http-*

# Update script database
nmap --script-updatedb
```

### Common NSE Scripts

```bash
# HTTP
nmap --script=http-title -p 80 192.168.1.1
nmap --script=http-headers -p 80 192.168.1.1
nmap --script=http-enum -p 80 192.168.1.1
nmap --script=http-methods -p 80 192.168.1.1
nmap --script=http-robots.txt -p 80 192.168.1.1

# SSL/TLS
nmap --script=ssl-cert -p 443 192.168.1.1
nmap --script=ssl-enum-ciphers -p 443 192.168.1.1
nmap --script=ssl-heartbleed -p 443 192.168.1.1

# SMB
nmap --script=smb-os-discovery -p 445 192.168.1.1
nmap --script=smb-enum-shares -p 445 192.168.1.1
nmap --script=smb-vuln-* -p 445 192.168.1.1

# DNS
nmap --script=dns-brute example.com
nmap --script=dns-zone-transfer --script-args dns-zone-transfer.domain=example.com -p 53 ns.example.com

# SSH
nmap --script=ssh-auth-methods -p 22 192.168.1.1
nmap --script=ssh2-enum-algos -p 22 192.168.1.1

# MySQL
nmap --script=mysql-info -p 3306 192.168.1.1
nmap --script=mysql-enum -p 3306 192.168.1.1

# SNMP
nmap --script=snmp-info -sU -p 161 192.168.1.1
nmap --script=snmp-brute -sU -p 161 192.168.1.1

# Vulnerability scanning
nmap --script=vuln 192.168.1.1
nmap --script=vulscan 192.168.1.1
```

### NSE Script Categories

| Category | Purpose |
|----------|---------|
| `auth` | Authentication bypass/testing |
| `broadcast` | Discover hosts via broadcast |
| `brute` | Brute-force credentials |
| `default` | Safe, informational scripts (-sC) |
| `discovery` | Service/host discovery |
| `dos` | Denial of service (use carefully) |
| `exploit` | Exploit vulnerabilities |
| `external` | Query external services |
| `fuzzer` | Fuzz testing |
| `intrusive` | May crash services |
| `malware` | Detect malware |
| `safe` | Won't crash or harm targets |
| `version` | Version detection |
| `vuln` | Vulnerability detection |

## Timing and Performance

```bash
# Timing templates (T0=paranoid to T5=insane)
nmap -T0 192.168.1.1    # Paranoid (very slow, IDS evasion)
nmap -T1 192.168.1.1    # Sneaky
nmap -T2 192.168.1.1    # Polite
nmap -T3 192.168.1.1    # Normal (default)
nmap -T4 192.168.1.1    # Aggressive (recommended for fast networks)
nmap -T5 192.168.1.1    # Insane (may miss ports)

# Custom timing
nmap --min-rate 1000 192.168.1.0/24        # Min packets/second
nmap --max-rate 500 192.168.1.0/24         # Max packets/second
nmap --max-retries 2 192.168.1.0/24        # Reduce retries
nmap --host-timeout 30s 192.168.1.0/24     # Timeout per host
nmap --scan-delay 1s 192.168.1.1           # Delay between probes

# Parallelism
nmap --min-parallelism 10 192.168.1.0/24
nmap --max-parallelism 100 192.168.1.0/24
```

## Output Formats

```bash
# Normal output to file
nmap -oN scan.txt 192.168.1.1

# XML output
nmap -oX scan.xml 192.168.1.1

# Grepable output
nmap -oG scan.gnmap 192.168.1.1

# All formats at once
nmap -oA scan_results 192.168.1.1

# Verbose output
nmap -v 192.168.1.1
nmap -vv 192.168.1.1    # More verbose

# Debug output
nmap -d 192.168.1.1
nmap -dd 192.168.1.1    # More debug

# Append to file
nmap --append-output -oN scan.txt 192.168.1.2

# Script kiddie output (for fun)
nmap -oS scan.txt 192.168.1.1
```

## Firewall/IDS Evasion

```bash
# Fragment packets
sudo nmap -f 192.168.1.1

# Set specific MTU
sudo nmap --mtu 24 192.168.1.1

# Use decoys (appear as multiple scanners)
sudo nmap -D RND:5 192.168.1.1
sudo nmap -D 10.0.0.1,10.0.0.2,ME 192.168.1.1

# Spoof source IP (won't receive results)
sudo nmap -S 10.0.0.1 -e eth0 192.168.1.1

# Spoof source port
sudo nmap --source-port 53 192.168.1.1
sudo nmap -g 80 192.168.1.1

# Append random data to packets
nmap --data-length 50 192.168.1.1

# Randomize host scan order
nmap --randomize-hosts 192.168.1.0/24

# Use specific DNS servers
nmap --dns-servers 8.8.8.8,8.8.4.4 192.168.1.1

# Disable DNS resolution
nmap -n 192.168.1.0/24

# MAC address spoofing
sudo nmap --spoof-mac 00:11:22:33:44:55 192.168.1.1
sudo nmap --spoof-mac Dell 192.168.1.1
sudo nmap --spoof-mac 0 192.168.1.1    # Random MAC
```

## Practical Examples

### Quick Network Survey

```bash
# Find live hosts on a subnet
nmap -sn 192.168.1.0/24

# Quick port scan of live hosts
nmap -T4 -F 192.168.1.0/24

# Comprehensive scan of a single host
sudo nmap -sS -sV -O -A -T4 192.168.1.1
```

### Web Server Scan

```bash
# Find web servers
nmap -p 80,443,8080,8443 --open 192.168.1.0/24

# Detailed web server info
nmap -sV -p 80,443 --script=http-title,http-headers,ssl-cert 192.168.1.1
```

### Security Audit

```bash
# Vulnerability scan
sudo nmap -sV --script=vuln 192.168.1.1

# Check for open relays
nmap --script=smtp-open-relay -p 25 192.168.1.1

# Check SSL configuration
nmap --script=ssl-enum-ciphers -p 443 192.168.1.1

# Find all open ports
sudo nmap -sS -p- -T4 --open 192.168.1.1
```

### Internal Network Discovery

```bash
# Find all hosts and their services
sudo nmap -sn 10.0.0.0/24 -oG - | grep "Up" | awk '{print $2}' > live-hosts.txt
sudo nmap -sV -iL live-hosts.txt -oA network-services

# Find hosts with specific port open
nmap -p 3389 --open 192.168.1.0/24    # RDP
nmap -p 22 --open 192.168.1.0/24      # SSH
nmap -p 445 --open 192.168.1.0/24     # SMB
```

### Firewall Detection

```bash
# Determine if a firewall is filtering
sudo nmap -sA -p 22,80,443 192.168.1.1

# Compare SYN vs ACK results
sudo nmap -sS -p 22,80,443 192.168.1.1
sudo nmap -sA -p 22,80,443 192.168.1.1
# Filtered on SYN but unfiltered on ACK = stateful firewall
```

## Port States

| State | Meaning |
|-------|---------|
| open | Port is accepting connections |
| closed | Port is accessible but no service listening |
| filtered | Firewall/filter is blocking probe (can't determine state) |
| unfiltered | Port is accessible but can't determine if open/closed (ACK scan) |
| open\|filtered | Can't determine if open or filtered (UDP/FIN/Xmas/NULL) |
| closed\|filtered | Can't determine if closed or filtered (IP ID idle scan) |

## IPv6 Scanning

```bash
# Enable IPv6 scanning
nmap -6 fe80::1
nmap -6 2001:db8::1

# IPv6 ping scan
nmap -6 -sn fe80::/64

# IPv6 with service detection
nmap -6 -sV -p 80,443 2001:db8::1
```

## Useful One-Liners

```bash
# Find all live hosts (quick)
nmap -sn 192.168.1.0/24 | grep "report" | awk '{print $5}'

# Find open HTTP servers
nmap -p 80 --open -oG - 192.168.1.0/24 | grep "/open" | awk '{print $2}'

# Scan and show only open ports
nmap --open 192.168.1.1

# Quick OS fingerprint of live hosts
sudo nmap -O --osscan-guess -T4 192.168.1.0/24

# Export open ports as CSV
nmap -p- --open -oG - 192.168.1.1 | awk '/open/{print $2","$0}' | sed 's/.*Ports: //'

# Scan top 20 ports on entire /16 (fast)
nmap --top-ports 20 -T4 --open 10.0.0.0/16

# Find hosts with default SSH
nmap -p 22 --open --script=ssh-auth-methods 192.168.1.0/24
```

## Miscellaneous

```bash
# Show only open ports
nmap --open 192.168.1.1

# Show reason for port state
nmap --reason 192.168.1.1

# Show packet trace (debugging)
nmap --packet-trace 192.168.1.1

# Show interface and route info
nmap --iflist

# Resume an interrupted scan (requires grepable output)
nmap --resume scan.gnmap

# DNS resolution control
nmap -n 192.168.1.1     # Never resolve DNS (faster)
nmap -R 192.168.1.1     # Always resolve DNS

# Traceroute
nmap --traceroute 192.168.1.1

# Scan ports by name
nmap -p http,https,ssh 192.168.1.1
```

## Tips

- Always get permission before scanning networks you don't own
- Use `-T4` for faster scans on modern networks
- Combine `-sV` with scripts for better service detection
- Use `-oA` to save all output formats at once
- Add `-v` for real-time feedback during scans
- Use `-Pn` if the host doesn't respond to pings but you know it's up
- Root/sudo is required for SYN scans (`-sS`) and OS detection (`-O`)
- Use `--open` to reduce noise — only shows ports that are open
- Start with `-F` (fast, top 100) before doing `-p-` (all 65535)
- Use `--reason` to understand why nmap reports a port state

## Quick Reference

| Action | Command |
|--------|---------|
| Basic scan | `nmap 192.168.1.1` |
| Ping sweep | `nmap -sn 192.168.1.0/24` |
| All ports | `nmap -p- 192.168.1.1` |
| Service detection | `nmap -sV 192.168.1.1` |
| OS detection | `sudo nmap -O 192.168.1.1` |
| Aggressive scan | `nmap -A 192.168.1.1` |
| Default scripts | `nmap -sC 192.168.1.1` |
| Fast scan | `nmap -T4 -F 192.168.1.1` |
| UDP scan | `sudo nmap -sU 192.168.1.1` |
| Stealth scan | `sudo nmap -sS 192.168.1.1` |
| Skip host discovery | `nmap -Pn 192.168.1.1` |
| Open ports only | `nmap --open 192.168.1.1` |
| Save all formats | `nmap -oA results 192.168.1.1` |
| Vuln scan | `nmap --script=vuln 192.168.1.1` |
| SSL ciphers | `nmap --script=ssl-enum-ciphers -p 443 host` |
