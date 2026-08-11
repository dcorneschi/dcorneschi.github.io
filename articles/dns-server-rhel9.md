# Setting Up a Local DNS Server on RHEL 9

This guide covers installing and configuring BIND (Berkeley Internet Name Domain) as a local DNS server on RHEL 9 for name resolution in a homelab or small network.

## Overview

| Component | Purpose |
|-----------|---------|
| `bind` | The DNS server daemon (`named`) |
| `bind-utils` | DNS client tools (dig, host, nslookup) |
| `bind-chroot` | Optional — runs named in a chroot jail for security |

## Installation

```bash
dnf install -y bind bind-utils

# Optional: chroot for security hardening
dnf install -y bind-chroot
```

## Configuration Files

| File | Purpose |
|------|---------|
| `/etc/named.conf` | Main configuration file |
| `/etc/sysconfig/named` | named daemon options (e.g., `-4` to disable IPv6) |
| `/var/named/` | Zone files directory |
| `/var/named/named.ca` | Root hints (root DNS servers) |
| `/var/named/data/named.run` | Default debug log |
| `/var/named/chroot/var/log/named.log` | Log file (when using bind-chroot) |
| `/var/log/named/` | Custom log directory (if configured) |
| `/etc/named.rfc1912.zones` | Default zone includes |

## Basic Configuration

### /etc/named.conf

```bash
vi /etc/named.conf
```

```conf
options {
    listen-on port 53 { 127.0.0.1; 192.168.50.10; };
    listen-on-v6 port 53 { ::1; };
    directory       "/var/named";
    dump-file       "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    secroots-file   "/var/named/data/named.secroots";
    recursing-file  "/var/named/data/named.recursing";

    // Allow queries from local network
    allow-query     { localhost; 192.168.50.0/24; };

    // Allow recursion for local clients
    allow-recursion { localhost; 192.168.50.0/24; };

    // Forward unresolved queries to upstream DNS
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    // Forward only (don't attempt to resolve if forwarders fail)
    // forward only;

    // Or forward first (try forwarders, then resolve directly)
    forward first;

    recursion yes;

    dnssec-validation yes;

    managed-keys-directory "/var/named/dynamic";
    geoip-directory "/usr/share/GeoIP";

    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";

    // Disable version exposure
    version "not available";
};

logging {
    channel default_debug {
        file "data/named.run";
        severity dynamic;
    };
};

zone "." IN {
    type hint;
    file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";

// Forward zone
zone "homelab.local" IN {
    type master;
    file "db.homelab.local";
    allow-update { none; };
};

// Reverse zone
zone "50.168.192.in-addr.arpa" IN {
    type master;
    file "db.192.168.50";
    allow-update { none; };
};
```

## Zone Files

### Forward Zone (hostname → IP)

```bash
vi /var/named/db.homelab.local
```

```dns
$TTL 86400
@   IN  SOA     ns1.homelab.local. admin.homelab.local. (
                2024010101  ; Serial (YYYYMMDDNN)
                3600        ; Refresh (1 hour)
                1800        ; Retry (30 minutes)
                604800      ; Expire (1 week)
                86400       ; Minimum TTL (1 day)
)

; Nameservers
@       IN  NS      ns1.homelab.local.

; A records (nameserver)
ns1     IN  A       192.168.50.10

; A records (hosts)
gateway     IN  A       192.168.50.1
satellite   IN  A       192.168.50.210
foreman     IN  A       192.168.50.210
proxmox     IN  A       192.168.50.20
docker01    IN  A       192.168.50.30
k8s-master  IN  A       192.168.50.40
k8s-node01  IN  A       192.168.50.41
k8s-node02  IN  A       192.168.50.42
nfs         IN  A       192.168.50.50

; CNAME records (aliases)
www         IN  CNAME   docker01.homelab.local.
git         IN  CNAME   docker01.homelab.local.

; MX record (mail)
@           IN  MX  10  mail.homelab.local.
mail        IN  A       192.168.50.60
```

### Reverse Zone (IP → hostname)

```bash
vi /var/named/db.192.168.50
```

```dns
$TTL 86400
@   IN  SOA     ns1.homelab.local. admin.homelab.local. (
                2024010101  ; Serial
                3600        ; Refresh
                1800        ; Retry
                604800      ; Expire
                86400       ; Minimum TTL
)

; Nameservers
@       IN  NS      ns1.homelab.local.

; PTR records (IP → hostname)
1       IN  PTR     gateway.homelab.local.
10      IN  PTR     ns1.homelab.local.
20      IN  PTR     proxmox.homelab.local.
30      IN  PTR     docker01.homelab.local.
40      IN  PTR     k8s-master.homelab.local.
41      IN  PTR     k8s-node01.homelab.local.
42      IN  PTR     k8s-node02.homelab.local.
50      IN  PTR     nfs.homelab.local.
60      IN  PTR     mail.homelab.local.
210     IN  PTR     satellite.homelab.local.
```

## Validate Configuration

```bash
# Check named.conf syntax
named-checkconf

# Check named.conf with chroot
named-checkconf -t /var/named/chroot

# Validate forward zone file
named-checkzone homelab.local /var/named/db.homelab.local

# Validate reverse zone file
named-checkzone 50.168.192.in-addr.arpa /var/named/db.192.168.50
```

Expected output:

```
zone homelab.local/IN: loaded serial 2024010101
OK
```

## Set Permissions

```bash
# Set ownership on zone files
chown root:named /var/named/db.homelab.local
chown root:named /var/named/db.192.168.50

# Set permissions
chmod 640 /var/named/db.homelab.local
chmod 640 /var/named/db.192.168.50
```

## Firewall

```bash
firewall-cmd --permanent --add-service=dns
firewall-cmd --reload

# Or by port
firewall-cmd --permanent --add-port=53/tcp
firewall-cmd --permanent --add-port=53/udp
firewall-cmd --reload
```

## Start and Enable named

```bash
# RHEL 9 / 8 / 7
systemctl enable --now named

# RHEL 6 (legacy)
chkconfig named on && service named start

# Check status
systemctl status named

# View logs
journalctl -u named -f
```

## Disable IPv6 for named

If your network doesn't use IPv6, disable it for named to avoid unnecessary lookups:

```bash
echo 'OPTIONS="-4"' >> /etc/sysconfig/named
systemctl restart named
```

Also set in `/etc/named.conf`:

```conf
listen-on-v6 port 53 { none; };
```

## Testing

### From the DNS Server Itself

```bash
# Test forward lookup
dig @localhost homelab.local
dig @localhost docker01.homelab.local
dig @localhost docker01.homelab.local +short

# Test reverse lookup
dig @localhost -x 192.168.50.30

# Test forwarding (external domains)
dig @localhost google.com +short

# Test MX record
dig @localhost homelab.local MX

# Test CNAME
dig @localhost www.homelab.local
```

### From a Client Machine

```bash
# Point the client to the new DNS server
dig @192.168.50.10 docker01.homelab.local +short
host docker01.homelab.local 192.168.50.10
nslookup docker01.homelab.local 192.168.50.10
```

## Configure Clients to Use This DNS Server

### Temporarily (Testing)

```bash
# Edit resolv.conf directly
cat > /etc/resolv.conf << EOF
search homelab.local
nameserver 192.168.50.10
nameserver 8.8.8.8
EOF
```

### Permanently via NetworkManager

```bash
nmcli connection modify "eth0" ipv4.dns "192.168.50.10"
nmcli connection modify "eth0" ipv4.dns-search "homelab.local"
nmcli connection modify "eth0" ipv4.ignore-auto-dns yes
nmcli connection up "eth0"

# Verify
cat /etc/resolv.conf
```

### Via DHCP

Configure your DHCP server to distribute `192.168.50.10` as the DNS server.

## Adding New Records

When adding or changing records:

1. Edit the appropriate zone file
2. **Increment the serial number** (most important step!)
3. Validate the zone file
4. Reload named

```bash
# Edit zone file
vi /var/named/db.homelab.local

# Increment serial: 2024010101 → 2024010102

# Validate
named-checkzone homelab.local /var/named/db.homelab.local

# Reload (no restart needed)
rndc reload
# Or reload specific zone
rndc reload homelab.local
```

## Caching-Only DNS Server

If you only want forwarding/caching (no local zones), the configuration is simpler:

```conf
options {
    listen-on port 53 { 127.0.0.1; 192.168.50.10; };
    directory       "/var/named";
    allow-query     { localhost; 192.168.50.0/24; };
    allow-recursion { localhost; 192.168.50.0/24; };
    recursion yes;

    forwarders {
        8.8.8.8;
        1.1.1.1;
    };
    forward first;

    dnssec-validation yes;
};

zone "." IN {
    type hint;
    file "named.ca";
};
```

## Split-Horizon DNS (Internal vs External Views)

Serve different records based on the client's source IP:

```conf
// Internal view (LAN clients)
view "internal" {
    match-clients { 192.168.50.0/24; localhost; };
    recursion yes;

    zone "homelab.local" IN {
        type master;
        file "internal/db.homelab.local";
    };

    zone "." IN {
        type hint;
        file "named.ca";
    };
};

// External view (everyone else)
view "external" {
    match-clients { any; };
    recursion no;

    zone "homelab.local" IN {
        type master;
        file "external/db.homelab.local";
    };
};
```

## Secondary (Slave) DNS Server

On a second server, configure it to replicate zones from the primary:

**On the primary (master), allow transfers:**

```conf
zone "homelab.local" IN {
    type master;
    file "db.homelab.local";
    allow-transfer { 192.168.50.11; };  // secondary server IP
    also-notify { 192.168.50.11; };
};
```

**On the secondary (slave):**

```conf
zone "homelab.local" IN {
    type slave;
    file "slaves/db.homelab.local";
    masters { 192.168.50.10; };  // primary server IP
};
```

## Logging

Configure detailed logging for troubleshooting:

```conf
logging {
    channel query_log {
        file "/var/log/named/queries.log" versions 3 size 5m;
        severity info;
        print-time yes;
        print-category yes;
    };
    channel default_log {
        file "/var/log/named/default.log" versions 3 size 5m;
        severity dynamic;
        print-time yes;
    };

    category queries { query_log; };
    category default { default_log; };
};
```

```bash
# Create log directory
mkdir -p /var/log/named
chown named:named /var/log/named

# Toggle query logging at runtime (no restart)
rndc querylog

# View query log
tail -f /var/log/named/queries.log
```

## SELinux

If SELinux is enforcing:

```bash
# Check for denials
ausearch -m AVC -ts recent | grep named

# Allow named to write to custom log directory
semanage fcontext -a -t named_log_t "/var/log/named(/.*)?"
restorecon -Rv /var/log/named

# Allow named to bind to non-standard port (if needed)
semanage port -a -t dns_port_t -p tcp 5353
semanage port -a -t dns_port_t -p udp 5353
```

## Troubleshooting

```bash
# Check named is running
systemctl status named

# Check what named is listening on
ss -tulnp | grep named

# Check for configuration errors
named-checkconf
journalctl -u named --no-pager | tail -20

# Test resolution
dig @127.0.0.1 homelab.local SOA

# Check if recursion works
dig @127.0.0.1 google.com +short

# Check named version
named -V

# Dump the cache
rndc dumpdb -cache
cat /var/named/data/cache_dump.db

# Flush the cache
rndc flush

# Show server status
rndc status

# Enable query logging for debugging
rndc querylog
tail -f /var/log/named/queries.log
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `connection refused` | named not running or not listening on interface | Check `listen-on` in named.conf, restart named |
| `REFUSED` | Client IP not in `allow-query` | Add client network to `allow-query` |
| `SERVFAIL` | Zone file syntax error or DNSSEC issue | Run `named-checkzone`, check logs |
| `NXDOMAIN` for local hosts | Zone not loaded or record missing | Check zone file, increment serial, `rndc reload` |
| Clients can't resolve external names | Forwarding not configured or blocked | Check `forwarders`, test with `dig @127.0.0.1 google.com` |
| Serial not incrementing | Forgot to update serial | Edit zone file, increment serial, reload |
| Permission denied in logs | SELinux or file permissions | Check `chown named:named`, SELinux context |

## Quick Reference

```bash
# Install
dnf install -y bind bind-utils

# Main config
/etc/named.conf

# Zone files
/var/named/db.<domain>

# Validate config
named-checkconf
named-checkzone <zone> <file>

# Start/enable
systemctl enable --now named

# Firewall
firewall-cmd --permanent --add-service=dns && firewall-cmd --reload

# Reload after changes (no restart needed)
rndc reload

# Test
dig @localhost <hostname>.<domain> +short
```
