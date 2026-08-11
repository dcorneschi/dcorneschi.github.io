# Linux SysRq (Magic SysRq Key) Guide

## What is SysRq?

The Magic SysRq Key is a key combination that allows sending commands directly to the Linux kernel, bypassing all user-space software. It works even when the system appears hung or unresponsive, as long as the kernel itself is still running.

SysRq provides a last-resort mechanism for:

- Safely rebooting a frozen system
- Killing runaway processes
- Syncing and unmounting filesystems before a hard reset
- Debugging kernel issues
- Triggering crash dumps

## Enabling SysRq

### Important Notes

- Red Hat Enterprise Linux disables SysRq by default for security reasons
- When enabled, any user with access to the physical console gains extra abilities
- Recommended to disable when not troubleshooting, or ensure physical console access is properly secured
- Consult with vendors before using, as third-party applications may be impacted

### Check Current Status

```bash
# Check if SysRq is enabled (0=disabled, 1=all enabled)
cat /proc/sys/kernel/sysrq

# Value meanings:
# 0   - disable sysrq completely
# 1   - enable all functions
# >1  - bitmask of allowed functions
```

### Enable at Runtime

```bash
# Enable all SysRq functions
echo 1 > /proc/sys/kernel/sysrq

# Or using sysctl
sysctl -w kernel.sysrq=1
```

### Enable Permanently

```bash
# /etc/sysctl.conf or /etc/sysctl.d/99-sysrq.conf
kernel.sysrq = 1

# Apply
sysctl -p
```

### Bitmask Values (Selective Enable)

Instead of enabling all functions, you can enable only specific ones:

| Bit | Value | Functions Enabled |
|-----|-------|-------------------|
| 0 | 1 | Enable control of console logging level |
| 1 | 2 | Enable control of keyboard (SAK, unraw) |
| 2 | 4 | Enable debugging dumps of processes |
| 3 | 8 | Enable sync command |
| 4 | 16 | Enable remount read-only |
| 5 | 32 | Enable signaling of processes (term, kill, oom-kill) |
| 6 | 64 | Allow reboot/poweroff |
| 7 | 128 | Allow nicing of all RT tasks |
| 8 | 256 | Allow setting console log level |
| 9 | 512 | Allow core dump on hung tasks |

```bash
# Example: enable only sync (8) + remount (16) + reboot (64) = 88
echo 88 > /proc/sys/kernel/sysrq

# Common safe setting: everything except debugging dumps
# 1+2+8+16+32+64+128+256 = 507 (excludes bit 2: dumps)
echo 507 > /proc/sys/kernel/sysrq
```

Default bitmask values by distribution:

| Distribution | Default Value | Functions Allowed |
|---|---|---|
| RHEL | 0 (disabled) | None (must be enabled manually) |
| Ubuntu | 176 | sync, remount, reboot |
| Debian | 438 | loglevel, unraw, sync, remount, nice-all-RT, reboot |

## How to Use SysRq

### Method 1: Keyboard (Physical Console)

Hold `Alt` + `SysRq` (Print Screen) + `<command key>`

Notes:
- `SysRq` may be released before pressing the command key, as long as `Alt` remains held down
- Combinations always assume the QWERTY keyboard layout
- Cannot work during a kernel panic or hardware failure preventing the kernel from running
- On laptops: `Alt` + `Fn` + `SysRq` + `<key>`
- On Dell laptops: press & hold `Alt+Fn+R`, release `Fn+R`, while still holding `Alt` press the command key
- On Chromebooks: `Alt+VolumeUp` (Alt+F10) replaces `SysRq`

**From X Window System / GUI:** SysRq won't work in raw keyboard mode. First switch keyboard mode with `Alt+SysRq+R` (unraw), then the SysRq key combinations will work.

### Method 2: /proc/sysrq-trigger (Remote/SSH)

```bash
# Send a SysRq command programmatically
echo <key> > /proc/sysrq-trigger

# Examples:
echo s > /proc/sysrq-trigger   # sync
echo b > /proc/sysrq-trigger   # reboot
```

### Method 3: Serial Console

Send a `Break` signal followed by the command key within 5 seconds.

This also works for virtual serial console access through out-of-band service processors:

- HP iLO (Virtual Serial Port)
- Dell DRAC
- IBM IMM (Integrated Management Module)
- Sun/Oracle ILOM
- ipmitool / ipmiconsole

## SysRq Commands Reference

### System Control

| Key | Command | Description |
|-----|---------|-------------|
| `b` | Reboot | Immediately reboot (no sync, no umount) |
| `o` | Power Off | Shut down the system |
| `s` | Sync | Flush all dirty buffers to disk |
| `u` | Unmount | Remount all filesystems read-only |
| `j` | Frozen fs | Thaw all frozen filesystems (FIFREEZE ioctl) |

### Process Control

| Key | Command | Description |
|-----|---------|-------------|
| `e` | tErm | Send SIGTERM to all processes (except init) |
| `i` | kIll | Send SIGKILL to all processes (except init) |
| `f` | OOM Kill | Call the OOM killer to kill a memory-hogging process |
| `k` | SAK | Secure Access Key — kill all processes on the current virtual console |

### Debugging / Information

| Key | Command | Description |
|-----|---------|-------------|
| `t` | Tasks | Show current task state and stack traces on all CPUs |
| `m` | Memory | Show memory information (similar to /proc/meminfo) |
| `p` | Registers | Show current CPU registers |
| `w` | Blocked | Show tasks in uninterruptible (D) state |
| `l` | Backtrace | Show stack backtrace for all active CPUs |
| `d` | Locks | Show all locks held (requires CONFIG_LOCKDEP) |
| `q` | Timers | Show all armed hrtimers and clockevent devices |
| `z` | ftrace | Dump the ftrace buffer |
| `D` | BPF sched | Debug dump of BPF scheduler (newer kernels) |
| `R` | Replay logs | Replay kernel log messages on consoles (newer kernels) |
| `S` | Reset sched | Disable BPF scheduler and revert to CFS (newer kernels) |

### Console / Logging

| Key | Command | Description |
|-----|---------|-------------|
| `0`–`9` | Log Level | Set console log level (0=emergency only, 9=all) |
| `h` | Help | Display help (any unlisted key also shows help) |
| `r` | Unraw | Turn off keyboard raw mode and set it to XLATE |
| `g` | Kgdb | Enter kernel debugger (kgdb on ppc and sh platforms) |
| `v` | Voyager | Dump Voyager SMP processor info |
| `x` | Xmon | Enter xmon interface on ppc/powerpc platforms |

### Crash Dump

| Key | Command | Description |
|-----|---------|-------------|
| `c` | Crash | Trigger a kernel panic (for kdump testing) |

### Scheduling

| Key | Command | Description |
|-----|---------|-------------|
| `n` | Nice | Reset all RT (real-time) tasks to normal priority |

## The REISUB Sequence (Safe Reboot)

When a system is frozen, use this sequence to safely reboot without data loss:

```
Alt+SysRq+R  →  unRaw (take keyboard back from X/raw mode)
Alt+SysRq+E  →  tErm (SIGTERM all processes)
Alt+SysRq+I  →  kIll (SIGKILL remaining processes)
Alt+SysRq+S  →  Sync (flush data to disk)
Alt+SysRq+U  →  Unmount (remount filesystems read-only)
Alt+SysRq+B  →  reBoot
```

**Mnemonic:** "**R**eboot **E**ven **I**f **S**ystem **U**tterly **B**roken"

Wait a few seconds between each key to allow the operation to complete.

Via `/proc/sysrq-trigger`:

```bash
echo r > /proc/sysrq-trigger
sleep 2
echo e > /proc/sysrq-trigger
sleep 10
echo i > /proc/sysrq-trigger
sleep 5
echo s > /proc/sysrq-trigger
sleep 2
echo u > /proc/sysrq-trigger
sleep 2
echo b > /proc/sysrq-trigger
```

## Practical Examples

### Safely Reboot a Frozen System

```bash
# If you have SSH access:
echo s > /proc/sysrq-trigger
sleep 2
echo u > /proc/sysrq-trigger
sleep 2
echo b > /proc/sysrq-trigger
```

Or from the keyboard: `Alt+SysRq+S`, wait, `Alt+SysRq+U`, wait, `Alt+SysRq+B`

### Kill All Processes (Emergency)

```bash
# Send SIGTERM to all (gives processes time to clean up)
echo e > /proc/sysrq-trigger

# Wait 10 seconds, then force kill remaining
sleep 10
echo i > /proc/sysrq-trigger
```

### Trigger a Crash Dump for Analysis

```bash
# Ensure kdump is configured first
systemctl status kdump

# Trigger panic → kdump captures vmcore
echo c > /proc/sysrq-trigger
```

### Debug a Hung System

```bash
# Show what tasks are doing (stack traces)
echo t > /proc/sysrq-trigger

# Show memory state
echo m > /proc/sysrq-trigger

# Show blocked (D state) tasks
echo w > /proc/sysrq-trigger

# Output appears in dmesg / kernel log
dmesg | tail -100
```

### Recover from Stuck X / GUI

If the display server is hung and keyboard doesn't work:

1. `Alt+SysRq+R` — reclaim keyboard from raw mode
2. `Ctrl+Alt+F2` — switch to a virtual terminal
3. Login and fix the issue

### Force OOM Kill

When the system is thrashing due to memory exhaustion:

```bash
# Manually trigger OOM killer
echo f > /proc/sysrq-trigger
```

The kernel will select and kill the process with the highest OOM score.

### Change Console Log Level

```bash
# Set maximum verbosity (see all kernel messages on console)
echo 9 > /proc/sysrq-trigger

# Set minimum (emergencies only)
echo 0 > /proc/sysrq-trigger

# Equivalent to:
dmesg -n 9
```

## Where SysRq Output Goes

SysRq output is written to:

- The kernel ring buffer (`dmesg`)
- The system console (physical terminal / serial console)
- `/var/log/messages` or `/var/log/syslog` (if logging daemon is running)
- `journalctl -k` (systemd journal)

When dealing with machines that are extremely unresponsive, the syslogd service is often unable to log these events. In these situations, a serial console is recommended for capturing the data. Make sure the proper `printk` log level is configured when using a serial console.

```bash
# View SysRq output after triggering
dmesg | grep -i sysrq
dmesg | tail -100

# Watch in real time (on another terminal)
dmesg -w

# Example: trigger help and view output
echo h > /proc/sysrq-trigger
dmesg | tail -1
# [171.748959] sysrq: HELP : loglevel(0-9) reboot(b) crash(c) terminate-all-tasks(e) ...
```

## SysRq on Virtual Machines

### KVM/QEMU

```bash
# Via virsh console
# Send Break: Ctrl+]  then type "sendkey sysrq-<key>"

# Via QEMU monitor
sendkey sysrq

# Or from the host
virsh send-key <domain> KEY_LEFTALT KEY_SYSRQ KEY_<letter>
```

### VMware

In VMware, SysRq can be sent via:

- `Alt+SysRq` key combination (if keyboard passthrough is enabled)
- VMware guest utilities

### VirtualBox

`Host+PrintScreen` sends SysRq to the guest.

### Xen

```bash
# Send SysRq to a Xen guest via xm/xl
xm sysrq <domain> <key>
xl sysrq <domain> <key>

# From Xen paravirtual console: send Break with Ctrl+O, then command key
```

### IBM Power Systems

From the Hardware Management Console (HMC): `Ctrl+O` followed by the desired key.

## SysRq via Hardware Management Consoles (IPMI/DRAC/iLO)

### Trigger NMI via iDRAC Web Interface

Send an NMI from the iDRAC web interface using the NMI button. Ensure the system is configured to panic on NMI first:

```bash
# Configure system to panic on NMI
sysctl -w kernel.panic_on_unrecovered_nmi=1
sysctl -w kernel.unknown_nmi_panic=1
```

### Trigger NMI via ipmitool (Serial Over LAN)

From a separate RHEL system with network access to the DRAC/iLO management IP:

```bash
# Install ipmitool
yum install -y ipmitool     # RHEL 6/7
dnf install -y ipmitool     # RHEL 8+

# Send NMI signal (immediately panics if NMI tunables are set)
ipmitool -H <idrac-ip> -I lanplus -U <user> -P <password> chassis power diag
```

### Trigger SysRq via ipmitool Serial Over LAN

```bash
# Connect to serial over LAN
ipmitool -H <idrac-ip> -I lanplus -U <user> -P <password> sol activate

# Send Break signal (from ipmitool SOL session):
# Press in quick succession:
#   [Shift]+[~]  then  [Shift]+[b]
# Output: ~B [send break]

# Then press the SysRq command key within 5 seconds:
#   m  — memory dump
#   t  — task list
#   w  — blocked tasks
#   c  — crash (trigger kernel panic for kdump)
```

Example output after sending Break + `m`:

```
SysRq : Show Memory
Node 0 DMA per-cpu:
cpu 0 hot: high 0, batch 1 used:0
cpu 0 cold: high 0, batch 1 used:0
...
```

### HP iLO Virtual Serial Port

```bash
# Connect via ipmitool
ipmitool -H <ilo-ip> -I lanplus -U <user> -P <password> sol activate

# Send Break: same as DRAC
# [Shift]+[~]  then  [Shift]+[b]
# Then press SysRq command key
```

### IBM IMM Remote Console

Use the IBM IMM web interface or ipmitool to send Break signals over the serial-over-LAN connection.

## SysRq on Serial Console

For headless servers using serial console:

```bash
# Send Break signal (method depends on terminal emulator):
# minicom: Ctrl+A, F
# screen: Ctrl+A, B
# putty: Break button or Special → Break
# tip: ~#

# Then press the command key within 5 seconds
```

Configure serial console SysRq in GRUB:

```bash
# /etc/default/grub
GRUB_CMDLINE_LINUX="console=ttyS0,115200n8"
```

## Security Considerations

SysRq can be dangerous if physical access is not controlled:

- `c` can crash the system
- `b` reboots without syncing (data loss)
- `e` / `i` kill all processes
- `k` (SAK) kills all processes on the console

### Restricting SysRq

```bash
# Disable completely
echo 0 > /proc/sys/kernel/sysrq

# Allow only safe operations (sync + remount + reboot)
# 8 (sync) + 16 (remount) + 64 (reboot) = 88
echo 88 > /proc/sys/kernel/sysrq

# Permanent
kernel.sysrq = 88
```

### Restrict /proc/sysrq-trigger Access

The `/proc/sysrq-trigger` file is writable only by root by default. Ensure proper permissions:

```bash
ls -la /proc/sysrq-trigger
# --w------- 1 root root 0 /proc/sysrq-trigger
```

## Troubleshooting

### SysRq Not Working

| Issue | Solution |
|-------|----------|
| Disabled in kernel | `echo 1 > /proc/sys/kernel/sysrq` |
| Keyboard not recognized | Try serial console or `/proc/sysrq-trigger` |
| Laptop keyboard | Use `Fn+Alt+SysRq+key` |
| USB keyboard | May not work before USB initialization in early boot |
| KVM/hypervisor | Use hypervisor-specific method (virsh, QEMU monitor) |
| Key not available | Some keyboards label it `PrtSc`, `Print Screen`, or share with another function |

### Verifying SysRq Works

```bash
# Safe test — show help (lists available commands)
echo h > /proc/sysrq-trigger
dmesg | tail -5

# Should show:
# sysrq: HELP : loglevel(0-9) reboot(b) crash(c) terminate-all-tasks(e) ...
```

## Quick Reference

| Action | Keyboard | /proc Method |
|--------|----------|--------------|
| Sync disks | `Alt+SysRq+S` | `echo s > /proc/sysrq-trigger` |
| Remount read-only | `Alt+SysRq+U` | `echo u > /proc/sysrq-trigger` |
| Reboot | `Alt+SysRq+B` | `echo b > /proc/sysrq-trigger` |
| Power off | `Alt+SysRq+O` | `echo o > /proc/sysrq-trigger` |
| SIGTERM all | `Alt+SysRq+E` | `echo e > /proc/sysrq-trigger` |
| SIGKILL all | `Alt+SysRq+I` | `echo i > /proc/sysrq-trigger` |
| Show tasks | `Alt+SysRq+T` | `echo t > /proc/sysrq-trigger` |
| Show memory | `Alt+SysRq+M` | `echo m > /proc/sysrq-trigger` |
| Show blocked | `Alt+SysRq+W` | `echo w > /proc/sysrq-trigger` |
| Crash (kdump) | `Alt+SysRq+C` | `echo c > /proc/sysrq-trigger` |
| OOM kill | `Alt+SysRq+F` | `echo f > /proc/sysrq-trigger` |
| Unraw keyboard | `Alt+SysRq+R` | `echo r > /proc/sysrq-trigger` |
| Safe reboot | R-E-I-S-U-B | See REISUB sequence above |
