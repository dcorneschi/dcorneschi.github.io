# FirewallD Cheatsheet

FirewallD is the default firewall management tool on RHEL, CentOS, Fedora, and Rocky Linux. It uses zones to define trust levels for network connections and supports runtime and permanent configurations.

FirewallD is a wrapper for iptables/nftables — not a replacement. While iptables commands still work, use `firewall-cmd` exclusively when FirewallD is active.

## Management Methods

- **CLI**: `firewall-cmd` (most common)
- **GUI**: `firewall-config` (requires X11)
- **Config files**: `/etc/firewalld/` (custom overrides `/usr/lib/firewalld/`)

## Configuration File Locations

| Path | Purpose |
|------|---------|
| `/etc/firewalld/firewalld.conf` | Main configuration file |
| `/etc/firewalld/zones/` | Custom zone definitions (overrides defaults) |
| `/etc/firewalld/services/` | Custom service definitions |
| `/usr/lib/firewalld/zones/` | System default zone files |
| `/usr/lib/firewalld/services/` | System default service definitions |
| `/var/log/firewalld` | FirewallD log file |

> **Priority:** If a file with the same name exists in both `/etc/firewalld/` and `/usr/lib/firewalld/`, the `/etc/firewalld/` version is used.

## Installation and Status

```bash
# Install (usually pre-installed on RHEL-based systems)
sudo dnf install -y firewalld

# Enable and start
sudo systemctl enable --now firewalld

# Check status
sudo systemctl status firewalld
sudo firewall-cmd --state

# Version
firewall-cmd --version
```

## Runtime vs Permanent

FirewallD maintains two configurations:

| Type | Applies | Survives Reboot | Flag |
|------|---------|:---------------:|------|
| Runtime | Immediately | No | (default, no flag) |
| Permanent | After reload | Yes | `--permanent` |

```bash
# Add rule to runtime only (testing)
sudo firewall-cmd --add-port=8080/tcp

# Add rule permanently
sudo firewall-cmd --permanent --add-port=8080/tcp

# Apply permanent rules to runtime (reload)
sudo firewall-cmd --reload

# Common pattern: add permanent + reload
sudo firewall-cmd --permanent --add-port=8080/tcp && sudo firewall-cmd --reload

# View runtime config
sudo firewall-cmd --list-all

# View permanent config
sudo firewall-cmd --permanent --list-all
```

## Zones

Zones define the trust level for network interfaces and connections.

### List and Check Zones

```bash
# List all available zones
sudo firewall-cmd --get-zones

# List active zones (zones with interfaces assigned)
sudo firewall-cmd --get-active-zones

# Get default zone
sudo firewall-cmd --get-default-zone

# Show zone details
sudo firewall-cmd --zone=public --list-all

# Show all zones with their config
sudo firewall-cmd --list-all-zones
```

### Common Zones

| Zone | Default Policy | Use Case |
|------|---------------|----------|
| `drop` | Drop all incoming, no ICMP reply | Maximum security, stealth mode |
| `block` | Reject all incoming with ICMP reply | Reject with notification |
| `public` | Only selected services allowed | Default for untrusted networks, cloud servers |
| `external` | NAT masquerading, selected services | Router WAN interface, requires LAN+WAN |
| `dmz` | Only selected services | Demilitarized zone, limited LAN access |
| `work` | Trust most LAN traffic | Corporate/workplace networks |
| `home` | Trust most LAN traffic | Home networks, laptops, desktops |
| `internal` | Trust most LAN traffic | Internal network interfaces |
| `trusted` | Allow ALL traffic | Fully trusted networks (not recommended for servers) |

### Manage Zones

```bash
# Set default zone
sudo firewall-cmd --set-default-zone=public

# Assign interface to zone
sudo firewall-cmd --zone=internal --change-interface=eth1 --permanent

# Remove interface from zone
sudo firewall-cmd --zone=internal --remove-interface=eth1 --permanent

# Add source to a zone (IP-based zone assignment)
sudo firewall-cmd --zone=trusted --add-source=192.168.1.0/24 --permanent

# Remove source from zone
sudo firewall-cmd --zone=trusted --remove-source=192.168.1.0/24 --permanent

# Create a new zone
sudo firewall-cmd --permanent --new-zone=myzone
sudo firewall-cmd --reload

# Delete a zone
sudo firewall-cmd --permanent --delete-zone=myzone
sudo firewall-cmd --reload
```

## Services

Services are predefined sets of ports and protocols.

### List Services

```bash
# List all available services
sudo firewall-cmd --get-services

# List services enabled in current zone
sudo firewall-cmd --list-services

# List services in a specific zone
sudo firewall-cmd --zone=public --list-services

# Get info about a service
sudo firewall-cmd --info-service=http
sudo firewall-cmd --info-service=ssh
```

### Add/Remove Services

```bash
# Allow a service
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=ssh

# Allow multiple services
sudo firewall-cmd --permanent --add-service={http,https,dns}

# Remove a service
sudo firewall-cmd --permanent --remove-service=http

# Allow service in specific zone
sudo firewall-cmd --permanent --zone=internal --add-service=mysql

# Apply changes
sudo firewall-cmd --reload
```

### Create Custom Service

```bash
# Create a new service definition
sudo firewall-cmd --permanent --new-service=myapp
sudo firewall-cmd --permanent --service=myapp --set-description="My Application"
sudo firewall-cmd --permanent --service=myapp --set-short="MyApp"
sudo firewall-cmd --permanent --service=myapp --add-port=8080/tcp
sudo firewall-cmd --permanent --service=myapp --add-port=8443/tcp
sudo firewall-cmd --reload

# Then use it like any other service
sudo firewall-cmd --permanent --add-service=myapp
sudo firewall-cmd --reload

# Service files are stored in
ls /usr/lib/firewalld/services/       # system defaults
ls /etc/firewalld/services/           # custom/overrides
```

## Ports

### Add/Remove Ports

```bash
# Allow a single port
sudo firewall-cmd --permanent --add-port=8080/tcp

# Allow UDP port
sudo firewall-cmd --permanent --add-port=53/udp

# Allow port range
sudo firewall-cmd --permanent --add-port=5000-5100/tcp

# Allow multiple ports
sudo firewall-cmd --permanent --add-port={8080/tcp,8443/tcp,9090/tcp}

# Remove port
sudo firewall-cmd --permanent --remove-port=8080/tcp

# List open ports
sudo firewall-cmd --list-ports

# Check if a port is open
sudo firewall-cmd --query-port=8080/tcp

# Apply
sudo firewall-cmd --reload
```

## Rich Rules

Rich rules provide fine-grained control with source/destination filtering, logging, and rate limiting.

### Syntax

```
rule [family="ipv4|ipv6"]
     [source address="<address>[/<mask>]" [invert="True"]]
     [destination address="<address>[/<mask>]" [invert="True"]]
     [service name="<service>"] | [port port="<port>" protocol="<protocol>"]
     [log [prefix="<prefix>"] [level="<level>"] [limit value="<rate/duration>"]]
     [accept|reject|drop]
```

### Examples

```bash
# Allow SSH from specific IP
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.100" service name="ssh" accept'

# Allow port from a subnet
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" port port="3306" protocol="tcp" accept'

# Deny all traffic from an IP
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="203.0.113.50" drop'

# Reject with ICMP message
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="203.0.113.0/24" reject'

# Log and accept
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/8" service name="http" log prefix="HTTP_ACCESS" level="info" accept'

# Rate limit SSH (max 3 connections per minute)
sudo firewall-cmd --permanent --add-rich-rule='rule service name="ssh" accept limit value="3/m"'

# Log dropped packets
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="0.0.0.0/0" service name="ssh" log prefix="SSH_DROP" level="warning" limit value="5/m" drop'

# Allow forwarding from source to destination
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" forward-port port="80" protocol="tcp" to-port="8080"'

# List rich rules
sudo firewall-cmd --list-rich-rules

# Remove a rich rule (paste the exact rule)
sudo firewall-cmd --permanent --remove-rich-rule='rule family="ipv4" source address="192.168.1.100" service name="ssh" accept'

# Apply
sudo firewall-cmd --reload
```

## Source IP Whitelisting

```bash
# Whitelist a single IP (add to default zone)
sudo firewall-cmd --permanent --add-source=192.168.1.100

# Whitelist a subnet
sudo firewall-cmd --permanent --add-source=192.168.1.0/24

# Whitelist to a specific zone
sudo firewall-cmd --permanent --zone=trusted --add-source=10.0.0.0/8

# Remove a whitelisted IP
sudo firewall-cmd --permanent --remove-source=192.168.1.100

# List sources in a zone
sudo firewall-cmd --zone=trusted --list-sources

# Apply
sudo firewall-cmd --reload
```

## Port Forwarding

```bash
# Forward port 80 to port 8080 on the same host
sudo firewall-cmd --permanent --add-forward-port=port=80:proto=tcp:toport=8080

# Forward port 80 to another host (DNAT)
sudo firewall-cmd --permanent --add-forward-port=port=80:proto=tcp:toport=8080:toaddr=192.168.1.100

# Enable masquerading (required for forwarding to other hosts)
sudo firewall-cmd --permanent --add-masquerade

# Remove port forward
sudo firewall-cmd --permanent --remove-forward-port=port=80:proto=tcp:toport=8080

# List forwards
sudo firewall-cmd --list-forward-ports

# Query if a specific forward exists
sudo firewall-cmd --query-forward-port=port=22:proto=tcp:toport=2222:toaddr=10.0.0.10

# Forward with source restriction (via rich rule)
sudo firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=192.168.1.0/24 forward-port port=22 protocol=tcp to-port=2222 to-addr=10.0.0.10'

# Apply
sudo firewall-cmd --reload
```

## Masquerading (NAT)

```bash
# Enable masquerading (outbound NAT)
sudo firewall-cmd --permanent --add-masquerade

# Check if masquerading is enabled
sudo firewall-cmd --query-masquerade

# Disable masquerading
sudo firewall-cmd --permanent --remove-masquerade

# Enable masquerading on specific zone
sudo firewall-cmd --permanent --zone=external --add-masquerade

# Apply
sudo firewall-cmd --reload
```

## ICMP

```bash
# List ICMP types
sudo firewall-cmd --get-icmptypes

# Block specific ICMP type
sudo firewall-cmd --permanent --add-icmp-block=echo-request

# Remove ICMP block
sudo firewall-cmd --permanent --remove-icmp-block=echo-request

# Block all ICMP except those explicitly allowed
sudo firewall-cmd --permanent --add-icmp-block-inversion

# List blocked ICMP types
sudo firewall-cmd --list-icmp-blocks
```

## Direct Rules (iptables Passthrough)

For complex rules that can't be expressed with zones/services/rich rules:

```bash
# Add a direct iptables rule
sudo firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 0 -s 192.168.1.0/24 -p tcp --dport 22 -j ACCEPT

# List direct rules
sudo firewall-cmd --direct --get-all-rules

# Remove direct rule
sudo firewall-cmd --permanent --direct --remove-rule ipv4 filter INPUT 0 -s 192.168.1.0/24 -p tcp --dport 22 -j ACCEPT

# Add chain
sudo firewall-cmd --permanent --direct --add-chain ipv4 filter MY_CHAIN

# Apply
sudo firewall-cmd --reload
```

## IPSets

Group IPs/networks for use in rules:

```bash
# Create an IP set
sudo firewall-cmd --permanent --new-ipset=blocklist --type=hash:ip

# Add IPs to the set
sudo firewall-cmd --permanent --ipset=blocklist --add-entry=203.0.113.1
sudo firewall-cmd --permanent --ipset=blocklist --add-entry=203.0.113.2

# Create set for networks
sudo firewall-cmd --permanent --new-ipset=trusted-nets --type=hash:net
sudo firewall-cmd --permanent --ipset=trusted-nets --add-entry=10.0.0.0/8

# Use in a rich rule
sudo firewall-cmd --permanent --add-rich-rule='rule source ipset="blocklist" drop'
sudo firewall-cmd --permanent --add-rich-rule='rule source ipset="trusted-nets" service name="ssh" accept'

# List IP sets
sudo firewall-cmd --get-ipsets
sudo firewall-cmd --ipset=blocklist --get-entries

# Remove entry
sudo firewall-cmd --permanent --ipset=blocklist --remove-entry=203.0.113.1

# Delete IP set
sudo firewall-cmd --permanent --delete-ipset=blocklist

# Apply
sudo firewall-cmd --reload
```

## Common Configurations

### Web Server

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

### Database Server (Internal Only)

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" port port="3306" protocol="tcp" accept'
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" port port="5432" protocol="tcp" accept'
sudo firewall-cmd --reload
```

### Kubernetes Node

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp         # API server
sudo firewall-cmd --permanent --add-port=2379-2380/tcp    # etcd
sudo firewall-cmd --permanent --add-port=10250/tcp        # Kubelet
sudo firewall-cmd --permanent --add-port=10259/tcp        # kube-scheduler
sudo firewall-cmd --permanent --add-port=10257/tcp        # kube-controller-manager
sudo firewall-cmd --permanent --add-port=30000-32767/tcp  # NodePort services
sudo firewall-cmd --reload
```

### Docker Host

```bash
sudo firewall-cmd --permanent --zone=trusted --add-interface=docker0
sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --reload
```

### Mail Server

```bash
sudo firewall-cmd --permanent --add-service={smtp,smtps,imap,imaps,pop3s}
sudo firewall-cmd --reload
```

## Panic Mode

Block all network traffic immediately (emergency):

```bash
# Enable panic mode (drops ALL traffic)
sudo firewall-cmd --panic-on

# Disable panic mode
sudo firewall-cmd --panic-off

# Check if panic mode is active
sudo firewall-cmd --query-panic
```

## Logging

```bash
# Set log denied packets (off, all, unicast, broadcast, multicast)
sudo firewall-cmd --set-log-denied=all

# Check current log setting
sudo firewall-cmd --get-log-denied

# View firewall logs
sudo journalctl -u firewalld
sudo dmesg | grep REJECT
sudo dmesg | grep DROP

# Disable logging
sudo firewall-cmd --set-log-denied=off
```

## Lockdown Mode

Prevent applications from changing the firewall:

```bash
# Enable lockdown
sudo firewall-cmd --lockdown-on

# Disable lockdown
sudo firewall-cmd --lockdown-off

# Query lockdown state
sudo firewall-cmd --query-lockdown
```

## Backup and Restore

```bash
# Export current runtime config
sudo firewall-cmd --runtime-to-permanent

# Backup zone files
sudo cp -r /etc/firewalld/ /etc/firewalld.bak

# Export iptables rules (useful for documentation/migration)
sudo iptables -S > /tmp/firewalld_rules_ipv4
sudo ip6tables -S > /tmp/firewalld_rules_ipv6

# Configuration files
ls /etc/firewalld/                    # Custom config
ls /usr/lib/firewalld/                # System defaults
cat /etc/firewalld/firewalld.conf     # Main config

# Reset to defaults
sudo rm -rf /etc/firewalld/zones/*
sudo firewall-cmd --reload
```

## Troubleshooting

### Check What's Allowed

```bash
# Full view of current zone
sudo firewall-cmd --list-all

# All zones
sudo firewall-cmd --list-all-zones

# Check specific port
sudo firewall-cmd --query-port=8080/tcp

# Check specific service
sudo firewall-cmd --query-service=http

# Check which zone an interface is in
sudo firewall-cmd --get-zone-of-interface=eth0
```

### Traffic Not Getting Through

```bash
# Verify firewalld is running
sudo firewall-cmd --state

# Check active zones and interfaces
sudo firewall-cmd --get-active-zones

# Check the correct zone
sudo firewall-cmd --zone=public --list-all

# Check for conflicting rules
sudo firewall-cmd --list-rich-rules
sudo firewall-cmd --direct --get-all-rules

# Check if nftables/iptables has extra rules
sudo nft list ruleset
sudo iptables -L -n
```

### Service Conflicts

```bash
# FirewallD conflicts with iptables service
# Only one should be active
sudo systemctl status iptables
sudo systemctl stop iptables
sudo systemctl disable iptables
sudo systemctl mask iptables
```

## Quick Reference

| Action | Command |
|--------|---------|
| Check state | `firewall-cmd --state` |
| List all | `firewall-cmd --list-all` |
| List zones | `firewall-cmd --get-zones` |
| Active zones | `firewall-cmd --get-active-zones` |
| Default zone | `firewall-cmd --get-default-zone` |
| Set default zone | `firewall-cmd --set-default-zone=X` |
| Add service | `firewall-cmd --permanent --add-service=http` |
| Remove service | `firewall-cmd --permanent --remove-service=http` |
| Add port | `firewall-cmd --permanent --add-port=8080/tcp` |
| Remove port | `firewall-cmd --permanent --remove-port=8080/tcp` |
| Add rich rule | `firewall-cmd --permanent --add-rich-rule='...'` |
| Port forward | `firewall-cmd --permanent --add-forward-port=port=80:proto=tcp:toport=8080` |
| Masquerade | `firewall-cmd --permanent --add-masquerade` |
| Reload | `firewall-cmd --reload` |
| Panic on | `firewall-cmd --panic-on` |
| Log denied | `firewall-cmd --set-log-denied=all` |
