# Setting the LXC Root Password with cloud-init Userdata

Instead of resetting a container's `root` password [after creation](articles/proxmox-lxc-change-root-password.md), you can set it during initialization with cloud-init userdata. The password is applied automatically the first time the container boots, which keeps the process consistent across single or bulk deployments and integrates cleanly with the Terraform lifecycle.

> This requires an **LXC template that supports cloud-init** (e.g. the Ubuntu standard templates). Older or minimal templates may need cloud-init installed first — see [Template compatibility](#template-compatibility).

## Why Userdata Instead of Post-Creation

| Aspect | Userdata method | Post-creation method |
|--------|-----------------|----------------------|
| Timing | During creation | After creation |
| Automation | Fully automated | Requires extra steps |
| Security | No exposed post-commands | Commands run against a live container |
| Flexibility | High — users, packages, files, commands | Medium |
| Reliability | Integrated into first boot | Depends on host reachability |
| Debugging | cloud-init logs | Command output |
| Template dependency | Needs cloud-init support | Works with any template |

## Configuration Methods

The examples below assume a Terraform module (using the `bpg/proxmox` provider) that renders a cloud-init userdata file from variables. Two modes are shown: a simple one and an advanced one.

### Method 1: Basic Userdata (recommended)

```hcl
# terraform.tfvars
enable_userdata        = true
userdata_advanced_mode = false
userdata_root_password = "MySecurePassword123!"
```

Sets the password via the cloud-init `users` section, lets cloud-init hash it, wires in SSH keys, and applies basic system config.

### Method 2: Advanced Userdata

```hcl
# terraform.tfvars
enable_userdata        = true
userdata_advanced_mode = true
userdata_root_password = "MySecurePassword123!"
```

Adds redundant password-change methods, extra SSH hardening, custom file writes, and additional system configuration.

## Password Configuration Options

### Plain-text password (simplest)

```hcl
userdata_root_password = "MySecurePassword123!"
```

cloud-init hashes it automatically. Works in both modes.

### Pre-hashed password

Generate a SHA-512 hash and pass that instead of a plain-text value:

```bash
# mkpasswd (from the 'whois' package)
mkpasswd -m sha-512 "MySecurePassword123!"

# openssl
openssl passwd -6 "MySecurePassword123!"

# Python
python3 -c "import crypt; print(crypt.crypt('MySecurePassword123!', crypt.mksalt(crypt.METHOD_SHA512)))"
```

```hcl
userdata_root_password_hash = "$6$rounds=4096$salt$hashedpassword..."
```

### Force a change on first login

```hcl
userdata_password_expire = true
```

## Complete Configuration Examples

### Example 1: Simple password change

```hcl
# terraform.tfvars
container_hostname = "test-container"
proxmox_endpoint   = "https://proxmox.example.com:8006"
proxmox_username   = "root@pam"
proxmox_password   = "proxmox-password"
proxmox_node       = "pve"

# Enable userdata with a simple password change
enable_userdata        = true
userdata_root_password = "NewRootPassword123!"

# Optional SSH configuration
ssh_public_keys            = ["ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... your-key"]
userdata_enable_ssh        = true
userdata_ssh_password_auth = true
```

### Example 2: Advanced config with an extra user

```hcl
# terraform.tfvars
container_hostname = "advanced-container"
# ... other proxmox settings ...

enable_userdata        = true
userdata_advanced_mode = true

userdata_root_password  = "SuperSecureRootPassword123!"
userdata_password_expire = false

additional_users = [
  {
    name                = "admin"
    password            = "AdminPassword123!"
    sudo                = ["ALL=(ALL) NOPASSWD:ALL"]
    shell               = "/bin/bash"
    ssh_authorized_keys = ["ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... admin-key"]
  }
]

userdata_packages = ["htop", "curl", "git", "nano"]

userdata_custom_commands = [
  "echo 'Container configured with advanced userdata' > /var/log/setup.log",
  "systemctl enable ssh",
  "ufw --force enable"
]

userdata_write_files = [
  {
    path        = "/etc/motd"
    content     = "Welcome to Advanced Container!\nRoot password set via userdata.\n"
    permissions = "0644"
  }
]
```

## Generated cloud-init Output

### Basic mode

```yaml
#cloud-config
hostname: my-container
manage_etc_hosts: true
package_update: true
package_upgrade: false

users:
  - name: root
    password: MySecurePassword123!
    lock_passwd: false
    sudo:
      - "ALL=(ALL) NOPASSWD:ALL"
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...

ssh_pwauth: true
disable_root: false

packages: []
timezone: UTC

runcmd:
  - echo 'root:MySecurePassword123!' | chpasswd
  - systemctl enable ssh
  - systemctl start ssh

final_message: "Container my-container setup completed successfully!"
```

### Advanced mode

```yaml
#cloud-config
hostname: my-container

users:
  - name: root
    password: MySecurePassword123!
    passwd: $6$rounds=4096$salt$hash...   # if a hash is provided
    lock_passwd: false
    sudo:
      - "ALL=(ALL) NOPASSWD:ALL"
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...

runcmd:
  # Redundant password-change methods
  - echo 'root:MySecurePassword123!' | chpasswd
  - echo -e 'MySecurePassword123!\nMySecurePassword123!' | passwd root
  - chage -d 0 root   # if password expiration is enabled
  - systemctl enable ssh
  - systemctl start ssh
  - sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
  - systemctl restart ssh
  - echo 'Root password changed via cloud-init userdata' >> /var/log/password-changes.log

write_files:
  - path: /etc/security/pwquality.conf.d/custom.conf
    content: |
      minlen = 12
      minclass = 3
    owner: root:root
    permissions: '0644'

final_message: "Container my-container configured with userdata!"
```

## Deployment

### Basic deployment

```bash
cp terraform.tfvars.userdata.example terraform.tfvars
nano terraform.tfvars

terraform init
terraform plan
terraform apply
```

### Password-only update

```bash
# Pass the value on the CLI (mark the variable sensitive in your module)
terraform apply -var="userdata_root_password=NewPassword123!"
```

### Generate a secure random password

```bash
NEW_PASS=$(openssl rand -base64 16 | tr -d "=+/" | cut -c1-14)
echo "Generated password: $NEW_PASS"
terraform apply -var="userdata_root_password=$NEW_PASS"
```

## Security Best Practices

### Password strength

```hcl
userdata_root_password = "MyStr0ng!P@ssw0rd#2024"
# At least 12 characters, mixed case, numbers, and symbols; not dictionary words.
```

### Prefer SSH keys

```hcl
ssh_public_keys = [
  "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... your-key",
  "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... backup-key"
]

# Optionally disable password auth entirely
userdata_ssh_password_auth = false
```

### Force expiration on first login

```hcl
userdata_password_expire = true
```

### Audit logging

```hcl
userdata_custom_commands = [
  "echo \"$(date): Root password set via userdata on $(hostname)\" >> /var/log/password-audit.log",
  "logger -t userdata 'Root password configured via cloud-init'",
  "chmod 600 /var/log/password-audit.log"
]
```

> Keep secrets out of committed `.tfvars`. Mark password variables as `sensitive = true`, and prefer environment variables (`TF_VAR_userdata_root_password`) or a secrets store over plain files.

## Troubleshooting

### Userdata not applied

```bash
# Is cloud-init present in the template?
pct exec <container_id> -- which cloud-init

# Inspect the logs
pct exec <container_id> -- cat /var/log/cloud-init.log
pct exec <container_id> -- cat /var/log/cloud-init-output.log

# Re-run cloud-init from scratch
pct exec <container_id> -- cloud-init clean
pct exec <container_id> -- cloud-init init
```

### Password not changed

```bash
# Verify the rendered userdata on the host
cat /var/lib/vz/snippets/<hostname>-userdata.yaml

# Confirm chpasswd ran
pct exec <container_id> -- grep "chpasswd" /var/log/cloud-init-output.log

# Set it manually as a fallback
pct exec <container_id> -- passwd root
```

### SSH access issues

```bash
pct exec <container_id> -- systemctl status ssh
pct exec <container_id> -- grep "PasswordAuthentication" /etc/ssh/sshd_config

pct exec <container_id> -- sed -i 's/#PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
pct exec <container_id> -- systemctl restart ssh
```

### Template compatibility

```bash
# Prefer a cloud-init-enabled template
template_file_id = "local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst"
```

```hcl
# For older templates, install cloud-init via userdata
userdata_packages = ["cloud-init"]
userdata_custom_commands = [
  "systemctl enable cloud-init",
  "systemctl start cloud-init"
]
```

## Validation

```bash
# Did the password change show up in the log?
pct exec <container_id> -- grep "password" /var/log/cloud-init-output.log

# Test login
ssh root@<container_ip>

# Confirm the userdata file was uploaded
ls -la /var/lib/vz/snippets/*userdata*

# Overall cloud-init status
pct exec <container_id> -- cloud-init status
```

## When to Use This

The userdata approach is the preferred choice when your LXC templates support cloud-init — especially for:

- Production environments needing consistent, repeatable configuration
- Automated deployments with minimal manual steps
- Security-conscious setups that avoid running commands against live containers
- Cases where you configure users, packages, and files alongside the password

For containers built from templates without cloud-init, or for one-off resets on existing containers, use the [post-creation methods](articles/proxmox-lxc-change-root-password.md) instead.
