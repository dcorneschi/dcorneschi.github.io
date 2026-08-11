# Linux Kernel Map

<img src="/articles/images/linux-kernel-map.svg" alt="Linux Kernel Map" width="800">

## Overview

The Linux kernel is a monolithic kernel with modular capabilities. It manages all hardware resources and provides services to user-space programs through system calls. This article maps the major subsystems, their relationships, and how data flows through the kernel.

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Space                               │
│  Applications  │  Libraries (glibc)  │  Daemons  │  Shells      │
├─────────────────────────────────────────────────────────────────┤
│                   System Call Interface                         │
├────────┬──────────┬──────────┬──────────┬──────────┬────────────┤
│Process │  Memory  │   VFS    │ Network  │  IPC     │  Security  │
│Mgmt    │  Mgmt    │          │  Stack   │          │            │
├────────┼──────────┼──────────┼──────────┼──────────┼────────────┤
│Scheduler│ Page    │Filesys-  │ Protocol │ Signals  │  SELinux   │
│         │ Cache   │ tems     │ Handlers │ Pipes    │  AppArmor  │
│         │ Slab    │ ext4/xfs │ TCP/UDP  │ Sockets  │  Seccomp   │
│         │ Alloc   │ btrfs    │ IP       │ SHM      │  Caps      │
├────────┴──────────┴──────────┴──────────┴──────────┴────────────┤
│                        Device Drivers                           │
│  Block  │  Character  │  Network  │  USB  │  GPU  │  Input      │
├─────────────────────────────────────────────────────────────────┤
│               Architecture-Dependent Code                       │
│  x86  │  ARM  │  RISC-V  │  Interrupt Handling  │  Boot         │
├─────────────────────────────────────────────────────────────────┤
│                          Hardware                               │
│  CPU  │  RAM  │  Disk  │  NIC  │  GPU  │  USB  │  Bus           │
└─────────────────────────────────────────────────────────────────┘
```

## Major Kernel Subsystems

### 1. Process Management

Handles process lifecycle, scheduling, and execution.

| Component | Purpose |
|-----------|---------|
| Scheduler (CFS) | Decides which process runs on which CPU |
| fork/clone | Creates new processes/threads |
| exec | Loads new programs into process memory |
| Signals | Asynchronous notifications between processes |
| Namespaces | Process isolation (PID, network, mount, etc.) |
| cgroups | Resource limits (CPU, memory, I/O) |

```
fork() → copy_process() → wake_up_new_task() → schedule()
                                                    │
                                          ┌─────────┴─────────┐
                                          │  CFS Scheduler    │
                                          │  (pick_next_task) │
                                          └───────────────────┘
```

Key files:

```bash
/proc/<PID>/status      # Process state
/proc/<PID>/sched       # Scheduling statistics
/proc/<PID>/cgroup      # cgroup membership
/proc/<PID>/ns/         # Namespace links
/proc/schedstat         # Scheduler statistics
```

### 2. Memory Management

Controls physical and virtual memory allocation, paging, and caching.

| Component | Purpose |
|-----------|---------|
| Virtual Memory (VMM) | Per-process address space isolation |
| Page Allocator (Buddy) | Physical page allocation in power-of-2 blocks |
| Slab Allocator (SLUB) | Efficient small object allocation |
| Page Cache | Cache file data in RAM |
| Swap | Move inactive pages to disk |
| OOM Killer | Free memory by killing processes when exhausted |
| Huge Pages | 2 MB / 1 GB pages for large memory workloads |
| NUMA | Non-Uniform Memory Access awareness |

```
User Process
    │
    │  malloc() / mmap()
    ▼
┌──────────────────┐
│  Virtual Memory  │  Per-process page tables
│  (mm_struct)     │
└────────┬─────────┘
         │ page fault
         ▼
┌──────────────────┐
│  Page Allocator  │  Buddy system (free_area[])
│  (Buddy System)  │
└────────┬─────────┘
         │
    ┌────┴────┐
    ▼         ▼
Page Cache   Anonymous
(file-backed) (heap/stack)
```

Key files:

```bash
/proc/meminfo           # System memory overview
/proc/buddyinfo         # Free pages per order (buddy allocator)
/proc/slabinfo          # Slab cache statistics
/proc/vmstat            # Virtual memory statistics
/proc/zoneinfo          # Memory zone details
/proc/<PID>/maps        # Process memory mappings
/proc/<PID>/smaps       # Detailed per-mapping memory usage
/sys/kernel/mm/         # Memory tuning knobs
```

### 3. Virtual File System (VFS)

Provides a unified interface to all filesystems.

| Component | Purpose |
|-----------|---------|
| VFS Layer | Abstract interface for all filesystem operations |
| Inode Cache | In-memory representation of file metadata |
| Dentry Cache | Directory entry cache (path lookups) |
| Page Cache | File data cached in memory |
| Filesystem Drivers | ext4, XFS, btrfs, NFS, tmpfs, proc, sysfs |
| Bio/Block Layer | I/O request management |

```
User Space:  open() → read() → write() → close()
                │
                ▼
┌──────────────────────────────────┐
│           VFS Layer              │
│ (superblock, inode, dentry, file)│
├──────────────────────────────────┤
│    Filesystem Implementation     │
│ ext4 │ xfs │ btrfs │ nfs │ tmpfs │
├──────────────────────────────────┤
│        Page Cache                │
├──────────────────────────────────┤
│        Block Layer               │
│ (I/O scheduler, merging, plug)   │
├──────────────────────────────────┤
│     Device Drivers (SCSI/NVMe)   │
├──────────────────────────────────┤
│         Hardware (Disk)          │
└──────────────────────────────────┘
```

Key files:

```bash
/proc/filesystems       # Registered filesystems
/proc/mounts            # Mounted filesystems
/proc/diskstats         # Disk I/O statistics
/sys/block/             # Block device attributes
/sys/fs/                # Filesystem-specific tuning
```

### 4. Network Stack

Implements networking protocols and socket interface.

| Layer | Components |
|-------|-----------|
| Socket Layer | socket(), bind(), connect(), send(), recv() |
| Transport | TCP, UDP, SCTP, DCCP |
| Network | IPv4, IPv6, routing, netfilter (iptables/nftables) |
| Link | ARP, Ethernet, bridging, bonding, VLAN |
| Device Drivers | NIC drivers (e1000e, ixgbe, mlx5, virtio-net) |
| Packet Processing | NAPI, GRO, GSO, XDP, tc (traffic control) |

```
Application
    │
    │ send()/recv()
    ▼
┌────────────────┐
│  Socket Layer  │  struct socket, struct sock
└───────┬────────┘
        ▼
┌────────────────┐
│   Transport    │  TCP (congestion, retransmit) / UDP
└───────┬────────┘
        ▼
┌────────────────┐
│    Network     │  IP routing, netfilter, fragmentation
└───────┬────────┘
        ▼
┌────────────────┐
│  Link Layer    │  ARP, Ethernet framing, queuing discipline
└───────┬────────┘
        ▼
┌────────────────┐
│  NIC Driver    │  DMA ring buffers, interrupts, NAPI
└───────┬────────┘
        ▼
     Hardware
```

Key files:

```bash
/proc/net/              # Network statistics
/proc/net/tcp           # TCP connections
/proc/net/udp           # UDP sockets
/proc/net/dev           # Network interface stats
/proc/net/route         # Routing table
/proc/sys/net/          # Network tuning (sysctl)
/sys/class/net/         # Network interface attributes
```

### 5. Device Driver Framework

Manages communication between the kernel and hardware.

| Type | Description | Examples |
|------|-------------|----------|
| Character | Byte-stream access, no buffering | `/dev/tty`, `/dev/random`, `/dev/null` |
| Block | Fixed-size block access, buffered | `/dev/sda`, `/dev/nvme0n1` |
| Network | Packet-based, no device file | `eth0`, `wlan0` |
| Platform | SoC-integrated peripherals | I2C, SPI, GPIO |
| USB | Universal Serial Bus devices | Storage, HID, audio |
| PCI/PCIe | Peripheral Component Interconnect | GPUs, NICs, NVMe |

```bash
/dev/                   # Device files
/sys/devices/           # Device hierarchy (real topology)
/sys/bus/               # Bus types (pci, usb, scsi)
/sys/class/             # Device classes (net, block, tty)
/proc/devices           # Registered char/block drivers
/proc/interrupts        # IRQ assignments
/proc/ioports           # I/O port allocations
/proc/iomem             # Memory-mapped I/O regions
```

### 6. Security Framework

Controls access and enforces policies.

| Component | Purpose |
|-----------|---------|
| DAC | Discretionary Access Control (traditional Unix permissions) |
| Capabilities | Fine-grained root privilege decomposition |
| SELinux | Mandatory Access Control (label-based, RHEL default) |
| AppArmor | Mandatory Access Control (path-based, Ubuntu default) |
| Seccomp | System call filtering (used by containers) |
| LSM | Linux Security Module framework (hooks for all above) |
| Audit | Kernel audit subsystem (auditd) |
| Keyring | Kernel key management (encryption keys, auth tokens) |

```bash
/sys/kernel/security/   # Security module interfaces
/proc/<PID>/attr/       # Process security attributes
/proc/<PID>/status      # Shows seccomp and capabilities
/sys/fs/selinux/        # SELinux interface
/etc/selinux/           # SELinux configuration
/etc/apparmor.d/        # AppArmor profiles
```

### 7. Inter-Process Communication (IPC)

| Mechanism | Description | Scope |
|-----------|-------------|-------|
| Pipes | Unidirectional byte stream | Parent-child |
| Named Pipes (FIFO) | Pipe with a filesystem name | Any processes |
| Unix Sockets | Bidirectional communication | Same host |
| Signals | Asynchronous notifications | Any process |
| Shared Memory (SHM) | Direct memory sharing | Any process |
| Semaphores | Synchronization primitives | Any process |
| Message Queues | Structured message passing | Any process |
| Futex | Fast userspace mutex | Threads |
| eventfd | Event notification | Threads/processes |

```bash
/proc/sysvipc/shm       # System V shared memory
/proc/sysvipc/sem       # System V semaphores
/proc/sysvipc/msg       # System V message queues
/dev/shm/               # POSIX shared memory
```

## Kernel Source Tree Layout

```
linux/
├── arch/               # Architecture-specific code (x86, arm64, riscv)
├── block/              # Block I/O layer
├── crypto/             # Cryptographic algorithms
├── Documentation/      # Kernel documentation
├── drivers/            # Device drivers (largest directory)
│   ├── block/          # Block device drivers
│   ├── char/           # Character device drivers
│   ├── gpu/            # GPU drivers (DRM)
│   ├── net/            # Network drivers
│   ├── nvme/           # NVMe drivers
│   ├── scsi/           # SCSI drivers
│   └── usb/            # USB drivers
├── fs/                 # Filesystem implementations
│   ├── ext4/           # ext4 filesystem
│   ├── xfs/            # XFS filesystem
│   ├── btrfs/          # btrfs filesystem
│   ├── nfs/            # NFS client
│   ├── proc/           # /proc filesystem
│   └── sysfs/          # /sys filesystem
├── include/            # Header files
├── init/               # Kernel initialization (main.c → start_kernel)
├── ipc/                # Inter-process communication
├── kernel/             # Core kernel (scheduler, signals, time, cgroups)
│   ├── sched/          # Scheduler implementation
│   ├── cgroup/         # Control groups
│   └── trace/          # Tracing infrastructure (ftrace)
├── lib/                # Helper library routines
├── mm/                 # Memory management
├── net/                # Networking stack
│   ├── core/           # Socket layer, skbuff management
│   ├── ipv4/           # IPv4 implementation
│   ├── ipv6/           # IPv6 implementation
│   ├── netfilter/      # Packet filtering (iptables/nftables)
│   └── sched/          # Traffic control (tc)
├── scripts/            # Build scripts and tools
├── security/           # Security modules (SELinux, AppArmor)
├── sound/              # Audio subsystem (ALSA)
└── tools/              # Userspace tools (perf, bpf)
```

## The Boot Process

```
Hardware Power On
    │
    ▼
┌──────────┐
│ Firmware │  BIOS / UEFI
└────┬─────┘
     ▼
┌──────────┐
│Bootloader│  GRUB2 (loads kernel + initramfs)
└────┬─────┘
     ▼
┌──────────────────────────────────┐
│  Kernel Initialization           │
│  start_kernel()                  │
│  ├── setup_arch()                │  Architecture setup
│  ├── mm_init()                   │  Memory management init
│  ├── sched_init()                │  Scheduler init
│  ├── vfs_caches_init()           │  VFS/filesystem init
│  ├── rest_init()                 │  Start kernel threads
│  │   ├── kernel_init()           │  → PID 1
│  │   └── kthreadd()              │  → PID 2
│  └── cpu_idle()                  │  Idle loop
└──────────────────────────────────┘
     │
     ▼
┌──────────┐
│ initramfs│  Early userspace (load drivers, find root)
└────┬─────┘
     ▼
┌──────────┐
│  init    │  PID 1 (systemd / init)
└────┬─────┘
     ▼
  User Space
```

## Key Kernel Data Structures

| Structure | Purpose |
|-----------|---------|
| `task_struct` | Process descriptor (one per process/thread) |
| `mm_struct` | Memory descriptor (address space) |
| `vm_area_struct` | Virtual memory area (one per mapping) |
| `inode` | File metadata on disk |
| `dentry` | Directory entry (path component cache) |
| `file` | Open file descriptor |
| `super_block` | Filesystem instance |
| `sk_buff` | Network packet buffer |
| `bio` | Block I/O request |
| `page` | Physical memory page descriptor |
| `cred` | Process credentials (UID, GID, capabilities) |

## Kernel Interfaces (/proc, /sys, /dev)

### /proc — Process and Kernel Information

```bash
/proc/PID/              # Per-process information
/proc/cpuinfo           # CPU details
/proc/meminfo           # Memory statistics
/proc/interrupts        # IRQ counters
/proc/loadavg           # Load average
/proc/uptime            # System uptime
/proc/version           # Kernel version
/proc/cmdline           # Kernel boot parameters
/proc/modules           # Loaded modules
/proc/filesystems       # Available filesystems
/proc/mounts            # Mount table
/proc/sys/              # Tunable parameters (sysctl)
```

### /sys — Kernel Object Model (sysfs)

```bash
/sys/block/             # Block devices
/sys/bus/               # Bus types and devices
/sys/class/             # Device classes
/sys/devices/           # Device tree (physical topology)
/sys/firmware/          # Firmware interfaces
/sys/fs/                # Filesystem parameters
/sys/kernel/            # Kernel subsystems
/sys/module/            # Loaded modules and parameters
/sys/power/             # Power management
```

### /dev — Device Files

```bash
/dev/null               # Discard all writes
/dev/zero               # Infinite zero bytes
/dev/random             # Cryptographic random
/dev/urandom            # Non-blocking random
/dev/sda, /dev/nvme0n1  # Block devices
/dev/tty, /dev/pts/     # Terminals
/dev/shm/               # Shared memory (tmpfs)
/dev/loop*              # Loop devices
```

## Kernel Modules

Modules extend the kernel at runtime without recompiling or rebooting.

```bash
# List loaded modules
lsmod

# Module information
modinfo <module_name>

# Load a module
modprobe <module_name>

# Remove a module
modprobe -r <module_name>

# Module parameters
cat /sys/module/<module_name>/parameters/*

# Module dependencies
modprobe --show-depends <module_name>

# All available modules
find /lib/modules/$(uname -r) -name '*.ko*'
```

## Tracing and Debugging

| Tool | Purpose |
|------|---------|
| `ftrace` | Kernel function tracer (built-in) |
| `perf` | Performance analysis and tracing |
| `bpftrace` / `bcc` | eBPF-based dynamic tracing |
| `strace` | System call tracer (user ↔ kernel boundary) |
| `SystemTap` | Kernel instrumentation scripts |
| `crash` | Analyze kernel crash dumps |
| `dmesg` | Kernel ring buffer messages |
| `kdump` | Capture kernel memory on crash |

```bash
# Kernel ring buffer
dmesg | tail -50

# Available tracers
cat /sys/kernel/debug/tracing/available_tracers

# Function tracing (ftrace)
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
echo 0 > /sys/kernel/debug/tracing/tracing_on

# Trace specific function
echo 'vfs_read' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# perf — top kernel functions
perf top

# perf — record and report
perf record -g -a sleep 10
perf report
```

## Kernel Configuration

```bash
# View current kernel config
cat /boot/config-$(uname -r)
# Or
zcat /proc/config.gz  # if CONFIG_IKCONFIG_PROC=y

# Common config checks
grep CONFIG_PREEMPT /boot/config-$(uname -r)
grep CONFIG_HZ /boot/config-$(uname -r)
grep CONFIG_NUMA /boot/config-$(uname -r)
grep CONFIG_CGROUPS /boot/config-$(uname -r)

# Kernel version
uname -r
cat /proc/version
```

## Kernel Tuning (sysctl)

Major tuning categories under `/proc/sys/`:

| Path | Category |
|------|----------|
| `/proc/sys/kernel/` | Process limits, scheduler, panic behavior |
| `/proc/sys/vm/` | Memory management (swappiness, dirty ratios, OOM) |
| `/proc/sys/net/` | Network stack (buffers, congestion, connections) |
| `/proc/sys/fs/` | Filesystem limits (file-max, inotify, aio) |
| `/proc/sys/dev/` | Device-specific parameters |

```bash
# View all sysctl parameters
sysctl -a

# View specific parameter
sysctl vm.swappiness

# Set parameter (runtime)
sysctl -w vm.swappiness=10

# Persistent configuration
echo "vm.swappiness=10" >> /etc/sysctl.d/99-tuning.conf
sysctl -p /etc/sysctl.d/99-tuning.conf
```

## Kernel Version Numbering

```
5.14.0-362.24.1.el9_3.x86_64
│ │  │  │         │    │
│ │  │  │         │    └── Architecture
│ │  │  │         └─────── Distribution release
│ │  │  └───────────────── Distribution patch level
│ │  └──────────────────── Sublevel (stable patches)
│ └─────────────────────── Minor version
└───────────────────────── Major version
```

```bash
uname -r                # Full version string
uname -v                # Build date and number
cat /proc/version       # Full version with compiler info
```

## Quick Reference: Exploring the Kernel

```bash
# Kernel version
uname -r

# Loaded modules
lsmod

# Kernel parameters
sysctl -a | grep <parameter>

# Hardware interrupts
cat /proc/interrupts

# CPU info
lscpu
cat /proc/cpuinfo

# Memory layout
cat /proc/meminfo
cat /proc/buddyinfo

# Block devices
lsblk
cat /proc/diskstats

# Network
cat /proc/net/dev
ss -s

# Process tree
ps auxf
pstree -p

# Kernel messages
dmesg -T | tail

# System calls of a process
strace -p <PID>
cat /proc/<PID>/syscall
```
