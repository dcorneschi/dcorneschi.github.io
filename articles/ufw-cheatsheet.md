# UFW Cheatsheet (Ubuntu Firewall)

UFW (Uncomplicated Firewall) is the default firewall management tool for Ubuntu. It provides a simplified interface to iptables/nftables.

## Installation and Status

```bash
# Install (usually pre-installed on Ubuntu)
sudo apt install -y ufw

# Check status
sudo ufw status
sudo ufw status verbose
sudo ufw status numbered

# Enable firewall
sudo ufw enable

# Disable firewall
sudo ufw disable

# Reset all rules to defaults
sudo ufw reset

# Reload rules without disabling
sudo ufw reload
```

## Default Policies

```bash
# Set default policies (recommended starting point)
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Block all outgoing (strict mode)
sudo ufw default deny outgoing

# Allow all incoming (not recommended)
sudo ufw default allow incoming

# Check current defaults
sudo ufw status verbose
# Look for "Default:" line
```

## Allow Rules

### By Port

```bash
# Allow a single port (TCP and UDP)
sudo ufw allow 80

# Allow TCP only
sudo ufw allow 80/tcp

# Allow UDP only
sudo ufw allow 53/udp

# Allow port range
sudo ufw allow 6000:6007/tcp
sudo ufw allow 6000:6007/udp
```

### By Service Name

```bash
# Allow by service name (from /etc/services)
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw allow ftp

# List available application profiles
sudo ufw app list

# Allow application profile
sudo ufw allow 'Nginx Full'
sudo ufw allow 'Nginx HTTP'
sudo ufw allow 'Nginx HTTPS'
sudo ufw allow 'Apache Full'
sudo ufw allow 'OpenSSH'

# Get info about an app profile
sudo ufw app info 'Nginx Full'
```

### By Source IP

```bash
# Allow all traffic from specific IP
sudo ufw allow from 192.168.1.100

# Allow from IP to specific port
sudo ufw allow from 192.168.1.100 to any port 22

# Allow from IP to specific port and protocol
sudo ufw allow from 192.168.1.100 to any port 22 proto tcp

# Allow from subnet
sudo ufw allow from 192.168.1.0/24

# Allow from subnet to specific port
sudo ufw allow from 192.168.1.0/24 to any port 3306

# Allow from IP to specific interface
sudo ufw allow in on eth0 from 192.168.1.0/24
```

### By Interface

```bash
# Allow on specific interface
sudo ufw allow in on eth0 to any port 80
sudo ufw allow in on eth1 to any port 3306

# Allow outgoing on interface
sudo ufw allow out on eth0 to any port 25
```

## Deny Rules

```bash
# Deny a port
sudo ufw deny 80
sudo ufw deny 80/tcp

# Deny from specific IP
sudo ufw deny from 203.0.113.50

# Deny from IP to specific port
sudo ufw deny from 203.0.113.50 to any port 22

# Deny from subnet
sudo ufw deny from 203.0.113.0/24

# Deny outgoing to specific port
sudo ufw deny out 25/tcp
```

## Reject Rules

Reject sends an ICMP unreachable message back (unlike deny which silently drops).

```bash
# Reject a port
sudo ufw reject 80/tcp

# Reject from specific IP
sudo ufw reject from 203.0.113.50
```

## Delete Rules

```bash
# Delete by rule number
sudo ufw status numbered
sudo ufw delete 3

# Delete by rule specification
sudo ufw delete allow 80/tcp
sudo ufw delete allow ssh
sudo ufw delete deny from 203.0.113.50
sudo ufw delete allow from 192.168.1.0/24 to any port 22
```

## Insert Rules (Priority/Order)

Rules are evaluated top to bottom. Insert to control order.

```bash
# Insert rule at position 1 (top priority)
sudo ufw insert 1 deny from 203.0.113.50

# Insert allow before a deny
sudo ufw insert 1 allow from 192.168.1.100 to any port 22
```

## Limit Rules (Rate Limiting)

Limit connections to prevent brute force. Denies connections if an IP has attempted 6+ connections in 30 seconds.

```bash
# Rate limit SSH (recommended)
sudo ufw limit ssh
sudo ufw limit 22/tcp

# Rate limit custom port
sudo ufw limit 2222/tcp
```

## Logging

```bash
# Enable logging
sudo ufw logging on

# Set log level (off, low, medium, high, full)
sudo ufw logging low
sudo ufw logging medium
sudo ufw logging high
sudo ufw logging full

# Disable logging
sudo ufw logging off

# View firewall logs
sudo tail -f /var/log/ufw.log
sudo grep UFW /var/log/syslog
sudo journalctl -u ufw
```

## IPv6 Support

```bash
# Enable IPv6 support (edit config)
sudo vi /etc/default/ufw
# Set: IPV6=yes

# Reload after changing
sudo ufw disable && sudo ufw enable

# Rules work the same way
sudo ufw allow from 2001:db8::/32 to any port 22
```

## Common Configurations

### Web Server

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

### Database Server (Internal Access Only)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow from 192.168.1.0/24 to any port 3306
sudo ufw allow from 192.168.1.0/24 to any port 5432
sudo ufw enable
```

### Mail Server

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 25/tcp     # SMTP
sudo ufw allow 587/tcp    # SMTP submission
sudo ufw allow 993/tcp    # IMAPS
sudo ufw allow 995/tcp    # POP3S
sudo ufw enable
```

### Docker Host

```bash
# Note: Docker manipulates iptables directly, bypassing UFW
# To force Docker through UFW, edit /etc/docker/daemon.json:
# { "iptables": false }

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 2376/tcp   # Docker API (TLS)
sudo ufw allow 2377/tcp   # Swarm management
sudo ufw allow 7946/tcp   # Swarm node communication
sudo ufw allow 7946/udp
sudo ufw allow 4789/udp   # Overlay network (VXLAN)
sudo ufw enable
```

### Kubernetes Node

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 6443/tcp         # API server
sudo ufw allow 2379:2380/tcp    # etcd
sudo ufw allow 10250/tcp        # Kubelet
sudo ufw allow 10259/tcp        # kube-scheduler
sudo ufw allow 10257/tcp        # kube-controller-manager
sudo ufw allow 30000:32767/tcp  # NodePort services
sudo ufw enable
```

### Allow Specific Countries/IPs Only (SSH)

```bash
# Only allow SSH from your office and home
sudo ufw default deny incoming
sudo ufw allow from 203.0.113.10 to any port 22   # Office
sudo ufw allow from 198.51.100.20 to any port 22  # Home
sudo ufw deny 22/tcp                               # Block all other SSH
sudo ufw enable
```

## NAT and Routing (Advanced)

Edit `/etc/ufw/before.rules` to add NAT rules:

```bash
# Add before the *filter section in /etc/ufw/before.rules
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
COMMIT
```

Enable IP forwarding:

```bash
# Edit /etc/ufw/sysctl.conf
net/ipv4/ip_forward=1

# Or /etc/sysctl.conf
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Reload UFW
sudo ufw disable && sudo ufw enable
```

## Custom Application Profiles

Create profiles in `/etc/ufw/applications.d/`:

```bash
# Create custom profile
sudo vi /etc/ufw/applications.d/myapp

# Example content:
# [MyApp]
# title=My Custom Application
# description=My app on port 8080
# ports=8080/tcp

# [MyApp-Full]
# title=My Custom Application (Full)
# description=My app on ports 8080 and 8443
# ports=8080,8443/tcp
```

```bash
# Reload app profiles
sudo ufw app update MyApp

# Use the profile
sudo ufw allow 'MyApp'

# List profiles
sudo ufw app list
```

## Troubleshooting

### UFW Not Blocking Traffic

```bash
# Check if UFW is active
sudo ufw status

# Check rule order (rules evaluated top to bottom)
sudo ufw status numbered

# Check if Docker is bypassing UFW
sudo iptables -L -n | grep DOCKER

# Check for conflicting iptables rules
sudo iptables -L -n -v
```

### Locked Out (Can't SSH)

```bash
# If you have console/physical access:
sudo ufw disable

# Or allow SSH then re-enable
sudo ufw allow ssh
sudo ufw enable

# If using cloud provider, use web console or recovery mode
```

### Check What's Actually Listening

```bash
# See what ports are open
sudo ss -tlnp
sudo netstat -tlnp

# Verify UFW rules match
sudo ufw status verbose
```

### Debug Rule Matching

```bash
# Enable high logging temporarily
sudo ufw logging high

# Watch logs
sudo tail -f /var/log/ufw.log

# Test with specific traffic
# From another machine:
nc -zv target-ip 80

# Reset logging when done
sudo ufw logging low
```

## Quick Reference

| Action | Command |
|--------|---------|
| Enable | `sudo ufw enable` |
| Disable | `sudo ufw disable` |
| Status | `sudo ufw status verbose` |
| Status (numbered) | `sudo ufw status numbered` |
| Allow port | `sudo ufw allow 80/tcp` |
| Allow service | `sudo ufw allow ssh` |
| Allow from IP | `sudo ufw allow from 1.2.3.4` |
| Deny port | `sudo ufw deny 80/tcp` |
| Deny from IP | `sudo ufw deny from 1.2.3.4` |
| Rate limit | `sudo ufw limit ssh` |
| Delete rule | `sudo ufw delete allow 80/tcp` |
| Delete by number | `sudo ufw delete 3` |
| Insert rule | `sudo ufw insert 1 deny from 1.2.3.4` |
| Reset all | `sudo ufw reset` |
| App list | `sudo ufw app list` |
| Logging | `sudo ufw logging medium` |

## Rule Comments

```bash
# Add a comment to a rule for documentation
sudo ufw allow 22 comment 'SSH access'
sudo ufw allow 80 comment 'Web traffic'
sudo ufw allow from 192.168.1.0/24 to any port 3306 comment 'MySQL from LAN'

# Comments appear in status output
sudo ufw status verbose
```

## Multi-Port Rules

```bash
# Allow multiple ports in one rule
sudo ufw allow 80,443/tcp           # HTTP and HTTPS
sudo ufw allow 25,587,993/tcp       # Mail ports
sudo ufw deny 135,139,445/tcp       # Block Windows SMB
```

## Tips

- Always allow SSH before enabling: `sudo ufw allow 22` to avoid lockout
- Use numbered status for easy deletion: `sudo ufw status numbered`
- Add comments to rules for documentation: `sudo ufw allow 80 comment 'Web traffic'`
- Specify `/tcp` or `/udp` for clarity — omitting allows both protocols
- Port ranges use colon syntax: `6000:6007/tcp`
- Rules are evaluated top to bottom — use `insert` to control priority
- No built-in dry-run — use `status` commands to verify before enabling
- Reset without prompt: `sudo ufw --force reset`

## Platform Notes

| Platform | Firewall Tool | Notes |
|----------|--------------|-------|
| Ubuntu/Debian | UFW | Pre-installed, `sudo apt install ufw` if missing |
| RHEL/CentOS/Rocky | firewalld | UFW not default, use `firewall-cmd` instead |
| macOS | pf (Packet Filter) | UFW not available |
| Arch Linux | UFW | Available via `pacman -S ufw` |
