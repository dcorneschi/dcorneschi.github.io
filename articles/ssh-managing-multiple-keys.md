# Managing Multiple SSH Keys

When working with multiple services — AWS, Proxmox, GitHub, GitLab, homelab servers — using a single SSH key for everything is a security risk. If that key is compromised, every service is exposed. The better approach is one key per service (or per environment), managed through `~/.ssh/config`.

## Directory Structure

Organize keys by service or environment:

```
~/.ssh/
├── config
├── known_hosts
├── id_ed25519              # personal default
├── id_ed25519.pub
├── aws_prod                # AWS production
├── aws_prod.pub
├── aws_dev                 # AWS development
├── aws_dev.pub
├── proxmox                 # Proxmox hypervisor
├── proxmox.pub
├── github                  # GitHub
├── github.pub
├── gitlab_work             # GitLab (work)
├── gitlab_work.pub
├── homelab                 # homelab servers
└── homelab.pub
```

## Generate Keys Per Service

Use descriptive filenames and comments to identify each key:

```bash
# AWS production
ssh-keygen -t ed25519 -f ~/.ssh/aws_prod -C "aws-prod"

# AWS development
ssh-keygen -t ed25519 -f ~/.ssh/aws_dev -C "aws-dev"

# Proxmox hypervisor
ssh-keygen -t ed25519 -f ~/.ssh/proxmox -C "proxmox"

# GitHub
ssh-keygen -t ed25519 -f ~/.ssh/github -C "user@github"

# GitLab (work)
ssh-keygen -t ed25519 -f ~/.ssh/gitlab_work -C "user@company-gitlab"

# Homelab servers
ssh-keygen -t ed25519 -f ~/.ssh/homelab -C "homelab"
```

## SSH Config File

The config file (`~/.ssh/config`) is where everything comes together. It maps hosts to their specific keys, usernames, and options.

```
# === GitHub ===
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github
    IdentitiesOnly yes

# === GitLab (work) ===
Host gitlab.company.com
    HostName gitlab.company.com
    User git
    IdentityFile ~/.ssh/gitlab_work
    IdentitiesOnly yes

# === AWS Production ===
Host aws-prod-*
    User ec2-user
    IdentityFile ~/.ssh/aws_prod
    IdentitiesOnly yes

Host aws-prod-web
    HostName 10.0.1.50
    
Host aws-prod-db
    HostName 10.0.1.51

Host aws-prod-bastion
    HostName 52.1.2.3

# Jump through bastion for production hosts
Host aws-prod-web aws-prod-db
    ProxyJump aws-prod-bastion

# === AWS Development ===
Host aws-dev-*
    User ec2-user
    IdentityFile ~/.ssh/aws_dev
    IdentitiesOnly yes

Host aws-dev-app
    HostName 172.16.0.10

Host aws-dev-bastion
    HostName 54.10.20.30

Host aws-dev-app
    ProxyJump aws-dev-bastion

# === Proxmox ===
Host pve1
    HostName 192.168.1.10
    User root
    IdentityFile ~/.ssh/proxmox
    IdentitiesOnly yes
    Port 22

Host pve2
    HostName 192.168.1.11
    User root
    IdentityFile ~/.ssh/proxmox
    IdentitiesOnly yes

# === Homelab ===
Host homelab-*
    User admin
    IdentityFile ~/.ssh/homelab
    IdentitiesOnly yes

Host homelab-docker
    HostName 192.168.1.20

Host homelab-k8s-master
    HostName 192.168.1.30

Host homelab-k8s-worker1
    HostName 192.168.1.31

Host homelab-nas
    HostName 192.168.1.40
    User root

# === Global Defaults ===
Host *
    AddKeysToAgent yes
    ServerAliveInterval 60
    ServerAliveCountMax 3
    IdentitiesOnly yes
    IdentityFile ~/.ssh/id_ed25519
```

The `Host *` block at the end serves as a fallback — if no other `Host` block matches, the default key (`~/.ssh/id_ed25519`) is used. Place specific rules above and defaults at the bottom.

### Domain and Subnet Wildcards

You can match entire domains or cloud provider patterns:

```
# All AWS instances (by domain)
Host *.amazonaws.com *.compute.internal
    User ec2-user
    IdentityFile ~/.ssh/aws_prod
    IdentitiesOnly yes

# All Proxmox hosts (by local domain)
Host *.proxmox.local proxmox-*
    User root
    IdentityFile ~/.ssh/proxmox
    IdentitiesOnly yes

# Entire subnet
Host 192.168.1.*
    User admin
    IdentityFile ~/.ssh/homelab
    IdentitiesOnly yes
```

## Key Concepts

### IdentitiesOnly

The most important directive for multi-key setups. Without it, the SSH client offers **all** keys loaded in the agent to every host, which leads to:

- "Too many authentication failures" errors (server's `MaxAuthTries` exceeded)
- Unintended key exposure to services that don't need them

```
Host *
    IdentitiesOnly yes
```

With `IdentitiesOnly yes`, SSH only offers the key specified in `IdentityFile` — nothing else from the agent.

### AddKeysToAgent

Automatically add keys to the running SSH agent after first use (so you type the passphrase only once per session):

```
Host *
    AddKeysToAgent yes
```

Combined values:

| Value | Behavior |
|-------|----------|
| `yes` | Add key to agent after successful authentication |
| `no` | Never auto-add (default) |
| `confirm` | Add but require confirmation for each use |
| `ask` | Ask before adding |
| `<time>` | Add with a lifetime (e.g., `1h`, `3600`) |

### Host Patterns

| Pattern | Matches |
|---------|---------|
| `Host aws-prod-*` | Any host starting with `aws-prod-` |
| `Host *.example.com` | Any subdomain of example.com |
| `Host 192.168.1.*` | Any host in that subnet |
| `Host server1 server2` | Exactly server1 or server2 |
| `Host !bastion *` | Everything except bastion |

## Deploy Keys to Services

### AWS EC2

AWS uses key pairs created in the console or imported:

```bash
# Import your public key to AWS
aws ec2 import-key-pair \
    --key-name "my-aws-prod-key" \
    --public-key-material fileb://~/.ssh/aws_prod.pub \
    --region us-east-1
```

Or paste the public key content when launching an instance.

### Proxmox

Copy your public key to the Proxmox host:

```bash
ssh-copy-id -i ~/.ssh/proxmox.pub root@192.168.1.10
```

Or via the Proxmox web UI: Datacenter → Permissions → Users → select user → SSH Keys.

### GitHub

Add your public key at [github.com/settings/keys](https://github.com/settings/keys):

```bash
# Copy to clipboard (macOS)
cat ~/.ssh/github.pub | pbcopy

# Copy to clipboard (Linux)
cat ~/.ssh/github.pub | xclip -selection clipboard

# Test the connection
ssh -T git@github.com
```

### GitLab

Add your public key at Settings → SSH Keys:

```bash
cat ~/.ssh/gitlab_work.pub | pbcopy

# Test
ssh -T git@gitlab.company.com
```

### General Linux Servers (Homelab)

```bash
# Copy key to each server
ssh-copy-id -i ~/.ssh/homelab.pub admin@192.168.1.20
ssh-copy-id -i ~/.ssh/homelab.pub admin@192.168.1.30
ssh-copy-id -i ~/.ssh/homelab.pub admin@192.168.1.31
ssh-copy-id -i ~/.ssh/homelab.pub root@192.168.1.40
```

## SSH Agent Management

### Start and Load Keys

```bash
# Start the agent
eval "$(ssh-agent -s)"

# Add specific keys
ssh-add ~/.ssh/github
ssh-add ~/.ssh/aws_prod
ssh-add ~/.ssh/proxmox

# Add all keys at once
ssh-add ~/.ssh/github ~/.ssh/aws_prod ~/.ssh/aws_dev ~/.ssh/proxmox ~/.ssh/homelab

# Add with lifetime (auto-removed after expiry)
ssh-add -t 8h ~/.ssh/aws_prod

# List loaded keys
ssh-add -l
```

### Specify Key Per Connection (No Config)

For one-off connections without relying on `~/.ssh/config`:

```bash
ssh -i ~/.ssh/aws_prod ec2-user@aws-server
ssh -i ~/.ssh/proxmox root@proxmox-server
ssh -i ~/.ssh/homelab admin@192.168.1.20
```

### Auto-Load Keys on Login

Add to `~/.bashrc` or `~/.zshrc` to start the agent and load keys automatically on every shell session:

```bash
# Auto-load SSH keys on login
if [ -z "$SSH_AUTH_SOCK" ]; then
    eval $(ssh-agent -s)
    ssh-add ~/.ssh/aws_prod 2>/dev/null
    ssh-add ~/.ssh/proxmox 2>/dev/null
    ssh-add ~/.ssh/github 2>/dev/null
    ssh-add ~/.ssh/homelab 2>/dev/null
fi
```

### Persistent Agent with Keychain (Linux)

To avoid re-adding keys after every reboot, use `keychain`:

```bash
# Install
sudo apt install keychain    # Debian/Ubuntu
sudo yum install keychain    # RHEL/CentOS

# Add to ~/.bashrc or ~/.zshrc
eval $(keychain --eval --agents ssh github aws_prod proxmox homelab)
```

### macOS Keychain Integration

macOS stores passphrases in the system Keychain:

```
Host *
    AddKeysToAgent yes
    UseKeychain yes
    IdentitiesOnly yes
```

```bash
# Add key passphrase to macOS Keychain
ssh-add --apple-use-keychain ~/.ssh/github
```

## Multiple Git Identities

When you need different SSH keys for different Git services (or the same service with different accounts):

### Different Services

```
# Personal GitHub
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_personal
    IdentitiesOnly yes

# Work GitLab
Host gitlab.company.com
    HostName gitlab.company.com
    User git
    IdentityFile ~/.ssh/gitlab_work
    IdentitiesOnly yes
```

### Multiple Accounts on the Same Service

Use host aliases:

```
# Personal GitHub account
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_personal
    IdentitiesOnly yes

# Work GitHub account
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_work
    IdentitiesOnly yes
```

Then clone with the alias:

```bash
# Personal repos
git clone git@github-personal:myuser/repo.git

# Work repos
git clone git@github-work:company/repo.git
```

Existing repos — update the remote:

```bash
git remote set-url origin git@github-work:company/repo.git
```

## Testing and Verification

### Test Each Key

```bash
# GitHub
ssh -T git@github.com

# GitLab
ssh -T git@gitlab.company.com

# AWS (will show username or error but confirms key works)
ssh -i ~/.ssh/aws_prod ec2-user@52.1.2.3 "whoami"

# Proxmox
ssh pve1 "hostname"

# Homelab
ssh homelab-docker "uptime"
```

### Verbose Connection (See Which Key is Used)

```bash
ssh -v homelab-docker 2>&1 | grep "Offering"
# debug1: Offering public key: /home/user/.ssh/homelab ED25519 ...

ssh -v git@github.com 2>&1 | grep "Offering"
# debug1: Offering public key: /home/user/.ssh/github ED25519 ...
```

### Verify Config Resolution

```bash
# Print effective config for a host (without connecting)
ssh -G aws-prod-web

# Check which IdentityFile is resolved
ssh -G aws-prod-web | grep identityfile
```

## Key Rotation

Rotate keys periodically, especially for production services:

```bash
# 1. Generate new key
ssh-keygen -t ed25519 -f ~/.ssh/aws_prod_new -C "aws-prod-$(date +%Y%m)"

# 2. Deploy new public key to the service
aws ec2 import-key-pair --key-name "aws-prod-new" \
    --public-key-material fileb://~/.ssh/aws_prod_new.pub

# 3. Test new key
ssh -i ~/.ssh/aws_prod_new ec2-user@host "whoami"

# 4. Update config to use new key
sed -i 's|~/.ssh/aws_prod|~/.ssh/aws_prod_new|' ~/.ssh/config

# 5. Remove old key from the service
aws ec2 delete-key-pair --key-name "aws-prod-old"

# 6. Remove old key locally
rm ~/.ssh/aws_prod ~/.ssh/aws_prod.pub

# 7. Rename new key (optional)
mv ~/.ssh/aws_prod_new ~/.ssh/aws_prod
mv ~/.ssh/aws_prod_new.pub ~/.ssh/aws_prod.pub
```

## Permissions

SSH refuses to use keys or configs with wrong permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/aws_prod ~/.ssh/proxmox ~/.ssh/github ~/.ssh/homelab
chmod 644 ~/.ssh/*.pub
```

## Troubleshooting

### "Too many authentication failures"

The SSH client is offering too many keys before the right one. Fix:

```
Host *
    IdentitiesOnly yes
```

This ensures only the key in `IdentityFile` is offered to each host.

### "Permission denied (publickey)"

```bash
# Check which key is being offered
ssh -v user@host 2>&1 | grep "Offering"

# Verify the public key is on the remote
ssh user@host "cat ~/.ssh/authorized_keys" | grep "$(cat ~/.ssh/mykey.pub)"

# Check permissions on the remote
ssh user@host "ls -la ~/.ssh/ && stat ~/.ssh/authorized_keys"
```

### Key Not Being Used Despite Config

```bash
# Print resolved config
ssh -G hostname | grep -i identity

# Common causes:
# - IdentityFile path is wrong
# - Host pattern doesn't match
# - Another Host block matches first (SSH uses first match)
# - Missing IdentitiesOnly (agent offers a different key first)
```

### Agent Has Too Many Keys

```bash
# List loaded keys
ssh-add -l

# Remove all and add only what you need
ssh-add -D
ssh-add ~/.ssh/github ~/.ssh/aws_prod
```

## Summary

| Practice | Why |
|----------|-----|
| One key per service/environment | Limits blast radius if a key is compromised |
| `IdentitiesOnly yes` | Prevents offering wrong keys to servers |
| Descriptive filenames | `aws_prod`, `proxmox`, `github` — not `id_rsa_2` |
| `-C` comment on keygen | Identify which key is which in `authorized_keys` |
| `AddKeysToAgent yes` | Type passphrase once per session |
| Host aliases in config | `ssh pve1` instead of `ssh -i ~/.ssh/proxmox root@192.168.1.10` |
| Regular rotation | Especially for production and shared keys |
| Passphrase on every key | Protects if the key file is stolen |
| Back up public keys | Keep `.pub` files in a safe location for recovery |
