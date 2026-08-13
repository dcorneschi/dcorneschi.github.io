# Ansible Vault: Storing Sudo Passwords Securely

This guide walks through setting up `ansible_become_pass` with Ansible Vault — the proper way to handle sudo passwords without exposing them in plain text.

## The Problem

You need `become: yes` (sudo) in your playbook, but storing `ansible_become_pass=plaintext_password` in inventory files is insecure. The solution: encrypt it with Ansible Vault and reference it as a variable.

```ini
# INSECURE — never do this in production
[servers:vars]
ansible_become_pass=MyPlainTextPassword

# SECURE — reference a vault-encrypted variable
[servers:vars]
ansible_become_pass="{{ vault_sudo_password }}"
```

## Step-by-Step Setup

### Step 1: Create a Vault File

```bash
# Create an encrypted vault file for group-wide secrets
ansible-vault create group_vars/all/vault.yml
```

This will:
1. Ask you to create a vault password (remember it!)
2. Open your default editor (`$EDITOR`)
3. Encrypt the file automatically when you save and exit

### Step 2: Add Variables to the Vault File

In the opened editor, add your sudo password:

```yaml
vault_sudo_password: your_actual_sudo_password_here
vault_ssh_password: your_ssh_password_if_needed
```

Save and exit. The file is now encrypted on disk.

### Step 3: Reference in Inventory

```ini
[rhel_servers]
server01 ansible_host=192.168.1.10
server02 ansible_host=192.168.1.11

[rhel_servers:vars]
ansible_become=yes
ansible_become_method=sudo
ansible_become_pass="{{ vault_sudo_password }}"
```

### Step 4: Run with Vault Password

```bash
# Interactive — prompts for vault password
ansible-playbook -i inventory.ini playbook.yml --ask-vault-pass

# Using a password file (for CI/CD)
ansible-playbook -i inventory.ini playbook.yml --vault-password-file ~/.ansible_vault_pass
```

## Vault Password Methods

### Method 1: Interactive Prompt

```bash
ansible-playbook playbook.yml --ask-vault-pass
```

Best for: manual/one-off runs.

### Method 2: Password File

```bash
# Create the password file
echo "your_vault_password" > ~/.ansible_vault_pass
chmod 600 ~/.ansible_vault_pass

# Use it
ansible-playbook playbook.yml --vault-password-file ~/.ansible_vault_pass
```

Best for: automation/CI/CD pipelines.

### Method 3: Environment Variable

```bash
export ANSIBLE_VAULT_PASSWORD_FILE=~/.ansible_vault_pass
ansible-playbook playbook.yml
```

Best for: persistent shell sessions, CI/CD environments.

### Method 4: ansible.cfg

```ini
# ansible.cfg
[defaults]
vault_password_file = ~/.ansible_vault_pass
```

Best for: project-level configuration (team shared settings).

## Directory Structure

### Global Vault (Single Environment)

```
project/
├── ansible.cfg
├── inventory.ini
├── playbook.yml
└── group_vars/
    └── all/
        └── vault.yml    ← encrypted (all hosts share same sudo pass)
```

### Per-Group Vaults (Different Passwords per Group)

```
project/
├── ansible.cfg
├── inventory.ini
├── playbook.yml
└── group_vars/
    ├── rhel_servers/
    │   └── vault.yml    ← encrypted (RHEL sudo password)
    ├── ubuntu_servers/
    │   └── vault.yml    ← encrypted (Ubuntu sudo password)
    └── all/
        └── vault.yml    ← encrypted (shared secrets: API keys, etc.)
```

### Per-Host Vaults (Different Password per Host)

```
project/
├── ansible.cfg
├── inventory.ini
├── playbook.yml
└── host_vars/
    ├── server01/
    │   └── vault.yml    ← encrypted (server01 sudo password)
    └── server02/
        └── vault.yml    ← encrypted (server02 sudo password)
```

## Complete Working Example

### 1. Create Directory Structure

```bash
mkdir -p ansible-project/group_vars/all
cd ansible-project
```

### 2. Create Encrypted Vault

```bash
ansible-vault create group_vars/all/vault.yml
```

Add in the editor:

```yaml
vault_sudo_password: mySecretSudoPassword123
```

### 3. Create Inventory (`inventory.ini`)

```ini
[rhel_servers]
server01 ansible_host=192.168.1.10
server02 ansible_host=192.168.1.11

[rhel_servers:vars]
ansible_user=deploy
ansible_become=yes
ansible_become_method=sudo
ansible_become_pass="{{ vault_sudo_password }}"
```

### 4. Create Playbook (`playbook.yml`)

```yaml
---
- name: Test sudo access with vault
  hosts: rhel_servers
  become: yes

  tasks:
    - name: Run command as root
      command: whoami
      register: result

    - name: Show result
      debug:
        msg: "Running as: {{ result.stdout }}"

    - name: Check sudo works for package management
      yum:
        name: curl
        state: present
```

### 5. Run It

```bash
# With interactive vault password
ansible-playbook -i inventory.ini playbook.yml --ask-vault-pass

# With password file
ansible-playbook -i inventory.ini playbook.yml --vault-password-file ~/.ansible_vault_pass
```

## Managing Vault Files

```bash
# Create new encrypted file
ansible-vault create group_vars/all/vault.yml

# View contents (read-only, no edit)
ansible-vault view group_vars/all/vault.yml

# Edit encrypted file
ansible-vault edit group_vars/all/vault.yml

# Change the vault password
ansible-vault rekey group_vars/all/vault.yml

# Encrypt an existing plain text file
ansible-vault encrypt group_vars/all/vars.yml

# Decrypt (WARNING: file becomes plain text)
ansible-vault decrypt group_vars/all/vault.yml

# Encrypt a single string (inline vault)
ansible-vault encrypt_string 'myPassword123' --name 'vault_sudo_password'
```

### Inline Vault Variables

Instead of a separate vault file, embed encrypted values directly in YAML:

```bash
# Generate an encrypted string
ansible-vault encrypt_string 'myPassword123' --name 'vault_sudo_password'
```

Output:

```yaml
vault_sudo_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  61626364656667686970...
```

Paste this into any YAML file (`group_vars/all/vars.yml`, playbook, etc.).

## Separating Vault from Non-Vault Variables

A common pattern is to keep encrypted and plain variables in separate files within the same directory:

```
group_vars/
└── rhel_servers/
    ├── vars.yml      ← plain text (non-sensitive settings)
    └── vault.yml     ← encrypted (passwords, keys)
```

```yaml
# group_vars/rhel_servers/vars.yml (plain text)
ansible_become: yes
ansible_become_method: sudo
ansible_become_pass: "{{ vault_sudo_password }}"
ntp_server: ntp.example.com
```

```yaml
# group_vars/rhel_servers/vault.yml (encrypted)
vault_sudo_password: actualPassword123
vault_api_key: sk-abc123...
```

Ansible auto-loads all YAML files in the directory and merges them.

## Multiple Vault IDs (Different Passwords per Environment)

```bash
# Create vaults with different IDs
ansible-vault create --vault-id dev@prompt group_vars/development/vault.yml
ansible-vault create --vault-id prod@~/.vault_pass_prod group_vars/production/vault.yml

# Run with multiple vault IDs
ansible-playbook playbook.yml \
  --vault-id dev@prompt \
  --vault-id prod@~/.vault_pass_prod
```

## Troubleshooting

### "Decryption failed"

```bash
# Wrong vault password — verify you're using the correct one
ansible-vault view group_vars/all/vault.yml

# If you forgot the password, you'll need to recreate the file
```

### "vault_sudo_password is undefined"

```bash
# Variable name mismatch — check spelling in both files
# Verify vault file location is in the Ansible auto-load path:
# group_vars/<group>/vault.yml OR host_vars/<host>/vault.yml

# Test variable resolution
ansible -i inventory.ini server01 -m debug -a "var=vault_sudo_password" --ask-vault-pass
```

### "Missing sudo password" (even with vault)

```bash
# The vault variable isn't being loaded — check path
ansible-inventory -i inventory.ini --host server01 --ask-vault-pass | grep become

# Ensure the inventory references the variable correctly
# Must use: ansible_become_pass="{{ vault_sudo_password }}"
# NOT:     ansible_become_pass=vault_sudo_password  (without quotes/braces)
```

### File permissions

```bash
# Vault password file must be restricted
chmod 600 ~/.ansible_vault_pass

# Vault-encrypted files can be world-readable (they're encrypted)
# But restrict anyway for defense in depth
chmod 640 group_vars/all/vault.yml
```

### Debug Commands

```bash
# Test inventory parsing with vault
ansible-inventory -i inventory.ini --list --ask-vault-pass

# Test variable resolution for a host
ansible -i inventory.ini server01 -m debug -a "var=ansible_become_pass" --ask-vault-pass

# Test connectivity with become
ansible -i inventory.ini server01 -m ping -b --ask-vault-pass
```

## .gitignore

Always exclude vault password files from version control:

```gitignore
# Vault password files
.ansible_vault_pass
*.vault_pass
~/.ansible_vault_pass

# Never commit these
*.retry
```

> **Note:** The encrypted vault files themselves (e.g., `group_vars/all/vault.yml`) ARE safe to commit — they're encrypted. Only the vault password file must be excluded.

## Security Best Practices

1. **Never commit plain text passwords** to version control
2. **Use different vault passwords** for different environments (dev vs prod)
3. **Restrict password file permissions** — `chmod 600`
4. **Use separate vault files** for different secret types (sudo, API keys, certs)
5. **Add `.ansible_vault_pass` to `.gitignore`** before your first commit
6. **Rotate passwords regularly** — use `ansible-vault rekey` to change vault password
7. **Use `--ask-vault-pass`** for interactive use, password files for automation
8. **Consider HashiCorp Vault or AWS Secrets Manager** for enterprise-scale secret management

## Quick Reference

```bash
# Create vault file
ansible-vault create group_vars/all/vault.yml

# Edit vault file
ansible-vault edit group_vars/all/vault.yml

# View vault file
ansible-vault view group_vars/all/vault.yml

# Encrypt existing file
ansible-vault encrypt plaintext_file.yml

# Encrypt a string
ansible-vault encrypt_string 'secret' --name 'var_name'

# Change vault password
ansible-vault rekey group_vars/all/vault.yml

# Run with vault
ansible-playbook playbook.yml --ask-vault-pass
ansible-playbook playbook.yml --vault-password-file ~/.ansible_vault_pass
```
