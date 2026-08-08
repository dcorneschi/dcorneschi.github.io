# cloud-init: bootcmd vs runcmd

## Overview

cloud-init provides two modules for running commands during instance initialization: **bootcmd** and **runcmd**. They look similar in syntax but execute at completely different stages of the boot process, have different frequency behaviors, and serve different purposes. Confusing them is a common source of "it worked once but not after reboot" or "my filesystem wasn't ready yet" bugs.

## cloud-init Boot Stages

To understand when `bootcmd` and `runcmd` execute, you need to know the cloud-init stage pipeline:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        cloud-init Boot Stages                               │
├──────────────┬──────────────────────────────────────────────────────────────┤
│ Generator    │ Decides if cloud-init should run at all                      │
├──────────────┼──────────────────────────────────────────────────────────────┤
│ Local        │ Network NOT available. Applies networking config.            │
│              │ Runs: ds-identify, local datasource                          │
├──────────────┼──────────────────────────────────────────────────────────────┤
│ Network      │ Network IS available. Fetches metadata/userdata.             │
│ (init)       │ ★ bootcmd runs here (cc_bootcmd module)                      │
├──────────────┼──────────────────────────────────────────────────────────────┤
│ Config       │ Configures the instance (ssh keys, ntp, locale, etc.)        │
│              │ ★ runcmd script written here (cc_runcmd module)              │
├──────────────┼──────────────────────────────────────────────────────────────┤
│ Final        │ User scripts, package installs, final setup.                 │
│              │ ★ runcmd script executed here (via scripts_user)             │
├──────────────┼──────────────────────────────────────────────────────────────┤
│ (boot done)  │ cloud-init.target reached, login prompt available            │
└──────────────┴──────────────────────────────────────────────────────────────┘
```

The systemd services that drive these stages:

```
cloud-init-local.service     → Local stage
cloud-init-network.service   → Network (init) stage    ← bootcmd
cloud-config.service         → Config stage            ← runcmd (script written here)
cloud-final.service          → Final stage             ← runcmd (script executed here via scripts_user)
```

> **Note:** On older distributions (Ubuntu 22.04 and earlier), the Network stage service is named `cloud-init.service` rather than `cloud-init-network.service`. The rename happened to reduce confusion between the service name and the stage name.

## bootcmd

### When It Runs

`bootcmd` executes during the **Network (init) stage** — early in the boot process. At this point:

- Network is available (interfaces are up, DHCP complete)
- The datasource has been read (metadata, userdata parsed)
- Filesystems declared in `/etc/fstab` may **not** be mounted yet
- Package managers are **not** yet configured (no `apt_update` has run)
- Users declared in `users:` may **not** exist yet
- SSH keys are **not** yet deployed
- `write_files` with `defer: true` have **not** run yet

### Frequency: Every Boot

By default, `bootcmd` runs on **every boot**, not just the first. This is by design — it's meant for commands that need to execute early on each startup, similar to rc.local-style scripts.

The module frequency is `always` by default. While it's technically possible to override this in `/etc/cloud/cloud.cfg` by changing the module entry (e.g., `[bootcmd, once]`), doing so would defeat the module's intended purpose.

### Syntax

Each entry in `bootcmd` can be either a string (passed to `/bin/sh -c`) or a list (executed directly without shell interpretation):

```yaml
#cloud-config
bootcmd:
  # String form — interpreted by /bin/sh
  - echo "Boot started at $(date)" >> /var/log/boot-times.log

  # List form — executed directly (no shell expansion)
  - [cloud-init-per, once, fix-swap, sh, -c, "fallocate -l 1G /swapfile && mkswap /swapfile && swapon /swapfile"]

  # Modify kernel parameters before services start
  - sysctl -w vm.swappiness=10

  # Create a directory needed by a service that starts later
  - mkdir -p /run/myapp
```

### cloud-init-per: Controlling Frequency Within bootcmd

Since `bootcmd` runs every boot, cloud-init provides the `cloud-init-per` helper to control how often individual commands within it execute:

```yaml
bootcmd:
  # Run only once, ever (survives reboots)
  - [cloud-init-per, once, setup-raid, mdadm, --assemble, /dev/md0, /dev/sdb1, /dev/sdc1]

  # Run once per instance (new instance ID = runs again)
  - [cloud-init-per, instance, format-ephemeral, mkfs.ext4, /dev/nvme1n1]

  # Run every boot (same as default behavior, but explicit)
  - [cloud-init-per, always, mount-tmpfs, mount, -t, tmpfs, -o, "size=512M", tmpfs, /mnt/ramdisk]
```

`cloud-init-per` stores state using the name you provide (e.g., `setup-raid`). For `once` frequency, the semaphore is stored in `/var/lib/cloud/sem/` (persists across instance ID changes). For `instance` frequency, it's in `/var/lib/cloud/instance/sem/` (reset when instance ID changes).

### Execution Model

- Commands run as **root**
- Each string entry is executed via `/bin/sh -c "<command>"`
- List entries are executed directly via `execve(3)` (no shell)
- Commands run **sequentially** in the order listed
- A non-zero exit code is logged as a warning but does **not** stop subsequent commands or abort cloud-init
- stdout and stderr go to `/var/log/cloud-init.log`

## runcmd

### When It Runs

`runcmd` executes during the **Config stage** — but with an important nuance. The `cc_runcmd` module runs in the Config stage (`cloud-config.service`), where it writes your commands as a shell script to `/var/lib/cloud/instance/scripts/runcmd`. That script is then **executed** by the `scripts_user` module in the **Final stage** (`cloud-final.service`).

From a user's perspective, your runcmd commands run in the Final stage. At that point:

- All filesystems are mounted
- Package mirrors are configured (`apt_sources`, `yum_repos`)
- Packages listed in `packages:` are installed
- Users and groups from `users:` exist
- SSH keys are deployed
- `write_files` (both immediate and `defer: true`) have completed
- Network is fully operational
- All system services are starting or have started

### Frequency: Once Per Instance

`runcmd` runs **only once per instance** by default. After successful execution, a semaphore file is written to `/var/lib/cloud/instance/sem/config_runcmd`. On subsequent boots, the module is skipped entirely.

> **Note:** If the instance ID changes (e.g., the instance is rebuilt from a snapshot with new metadata), `runcmd` will run again because the semaphore is tied to the instance path.

### Syntax

Same syntax as `bootcmd` — string or list form:

```yaml
#cloud-config
runcmd:
  # String form — shell interpretation
  - echo "Instance setup complete at $(date)" >> /var/log/cloud-init-setup.log

  # List form — no shell
  - [systemctl, enable, --now, nginx]

  # Multi-line shell script
  - |
    #!/bin/bash
    apt-get update
    apt-get install -y docker.io
    systemctl enable --now docker
    usermod -aG docker ubuntu

  # Conditional execution
  - "[ -f /etc/needs-reboot ] && reboot || true"
```

### Execution Model

- Commands run as **root**
- String entries run via `/bin/sh -c`
- List entries run directly
- Sequential execution
- Non-zero exit codes are logged but do not halt execution
- stdout/stderr → `/var/log/cloud-init-output.log` (note: different log than bootcmd)

## Side-by-Side Comparison

| Aspect | bootcmd | runcmd |
|--------|---------|--------|
| Stage | Network (init) — early | Config (written) / Final (executed) — late |
| Frequency | Every boot | Once per instance |
| Filesystems mounted | Not guaranteed | Yes |
| Packages installed | No | Yes (if `packages:` defined) |
| Users created | No | Yes |
| Network available | Yes | Yes |
| `write_files` completed | Only non-deferred | All (including `defer: true`) |
| Typical use | Kernel params, block device setup, early mounts | App install, service config, one-time setup |
| Systemd service | `cloud-init-network.service` | `cloud-config.service` (write) / `cloud-final.service` (execute) |
| Semaphore location | N/A (always runs, use `cloud-init-per`) | `/var/lib/cloud/instance/sem/config_runcmd` |
| Log file | `/var/log/cloud-init.log` | `/var/log/cloud-init-output.log` |

## When to Use bootcmd

Use `bootcmd` when the command must run **before other cloud-init modules** or **on every boot**:

- **Block device setup** — assembling RAID arrays, formatting ephemeral storage, creating LVM volumes
- **Kernel tuning** — sysctl parameters that must be set before services start
- **Filesystem pre-mount** — creating mount points, setting up tmpfs
- **Network workarounds** — adding static routes or iptables rules needed before SSH or cloud-config
- **Hardware initialization** — loading kernel modules, configuring device parameters

```yaml
#cloud-config
bootcmd:
  # Load a kernel module needed for NVMe devices
  - modprobe nvme

  # Set kernel parameters for a database workload
  - sysctl -w vm.dirty_ratio=15
  - sysctl -w vm.dirty_background_ratio=5
  - sysctl -w net.core.somaxconn=65535

  # Format and mount ephemeral NVMe (once only)
  - [cloud-init-per, once, fmt-nvme, mkfs.xfs, /dev/nvme1n1]
  - [cloud-init-per, always, mnt-nvme, mount, /dev/nvme1n1, /mnt/ephemeral]
```

## When to Use runcmd

Use `runcmd` for **one-time setup** that depends on the system being fully initialized:

- **Application installation** — installing and configuring software
- **Service enablement** — starting services that depend on config files and users being present
- **Post-provisioning** — registering with a config management tool, joining a cluster
- **File manipulation** — operations that depend on `write_files` or `packages` being complete

```yaml
#cloud-config
packages:
  - nginx
  - certbot

write_files:
  - path: /etc/nginx/conf.d/myapp.conf
    content: |
      server {
          listen 80;
          server_name myapp.example.com;
          root /var/www/myapp;
      }

runcmd:
  # At this point: nginx is installed, config file is written, 'deploy' user exists
  - systemctl reload nginx
  - certbot --nginx -d myapp.example.com --non-interactive --agree-tos -m admin@example.com
  - chown -R deploy:deploy /var/www/myapp
  - sudo -u deploy git clone https://github.com/org/myapp.git /var/www/myapp
```

## Common Mistakes

### Mistake 1: Using runcmd for Something Needed on Every Boot

```yaml
# WRONG — this only runs once, not after reboot
runcmd:
  - mount /dev/xvdf /data

# CORRECT — runs on every boot
bootcmd:
  - [cloud-init-per, always, mount-data, mount, /dev/xvdf, /data]
```

Better yet, use the `mounts` module for persistent mounts:

```yaml
mounts:
  - [/dev/xvdf, /data, ext4, "defaults,nofail", "0", "2"]
```

### Mistake 2: Using bootcmd for Package Installation

```yaml
# WRONG — apt sources aren't configured yet during bootcmd
bootcmd:
  - apt-get update && apt-get install -y nginx

# CORRECT — packages are available during the final stage
packages:
  - nginx
# or
runcmd:
  - apt-get update && apt-get install -y nginx
```

### Mistake 3: Assuming Users Exist in bootcmd

```yaml
users:
  - name: deploy
    shell: /bin/bash

# WRONG — 'deploy' user doesn't exist yet during bootcmd
bootcmd:
  - sudo -u deploy mkdir /home/deploy/app

# CORRECT — user exists by the time runcmd executes
runcmd:
  - sudo -u deploy mkdir /home/deploy/app
```

### Mistake 4: Forgetting bootcmd Runs Every Boot

```yaml
# DANGEROUS — appends a line to fstab on EVERY boot
bootcmd:
  - echo "/dev/xvdf /data ext4 defaults 0 2" >> /etc/fstab

# SAFE — only runs once
bootcmd:
  - [cloud-init-per, once, add-fstab, sh, -c, 'echo "/dev/xvdf /data ext4 defaults 0 2" >> /etc/fstab']
```

### Mistake 5: Relying on write_files Content in bootcmd

```yaml
write_files:
  - path: /etc/myapp/config.json
    content: '{"port": 8080}'

# WRONG — write_files runs AFTER bootcmd in the init stage
bootcmd:
  - cat /etc/myapp/config.json  # File may not exist yet

# CORRECT — write_files has completed by the time runcmd runs
runcmd:
  - cat /etc/myapp/config.json
```

> **Note:** Non-deferred `write_files` entries actually run in the init stage too, but the ordering relative to `bootcmd` within that stage is: bootcmd first, then write_files. Deferred (`defer: true`) entries run in the final stage, before runcmd.

## Debugging

### Check Which Stage Failed

```bash
# See the full cloud-init status
cloud-init status --long

# Check individual stage results
cat /run/cloud-init/result.json
```

### View bootcmd Output

```bash
# bootcmd output is in the main cloud-init log
grep -A2 "cc_bootcmd" /var/log/cloud-init.log

# Or search for your specific command
grep "bootcmd" /var/log/cloud-init.log
```

### View runcmd Output

```bash
# runcmd output goes to cloud-init-output.log
cat /var/log/cloud-init-output.log

# Module-level logging
grep "cc_runcmd" /var/log/cloud-init.log
```

### Check Semaphores

```bash
# See which modules have run (once-per-instance tracking)
ls /var/lib/cloud/instance/sem/

# Check cloud-init-per semaphores with "once" frequency (persists across instances)
ls /var/lib/cloud/sem/

# Check cloud-init-per semaphores with "instance" frequency
ls /var/lib/cloud/instance/sem/ | grep "bootcmd"
```

### Re-run cloud-init (for Testing)

```bash
# Clean all instance state (CAUTION: re-runs everything on next boot)
sudo cloud-init clean

# Clean and reboot
sudo cloud-init clean --reboot

# Re-run only the final stage (re-runs runcmd)
sudo cloud-init single --name runcmd --frequency once
```

### Analyze Module Execution Order

```bash
# Show timestamps for each module
cloud-init analyze show

# Show blame-style slowest modules
cloud-init analyze blame
```

## Module Execution Order Reference

For context, here's where `bootcmd` and `runcmd` sit relative to other commonly-used modules (based on the default Ubuntu cloud.cfg):

```
Network (init) stage — cloud-init-network.service:
  1. bootcmd            ← runs first in this stage
  2. write_files        (non-deferred only)
  3. growpart
  4. resizefs
  5. disk_setup
  6. mounts
  7. set_hostname
  8. update_hostname
  9. users_groups
  10. ssh

Config stage — cloud-config.service:
  11. ssh_import_id
  12. locale
  13. set_passwords
  14. apt_pipelining
  15. apt_configure
  16. ntp
  17. timezone
  18. runcmd            ← script written to disk here

Final stage — cloud-final.service:
  19. package_update_upgrade_install
  20. write_files       (deferred)
  21. puppet/chef/ansible
  22. scripts_per_boot
  23. scripts_per_instance
  24. scripts_user      ← runcmd script actually executes here
  25. phone_home
  26. final_message
```

## What They Are

- **bootcmd**
  - Runs very early in the boot process (during the Network stage, after networking is up but before most other cloud-init modules).
  - Runs on every boot by default.
  - Good for low-level, early tweaks (kernel/sysctl, early mounts, console tweaks, writing simple files), or actions that must happen before config-stage modules.

- **runcmd**
  - Runs near the end of cloud-init ("final" stage), after users, packages, and networking are configured.
  - Runs once per instance by default.
  - Good for provisioning steps: installing software, fetching from the network, configuring and starting services.

## Execution Characteristics

- **User:** both run as root.
- **Order:** commands run in the order you list them.
- **Shell vs exec:**
  - If you provide a string, it runs via a shell (`sh -c`).
  - If you provide a list, it executes the program directly (no shell expansion).
- **Logging:** output appears in `/var/log/cloud-init-output.log` (and details in `/var/log/cloud-init.log`).

## When to Choose Which

- Need package managers, systemd services, or fully-configured users? Use `runcmd`.
- Need to do something before disk_setup, mounts, users_groups, or other init-stage modules? Use `bootcmd`.
- Need something to run on every boot but late? Prefer a systemd unit or scripts in `/var/lib/cloud/scripts/per-boot` over `runcmd`.

## Examples

Minimal example using both:

```yaml
#cloud-config
bootcmd:
  # Runs early, every boot
  - [sysctl, -w, "net.ipv4.ip_forward=1"]
  - echo "tuned early at $(date)" >> /var/log/bootcmd.log

runcmd:
  # Runs once, late in first boot
  - [apt-get, update]
  - [apt-get, install, "-y", nginx]
  - systemctl enable --now nginx
```

Gate bootcmd to "only once" (even though `bootcmd` runs every boot by default):

```yaml
#cloud-config
bootcmd:
  - [cloud-init-per, instance, early-net, sysctl, -w, "net.ipv4.ip_forward=1"]
```

## Notes and Gotchas

- Network IS available for `bootcmd` (it runs in the Network stage, after interfaces are up). However, higher-level services (DNS resolvers, package repos) may not be fully ready yet.
- Prefer absolute paths in both sections.
- For long-running daemons, create a systemd unit in `write_files` and enable it, rather than starting it in `runcmd`.
- If you need "every boot" behavior at the end of boot, use `/var/lib/cloud/scripts/per-boot` or a systemd service instead of `runcmd`.

## Summary

`bootcmd` is your "early boot, every boot" primitive — use it for block device setup, kernel tuning, and anything that must happen before the system is fully configured. It has no frequency control built in, so wrap individual commands with `cloud-init-per` when you need once-only semantics.

`runcmd` is your "one-time, fully-provisioned" primitive — use it for application setup, service configuration, and post-install tasks that depend on users, packages, and files being in place. It runs once per instance and won't repeat on reboot.

The architectural decision: if your command must run before filesystems, users, and packages are ready, or must repeat on every boot — use `bootcmd`. If it's one-time setup that depends on the full system being available — use `runcmd`.
