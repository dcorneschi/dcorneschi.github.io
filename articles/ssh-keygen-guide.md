# Generate SSH Keys

## Quick Start

```bash
# Recommended: Ed25519 (fastest, most secure, smallest key)
ssh-keygen -t ed25519 -C "user@host"

# RSA 4096-bit (wider compatibility with older systems)
ssh-keygen -t rsa -b 4096 -C "user@host"
```

## Key Types Comparison

| Type | Command | Key Size | Security | Speed | Compatibility |
|------|---------|----------|----------|-------|---------------|
| Ed25519 | `ssh-keygen -t ed25519` | 256-bit (fixed) | Excellent | Fastest | OpenSSH 6.5+, modern systems |
| RSA | `ssh-keygen -t rsa -b 4096` | 2048–4096 bit | Good (4096) | Slower | Universal (all systems) |
| ECDSA | `ssh-keygen -t ecdsa -b 521` | 256/384/521 bit | Good | Fast | OpenSSH 5.7+ |
| DSA | `ssh-keygen -t dsa` | 1024-bit (fixed) | Weak | — | **Deprecated** (disabled in OpenSSH 7.0+) |

**Recommendation:** Use Ed25519 unless you need compatibility with very old systems (then use RSA 4096).

## Generate Ed25519 Key (Recommended)

```bash
ssh-keygen -t ed25519 -C "user@host"
```

```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/user/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/user/.ssh/id_ed25519
Your public key has been saved in /home/user/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:xR3tVqMOoXCfQPbU8oAaYNgiK1rfZQXpLa5Zh8VKU2M user@host
```

### With Custom Filename

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_github -C "github-key"
ssh-keygen -t ed25519 -f ~/.ssh/id_work -C "work-key"
ssh-keygen -t ed25519 -f ~/.ssh/id_aws -C "aws-prod"
```

### Without Passphrase (Automation Only)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_deploy -N "" -C "deploy-key"
```

> **Warning:** Keys without passphrases are convenient but risky. Anyone who gets the file has full access. Use only for automation/CI with restricted permissions.

### Non-Interactive (No Prompts)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_auto -N "" -q -C "automated"
# -q = quiet (no output)
# -N "" = no passphrase
```

## Generate RSA Key

```bash
# 4096-bit (recommended minimum for RSA)
ssh-keygen -t rsa -b 4096 -C "user@host"

# With custom filename
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_legacy -C "legacy-system"

# Without passphrase
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_deploy -N "" -C "deploy"
```

### Why 4096 Bits?

| RSA Size | Security Level | Recommendation |
|----------|---------------|----------------|
| 1024 | Weak | **Never use** (breakable) |
| 2048 | Acceptable | Minimum for legacy systems |
| 3072 | Good | Safe through ~2030 |
| 4096 | Strong | Recommended for new RSA keys |

## Generate ECDSA Key

```bash
# 521-bit curve (strongest)
ssh-keygen -t ecdsa -b 521 -C "user@host"

# 384-bit curve
ssh-keygen -t ecdsa -b 384 -C "user@host"

# 256-bit curve (default)
ssh-keygen -t ecdsa -b 256 -C "user@host"
```

Valid ECDSA key sizes: 256, 384, or 521 only (not 512).

## ssh-keygen Options Reference

| Flag | Description |
|------|-------------|
| `-t type` | Key type (ed25519, rsa, ecdsa) |
| `-b bits` | Key size in bits (RSA: 2048–4096, ECDSA: 256/384/521) |
| `-f file` | Output filename |
| `-C comment` | Comment (usually email or purpose) |
| `-N passphrase` | Set passphrase (use `""` for none) |
| `-q` | Quiet mode (suppress output) |
| `-a rounds` | KDF rounds for passphrase (higher = slower brute force) |
| `-o` | Force new OpenSSH format (default for Ed25519) |
| `-m PEM` | Use old PEM format (for compatibility) |

## Passphrase Best Practices

### Set a Strong Passphrase

```bash
# Interactive (prompted)
ssh-keygen -t ed25519 -C "user@host"
# Enter passphrase: (type a strong passphrase)

# With increased KDF rounds (slower brute force)
ssh-keygen -t ed25519 -a 100 -C "user@host"
# Default is 16 rounds; 100 makes brute-forcing ~6x slower
```

### Change Passphrase on Existing Key

```bash
# Interactive
ssh-keygen -p -f ~/.ssh/id_ed25519

# Remove passphrase
ssh-keygen -p -f ~/.ssh/id_ed25519 -P "old-passphrase" -N ""

# Add passphrase to unprotected key
ssh-keygen -p -f ~/.ssh/id_ed25519 -P "" -N "new-passphrase"
```

### Use SSH Agent (Type Passphrase Once)

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
# Enter passphrase once — agent remembers it for the session
```

## File Permissions

SSH refuses to use keys with wrong permissions:

```bash
# Set correct permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_ed25519.pub
chmod 644 ~/.ssh/id_rsa.pub
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/authorized_keys
```

## Verify Generated Keys

```bash
# View fingerprint
ssh-keygen -lf ~/.ssh/id_ed25519.pub
# 256 SHA256:xR3tVqMOoXCfQPbU8oAaYNgiK1rfZQXpLa5Zh8VKU2M user@host (ED25519)

# View fingerprint (MD5 format, for older tools)
ssh-keygen -lf ~/.ssh/id_ed25519.pub -E md5

# Visual ASCII art fingerprint
ssh-keygen -lv -f ~/.ssh/id_ed25519.pub

# View public key content
cat ~/.ssh/id_ed25519.pub

# Verify key pair matches (compare derived public key vs stored)
diff <(ssh-keygen -y -f ~/.ssh/id_ed25519) <(cat ~/.ssh/id_ed25519.pub)
# No output = they match
```

## Deploy Public Key

### ssh-copy-id (Easiest)

```bash
ssh-copy-id user@host
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host
ssh-copy-id -p 2222 user@host
```

### Manual Copy

```bash
# Simple pipe (assumes ~/.ssh exists on remote)
cat ~/.ssh/id_ed25519.pub | ssh user@host "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# Create directory with explicit permissions in one line
cat ~/.ssh/id_ed25519.pub | ssh user@host 'mkdir -m 700 ~/.ssh; cat >> ~/.ssh/authorized_keys; chmod 600 ~/.ssh/authorized_keys'
```

### Manual Setup on the Remote Host

If you can't pipe or use ssh-copy-id (e.g., copy-paste from console):

```bash
# On the remote host:
su - <user>
mkdir ~/.ssh
chmod 700 ~/.ssh
vi ~/.ssh/authorized_keys    # paste the public key content
chmod 600 ~/.ssh/authorized_keys
```

### Using scp (Overwrites Existing Keys)

```bash
# Warning: this replaces authorized_keys entirely
scp ~/.ssh/id_ed25519.pub user@hostname:~/.ssh/authorized_keys
```

> **Caution:** The `scp` method overwrites the remote `authorized_keys` file. Use only for first-time setup on a fresh server.

## Server-Side: Regenerate Host Keys

```bash
# Regenerate all host keys (after cloning a VM, etc.)
sudo ssh-keygen -A

# Restart sshd
sudo systemctl restart sshd
```

### Copy to Clipboard

```bash
# macOS
pbcopy < ~/.ssh/id_ed25519.pub

# Linux (xclip)
xclip -selection clipboard < ~/.ssh/id_ed25519.pub

# Linux (xsel)
xsel --clipboard --input < ~/.ssh/id_ed25519.pub

# Windows (WSL / Git Bash)
clip < ~/.ssh/id_ed25519.pub

# Display in terminal (copy manually)
cat ~/.ssh/id_ed25519.pub
```

### Test Connection

```bash
# General
ssh -i ~/.ssh/id_ed25519 user@host "whoami"

# GitHub
ssh -T git@github.com

# GitLab
ssh -T git@gitlab.com
```

## Multiple Keys

Generate separate keys for different purposes:

```bash
# Create a subdirectory per service
mkdir ~/.ssh/proxmox
ssh-keygen -t ed25519 -f ~/.ssh/proxmox/id_rsa -C "proxmox"

# Or use flat filenames
ssh-keygen -t ed25519 -f ~/.ssh/id_github_personal -C "personal@github"

# Work GitLab
ssh-keygen -t ed25519 -f ~/.ssh/id_gitlab_work -C "user@company.com"

# AWS production
ssh-keygen -t ed25519 -f ~/.ssh/id_aws_prod -C "aws-prod"

# Homelab
ssh-keygen -t ed25519 -f ~/.ssh/id_homelab -C "homelab"
```

Then configure `~/.ssh/config`:

```
Host github.com
    IdentityFile ~/.ssh/id_github_personal
    IdentitiesOnly yes

Host gitlab.company.com
    IdentityFile ~/.ssh/id_gitlab_work
    IdentitiesOnly yes

Host aws-*
    IdentityFile ~/.ssh/id_aws_prod
    IdentitiesOnly yes
```

See [SSH Managing Multiple Keys](articles/ssh-managing-multiple-keys.md) for a full guide.

## Security Hardening

### Use Maximum KDF Rounds

```bash
# 100 rounds (recommended for keys stored on disk)
ssh-keygen -t ed25519 -a 100 -C "secure-key"

# The higher the rounds, the longer it takes to crack via brute force
# 16 (default) → instant verification, fast to crack if passphrase is weak
# 100 → slight delay on login, much harder to brute force
```

### Use FIDO2/Security Key (Hardware-Bound)

```bash
# Generate key bound to a hardware security key (YubiKey, etc.)
ssh-keygen -t ed25519-sk -C "security-key"
# or
ssh-keygen -t ecdsa-sk -C "security-key"

# The private key cannot be extracted from the hardware device
# Requires physical touch on the key for each authentication
```

### Restrict Key Usage (authorized_keys Options)

On the server, restrict what a key can do:

```bash
# In ~/.ssh/authorized_keys:

# Restrict to specific command only
command="/usr/bin/rsync --server" ssh-ed25519 AAAA... backup-key

# Restrict source IP
from="192.168.1.0/24" ssh-ed25519 AAAA... internal-key

# Disable forwarding and PTY
no-port-forwarding,no-X11-forwarding,no-pty ssh-ed25519 AAAA... deploy-key

# Combined restrictions
from="10.0.0.5",command="/usr/local/bin/backup.sh",no-pty,no-port-forwarding ssh-ed25519 AAAA... backup-key
```

## Key Rotation

```bash
# 1. Generate new key
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_new -C "rotated-$(date +%Y%m)"

# 2. Deploy new public key (while old key still works)
ssh-copy-id -i ~/.ssh/id_ed25519_new.pub user@host

# 3. Test new key
ssh -i ~/.ssh/id_ed25519_new user@host "whoami"

# 4. Remove old key from authorized_keys on server
ssh user@host "sed -i '/old-key-comment/d' ~/.ssh/authorized_keys"

# 5. Replace local key
mv ~/.ssh/id_ed25519 ~/.ssh/id_ed25519_old
mv ~/.ssh/id_ed25519_new ~/.ssh/id_ed25519
mv ~/.ssh/id_ed25519_new.pub ~/.ssh/id_ed25519.pub

# 6. Delete old key after confirming everything works
rm ~/.ssh/id_ed25519_old
```

## Summary

| Use Case | Command |
|----------|---------|
| General purpose (modern) | `ssh-keygen -t ed25519 -C "user@host"` |
| Legacy compatibility | `ssh-keygen -t rsa -b 4096 -C "user@host"` |
| Automation (no passphrase) | `ssh-keygen -t ed25519 -f key -N "" -C "deploy"` |
| Non-interactive | `ssh-keygen -t ed25519 -f key -N "" -q` |
| Custom filename | `ssh-keygen -t ed25519 -f ~/.ssh/id_purpose -C "purpose"` |
| High security | `ssh-keygen -t ed25519 -a 100 -C "secure"` |
| Hardware key (FIDO2) | `ssh-keygen -t ed25519-sk -C "hardware"` |
