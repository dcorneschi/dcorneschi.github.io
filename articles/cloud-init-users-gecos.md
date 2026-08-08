# cloud-init: User Management and the gecos Field

cloud-init's `users` module creates and configures user accounts on first boot. The `gecos` field is one of the most misunderstood options — it's inherited from a 1970s mainframe system but still serves a practical purpose today.

## What gecos Is

The GECOS field (General Electric Comprehensive Operating System) is the fifth field in `/etc/passwd`. It holds human-readable information about the account — typically the user's full name, but historically also room number, phone numbers, and other contact info:

```
username:x:UID:GID:GECOS_FIELD:home_dir:shell
dcorneschi:x:1000:1000:Daniel Corneschi:/home/dcorneschi:/bin/bash
```

In cloud-init, the `gecos` key sets this field. If omitted, it defaults to an empty string.

## Basic User Configuration

```yaml
#cloud-config
users:
  - name: dcorneschi
    gecos: Daniel Corneschi
    shell: /bin/bash
    groups: [sudo, docker]
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: true
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... user@workstation
```

After boot, verify with:

```bash
$ getent passwd dcorneschi
dcorneschi:x:1000:1000:Daniel Corneschi:/home/dcorneschi:/bin/bash

$ finger dcorneschi
Login: dcorneschi                Name: Daniel Corneschi
```

## All User Keys

| Key | Description |
|-----|-------------|
| `name` | Username (required) |
| `gecos` | Full name / account description (the GECOS field in `/etc/passwd`) |
| `primary_group` | Primary group (defaults to username) |
| `groups` | Supplementary groups |
| `shell` | Login shell |
| `homedir` | Home directory path (defaults to `/home/<name>`) |
| `no_create_home` | Skip home directory creation |
| `system` | Create a system account (low UID, no home) |
| `sudo` | Sudo rule (string or list) |
| `lock_passwd` | Lock password login (key-only access) |
| `passwd` | Hashed password (`$6$...`) |
| `plain_text_passwd` | Plaintext password (not recommended) |
| `hashed_passwd` | Same as `passwd` (alternative key name) |
| `ssh_authorized_keys` | List of SSH public keys |
| `ssh_import_id` | Import keys from GitHub (`gh:user`) or Launchpad (`lp:user`) |
| `ssh_redirect_user` | Redirect SSH connections to default user |
| `inactive` | Days after password expiry before account is disabled |
| `expiredate` | Account expiration date (`YYYY-MM-DD`) |
| `selinux_user` | SELinux user mapping |
| `doas` | doas/opendoas rules (list) |
| `uid` | Specific UID to assign |

## The default User

Most cloud images define a default user (e.g., `ubuntu`, `ec2-user`, `centos`). When you specify a `users` list, you must include `default` to preserve it:

```yaml
#cloud-config
users:
  - default
  - name: dcorneschi
    gecos: Daniel Corneschi
    groups: [sudo]
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: true
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... user@workstation
```

If you omit `default`, the distribution's default user is **not** created, and cloud-provided SSH keys won't be installed for it.

### Override the Default User

```yaml
#cloud-config
# Change the default user's name and remove sudo
user: {name: myadmin, sudo: null}
ssh_import_id: [gh:myuser]
```

### Disable Default User Entirely

```yaml
#cloud-config
users: []
```

## gecos Subfields

Traditionally, the GECOS field holds comma-separated subfields:

```
Full Name,Room Number,Work Phone,Home Phone,Other
```

Example in `/etc/passwd`:

```
jhill:x:1003:1003:Jill Hill,828,212-555-0000,212-555-3456,jhill@example.com:/home/jhill:/bin/bash
```

In cloud-init, you can use the full comma-separated format:

```yaml
users:
  - name: dcorneschi
    gecos: "Daniel Corneschi,Room 42,+1-555-0100,,"
```

In practice, most people just put the full name. Tools like `finger`, `chfn`, and some mail clients read subfields, but for cloud instances the full name is all that matters.

## Common Patterns

### Developer Account with GitHub Keys

```yaml
#cloud-config
users:
  - default
  - name: dcorneschi
    gecos: Daniel Corneschi
    groups: [sudo, docker]
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: true
    ssh_import_id:
      - gh:dcorneschi
```

### Service Account (No Login)

```yaml
users:
  - name: appservice
    gecos: Application Service Account
    system: true
    shell: /usr/sbin/nologin
    no_create_home: true
```

### Multiple Users with Different Access

```yaml
#cloud-config
users:
  - default
  - name: dcorneschi
    gecos: Daniel Corneschi
    groups: [sudo, docker]
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: true
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... daniel@workstation

  - name: readonly
    gecos: Read-Only Operator
    shell: /bin/bash
    lock_passwd: true
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... operator@monitoring

  - name: deploy
    gecos: Deployment Service
    system: true
    shell: /bin/bash
    groups: [docker]
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... ci@pipeline
```

### User with Password (Expiry on First Login)

```bash
# Generate password hash locally
mkpasswd --method=sha-512 --rounds=4096
```

```yaml
users:
  - name: tempuser
    gecos: Temporary Access User
    shell: /bin/bash
    groups: [sudo]
    lock_passwd: false
    passwd: $6$rounds=4096$saltsalt$hash...
    chpasswd:
      expire: true  # Force password change on first login
```

### User with Specific UID and Home Directory

```yaml
users:
  - name: dcorneschi
    gecos: Daniel Corneschi
    uid: 1500
    homedir: /opt/dcorneschi
    shell: /bin/bash
```

## Module Execution Order

The `cc_users_groups` module runs during the **Network (init) stage**, which means:

- Users are created **after** `bootcmd`
- Users are created **after** non-deferred `write_files`
- Users exist **before** `runcmd` executes
- SSH keys are deployed **after** users are created

If you need to write files to a user's home directory, use `defer: true` on the `write_files` entry — or better yet, use `runcmd` to ensure the home directory exists:

```yaml
#cloud-config
users:
  - name: dcorneschi
    gecos: Daniel Corneschi
    shell: /bin/bash

write_files:
  # This can fail because /home/dcorneschi may not exist yet when write_files runs
  # Use defer: true to push it to the final stage
  - path: /home/dcorneschi/.bashrc.d/aliases.sh
    defer: true
    owner: dcorneschi:dcorneschi
    permissions: '0644'
    content: |
      alias ll='ls -la'
      alias gs='git status'
```

## Options That Work on Existing Users

Most user options only apply when creating a new user. These keys can be applied even if the user already exists:

- `plain_text_passwd` / `hashed_passwd`
- `lock_passwd`
- `sudo`
- `ssh_authorized_keys`
- `ssh_redirect_user`

This is important for cloud images that ship with a pre-created user — you can still manage their SSH keys and sudo access via cloud-init.

## Debugging User Creation

```bash
# Check if the module ran
grep "cc_users_groups" /var/log/cloud-init.log

# Verify user exists
id dcorneschi
getent passwd dcorneschi

# Check groups
groups dcorneschi

# Check SSH keys
cat /home/dcorneschi/.ssh/authorized_keys

# Check sudo access
sudo -l -U dcorneschi

# Check the gecos field
getent passwd dcorneschi | cut -d: -f5
```

## gecos vs No gecos

| With gecos | Without gecos |
|-----------|---------------|
| `finger` shows full name | `finger` shows empty name |
| `w` command shows name | `w` shows blank |
| Git commits can use it (`user.name`) | Must be set manually |
| Audit logs are more readable | UIDs only in logs |
| Mail clients show real name | Shows username only |

Setting `gecos` costs nothing and makes system administration easier — always include it.

## Direct Format vs Nested Format

### Direct format: `users:` (Standalone cloud-config)

```yaml
#cloud-config
users:
  - name: dcorneschi
    gecos: Daniel Corneschi
```

This is **direct cloud-config syntax** used when:
- Writing a standalone cloud-config file
- The entire file/block IS the user-data
- Used in `.yaml` or `.yml` files for cloud-init
- Common in configuration files or templates

### Nested format: `user-data:` containing `users:` (Wrapped structure)

```yaml
user-data:
  users:
    - name: dcorneschi
      gecos: Daniel Corneschi
```

This is a **nested structure** typically used in:
- **Terraform configurations** (like `cloud_init` data source)
- **Ansible playbooks** with cloud-init modules
- **Configuration management tools** that wrap cloud-init
- **API calls** where user-data is a parameter

### Key Differences

| Aspect | Direct `users:` | Nested `user-data: users:` |
|--------|-----------------|---------------------------|
| **Context** | Standalone cloud-config | Part of larger configuration |
| **File type** | `.yaml` cloud-config file | Terraform/Ansible/API config |
| **Usage** | Direct cloud-init consumption | Wrapped by other tools |
| **Header** | Needs `#cloud-config` | Handled by parent tool |

### Which to Use

- **Direct format** (`users:`): When writing cloud-config files directly
- **Nested format** (`user-data: users:`): When using infrastructure-as-code tools that generate the cloud-config

Both result in the same user being created on the target system — the difference is just in how the configuration is structured and delivered to cloud-init.

### Terraform Example (Nested Format)

```hcl
data "cloudinit_config" "example" {
  gzip = false
  base64_encode = false

  part {
    content_type = "text/cloud-config"
    content = yamlencode({
      users = [
        {
          name  = "dcorneschi"
          gecos = "Daniel Corneschi"
        }
      ]
    })
  }
}
```

## Combined Cloud-Config Examples

### Multiple Directives with User Creation

```yaml
#cloud-config
packages:
  - nginx
  - git
  - docker.io

users:
  - name: dcorneschi
    gecos: Daniel Corneschi
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    home: /home/dcorneschi
    groups: docker,sudo,admin
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2E...
    lock_passwd: false
    passwd: $6$rounds=4096$...
    create_home: true

runcmd:
  - systemctl start nginx
  - systemctl enable nginx
```

### File Creation with User and Service Setup

```yaml
#cloud-config
write_files:
  - path: /etc/nginx/sites-available/mysite
    content: |
      server {
          listen 80;
          server_name example.com;
          root /var/www/html;
      }
    permissions: '0644'
    owner: dcorneschi:dcorneschi

users:
  - name: dcorneschi
    gecos: Daniel Corneschi

runcmd:
  - ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
  - systemctl reload nginx
```
