# VNC Cheatsheet

VNC (Virtual Network Computing) provides remote graphical desktop access to Linux systems. This cheatsheet covers TigerVNC on RHEL 6–10 and Ubuntu 22.04/24.04 — installation, configuration, service management, security, and troubleshooting.

## Overview

| Component | Purpose |
|-----------|---------|
| `vncserver` | Starts a VNC session (legacy command, RHEL 6/7) |
| `Xvnc` | The actual X server process for VNC |
| `vncpasswd` | Sets VNC password for a user |
| `vncviewer` | Client to connect to a VNC server |
| `x0vncserver` | Shares the existing physical display (display :0) |
| `systemd-vnc` | systemd unit template for VNC sessions (RHEL 8+) |

### Default Ports

| Display | Port | Connection |
|---------|------|------------|
| :0 | 5900 | Physical display (x0vncserver) |
| :1 | 5901 | First virtual display |
| :2 | 5902 | Second virtual display |
| :N | 5900+N | Nth virtual display |

Formula: **port = 5900 + display number**

---

## Installation

### RHEL 6 / 7

```bash
yum install tigervnc-server tigervnc xorg-x11-fonts-Type1

# RHEL 6: also install a desktop group if not present
yum groupinstall "Desktop"
```

### RHEL 8 / 9 / 10

```bash
dnf install tigervnc-server tigervnc
```

### Ubuntu 22.04 / 24.04

```bash
apt install tigervnc-standalone-server tigervnc-common
```

### Install a Desktop Environment (if needed)

```bash
# RHEL 7–10 — GNOME
dnf groupinstall "Server with GUI"

# RHEL 7–10 — lightweight (Xfce)
dnf install epel-release
dnf groupinstall "Xfce"

# Ubuntu — Xfce (lightweight)
apt install xfce4 xfce4-goodies
```

---

## Set VNC Password

Each user sets their own password:

```bash
vncpasswd
```

This creates `~/.vnc/passwd`. Optionally set a view-only password when prompted.

---

## Configuration

### RHEL 6 / 7: /etc/sysconfig/vncservers (Legacy)

Edit `/etc/sysconfig/vncservers`:

```bash
VNCSERVERS="1:user1 2:user2"
VNCSERVERARGS[1]="-geometry 1280x1024 -depth 24"
VNCSERVERARGS[2]="-geometry 1920x1080 -depth 24"
```

### RHEL 7: systemd Unit Template

Copy the template for each user/display:

```bash
cp /lib/systemd/system/vncserver@.service /etc/systemd/system/vncserver@:1.service
```

Edit `/etc/systemd/system/vncserver@:1.service` — replace `<USER>` with the actual username:

```ini
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target

[Service]
Type=simple
User=<USER>
ExecStartPre=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
ExecStart=/usr/bin/vncserver %i -geometry 1280x1024 -depth 24
ExecStop=/usr/bin/vncserver -kill %i

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now vncserver@:1.service
```

### RHEL 8 / 9 / 10: /etc/tigervnc/vncserver.users

Map display numbers to system users in `/etc/tigervnc/vncserver.users`:

```
:1=user1
:2=user2
```

Configure per-user settings in `~/.vnc/config`:

```
session=gnome
geometry=1920x1080
depth=24
securitytypes=vncauth,tlsvnc
```

Or use the system-wide defaults in `/etc/tigervnc/vncserver-config-defaults`:

```
session=gnome
geometry=1280x1024
depth=24
```

Start the service:

```bash
systemctl enable --now vncserver@:1
```

### Ubuntu: ~/.vnc/xstartup

Create `~/.vnc/xstartup`:

```bash
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
exec startxfce4 &
```

Make it executable:

```bash
chmod +x ~/.vnc/xstartup
```

### RHEL 6: ~/.vnc/xstartup

On RHEL 6, the default xstartup starts `twm` (a minimal window manager). Comment it out and replace with your preferred desktop:

```bash
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
#twm &
exec gnome-session &
```

---

## Service Management

### RHEL 6 (SysVinit)

```bash
service vncserver start
service vncserver stop
service vncserver restart
chkconfig vncserver on
```

### RHEL 7–10 (systemd)

```bash
# Start display :1
systemctl start vncserver@:1

# Stop display :1
systemctl stop vncserver@:1

# Enable on boot
systemctl enable vncserver@:1

# Check status
systemctl status vncserver@:1

# View logs
journalctl -u vncserver@:1
```

### Ubuntu (systemd)

Create `/etc/systemd/system/vncserver@.service`:

```ini
[Unit]
Description=TigerVNC server for display %i
After=syslog.target network.target

[Service]
Type=simple
User=<USER>
PAMName=login
PIDFile=/home/<USER>/.vnc/%H%i.pid
ExecStartPre=/bin/sh -c '/usr/bin/tigervncserver -kill :%i > /dev/null 2>&1 || :'
ExecStart=/usr/bin/tigervncserver :%i -geometry 1920x1080 -depth 24
ExecStop=/usr/bin/tigervncserver -kill :%i

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now vncserver@1
```

### Manual Start/Stop (Any Distro)

```bash
# Start on display :1
vncserver :1 -geometry 1920x1080 -depth 24

# Kill display :1
vncserver -kill :1

# List running VNC sessions
vncserver -list
```

---

## Firewall Configuration

### RHEL 6 (iptables)

```bash
iptables -I INPUT -m state --state NEW -m tcp -p tcp --dport 5901 -j ACCEPT
service iptables save
```

### RHEL 7–10 (firewalld)

```bash
# Single port
firewall-cmd --permanent --add-port=5901/tcp
firewall-cmd --reload

# Port range (displays :1 through :5)
firewall-cmd --permanent --add-port=5901-5905/tcp
firewall-cmd --reload

# Or use the VNC service (covers 5900-5903)
firewall-cmd --permanent --add-service=vnc-server
firewall-cmd --reload
```

### Ubuntu (ufw)

```bash
ufw allow 5901/tcp

# Port range
ufw allow 5901:5905/tcp
```

---

## Sharing the Physical Display (x0vncserver)

To share the actual screen (display :0) instead of a virtual session:

```bash
x0vncserver -PasswordFile=~/.vnc/passwd -display :0
```

This is useful for providing remote assistance on the user's active session.

---

## SSH Tunnel (Recommended for Security)

VNC traffic is unencrypted by default. Always tunnel through SSH:

```bash
# From the client machine, create a tunnel
ssh -L 5901:localhost:5901 user@server

# Then connect the VNC viewer to
localhost:5901
```

Or as a one-liner:

```bash
ssh -f -N -L 5901:localhost:5901 user@server && vncviewer localhost:5901
```

Flags:
- `-f` — background after authentication
- `-N` — no remote command (tunnel only)
- `-L` — local port forwarding

---

## One-Liners and Tips

### Quick Start a Session

```bash
vncserver :1 -geometry 1920x1080 -depth 24 -localhost no
```

### Kill All VNC Sessions for Current User

```bash
vncserver -kill :* 2>/dev/null; killall Xvnc
```

### Find Running VNC Sessions

```bash
ps aux | grep Xvnc
ss -tlnp | grep 590
```

### Reset VNC Password

```bash
rm -f ~/.vnc/passwd && vncpasswd
```

### Start VNC with a Specific Desktop

```bash
# Xfce
vncserver :1 -geometry 1920x1080 -depth 24 -xstartup /usr/bin/startxfce4

# GNOME (RHEL 8+)
vncserver :1 -geometry 1920x1080 -depth 24 -xstartup /usr/bin/gnome-session
```

### Check Which Display Is on Which Port

```bash
ss -tlnp | grep vnc
```

### Remove Stale Lock Files

If VNC fails to start with "A VNC server is already running":

```bash
rm -f /tmp/.X1-lock /tmp/.X11-unix/X1
vncserver :1
```

### Resize a Running Session

```bash
xrandr --size 1920x1080
```

Or connect with:

```bash
vncviewer -AutoResize=1 server:5901
```

### Clipboard Sharing

Ensure `vncconfig` is running inside the VNC session for clipboard sync:

```bash
vncconfig -nowin &
```

### Connect Without GUI (for Testing)

```bash
vncviewer -via user@server localhost:1
```

---

## VNC Configuration Options

### ~/.vnc/config (RHEL 8+)

| Option | Description | Example |
|--------|-------------|---------|
| `session` | Desktop environment to start | `gnome`, `xfce`, `mate` |
| `geometry` | Screen resolution | `1920x1080` |
| `depth` | Color depth (bits) | `16`, `24`, `32` |
| `securitytypes` | Authentication methods | `vncauth,tlsvnc` |
| `localhost` | Only allow local connections | `yes` / `no` |
| `alwaysshared` | Allow multiple connections | `yes` / `no` |
| `dpi` | Dots per inch | `96`, `120` |

### vncserver Command-Line Options

| Flag | Description |
|------|-------------|
| `:N` | Display number |
| `-geometry WxH` | Screen resolution |
| `-depth N` | Color depth |
| `-localhost` | Only accept connections from localhost |
| `-nolisten tcp` | Disable direct TCP connections |
| `-kill :N` | Kill session on display N |
| `-list` | List active sessions |
| `-fg` | Run in foreground |
| `-xstartup cmd` | Override xstartup script |
| `-SecurityTypes` | Comma-separated auth types |

---

## SELinux (RHEL Only)

If VNC fails to start or connect due to SELinux:

```bash
# Check for denials
ausearch -m avc -ts recent | grep vnc

# Allow VNC to connect to network
setsebool -P vnc_use_rawip on

# Restore context on .vnc directory
restorecon -Rv ~/.vnc
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Black screen | xstartup missing or broken | Create/fix `~/.vnc/xstartup` with desktop session |
| Grey screen with X cursor | No window manager started | Add `exec startxfce4 &` or similar to xstartup |
| "A VNC server is already running" | Stale lock file | `rm /tmp/.X1-lock /tmp/.X11-unix/X1` |
| Connection refused | VNC not running or firewall | Check `ss -tlnp | grep 590` and firewall rules |
| Authentication failure | Wrong password | `vncpasswd` to reset |
| Permission denied (systemd) | Unit file has wrong user | Verify `User=` in service file |
| "Xvnc TLS" error on connect | Client doesn't support TLS | Set `securitytypes=vncauth` in config |
| Display too small | Geometry not set | Add `-geometry 1920x1080` or set in `~/.vnc/config` |
| Can't copy/paste | vncconfig not running | Run `vncconfig -nowin &` inside session |
| Session dies immediately | Broken xstartup script | Check `~/.vnc/*.log` for errors |

### VNC Log Files

```bash
# Per-session log
cat ~/.vnc/hostname:1.log

# Systemd journal
journalctl -u vncserver@:1 --no-pager

# Verbose startup (foreground)
vncserver :1 -fg
```

---

## Version Differences by Distribution

| Feature | RHEL 6 | RHEL 7 | RHEL 8 | RHEL 9/10 | Ubuntu 22.04/24.04 |
|---------|--------|--------|--------|-----------|---------------------|
| Package | `tigervnc-server` | `tigervnc-server` | `tigervnc-server` | `tigervnc-server` | `tigervnc-standalone-server` |
| Config | `/etc/sysconfig/vncservers` | systemd unit template | `/etc/tigervnc/vncserver.users` | `/etc/tigervnc/vncserver.users` | `~/.vnc/xstartup` |
| Init system | SysVinit | systemd | systemd | systemd | systemd |
| Service name | `vncserver` | `vncserver@:N` | `vncserver@:N` | `vncserver@:N` | manual/custom unit |
| Default TLS | No | No | Yes | Yes | No |
| Command | `vncserver` | `vncserver` | `vncserver` | `vncserver` | `tigervncserver` |

---

## Useful Links

- Comparison of Remote Desktop Software: https://en.wikipedia.org/wiki/Comparison_of_remote_desktop_software
- TigerVNC Documentation: https://tigervnc.org/
