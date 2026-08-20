# Ubuntu Repositories Guide

Ubuntu packages are organized into four main repositories based on support level, licensing, and maintenance responsibility.

## The Four Main Repositories

| Repository | Maintained By | License | Security Updates |
|-----------|---------------|---------|-----------------|
| Main | Canonical | Free/Open Source | Yes (5 years LTS) |
| Universe | Community | Free/Open Source | Best-effort / Ubuntu Pro ESM |
| Restricted | Canonical | Proprietary | Yes (5 years LTS) |
| Multiverse | Community | Non-free / Patent-encumbered | Best-effort |

## Main

Canonical-supported free and open-source software. This is the core of Ubuntu.

### What's in Main

- Linux kernel
- systemd, coreutils, bash
- GNOME desktop environment
- Python, GCC, binutils
- OpenSSH, OpenSSL
- apt, dpkg, snap
- NetworkManager
- Firefox (snap), Thunderbird
- LibreOffice
- PostgreSQL, MySQL (community edition)
- Apache, Nginx
- Docker.io (Ubuntu's packaged version)
- LXD/LXC

### Characteristics

- Full security updates from Canonical for the LTS lifecycle (5 years standard, 10 years with ESM)
- Bug fixes and SRUs (Stable Release Updates)
- Packages must meet Ubuntu's licensing policy (DFSG-compatible)
- Smaller set (~5,000–7,000 source packages)
- Covered by `ubuntu-security` and `ubuntu-updates` pockets

### Example Packages

```bash
# Core system
apt, bash, coreutils, systemd, linux-image-generic

# Development
gcc, g++, make, python3, git, cmake

# Server
openssh-server, nginx, apache2, postgresql, mysql-server

# Desktop
gnome-shell, nautilus, gedit, gnome-terminal
```

## Universe

Community-maintained free and open-source software. The largest repository.

### What's in Universe

- Most programming languages and tools (Ruby, Go, Rust, Node.js)
- Desktop applications (GIMP, Inkscape, VLC, Blender)
- Window managers (i3, Sway, Openbox)
- Development frameworks and libraries
- Scientific and academic software
- Games
- Smaller server applications
- Alternative shells (zsh, fish)

### Characteristics

- Maintained by MOTU (Masters of the Universe) community team
- No guaranteed security updates from Canonical (best-effort by community)
- Ubuntu Pro ESM covers Universe packages (10 years security maintenance)
- Largest repository (~30,000+ source packages)
- Packages must be free/open-source but don't require Canonical QA

### Example Packages

```bash
# Desktop apps
gimp, inkscape, vlc, audacity, obs-studio, blender

# Development tools
nodejs, ruby, golang, rustc, cargo, vagrant

# System tools
htop, tmux, zsh, fish, neovim, fzf, ripgrep, bat

# Server software
redis-server, mongodb, rabbitmq-server, grafana

# Libraries
libboost-all-dev, libsdl2-dev, libcurl4-openssl-dev
```

## Restricted

Proprietary drivers and firmware that Canonical supports but cannot modify.

### What's in Restricted

- NVIDIA GPU drivers
- Broadcom wireless drivers
- Intel microcode updates
- Binary firmware blobs
- Some proprietary kernel modules

### Characteristics

- Canonical provides support and updates
- Source code not available (proprietary/binary-only)
- Needed for hardware compatibility
- Enabled by default on Ubuntu Desktop installs
- Small set of packages (hardware-specific)

### Example Packages

```bash
# GPU drivers
nvidia-driver-535, nvidia-driver-545, nvidia-utils-535

# Firmware
linux-firmware, intel-microcode, amd64-microcode

# Wireless
bcmwl-kernel-source
```

## Multiverse

Software restricted by copyright, patents, or legal issues.

### What's in Multiverse

- Media codecs (patent-encumbered)
- RAR/unrar utilities
- Flash player (legacy)
- Some fonts with restrictive licenses
- Software with advertising clauses
- Packages with geographic distribution restrictions
- VirtualBox extension pack

### Characteristics

- Community-maintained, best-effort security fixes
- May have legal restrictions in some jurisdictions
- Not free software (fails DFSG or has patent issues)
- Not enabled by default on minimal installs
- Users accept responsibility for legal compliance

### Example Packages

```bash
# Codecs and media
libdvd-pkg, ubuntu-restricted-extras, libavcodec-extra

# Utilities
unrar, rar, p7zip-rar

# Fonts
ttf-mscorefonts-installer

# Virtualization
virtualbox-ext-pack
```

## Additional Repositories and Pockets

Beyond the four main components, Ubuntu uses pockets for different update stages:

| Pocket | Purpose |
|--------|---------|
| `release` | Original packages at release time |
| `security` | Critical security patches |
| `updates` | Stable release updates (bug fixes) |
| `proposed` | Updates awaiting verification |
| `backports` | Newer software backported from future releases |

### sources.list Structure

```bash
# Format (traditional):
# deb [options] URL suite component [component...]

# Ubuntu 22.04 example
deb http://archive.ubuntu.com/ubuntu jammy main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu jammy-updates main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu jammy-security main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu jammy-backports main restricted universe multiverse
```

### DEB822 Format (Ubuntu 24.04+)

```bash
# /etc/apt/sources.list.d/ubuntu.sources
Types: deb deb-src
URIs: http://archive.ubuntu.com/ubuntu
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

Types: deb deb-src
URIs: http://security.ubuntu.com/ubuntu
Suites: noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

## Managing Repositories

### Enable/Disable Repositories

```bash
# Add Universe (if not enabled)
sudo add-apt-repository universe

# Add Multiverse
sudo add-apt-repository multiverse

# Add Restricted
sudo add-apt-repository restricted

# Remove a repository
sudo add-apt-repository --remove multiverse

# Enable backports
sudo add-apt-repository "deb http://archive.ubuntu.com/ubuntu $(lsb_release -cs)-backports main restricted universe multiverse"
```

### Check Enabled Repositories

```bash
# View sources (traditional)
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/

# View sources (DEB822, Ubuntu 24.04+)
cat /etc/apt/sources.list.d/ubuntu.sources

# Check which repos a package comes from
apt policy nginx

# List all configured repositories
apt-cache policy | grep -E '(http|https)://'
```

### Find Which Repository a Package Belongs To

```bash
# Show package origin
apt-cache showpkg nginx | head -20

# Show component (main/universe/etc.)
apt-cache policy nginx

# Madison format (all available versions and repos)
apt-cache madison nginx

# Check package section
apt-cache show nginx | grep -E '^(Section|Origin|Component):'
```

## PPAs (Personal Package Archives)

Third-party repositories hosted on Launchpad.

```bash
# Add a PPA
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update

# Remove a PPA
sudo add-apt-repository --remove ppa:deadsnakes/ppa

# List installed PPAs
ls /etc/apt/sources.list.d/

# PPA format: ppa:<user>/<ppa-name>
# Translates to: http://ppa.launchpad.net/<user>/<ppa-name>/ubuntu
```

## Security Coverage by Repository

| Repository | Standard LTS (5yr) | ESM / Ubuntu Pro (10yr) |
|-----------|-------------------|------------------------|
| Main | Full coverage | Full coverage |
| Universe | Best-effort | Full coverage (23,000+ packages) |
| Restricted | Full coverage | Full coverage |
| Multiverse | Best-effort | Best-effort |

```bash
# Check if a package has security support
ubuntu-security-status

# With Ubuntu Pro
pro security-status

# Check specific package
apt-cache policy <package> | grep -i security
```

## Ubuntu Pro and ESM

```bash
# Check Pro status
pro status

# Attach Ubuntu Pro (free for personal use, up to 5 machines)
sudo pro attach <token>

# Enable ESM for Universe
sudo pro enable esm-infra
sudo pro enable esm-apps

# Check what packages are covered
pro security-status --esm-apps
pro security-status --esm-infra
```

## Common Tasks

### Install ubuntu-restricted-extras (Codecs)

```bash
# Enable Multiverse first
sudo add-apt-repository multiverse
sudo apt update

# Install restricted extras (codecs, fonts, Flash)
sudo apt install ubuntu-restricted-extras
```

### Minimal Server (Main Only)

```bash
# Check sources on minimal install
cat /etc/apt/sources.list

# Typically only Main and Restricted are enabled
# Add Universe if you need additional tools
sudo add-apt-repository universe
sudo apt update
```

### Pin Packages to a Specific Repository

```bash
# Prefer packages from Main over Universe
cat << 'EOF' | sudo tee /etc/apt/preferences.d/prefer-main
Package: *
Pin: release c=main
Pin-Priority: 900

Package: *
Pin: release c=universe
Pin-Priority: 500
EOF
```

## Quick Reference

| Repository | Support | License | Size | Default |
|-----------|---------|---------|------|---------|
| Main | Canonical | Free | ~6,000 pkgs | Yes |
| Universe | Community | Free | ~30,000 pkgs | Yes (Desktop) |
| Restricted | Canonical | Proprietary | ~100 pkgs | Yes |
| Multiverse | Community | Non-free | ~500 pkgs | Yes (Desktop) |
