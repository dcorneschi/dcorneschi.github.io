# Cloud-Init Run Modes and Frequencies

Cloud-init modules run at different frequencies depending on whether they should execute once, on every boot, or per instance. Understanding these modes is key to writing reliable cloud-init configurations.

## Module Frequencies

| Frequency | Runs When | Semaphore Location | Use Case |
|-----------|-----------|-------------------|----------|
| `once` / `PER_ONCE` | Only the very first time ever | `/var/lib/cloud/instance/sem/` | One-time setup that should never repeat |
| `instance` / `PER_INSTANCE` | Once per instance ID | `/var/lib/cloud/instance/sem/` | Setup that reruns if instance ID changes (new deployment) |
| `always` / `PER_ALWAYS` | Every boot | No semaphore (always runs) | Things that must run on every boot |

### How It Works

When a module runs, cloud-init creates a semaphore file:

```
/var/lib/cloud/instance/sem/config_<module_name>.<frequency>
```

On subsequent boots, cloud-init checks for this file. If it exists and the frequency condition is met, the module is skipped.

## Boot Stages

Cloud-init runs in four stages, each with its own set of modules:

| Stage | When | Config Key | Typical Frequency |
|-------|------|-----------|------------------|
| Network | Very early (before networking on some) | `cloud_init_modules` | per_instance |
| Config | After networking, before user login | `cloud_config_modules` | per_instance |
| Final | Late boot, after all services started | `cloud_final_modules` | per_instance |

```bash
# View the stages and their modules
cat /etc/cloud/cloud.cfg
```

## runcmd vs bootcmd

| Feature | `runcmd` | `bootcmd` |
|---------|----------|-----------|
| Frequency | **Once** (first boot only) | **Every boot** |
| Stage | Final (late boot) | Early boot (before other modules) |
| When to use | Package installs, one-time config | Early network config, mount points |
| Runs as | `/bin/sh` | `/bin/sh` |

### runcmd (Runs Once)

```yaml
#cloud-config
runcmd:
  - apt-get update && apt-get install -y nginx
  - systemctl enable nginx
  - echo "Setup complete" > /var/log/cloud-init-done.txt
```

### bootcmd (Runs Every Boot)

```yaml
#cloud-config
bootcmd:
  - echo 192.168.1.130 us.archive.ubuntu.com >> /etc/hosts
  - sysctl -w vm.swappiness=10
  - echo "Boot at $(date)" >> /var/log/boot-times.log
```

## cloud-init-per Command

`cloud-init-per` wraps a command to control its execution frequency, primarily used inside `bootcmd`:

```bash
cloud-init-per <frequency> <name> <command> [args...]
```

| Frequency | Meaning |
|-----------|---------|
| `once` | Run only the first time (creates semaphore) |
| `always` | Run on every boot |
| `instance` | Run once per instance ID |

### Examples

```yaml
#cloud-config
bootcmd:
  # Format a disk only once (even though bootcmd runs every boot)
  - [cloud-init-per, once, mymkfs, mkfs, /dev/vdb]

  # Run only once per instance
  - [cloud-init-per, instance, setup-swap, swapon, /dev/vdc]

  # This runs every boot (same as not using cloud-init-per)
  - [cloud-init-per, always, log-boot, sh, -c, 'echo "$(date)" >> /var/log/boots.log']
```

The semaphore files for `cloud-init-per` are stored at:

```
/var/lib/cloud/instance/sem/config_bootcmd_<name>.<frequency>
```

## Changing Module Frequency

### Override in /etc/cloud/cloud.cfg

Change a module from its default frequency by converting the string entry to a list:

```yaml
# /etc/cloud/cloud.cfg

# Default (module name as string uses its built-in default frequency)
cloud_final_modules:
  - final_message
  - power_state_change
  - phone_home

# Override frequency (module name + frequency as list)
cloud_final_modules:
  - [final_message, always]
  - [phone_home, always]
  - power_state_change
```

### Valid Frequency Values

| Value | Meaning |
|-------|---------|
| `once` | Run once ever (survives reboots and instance ID changes) |
| `instance` | Run once per instance (re-runs if instance ID changes) |
| `always` | Run on every boot |

## Per-Boot Scripts Directory

Place scripts in specific directories to control their run frequency:

```bash
# Runs on EVERY boot
/var/lib/cloud/scripts/per-boot/

# Runs ONCE per instance
/var/lib/cloud/scripts/per-instance/

# Runs ONCE ever
/var/lib/cloud/scripts/per-once/
```

### Example

```bash
# Create a per-boot script
cat << 'EOF' > /var/lib/cloud/scripts/per-boot/update-motd.sh
#!/bin/bash
echo "Last boot: $(date)" > /etc/motd
EOF
chmod +x /var/lib/cloud/scripts/per-boot/update-motd.sh

# Create a per-instance script
cat << 'EOF' > /var/lib/cloud/scripts/per-instance/register-host.sh
#!/bin/bash
curl -X POST http://cmdb.internal/register -d "hostname=$(hostname)"
EOF
chmod +x /var/lib/cloud/scripts/per-instance/register-host.sh

# Create a per-once script
cat << 'EOF' > /var/lib/cloud/scripts/per-once/initial-setup.sh
#!/bin/bash
apt-get update && apt-get install -y monitoring-agent
EOF
chmod +x /var/lib/cloud/scripts/per-once/initial-setup.sh
```

## Instance Identity and Re-Runs

Cloud-init identifies an "instance" by its instance ID (e.g., `i-0123456789abcdef0` on AWS). The `per_instance` frequency re-runs when this ID changes.

```bash
# Check current instance ID
cloud-init query instance-id

# Instance data location
cat /var/lib/cloud/data/instance-id

# The symlink points to the current instance
ls -la /var/lib/cloud/instance
# /var/lib/cloud/instance -> /var/lib/cloud/instances/<instance-id>
```

### When Does Instance ID Change?

| Scenario | Instance ID Changes | per_instance Re-Runs |
|----------|:-------------------:|:--------------------:|
| Reboot | No | No |
| Stop/Start (EC2) | No | No |
| Create new VM from snapshot | Yes | Yes |
| Terminate and recreate | Yes | Yes |
| `cloud-init clean` | Treated as new | Yes |

## Semaphore Files

```bash
# View all semaphore files (what has run)
ls /var/lib/cloud/instance/sem/

# Example output:
# config_apt_configure.instance
# config_locale.once
# config_runcmd.once
# config_set_hostname.instance

# Delete a semaphore to force re-run
sudo rm /var/lib/cloud/instance/sem/config_runcmd.once

# Then re-run cloud-init
sudo cloud-init single --name runcmd --frequency once
```

## Re-Running Cloud-Init

### Re-Run a Single Module

```bash
# Re-run runcmd regardless of semaphore
sudo cloud-init single --name runcmd --frequency always

# Re-run with its configured frequency
sudo cloud-init single --name runcmd
```

### Re-Run All Modules

```bash
# Re-run all final modules
sudo cloud-init modules --mode final

# Re-run all config modules
sudo cloud-init modules --mode config

# Re-run all init modules
sudo cloud-init modules --mode init
```

### Full Clean and Re-Run (Treat as New Instance)

```bash
# Remove all cloud-init data (semaphores, logs, instance data)
sudo cloud-init clean

# Clean and reboot (re-runs everything as if first boot)
sudo cloud-init clean --reboot

# Clean logs only
sudo cloud-init clean --logs
```

## Packer Image Builds

When building images with Packer, the frequency determines whether scripts run only during the build or also in production instances:

| Frequency | During Packer Build | First Boot in Production | Subsequent Reboots |
|-----------|:-------------------:|:------------------------:|:------------------:|
| `per-once` | Yes | No | No |
| `per-instance` | Yes | Yes | No |
| `per-boot` | Yes | Yes | Yes |

### Choosing the Right Frequency for Packer

| Goal | Frequency | Example |
|------|-----------|---------|
| Install packages into the image | `per-once` | `apt-get install nginx` |
| Bake config that shouldn't rerun | `per-once` | Hardening scripts, image prep |
| Setup that runs on each new deployment | `per-instance` | Generate host keys, register with CMDB |
| Tasks needed on every startup | `per-boot` | Update dynamic DNS, check mounts |

### Example: Packer + cloud-init

```yaml
#cloud-config

# Runs ONLY during Packer build (baked into image)
write_files:
  - path: /var/lib/cloud/scripts/per-once/install-packages.sh
    permissions: '0755'
    content: |
      #!/bin/bash
      apt-get update && apt-get install -y nginx monitoring-agent

# Runs on each new instance launched from the image
  - path: /var/lib/cloud/scripts/per-instance/register-host.sh
    permissions: '0755'
    content: |
      #!/bin/bash
      curl -X POST http://cmdb.internal/register -d "hostname=$(hostname)"

# Runs on every boot (including Packer build)
  - path: /var/lib/cloud/scripts/per-boot/update-dns.sh
    permissions: '0755'
    content: |
      #!/bin/bash
      /usr/local/bin/update-dynamic-dns.sh
```

> **Important:** After a Packer build, run `cloud-init clean` before creating the image. This removes semaphores so that `per-instance` scripts will run on the first boot of production instances.

## Practical Examples

### Run a Script on Every Boot

```yaml
#cloud-config
bootcmd:
  - echo "$(date) - system booted" >> /var/log/boot-times.log

# Or use per-boot scripts directory
write_files:
  - path: /var/lib/cloud/scripts/per-boot/log-boot.sh
    permissions: '0755'
    content: |
      #!/bin/bash
      echo "$(date) - system booted" >> /var/log/boot-times.log
```

### One-Time Package Install (Never Repeat)

```yaml
#cloud-config
runcmd:
  - apt-get update
  - apt-get install -y nginx certbot python3-certbot-nginx
  - certbot --nginx -d example.com --non-interactive --agree-tos -m admin@example.com
```

### Format Disk Once, Mount Every Boot

```yaml
#cloud-config
bootcmd:
  # Format only once (even though bootcmd runs every boot)
  - [cloud-init-per, once, format-data, mkfs.ext4, /dev/nvme1n1]
  # Mount every boot
  - mkdir -p /data
  - mount /dev/nvme1n1 /data

# Or better — use mounts module (handles fstab)
mounts:
  - [/dev/nvme1n1, /data, ext4, "defaults,nofail", "0", "2"]
```

### Register with Configuration Management on First Boot Only

```yaml
#cloud-config
runcmd:
  - curl -L https://chef.io/install.sh | bash
  - chef-client -o 'role[webserver]'

# Or per-instance (re-register if instance recreated)
write_files:
  - path: /var/lib/cloud/scripts/per-instance/register-chef.sh
    permissions: '0755'
    content: |
      #!/bin/bash
      chef-client -o 'role[webserver]'
```

## Default Frequencies by Module

| Module | Default Frequency |
|--------|------------------|
| `runcmd` | once |
| `bootcmd` | always |
| `package_update_upgrade_install` | once |
| `set_hostname` | instance |
| `update_hostname` | always |
| `write_files` | once |
| `users_groups` | instance |
| `ssh` | instance |
| `locale` | once |
| `timezone` | once |
| `ntp` | once |
| `final_message` | always |
| `phone_home` | once |
| `power_state_change` | once |
| `apt_configure` | once |
| `disk_setup` | once |
| `mounts` | once |

## Troubleshooting

### Module Didn't Run on Reboot

```bash
# Check if semaphore exists
ls /var/lib/cloud/instance/sem/ | grep <module_name>

# If semaphore exists and frequency is "once", it won't re-run
# Remove the semaphore to force re-run
sudo rm /var/lib/cloud/instance/sem/config_<module_name>.<frequency>

# Or change the frequency to "always" in /etc/cloud/cloud.cfg
```

### Module Ran When It Shouldn't Have

```bash
# Check if instance ID changed
cat /var/lib/cloud/data/instance-id

# per_instance modules re-run when instance ID changes
# If this is unexpected, check your cloud provider's behavior
```

### Verify What Ran

```bash
# Cloud-init log shows all module executions
sudo cat /var/log/cloud-init.log | grep "running"

# Or check output log
sudo cat /var/log/cloud-init-output.log
```

## Quick Reference

| Goal | Method |
|------|--------|
| Run once (first boot) | `runcmd` or scripts in `/var/lib/cloud/scripts/per-once/` |
| Run every boot | `bootcmd` or scripts in `/var/lib/cloud/scripts/per-boot/` |
| Run once per instance | Scripts in `/var/lib/cloud/scripts/per-instance/` |
| Run bootcmd once | `[cloud-init-per, once, name, command]` |
| Override module frequency | `[module_name, always]` in cloud.cfg |
| Force re-run a module | `sudo cloud-init single --name <module> --frequency always` |
| Reset everything | `sudo cloud-init clean --reboot` |
| Check semaphores | `ls /var/lib/cloud/instance/sem/` |
