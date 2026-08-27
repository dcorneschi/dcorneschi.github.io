# TLS/SSL Certificates Explained

Chain of trust, Certificate Authorities, CSRs, SANs, certificate types, Let's Encrypt automation, and cert-manager flow in Kubernetes — from basics to production patterns.

## How TLS Works — The Handshake

```
┌──────────┐                              ┌──────────────┐
│  Client  │                              │   Server     │
│ (browser)│                              │ (web server) │
│          │── ClientHello ──────────────▶│              │
│          │ (supported ciphers, TLS ver) │              │
│          │                              │              │
│          │◀─ ServerHello ────────────── │              │
│          │   (chosen cipher, TLS ver)   │              │
│          │                              │              │
│          │◀─ Certificate ────────────── │              │
│          │   (server's public cert +    │              │
│          │    intermediate chain)       │              │
│          │                              │              │
│          │  Verify certificate:         │              │
│          │  1. Check signature chain    │              │
│          │  2. Check expiry             │              │
│          │  3. Check domain (CN/SAN)    │              │
│          │  4. Check revocation (OCSP)  │              │
│          │                              │              │
│          │── Key Exchange ─────────────▶│              │
│          │   (generate shared secret)   │              │
│          │                              │              │
│          │══ Encrypted traffic ════════▶│              │
│          │◀════════════════════════════ │              │
└──────────┘                              └──────────────┘
```

## Certificate Chain of Trust

```
┌──────────────────────────────────────────────────────────────┐
│  Root CA Certificate (self-signed, in OS/browser trust store)│
│  Issuer: DigiCert Global Root G2                             │
│  Subject: DigiCert Global Root G2                            │
│  Validity: 25 years                                          │
│  Stored: OS trust store (/etc/ssl/certs/ or system keychain) │
└──────────────────────┬───────────────────────────────────────┘
                       │ signs
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Intermediate CA Certificate                                │
│  Issuer: DigiCert Global Root G2                            │
│  Subject: DigiCert TLS RSA SHA256 2020 CA1                  │
│  Validity: 10 years                                         │
│  Purpose: Issues end-entity certs (keeps root offline)      │
└──────────────────────┬──────────────────────────────────────┘
                       │ signs
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  End-Entity Certificate (your server's cert)                │
│  Issuer: DigiCert TLS RSA SHA256 2020 CA1                   │
│  Subject: app.example.com                                   │
│  SAN: app.example.com, *.example.com                        │
│  Validity: 1 year (or 90 days for Let's Encrypt)            │
└─────────────────────────────────────────────────────────────┘
```

The client verifies the chain from bottom to top:
1. Server cert signed by intermediate? → Check signature with intermediate's public key
2. Intermediate signed by root? → Check signature with root's public key
3. Root is in trusted store? → Trust established

## Certificate Components

### X.509 Certificate Fields

| Field | Purpose | Example |
|-------|---------|---------|
| Subject (CN) | Common Name — the primary domain (legacy) | `CN=app.example.com` |
| Subject Alternative Name (SAN) | All valid domains/IPs | `DNS:app.example.com, DNS:*.example.com, IP:10.0.1.5` |
| Issuer | Who signed this certificate | `CN=Let's Encrypt Authority X3` |
| Validity (Not Before / Not After) | Valid time window | `2024-01-01 → 2024-04-01` |
| Public Key | Server's public key (RSA/ECDSA) | 2048-bit RSA or P-256 ECDSA |
| Serial Number | Unique ID from the CA | `03:A1:B2:...` |
| Key Usage | What the key can do | digitalSignature, keyEncipherment |
| Extended Key Usage | Specific purposes | serverAuth, clientAuth |
| Authority Key Identifier | Links to issuer's key | Hash of CA's public key |

### Subject Alternative Name (SAN) — The Important One

Modern TLS ignores the CN field for domain validation. Only SAN matters:

```bash
# Check SAN on a certificate:
openssl x509 -in cert.pem -noout -text | grep -A 5 "Subject Alternative Name"
# X509v3 Subject Alternative Name:
#     DNS:app.example.com, DNS:*.example.com, DNS:api.example.com

# Check a live server's SAN:
echo | openssl s_client -connect app.example.com:443 2>/dev/null | \
  openssl x509 -noout -text | grep -A 3 "Subject Alternative Name"
```

SAN types:
- `DNS:domain.com` — matches this exact domain
- `DNS:*.domain.com` — wildcard (one level only, doesn't match `a.b.domain.com`)
- `IP:10.0.1.5` — matches IP address directly

## CSR — Certificate Signing Request

A CSR is what you send to a CA to request a certificate:

```bash
# Generate private key + CSR:
openssl req -new -newkey rsa:2048 -nodes \
  -keyout server.key \
  -out server.csr \
  -subj "/CN=app.example.com" \
  -addext "subjectAltName=DNS:app.example.com,DNS:*.example.com"

# View CSR contents:
openssl req -in server.csr -noout -text

# The CSR contains:
# - Your public key (derived from the private key you generated)
# - Requested subject/SAN
# - Proof you own the private key (signature)
# It does NOT contain your private key (never share that)
```

```
┌────────────────────────────────────────────────────────────┐
│  CSR Flow                                                  │
│                                                            │
│  1. Generate private key (keep secret)                     │
│  2. Generate CSR (contains public key + domain info)       │
│  3. Send CSR to CA                                         │
│  4. CA validates domain ownership (DNS/HTTP challenge)     │
│  5. CA signs and returns the certificate                   │
│  6. Install cert + private key on server                   │
│                                                            │
│  The private key NEVER leaves your server                  │
└────────────────────────────────────────────────────────────┘
```

## Certificate Types

### By Validation Level

| Type | Validation | Time | Cost | Shows in Browser |
|------|-----------|------|------|-----------------|
| DV (Domain Validation) | Prove you control the domain | Minutes | Free (Let's Encrypt) | Padlock |
| OV (Organization Validation) | DV + verify org exists | Days | $50-200/yr | Padlock + org name in details |
| EV (Extended Validation) | OV + thorough legal checks | Weeks | $200-1000/yr | Padlock (green bar removed in modern browsers) |

### By Scope

| Type | Covers | Example |
|------|--------|---------|
| Single domain | One exact domain | `app.example.com` |
| Wildcard | All subdomains (one level) | `*.example.com` |
| Multi-domain (SAN) | Multiple listed domains | `app.com, api.com, cdn.com` |

## Let's Encrypt — Free Automated Certificates

### How ACME (Automatic Certificate Management Environment) Works

```
┌──────────────┐                         ┌────────────────────┐
│  ACME Client │                         │  Let's Encrypt CA  │
│  (certbot,   │                         │                    │
│   cert-mgr)  │── 1.Request cert ──────▶│                    │
│              │                         │                    │
│              │◀─ 2.Challenge ──────────│  "Prove you own    │
│              │   (HTTP-01 or DNS-01)   │   this domain"     │
│              │                         │                    │
│              │── 3.Fulfill challenge ─▶│                    │
│              │   (place file or DNS)   │                    │
│              │                         │                    │
│              │◀─ 4.Signed certificate ─│  "Here's your cert"│
└──────────────┘                         └────────────────────┘
```

### Challenge Types

| Challenge | How It Works | Best For |
|-----------|-------------|----------|
| HTTP-01 | CA hits `http://domain/.well-known/acme-challenge/<token>` | Web servers with port 80 access |
| DNS-01 | Add a TXT record: `_acme-challenge.domain` | Wildcards, internal services, no port 80 |
| TLS-ALPN-01 | Respond on port 443 with special TLS extension | When only port 443 is available |

```bash
# Manual certbot with HTTP-01:
certbot certonly --standalone -d app.example.com

# DNS-01 (for wildcards):
certbot certonly --dns-route53 -d "*.example.com" -d "example.com"

# Output:
# /etc/letsencrypt/live/example.com/fullchain.pem  (cert + intermediate)
# /etc/letsencrypt/live/example.com/privkey.pem    (private key)
```

### Let's Encrypt Limits

| Limit | Value |
|-------|-------|
| Certificates per domain | 50/week |
| SANs per certificate | 100 |
| Certificate validity | 90 days |
| Renewals | Unlimited (renew at 60 days) |
| Rate limit reset | Weekly rolling window |

## cert-manager in Kubernetes

cert-manager automates certificate lifecycle in Kubernetes — request, validate, issue, store, and renew.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  cert-manager (Deployment in cert-manager namespace)            │
│                                                                 │
│  Watches:                                                       │
│    Certificate objects → triggers issuance                      │
│    Ingress annotations → auto-creates Certificates              │
│    CertificateRequest → tracks ACME/CA flow                     │
│                                                                 │
│  Creates:                                                       │
│    Secret (tls.crt + tls.key) → used by Ingress/pods            │
│    CertificateRequest → sent to Issuer                          │
│    Order + Challenge → ACME protocol objects                    │
│                                                                 │
│  Renews automatically before expiry (default: 2/3 of lifetime)  │
└─────────────────────────────────────────────────────────────────┘
```

### Objects

```
Issuer/ClusterIssuer → "Where to get certificates" (Let's Encrypt, Vault, self-signed)
Certificate          → "What certificate I want" (domain, secret name, issuer reference)
CertificateRequest   → "Active request to CA" (internal tracking)
Order                → "ACME order in progress" (for Let's Encrypt)
Challenge            → "Pending domain validation" (HTTP-01 or DNS-01)
Secret               → "The actual cert + key" (type: kubernetes.io/tls)
```

### Setup: Let's Encrypt with DNS-01 (Route 53)

```yaml
# ClusterIssuer (cluster-wide, any namespace can use it):
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - dns01:
        route53:
          region: us-east-1
          # Uses IRSA/Pod Identity for AWS credentials
---
# Certificate:
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-tls
  namespace: production
spec:
  secretName: app-tls-secret       # Secret will contain tls.crt + tls.key
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - app.example.com
  - "*.example.com"
  duration: 2160h                  # 90 days
  renewBefore: 720h                # Renew 30 days before expiry
```

### Setup: Let's Encrypt with HTTP-01 (Ingress)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        ingress:
          class: nginx
---
# Ingress with automatic certificate:
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls-secret      # cert-manager creates this
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app
            port:
              number: 80
```

### Certificate Lifecycle in cert-manager

```
Certificate created
    │
    ▼
cert-manager creates CertificateRequest
    │
    ▼
CertificateRequest creates Order (ACME)
    │
    ▼
Order creates Challenge (DNS-01 or HTTP-01)
    │
    ▼
Challenge fulfilled (DNS record or HTTP endpoint)
    │
    ▼
CA validates and issues certificate
    │
    ▼
cert-manager stores cert in Secret (tls.crt + tls.key)
    │
    ▼
Ingress controller picks up the Secret
    │
    ▼
HTTPS is live

... 60 days later ...

cert-manager detects renewBefore threshold
    │
    ▼
Automatic renewal (same flow, no downtime)
```

### Checking Certificate Status

```bash
# See all certificates:
kubectl get certificates -A

# Check a specific certificate:
kubectl describe certificate app-tls -n production
# Look for:
#   Status: True (Ready)
#   Not After: 2024-06-01T00:00:00Z
#   Renewal Time: 2024-05-02T00:00:00Z

# Check the Secret:
kubectl get secret app-tls-secret -n production -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -dates -subject

# Check cert-manager logs:
kubectl logs -n cert-manager -l app=cert-manager --tail=30

# Check challenges (if stuck):
kubectl get challenges -A
kubectl describe challenge <name>
```

## Self-Signed Certificates (Development/Internal)

```yaml
# Self-signed issuer:
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: self-signed
spec:
  selfSigned: {}
---
# Internal CA (signed by self-signed):
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-ca
  namespace: cert-manager
spec:
  isCA: true
  secretName: internal-ca-secret
  issuerRef:
    name: self-signed
    kind: ClusterIssuer
  commonName: "Internal CA"
  duration: 87600h   # 10 years
---
# ClusterIssuer using the internal CA:
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca-issuer
spec:
  ca:
    secretName: internal-ca-secret
```

## OpenSSL Commands Reference

```bash
# Generate self-signed certificate (testing):
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout key.pem -out cert.pem -days 365 \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

# Generate CSR:
openssl req -new -newkey rsa:2048 -nodes \
  -keyout server.key -out server.csr \
  -subj "/CN=app.example.com"

# View certificate details:
openssl x509 -in cert.pem -noout -text

# Check certificate expiry:
openssl x509 -in cert.pem -noout -enddate

# Check a remote server's certificate:
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | \
  openssl x509 -noout -subject -dates -issuer

# Verify certificate chain:
openssl verify -CAfile ca-bundle.crt -untrusted intermediate.crt server.crt

# Check if key matches certificate:
openssl x509 -noout -modulus -in cert.pem | md5sum
openssl rsa -noout -modulus -in key.pem | md5sum
# Both should produce the same hash

# Convert PEM to PKCS12 (for Java/browsers):
openssl pkcs12 -export -out cert.pfx -inkey key.pem -in cert.pem -certfile ca.pem

# Decode base64 cert from Kubernetes Secret:
kubectl get secret my-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
```

## Kubernetes TLS Secrets

```yaml
# TLS Secret structure:
apiVersion: v1
kind: Secret
metadata:
  name: my-tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded certificate + chain>
  tls.key: <base64-encoded private key>
```

```bash
# Create TLS Secret from files:
kubectl create secret tls my-tls-secret \
  --cert=fullchain.pem \
  --key=privkey.pem \
  -n production

# Verify the secret's cert:
kubectl get secret my-tls-secret -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -subject -dates
```

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Browser shows "Not Secure" | Cert expired, wrong domain, or self-signed | Check expiry, SAN, and chain |
| cert-manager Challenge stuck Pending | DNS not propagated, or HTTP-01 not reachable | Check DNS, Ingress, and firewall |
| "certificate signed by unknown authority" | Missing intermediate in chain, or self-signed CA not trusted | Include full chain (cert + intermediate) |
| Key mismatch | Private key doesn't match certificate | Regenerate CSR with the correct key |
| Let's Encrypt rate limit | Too many requests for same domain | Wait for weekly reset, use staging first |
| Wildcard cert not matching subdomain | Wildcard only covers one level | `*.example.com` doesn't cover `a.b.example.com` |

## Quick Reference

```bash
# Chain of trust: Root CA → Intermediate CA → End-Entity (your cert)
# Client verifies bottom-up: cert → intermediate → root (in trust store)

# SAN (Subject Alternative Name): what domains the cert is valid for
# CN is legacy — only SAN matters for modern validation

# CSR: contains your public key + requested domains
# Private key: NEVER leaves your server

# Let's Encrypt:
# - Free DV certificates
# - 90-day validity (auto-renew at 60 days)
# - HTTP-01 (port 80) or DNS-01 (TXT record) validation
# - Wildcards require DNS-01

# cert-manager (Kubernetes):
# ClusterIssuer → Certificate → CertificateRequest → Order → Challenge → Secret
# Secret contains: tls.crt (cert + chain) + tls.key (private key)
# Auto-renews before expiry (renewBefore field)

# Key commands:
openssl x509 -in cert.pem -noout -text              # View cert
openssl x509 -in cert.pem -noout -enddate           # Check expiry
echo | openssl s_client -connect host:443 | openssl x509 -noout -dates  # Remote check
kubectl get certificates -A                          # cert-manager status
kubectl describe certificate <name>                  # Detailed status
kubectl get challenges -A                            # Stuck validations
```
