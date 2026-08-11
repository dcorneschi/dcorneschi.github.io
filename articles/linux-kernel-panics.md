# Linux Kernel Panics: Hard Panics and Soft Panics

Linux kernel failures are classified into two categories: hard panics (`Aieee!`) and soft panics (`Oops`). Understanding the difference helps determine severity, recovery options, and root-cause analysis.

A kernel panic is a voluntary halt to all system activity when an abnormal situation is detected by the kernel — an internal fatal error from which it cannot safely recover.

## Overview

| Type | Name | Severity | System State |
|------|------|----------|--------------|
| Hard Panic | `Aieee!` | Critical | System halts completely, no recovery |
| Soft Panic | `Oops` | Serious | Process killed, system may continue running |

### Panic Causes

**Hardware:**

- Machine Check Exceptions (MCE)
- Error Detection and Correction (EDAC)
- Non-Maskable Interrupts (NMIs)
  - Hardware NMI Button
  - NMI Watchdog
  - `unknown_nmi_panic`
  - `panic_on_unrecovered_nmi`
  - `panic_on_io_nmi`

**Software:**

- `BUG()` macro (intentional kernel assertion failure)
- Bad pointer handling (NULL dereference, use-after-free)
- Pseudo-hangs (deadlock, livelock)
- Out-of-Memory killer

## Hardware Causes in Detail

### Machine Check Exceptions (MCE)

<cite index="1-13">Component failures detected and reported by the hardware via an exception. Almost always indicates a hardware problem (could be a firmware issue in rare cases).</cite>

Typical output:

```
kernel: CPU 0: Machine Check Exception: 4
       Bank 0: b278c00000000175
kernel: TSC 4d9eab664a9a60
kernel: Kernel panic - not syncing: Machine check
```

<cite index="1-13">To decode, pipe the entire line through `mcelog --ascii`.</cite>

### Error Detection and Correction (EDAC)

<cite index="1-14">Hardware mechanism to detect and report memory chip and PCI transfer errors.</cite>

<cite index="1-15">Reported in `/sys/devices/system/edac/{mc/,pci}` and logged by the kernel as:</cite>

```
EDAC MC0: CE page 0x283, offset 0xce0, grain 8,
syndrome 0x6ec3, row 0, channel 1 "DIMM_B1":
amd76x_edac
```

<cite index="1-15">Informational EDAC messages (such as a corrected ECC error) are printed to the system log. Critical EDAC messages (such as exceeding a hardware-defined temperature threshold) trigger a kernel panic.</cite>

### Non-Maskable Interrupts (NMIs)

<cite index="1-16,1-17">NMIs are hardware-generated interrupts that cannot be masked by normal means. Generally used to signal hardware errors.</cite>

**Unknown NMIs** — <cite index="1-18,1-19,1-20,1-21,1-22,1-23">The kernel has mechanisms to handle certain known NMIs appropriately, unknown ones typically result in kernel log warnings such as:</cite>

```
Uhhuh. NMI received.
Dazed and confused, but trying to continue
You probably have a hardware problem with your RAM chips
Uhhuh. NMI received for unknown reason 32.
Dazed and confused, but trying to continue.
Do you have a strange power saving mode enabled?
```

<cite index="1-24,1-25">These unknown NMI messages can be produced by ECC and other hardware problems. The kernel can be configured to panic when these are received through this sysctl:</cite>

```bash
kernel.unknown_nmi_panic=1
```

This is generally only enabled for troubleshooting.

**NMI Watchdog** — <cite index="1-26,1-27">Enables the built-in kernel deadlock detector. By executing periodic NMI interrupts, the kernel can monitor whether any CPU has locked up.</cite>

<cite index="1-28">To enable, boot with `nmi_watchdog=[1|2]`.</cite>

- <cite index="1-29">When active, the "NMI" count should keep increasing in `/proc/interrupts`.</cite>
- <cite index="1-29,1-30">When a CPU fails to acknowledge an NMI interrupt after some time, the hardware triggers an interrupt and the corresponding handler calls `panic()`. Typically indicates a deadlock situation: a running process attempts to acquire a spinlock which is never granted.</cite>
- <cite index="1-31">The NMI Watchdog cannot be used at the same time as `unknown_nmi_panic`.</cite>

## Software Causes in Detail

### The BUG() Macro

<cite index="1-32">Called by kernel code when an abnormal situation is seen. Typically indicates a programming error when triggered. The calling code is intentional code written by the developer.</cite>

Calls look like:

```c
BUG_ON(in_interrupt());
```

<cite index="1-32">Inserts an invalid operand (0x0000) to serve as a landmark by the trap handler.</cite>

Output example:

```
Kernel BUG at spinlock:118
invalid operand: 0000 [1] SMP
CPU 0
```

### Bad Pointer Handling

<cite index="1-33">Typically indicates a programming error. Detection is hardware assisted (MMU).</cite>

Typical messages:

```
NULL pointer dereference at 0x1122334455667788 ..
```

or:

```
Unable to handle kernel paging request at virtual address 0x11223344
```

<cite index="1-33">Typically due to NULL pointer dereference, accessing an illegal address on this architecture, or possibly memory corruption.</cite>

### Pseudo-hangs

<cite index="1-34,1-35">In certain situations, the system appears to be hung, but some progress is being made. Livelock — if running a realtime kernel, application load could be too high, leading the system into a state where it becomes effectively unresponsive in a "live lock/busy wait" state. The system is not actually hung, but just moving so slowly that it appears to be hung.</cite>

Other causes of pseudo-hangs:

- <cite index="1-36">Thrashing — continuous swapping with close to no useful processing done</cite>
- <cite index="1-36">Lower zone starvation — on i386 the low memory has a special significance and the system may "hang" even when there's plenty of free memory</cite>
- <cite index="1-36">Memory starvation in one node in a NUMA system</cite>

<cite index="1-36">Hangs not detected by the hardware are trickier to debug:</cite>

- Use `sysrq + t` to collect process stack traces when possible
- Enable the NMI watchdog which should detect those situations
- Run hardware diagnostics when it's a hard hang: memtest86, HP diagnostics

### Out-of-Memory Killer

<cite index="1-37">In certain memory starvation cases, the OOM killer is triggered to force the release of some memory by killing a "suitable" process. In severe starvation cases, the OOM killer may have to panic the system when no killable processes are found:</cite>

```
Kernel panic - not syncing: Out of memory and no killable processes...
```

<cite index="1-37">The kernel can also be configured to always panic during an OOM by setting the `vm.panic_on_oom = 1` sysctl.</cite>

## Hard Panic (Aieee!)

A hard panic is a fatal, unrecoverable error. The kernel determines it cannot safely continue execution and halts the system immediately.

### Characteristics

- System freezes completely — no keyboard input, no network, no disk I/O
- Console displays `Kernel panic - not syncing:` followed by a reason
- Requires a physical reboot (or watchdog/IPMI reset)
- All unsaved data is lost
- The system cannot write to disk (crash dump requires special configuration)

### Common Causes

| Cause | Description |
|-------|-------------|
| Root filesystem failure | Cannot mount or access `/` |
| Init process death | PID 1 (init/systemd) exits or is killed |
| Unrecoverable memory corruption | Critical kernel data structures destroyed |
| Hardware failure | RAM errors, CPU exceptions, bus errors |
| Stack overflow | Kernel stack exhausted (typically 8 KB or 16 KB) |
| Double fault | Exception occurred while handling another exception |
| Out of memory (no OOM recovery) | All memory exhausted with no process to kill |
| Missing critical module | Required driver not available at boot |

### Example Hard Panic Output

```
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
CPU: 0 PID: 1 Comm: swapper/0 Not tainted 5.14.0-362.el9.x86_64 #1
Hardware name: QEMU Standard PC (i440FX + PIIX, 1996)
Call Trace:
 panic+0x372/0x3d0
 mount_block_root+0x1d2/0x280
 prepare_namespace+0x130/0x160
 kernel_init_freeable+0x22e/0x240
 kernel_init+0x16/0x130
 ret_from_fork+0x22/0x30
---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) ]---
```

```
Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000009
```

### Triggering a Hard Panic (Testing)

```bash
# Trigger a panic via sysrq (requires root)
echo c > /proc/sysrq-trigger

# Set system to auto-reboot after panic (N seconds, 0=disabled)
echo 10 > /proc/sys/kernel/panic
# Or in /etc/sysctl.conf:
# kernel.panic = 10
```

## Soft Panic (Oops)

An Oops is a non-fatal kernel error. The kernel detects a bug (null pointer dereference, invalid memory access, etc.) but determines it can recover by killing the offending process and continuing.

### Characteristics

- The offending process is killed
- System typically continues running (but may be unstable)
- Error message is logged to kernel ring buffer and syslog
- If the Oops occurs in interrupt context or a critical path, it may escalate to a full panic
- Can be a symptom of hardware issues, driver bugs, or memory corruption

### When an Oops Becomes a Panic

An Oops escalates to a hard panic when:

- It occurs in interrupt context (no process to kill)
- It occurs in a kernel thread critical to system operation
- The `panic_on_oops` sysctl is set to `1`
- The kernel determines the system state is too corrupted to continue

```bash
# Check current setting
cat /proc/sys/kernel/panic_on_oops

# Force panic on any Oops (useful for crash dumps)
echo 1 > /proc/sys/kernel/panic_on_oops

# Or permanently in /etc/sysctl.conf:
# kernel.panic_on_oops = 1
```

### Example Oops Output

```
BUG: unable to handle kernel NULL pointer dereference at 0000000000000010
IP: [<ffffffff812345ab>] some_function+0x1b/0x30
PGD 0
Oops: 0000 [#1] SMP
Modules linked in: ext4 mbcache jbd2 xfs libcrc32c sr_mod cdrom ata_generic pata_acpi
CPU: 2 PID: 3456 Comm: httpd Not tainted 4.18.0-553.el8.x86_64 #1
RIP: 0010:some_function+0x1b/0x30
RSP: 0018:ffffb456c0d4be00 EFLAGS: 00010246
RAX: 0000000000000000 RBX: ffff8a3c4b2f1000 RCX: 0000000000000001
RDX: 0000000000000000 RSI: ffff8a3c4b2f1000 RDI: 0000000000000000
Call Trace:
 calling_function+0x45/0x90
 sys_write+0x55/0xa0
 do_syscall_64+0x5b/0x1a0
 entry_SYSCALL_64_after_hwframe+0x65/0xca
---[ end trace 0000000000000001 ]---
```

### Understanding the Oops Fields

| Field | Description |
|-------|-------------|
| `BUG:` | Type of error (NULL pointer, page fault, etc.) |
| `IP:` / `RIP:` | Instruction pointer — the exact code location that faulted |
| `Oops: 0000` | Error code (bit field: bit 0=protection, bit 1=write, bit 2=user mode) |
| `[#1]` | Oops count since boot |
| `SMP` | System is SMP (multi-processor) |
| `Modules linked in:` | Loaded kernel modules (tainted modules may be flagged) |
| `CPU:` | Which CPU the Oops occurred on |
| `PID:` / `Comm:` | Process ID and name that triggered the Oops |
| `Not tainted` | Kernel taint status (see below) |
| `Call Trace:` | Stack backtrace showing function call chain |

### Oops Error Code Bits

The number after `Oops:` is a bit field:

| Bit | Meaning when set |
|-----|------------------|
| 0 | Protection fault (vs. page not present) |
| 1 | Write access (vs. read) |
| 2 | User-mode access (vs. kernel mode) |
| 3 | Use of reserved bits in page table |
| 4 | Instruction fetch fault |

Example: `Oops: 0002` = write fault in kernel mode to a non-present page.

## Kernel Taint Flags

The `Not tainted` or `Tainted:` field indicates if the kernel has been "tainted" by non-standard conditions:

| Flag | Description |
|------|-------------|
| `P` | Proprietary module loaded (no GPL license) |
| `F` | Module forced loaded |
| `S` | SMP kernel running on non-SMP hardware |
| `R` | Module forcibly unloaded |
| `M` | Machine check exception occurred |
| `B` | Bad page referenced |
| `U` | User requested taint |
| `D` | Kernel died recently (OOPS or BUG) |
| `A` | ACPI table overridden |
| `W` | Warning issued |
| `C` | Staging driver loaded |
| `I` | Workaround for firmware bug applied |
| `O` | Out-of-tree module loaded |
| `E` | Unsigned module loaded |
| `L` | Soft lockup occurred |
| `K` | Live-patched kernel |
| `X` | Auxiliary taint (distro-specific) |
| `T` | Kernel built with struct randomization plugin |

```bash
# Check current taint status
cat /proc/sys/kernel/tainted

# 0 = not tainted
# Non-zero = tainted (decode the bit field)
```

## Crash Dump Configuration (kdump)

<cite index="1-42,1-43,1-44">Kernel crash dumps are captured using the kdump mechanism. Kexec is used to start another complete copy of the Linux kernel in a reserved area of memory. This secondary kernel takes over and copies the memory pages to the crash dump location.</cite>

Analyzing the kernel core requires:

- The `crash` utility
- The vmcore file
- The Linux kernel debugging symbols (vmlinux)

<cite index="1-64">The major version of RHEL is not relevant — crash on RHEL6 can process RHEL5 vmcores provided the correct debugging info is available. However architecture is relevant: crash on x86_64 can only process x86_64 vmcores.</cite>

### Install and Configure kdump

```bash
# RHEL 6
yum install kexec-tools
chkconfig kdump on
service kdump start

# RHEL 7
yum install kexec-tools
systemctl enable --now kdump

# RHEL 8 / 9 / 10
dnf install kexec-tools
systemctl enable --now kdump

# Ubuntu 22.04 / 24.04
apt install linux-crashdump kdump-tools
# Answer "Yes" when asked to enable kdump
systemctl enable --now kdump-tools
```

### Check kdump Status

```bash
# RHEL 7+
systemctl status kdump

# RHEL 6
service kdump status

# Ubuntu
systemctl status kdump-tools
kdump-config show
```

### Reserve Memory for kdump

Add to GRUB kernel command line:

```bash
# /etc/default/grub — add to GRUB_CMDLINE_LINUX:
crashkernel=256M

# Or use auto (recommended for RHEL 6.2+, RHEL 7+)
# On x86, this reserves 128MB base + 64MB per TB
crashkernel=auto

# Rebuild grub (RHEL 7+)
grub2-mkconfig -o /boot/grub2/grub.cfg              # BIOS
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg     # UEFI

# Rebuild grub (Ubuntu)
update-grub
```

<cite index="1-48">Manual crashkernel sizing (RHEL 5.x, 6.0, and 6.1):</cite>

| RAM Size | crashkernel Parameter |
|----------|----------------------|
| Up to 2 GB | 128 MB |
| 2 GB – 6 GB | 256 MB |
| 6 GB – 8 GB | 512 MB |
| Over 8 GB | 768 MB |

### Configure Dump Location

<cite index="1-49">kdump destination configured in `/etc/kdump.conf`. vmcores can be sent to:</cite>

```bash
# RHEL: /etc/kdump.conf
path /var/crash
core_collector makedumpfile -l --message-level 7 -d 31

# Raw device
raw /dev/sda4

# Filesystem
ext4 /dev/sda3
# Will dump vmcore to /dev/sda3:/var/crash

# NFS share
nfs nfs.example.com:/export/vmcores

# Another system via SSH
ssh kdump@crash.example.com
# Run 'service kdump propagate' after SSH config changes

# Ubuntu: /etc/default/kdump-tools
KDUMP_COREDIR="/var/crash"
```

### Core Collector Page Filtering

<cite index="1-50,1-51">The core collector can optionally be configured to discard unneeded pages and compress the needed ones. Zero and free pages are rarely needed. In most cases, cache and user pages are not needed.</cite>

```bash
# Discard all optional pages and compress
core_collector makedumpfile -d 31 -c
```

| Option Value | Discard |
|---|---|
| 1 | Zero pages |
| 2 | Cache pages |
| 4 | Cache private |
| 8 | User pages |
| 16 | Free pages |

`-d 31` = 1+2+4+8+16 (discard all optional pages)

### Allowing kdump to Complete

<cite index="1-52">Intervention can interrupt a complete core collection.</cite>

**HP Automated Server Recovery:**

```bash
# Check ASR status
hpasmcli -s 'SHOW ASR'

# Disable ASR
hpasmcli -s 'DISABLE ASR'

# Set longer timeout (minutes)
hpasmcli -s 'SET ASR 30'
```

**Red Hat Cluster Suite (Power fencing):**

Give the server time to fully kdump before fencing:

```xml
<fence_daemon ... post_fail_delay="300" ... />
```

Or in RHEL 6.2+ use `fence_kdump`.

### Ensure Panic Triggers Dump

```bash
# /etc/sysctl.conf or /etc/sysctl.d/kdump.conf
kernel.panic = 10
kernel.panic_on_oops = 1
```

### Analyze Crash Dumps

```bash
# Install crash utility
# RHEL 6
yum install --enablerepo=rhel-6-server-debug-rpms crash kernel-debuginfo kernel-debuginfo-common

# RHEL 7
yum install crash kernel-debuginfo

# RHEL 8 / 9 / 10
dnf install crash kernel-debuginfo

# Ubuntu
apt install crash linux-image-$(uname -r)-dbgsym

# Open dump
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/<timestamp>/vmcore

# If vmcore is incomplete
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/<timestamp>/vmcore --minimal

# Basic crash commands
crash> bt          # backtrace
crash> log         # kernel log buffer
crash> ps          # process list
crash> sys         # system info
crash> files       # open files
crash> mount       # mounted filesystems
crash> exit
```

## Investigating Panics

### Investigating Userspace Process Crashes

```bash
# Enable application cores of unlimited size
ulimit -c unlimited

# Send QUIT signal to abort and dump the core
kill -s SIGQUIT <PID>

# Use gdb to debug the resulting core file
gdb /path/to/application /path/to/core
```

### Check Logs After Reboot

```bash
# Current boot kernel messages
dmesg | grep -iE 'panic|oops|bug|error|fault'

# Previous boot logs (if journald persistent)
journalctl -b -1 | grep -iE 'panic|oops|bug'

# System log
grep -iE 'panic|oops|kernel' /var/log/messages   # RHEL
grep -iE 'panic|oops|kernel' /var/log/syslog     # Debian/Ubuntu

# Check for crash dumps
ls -la /var/crash/
```

### Decode a Call Trace

```bash
# Find which module/function caused the crash
# Look at the first few entries in the Call Trace

# Check if a module is responsible
modinfo <module_name>

# Check kernel version and patches
uname -r
rpm -qa kernel              # RHEL
dpkg -l linux-image-*       # Ubuntu

# Search for known bugs
# Use the function name from RIP/IP line to search Red Hat Bugzilla or kernel.org
```

### SysRq Emergency Commands

If the system is hung but SysRq still works:

```bash
# Enable SysRq (if not already)
echo 1 > /proc/sys/kernel/sysrq

# Key combinations (Alt+SysRq+key):
# b - Immediately reboot (no sync/umount)
# s - Sync all mounted filesystems
# u - Remount all filesystems read-only
# o - Power off
# c - Trigger a crash dump (for testing kdump)
# t - Show task states
# m - Show memory info
# e - Send SIGTERM to all processes (except init)
# i - Send SIGKILL to all processes (except init)

# Safe reboot sequence: Alt+SysRq+R E I S U B
# (Raise keyboard, tErm all, kIll all, Sync, Unmount, reBoot)
```

## Preventing Panics

### Hardware

```bash
# Test memory
memtest86+

# Check for MCE (Machine Check Exceptions)
dmesg | grep -i mce
mcelog --client

# Monitor hardware health
smartctl -a /dev/sda
ipmitool sel list
```

### Software

```bash
# Keep kernel updated
yum update kernel       # RHEL 6 / 7
dnf update kernel       # RHEL 8 / 9 / 10
apt upgrade linux-image-generic   # Ubuntu

# Check for known issues before applying changes
rpm -q --changelog kernel | head -50    # RHEL
apt changelog linux-image-generic       # Ubuntu

# Test kernel modules in staging before production
modprobe --dry-run <module>

# Verify filesystem integrity
xfs_repair -n /dev/sda1    # dry run
fsck -n /dev/sda1          # dry run
```

### System Configuration

```bash
# /etc/sysctl.conf — panic-related settings

# Auto-reboot after panic (seconds, 0=disabled)
kernel.panic = 10

# Panic on Oops (recommended for servers with kdump)
kernel.panic_on_oops = 1

# Panic on hung task (seconds, 0=disabled)
kernel.hung_task_panic = 0
kernel.hung_task_timeout_secs = 120

# Panic on soft lockup
kernel.softlockup_panic = 0

# Panic on NMI (Non-Maskable Interrupt)
kernel.panic_on_unrecovered_nmi = 1
kernel.panic_on_io_nmi = 1
kernel.unknown_nmi_panic = 0

# Panic on OOM (when no killable processes found)
vm.panic_on_oom = 0

# Apply
sysctl -p
```

### Auto-Generate vmcore on Soft Lockups

To automatically capture a vmcore when a soft lockup occurs (useful for analysis by Red Hat Support):

```bash
# Enable panic on soft lockup (runtime)
echo 1 > /proc/sys/kernel/softlockup_panic

# Or using sysctl
sysctl -w kernel.softlockup_panic=1

# Make permanent in /etc/sysctl.conf or /etc/sysctl.d/:
kernel.softlockup_panic = 1
```

A soft lockup is defined as a bug that causes the kernel to loop in kernel mode for an unreasonable amount of time (60 seconds under RHEL 6+) without giving other tasks a chance to run. When `softlockup_panic=1` is set, the system will deliberately crash and generate a vmcore at the time of a soft lockup, provided kdump is configured and operational.

Ensure kdump is configured and tested before enabling this. The system will reboot after the vmcore is captured.

## Kernel Debugging Categories

### Category 1: OOPS

An OOPS can indicate either a bug in the kernel code or a hardware problem. The text of the OOPS contains critical information. OOPS messages may appear on the machine's console, in `/var/log/messages`, or both. If X is running, check virtual console one (`Ctrl+Alt+F1`).

OOPS messages take the general form:

1. A line explaining the reason the OOPS was generated
2. Additional lines of information varying by failure type
3. The keyword `Oops:`
4. A list of loaded modules (in some versions)
5. A register and stack dump

**Capturing an OOPS** can be a challenge if it doesn't get recorded in logs. Options:

- Hand-copying from the console (tedious but sometimes the only option)
- Serial console capture
- Netdump / kdump

### Category 2: Mysterious Freezes or Hangs

These symptoms typically indicate deadlock or livelock:

- **Deadlock** — each process in a set is blocked, waiting on a resource held by another process in the set
- **Livelock** — a kernel subsystem is given work at a rate greater than it can complete it

Signs:

- Completely hung system (no virtual console switch, no keyboard echo)
- Processes stuck in `D` (uninterruptible sleep) state
- High load average but low CPU utilization

For these cases:

- Enable SysRq to send commands directly to the kernel
- Enable the NMI watchdog to detect hung processors and auto-generate an OOPS

### Category 3: Performance Problems

The vaguest and most difficult to diagnose. Useful information to gather:

- What subsystem is causing the problem (network, virtual memory, NFS, etc.)
- Why specifically you believe there is a performance problem
- Did the problem appear suddenly?
- Did it correspond to a kernel upgrade?
- Does it disappear when booting into the old kernel?
- Were there other configuration changes that coincided?
- Is there a specific test case that demonstrates the problem?

### Category 4: Data Corruption

Very rare due to extensive testing, but possible. Red Hat will need a reproducible test case. Data corruption often manifests with OOPS messages — check system logs. Submit exact symptoms (corrupt file contents, destroyed directory, volume won't mount, etc.).

## Crash Analysis on RHEL 6 / CentOS 6

For RHEL 6 specifically (older kernel, different debug repo names):

### Installation

**Red Hat:**

```sh
yum install --enablerepo=rhel-6-server-debug-rpms crash
```

Find the kernel version needed for the crash dump:

```sh
crash --osrelease /var/crash/127.0.0.1-2017-09-13-02\:18\:39/vmcore
```

Install the matching debuginfo:

```sh
yum install kernel-debuginfo-<version> kernel-debuginfo-common-<version>
```

**CentOS:**

```sh
yum install --enablerepo=base-debuginfo crash kernel-debuginfo kernel-debuginfo-common
```

## Crash Utility Commands

### Collecting a vmcore

```bash
# Manually trigger a panic
echo c > /proc/sysrq-trigger

# Or enable Magic SysRq keys and press SysRq+c on console keyboard
echo 1 > /proc/sys/kernel/sysrq
```

### Checking the vmcore

```bash
# Check the kernel version of the vmcore
crash --osrelease 127.0.0.1-2017-09-13-02\:18\:39/vmcore

# Alternative
strings vmcore | head
```

### Common crash Commands

```bash
# Find the reason for the panic
crash> sys | grep -e RELEASE -e PANIC

# Dump the kernel message buffer
crash> log

# Display process status information
crash> ps

# Display blocked processes (uninterruptible sleep)
crash> ps | grep UN

# Display a kernel stack backtrace
crash> bt

# Display kernel command line
crash> p saved_command_line

# Display a sysctl parameter
crash> p panic_on_oops

# Display load average
crash> sys | grep -e CPUS -e LOAD

# Display memory usage
crash> kmem -i

# Display task state summary
crash> ps -S

# Count running/runnable tasks
crash> ps | grep RU | wc -l

# Display system info
crash> sys

# Display open files
crash> files

# Display mounted filesystems
crash> mount

# Display configured network interfaces
crash> net

# Exit crash
crash> exit
```

### Crash Output: Pipes and Redirects

<cite index="1-79">All crash commands can be piped to external programs or redirected to files.</cite>

```bash
# Redirect log to a file
crash> log > log.txt

# Filter output through external programs
crash> ps | fgrep bash | wc -l

# Sort processes by memory
crash> ps | sed "s/^>//" | sort -n -k7 | tail -20
```

### Incomplete vmcores

<cite index="1-81">A full kernel core dump may not always be captured. Can happen if there is insufficient space to capture the complete core, or if the system is reset by external means (power fencing, HP ASR, etc.).</cite>

When trying to open an incomplete vmcore, crash may give errors such as:

```
crash: read error: kernel virtual address: ffff81082ff147c0 type: "cpu_pda entry"
crash: unable to initialize kmem slab cache subsystem
```

<cite index="1-83,1-88">Sometimes the vmcore can still provide useful information in "minimal mode". In minimal mode, available commands are: `log`, `dis`, `rd`, `sym`, `eval`, `set`, and `exit`.</cite>

```bash
crash --minimal vmlinux vmcore
crash> log | tail -5
```

## Generating a Core Dump from a Running Process

```sh
# Install the gcore plugin
yum install crash-gcore-command       # RHEL 6 / 7
dnf install crash-gcore-command       # RHEL 8 / 9 / 10
apt install crash                     # Ubuntu (gcore included)

# Generate core dump from a running process
gcore -o /tmp/gcore_crond $(pidof -s crond)

# Find out what program generated the core file
file /tmp/gcore_crond.1990

# View the core file with gdb
gdb /usr/sbin/crond /tmp/gcore_crond.1990
```

## Analyzing Core Dumps

```sh
# Display ELF headers
readelf -Wa /core

# Display section contents
objdump -s /core

# Display archive headers
objdump -a /core
```

## Best Practices

- <cite index="1-113,1-114">Is system tainted? Try to reproduce in an untainted configuration, if possible.</cite>
- <cite index="1-115">After a crash, capture a sosreport, and supply both the sosreport and the vmcore to Red Hat GSS.</cite>
- <cite index="1-116">Don't attach vmcores to support cases, upload them to Red Hat's FTP dropbox instead.</cite>

## Other Troubleshooting Tools

| Tool | Purpose |
|------|---------|
| `sysstat` | Capture system activity over time (sar, iostat, mpstat) |
| `Ksar` | Visualize sysstat output |
| `gdb` | Application core dump analysis |
| `strace` | Kernel/user space debugging |
| `SystemTap` | Instrument a running kernel |

## Quick Reference

| Scenario | Indicator | Recovery |
|----------|-----------|----------|
| Hard panic | `Kernel panic - not syncing:` | Reboot required |
| Soft panic | `Oops:` or `BUG:` | Process killed, system may continue |
| Hung system | No response, no logs | SysRq reboot or hardware reset |
| Soft lockup | `BUG: soft lockup - CPU#X stuck for Xs!` | May self-recover or escalate |
| Hard lockup | `NMI watchdog: Watchdog detected hard LOCKUP` | Typically requires reboot |
| OOM kill | `Out of memory: Killed process` | System continues (process lost) |

### Key Files and Paths

| Path | Purpose |
|------|---------|
| `/proc/sys/kernel/panic` | Auto-reboot timeout after panic |
| `/proc/sys/kernel/panic_on_oops` | Convert Oops to panic |
| `/proc/sys/kernel/tainted` | Kernel taint status |
| `/proc/sysrq-trigger` | Trigger SysRq actions |
| `/var/crash/` | kdump crash dump location |
| `/var/log/messages` | System log (RHEL) |
| `/etc/kdump.conf` | kdump configuration |
| `/etc/sysctl.conf` | Kernel parameter tuning |
