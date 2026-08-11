# Running virt-manager Remotely with X11 Forwarding

`virt-manager` is a graphical tool for managing KVM virtual machines. Since it requires a display, running it on a remote headless server requires X11 forwarding from the server to your local machine.

## Install virt-manager

On the KVM host (RHEL/CentOS):

```bash
yum install -y virt-manager

# RHEL 8+ / Fedora
dnf install -y virt-manager
```

On Ubuntu:

```bash
apt install -y virt-manager
```

## Method 1: PuTTY + Xming (Windows)

### Prerequisites

1. Download and install [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)
2. Download and install [Xming](http://www.straightrunning.com/XmingNotes)
3. Start Xming before connecting with PuTTY

### Configure X11 Forwarding in PuTTY

1. Open PuTTY
2. Navigate to **Connection → SSH → X11**
3. Check **Enable X11 forwarding**
4. Set **X display location** to `localhost:0.0`
5. Go back to **Session**, enter the hostname, and connect

### Verify the Connection

After logging in via PuTTY:

```bash
echo $DISPLAY
# Should show: localhost:10.0 (or similar)
```

### Launch virt-manager

```bash
virt-manager
```

The GUI window will appear on your Windows desktop via Xming.

## Method 2: cmd.exe + Xming (Windows 10/11)

If you prefer using the Windows command prompt or PowerShell with OpenSSH:

1. Start Xming
2. Open `cmd.exe` or PowerShell:

```cmd
set DISPLAY=localhost:0.0
mkdir \dev
echo x > \dev\tty
ssh -Y user@hostname
```

Then on the remote host:

```bash
virt-manager
```

## Method 3: XQuartz (macOS)

### Prerequisites

1. Download and install [XQuartz](https://www.xquartz.org)
2. Log out and log back in (or reboot) after installation
3. Open XQuartz from Applications → Utilities

### Connect with X11 Forwarding

```bash
ssh -X user@hostname
```

Or with trusted forwarding (less restrictive):

```bash
ssh -Y user@hostname
```

### Launch virt-manager

```bash
virt-manager
```

The GUI appears in an XQuartz window on your Mac.

## Method 4: Native Linux (X11 Forwarding)

From a Linux workstation with X11:

```bash
# Connect with X11 forwarding
ssh -X user@hostname

# Or trusted forwarding
ssh -Y user@hostname

# Verify
echo $DISPLAY

# Launch
virt-manager
```

## SSH X11 Forwarding Configuration

### Server Side (/etc/ssh/sshd_config)

Ensure these are set on the KVM host:

```bash
X11Forwarding yes
X11DisplayOffset 10
X11UseLocalhost yes
```

Restart sshd if changed:

```bash
systemctl restart sshd
```

### Required Packages on the Server

```bash
# X11 forwarding requires xauth on the server
yum install -y xauth xorg-x11-fonts-Type1    # RHEL/CentOS
dnf install -y xauth xorg-x11-fonts-Type1    # RHEL 8+
apt install -y xauth                          # Ubuntu
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `cannot open display` | DISPLAY not set or X server not running | Start Xming/XQuartz, check `echo $DISPLAY` |
| `X11 forwarding request failed` | `X11Forwarding no` on server or xauth missing | Enable in sshd_config, install xauth |
| Slow/laggy display | Network latency | Use SSH compression: `ssh -XC user@host` |
| `Authorization required` | xauth cookie mismatch | Run `xauth generate :0 .` or reconnect |
| Window doesn't appear | Xming/XQuartz not started | Start the X server before SSH |
| `Gtk-WARNING: cannot open display` | GTK can't connect to X | Verify `$DISPLAY`, try `ssh -Y` instead of `-X` |

### Verify X11 Forwarding Works

```bash
# Quick test with a simple X application
xeyes
# or
xclock
# or
xterm
```

If these open a window, X11 forwarding is working and `virt-manager` should work too.

### SSH Flags

| Flag | Description |
|------|-------------|
| `-X` | Enable X11 forwarding (with security restrictions) |
| `-Y` | Enable trusted X11 forwarding (less restrictive, needed for some apps) |
| `-C` | Enable compression (helps with slow connections) |

```bash
# Best combination for remote virt-manager over slow links
ssh -YC user@hostname
```

## Alternative: virt-manager with SSH URI (No X11 Needed)

If you have virt-manager installed locally, you can connect to a remote KVM host without X11 forwarding:

```bash
# Connect to remote libvirt from your local virt-manager
virt-manager -c qemu+ssh://user@hostname/system
```

This runs the GUI locally but manages VMs on the remote host via SSH. No X server needed on the remote side.

## Quick Reference

| Platform | Steps |
|----------|-------|
| Windows (PuTTY) | Install Xming → PuTTY X11 forwarding → `virt-manager` |
| Windows (cmd) | Install Xming → `set DISPLAY=localhost:0.0` → `ssh -Y` → `virt-manager` |
| macOS | Install XQuartz → `ssh -X user@host` → `virt-manager` |
| Linux | `ssh -X user@host` → `virt-manager` |
| Local GUI (no X11) | `virt-manager -c qemu+ssh://user@host/system` |
