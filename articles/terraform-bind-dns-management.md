# Managing DNS Entries with Terraform and Local BIND Server

Use Terraform to manage DNS records on a local BIND (named) server — automating A, CNAME, PTR, and other record types via the DNS provider. Useful for homelabs, on-premises infrastructure, and hybrid environments where you control the authoritative DNS server.

## Overview

The `hashicorp/dns` provider uses RFC 2136 (Dynamic DNS Updates) to create and manage records on a BIND server. This means Terraform communicates directly with your DNS server over the DNS protocol — no SSH, no API, no file editing.

### Prerequisites

- A BIND server configured to accept dynamic updates (TSIG or IP-based)
- The `dns` Terraform provider
- Network access from the Terraform machine to the BIND server on port 53

## BIND Server Configuration

### Enable Dynamic Updates

Configure your zone to accept dynamic updates using TSIG (recommended) or allow-update by IP.

#### Generate a TSIG Key

```bash
# Generate a TSIG key for Terraform
tsig-keygen terraform-key > /etc/named/terraform-key.conf

# Output looks like:
# key "terraform-key" {
#     algorithm hmac-sha256;
#     secret "base64-encoded-secret";
# };
```

#### Include the Key in named.conf

```bash
# /etc/named.conf (or /etc/bind/named.conf on Debian)

include "/etc/named/terraform-key.conf";

zone "homelab.local" IN {
    type master;
    file "/var/named/homelab.local.zone";
    allow-update { key "terraform-key"; };
};

zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "/var/named/192.168.1.rev";
    allow-update { key "terraform-key"; };
};
```

#### Alternative: Allow Updates by IP (Less Secure)

```bash
zone "homelab.local" IN {
    type master;
    file "/var/named/homelab.local.zone";
    allow-update { 192.168.1.10; };  # Terraform machine IP
};
```

```bash
# Restart BIND after changes
sudo systemctl restart named       # RHEL/Rocky
sudo systemctl restart bind9       # Ubuntu/Debian

# Verify zone is loaded
sudo named-checkzone homelab.local /var/named/homelab.local.zone
```

## Terraform Provider Configuration

### Using TSIG Authentication (Recommended)

```hcl
terraform {
  required_providers {
    dns = {
      source  = "hashicorp/dns"
      version = "~> 3.4"
    }
  }
}

variable "dns_server" {
  type        = string
  default     = "192.168.1.5"
  description = "IP address of the BIND DNS server"
}

variable "tsig_key_name" {
  type        = string
  default     = "terraform-key."
  description = "TSIG key name (must end with a dot)"
}

variable "tsig_key_secret" {
  type        = string
  sensitive   = true
  description = "Base64-encoded TSIG key secret"
}

variable "tsig_key_algorithm" {
  type        = string
  default     = "hmac-sha256"
  description = "TSIG algorithm (hmac-md5, hmac-sha1, hmac-sha256, hmac-sha512)"
}

provider "dns" {
  update {
    server        = var.dns_server
    key_name      = var.tsig_key_name
    key_algorithm = var.tsig_key_algorithm
    key_secret    = var.tsig_key_secret
  }
}
```

### With All Provider Options

```hcl
provider "dns" {
  update {
    server        = var.dns_server
    port          = 53
    transport     = "udp"
    timeout       = "5s"
    retries       = 3
    key_name      = var.tsig_key_name
    key_algorithm = var.tsig_key_algorithm
    key_secret    = var.tsig_key_secret
  }
}
```

### Without Authentication (IP-Based allow-update)

```hcl
provider "dns" {
  update {
    server = "192.168.1.5"
  }
}
```

## Managing DNS Records

### A Record (Forward Lookup)

```hcl
variable "domain" {
  type    = string
  default = "homelab.local"
}

resource "dns_a_record_set" "web" {
  zone      = "${var.domain}."
  name      = "web"
  addresses = ["192.168.1.100"]
  ttl       = 300
}
# Result: web.homelab.local → 192.168.1.100
```

### Multiple A Records (Round-Robin)

```hcl
resource "dns_a_record_set" "app" {
  zone = "${var.domain}."
  name = "app"
  addresses = [
    "192.168.1.101",
    "192.168.1.102",
    "192.168.1.103",
  ]
  ttl = 300
}
# Result: app.homelab.local → 3 IPs (round-robin DNS)
```

### CNAME Record

```hcl
resource "dns_cname_record" "www" {
  zone  = "${var.domain}."
  name  = "www"
  cname = "web.${var.domain}."
  ttl   = 300
}
# Result: www.homelab.local → web.homelab.local
```

### AAAA Record (IPv6)

```hcl
resource "dns_aaaa_record_set" "ipv6_web" {
  zone      = "${var.domain}."
  name      = "ipv6"
  addresses = ["2001:db8::1"]
  ttl       = 300
}
# Result: ipv6.homelab.local → 2001:db8::1
```

### PTR Record (Reverse Lookup)

```hcl
resource "dns_ptr_record" "web_ptr" {
  zone = "1.168.192.in-addr.arpa."
  name = "100"
  ptr  = "web.${var.domain}."
  ttl  = 300
}
# Result: 192.168.1.100 → web.homelab.local
```

### TXT Record

```hcl
resource "dns_txt_record_set" "spf" {
  zone = "${var.domain}."
  name = ""
  txt  = ["v=spf1 mx a ~all"]
  ttl  = 3600
}

resource "dns_txt_record_set" "verification" {
  zone = "${var.domain}."
  name = "_verification"
  txt  = ["verify=abc123def456"]
  ttl  = 3600
}

resource "dns_txt_record_set" "dmarc" {
  zone = "${var.domain}."
  name = "_dmarc"
  txt  = ["v=DMARC1; p=quarantine; rua=mailto:dmarc@${var.domain}"]
  ttl  = 3600
}
```

### MX Record

```hcl
resource "dns_mx_record_set" "mail" {
  zone = "${var.domain}."
  name = ""

  mx {
    preference = 10
    exchange   = "mail.${var.domain}."
  }

  mx {
    preference = 20
    exchange   = "mail-backup.${var.domain}."
  }

  ttl = 3600
}
# Result: homelab.local MX → mail.homelab.local (priority 10)
```

### SRV Record

```hcl
resource "dns_srv_record_set" "ldap" {
  zone = "${var.domain}."
  name = "_ldap._tcp"

  srv {
    priority = 0
    weight   = 100
    port     = 389
    target   = "ldap.${var.domain}."
  }

  ttl = 3600
}
```

### NS Record

```hcl
resource "dns_ns_record_set" "subdomain" {
  zone = "${var.domain}."
  name = "k8s"
  nameservers = [
    "ns1.k8s.${var.domain}.",
    "ns2.k8s.${var.domain}.",
  ]
  ttl = 3600
}
# Result: delegates k8s.homelab.local to separate nameservers
```

## Real-World Patterns

### DNS for VM Fleet

```hcl
variable "vms" {
  type = map(object({
    ip   = string
    role = string
  }))
  default = {
    "node1" = { ip = "192.168.1.10", role = "master" }
    "node2" = { ip = "192.168.1.11", role = "worker" }
    "node3" = { ip = "192.168.1.12", role = "worker" }
  }
}

# Forward records
resource "dns_a_record_set" "vms" {
  for_each  = var.vms
  zone      = "${var.domain}."
  name      = each.key
  addresses = [each.value.ip]
  ttl       = 300
}

# Reverse records
resource "dns_ptr_record" "vms" {
  for_each = var.vms
  zone     = "1.168.192.in-addr.arpa."
  name     = split(".", each.value.ip)[3]
  ptr      = "${each.key}.${var.domain}."
  ttl      = 300
}
# Result: creates forward and reverse DNS for all VMs from a single map
```

### DNS for Kubernetes Services

```hcl
variable "k8s_services" {
  type = map(string)
  default = {
    "grafana"      = "192.168.1.200"
    "prometheus"   = "192.168.1.201"
    "argocd"       = "192.168.1.202"
    "traefik"      = "192.168.1.203"
    "longhorn"     = "192.168.1.204"
  }
}

resource "dns_a_record_set" "k8s" {
  for_each  = var.k8s_services
  zone      = "${var.domain}."
  name      = each.key
  addresses = [each.value]
  ttl       = 300
}

# Wildcard for ingress
resource "dns_a_record_set" "wildcard" {
  zone      = "${var.domain}."
  name      = "*"
  addresses = ["192.168.1.203"]  # Traefik/ingress IP
  ttl       = 300
}
# Result: *.homelab.local → Traefik, plus explicit records for each service
```

### DNS with Terraform and KVM/Proxmox VMs

```hcl
# Combine with libvirt or proxmox provider
resource "libvirt_domain" "vm" {
  for_each = var.vms
  name     = each.key
  memory   = 2048
  vcpu     = 2
  # ...
}

# Create DNS entries for each VM
resource "dns_a_record_set" "vm_dns" {
  for_each  = var.vms
  zone      = "${var.domain}."
  name      = each.key
  addresses = [each.value.ip]
  ttl       = 300

  depends_on = [libvirt_domain.vm]
}
```

### Split DNS (Internal vs External)

```hcl
# Internal DNS (BIND)
provider "dns" {
  alias = "internal"
  update {
    server        = "192.168.1.5"
    key_name      = var.tsig_key_name
    key_algorithm = var.tsig_key_algorithm
    key_secret    = var.tsig_key_secret
  }
}

# External DNS (Route 53)
provider "aws" {
  region = "eu-west-1"
}

# Internal record
resource "dns_a_record_set" "app_internal" {
  provider  = dns.internal
  zone      = "homelab.local."
  name      = "app"
  addresses = ["192.168.1.100"]
  ttl       = 300
}

# External record
resource "aws_route53_record" "app_external" {
  zone_id = var.route53_zone_id
  name    = "app.example.com"
  type    = "A"
  ttl     = 300
  records = ["203.0.113.50"]
}
```

## Using DNS Data Sources (Read-Only)

Query existing DNS records without managing them.

```hcl
# Look up an A record
data "dns_a_record_set" "existing" {
  host = "web.homelab.local"
}

output "web_ips" {
  value = data.dns_a_record_set.existing.addrs
}

# Look up a CNAME
data "dns_cname_record_set" "alias" {
  host = "www.homelab.local"
}

# Look up PTR
data "dns_ptr_record_set" "reverse" {
  ip_address = "192.168.1.100"
}

# Look up MX records
data "dns_mx_record_set" "mail" {
  domain = "homelab.local."
}
```

## Variables File (terraform.tfvars)

```hcl
# terraform.tfvars
dns_server         = "192.168.1.5"
tsig_key_name      = "terraform-key."
tsig_key_algorithm = "hmac-sha256"
tsig_key_secret    = "your-base64-encoded-secret-here"
domain             = "homelab.local"
```

Keep the TSIG secret out of version control:

```bash
# .gitignore
*.tfvars
!terraform.tfvars.example
```

```bash
# Or use environment variable
export TF_VAR_tsig_key_secret="your-base64-secret"
```

## Verification

### Test with dig

```bash
# Verify A record
dig @192.168.1.5 web.homelab.local A +short

# Verify CNAME
dig @192.168.1.5 www.homelab.local CNAME +short

# Verify PTR
dig @192.168.1.5 -x 192.168.1.100 +short

# Verify MX
dig @192.168.1.5 homelab.local MX +short

# Verify TXT
dig @192.168.1.5 homelab.local TXT +short

# Check all records for a name
dig @192.168.1.5 web.homelab.local ANY
```

### Test Dynamic Updates Manually

```bash
# Test TSIG update with nsupdate
nsupdate -k /etc/named/terraform-key.conf <<EOF
server 192.168.1.5
zone homelab.local
update add test.homelab.local 300 A 192.168.1.250
send
EOF

# Verify
dig @192.168.1.5 test.homelab.local A +short

# Clean up
nsupdate -k /etc/named/terraform-key.conf <<EOF
server 192.168.1.5
zone homelab.local
update delete test.homelab.local A
send
EOF
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `REFUSED` | Check `allow-update` in zone config, verify TSIG key name matches (must end with dot) |
| `NOTAUTH` | Terraform is updating the wrong zone — check zone name (must end with dot) |
| `TSIG verify failure` | Key secret or algorithm mismatch — regenerate with `tsig-keygen` |
| `connection refused` | BIND not listening on correct interface, check `listen-on` in named.conf |
| Record not appearing | Check journal files (`/var/named/*.jnl`), run `rndc sync` to flush |
| `SERVFAIL` | Zone file syntax error — run `named-checkzone` |
| Provider can't connect | Verify port 53 TCP/UDP is open, check firewall rules |

### Debug BIND

```bash
# Check BIND logs
journalctl -u named -f
tail -f /var/log/named.log

# Reload zone without restart
rndc reload homelab.local

# Sync journal to zone file (view current records)
rndc sync homelab.local

# Freeze zone (stop dynamic updates temporarily)
rndc freeze homelab.local
cat /var/named/homelab.local.zone
rndc thaw homelab.local

# Check zone transfer
dig @192.168.1.5 homelab.local AXFR
```

## Project Structure

```
dns-terraform/
├── main.tf              # Provider and shared resources
├── variables.tf         # Variable declarations
├── outputs.tf           # DNS lookup outputs
├── terraform.tfvars     # TSIG secret and server config
├── records-vms.tf       # VM DNS records
├── records-k8s.tf       # Kubernetes service records
├── records-infra.tf     # Infrastructure records (NAS, switches, etc.)
└── data.tf              # DNS data source lookups
```

## Alternative: Using nsupdate with null_resource

When you need more control or the DNS provider doesn't support a specific record type, use `nsupdate` directly via a provisioner.

```hcl
variable "hostname" {
  type        = string
  default     = "test"
  description = "Hostname for the DNS record"
}

variable "ip_address" {
  type        = string
  default     = "192.168.1.200"
  description = "IP address for the A record"
}

resource "null_resource" "dns_update_a_record" {
  provisioner "local-exec" {
    command = <<-EOT
      nsupdate -k /etc/bind/mykey.key <<EOF
      server ${var.dns_server}
      zone ${var.domain}.
      update delete ${var.hostname}.${var.domain}. A
      update add ${var.hostname}.${var.domain}. 300 A ${var.ip_address}
      send
      EOF
    EOT
  }

  # Remove the record on destroy
  provisioner "local-exec" {
    when    = destroy
    command = <<-EOT
      nsupdate -k /etc/bind/mykey.key <<EOF
      server ${var.dns_server}
      zone ${var.domain}.
      update delete ${self.triggers.fqdn} A
      send
      EOF
    EOT
  }

  triggers = {
    ip_address = var.ip_address
    hostname   = var.hostname
    fqdn       = "${var.hostname}.${var.domain}."
  }
}
# Result: creates the record on apply, removes it on destroy
# Re-runs if ip_address or hostname changes
```

## Security Hardening

### ACL Configuration

Restrict queries and zone transfers to internal networks only.

```bash
# /etc/named.conf (or /etc/bind/named.conf)

acl internal-networks {
    127.0.0.1;
    192.168.1.0/24;
    10.0.0.0/8;
};

zone "homelab.local" IN {
    type master;
    file "/var/named/homelab.local.zone";
    allow-update { key "terraform-key"; };
    allow-query { internal-networks; };
    allow-transfer { none; };
};
```

### Zone File Permissions

Ensure BIND can write to zone files for dynamic updates.

```bash
# RHEL/Rocky (named user)
sudo chown named:named /var/named/
sudo chmod 755 /var/named/

# Ubuntu/Debian (bind user)
sudo chown bind:bind /var/lib/bind/
sudo chmod 755 /var/lib/bind/

# Verify
ls -la /var/named/homelab.local.zone
ls -la /var/named/homelab.local.zone.jnl   # Journal file (created by dynamic updates)
```

### BIND Logging for Audit

```bash
# /etc/named.conf — add logging channel
logging {
    channel dynamic_updates {
        file "/var/log/named/dynamic-updates.log" versions 3 size 5m;
        severity info;
        print-time yes;
        print-category yes;
    };
    category update { dynamic_updates; };
    category update-security { dynamic_updates; };
};
```

```bash
# Create log directory
sudo mkdir -p /var/log/named
sudo chown named:named /var/log/named   # RHEL
sudo chown bind:bind /var/log/named     # Ubuntu

# Monitor updates
tail -f /var/log/named/dynamic-updates.log
```

## Best Practices

1. **Use TSIG authentication** — never rely on IP-based `allow-update` in production
2. **End zone names with a dot** — `homelab.local.` not `homelab.local` (trailing dot = fully qualified)
3. **Keep TSIG secrets in tfvars or env vars** — never commit to version control
4. **Use `for_each` with maps** — manage DNS for fleets of VMs/services from a single variable
5. **Create forward and reverse records together** — PTR records are easy to forget
6. **Set appropriate TTLs** — low (300s) for dynamic hosts, high (3600s) for static infrastructure
7. **Use data sources** to query existing records before modifying them
8. **Run `rndc sync`** after Terraform changes if you need to inspect the zone file
9. **Separate records by purpose** — use multiple `.tf` files (VMs, K8s, infra) for clarity
10. **Test with `nsupdate`** first to verify BIND accepts dynamic updates before using Terraform
