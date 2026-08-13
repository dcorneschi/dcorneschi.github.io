# Securing Kubernetes Containers: A Deep Dive into Security Contexts

Understanding how Linux kernel primitives map to Kubernetes security settings, and how to apply them effectively in production.

---

## Linux Primitives Beneath Kubernetes

A container in Kubernetes is just a regular Linux process. It runs directly on the host's kernel, but it's isolated using namespaces (for things like networking, mounts, and process IDs) and cgroups (for resource limits). It also gets its own virtual filesystem, which is an isolated root directory.

There's no virtual machine here. It's just Linux, sandboxed.

That's why it really matters who the process runs as inside the container; especially if it's root. The `securityContext` field in a pod spec is a declarative way to configure how that process interacts with the kernel.

But Kubernetes doesn't enforce these settings itself. It passes them down to the container runtime, which translates them into kernel-level instructions at container start.

Every setting you apply in a pod spec ultimately maps to low-level kernel features like syscalls, namespaces, cgroups, or capabilities. To configure these effectively, you need to understand what each one does.

---

## Syscalls: The Kernel's API Surface

In Linux, user-space applications can't interact with hardware or kernel-managed resources directly. Every interaction must go through the kernel, and the only sanctioned way to do that is via system calls.

System calls are predefined entry points that expose core kernel functionality. You can think of system calls as the kernel's API — over 300 entry points grouped by what they do. Some handle files, some manage processes, others deal with networking or memory.

Take the `open` syscall — it needs a file path, an access mode like read or write, and permission bits:

```bash
man 2 open | grep -A 8 "SYNOPSIS"

SYNOPSIS
       #include <fcntl.h>

       int open(const char *pathname, int flags, ...
                  /* mode_t mode */ );
```

Here's how it looks when a real application makes these calls:

```bash
strace -e trace=openat ls /tmp 2>&1 | head -5

openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libselinux.so.1", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/tmp", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
```

Or the `socket` syscall for networking:

```bash
strace -e trace=socket,connect curl -s http://httpbin.org/ip 2>&1 | grep -E "(socket|connect)"

socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 5
connect(5, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("192.168.2.1")}, 16) = 0
```

This level of access gives user-space applications immense power but also exposes a large attack surface, especially when privileged capabilities are involved.

---

## Control Groups (cgroups): Enforcing Resource Limits

Control groups are kernel-level mechanisms that restrict how much CPU, memory, I/O, and other resources a process — or a group of processes — can consume.

Say you're running a JVM application. You can assign it to a cgroup that caps its memory at 256MB and restricts it to a single CPU core. Each cgroup defines a resource boundary. You control how much each process gets, and the kernel enforces it.

While cgroups enforce resource boundaries, they don't isolate what a process can see or interact with. For that, Linux relies on namespaces.

---

## Namespaces: Isolating What a Process Can See

While cgroups control how much a process can consume, namespaces control what a process can see. They determine which part of the system a process believes it's running in.

- With **network namespaces**, a process only sees its own interfaces and traffic.
- With a **mount namespace**, a process sees a private filesystem view.

As of Linux kernel 5.6, there are eight namespace types:

```bash
lsns

NS TYPE    NPROCS     PID USER
4026531834 time       17  1035
4026531835 cgroup     17  1035
4026531837 user       17  1035
4026531840 net        17  1035
4026532242 ipc        17  1035
4026532253 mnt        17  1035
4026532254 uts        17  1035
4026532255 pid        17  1035
```

Together with cgroups, namespaces form the backbone of container isolation.

---

## From Kernel Primitives to Developer Abstractions

Docker is a popular way to manage containers in a developer-friendly way. Instead of managing syscalls, cgroups, and namespaces manually, you define a container image, run a single command, and the runtime sets everything up behind the scenes.

At their core, container platforms are just orchestration layers on top of three key Linux kernel features: **Syscalls**, **Control Groups**, and **Namespaces**.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          Container Processes (user space)                           │
│                                                                                     │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐               │
│  │   Container A    │    │   Container B    │    │   Container C    │               │
│  │  nginx (UID 101) │    │ python (UID 1000)│    │  java (UID 1001) │               │
│  │  /var/www, eth0  │    │  /app, eth0      │    │  /opt/app, eth0  │               │
│  └────────┬─────────┘    └────────┬─────────┘    └────────┬─────────┘               │
│           │                       │                       │                         │
├───────────┼───────────────────────┼───────────────────────┼─────────────────────────┤
│           ▼                       ▼                       ▼                         │
│  Control Groups (cgroups) — limit how much a process can consume                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐    │
│  │CPU limit │ │Mem limit │ │I/O bw    │ │PIDs limit│ │Net class │ │Deviceaccess│    │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ └────────────┘    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  Namespaces — isolate what a process can see                                        │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌──────┐ ┌───────┐ ┌──────┐                │
│  │ PID │ │ NET │ │ MNT │ │ UTS │ │ IPC │ │ USER │ │CGROUP │ │ TIME │                │
│  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └──────┘ └───────┘ └──────┘                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  Syscall boundary (open, read, write, socket, clone, mount, ...)                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                              Linux Kernel                                           │
│  Syscall Interface  │  VFS / Block Layer  │  Network Stack  │  Memory Management    │
└─────────────────────────────────────────────────────────────────────────────────────┘

Each container gets its own set of namespaces and cgroup limits.
The kernel enforces isolation — no VM needed.
```

---

## Linux User IDs and Permissions

In Linux, every process runs with a User ID, or UID. The UID is the kernel's primary access control mechanism. It defines ownership and tells what the process can access or modify.

```bash
grep UID_ /etc/login.defs

UID_MIN                  1000
UID_MAX                 60000
#SYS_UID_MIN              100
#SYS_UID_MAX              999
```

By convention:
- **UID 0** means root — full control over everything
- **UIDs 1-999** are usually for system services
- **UIDs 1000+** are assigned to regular users and applications

Each time a process attempts a privileged operation, the Linux kernel checks its UID to decide whether that action should be allowed.

In containerized environments, this creates a subtle risk. Many containers still run as UID 0. If that UID maps directly to the host, a compromised container could gain real root access on the node.

---

## User Namespaces and UID Mapping in Containers

The user namespace lets you remap UIDs inside a container to unprivileged UIDs on the host. A process can run as UID 0 (root) inside the container, while the kernel maps it to an unprivileged UID like 2000 outside.

From the container's perspective, the process is root. But from the host's perspective, it's just a regular user with no elevated privileges.

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         UID Mapping: Host vs Container                              │
├─────────────────────────────────┬───────────────────────────────────────────────────┤
│                                 │                                                   │
│  WITHOUT User Namespace         │  WITH User Namespace                              │
│  (default behavior)             │  (UID remapping enabled)                          │
│                                 │                                                   │
│  ┌───────────────────────────┐  │  ┌───────────────────────────┐                    │
│  │ Container                 │  │  │ Container                 │                    │
│  │  app runs as UID 0        │  │  │  app runs as UID 0        │                    │
│  │  (root inside)            │  │  │  (thinks it's root)       │                    │
│  └─────────────┬─────────────┘  │  └─────────────┬─────────────┘                    │
│                │                │                │                                  │
│                │ UID 0 = UID 0  │                │ UID 0 → UID 100000               │
│                ▼                │                ▼                                  │
│  ┌───────────────────────────┐  │  ┌───────────────────────────┐                    │
│  │ Host                      │  │  │ Host                      │                    │
│  │  REAL ROOT on host!       │  │  │  Unprivileged user        │                    │
│  │  ⚠ Container escape =     │  │  │  ✓ Container escape =     │                    │
│  │    full host compromise   │  │  │    limited damage         │                    │ 
│  └───────────────────────────┘  │  └───────────────────────────┘                    │
│                                 │                                                   │
├─────────────────────────────────┴───────────────────────────────────────────────────┤
│                                                                                     │
│  Kubernetes securityContext settings:                                               │
│                                                                                     │
│    runAsUser: 1000        → Forces specific non-root UID                            │
│    runAsNonRoot: true     → Blocks UID 0 at admission time                          │
│    runAsGroup: 1000       → Sets primary GID                                        │
│    fsGroup: 2000          → Volume GID ownership                                    │
│                                                                                     │
│  UIDs are numeric — names (/etc/passwd) are cosmetic.                               │
│  The kernel only checks the number.                                                 │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## How Docker Manages UIDs

By default, the process runs with the UID defined in the container image — usually UID 0. Unless user namespaces are explicitly enabled, UID 0 inside the container is also UID 0 on the host.

If the container is compromised, the attacker gains root-level privileges over any accessible host resources. This is why running containers as a non-root user is a critical best practice.

---

## runAsUser and runAsNonRoot

Kubernetes builds on these kernel-level security controls using SecurityContext.

### runAsUser

Sets the exact UID for the process:

```yaml
spec:
  securityContext:
    runAsUser: 1000
```

This means the container will start as UID 1000 instead of root. This setting applies to all containers in the Pod unless overridden.

### runAsNonRoot

Makes sure the container never runs as root:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
```

If the container tries to start as UID 0, Kubernetes blocks it.

### Combining both

```yaml
spec:
  securityContext:
    runAsUser: 1000
    runAsNonRoot: true
```

This ensures the process runs as a specific non-root UID, and any future changes to the image or spec won't accidentally elevate privileges.

### Practical example

The Node.js official image includes a non-root user:

```bash
docker run --rm -it node:20-slim cat /etc/passwd
node:x:1000:1000::/home/node:/bin/bash
```

However, it still runs as root by default:

```bash
docker run --rm node:20-slim id
uid=0(root) gid=0(root) groups=0(root)
```

To enforce non-root execution in Kubernetes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-nonroot
spec:
  securityContext:
    runAsUser: 1000
  containers:
    - name: node
      image: node:20-slim
      command: ["sleep", "3600"]
```

Verify:

```bash
kubectl exec node-nonroot -- id
uid=1000(node) gid=1000(node) groups=1000(node)
```

---

## Linux Capabilities

Capabilities define which privileged operations a process is allowed to perform. Rather than granting full root access, the Linux kernel splits root privileges into fine-grained units like `CHOWN`, `NET_ADMIN`, and `SYS_PTRACE`.

```
Linux Capabilities: From Privileged to Fully Locked Down

  DANGEROUS ◄─────────────────────────────────────────────────────────► SECURE
  ██████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐
  │privileged:   │  │  Docker      │  │  Selective   │  │  drop: ALL   │  │   Full     │
  │   true       │  │  Defaults    │  │    Add       │  │  (no add)    │  │  Lockdown  │
  ├──────────────┤  ├──────────────┤  ├──────────────┤  ├──────────────┤  ├────────────┤
  │~38 caps (all)│  │ 14 caps      │  │ 1-3 caps     │  │ 0 caps       │  │ 0 caps     │
  │              │  │              │  │              │  │              │  │ + MAC      │
  │Host devices  │  │CHOWN         │  │drop: ALL     │  │No port <1024 │  │ + seccomp  │
  │/proc, /sys   │  │DAC_OVERRIDE  │  │add:          │  │No chown      │  │ + nonroot  │
  │Mount anything│  │NET_BIND_SVC  │  │ NET_BIND_SVC │  │No raw sockets│  │ + RO rootfs│
  │Load modules  │  │NET_RAW       │  │ or SYS_PTRACE│  │No ptrace     │  │ + no-new-  │
  │No seccomp    │  │SETUID/SETGID │  │ or CHOWN     │  │No mount      │  │   privs    │
  │              │  │KILL, MKNOD   │  │              │  │              │  │            │
  │              │  │FOWNER, FSETID│  │              │  │              │  │            │
  ├──────────────┤  ├──────────────┤  ├──────────────┤  ├──────────────┤  ├────────────┤
  │ ⚠ NEVER use  │  │ Too many for │  │ Right-sized  │  │ Most apps    │  │ PSS        │
  │ unless CNI/  │  │ most apps    │  │ privileges   │  │ work here    │  │ Restricted │
  │ storage drv  │  │              │  │              │  │              │  │            │
  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘

  Start with drop: ALL. If the container fails, use strace to find which
  capability is missing, then add only that one.
```

### Example: Adding only CHOWN

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-cap-chown
spec:
  containers:
  - name: node
    image: node:20-slim
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 0
      capabilities:
        drop: ["ALL"]
        add: ["CHOWN"]
```

We explicitly drop all capabilities first, then add back only CHOWN. Verify:

```bash
kubectl exec node-cap-chown -- sh -c "touch /tmp/test && chown 1234:1234 /tmp/test && ls -l /tmp/test"
-rw-r--r-- 1 1234 1234 0 Jul  5 08:53 /tmp/test
```

Other privileged operations fail:

```bash
kubectl exec node-cap-chown -- mount -t tmpfs tmpfs /mnt
mount: /mnt: permission denied.

kubectl exec node-cap-chown -- date -s "2024-01-01"
date: cannot set date: Operation not permitted
```

Confirm at the kernel level:

```bash
kubectl exec node-cap-chown -- cat /proc/1/status | grep CapEff
CapEff: 0000000000000001
```

Decoded: `cap_chown` only.

**Best practice:** Start with `drop: ["ALL"]`, then add capabilities back one by one until the container works.

---

## allowPrivilegeEscalation

Some binaries are designed to gain more privileges after startup (e.g., setuid binaries). Setting `allowPrivilegeEscalation: false` tells the container runtime to set the `no_new_privs` flag on the process at launch.

```bash
cat /proc/1/status | grep NoNewPrivs
NoNewPrivs: 1
```

### Demonstration

With `allowPrivilegeEscalation: true`:

```bash
kubectl logs privilege-escalation

Current user: nobody
Current UID: 65534
Attempting sudo whoami...
root
```

With `allowPrivilegeEscalation: false`:

```bash
kubectl logs privilege-escalation-false

Current user: nobody
Current UID: 65534
Attempting sudo whoami...
sudo: The "no new privileges" flag is set, which prevents sudo from running as root.
sudo exit status: 1
```

The kernel blocks the privilege escalation attempt even though sudo is installed and configured.

### Recommended baseline

```yaml
securityContext:
  runAsUser: 1000
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
    add: ["CHOWN"]
```

---

## Privileged Containers

Setting `privileged: true` removes nearly all security boundaries between the container and the host:

```yaml
spec:
  containers:
    securityContext:
      privileged: true  # Never do this unless absolutely necessary
```

A privileged container:
- Has access to all host devices
- Can see all processes on the host
- Bypasses most security restrictions
- Can manipulate cgroup settings
- Can mount sensitive filesystems like `/proc`, `/sys`, or the host root

**Legitimate use cases are extremely rare:** container runtime components, network plugins that configure host networking, storage drivers that manage host volumes.

If you believe your workload needs `privileged: true`, first validate whether granting a specific capability solves the problem.

---

## readOnlyRootFilesystem

Most containers don't need to write to their own filesystem after startup. Setting `readOnlyRootFilesystem: true` passes a read-only flag to the container runtime.

```yaml
spec:
  containers:
    securityContext:
      readOnlyRootFilesystem: true
```

Under the hood, the runtime uses the `mount()` syscall with the `MS_RDONLY` flag. The kernel marks this mount with `MNT_RDONLY`, and the VFS layer rejects any write operations (`write()`, `mkdir()`, `unlink()`).

For applications that need writable paths (logs, tmp files):

```yaml
    volumeMounts:
      - name: tmp
        mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```

---

## Seccomp Profiles

Seccomp (secure computing mode) provides syscall-level control by attaching a BPF program directly to a process. Each time the process invokes a system call, the kernel intercepts it and the filter can:

- **Allow** the call
- **Block** it with an EPERM
- **Terminate** the process with a SIGSYS
- **Log** the attempt for auditing

### RuntimeDefault profile

```yaml
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
```

The RuntimeDefault profile allows only a minimal set of system calls. Blocked syscalls include: `mount/umount`, `ptrace`, `reboot`, `setns`, `keyctl`.

### Custom profiles

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "stat", "fstat"],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": ["socket", "connect", "accept"],
      "action": "SCMP_ACT_ALLOW",
      "args": [
        {
          "index": 0,
          "value": 2,
          "op": "SCMP_CMP_EQ"
        }
      ]
    }
  ]
}
```

Reference it in your pod:

```yaml
securityContext:
  seccompProfile:
    type: Localhost
    localhostProfile: profiles/app-specific.json
```

### Most restrictive configuration

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  capabilities:
    drop: ["ALL"]
```

If your application fails, use `strace` to trace syscall activity:

```bash
kubectl debug -it your-pod --image=nicolaka/netshoot -- strace -f -e trace=all your-command
```

---

## AppArmor

AppArmor provides another layer of access control through path-based access control profiles. Unlike seccomp which filters system calls, AppArmor focuses on file system access, network access, and capability usage.

```yaml
spec:
  securityContext:
    appArmorProfile:
      type: RuntimeDefault
```

AppArmor builds on capabilities by further restricting how operations interact with the filesystem and system objects. For example, AppArmor may allow a program to read `/etc/passwd` but not `/etc/shadow`.

---

## fsGroup and supplementalGroups

Even with a non-root user, file access can become a problem with mounted volumes. Kubernetes provides `fsGroup` to solve this:

```yaml
spec:
  securityContext:
    fsGroup: 2000
```

When `fsGroup` is set, the kubelet:
1. Recursively changes the group ownership (GID) of the mounted volume's files to match the provided fsGroup
2. Ensures new files created by the container inherit that group ID

This only affects mounted volumes — not the image's internal filesystem.

### fsGroupChangePolicy

By default, Kubernetes recursively `chown`s the entire volume on every pod start. For large volumes with millions of files, this can add minutes to startup time. Use `fsGroupChangePolicy` to control this:

```yaml
spec:
  securityContext:
    fsGroup: 2000
    fsGroupChangePolicy: "OnRootMismatch"
```

| Policy | Behavior |
|--------|----------|
| `Always` (default) | Recursively change ownership on every pod start |
| `OnRootMismatch` | Only change ownership if the root directory GID doesn't match fsGroup |

`OnRootMismatch` dramatically speeds up pod startup for persistent volumes that are reused across restarts.

For multiple groups:

```yaml
securityContext:
  supplementalGroups: [2000, 3000]
```

---

## How fsGroup Works with CSI Volumes

CSI drivers report their fsGroup support through capabilities and implement one of three policies:

| Policy | Behavior |
|--------|----------|
| **None** | Driver doesn't support permission modifications |
| **File** | Kubelet recursively changes ownership (like in-tree volumes) |
| **ReadWriteOnceWithFSType** | Only applies changes when the volume has a specified filesystem type and uses ReadWriteOnce |

This flexible approach solves:
- Performance issues from recursive permission changes on large volumes
- Compatibility with distributed storage systems
- Enterprise storage with specialized security requirements

---

## Live Debugging: Diagnosing SecurityContext Failures

### Example 1: Legacy Monitoring Tool

A monitoring agent needs low-level access to inspect processes and collect network metrics:

```yaml
securityContext:
  runAsUser: 1000
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
    add: ["SYS_PTRACE", "NET_ADMIN"]
```

### Example 2: Backup process that needs chown

```yaml
securityContext:
  runAsUser: 1001
  runAsNonRoot: true
  capabilities:
    drop: ["ALL"]
    add: ["CHOWN"]
```

### Example 3: Read-Only containers for public-facing apps

```yaml
securityContext:
  runAsUser: 1000
  readOnlyRootFilesystem: true
```

An emptyDir volume mounted to `/tmp` handles temporary files.

---

## Troubleshooting: When Security Context Conflicts with Container Behavior

A common scenario: deploying a Node.js application that needs to write logs, but the pod crashes with permission errors.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodejs-app
spec:
  securityContext:
    runAsUser: 1000
    runAsNonRoot: true
    fsGroup: 2000
  containers:
  - name: app
    image: node:18
    command: ["node"]
    args:
      - -e
      - |
        const fs = require('fs');
        const path = '/var/log/app.log';
        fs.writeFileSync(path, 'Starting app');
        console.log('App started');
```

**Error:**

```
Error: EACCES: permission denied, open '/var/log/app.log'
```

**Root cause:** `/var/log` is owned by root and not writable by our user.

### Solution: Use a writable volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodejs-fixed-volume
spec:
  securityContext:
    runAsUser: 1000
    runAsNonRoot: true
    fsGroup: 2000
  volumes:
    - name: logs
      emptyDir: {}
  containers:
    - name: app
      image: node:18
      volumeMounts:
        - name: logs
          mountPath: /var/log
```

The `fsGroup` ensures the volume is writable by the container.

### Production-ready configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodejs-production
spec:
  securityContext:
    runAsUser: 1000
    runAsNonRoot: true
    fsGroup: 2000
    fsGroupChangePolicy: "OnRootMismatch"
  volumes:
    - name: app-logs
      emptyDir: {}
    - name: tmp
      emptyDir: {}
  containers:
    - name: app
      image: node:18
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - name: app-logs
          mountPath: /logs
        - name: tmp
          mountPath: /tmp
      env:
        - name: NODE_ENV
          value: "production"
      command: ["node"]
      args:
        - -e
        - |
          const fs = require('fs');
          const logFile = '/logs/app.log';
          fs.writeFileSync(logFile, `App started at ${new Date().toISOString()}\n`);
          console.log('Application running securely!');
          setInterval(() => {
            fs.appendFileSync(logFile, `Heartbeat at ${new Date().toISOString()}\n`);
          }, 5000);
```

---

## Pod Security Standards and Pod Security Admission

Individual `securityContext` settings secure a single pod, but they don't prevent someone from deploying an insecure pod in the first place. Pod Security Standards (PSS) define cluster-wide policies that enforce minimum security baselines across namespaces.

Kubernetes ships with three built-in profiles:

| Profile | Purpose |
|---------|---------|
| **Privileged** | Unrestricted. No security constraints applied. Used for system-level workloads like CNI plugins and storage drivers. |
| **Baseline** | Prevents known privilege escalations. Blocks `hostNetwork`, `hostPID`, privileged containers, and dangerous capabilities while still being broadly compatible with most workloads. |
| **Restricted** | Maximum hardening. Requires non-root, drops all capabilities, enforces read-only root filesystem, and mandates seccomp profiles. |

Pod Security Admission (PSA) is the built-in admission controller that enforces these standards. It replaced the deprecated PodSecurityPolicy (PSP) starting in Kubernetes 1.25.

### Applying PSA to a namespace

You apply enforcement through namespace labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

The three modes work independently:

- **enforce** — Rejects pods that violate the policy
- **warn** — Allows the pod but displays a warning to the user
- **audit** — Allows the pod but logs the violation in the audit log

A common migration strategy is to start with `warn` and `audit` on your target profile while keeping `enforce` at a lower level. Once you've resolved all warnings, promote `enforce` to the stricter profile.

### Checking violations before enforcement

You can dry-run a policy against existing pods:

```bash
kubectl label --dry-run=server --overwrite ns production \
  pod-security.kubernetes.io/enforce=restricted
```

This shows which pods would be rejected without actually blocking anything.

### Exemptions

Some workloads legitimately need elevated privileges (monitoring agents, log collectors, CSI drivers). PSA supports exemptions by username, namespace, or runtime class through the admission controller configuration:

```yaml
apiVersion: pod-security.admission.config.k8s.io/v1
kind: PodSecurityConfiguration
exemptions:
  usernames: []
  runtimeClasses: []
  namespaces: [kube-system, monitoring]
```

PSA provides the enforcement layer. SecurityContext provides the configuration. Together they form a defense-in-depth strategy: PSA prevents insecure pods from being admitted, while securityContext defines what "secure" looks like for each workload.

---

## SELinux

SELinux (Security-Enhanced Linux) is a mandatory access control (MAC) system that assigns security labels to every process, file, and system object. Unlike discretionary access control (DAC) where file owners set permissions, SELinux policies are enforced by the kernel regardless of user permissions.

Where AppArmor uses path-based rules (e.g., "this process can read `/etc/passwd`"), SELinux uses label-based rules (e.g., "processes with label `httpd_t` can read files with label `httpd_content_t`").

### SELinux in Kubernetes

Kubernetes exposes SELinux configuration through the `seLinuxOptions` field:

```yaml
spec:
  securityContext:
    seLinuxOptions:
      level: "s0:c123,c456"
  containers:
  - name: app
    image: nginx
    securityContext:
      seLinuxOptions:
        level: "s0:c123,c456"
        type: "container_t"
```

The key fields are:

| Field | Purpose |
|-------|---------|
| **user** | SELinux user (e.g., `system_u`) |
| **role** | SELinux role (e.g., `system_r`) |
| **type** | SELinux type/domain (e.g., `container_t`) — this is the most commonly configured field |
| **level** | Multi-Category Security (MCS) label for isolating containers from each other |

### How container runtimes use SELinux

On SELinux-enabled nodes (common on RHEL, CentOS, Fedora), the container runtime automatically assigns a unique MCS label to each container. This means:

- Container A with label `s0:c100,c200` cannot read files labeled `s0:c300,c400` belonging to Container B
- Even if both containers run as root, SELinux prevents cross-container file access
- Host files without a matching label are also inaccessible

You can check a container's SELinux context:

```bash
kubectl exec my-pod -- cat /proc/1/attr/current
system_u:system_r:container_t:s0:c200,c500
```

### AppArmor vs SELinux

| Aspect | AppArmor | SELinux |
|--------|----------|---------|
| Approach | Path-based rules | Label-based rules |
| Complexity | Simpler to write and debug | More powerful but steeper learning curve |
| Default on | Ubuntu, Debian, SUSE | RHEL, CentOS, Fedora |
| Container isolation | Per-profile restrictions | Automatic MCS label isolation |
| Kubernetes support | `appArmorProfile` field | `seLinuxOptions` field |

In practice, you use whichever your node OS ships with. Most managed Kubernetes offerings handle this transparently — GKE uses AppArmor, OpenShift uses SELinux.

### SELinux volume relabeling

When SELinux is active, volume mounts need compatible labels. Kubernetes handles this automatically for most volume types by relabeling the volume content to match the container's SELinux label.

Starting in Kubernetes 1.27+, you can control this with `seLinuxChangePolicy`:

```yaml
spec:
  securityContext:
    seLinuxChangePolicy: MountOption
```

This uses mount-level relabeling instead of recursive `chcon`, which is significantly faster for large volumes.

---

## Quick Reference: securityContext Fields

```
Security Context Decision Flowchart
════════════════════════════════════

                    ┌─────────────────────┐
                    │  New Pod Deployment │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Does the app need   │──── YES ──▶ Reconsider: use gosu/su-exec
                    │ to run as root?     │            pattern instead?
                    └──────────┬──────────┘
                               │ NO
                               ▼
              ┌────────────────────────────────┐
              │  Set: runAsNonRoot: true       │
              │       runAsUser: 1000          │
              └───────────────┬────────────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │ Binds to port       │──── YES ──▶ Add: NET_BIND_SERVICE
                   │ < 1024?             │            or remap to high port
                   └──────────┬──────────┘
                              │ NO
                              ▼
                   ┌─────────────────────┐
                   │ Writes to container │──── YES ──▶ Mount emptyDir/tmpfs
                   │ root filesystem?    │            for writable paths
                   └──────────┬──────────┘
                              │
                              ▼
              ┌────────────────────────────────┐
              │  Set: readOnlyRootFilesystem:  │
              │       true                     │
              └───────────────┬────────────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │ Needs specific      │──── YES ──▶ drop: ALL
                   │ Linux capabilities? │            add: [only what's needed]
                   └──────────┬──────────┘
                              │ NO
                              ▼
              ┌────────────────────────────────┐
              │  Set: capabilities:            │
              │       drop: ["ALL"]            │
              └───────────────┬────────────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │ Mounts PVC /        │──── YES ──▶ Set: fsGroup: <GID>
                   │ hostPath volumes?   │            fsGroupChangePolicy: OnRootMismatch
                   └──────────┬──────────┘
                              │
                              ▼
              ┌────────────────────────────────┐
              │  Always apply:                 │
              │    allowPrivilegeEscalation:   │
              │      false                     │
              │    seccompProfile:             │
              │      type: RuntimeDefault      │
              └───────────────┬────────────────┘
                              │
                              ▼
              ╔════════════════════════════════╗
              ║  Hardened Pod Ready for        ║
              ║  Production                    ║
              ╚════════════════════════════════╝
```

A consolidated cheat sheet of all securityContext fields and where they apply.

### Pod-level securityContext

These apply to all containers in the pod unless overridden at container level.

| Field | Type | Effect |
|-------|------|--------|
| `runAsUser` | int | UID for all containers |
| `runAsGroup` | int | Primary GID for all containers |
| `runAsNonRoot` | bool | Block pod if any container would run as UID 0 |
| `fsGroup` | int | GID applied to mounted volumes |
| `fsGroupChangePolicy` | string | `Always` or `OnRootMismatch` — controls when fsGroup ownership is applied |
| `supplementalGroups` | []int | Additional GIDs added to the process |
| `seLinuxOptions` | object | SELinux labels for the pod |
| `seccompProfile` | object | Seccomp profile (`RuntimeDefault`, `Localhost`, `Unconfined`) |
| `appArmorProfile` | object | AppArmor profile (`RuntimeDefault`, `Localhost`, `Unconfined`) |
| `sysctls` | []object | Kernel parameters to set for the pod's network namespace |

### Container-level securityContext

These override pod-level settings for a specific container.

| Field | Type | Effect |
|-------|------|--------|
| `runAsUser` | int | UID for this container |
| `runAsGroup` | int | Primary GID for this container |
| `runAsNonRoot` | bool | Block container if it would run as UID 0 |
| `readOnlyRootFilesystem` | bool | Mount root filesystem as read-only |
| `allowPrivilegeEscalation` | bool | Allow process to gain privileges via setuid/setgid |
| `privileged` | bool | Run with all capabilities and no isolation |
| `capabilities` | object | `add` / `drop` Linux capabilities |
| `seLinuxOptions` | object | SELinux labels for this container |
| `seccompProfile` | object | Seccomp profile for this container |
| `appArmorProfile` | object | AppArmor profile for this container |
| `procMount` | string | Type of proc mount (`Default` or `Unmasked`) |

### Hardened baseline template

Copy-paste starting point for production workloads:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-app
spec:
  securityContext:
    runAsUser: 10000
    runAsGroup: 10000
    runAsNonRoot: true
    fsGroup: 10000
    fsGroupChangePolicy: "OnRootMismatch"
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: my-app:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache
  volumes:
  - name: tmp
    emptyDir:
      sizeLimit: 100Mi
  - name: cache
    emptyDir:
      sizeLimit: 200Mi
```

This template satisfies the PSS **Restricted** profile and works with most stateless applications out of the box.

---

## Common Linux Capabilities Reference

The most frequently needed capabilities for containerized workloads:

| Capability | What it allows | Common use case |
|-----------|---------------|-----------------|
| `NET_BIND_SERVICE` | Bind to ports below 1024 | Web servers on port 80/443 |
| `NET_ADMIN` | Modify routing tables, configure interfaces | Service meshes, CNI plugins |
| `NET_RAW` | Use RAW/PACKET sockets | `ping`, network diagnostics |
| `SYS_PTRACE` | Trace/debug other processes | APM agents, debuggers |
| `SYS_ADMIN` | Broad admin ops (mount, namespaces, etc.) | Avoid — almost as dangerous as `privileged` |
| `SYS_TIME` | Set the system clock | NTP daemons |
| `CHOWN` | Change file ownership | Backup tools, init scripts |
| `DAC_OVERRIDE` | Bypass file permission checks | Legacy apps that read files owned by other users |
| `FOWNER` | Bypass permission checks on file owner | Apps that modify files they don't own |
| `SETUID` / `SETGID` | Change process UID/GID | Process managers, su/sudo |
| `KILL` | Send signals to any process | Process supervisors |
| `AUDIT_WRITE` | Write to the kernel audit log | Logging agents |
| `MKNOD` | Create device special files | Storage drivers |

**Docker default capabilities** (what containers get if you don't configure anything):

`AUDIT_WRITE`, `CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `FSETID`, `KILL`, `MKNOD`, `NET_BIND_SERVICE`, `NET_RAW`, `SETFCAP`, `SETGID`, `SETPCAP`, `SETUID`, `SYS_CHROOT`

Most workloads need far fewer than these defaults.

---

## Sysctls in Pods

Kernel parameters can be tuned per-pod using the `sysctls` field. Only namespaced sysctls (safe sysctls) are allowed by default:

```yaml
spec:
  securityContext:
    sysctls:
    - name: net.core.somaxconn
      value: "1024"
    - name: net.ipv4.ip_local_port_range
      value: "1024 65535"
    - name: net.ipv4.tcp_syncookies
      value: "1"
```

### Safe vs unsafe sysctls

| Type | Description | Examples |
|------|-------------|---------|
| **Safe** (namespaced) | Isolated per pod, no host impact | `net.ipv4.ip_local_port_range`, `net.core.somaxconn`, `net.ipv4.ping_group_range` |
| **Unsafe** | May affect the node or other pods | `net.core.message_burst`, `kernel.shm*`, `vm.overcommit_memory` |

Unsafe sysctls must be explicitly allowed on the kubelet:

```bash
# kubelet flag
--allowed-unsafe-sysctls="net.core.message_burst,net.core.message_cost"
```

---

## Runtime Debugging Commands

Commands for inspecting and troubleshooting security contexts on running pods.

### Inspect current identity

```bash
# Check UID/GID of the running process
kubectl exec my-pod -- id
# uid=1000(app) gid=1000(app) groups=1000(app),2000

# Check the process tree with UIDs
kubectl exec my-pod -- ps aux

# Verify the user inside the container
kubectl exec my-pod -- whoami
```

### Inspect capabilities

```bash
# View effective capabilities (hex bitmask)
kubectl exec my-pod -- cat /proc/1/status | grep -i cap
# CapInh: 0000000000000000
# CapPrm: 0000000000000000
# CapEff: 0000000000000000
# CapBnd: 0000000000000000
# CapAmb: 0000000000000000

# Decode capabilities (requires capsh — may not be in minimal images)
kubectl exec my-pod -- capsh --decode=0000000000000001
# 0x0000000000000001=cap_chown

# List capabilities with getpcaps (if available)
kubectl exec my-pod -- getpcaps 1

# Use a debug container to inspect capabilities
kubectl debug -it my-pod --image=ubuntu -- bash -c "apt-get update && apt-get install -y libcap2-bin && getpcaps 1"
```

### Inspect seccomp

```bash
# Check if seccomp is active (2 = filter mode)
kubectl exec my-pod -- cat /proc/1/status | grep Seccomp
# Seccomp:         2
# Seccomp_filters: 1

# Values: 0 = disabled, 1 = strict, 2 = filter (BPF)
```

### Inspect SELinux context

```bash
# Check the SELinux label of the container process
kubectl exec my-pod -- cat /proc/1/attr/current
# system_u:system_r:container_t:s0:c200,c500

# Check SELinux labels on files
kubectl exec my-pod -- ls -Z /data
```

### Inspect filesystem and mounts

```bash
# Check if root filesystem is read-only
kubectl exec my-pod -- touch /testfile
# touch: /testfile: Read-only file system

# View mount options (look for "ro" flag)
kubectl exec my-pod -- cat /proc/mounts | grep "/ "

# Check writable volumes
kubectl exec my-pod -- df -h
kubectl exec my-pod -- mount | grep -v "ro,"
```

### Inspect privilege escalation flag

```bash
# Check no_new_privs flag
kubectl exec my-pod -- cat /proc/1/status | grep NoNewPrivs
# NoNewPrivs:      1    (1 = blocked, 0 = allowed)
```

### Inspect namespace isolation

```bash
# View the namespaces of process 1
kubectl exec my-pod -- ls -la /proc/1/ns/

# Compare with host (from node access)
ls -la /proc/1/ns/
```

### Test specific permissions

```bash
# Test network binding to privileged port
kubectl exec my-pod -- python3 -c "import socket; s=socket.socket(); s.bind(('',80))"
# If NET_BIND_SERVICE is dropped: Permission denied

# Test raw socket (ping)
kubectl exec my-pod -- ping -c 1 8.8.8.8
# If NET_RAW is dropped: Operation not permitted

# Test file ownership change
kubectl exec my-pod -- chown 1234:1234 /tmp/testfile
# If CHOWN is dropped: Operation not permitted

# Test mounting
kubectl exec my-pod -- mount -t tmpfs tmpfs /mnt
# If SYS_ADMIN is dropped: Operation not permitted
```

### Useful debug image

When the target container is too minimal for debugging:

```bash
# Attach an ephemeral debug container (shares process namespace)
kubectl debug -it my-pod --image=nicolaka/netshoot --target=app -- bash

# From inside the debug container, inspect the target process
cat /proc/1/status | grep -i cap
cat /proc/1/status | grep Seccomp
cat /proc/1/attr/current
```

---

## Summary and Best Practices

Securing containers in Kubernetes isn't about one magic setting — it's about layering protections.

```
Defense in Depth: Security Layers
══════════════════════════════════
Each layer blocks a different class of attack. An attacker must bypass ALL of them.

┌─────────────────────────────────────────────────────────────────────────────────────┐
│ Layer 1: Pod Security Admission (PSA)                                               │
│ Namespace-level policy enforcement — rejects insecure pods before they run          │
│ Blocks: privileged pods, hostNetwork/hostPID, missing hardening                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ Layer 2: Seccomp / AppArmor / SELinux                                               │
│ Mandatory access control and syscall filtering at the kernel level                  │
│ Blocks: dangerous syscalls (mount, ptrace, reboot), unauthorized file access        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ Layer 3: Linux Capabilities                                                         │
│ Fine-grained privilege control — drop ALL, add back only what's needed              │
│ Blocks: network manipulation, kernel module loading, raw socket creation            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ Layer 4: runAsNonRoot / runAsUser / fsGroup                                         │
│ UID/GID identity and filesystem ownership — no root, no host-level access           │
│ Blocks: container breakout with root UID, reading host files, cross-pod access      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ Layer 5: readOnlyRootFilesystem / allowPrivilegeEscalation: false                   │
│ Filesystem immutability and privilege escalation prevention                         │
│ Blocks: malware persistence, setuid escalation, config tampering                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ Layer 6: Namespaces / cgroups                                                       │
│ Kernel-level process isolation — the foundational boundary                          │
│ Blocks: visibility into other containers, resource exhaustion                       │
└─────────────────────────────────────────────────────────────────────────────────────┘
       ▲                                                                        ▲
       │                    attacker must breach ALL layers                     │
  CLUSTER-WIDE                                                            PER-CONTAINER
  (admission)                                                            (kernel-level)
```

1. **Avoid root entirely** — Always set `runAsNonRoot: true` and define an explicit `runAsUser`
2. **Drop all Linux capabilities by default** — Use `drop: ["ALL"]` as your baseline, add back only what's required
3. **Block privilege escalation** — Set `allowPrivilegeEscalation: false`
4. **Lock down the root filesystem** — Use `readOnlyRootFilesystem: true`, mount writable `emptyDir` volumes only where needed
5. **Use fsGroup** — Set it to match the container's runtime group ID for volume access
6. **Don't trust third-party images by default** — Review base images, override insecure defaults, scan for risks
7. **Apply Seccomp** — Start with `RuntimeDefault` and only create custom profiles if needed

Each of these controls blocks a specific class of attacks — privilege escalation, lateral movement, and persistence. Build incrementally, test behavior, and validate permissions at runtime.
