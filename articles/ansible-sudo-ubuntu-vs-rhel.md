# Ansible Sudo: Ubuntu vs RHEL Differences

A common issue: your Ansible playbook runs fine on Ubuntu but fails with "Missing sudo password" on RHEL/CentOS/Rocky Linux. This happens because the two families handle sudo configuration differently out of the box.

## The Problem

```
FAILED! => {"msg": "Missing sudo password"}
```

The playbook uses `become: yes` and works on Ubuntu without prompting for a password, but RHEL-based systems require one. The difference isn't in Ansible — it's in how each distro configures `sudo` by default.

## Key Differences

| Aspect | Ubuntu/Debian | RHEL/CentOS/Rocky |
|--------|--------------|-------------------|
| Sudo group name | `sudo` | `wheel` |
| Default user in sudo group | Yes (during install) | Not always |
| Timestamp timeout | 15 minutes (passwordless window) | Often 0 (password every time) |
| Passwordless sudo | Common (especially cloud images) | Rare by default |
| SSH key auth | Usually pre-configured | May need explicit setup |

### 1. Sudo Group Names

**Ubuntu/Debian** — uses the `sudo` group:

```bash
# /etc/sudoers
%sudo   ALL=(ALL:ALL) ALL
```

**RHEL/CentOS/Rocky** — uses the `wheel` group:

```bash
# /etc/sudoers
%wheel  ALL=(ALL)       ALL
```

### 2. User Group Membership

**Ubuntu:**
- The initial user is automatically added to `sudo` group during installation
- Cloud images (AWS, Azure) come with a user already in `sudo` with NOPASSWD

**RHEL/CentOS/Rocky:**
- Users may not be in `wheel` group by default
- You must explicitly add them: `usermod -aG wheel username`

### 3. Sudo Timestamp Behavior

**Ubuntu:**

```bash
# Allows sudo without password for 15 minutes after first auth
Defaults   timestamp_timeout=15
```

After entering the password once, subsequent `sudo` calls within 15 minutes don't ask again. This is why Ansible "just works" — the first task authenticates, and the rest ride the cache.

**RHEL/CentOS/Rocky:**

```bash
# Often requires password for every sudo command (timeout=0 or very short)
Defaults   timestamp_timeout=0
```

Every `sudo` invocation requires the password, which breaks Ansible's become mechanism.

## How to Diagnose

### Check group membership:

```bash
# On the target host
groups $USER
id $USER

# Ubuntu: should show "sudo"
# RHEL:   should show "wheel"
```

### Check sudoers configuration:

```bash
sudo cat /etc/sudoers | grep -E "(sudo|wheel|NOPASSWD)"
ls -la /etc/sudoers.d/
```

### Test passwordless sudo:

```bash
# Returns success if no password needed
sudo -n true && echo "Can sudo without password" || echo "Password required"
```

### Test from Ansible:

```bash
# Quick check if become works
ansible all -m ping -b
ansible all -m command -a "whoami" -b
```

## Solutions

### Option 1: Configure Passwordless Sudo on RHEL

The cleanest approach for automation — give the Ansible user passwordless sudo:

```bash
# On each RHEL/CentOS/Rocky host:

# Add user to wheel group (if not already)
sudo usermod -aG wheel ansible_user

# Create a sudoers drop-in file for passwordless access
echo "ansible_user ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ansible_user
sudo chmod 0440 /etc/sudoers.d/ansible_user

# Validate (always validate after editing sudoers!)
sudo visudo -cf /etc/sudoers.d/ansible_user
```

Or for a group:

```bash
# Allow entire wheel group passwordless sudo
echo "%wheel ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/wheel-nopasswd
sudo chmod 0440 /etc/sudoers.d/wheel-nopasswd
```

### Option 2: Provide the Sudo Password to Ansible

If you can't configure passwordless sudo (security policy), provide the password:

```bash
# Prompt for password at runtime
ansible-playbook playbook.yml --ask-become-pass

# Or -K shorthand
ansible-playbook playbook.yml -K

# Via extra vars (useful in CI/CD — password visible in process list!)
ansible-playbook playbook.yml -e "ansible_become_pass=the_sudo_password"
```

### Option 3: Store Password in Vault

For automation (CI/CD) where prompting isn't possible:

```bash
# Create vault-encrypted variable
ansible-vault encrypt_string 'the_sudo_password' --name 'ansible_become_pass'
```

```yaml
# group_vars/rhel_servers/vault.yml (encrypted)
ansible_become_pass: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...encrypted...
```

```bash
# Run with vault
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass
```

### Option 4: Per-Group Inventory Configuration

Handle mixed environments by setting `ansible_become_pass` only where needed:

```ini
[ubuntu_hosts]
ubuntu01 ansible_host=192.168.1.10
ubuntu02 ansible_host=192.168.1.11

[ubuntu_hosts:vars]
ansible_become=yes
# No become_pass needed — Ubuntu allows passwordless sudo

[rhel_hosts]
rhel01 ansible_host=192.168.1.20
rhel02 ansible_host=192.168.1.21

[rhel_hosts:vars]
ansible_become=yes
ansible_become_pass="{{ vault_sudo_password }}"
```

### Option 5: Bootstrap Playbook

Automate the RHEL sudo setup itself:

```yaml
---
- name: Configure passwordless sudo for Ansible user
  hosts: rhel_hosts
  become: yes
  # First run requires -K (--ask-become-pass)
  tasks:
    - name: Ensure user is in wheel group
      user:
        name: "{{ ansible_user }}"
        groups: wheel
        append: yes

    - name: Configure passwordless sudo
      copy:
        content: "{{ ansible_user }} ALL=(ALL) NOPASSWD:ALL\n"
        dest: "/etc/sudoers.d/{{ ansible_user }}"
        mode: "0440"
        validate: "visudo -cf %s"
```

```bash
# First run — needs password
ansible-playbook bootstrap-sudo.yml -K

# Subsequent runs — no password needed
ansible-playbook site.yml
```

## Cloud Image Defaults

| Cloud Image | Default User | Sudo Config |
|-------------|-------------|-------------|
| Ubuntu (AWS/Azure/GCP) | `ubuntu` | NOPASSWD via `/etc/sudoers.d/90-cloud-init-users` |
| Amazon Linux 2/2023 | `ec2-user` | NOPASSWD via `/etc/sudoers.d/90-cloud-init-users` |
| RHEL (AWS) | `ec2-user` | NOPASSWD (cloud image pre-configured) |
| CentOS Stream (generic) | `centos` or `cloud-user` | May require password |
| Rocky Linux (generic) | `rocky` | May require password |
| Debian (AWS) | `admin` | NOPASSWD via cloud-init |

> **Note:** Cloud marketplace images (AWS, Azure, GCP) usually have NOPASSWD configured for the default user. The problem typically occurs with manually installed or on-premise systems.

## Security Considerations

| Approach | Security | Convenience | Best For |
|----------|----------|-------------|----------|
| NOPASSWD in sudoers | Lower (no password barrier) | Highest | Automation accounts, CI/CD |
| Vault-stored password | Good (encrypted at rest) | High | Mixed environments |
| `--ask-become-pass` (-K) | Good (never stored) | Lower (manual) | One-off runs, testing |
| SSH agent + password | Good | Medium | Interactive sessions |

**Recommendations:**
- **Dedicated automation user** — create a user specifically for Ansible (e.g., `ansible`) with NOPASSWD, separate from human accounts
- **Limit NOPASSWD scope** — if possible, restrict to specific commands: `ansible ALL=(ALL) NOPASSWD: /usr/bin/systemctl, /usr/bin/apt-get`
- **Audit trail** — `sudo` logs all commands to `/var/log/secure` (RHEL) or `/var/log/auth.log` (Ubuntu) regardless of NOPASSWD
- **Don't disable requiretty** — older RHEL required `Defaults !requiretty` in sudoers for Ansible; modern versions don't need this

## Troubleshooting

### "sudo: a terminal is required to read the password"

```bash
# On RHEL 6/7 — requiretty is enabled by default
# Fix: add to /etc/sudoers or sudoers.d file
Defaults:ansible_user !requiretty
```

### "Missing sudo password" but user IS in wheel

```bash
# Check if the sudoers file uses password
sudo grep wheel /etc/sudoers
# If it shows: %wheel ALL=(ALL) ALL
# That requires a password. Change to:
# %wheel ALL=(ALL) NOPASSWD: ALL
# Or add a drop-in file
```

### Password works manually but not via Ansible

```bash
# Ansible needs the become_pass in the right place
# Check your inventory/group_vars:
ansible-inventory --host rhel01 | grep become

# Test explicitly:
ansible rhel01 -m ping -b -K
```

### "sudo: unable to resolve host"

```bash
# Add hostname to /etc/hosts on the target
echo "127.0.0.1 $(hostname)" >> /etc/hosts
```

## Quick Reference

```bash
# Test sudo access from Ansible
ansible all -m command -a "whoami" -b
ansible all -m shell -a "sudo -n true && echo OK || echo NEED_PASS" 

# Prompt for password
ansible-playbook playbook.yml -K

# Bootstrap RHEL for passwordless sudo
ansible rhel_hosts -m shell -a 'echo "ansible ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ansible' -b -K

# Verify sudoers file is valid
ansible all -m command -a "visudo -cf /etc/sudoers" -b
```
