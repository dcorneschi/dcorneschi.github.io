# SNI and TLS Certificates Guide

Server Name Indication (SNI) allows multiple TLS certificates to be served from a single IP address. This guide covers how SNI works, certificate configuration with common web servers and reverse proxies, testing, and troubleshooting.

## How SNI Works

Without SNI, each SSL/TLS certificate requires a dedicated IP address. SNI solves this by including the requested hostname in the TLS handshake (ClientHello), allowing the server to select the correct certificate before the encrypted connection is established.

```
Client                          Server (single IP, multiple certs)
  |                                |
  |--- ClientHello (SNI: a.com) -->|
  |                                | (selects cert for a.com)
  |<-- ServerHello + cert a.com ---|
  |                                |
  |--- ClientHello (SNI: b.com) -->|
  |                                | (selects cert for b.com)
  |<-- ServerHello + cert b.com ---|
```

### Key Points

- SNI is a TLS extension (defined in RFC 6066)
- The hostname is sent in plaintext during the ClientHello (before encryption)
- Supported by all modern browsers and clients
- Required for hosting multiple HTTPS sites on one IP
- Encrypted SNI (ESNI/ECH) is a newer extension that encrypts the hostname

## Certificate Types

| Type | Covers | Use Case |
|------|--------|----------|
| Single-domain | `example.com` only | One specific site |
| Wildcard | `*.example.com` | All subdomains of one level |
| Multi-domain (SAN) | Multiple specific domains | Several unrelated domains |
| Wildcard + SAN | `*.a.com`, `*.b.com`, `c.com` | Complex multi-domain setups |

### Subject Alternative Names (SANs)

SANs allow a single certificate to cover multiple domains:

```bash
# View SANs in a certificate
openssl x509 -in cert.pem -noout -text | grep -A1 "Subject Alternative Name"

# Example output:
# X509v3 Subject Alternative Name:
#     DNS:example.com, DNS:www.example.com, DNS:api.example.com
```

## Generate Certificates

### Self-Signed (Testing)

```bash
# Single domain
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout server.key -out server.crt \
    -subj "/CN=example.com"

# With SANs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout server.key -out server.crt \
    -subj "/CN=example.com" \
    -addext "subjectAltName=DNS:example.com,DNS:www.example.com,DNS:api.example.com"

# Wildcard
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout wildcard.key -out wildcard.crt \
    -subj "/CN=*.example.com" \
    -addext "subjectAltName=DNS:*.example.com,DNS:example.com"
```

### Let's Encrypt (Production)

```bash
# Single domain
certbot certonly --standalone -d example.com

# Multiple domains (one cert with SANs)
certbot certonly --standalone -d example.com -d www.example.com -d api.example.com

# Wildcard (requires DNS challenge)
certbot certonly --manual --preferred-challenges dns -d "*.example.com" -d example.com

# With Nginx plugin
certbot --nginx -d example.com -d www.example.com

# With Apache plugin
certbot --apache -d example.com -d www.example.com

# Renew all certificates
certbot renew
```

## Nginx SNI Configuration

### Multiple Server Blocks (SNI-Based)

```nginx
# Site 1
server {
    listen 443 ssl;
    server_name app1.example.com;

    ssl_certificate     /etc/letsencrypt/live/app1.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app1.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:8001;
    }
}

# Site 2
server {
    listen 443 ssl;
    server_name app2.example.com;

    ssl_certificate     /etc/letsencrypt/live/app2.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app2.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:8002;
    }
}

# Default (fallback when SNI doesn't match)
server {
    listen 443 ssl default_server;
    server_name _;

    ssl_certificate     /etc/ssl/certs/default.crt;
    ssl_certificate_key /etc/ssl/private/default.key;

    return 444;  # Close connection
}
```

### Wildcard Certificate Shared Across Blocks

```nginx
# Shared SSL settings
ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

server {
    listen 443 ssl;
    server_name app1.example.com;
    # Uses the shared wildcard cert
    location / { proxy_pass http://localhost:8001; }
}

server {
    listen 443 ssl;
    server_name app2.example.com;
    # Same wildcard cert covers this too
    location / { proxy_pass http://localhost:8002; }
}
```

## Apache SNI Configuration

```apache
# Enable SSL and SNI
<VirtualHost *:443>
    ServerName app1.example.com
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/app1.example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/app1.example.com/privkey.pem
    ProxyPass / http://localhost:8001/
    ProxyPassReverse / http://localhost:8001/
</VirtualHost>

<VirtualHost *:443>
    ServerName app2.example.com
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/app2.example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/app2.example.com/privkey.pem
    ProxyPass / http://localhost:8002/
    ProxyPassReverse / http://localhost:8002/
</VirtualHost>
```

## Traefik SNI Configuration

### Dynamic Routing (Docker Labels)

```yaml
# docker-compose.yml
services:
  app1:
    image: myapp1
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app1.rule=Host(`app1.example.com`)"
      - "traefik.http.routers.app1.tls=true"
      - "traefik.http.routers.app1.tls.certresolver=letsencrypt"

  app2:
    image: myapp2
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app2.rule=Host(`app2.example.com`)"
      - "traefik.http.routers.app2.tls=true"
      - "traefik.http.routers.app2.tls.certresolver=letsencrypt"
```

### Static File Configuration

```yaml
# traefik.yml
entryPoints:
  websecure:
    address: ":443"

certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@example.com
      storage: /acme.json
      httpChallenge:
        entryPoint: web

tls:
  certificates:
    - certFile: /certs/app1.crt
      keyFile: /certs/app1.key
    - certFile: /certs/app2.crt
      keyFile: /certs/app2.key
```

### SNI Routing (TCP Passthrough)

```yaml
# Route based on SNI without terminating TLS
tcp:
  routers:
    app1:
      rule: "HostSNI(`app1.example.com`)"
      service: app1
      tls:
        passthrough: true
    app2:
      rule: "HostSNI(`app2.example.com`)"
      service: app2
      tls:
        passthrough: true
```

## HAProxy SNI Configuration

```haproxy
frontend https_front
    bind *:443 ssl crt /etc/haproxy/certs/

    # SNI-based routing
    use_backend app1_backend if { ssl_fc_sni app1.example.com }
    use_backend app2_backend if { ssl_fc_sni app2.example.com }
    default_backend default_backend

backend app1_backend
    server app1 127.0.0.1:8001

backend app2_backend
    server app2 127.0.0.1:8002
```

### HAProxy SNI Passthrough (No TLS Termination)

```haproxy
frontend tcp_front
    bind *:443
    mode tcp
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }

    use_backend app1_backend if { req_ssl_sni -i app1.example.com }
    use_backend app2_backend if { req_ssl_sni -i app2.example.com }

backend app1_backend
    mode tcp
    server app1 192.168.1.10:443

backend app2_backend
    mode tcp
    server app2 192.168.1.20:443
```

## Testing SNI

### openssl s_client

```bash
# Connect with specific SNI hostname
openssl s_client -connect 192.168.1.5:443 -servername app1.example.com

# Show certificate details
openssl s_client -connect 192.168.1.5:443 -servername app1.example.com 2>/dev/null | \
    openssl x509 -noout -subject -issuer -dates

# Check SANs
openssl s_client -connect 192.168.1.5:443 -servername app1.example.com 2>/dev/null | \
    openssl x509 -noout -text | grep -A1 "Subject Alternative Name"

# Test without SNI (should get default cert or error)
openssl s_client -connect 192.168.1.5:443 -noservername

# Test specific TLS version
openssl s_client -connect 192.168.1.5:443 -servername app1.example.com -tls1_2
openssl s_client -connect 192.168.1.5:443 -servername app1.example.com -tls1_3
```

### curl

```bash
# Test with SNI (curl does this automatically)
curl -I https://app1.example.com

# Force connection to specific IP (resolve override)
curl -I --resolve app1.example.com:443:192.168.1.5 https://app1.example.com

# Show certificate info
curl -vI --resolve app1.example.com:443:192.168.1.5 https://app1.example.com 2>&1 | grep -E "subject:|issuer:|expire"

# Skip certificate verification (testing)
curl -kI --resolve app1.example.com:443:192.168.1.5 https://app1.example.com
```

### Check Certificate Expiration

```bash
# Remote check
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | \
    openssl x509 -noout -dates

# Local file
openssl x509 -in cert.pem -noout -enddate

# Days until expiration
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | \
    openssl x509 -noout -checkend 2592000 && echo "Valid > 30 days" || echo "Expires within 30 days"
```

### Batch Test Multiple Domains

```bash
#!/bin/bash
# Check certificates for multiple domains on the same IP
IP="192.168.1.5"
DOMAINS=("app1.example.com" "app2.example.com" "app3.example.com")

for domain in "${DOMAINS[@]}"; do
    echo "=== $domain ==="
    echo | openssl s_client -connect "${IP}:443" -servername "$domain" 2>/dev/null | \
        openssl x509 -noout -subject -dates
    echo
done
```

## Certificate Chain Verification

```bash
# Verify certificate chain
openssl verify -CAfile ca-bundle.crt -untrusted intermediate.crt server.crt

# Show full chain
openssl s_client -connect example.com:443 -servername example.com -showcerts

# Check chain order
openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | \
    grep -E "s:|i:" | head -10

# Verify cert matches key
openssl x509 -noout -modulus -in server.crt | md5sum
openssl rsa -noout -modulus -in server.key | md5sum
# Both should produce the same hash
```

## Troubleshooting

### Wrong Certificate Served

```bash
# Check what SNI is being sent
openssl s_client -connect ip:443 -servername hostname 2>/dev/null | \
    openssl x509 -noout -subject

# If wrong cert is served:
# 1. Verify server_name/ServerName matches in config
# 2. Check certificate file paths are correct
# 3. Ensure default_server block has a catch-all cert
# 4. Restart web server after config changes
```

### "SSL: error:... :tlsv1 alert internal error"

```bash
# Usually means SNI hostname doesn't match any configured cert
# Check available certs on the server
openssl s_client -connect ip:443 -servername hostname -tlsextdebug 2>&1 | grep -i "peer"

# Verify the correct cert covers the hostname
openssl x509 -in cert.pem -noout -text | grep -A1 "Subject Alternative Name"
```

### Certificate Not Trusted

```bash
# Check if intermediate CA is included
openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | \
    grep "Verify return code"

# Return code 0 = valid chain
# Return code 21 = unable to verify first certificate (missing intermediate)

# Fix: concatenate cert + intermediate
cat server.crt intermediate.crt > fullchain.crt
```

### SNI Not Working (Legacy Clients)

Clients that don't support SNI:
- Python < 2.7.9
- Java < 7
- Android < 3.0
- iOS < 4.0
- Windows XP (IE 6/7/8)
- curl < 7.18.1 (compiled without SNI)

Modern support (all of these work):
- All current browsers (Chrome, Firefox, Safari, Edge)
- iOS 4+, Android 3+
- Python 2.7.9+ / 3.2+
- Java 7+
- curl 7.18.1+

For legacy clients, use a wildcard or SAN certificate instead of relying on SNI.

### Nagios check_http with SNI

```bash
# Correct syntax for SNI-based checks
/usr/local/nagios/libexec/check_http -H hostname.example.com -I 192.168.1.5 -S

# -H sets the Host header AND SNI hostname
# -I sets the IP to connect to
# -S enables SSL/TLS
```

## Best Practices

- Use SAN certificates when possible (covers multiple domains in one cert)
- Always include both `example.com` and `www.example.com` in SANs
- Configure a default/fallback certificate for unmatched SNI requests
- Automate renewal with certbot or ACME clients
- Monitor certificate expiration (30-day warning threshold)
- Include intermediate certificates in the chain (fullchain)
- Use TLS 1.2+ only (disable SSLv3, TLS 1.0, TLS 1.1)
- Test with `openssl s_client -servername` after any certificate change
- Keep private keys with 600 permissions, owned by root

## Quick Reference

| Action | Command |
|--------|---------|
| Test SNI connection | `openssl s_client -connect ip:443 -servername host` |
| Check cert subject | `openssl x509 -in cert.pem -noout -subject` |
| Check SANs | `openssl x509 -in cert.pem -noout -text \| grep -A1 "Alternative"` |
| Check expiration | `openssl x509 -in cert.pem -noout -enddate` |
| Verify chain | `openssl verify -CAfile ca.crt server.crt` |
| Cert matches key | `openssl x509 -noout -modulus -in cert.pem \| md5sum` |
| curl with SNI | `curl --resolve host:443:ip https://host` |
| Generate self-signed | `openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out cert.pem` |
| Let's Encrypt | `certbot certonly -d example.com` |
| Wildcard cert | `certbot certonly --manual --preferred-challenges dns -d "*.example.com"` |
