# DEBIAN_FRONTEND: Non-Interactive Package Management

`DEBIAN_FRONTEND` is an environment variable that controls how `debconf` (Debian's configuration management system) interacts with users during package installation and configuration. Setting it to `noninteractive` suppresses all prompts — essential for unattended installs in scripts, Dockerfiles, CI/CD, and automation.

## The Problem It Solves

Many Debian/Ubuntu packages ask configuration questions during install:
- "What mail server type?" (postfix)
- "Restart services during upgrades?" (needrestart on Ubuntu 22.04+)
- "Configure timezone?" (tzdata)
- "Accept EULA?" (ttf-mscorefonts-installer)

In a script, these prompts hang indefinitely waiting for input that never comes.

## Available Frontends

| Value | Behavior | Use Case |
|-------|----------|----------|
| `dialog` | Curses/ncurses TUI boxes (default on terminals) | Interactive terminal |
| `readline` | Plain text Q&A prompts | Terminals without curses |
| `gnome` | GTK graphical dialogs | Desktop environments |
| `kde` | KDE graphical dialogs | KDE desktop |
| `editor` | Opens questions in a text editor | Bulk configuration |
| `web` | Web server you connect to with a browser | Proof of concept only — avoid for security reasons |
| `noninteractive` | No prompts — uses defaults silently | Scripts, Docker, CI/CD |
| `teletype` | Bare-bones line-based I/O | Legacy/minimal environments |

## Usage

### Inline (Single Command)

```bash
# Set for one command only (doesn't persist)
DEBIAN_FRONTEND=noninteractive sudo apt install -y postfix

# Or with sudo -E (preserve environment)
export DEBIAN_FRONTEND=noninteractive
sudo -E apt install -y tzdata
```

### For an Entire Script

```bash
#!/bin/bash
export DEBIAN_FRONTEND=noninteractive

apt-get update
apt-get install -y \
  postfix \
  tzdata \
  keyboard-configuration \
  console-setup

# Reset when done (optional — good practice)
unset DEBIAN_FRONTEND
```

### In Dockerfiles

```dockerfile
# Set as ENV (persists for all RUN commands)
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y \
      tzdata \
      locales \
      postfix && \
    rm -rf /var/lib/apt/lists/*

# Better: use ARG so it doesn't persist into the running container
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y tzdata
```

> **Best practice in Docker:** Use `ARG` instead of `ENV`. With `ENV`, the variable persists into the running container, which can mask problems if someone runs `dpkg-reconfigure` inside it. `ARG` only applies during the build.

### In Cloud-Init

```yaml
#cloud-config
package_update: true
packages:
  - postfix
  - tzdata

# cloud-init sets DEBIAN_FRONTEND=noninteractive automatically
# for package installation — no action needed
```

### In Ansible

```yaml
- name: Install packages non-interactively
  apt:
    name:
      - postfix
      - tzdata
    state: present
  environment:
    DEBIAN_FRONTEND: noninteractive
```

### In Packer

```hcl
provisioner "shell" {
  environment_vars = [
    "DEBIAN_FRONTEND=noninteractive",
  ]
  inline = [
    "apt-get update",
    "apt-get install -y postfix tzdata",
  ]
}
```

### In systemd Services / Cron

```bash
# In a cron job
0 3 * * * DEBIAN_FRONTEND=noninteractive apt-get upgrade -y

# In a systemd service
[Service]
Environment="DEBIAN_FRONTEND=noninteractive"
ExecStart=/usr/local/bin/auto-update.sh
```

## Combining with dpkg Options

`DEBIAN_FRONTEND=noninteractive` handles debconf prompts, but config file conflicts are handled separately by `dpkg`. Combine them for fully unattended upgrades:

```bash
# Full non-interactive upgrade (no prompts of any kind)
sudo DEBIAN_FRONTEND=noninteractive apt-get upgrade -y \
  -o Dpkg::Options::="--force-confdef" \
  -o Dpkg::Options::="--force-confold"
```

| dpkg Option | Behavior |
|-------------|----------|
| `--force-confdef` | Use the default action for config file conflicts |
| `--force-confold` | Always keep the old (existing) config file |
| `--force-confnew` | Always install the new config file |
| `--force-confmiss` | Install missing config files |

```bash
# Keep old configs (most common for production)
sudo DEBIAN_FRONTEND=noninteractive apt-get dist-upgrade -y \
  -o Dpkg::Options::="--force-confold" \
  -o Dpkg::Options::="--force-confdef"

# Always take new configs (for fresh/disposable systems)
sudo DEBIAN_FRONTEND=noninteractive apt-get dist-upgrade -y \
  -o Dpkg::Options::="--force-confnew"
```

### Make It Permanent (apt.conf.d)

```bash
# Apply to all apt operations on this system
cat <<'EOF' | sudo tee /etc/apt/apt.conf.d/99unattended
DPkg::options { "--force-confdef"; "--force-confold"; }
APT::Get::Assume-Yes "true";
EOF
```

## Pre-Seeding Answers

For packages that need specific answers (not just defaults), preseed the values before installing:

```bash
# Set timezone non-interactively
echo "tzdata tzdata/Areas select Europe" | debconf-set-selections
echo "tzdata tzdata/Zones/Europe select London" | debconf-set-selections
DEBIAN_FRONTEND=noninteractive apt-get install -y tzdata

# Set postfix type
echo "postfix postfix/main_mailer_type select Internet Site" | debconf-set-selections
echo "postfix postfix/mailname string mail.example.com" | debconf-set-selections
DEBIAN_FRONTEND=noninteractive apt-get install -y postfix

# Accept an EULA
echo "ttf-mscorefonts-installer msttcorefonts/accepted-mscorefonts-eula select true" | debconf-set-selections
DEBIAN_FRONTEND=noninteractive apt-get install -y ttf-mscorefonts-installer

# Configure keyboard layout
echo "keyboard-configuration keyboard-configuration/layout select English (US)" | debconf-set-selections
echo "keyboard-configuration keyboard-configuration/variant select English (US)" | debconf-set-selections
DEBIAN_FRONTEND=noninteractive apt-get install -y keyboard-configuration
```

### Find Available debconf Questions

```bash
# Show questions a package will ask
sudo debconf-show package-name

# Show all debconf selections (system-wide)
sudo debconf-get-selections

# Show selections for a specific package
sudo debconf-get-selections | grep tzdata

# Install debconf-utils if debconf-get-selections is not found
sudo apt install debconf-utils
```

## Suppress needrestart Prompts (Ubuntu 22.04+)

Ubuntu 22.04+ includes `needrestart` which prompts about restarting services after upgrades. This is independent of `DEBIAN_FRONTEND`:

```bash
# Disable needrestart prompts (set to auto-restart)
echo '$nrconf{restart} = "a";' | sudo tee /etc/needrestart/conf.d/99-auto.conf

# Or suppress for a single command
sudo NEEDRESTART_MODE=a apt-get upgrade -y

# Or disable needrestart entirely
sudo NEEDRESTART_SUSPEND=1 apt-get upgrade -y

# Combined: fully silent upgrade on Ubuntu 22.04+
sudo DEBIAN_FRONTEND=noninteractive NEEDRESTART_MODE=a \
  apt-get upgrade -y \
  -o Dpkg::Options::="--force-confold" \
  -o Dpkg::Options::="--force-confdef"
```

## One-Liners

```bash
# Fully non-interactive install
DEBIAN_FRONTEND=noninteractive apt-get install -y package-name

# Non-interactive upgrade keeping old configs
DEBIAN_FRONTEND=noninteractive apt-get upgrade -y -o Dpkg::Options::="--force-confold"

# Non-interactive dist-upgrade (full upgrade)
DEBIAN_FRONTEND=noninteractive apt-get dist-upgrade -y -o Dpkg::Options::="--force-confold" -o Dpkg::Options::="--force-confdef"

# Fully automated unattended upgrade (skip all low-priority questions, quiet output)
export DEBIAN_FRONTEND=noninteractive
export DEBIAN_PRIORITY=critical
export NEEDRESTART_MODE=a
apt-get -qy clean
apt-get -qy update
apt-get -qy -o "Dpkg::Options::=--force-confdef" -o "Dpkg::Options::=--force-confold" upgrade

# Reconfigure package non-interactively (uses existing/default answers)
DEBIAN_FRONTEND=noninteractive dpkg-reconfigure -f noninteractive tzdata

# Set timezone without prompts
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime && \
  echo "Europe/London" > /etc/timezone && \
  DEBIAN_FRONTEND=noninteractive dpkg-reconfigure -f noninteractive tzdata

# Set locale without prompts
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen && \
  locale-gen && \
  update-locale LANG=en_US.UTF-8

# Docker one-liner: install with no cache left behind
RUN DEBIAN_FRONTEND=noninteractive apt-get update && \
    apt-get install -y --no-install-recommends curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# Ubuntu 22.04+ fully silent (debconf + needrestart + dpkg configs)
DEBIAN_FRONTEND=noninteractive NEEDRESTART_MODE=a apt-get upgrade -y \
  -o Dpkg::Options::="--force-confold" -o Dpkg::Options::="--force-confdef"

# Preseed + install in one shot
echo "postfix postfix/main_mailer_type select No configuration" | debconf-set-selections && \
  DEBIAN_FRONTEND=noninteractive apt-get install -y postfix
```

## Tips & Tricks

### Don't Export Globally on Interactive Systems

```bash
# BAD — breaks interactive package management for the whole session
export DEBIAN_FRONTEND=noninteractive
apt install postfix    # Silent, but you lose the ability to answer questions

# GOOD — scope it to just the command
DEBIAN_FRONTEND=noninteractive apt install -y postfix
```

### Check What Frontend Is Active

```bash
# Show current DEBIAN_FRONTEND value
echo $DEBIAN_FRONTEND

# Show debconf's configured default frontend
debconf-show debconf | grep frontend
# Or check the debconf database directly
echo "get debconf/frontend" | debconf-communicate
```

### Use with dpkg-reconfigure

```bash
# Reconfigure a package non-interactively (accepts existing values)
DEBIAN_FRONTEND=noninteractive dpkg-reconfigure package-name

# Reconfigure with specific frontend
dpkg-reconfigure --frontend=readline package-name
dpkg-reconfigure --frontend=dialog package-name

# Set priority (low = show all questions, critical = almost none)
dpkg-reconfigure --priority=low package-name
dpkg-reconfigure --priority=critical package-name
```

### Handle Packages That Ignore DEBIAN_FRONTEND

Some packages use their own prompts outside debconf. Common workarounds:

```bash
# grub — avoid "which disk to install to" prompt
echo "grub-pc grub-pc/install_devices string /dev/sda" | debconf-set-selections

# mysql/mariadb — set root password without prompt
echo "mysql-server mysql-server/root_password password mypassword" | debconf-set-selections
echo "mysql-server mysql-server/root_password_again password mypassword" | debconf-set-selections

# iptables-persistent — auto-save current rules
echo "iptables-persistent iptables-persistent/autosave_v4 boolean true" | debconf-set-selections
echo "iptables-persistent iptables-persistent/autosave_v6 boolean true" | debconf-set-selections
```

### Test Your Script Interactivity

```bash
# Run your script and check if it hangs — set a timeout
timeout 300 bash -c 'DEBIAN_FRONTEND=noninteractive apt-get install -y package-name'
# If it exits with code 124, something still prompted
```

## Complete Unattended Install Script Template

```bash
#!/bin/bash
# unattended-install.sh — fully non-interactive package installation
set -euo pipefail

export DEBIAN_FRONTEND=noninteractive

# Pre-seed any required answers
debconf-set-selections <<'PRESEED'
tzdata tzdata/Areas select Etc
tzdata tzdata/Zones/Etc select UTC
postfix postfix/main_mailer_type select No configuration
PRESEED

# Configure dpkg to keep old config files
APT_OPTS='-o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold"'

# Update and install
apt-get update -qq
eval apt-get install -y -qq $APT_OPTS \
  tzdata \
  postfix \
  nginx \
  curl

# Clean up
apt-get autoremove -y -qq
apt-get clean
rm -rf /var/lib/apt/lists/*

echo "Installation complete."
```

## Dockerfile Best Practice

```dockerfile
FROM ubuntu:24.04

# Use ARG (build-time only) not ENV (persists into runtime)
ARG DEBIAN_FRONTEND=noninteractive

# Combine update + install + cleanup in one layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      curl \
      ca-certificates \
      tzdata && \
    # Set timezone without prompts
    ln -sf /usr/share/zoneinfo/UTC /etc/localtime && \
    echo "UTC" > /etc/timezone && \
    dpkg-reconfigure -f noninteractive tzdata && \
    # Cleanup
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

## AWS EC2 / Cloud User-Data Script Example

A real-world provisioning script for cloud VMs (EC2, Lightsail, Linode) using user-data:

```bash
#!/bin/bash
# cloud-init user-data script for Ubuntu VMs
# Runs on first boot — fully non-interactive

export DEBIAN_FRONTEND=noninteractive
export DEBIAN_PRIORITY=critical
export NEEDRESTART_MODE=a

HOSTNAME="server1.example.com"

# Update system
apt-get -qy update
apt-get -qy -o "Dpkg::Options::=--force-confdef" -o "Dpkg::Options::=--force-confold" upgrade

# Set hostname
hostnamectl set-hostname "$HOSTNAME"

# Install packages
apt-get -qy install \
  ufw \
  fail2ban \
  curl \
  wget \
  unzip

# Configure firewall
ufw default deny incoming
ufw default allow outgoing
ufw allow from 10.0.0.0/8 to any port 22 proto tcp comment 'SSH from VPC'
ufw --force enable

# Apply security sysctl settings
cat <<'EOF' > /etc/sysctl.d/99-security.conf
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
EOF
sysctl --system

# Reboot to apply kernel upgrades
sync
reboot
```

Launch with AWS CLI:

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name my-key \
  --subnet-id subnet-12345 \
  --security-group-ids sg-12345 \
  --user-data file://user-data.sh
```

> **Note:** The `-q` flag in `apt-get -qy` produces output suitable for logging (omits progress indicators). Combined with `-y` it's ideal for unattended scripts.

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Package hangs waiting for input | Add `DEBIAN_FRONTEND=noninteractive` |
| Config file prompt during upgrade | Add `-o Dpkg::Options::="--force-confold"` |
| needrestart prompt (Ubuntu 22.04+) | Add `NEEDRESTART_MODE=a` or configure `/etc/needrestart/conf.d/` |
| Package needs specific answer (not default) | Preseed with `debconf-set-selections` before install |
| `DEBIAN_FRONTEND` set globally in Docker image | Use `ARG` instead of `ENV` |
| Script works locally but hangs in CI | Missing `DEBIAN_FRONTEND=noninteractive` + `-y` flag |
| `dpkg-reconfigure` still prompts | Add `-f noninteractive` flag |
| Grub asks which disk to install to | Preseed `grub-pc/install_devices` |

## Related Environment Variables

| Variable | Purpose |
|----------|---------|
| `DEBIAN_FRONTEND` | Controls debconf prompt behavior |
| `DEBIAN_PRIORITY` | Only show questions at this priority or higher (`critical`, `high`, `medium`, `low`). Setting to `critical` skips almost all prompts. |
| `DEBCONF_NONINTERACTIVE_SEEN` | Mark questions as "seen" (won't be asked again on reconfigure) |
| `NEEDRESTART_MODE` | Controls needrestart: `a` (auto), `l` (list only), `i` (interactive) |
| `NEEDRESTART_SUSPEND` | Set to `1` to disable needrestart entirely for a command |
| `UCF_FORCE_CONFFOLD` | Force ucf-managed config files to keep old version |
| `UCF_FORCE_CONFFNEW` | Force ucf-managed config files to use new version |
