# Linux System Calls (Syscalls)

## What Are System Calls?

System calls are the interface between user-space programs and the Linux kernel. When a program needs to do anything that requires kernel privileges — open a file, send a network packet, allocate memory, create a process — it makes a system call.

```
┌─────────────────────────────────┐
│         User Space              │
│  (applications, libraries)      │
├─────────────────────────────────┤
│      System Call Interface      │  ← syscall boundary
├─────────────────────────────────┤
│         Kernel Space            │
│  (drivers, filesystems, net)    │
├─────────────────────────────────┤
│           Hardware              │
└─────────────────────────────────┘
```

Programs cannot access hardware or kernel resources directly. They must ask the kernel via syscalls. This separation provides:

- **Security** — user programs can't corrupt kernel memory or access other processes
- **Stability** — a buggy program can't crash the entire system
- **Abstraction** — programs don't need to know hardware details

## User Mode vs Kernel Mode

| | User Mode | Kernel Mode |
|-|-----------|-------------|
| **Who runs here** | Regular applications | OS kernel |
| **Privileges** | Restricted | Full CPU privileges |
| **Hardware access** | Cannot access directly | Full access |
| **Memory access** | Own process memory only | All memory |
| **CPU instructions** | Unprivileged only | All instructions |
| **Purpose** | Isolation and security | Resource management |

User-space programs cannot directly access hardware, other processes' memory, or privileged CPU instructions. They must request these services from the kernel via syscalls.

## How a System Call Works

1. User program calls a C library wrapper (e.g., `write()`)
2. Library places the syscall number in a CPU register (`rax` on x86_64)
3. Arguments go into specific registers (`rdi`, `rsi`, `rdx`, `r10`, `r8`, `r9`)
4. The `syscall` instruction triggers a CPU mode switch (user → kernel)
5. Kernel looks up the handler in the syscall table
6. Handler executes, result goes in `rax`
7. CPU returns to user mode
8. Library returns the result to the program

```
Application          C Library          Kernel
    │                    │                 │
    │  write(fd,buf,n)   │                 │
    │───────────────────>│                 │
    │                    │  syscall(1,...) │
    │                    │────────────────>│
    │                    │                 │  sys_write()
    │                    │                 │  ├─ check permissions
    │                    │                 │  ├─ copy from user
    │                    │                 │  └─ write to device
    │                    │    return       │
    │                    │<────────────────│
    │     return n       │                 │
    │<───────────────────│                 │
```

## Syscall Categories

### Process Management

| Syscall | Description |
|---------|-------------|
| `fork()` | Create a child process (copy of parent) |
| `clone()` | Create a child process with fine-grained control |
| `execve()` | Replace current process with a new program |
| `exit()` | Terminate the current process |
| `wait4()` | Wait for a child process to change state |
| `getpid()` | Get process ID |
| `getppid()` | Get parent process ID |
| `kill()` | Send a signal to a process |
| `prctl()` | Control process attributes |
| `ptrace()` | Trace/debug a process |

### File Operations

| Syscall | Description |
|---------|-------------|
| `open()` / `openat()` | Open a file |
| `close()` | Close a file descriptor |
| `read()` | Read from a file descriptor |
| `write()` | Write to a file descriptor |
| `lseek()` | Reposition file offset |
| `stat()` / `fstat()` | Get file status |
| `chmod()` / `fchmod()` | Change file permissions |
| `chown()` / `fchown()` | Change file ownership |
| `unlink()` | Delete a file |
| `rename()` | Rename a file |
| `mkdir()` | Create a directory |
| `rmdir()` | Remove a directory |
| `dup()` / `dup2()` | Duplicate a file descriptor |
| `pipe()` | Create a pipe |
| `fcntl()` | File descriptor control |
| `ioctl()` | Device-specific control |

### Memory Management

| Syscall | Description |
|---------|-------------|
| `mmap()` | Map files or devices into memory |
| `munmap()` | Unmap memory |
| `mprotect()` | Set memory protection |
| `brk()` | Change data segment size (heap) |
| `mlock()` / `munlock()` | Lock/unlock memory pages |
| `madvise()` | Give advice about memory usage |
| `msync()` | Synchronize mapped memory with file |

### Network

| Syscall | Description |
|---------|-------------|
| `socket()` | Create a network socket |
| `bind()` | Bind a socket to an address |
| `listen()` | Mark socket as passive (server) |
| `accept()` | Accept a connection |
| `connect()` | Initiate a connection |
| `send()` / `sendto()` | Send data on a socket |
| `recv()` / `recvfrom()` | Receive data from a socket |
| `shutdown()` | Shut down part of a full-duplex connection |
| `setsockopt()` | Set socket options |
| `getsockopt()` | Get socket options |

### I/O Multiplexing

| Syscall | Description |
|---------|-------------|
| `select()` | Monitor multiple file descriptors (legacy) |
| `poll()` | Monitor multiple file descriptors |
| `epoll_create()` | Create an epoll instance |
| `epoll_ctl()` | Control an epoll instance |
| `epoll_wait()` | Wait for events on an epoll instance |
| `io_uring_setup()` | Create an io_uring instance (modern async I/O) |

### Signal Handling

| Syscall | Description |
|---------|-------------|
| `rt_sigaction()` | Set signal handler |
| `rt_sigprocmask()` | Block/unblock signals |
| `rt_sigreturn()` | Return from signal handler |
| `sigaltstack()` | Set alternate signal stack |
| `pause()` | Wait for a signal |

### Time and Scheduling

| Syscall | Description |
|---------|-------------|
| `clock_gettime()` | Get current time |
| `nanosleep()` | High-resolution sleep |
| `timer_create()` | Create a POSIX timer |
| `sched_setscheduler()` | Set scheduling policy |
| `sched_yield()` | Yield the processor |
| `getrusage()` | Get resource usage |

### System Information

| Syscall | Description |
|---------|-------------|
| `uname()` | Get system information |
| `sysinfo()` | Get overall system statistics |
| `getuid()` / `getgid()` | Get user/group ID |
| `setuid()` / `setgid()` | Set user/group ID |
| `getcwd()` | Get current working directory |
| `gethostname()` | Get hostname |

### Filesystem

| Syscall | Description |
|---------|-------------|
| `mount()` | Mount a filesystem |
| `umount2()` | Unmount a filesystem |
| `statfs()` | Get filesystem statistics |
| `sync()` | Flush filesystem buffers to disk |
| `fsync()` / `fdatasync()` | Synchronize a file to disk |
| `chroot()` | Change root directory |
| `pivot_root()` | Change root filesystem |

### Namespaces and Containers

| Syscall | Description |
|---------|-------------|
| `unshare()` | Disassociate parts of the execution context |
| `setns()` | Join an existing namespace |
| `clone()` with `CLONE_NEW*` | Create process in new namespace |
| `pivot_root()` | Change root for container isolation |
| `seccomp()` | Restrict available syscalls |

## Syscall Numbers

Each syscall has a unique number used to identify it. Numbers differ by architecture:

```bash
# x86_64 syscall numbers
cat /usr/include/asm/unistd_64.h | head -20

# Or from the kernel source
# arch/x86/entry/syscalls/syscall_64.tbl
```

Common x86_64 syscall numbers:

| Number | Syscall | Description |
|--------|---------|-------------|
| 0 | `read` | Read from file descriptor |
| 1 | `write` | Write to file descriptor |
| 2 | `open` | Open file |
| 3 | `close` | Close file descriptor |
| 4 | `stat` | Get file status |
| 5 | `fstat` | Get file status (by fd) |
| 9 | `mmap` | Map memory |
| 10 | `mprotect` | Set memory protection |
| 11 | `munmap` | Unmap memory |
| 12 | `brk` | Change heap size |
| 39 | `getpid` | Get process ID |
| 41 | `socket` | Create socket |
| 56 | `clone` | Create child process |
| 57 | `fork` | Fork process |
| 59 | `execve` | Execute program |
| 60 | `exit` | Exit process |
| 62 | `kill` | Send signal |

## Tracing System Calls

### strace — Trace Syscalls of a Process

```bash
# Trace a command
strace ls /tmp

# Trace a running process
strace -p <PID>

# Trace with timestamps
strace -t ls /tmp         # HH:MM:SS
strace -tt ls /tmp        # HH:MM:SS.microseconds
strace -T ls /tmp         # show time spent in each syscall

# Trace specific syscalls only
strace -e trace=open,read,write ls /tmp
strace -e trace=network curl example.com
strace -e trace=file ls /tmp
strace -e trace=process bash -c 'ls'
strace -e trace=memory cat /etc/passwd

# Trace categories
strace -e trace=file         # file-related syscalls
strace -e trace=process      # process management
strace -e trace=network      # network syscalls
strace -e trace=signal       # signal-related
strace -e trace=ipc          # IPC syscalls
strace -e trace=memory       # memory management
strace -e trace=desc         # file descriptor related

# Summary of syscalls (count and time)
strace -c ls /tmp

# Follow child processes
strace -f bash -c 'ls | grep tmp'

# Write output to file
strace -o /tmp/trace.log ls /tmp

# Limit string output length
strace -s 200 cat /etc/passwd

# Show paths instead of file descriptors
strace -y ls /tmp

# Trace only failed syscalls
strace -Z ls /nonexistent
```

### Example strace Output

```
execve("/usr/bin/ls", ["ls", "/tmp"], ...) = 0
brk(NULL)                               = 0x55f5a8c9a000
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=32768, ...}) = 0
mmap(NULL, 32768, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f8b2a1c0000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
...
openat(AT_FDCWD, "/tmp", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
getdents64(3, /* 5 entries */, 32768)   = 152
write(1, "file1.txt  file2.txt\n", 21)  = 21
close(3)                                = 0
exit_group(0)                           = ?
```

Each line shows: `syscall(arguments) = return_value`

### strace -c Output (Summary)

```bash
strace -c ls /tmp
```

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 25.00    0.000050          10         5           mmap
 20.00    0.000040          13         3           openat
 15.00    0.000030          10         3           close
 10.00    0.000020          10         2           fstat
 10.00    0.000020          20         1           write
  5.00    0.000010          10         1           read
  5.00    0.000010          10         1           execve
  5.00    0.000010          10         1           getdents64
  5.00    0.000010          10         1           brk
------ ----------- ----------- --------- --------- ----------------
100.00    0.000200                    18           total
```

### ltrace — Trace Library Calls

```bash
# Trace library calls (not syscalls)
ltrace ls /tmp

# Library calls eventually lead to syscalls:
# printf() → write()
# malloc() → brk() or mmap()
# fopen() → openat()
```

### perf — Performance Counters

```bash
# Count syscalls system-wide
perf stat -e 'syscalls:sys_enter_*' -a sleep 5

# Trace specific syscall
perf trace -e openat ls /tmp

# Record syscalls for later analysis
perf trace record -p <PID> sleep 10
perf trace report
```

### /proc/PID/syscall

```bash
# See what syscall a process is currently blocked in
cat /proc/<PID>/syscall

# Output: syscall_number arg1 arg2 arg3 arg4 arg5 arg6 stack_pointer instruction_pointer
# Example:
# 0 0x3 0x7f5c8a000000 0x20000 0x0 0x0 0x0 0x7ffd2c3ae768 0x7f5c89e23a70
# (syscall 0 = read, reading from fd 3)
```

## Syscall Return Values

| Return | Meaning |
|--------|---------|
| >= 0 | Success (actual meaning depends on the syscall) |
| -1 | Error (errno is set) |

Common errno values:

| errno | Name | Description |
|-------|------|-------------|
| 1 | `EPERM` | Operation not permitted |
| 2 | `ENOENT` | No such file or directory |
| 5 | `EIO` | I/O error |
| 9 | `EBADF` | Bad file descriptor |
| 11 | `EAGAIN` | Resource temporarily unavailable |
| 12 | `ENOMEM` | Cannot allocate memory |
| 13 | `EACCES` | Permission denied |
| 17 | `EEXIST` | File exists |
| 22 | `EINVAL` | Invalid argument |
| 28 | `ENOSPC` | No space left on device |
| 32 | `EPIPE` | Broken pipe |
| 110 | `ETIMEDOUT` | Connection timed out |
| 111 | `ECONNREFUSED` | Connection refused |

```bash
# Look up errno numbers
python3 -c "import errno; print(errno.errorcode)"

# Or use perror
perror 13
# OS error code  13:  Permission denied
```

## Syscall Filtering with seccomp

seccomp (Secure Computing Mode) restricts which syscalls a process can make:

```bash
# Check if seccomp is enabled for a process
grep Seccomp /proc/<PID>/status
# Seccomp:  0  (disabled)
# Seccomp:  1  (strict - only read, write, exit, sigreturn)
# Seccomp:  2  (filter - BPF filter decides)
```

### seccomp in Docker

Docker uses seccomp profiles to restrict container syscalls:

```bash
# Default Docker profile blocks ~44 syscalls including:
# - mount, umount2 (filesystem manipulation)
# - reboot, kexec_load (system control)
# - create_module, init_module (kernel modules)
# - ptrace (process debugging)
# - swapon, swapoff (swap management)

# Run container with no seccomp filtering
docker run --security-opt seccomp=unconfined alpine

# Run container with custom seccomp profile
docker run --security-opt seccomp=/path/to/profile.json alpine

# View default profile
docker info --format '{{.SecurityOptions}}'
```

### seccomp in systemd

```ini
[Service]
# Restrict to specific syscalls
SystemCallFilter=@system-service
SystemCallFilter=~@mount

# Syscall groups (prefixed with @):
# @system-service — common service syscalls
# @network-io — network I/O
# @file-system — filesystem operations
# @process — process management
# @signal — signal handling
# @mount — mount operations (often blocked)
# @privileged — privileged operations
# @raw-io — raw I/O
```

## Writing a Program That Uses Syscalls Directly

### C — Using the syscall() Function

```c
#include <unistd.h>
#include <sys/syscall.h>
#include <stdio.h>

int main() {
    // Using the write syscall directly (number 1 on x86_64)
    const char *msg = "Hello from syscall\n";
    syscall(SYS_write, 1, msg, 19);  // fd=1 (stdout), msg, length

    // Get PID via syscall
    long pid = syscall(SYS_getpid);
    printf("PID: %ld\n", pid);

    return 0;
}
```

### Assembly — Direct Syscall (x86_64)

```nasm
; write(1, msg, 13)
section .data
    msg db "Hello, World!", 10

section .text
    global _start

_start:
    mov rax, 1          ; syscall number (write)
    mov rdi, 1          ; arg1: fd (stdout)
    mov rsi, msg        ; arg2: buffer
    mov rdx, 14         ; arg3: length
    syscall             ; invoke kernel

    mov rax, 60         ; syscall number (exit)
    mov rdi, 0          ; arg1: exit code
    syscall
```

### x86_64 Register Convention for Syscalls

| Register | Purpose |
|----------|---------|
| `rax` | Syscall number (before) / return value (after) |
| `rdi` | 1st argument |
| `rsi` | 2nd argument |
| `rdx` | 3rd argument |
| `r10` | 4th argument |
| `r8` | 5th argument |
| `r9` | 6th argument |

## Counting Syscalls

```bash
# How many syscalls does a program make?
strace -c -f ls /tmp 2>&1 | tail -3

# System-wide syscall rate
perf stat -e raw_syscalls:sys_enter -a sleep 1

# Audit specific syscalls
auditctl -a always,exit -F arch=b64 -S openat -k file_opens
ausearch -k file_opens
```

## Common Syscall Patterns

### File Open → Read → Close

```
openat(AT_FDCWD, "/etc/passwd", O_RDONLY) = 3
fstat(3, {...})                           = 0
read(3, "root:x:0:0:...", 4096)          = 1842
close(3)                                 = 0
```

### Process Fork → Exec

```
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD) = 12345
[pid 12345] execve("/usr/bin/ls", ["ls"], ...) = 0
wait4(12345, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 12345
```

### Network Connection

```
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("93.184.216.34")}, 16) = 0
sendto(3, "GET / HTTP/1.1\r\n...", 76, 0, NULL, 0) = 76
recvfrom(3, "HTTP/1.1 200 OK\r\n...", 4096, 0, NULL, NULL) = 1256
close(3)                                  = 0
```

### Memory Allocation (malloc → mmap)

```
brk(NULL)                                = 0x55a5c8e9a000
brk(0x55a5c8ebb000)                      = 0x55a5c8ebb000
mmap(NULL, 1048576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f8b2a000000
```

## Useful Commands

```bash
# List all syscalls available on this system
ausyscall --dump

# Look up syscall number by name
ausyscall openat
# 257

# Look up syscall name by number
ausyscall 257
# openat

# Count number of syscalls in kernel
grep -c "^[0-9]" /usr/include/asm/unistd_64.h

# Check what syscall a blocked process is in
cat /proc/<PID>/syscall

# Man page for a specific syscall
man 2 open
man 2 write
man 2 mmap
```

## Performance Considerations

Syscalls are relatively expensive because they involve:

- **Context switching** — saving and restoring CPU state (registers, stack pointer)
- **Mode switching** — transitioning between user mode and kernel mode
- **TLB flushes** — on some architectures, the Translation Lookaside Buffer may be flushed
- **Validation** — the kernel checks permissions, validates arguments, copies data from user memory
- **Cache effects** — kernel code displaces user code in CPU caches

This is why:

- C libraries (glibc, musl) buffer I/O to reduce syscall frequency (`stdio` buffers multiple `write()` calls into one)
- `vDSO` (virtual Dynamic Shared Object) provides some syscalls without a mode switch (`clock_gettime`, `gettimeofday`)
- Performance-critical code minimizes syscall count (batch operations, `io_uring`, `sendmmsg`)
- `epoll` replaces `select`/`poll` to avoid repeated setup syscalls

```bash
# Measure syscall overhead
strace -c -e trace=write echo "hello" 2>&1 | grep write

# Compare buffered vs unbuffered writes
strace -c dd if=/dev/zero of=/dev/null bs=1 count=10000 2>&1 | tail -5
strace -c dd if=/dev/zero of=/dev/null bs=4096 count=10000 2>&1 | tail -5
```

### vDSO — Syscalls Without Mode Switch

The kernel maps a small shared library (`vDSO`) into every process's address space. Functions like `clock_gettime()` and `gettimeofday()` can execute entirely in user mode by reading kernel-exported data directly.

```bash
# See vDSO mapped in a process
cat /proc/self/maps | grep vdso

# List vDSO functions
objdump -T /usr/lib/x86_64-linux-gnu/libc.so.6 | grep __vdso
```

## Quick Reference

| Task | Command |
|------|---------|
| Trace syscalls of a command | `strace <command>` |
| Trace a running process | `strace -p <PID>` |
| Trace only file syscalls | `strace -e trace=file <command>` |
| Trace only network syscalls | `strace -e trace=network <command>` |
| Syscall summary (count/time) | `strace -c <command>` |
| Follow child processes | `strace -f <command>` |
| See current syscall of PID | `cat /proc/<PID>/syscall` |
| List all syscall numbers | `ausyscall --dump` |
| Trace library calls | `ltrace <command>` |
| Man page for syscall | `man 2 <syscall_name>` |
