# Convert SSH Keys

SSH keys come in different formats depending on the tool that generated them (OpenSSH, PuTTY, SSH.com, PKCS#8). This guide covers converting between all common formats.

## Key Formats Overview

| Format | Extension | Used By | Recognizable By |
|--------|-----------|---------|-----------------|
| OpenSSH (new) | No extension / `.pem` | Linux/macOS ssh, Git | `-----BEGIN OPENSSH PRIVATE KEY-----` |
| OpenSSH (old/PEM) | `.pem` | Older OpenSSH | `-----BEGIN RSA PRIVATE KEY-----` |
| PuTTY (PPK) | `.ppk` | PuTTY, WinSCP, FileZilla | `PuTTY-User-Key-File-` |
| SSH.com (SSH2) | `.pub` | Tectia, some enterprise tools | `---- BEGIN SSH2 PUBLIC KEY ----` |
| PKCS#8 | `.pem` | OpenSSL, Java, APIs | `-----BEGIN PRIVATE KEY-----` or `-----BEGIN ENCRYPTED PRIVATE KEY-----` |
| DER | `.der` | Binary format (Java, Windows) | Binary (not readable as text) |
| RFC 4716 | `.pub` | SSH.com standard public key | `---- BEGIN SSH2 PUBLIC KEY ----` |

## OpenSSH Key Conversions

### Extract Public Key from Private Key

```bash
# From any OpenSSH private key
ssh-keygen -y -f ~/.ssh/id_rsa > ~/.ssh/id_rsa.pub
ssh-keygen -y -f ~/.ssh/id_ed25519 > ~/.ssh/id_ed25519.pub
```

### Convert Old Format (PEM) to New Format (OpenSSH)

```bash
# Convert RSA PEM to new OpenSSH format
ssh-keygen -p -m PEM -f ~/.ssh/id_rsa -N "" -P ""
# This rewrites the file in-place to the new format

# Or explicitly set the new format
ssh-keygen -p -o -f ~/.ssh/id_rsa
```

> **Note:** The new OpenSSH format (since 6.5) starts with `-----BEGIN OPENSSH PRIVATE KEY-----` and supports modern ciphers for passphrase protection.

### Convert New Format to Old PEM Format

```bash
# Convert to old PEM format (needed for some tools)
ssh-keygen -p -m PEM -f ~/.ssh/id_rsa -N "" -P ""
# After this, the file starts with -----BEGIN RSA PRIVATE KEY-----
```

### Change Key Passphrase

```bash
# Add or change passphrase
ssh-keygen -p -f ~/.ssh/id_ed25519

# Remove passphrase
ssh-keygen -p -f ~/.ssh/id_ed25519 -N "" -P "old-passphrase"

# Add passphrase (key has none)
ssh-keygen -p -f ~/.ssh/id_ed25519 -N "new-passphrase" -P ""
```

## PuTTY (.ppk) Conversions

### PuTTY PPK → OpenSSH (Private Key)

```bash
# Using puttygen (install putty-tools)
sudo apt install putty-tools    # Debian/Ubuntu
sudo yum install putty          # RHEL/CentOS
brew install putty              # macOS

# Convert PPK to OpenSSH private key
puttygen key.ppk -O private-openssh -o id_rsa

# Convert PPK v3 to OpenSSH (newer PuTTY versions)
puttygen key.ppk -O private-openssh-new -o id_rsa

# Set permissions
chmod 600 id_rsa
```

### PuTTY PPK → OpenSSH (Public Key)

```bash
puttygen key.ppk -O public-openssh -o id_rsa.pub
```

### OpenSSH → PuTTY PPK

```bash
# Convert OpenSSH private key to PPK
puttygen id_rsa -o key.ppk

# Convert with passphrase
puttygen id_rsa -o key.ppk -P
```

### PuTTY PPK → PEM (for AWS, etc.)

```bash
# PPK to PEM format
puttygen key.ppk -O private -o key.pem
chmod 600 key.pem
```

## SSH.com / RFC 4716 Conversions

### OpenSSH → SSH.com (RFC 4716) Public Key

```bash
ssh-keygen -e -f ~/.ssh/id_rsa.pub -m RFC4716 > id_rsa_ssh2.pub
```

### SSH.com (RFC 4716) → OpenSSH Public Key

```bash
ssh-keygen -i -f id_rsa_ssh2.pub -m RFC4716 > id_rsa.pub
```

### OpenSSH → SSH.com Private Key

```bash
ssh-keygen -e -f ~/.ssh/id_rsa -m SSH2 > id_rsa_ssh2
```

### SSH.com → OpenSSH Private Key

```bash
ssh-keygen -i -f id_rsa_ssh2 -m SSH2 > id_rsa_openssh
```

## PKCS#8 / PEM / DER Conversions (OpenSSL)

### OpenSSH → PKCS#8

```bash
# Convert to PKCS#8 (unencrypted)
openssl pkey -in id_rsa -out id_rsa_pkcs8.pem

# Convert to PKCS#8 (encrypted)
openssl pkey -in id_rsa -out id_rsa_pkcs8_enc.pem -aes256
```

### PKCS#8 → OpenSSH

```bash
# First convert PKCS#8 to traditional PEM
openssl rsa -in id_rsa_pkcs8.pem -out id_rsa_pem.pem

# Then extract public key in OpenSSH format
ssh-keygen -y -f id_rsa_pem.pem > id_rsa.pub
# The private key in PEM format works directly with ssh -i
```

### PEM → DER (Binary)

```bash
# Private key to DER
openssl rsa -in id_rsa.pem -outform DER -out id_rsa.der

# Public key to DER
openssl rsa -in id_rsa.pem -pubout -outform DER -out id_rsa_pub.der
```

### DER → PEM

```bash
# Private key
openssl rsa -in id_rsa.der -inform DER -out id_rsa.pem

# Public key
openssl rsa -in id_rsa_pub.der -inform DER -pubin -out id_rsa_pub.pem
```

### RSA PEM → OpenSSH Public Key

```bash
ssh-keygen -y -f key.pem > key.pub
```

## AWS Key Pair Conversions

### AWS PEM → OpenSSH (Already Compatible)

AWS `.pem` files are standard OpenSSH private keys. Just set permissions:

```bash
chmod 600 my-aws-key.pem
ssh -i my-aws-key.pem ec2-user@<ip>
```

### AWS PEM → PuTTY PPK

```bash
puttygen my-aws-key.pem -o my-aws-key.ppk
```

### Generate OpenSSH Key and Import to AWS

```bash
# Generate
ssh-keygen -t ed25519 -f ~/.ssh/aws-key -C "aws"

# Import public key to AWS
aws ec2 import-key-pair \
    --key-name my-key \
    --public-key-material fileb://~/.ssh/aws-key.pub
```

### PuTTY PPK → Use with AWS

```bash
# Convert PPK to OpenSSH PEM
puttygen my-key.ppk -O private-openssh -o my-key.pem
chmod 600 my-key.pem
ssh -i my-key.pem ec2-user@<ip>
```

## Ed25519 Specific

```bash
# Generate Ed25519 key
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "comment"

# Extract public key
ssh-keygen -y -f ~/.ssh/id_ed25519 > ~/.ssh/id_ed25519.pub

# Ed25519 to PuTTY PPK
puttygen ~/.ssh/id_ed25519 -o id_ed25519.ppk

# Note: Ed25519 keys use the new OpenSSH format only
# They cannot be converted to old RSA PEM format
# They work with puttygen 0.75+ (PPK v3)
```

## View Key Information

```bash
# View key fingerprint
ssh-keygen -lf ~/.ssh/id_rsa
ssh-keygen -lf ~/.ssh/id_rsa.pub

# View fingerprint in MD5 format (older style)
ssh-keygen -lf ~/.ssh/id_rsa -E md5

# View key type and size
ssh-keygen -lf ~/.ssh/id_rsa.pub
# 4096 SHA256:xxxx user@host (RSA)
# 256 SHA256:xxxx user@host (ED25519)

# View full public key
cat ~/.ssh/id_rsa.pub

# View PEM key details with OpenSSL
openssl rsa -in id_rsa.pem -text -noout

# View PKCS#8 key details
openssl pkey -in id_rsa_pkcs8.pem -text -noout

# Check if a key is encrypted
head -1 ~/.ssh/id_rsa
# "-----BEGIN OPENSSH PRIVATE KEY-----" — check inside for "aes256-ctr" or similar
# "-----BEGIN RSA PRIVATE KEY-----" — look for "Proc-Type: 4,ENCRYPTED"
# "-----BEGIN ENCRYPTED PRIVATE KEY-----" — PKCS#8 encrypted

# Verify public/private key match
diff <(ssh-keygen -y -f id_rsa) <(cat id_rsa.pub)
# No output = they match
```

## Identify Key Format

```bash
# Read the first line to identify format:

# OpenSSH new format:
# -----BEGIN OPENSSH PRIVATE KEY-----

# OpenSSH old RSA PEM:
# -----BEGIN RSA PRIVATE KEY-----

# OpenSSH old DSA PEM:
# -----BEGIN DSA PRIVATE KEY-----

# OpenSSH old EC PEM:
# -----BEGIN EC PRIVATE KEY-----

# PKCS#8 unencrypted:
# -----BEGIN PRIVATE KEY-----

# PKCS#8 encrypted:
# -----BEGIN ENCRYPTED PRIVATE KEY-----

# PuTTY PPK:
# PuTTY-User-Key-File-2: ssh-rsa  (or PuTTY-User-Key-File-3:)

# SSH.com / RFC 4716 public:
# ---- BEGIN SSH2 PUBLIC KEY ----

# OpenSSH public:
# ssh-rsa AAAA... (or ssh-ed25519 AAAA...)
```

## Common Conversion Scenarios

### Scenario 1: Windows → Linux

Copied a PuTTY key to Linux and need it for `ssh`:

```bash
puttygen key.ppk -O private-openssh -o ~/.ssh/id_rsa
puttygen key.ppk -O public-openssh -o ~/.ssh/id_rsa.pub
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```

### Scenario 2: Linux → Windows

Need to use a Linux SSH key in PuTTY:

```bash
puttygen ~/.ssh/id_rsa -o key.ppk
# Transfer key.ppk to Windows
```

Or in PuTTYgen GUI: Conversions → Import key → select the OpenSSH key → Save private key.

### Scenario 3: AWS PEM for PuTTY

Downloaded `.pem` from AWS, need to use with PuTTY:

```bash
puttygen aws-key.pem -o aws-key.ppk
# Use aws-key.ppk in PuTTY → Connection → SSH → Auth → Private key
```

### Scenario 4: Old Key Format for Legacy System

A legacy system requires the old PEM format:

```bash
# Create a copy in old format
cp ~/.ssh/id_rsa ~/.ssh/id_rsa_old
ssh-keygen -p -m PEM -f ~/.ssh/id_rsa_old -N "" -P ""
# File now starts with -----BEGIN RSA PRIVATE KEY-----
```

### Scenario 5: Extract Public Key (Lost .pub File)

```bash
ssh-keygen -y -f ~/.ssh/id_rsa > ~/.ssh/id_rsa.pub
ssh-keygen -y -f ~/.ssh/id_ed25519 > ~/.ssh/id_ed25519.pub
```

### Scenario 6: Verify Key Pair Match

```bash
# Compare fingerprints
ssh-keygen -lf ~/.ssh/id_rsa
ssh-keygen -lf ~/.ssh/id_rsa.pub
# If they match, the keys are a pair

# Or directly compare derived public key vs stored public key
diff <(ssh-keygen -y -f ~/.ssh/id_rsa) <(cat ~/.ssh/id_rsa.pub)
```

## Troubleshooting

### "Load key: invalid format"

The key is in a format SSH doesn't recognize (likely PuTTY PPK or wrong permissions):

```bash
# Check format
head -1 key-file

# If PuTTY PPK → convert
puttygen key.ppk -O private-openssh -o key_openssh

# If permissions wrong
chmod 600 key-file
```

### "Permissions too open"

```bash
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 700 ~/.ssh
```

### "Unable to load key" with puttygen

The key may be encrypted or in an unsupported format:

```bash
# Remove passphrase first, then convert
ssh-keygen -p -f id_rsa -N "" -P "old-passphrase"
puttygen id_rsa -o key.ppk
```

### Ed25519 Not Supported by Old PuTTY

PuTTY versions before 0.75 don't support Ed25519. Upgrade PuTTY or use RSA:

```bash
# Generate RSA key instead for compatibility
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_compat
```
