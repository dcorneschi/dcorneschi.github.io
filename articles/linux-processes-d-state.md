# Why Linux Processes in D State Can't Be Killed

## What is D State?

In Linux, process states are shown by tools like `ps` and `top`. The `D` state (`TASK_UNINTERRUPTIBLE`) means a process is in **uninterruptible sleep** — it is waiting for a condition that cannot be interrupted by signals, including `SIGKILL`.

## Process States

| State | Code | Description |
|-------|------|-------------|
| Running | `R` | Currently executing or in the run queue |
| Sleeping | `S` | Interruptible sleep — waiting for an event, can be woken by signals |
| Uninterruptible sleep | `D` | Waiting for I/O or a kernel resource, cannot be interrupted |
| Stopped | `T` | Stopped by a signal (SIGSTOP, SIGTSTP) or debugger |
| Zombie | `Z` | Terminated but parent hasn't collected exit status (`wait()`) |
| Dead | `X` | Process is being removed (rarely seen) |

## Why D State Exists

The D state protects critical kernel operations from being interrupted mid-way. When a process enters the kernel to perform an operation that must complete atomically (or would leave data structures in a corrupt state if interrupted), the kernel puts it in `TASK_UNINTERRUPTIBLE`.

Common scenarios:

- Waiting for disk I/O to complete
- Waiting for an NFS server response
- Waiting for a device driver to respond
- Waiting for a lock held by another kernel thread
- Performing direct I/O operations
- Memory page faults that require disk reads

If the kernel allowed signals (including `SIGKILL`) during these operations, it would need to handle partial completions, rollbacks, and inconsistent states — which is extremely complex and error-prone for low-level operations.

## Why kill -9 Doesn't Work

```bash
kill -9 <PID>
```

`SIGKILL` (signal 9) is delivered by the kernel when the process returns to a state where it can handle signals. A process in D state is **inside the kernel**, waiting for something, and the kernel deliberately does not check for pending signals until that wait completes.

The signal is queued and will be delivered when:

1. The I/O operation completes
2. The resource becomes available
3. A timeout expires (if one was set)

Until then, the process is immune to all signals — there is no mechanism in userspace to force-kill it.

## Common Causes of Stuck D State Processes

### 1. NFS/Network Filesystem Hangs

The most common cause of persistently stuck D state processes:

```bash
# Identify NFS-related D state processes
ps aux | awk '$8=="D" {print}' | grep -i nfs

# Check NFS mount status
mount -t nfs4
nfsstat -c

# Show pending NFS operations
cat /proc/net/rpc/nfs

# Check if NFS server is reachable
showmount -e nfs-server
```

NFS defaults to "hard" mounts which retry indefinitely. The process will stay in D state until the NFS server responds.

**Fix:**

```bash
# Remount with soft option (allows timeout)
mount -o remount,soft,timeo=10,retrans=3 /mnt/nfs

# Or use intr option (allows interruption — deprecated in newer kernels)
mount -o remount,intr /mnt/nfs

# Force unmount stale NFS mount
umount -f /mnt/nfs

# Lazy unmount (detach and clean up when no longer busy)
umount -l /mnt/nfs
```

### 2. Disk I/O Issues

```bash
# Check for I/O errors in dmesg
dmesg | grep -iE 'error|i/o|timeout|reset'

# Check disk health
smartctl -a /dev/sda

# Check I/O wait
iostat -x 1

# Check which device the process is waiting on
cat /proc/<PID>/wchan
cat /proc/<PID>/stack
```

### 3. Device Driver Problems

```bash
# Check for driver issues
dmesg | tail -50

# See what the process is waiting on
cat /proc/<PID>/stack

# Check loaded modules for problems
lsmod | grep <suspected_module>
```

### 4. Storage/SAN Issues

```bash
# Multipath status
multipath -ll

# Check for path failures
dmesg | grep -i "path\|failover\|offline"

# SCSI device status
cat /sys/class/scsi_device/*/device/state
```

### 5. Kernel Deadlocks

When a kernel lock is held indefinitely, all processes waiting on that lock enter D state:

```bash
# Check for lock dependency issues
dmesg | grep -i "lockdep\|deadlock"

# All processes in D state
ps -eo pid,stat,wchan:30,comm | grep " D"
```

## Diagnosing D State Processes

### Identify D State Processes

```bash
# List all processes in D state
ps aux | awk '$8 ~ /D/ {print}'

# With wait channel (shows what kernel function they're stuck in)
ps -eo pid,stat,wchan:30,comm | grep " D"

# More detail
ps -eo pid,ppid,stat,wchan:40,args | awk '$3 ~ /D/'

# Count processes in D state
ps -eo stat | grep -c "^D"
```

### Determine What the Process is Waiting On

```bash
# Kernel stack trace (most useful)
cat /proc/<PID>/stack

# Wait channel (abbreviated)
cat /proc/<PID>/wchan

# Syscall the process is blocked in
cat /proc/<PID>/syscall

# File descriptors (may show stuck I/O)
ls -la /proc/<PID>/fd/

# Open files
cat /proc/<PID>/fdinfo/*
```

### Example: Reading /proc/PID/stack

```bash
cat /proc/12345/stack
[<0>] rpc_wait_bit_killable+0x24/0x80 [sunrpc]
[<0>] __rpc_execute+0x15c/0x370 [sunrpc]
[<0>] rpc_execute+0x65/0xa0 [sunrpc]
[<0>] nfs4_do_call_sync+0x48/0x70 [nfsv4]
[<0>] nfs4_proc_getattr+0xd6/0x120 [nfsv4]
[<0>] __nfs_revalidate_inode+0xfe/0x2a0 [nfs]
[<0>] nfs_permission+0xf5/0x1a0 [nfs]
[<0>] inode_permission+0xbe/0x160
```

This shows an NFS process stuck waiting for an RPC call to the NFS server.

### Monitor D State Processes Over Time

```bash
# Watch for D state processes
watch -n 1 'ps -eo pid,stat,wchan:30,comm | grep " D"'

# Log D state processes periodically
while true; do
    echo "=== $(date) ==="
    ps -eo pid,stat,wchan:30,comm | grep " D"
    sleep 10
done >> /tmp/dstate_monitor.log
```

## The TASK_KILLABLE State (D State Evolution)

Since Linux kernel 2.6.25, a third sleep state was introduced: `TASK_KILLABLE`. This is a compromise between `TASK_INTERRUPTIBLE` (S) and `TASK_UNINTERRUPTIBLE` (D).

| State | Can be interrupted by | Use case |
|-------|----------------------|----------|
| `TASK_INTERRUPTIBLE` (S) | Any signal | General waiting |
| `TASK_UNINTERRUPTIBLE` (D) | Nothing | Critical kernel operations |
| `TASK_KILLABLE` (D) | Fatal signals only (SIGKILL) | NFS, filesystem waits |

`TASK_KILLABLE` allows `SIGKILL` to terminate the process while still preventing non-fatal signals from interrupting the operation. Many NFS and filesystem operations have been converted to use `TASK_KILLABLE` in modern kernels.

```bash
# Processes using TASK_KILLABLE still show as D in ps
# but can be killed with kill -9
# Check if a D state process responds to SIGKILL:
kill -9 <PID>
# Wait a moment — if it exits, it was TASK_KILLABLE
```

## Resolving Stuck D State Processes

### 1. Wait for the I/O to Complete

Often the best approach. The process will unblock when:

- The disk I/O finishes
- The NFS server responds
- The device driver completes the operation
- A timeout fires

### 2. Fix the Underlying I/O Problem

```bash
# For NFS: check/fix the NFS server
ping nfs-server
showmount -e nfs-server

# For disk: check for hardware errors
dmesg | grep -i error
smartctl -a /dev/sda

# For multipath: check path status
multipath -ll
```

### 3. Force Unmount (NFS)

```bash
# Force unmount
umount -f /mnt/nfs

# Lazy unmount (last resort)
umount -l /mnt/nfs
```

### 4. Remove the Offending Kernel Module

```bash
# If a driver is the cause
rmmod <module_name>

# Force removal (dangerous)
rmmod -f <module_name>
```

### 5. Reboot

If none of the above works and the processes are truly stuck, a reboot is the only option:

```bash
# Try clean reboot first
reboot

# If system is unresponsive, use SysRq
echo b > /proc/sysrq-trigger

# Or the safe sequence: sync, unmount, reboot
echo s > /proc/sysrq-trigger
echo u > /proc/sysrq-trigger
echo b > /proc/sysrq-trigger
```

## Preventing D State Problems

### NFS Mount Options

```bash
# Use soft mounts with timeouts (allows failure instead of hanging)
mount -t nfs -o soft,timeo=10,retrans=3 server:/share /mnt/nfs

# Use background mount (won't block boot)
mount -t nfs -o bg server:/share /mnt/nfs

# Set actimeo for attribute caching
mount -t nfs -o actimeo=30 server:/share /mnt/nfs
```

In `/etc/fstab`:

```
server:/share  /mnt/nfs  nfs  soft,timeo=10,retrans=3,bg  0 0
```

### I/O Timeout Configuration

```bash
# Set SCSI device timeout (seconds)
echo 30 > /sys/block/sda/device/timeout

# Set NFS timeout at mount time
mount -t nfs -o timeo=100,retrans=3 server:/share /mnt

# Set hung_task_timeout to detect stuck processes
echo 120 > /proc/sys/kernel/hung_task_timeout_secs

# Panic on hung tasks (generates vmcore for analysis)
echo 1 > /proc/sys/kernel/hung_task_panic
```

### Monitoring

```bash
# Alert when D state processes exceed threshold
dstate_count=$(ps -eo stat | grep -c "^D")
if [ "$dstate_count" -gt 5 ]; then
    echo "WARNING: $dstate_count processes in D state" | logger -t dstate-monitor
fi

# Log hung task warnings
dmesg | grep "hung_task"
```

## Hung Task Detection

The kernel has a built-in hung task detector (khungtaskd) that monitors processes in D state:

```bash
# View current settings
sysctl kernel.hung_task_timeout_secs
sysctl kernel.hung_task_panic
sysctl kernel.hung_task_warnings

# Configure
# Timeout before warning (0=disabled, default=120)
kernel.hung_task_timeout_secs = 120

# Number of warnings before stopping (use -1 for unlimited)
kernel.hung_task_warnings = 10

# Panic when hung task detected (for vmcore capture)
kernel.hung_task_panic = 0
```

When triggered, the kernel logs:

```
INFO: task process_name:PID blocked for more than 120 seconds.
      Not tainted 5.14.0-362.el9.x86_64 #1
"echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| Identify D state processes | `ps -eo pid,stat,wchan:30,comm \| grep " D"` |
| See what it's waiting on | `cat /proc/<PID>/stack` |
| Check wait channel | `cat /proc/<PID>/wchan` |
| NFS hang | `umount -f` or `umount -l`, check NFS server |
| Disk I/O stuck | `dmesg \| grep error`, check smartctl |
| Try to kill anyway | `kill -9 <PID>` (works if TASK_KILLABLE) |
| Monitor D state | `watch 'ps -eo stat \| grep -c D'` |
| Last resort | Reboot (SysRq+S, SysRq+U, SysRq+B) |

## Kernel Internals

### TASK_UNINTERRUPTIBLE in Kernel Source

In the kernel source (`include/linux/sched.h`), process states are defined as:

```c
#define TASK_RUNNING           0
#define TASK_INTERRUPTIBLE     1    // S state - can receive signals
#define TASK_UNINTERRUPTIBLE   2    // D state - ignores all signals
#define __TASK_STOPPED         4
#define __TASK_TRACED          8
```

When a process calls functions like `wait_event()` or `schedule()` with `TASK_UNINTERRUPTIBLE`, the scheduler marks it non-runnable until a specific wake-up condition occurs.

The process shows as `D` or `D+` in the `STAT` column of `ps aux` or `top` (`D+` means it's in the foreground process group).

### Signal Delivery Mechanism

When you send a signal (including SIGKILL), the kernel sets a pending signal flag in the process's `task_struct`. However, signals are only delivered when the process:

- Returns from kernel mode to user mode
- Checks for pending signals in the scheduler

Processes in D state never reach these checkpoints because they're blocked in kernel code.

Normal signal delivery path:

```c
do_signal() → get_signal() → dequeue_signal()
```

This only runs when returning to userspace via:

```c
ret_to_user() → do_notify_resume() → do_signal()
```

D state processes never call `ret_to_user()` because they're stuck in kernel mode.

### Why the Kernel Uses D State

During operations that cannot be safely interrupted:

- `mutex_lock()` with certain lock types
- Direct I/O operations (`bio_submit()`, block layer operations)
- Memory-mapped I/O writes
- Filesystem metadata updates

The kernel cannot allow interruption because:

1. **Data corruption** — Partial writes to disk structures would corrupt filesystems
2. **Deadlocks** — Killing a process holding a kernel lock would leave it permanently locked
3. **Hardware inconsistency** — Interrupting DMA transfers could leave hardware in undefined states

### Specific Wait Primitives

Common kernel functions that cause D state:

**Block I/O:**

```c
wait_for_completion_io()       // Waits for I/O completion
blk_execute_rq()               // Synchronous block requests
```

**Filesystem operations:**

```c
sync_filesystem()              // Forces dirty data to disk
lock_page()                    // Page cache lock (UNINTERRUPTIBLE)
```

**NFS operations:**

```c
nfs_wait_on_request()          // Waiting for RPC response
rpc_wait_for_completion_task() // RPC call completion
```

### Common wchan Values

The `wchan` field shows what kernel function the process is blocked in:

| wchan Value | Meaning |
|---|---|
| `io_schedule` | Waiting for block I/O |
| `rpc_wait_bit_killable` | NFS RPC call |
| `call_rwsem_down_read_failed` | Lock contention (rwsem) |
| `wait_on_page_bit` | Page cache wait |
| `wait_for_completion_io` | Waiting for I/O completion |
| `blk_execute_rq` | Synchronous block request |
| `nfs4_do_call_sync` | NFS4 synchronous operation |

### Hardware-Level Blocking

When D state involves hardware:

1. Process calls `submit_bio()` to queue I/O
2. Kernel sends SCSI/NVMe command to disk controller
3. Process enters D state waiting for interrupt
4. If disk is dead/hung, no interrupt arrives
5. Process stuck forever (timeout may be 30–120 seconds or infinite)

The kernel can't just "cancel" the operation because:

- The hardware might complete it later, corrupting data
- Kernel data structures are in an intermediate state
- Other processes might be waiting on the same resource

This is why hardware issues often require a reboot — the only way to reset the hardware state machine and kernel data structures completely.

## Key Takeaways

- **D state is by design** — it protects kernel data integrity during I/O operations
- **`kill -9` is queued, not ignored** — it will be delivered when the process exits D state
- **The process isn't broken** — the underlying I/O subsystem is (NFS server, disk, driver)
- **Fix the cause, not the symptom** — restore the I/O path and the process will unblock
- **Modern kernels use TASK_KILLABLE** — many D state waits can now be killed with SIGKILL
- **Hung task detector** helps identify stuck processes before they become a major problem
