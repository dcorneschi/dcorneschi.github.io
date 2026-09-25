# Configuring kdump on RHEL 6–10

`kdump` captures a kernel crash dump (a **vmcore**) when the kernel panics, so you can analyze *why* the machine went down instead of just rebooting blind. It works by reserving a slice of memory at boot for a second "capture" kernel; when the primary kernel crashes, `kexec` boots straight into that capture kernel, which copies the crashed kernel's memory to disk (or a remote target) and reboots.

This guide covers enabling and configuring kdump across RHEL 6 through 10, including the important behavior changes in RHEL 9+.

For what happens at crash time and how to read the result, see [Linux Kernel Panics](articles/linux-kernel-panics.md).

## How kdump Works

1. At boot, the kernel reserves memory via the `crashkernel=` command-line parameter.
2. The `kdump` service loads a capture kernel into that reserved region with `kexec`.
3. On panic, the system jumps to the capture kernel (skipping firmware/POST).
4. The capture kernel runs `makedumpfile` to write a filtered, compressed vmcore, then reboots.

The two things you always configure: **how much memory to reserve** (`crashkernel=`) and **where the vmcore goes** (`/etc/kdump.conf`).

## Version Differences at a Glance

kdump mechanics are broadly the same, but the tooling and the way you set `crashkernel=` changed over releases:

| RHEL | Package | Service mgmt | crashkernel set via |
|------|---------|--------------|---------------------|
| 6 | `kexec-tools` | `service` / `chkconfig` | `/boot/grub/grub.conf` (or `grubby`) |
| 7 | `kexec-tools` | `systemctl` | `grubby` → GRUB2 |
| 8 | `kexec-tools` | `systemctl` | `grubby`, `crashkernel=auto` supported |
| 9 | `kexec-tools` | `systemctl` | `kdumpctl reset-crashkernel` → `/boot/loader/entries` |
| 10 | `kdump-utils` | `systemctl` | `kdumpctl reset-crashkernel` → `/boot/loader/entries` |

> **Biggest change (RHEL 9+):** `crashkernel=auto` is deprecated and you no longer put `crashkernel=` in `/etc/default/grub`. Instead, `kdumpctl reset-crashkernel` writes an appropriate value into the boot loader entries under `/boot/loader/entries`. In RHEL 10 the package was also renamed from `kexec-tools` to `kdump-utils`. ([Red Hat solution](https://access.redhat.com/solutions/6683331))

## 1. Install the Package

```sh
# RHEL 6
yum install kexec-tools

# RHEL 7 / 8 / 9
yum install kexec-tools        # RHEL 7
dnf install kexec-tools        # RHEL 8, 9

# RHEL 10 (package renamed)
dnf install kdump-utils
```

## 2. Reserve Memory (crashkernel=)

kdump needs dedicated memory that the normal kernel won't touch. The amount depends on RAM, architecture, and dump target — the package ships sensible defaults you can start from.

### RHEL 6, 7, 8 — grubby (or crashkernel=auto)

On RHEL 8 (and 7) the automatic value is the simplest start:

```sh
# RHEL 8: let the kernel size the reservation automatically
grubby --update-kernel=ALL --args="crashkernel=auto"
```

Or set an explicit size (recommended when `auto` under-provisions, e.g. with dump filtering, large RAM, or network/encrypted targets):

```sh
grubby --update-kernel=ALL --args="crashkernel=512M"
# a scaled range, reserving more as RAM grows:
grubby --update-kernel=ALL --args="crashkernel=1G-4G:192M,4G-64G:256M,64G-:512M"
```

On **RHEL 6**, edit `/boot/grub/grub.conf` and append `crashkernel=128M` (or use `grubby` if available).

On RHEL 6, `crashkernel=auto` only reserves memory above a physical-RAM threshold: **4 GB** originally, lowered to **2 GB** from the 6.3 GA release (kernel-2.6.32-279.el6). Below that threshold `auto` reserves nothing, so you must request an explicit size such as `crashkernel=128M`. When `auto` does apply, it reserves by RAM size:

| RAM size | crashkernel reservation |
|----------|-------------------------|
| 0 – 2 GB | 128M |
| 2 – 6 GB | 256M |
| 6 – 8 GB | 512M |
| > 8 GB | 768M |

### RHEL 9 and 10 — kdumpctl reset-crashkernel

Do **not** edit `/etc/default/grub`. Let `kdumpctl` compute and install the value:

```sh
# apply the recommended default to the running (and all) kernels
kdumpctl reset-crashkernel --kernel=ALL

# or set an explicit value
kdumpctl reset-crashkernel --crashkernel=512M
```

This writes `crashkernel=` into the entries under `/boot/loader/entries/`.

### Reboot to apply

The reservation only takes effect after a reboot, because memory is carved out early in boot:

```sh
reboot
# verify after reboot:
cat /proc/cmdline | tr ' ' '\n' | grep crashkernel
cat /sys/kernel/kexec_crash_size          # non-zero = memory reserved
cat /proc/iomem | grep "Crash kernel"     # shows the reserved physical range
```

A `kexec_crash_size` of `0` (or no "Crash kernel" line in `/proc/iomem`) means no memory was reserved — kdump cannot work until this is fixed.

## 3. Configure the Dump Target (/etc/kdump.conf)

`/etc/kdump.conf` controls where the vmcore is written and how it's filtered. The default is a local directory.

### Local filesystem (default)

```ini
# /etc/kdump.conf
path /var/crash
core_collector makedumpfile -l --message-level 7 -d 31
```

- `path` — directory (vmcores land in a timestamped subdirectory).
- `core_collector` — `makedumpfile` filters and compresses. `-l` = lzo compression, `-d 31` drops zero/free/cache pages to shrink the dump.

Optionally write to a **separate filesystem** so a full root disk can't block the dump:

```ini
ext4 /dev/mapper/vg-crash
path /var/crash
```

### Remote target over SSH or NFS

```ini
# SSH
ssh user@dumpserver.example.com
sshkey /root/.ssh/kdump_id_rsa
path /var/crash

# NFS
nfs dumpserver.example.com:/export/crash
path /
```

For SSH, propagate the key first with `kdumpctl propagate` so the capture kernel can log in unattended.

### What to do after a successful dump

```ini
default reboot        # reboot after saving (also: halt, poweroff, shell, dump_to_rootfs)
```

Any change to `/etc/kdump.conf` requires restarting the service (next step) to rebuild the kdump initramfs.

## 4. Enable and Start the Service

### RHEL 7–10 (systemd)

```sh
systemctl enable --now kdump
systemctl status kdump
```

### RHEL 6 (SysV init)

```sh
chkconfig kdump on
service kdump start
service kdump status
```

If the service fails to start, the usual cause is that no memory was reserved (step 2) or the target in `/etc/kdump.conf` is unreachable. Check `journalctl -u kdump` (RHEL 7+) or `/var/log/messages`.

## 5. Test It (Trigger a Crash on Purpose)

Only on a machine you can afford to crash — this **immediately** panics the kernel:

```sh
# arm the sysrq trigger, then force a crash
echo 1 > /proc/sys/kernel/sysrq
echo c > /proc/sysrq-trigger
```

The system panics, boots the capture kernel, writes the vmcore to your configured target, and reboots. Afterward, confirm the dump exists:

```sh
ls -lh /var/crash/*/vmcore        # local target
```

## Controlling When kdump Triggers

kdump only produces a vmcore when the kernel actually **panics**. Many failure conditions (an oops, a hung task, an OOM event, certain NMIs) don't panic by default — you opt in with kernel sysctls. Set them in `/etc/sysctl.conf` (or a drop-in under `/etc/sysctl.d/`) and apply with `sysctl -p`.

```ini
# /etc/sysctl.conf
kernel.panic_on_oops=1          # panic on a kernel oops (default on in RHEL)
kernel.hung_task_panic=1        # panic when a task is hung (blocked too long)
kernel.panic=10                 # after a panic, reboot in 10s (0 = wait forever)
vm.panic_on_oom=1               # panic on out-of-memory instead of killing a process
```

```sh
sysctl -p
```

- `kernel.panic_on_oops=1` — turn a non-fatal oops into a full panic so it's captured.
- `kernel.hung_task_panic=1` — panic when the hung-task watchdog fires. For a **one-time** enable without editing config: `echo 1 > /proc/sys/kernel/hung_task_panic`.
- `vm.panic_on_oom=1` and `kernel.panic=10` — panic (then reboot) on OOM. These overlap with the OOM-killer knobs; see [Triggering and Testing the Linux OOM Killer](articles/linux-oom-killer-testing.md) for the trade-offs.

### NMI-triggered panics (older RHEL)

On older RHEL (6/7) you can also panic on Non-Maskable Interrupt conditions. These knobs and the NMI-watchdog behavior have changed in later kernels, so treat this as legacy guidance:

```ini
kernel.unknown_nmi_panic=1           # panic on an unknown NMI (e.g. hardware NMI button)
kernel.panic_on_unrecovered_nmi=1    # panic on an unrecoverable NMI
```

To use a **hardware NMI button** as the trigger, first disable the NMI watchdog (it claims the NMI otherwise): append `nmi_watchdog=0` to the kernel line in `/boot/grub/grub.conf`, and set `kernel.panic_on_io_nmi=1` in `/etc/sysctl.conf`.

## 6. Analyze the vmcore

Install the analysis tools and the matching debug kernel:

```sh
dnf install crash
debuginfo-install kernel          # or dnf install kernel-debuginfo matching the crashed kernel

crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/127.0.0.1-*/vmcore
```

Inside `crash`, useful starting commands are `bt` (backtrace of the panicking task), `log` (kernel ring buffer), `ps`, and `sys`. Deeper analysis is its own topic — see [Linux Kernel Panics](articles/linux-kernel-panics.md).

## Troubleshooting

| Symptom | Likely cause / fix |
|---------|--------------------|
| `kexec_crash_size` is `0` | No memory reserved — set `crashkernel=` and reboot (step 2) |
| kdump service won't start | Bad/unreachable target in `/etc/kdump.conf`, or no reservation |
| Dump target out of space | vmcore too large — increase `-d` dump level in `core_collector`, or use a bigger/separate target |
| No vmcore after crash | Service wasn't enabled, or capture kernel couldn't reach the remote target |
| RHEL 9/10: `crashkernel=auto` ignored | Deprecated — use `kdumpctl reset-crashkernel` instead |

## Key Takeaways

- kdump reserves memory (`crashkernel=`) for a `kexec` capture kernel that writes a **vmcore** on panic.
- **RHEL 6–8:** set `crashkernel=` with `grubby` (RHEL 8 also supports `crashkernel=auto`); reboot to apply.
- **RHEL 9–10:** don't edit grub — use `kdumpctl reset-crashkernel`; the package is `kdump-utils` on RHEL 10.
- Configure the target in `/etc/kdump.conf` (local, separate FS, SSH, or NFS) and restart the service after edits.
- Verify with `cat /sys/kernel/kexec_crash_size` (non-zero) or `grep "Crash kernel" /proc/iomem`, and test with `echo c > /proc/sysrq-trigger` on a disposable host.
- kdump only fires on a **panic** — use sysctls like `kernel.panic_on_oops`, `kernel.hung_task_panic`, and `vm.panic_on_oom` to turn other failures into capturable panics.

## Quick Reference

```sh
# Install
dnf install kexec-tools      # RHEL 8/9  (kdump-utils on RHEL 10)

# Reserve memory
grubby --update-kernel=ALL --args="crashkernel=auto"   # RHEL 6-8
kdumpctl reset-crashkernel --kernel=ALL                # RHEL 9/10
reboot

# Verify reservation
cat /sys/kernel/kexec_crash_size        # must be non-zero

# Enable service
systemctl enable --now kdump            # RHEL 7-10

# Test (crashes the box!)
echo 1 > /proc/sys/kernel/sysrq
echo c > /proc/sysrq-trigger

# Analyze
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/*/vmcore
```

For related material, see [Linux Kernel Panics](articles/linux-kernel-panics.md).

## Links

- Red Hat: [Configuring kdump on the command line (RHEL 10)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_monitoring_and_updating_the_kernel/configuring-kdump-on-the-command-line)
- Red Hat: [crashkernel=auto is deprecated in RHEL 9](https://access.redhat.com/solutions/6683331)
