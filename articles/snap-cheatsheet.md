# Snap Cheatsheet (Ubuntu)

Snap is a universal package manager developed by Canonical. Snaps are self-contained, sandboxed applications that bundle their dependencies and update automatically.

## Basic Commands

### Install and Remove

```bash
# Install a snap
sudo snap install vlc

# Install from a specific channel
sudo snap install firefox --channel=latest/stable
sudo snap install node --channel=18/stable
sudo snap install code --classic

# Install with classic confinement (full system access)
sudo snap install code --classic

# Install in devmode (no confinement, for testing)
sudo snap install myapp --devmode

# Install a dangerous snap (unsigned)
sudo snap install myapp --dangerous

# Remove a snap
sudo snap remove vlc

# Remove and purge all data
sudo snap remove --purge vlc
```

### Search and Info

```bash
# Search for snaps
snap find firefox
snap find "media player"

# Show detailed info about a snap
snap info vlc

# Show info about an installed snap
snap info --verbose vlc

# List available channels
snap info node | grep -A 20 "channels:"
```

### List Installed Snaps

```bash
# List all installed snaps
snap list

# List with revision details
snap list --all

# List specific snap
snap list firefox
```

## Updates

### Manual Updates

```bash
# Update all snaps
sudo snap refresh

# Update a specific snap
sudo snap refresh vlc

# Update to a specific channel
sudo snap refresh firefox --channel=latest/beta

# Check for available updates
sudo snap refresh --list
```

### Auto-Update Control

```bash
# Hold updates for a snap (prevent auto-update)
sudo snap refresh --hold vlc

# Hold all snaps
sudo snap refresh --hold

# Hold for a specific duration
sudo snap refresh --hold=72h vlc
sudo snap refresh --hold=30d vlc

# Unhold (resume auto-updates)
sudo snap refresh --unhold vlc

# Unhold all
sudo snap refresh --unhold

# Set refresh schedule (system-wide)
sudo snap set system refresh.timer=4:00-7:00,19:00-22:30

# Set refresh on specific days
sudo snap set system refresh.timer=sat,sun,4:00-7:00

# Disable auto-refresh temporarily (metered connection)
sudo snap set system refresh.metered=hold

# Check refresh timer
snap get system refresh.timer
```

## Channels and Tracks

```bash
# Channel format: <track>/<risk>/<branch>
# Tracks: latest, numbered versions (e.g., 18, 20, 22)
# Risk levels: stable, candidate, beta, edge

# Switch channel
sudo snap switch --channel=latest/beta vlc

# Refresh after switching
sudo snap refresh vlc

# Install from specific track
sudo snap install node --channel=20/stable
```

## Revisions and Rollback

```bash
# List all revisions of a snap
snap list --all vlc

# Revert to previous revision
sudo snap revert vlc

# Revert to a specific revision
sudo snap revert vlc --revision=123

# Re-enable a disabled revision
sudo snap enable vlc
```

## Snap Services

```bash
# List services provided by snaps
snap services

# List services for a specific snap
snap services lxd

# Start a snap service
sudo snap start lxd.daemon

# Stop a snap service
sudo snap stop lxd.daemon

# Restart a snap service
sudo snap restart lxd.daemon

# Enable a service (start on boot)
sudo snap start --enable lxd.daemon

# Disable a service (don't start on boot)
sudo snap stop --disable lxd.daemon

# View service logs
sudo snap logs lxd
sudo snap logs lxd.daemon
sudo snap logs -f lxd    # follow mode
sudo snap logs -n 50 lxd # last 50 lines
```

## Connections and Interfaces

Snaps use interfaces to access system resources. Connections grant permissions.

```bash
# List all connections
snap connections

# List connections for a specific snap
snap connections vlc

# List available interfaces
snap interface

# Show details of an interface
snap interface audio-playback

# Connect an interface manually
sudo snap connect vlc:audio-playback

# Disconnect an interface
sudo snap disconnect vlc:audio-playback

# Connect to a specific slot
sudo snap connect myapp:network-control :network-control
```

### Common Interfaces

| Interface | Access Granted |
|-----------|--------------|
| `network` | Outbound network access |
| `network-bind` | Listen on ports |
| `home` | Access user's home directory |
| `removable-media` | Access /media and /mnt |
| `audio-playback` | Play audio |
| `audio-record` | Record audio |
| `camera` | Access camera |
| `x11` | X11 display access |
| `wayland` | Wayland display access |
| `desktop` | Desktop integration |
| `cups-control` | Printing |
| `raw-usb` | Direct USB device access |
| `serial-port` | Serial port access |

## Snap Configuration

```bash
# Get snap configuration
snap get lxd

# Get specific config key
snap get lxd daemon.group

# Set configuration
sudo snap set lxd daemon.group=lxd

# Unset configuration
sudo snap unset lxd daemon.group
```

## Disk Space and Storage

```bash
# Show snap disk usage
du -sh /snap/
du -sh /var/lib/snapd/snaps/

# Show disk usage per snap
snap list | awk '{print $1}' | tail -n +2 | while read snap; do
    size=$(du -sh /snap/$snap/current 2>/dev/null | cut -f1)
    echo "$snap: $size"
done

# Set number of retained revisions (default: 3)
sudo snap set system refresh.retain=2

# Remove old disabled revisions to free space
snap list --all | awk '/disabled/{print $1, $3}' | while read name rev; do
    sudo snap remove "$name" --revision="$rev"
done
```

## Aliases

```bash
# List aliases
snap aliases

# Create an alias
sudo snap alias vlc mediaplayer

# Remove an alias
sudo snap unalias mediaplayer

# Prefer a snap's aliases over other commands
sudo snap prefer vlc
```

## Snapd Management

```bash
# Check snapd version
snap version

# Check snapd status
systemctl status snapd

# Restart snapd
sudo systemctl restart snapd

# View snapd logs
journalctl -u snapd -f

# Check snapd changes (operations history)
snap changes

# View details of a specific change
snap change 123

# Abort a stuck change
sudo snap abort 123
```

## Snap Store Account

```bash
# Login to Snap Store
sudo snap login

# Logout
sudo snap logout

# Login with email
sudo snap login user@example.com

# Check who is logged in
snap whoami
```

## Creating and Development

```bash
# Download a snap without installing
snap download vlc

# Install from local file
sudo snap install ./myapp_1.0_amd64.snap --dangerous

# Run snap in try mode (mount local directory as snap)
sudo snap try ./prime

# Check snap confinement
snap run --shell myapp

# Build a snap (requires snapcraft)
snapcraft

# List snap files
unsquashfs -l myapp_1.0_amd64.snap
```

## Security and Confinement

```bash
# Check confinement mode of installed snaps
snap list | awk '{print $1, $6}'

# Confinement modes:
# strict  — full sandboxing (default)
# classic — full system access (like apt packages)
# devmode — no enforcement, logs violations

# Check AppArmor status for snaps
sudo aa-status | grep snap

# View snap's security profile
cat /var/lib/snapd/apparmor/profiles/snap.vlc.*

# Debug permission issues
journalctl -xe | grep DENIED
dmesg | grep apparmor | grep snap
```

## Disable/Remove Snapd Entirely

```bash
# List installed snaps first
snap list

# Remove all snaps
for snap in $(snap list | awk 'NR>1 {print $1}' | grep -v snapd); do
    sudo snap remove --purge "$snap"
done

# Remove snapd
sudo snap remove --purge snapd
sudo apt remove --purge -y snapd
sudo apt-mark hold snapd

# Prevent snapd from being reinstalled
cat << 'EOF' | sudo tee /etc/apt/preferences.d/nosnap.pref
Package: snapd
Pin: release a=*
Pin-Priority: -10
EOF

# Clean up
sudo rm -rf /snap /var/snap /var/lib/snapd ~/snap
```

## Troubleshooting

### Snap Won't Install

```bash
# Check snapd is running
systemctl status snapd.socket snapd.service

# Restart snapd
sudo systemctl restart snapd

# Check for broken snaps
snap changes | grep Error

# Abort a stuck change and retry manually
sudo snap abort 123
```

### Snap Won't Launch

```bash
# Run from terminal to see errors
snap run vlc

# Check connections/permissions
snap connections vlc

# Check logs
snap logs vlc
journalctl -xe | grep vlc
```

### Snap Using Too Much Disk

```bash
# Reduce retained revisions
sudo snap set system refresh.retain=2

# Remove disabled revisions
snap list --all | awk '/disabled/{print $1, $3}' | while read name rev; do
    sudo snap remove "$name" --revision="$rev"
done

# Check largest snaps
du -sh /snap/*/current | sort -rh | head -10
```

### Slow Startup (Snap Mounts)

```bash
# Check snap mount count
mount | grep snap | wc -l

# Each snap revision adds loop mounts
# Reduce with fewer retained revisions
sudo snap set system refresh.retain=2
```

## Quick Reference

| Action | Command |
|--------|---------|
| Install | `sudo snap install <pkg>` |
| Install (classic) | `sudo snap install <pkg> --classic` |
| Remove | `sudo snap remove <pkg>` |
| Remove + purge data | `sudo snap remove --purge <pkg>` |
| List installed | `snap list` |
| Search | `snap find <query>` |
| Info | `snap info <pkg>` |
| Update all | `sudo snap refresh` |
| Update one | `sudo snap refresh <pkg>` |
| Check updates | `sudo snap refresh --list` |
| Hold updates | `sudo snap refresh --hold <pkg>` |
| Revert | `sudo snap revert <pkg>` |
| Services | `snap services` |
| Connections | `snap connections <pkg>` |
| Logs | `snap logs <pkg>` |
| Version | `snap version` |
| Changes | `snap changes` |
