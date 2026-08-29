# Changing the Root Password in Proxmox LXC Containers

Several ways to (re)set the `root` password of an LXC container after it has been created — from a quick one-liner on the Proxmox host to fully automated batch and Terraform workflows. Most methods use `pct exec`, which runs a command inside the container from the host without needing SSH or the current password.

> Run these commands on the **Proxmox VE host** as `root` (or with `sudo`). The container must be **running** for `pct exec` to work.

## Method 1: Direct Commands on the Proxmox Host

The simplest approach. `pct exec <id> -- <cmd>` executes a command inside the container.

```bash
# Interactive — prompts for the new password twice
pct exec <container_id> -- passwd root

# Non-interactive — pipe the new credentials into chpasswd
echo "root:new_password" | pct exec <container_id> -- chpasswd

# Feed both prompts to passwd via a here-string
pct exec <container_id> -- bash -c "echo -e 'new_password\nnew_password' | passwd root"
```

The `chpasswd` form is the cleanest for scripting — it reads `user:password` pairs on stdin and needs no interaction.

## Method 2: SSH Access (if enabled)

If the container has SSH running and reachable, change the password from inside:

```bash
# Interactive
ssh root@<container_ip>
passwd

# One-liner (requires an existing way to authenticate, e.g. an SSH key)
ssh root@<container_ip> "echo 'root:new_password' | chpasswd"
```

This only works if you can already log in. If you have lost the password entirely, use Method 1 from the host instead.

## Practical Examples

### Container 100 with a specific password

```bash
# Interactive
pct exec 100 -- passwd root

# Non-interactive
echo "root:MySecurePassword123!" | pct exec 100 -- chpasswd
```

### Multiple containers in a loop

```bash
for container_id in 100 101 102; do
    echo "Changing password for container $container_id"
    echo "root:MySecurePassword123!" | pct exec "$container_id" -- chpasswd
done
```

### Generate and set a random password

```bash
NEW_PASSWORD=$(openssl rand -base64 16)
echo "Generated password: $NEW_PASSWORD"
echo "root:$NEW_PASSWORD" | pct exec 100 -- chpasswd
```

## Advanced Examples

### Automate the interactive prompt with expect

Useful when a tool insists on the interactive `passwd` prompts:

```bash
#!/bin/bash
CONTAINER_ID=100
NEW_PASSWORD="MyNewPassword123!"

expect -c "
spawn pct exec $CONTAINER_ID -- passwd root
expect \"New password:\"
send \"$NEW_PASSWORD\r\"
expect \"Retype new password:\"
send \"$NEW_PASSWORD\r\"
expect eof
"
```

### Batch change with logging

```bash
#!/bin/bash
LOGFILE="/var/log/lxc_password_changes.log"
NEW_PASSWORD="MyNewPassword123!"

change_container_password() {
    local container_id=$1
    local password=$2

    echo "$(date): Changing password for container $container_id" >> "$LOGFILE"

    if echo "root:$password" | pct exec "$container_id" -- chpasswd; then
        echo "$(date): SUCCESS - container $container_id" >> "$LOGFILE"
        return 0
    else
        echo "$(date): FAILED - container $container_id" >> "$LOGFILE"
        return 1
    fi
}

# Only touch containers that exist and are running
for i in {100..105}; do
    if pct status "$i" &>/dev/null; then
        change_container_password "$i" "$NEW_PASSWORD"
    fi
done
```

## Security Considerations

### Generate strong passwords

```bash
# openssl, stripped of ambiguous symbols
openssl rand -base64 32 | tr -d "=+/" | cut -c1-25

# pwgen, secure mode
pwgen -s 16 1

# Python secrets module with a defined character set
python3 -c "import secrets, string; chars = string.ascii_letters + string.digits + '!@#\$%^&*'; print(''.join(secrets.choice(chars) for _ in range(16)))"
```

### Store passwords securely

```bash
# With the standard 'pass' password manager
NEW_PASSWORD=$(openssl rand -base64 16)
echo "$NEW_PASSWORD" | pass insert -m "proxmox/lxc-$CONTAINER_ID-root"

# Via an environment variable, unset immediately after use
export LXC_ROOT_PASSWORD="your_secure_password"
echo "root:$LXC_ROOT_PASSWORD" | pct exec 100 -- chpasswd
unset LXC_ROOT_PASSWORD
```

Avoid leaving passwords in your shell history. Prefix commands with a space (with `HISTCONTROL=ignorespace`) or read them from a variable/file rather than typing them inline.

### Keep an audit trail

```bash
echo "$(date): Password changed for container $CONTAINER_ID by $(whoami)" | \
    logger -t lxc-password-change
```

## Troubleshooting

### Container not running

`pct exec` fails if the container is stopped.

```bash
pct status <container_id>
pct start <container_id>

# Wait until it reports "running"
while [[ $(pct status <container_id> | awk '{print $2}') != "running" ]]; do
    sleep 2
done
```

### Permission issues

```bash
# pct requires root on the host
sudo pct exec <container_id> -- passwd root

# Or become root first
sudo -i
pct exec <container_id> -- passwd root
```

### Network / SSH access issues

```bash
# Find the container's IP
pct exec <container_id> -- ip addr show | grep inet

# Test SSH reachability
nc -zv <container_ip> 22

# Enable and start SSH inside the container
pct exec <container_id> -- systemctl enable --now ssh
```

## Integration with Terraform

When containers are managed by the `bpg/proxmox` provider, use a `null_resource` to (re)set the password after creation. It runs on the Proxmox host via `local-exec`.

```hcl
resource "null_resource" "change_root_password" {
  depends_on = [proxmox_virtual_environment_container.lxc_container]

  # Re-run when the container is recreated or the password variable changes
  triggers = {
    container_id  = proxmox_virtual_environment_container.lxc_container.vm_id
    password_hash = var.root_password != "" ? md5(var.root_password) : ""
  }

  provisioner "local-exec" {
    command = var.root_password != "" ? "echo 'root:${var.root_password}' | pct exec ${proxmox_virtual_environment_container.lxc_container.vm_id} -- chpasswd" : "echo 'No password change requested'"
    working_dir = "/tmp"
  }

  # Alternative: run inside the container over SSH
  # provisioner "remote-exec" {
  #   connection {
  #     type        = "ssh"
  #     user        = "root"
  #     private_key = file(var.ssh_private_key_path)
  #     host        = proxmox_virtual_environment_container.lxc_container.initialization[0].ip_config[0].ipv4[0].address
  #   }
  #   inline = ["echo 'root:${var.root_password}' | chpasswd"]
  # }
}
```

> Mark `var.root_password` as `sensitive = true` so Terraform does not print it in plan/apply output. Note that `local-exec` only works when Terraform runs on the Proxmox host itself.

## Reusable Script

A small wrapper (`change_lxc_password.sh`) keeps day-to-day use tidy:

```bash
# Interactive prompt for the password
./change_lxc_password.sh 100

# Set a specific password
./change_lxc_password.sh 100 "MyNewPassword123!"

# Generate a secure random password
./change_lxc_password.sh 100 --generate

# Interactive mode with prompts
./change_lxc_password.sh 100 --interactive

# Help
./change_lxc_password.sh --help
```

## Best Practices

1. Use strong passwords — at least 12 characters, mixed case, numbers, and symbols.
2. Log every password change for audit purposes.
3. Prefer SSH keys over passwords wherever possible.
4. Store passwords in a manager or encrypted store, never in plain files or shell history.
5. Verify the change afterward (e.g. `pct exec <id> -- passwd -S root`).
6. Use configuration management (Ansible, Terraform) for consistency across many containers.
7. Implement a rotation policy for production environments.
