# Triggering and Testing the Linux OOM Killer

When Linux runs out of usable memory (and swap), the kernel's **Out-Of-Memory (OOM) killer** picks a process and kills it to reclaim RAM and keep the system alive. This guide shows how to reproduce an OOM event on purpose with a small memory-hungry C program, how to make the kernel panic (and reboot) on OOM instead of killing a single process, and how to watch it happen in the logs.

This is a lab exercise — run it on a disposable VM, never on a production host. For related background, see [Linux Memory: RSS, VSZ, and Why RSS Alone Is Misleading](articles/linux-memory-rss-vsz.md), [Linux Swap Management](articles/linux-swap-management.md), and [Linux Kernel Panics](articles/linux-kernel-panics.md).

## How the OOM Killer Behaves

By default, when memory is exhausted the kernel selects a victim process (based on an OOM score that weighs memory use and other factors) and kills it. You can instead make the kernel **panic** on OOM, which is useful in clustered or HA setups where a clean reboot is preferable to a half-broken node:

- `vm.panic_on_oom=1` — panic instead of killing a process when OOM occurs.
- `kernel.panic=10` — after a panic, wait 10 seconds, then reboot automatically. (`0` means "halt and wait forever".)

## 1. Set the sysctl Values

Create a drop-in config so the settings persist across reboots:

```sh
vi /etc/sysctl.d/oom.conf
```

```ini
vm.panic_on_oom=1
kernel.panic=10
```

Apply it without rebooting:

```sh
sysctl -p /etc/sysctl.d/oom.conf
```

> With `vm.panic_on_oom=1` the machine will **kernel-panic and reboot** when it hits OOM. If you'd rather observe the classic single-process kill (the log messages shown later), skip this step or set `vm.panic_on_oom=0`, and the kernel will kill the `oom` test program instead of panicking.

## 2. Install a Compiler

```sh
yum install gcc          # RHEL/CentOS 7
dnf install gcc          # RHEL 8+/Fedora
```

## 3. Write the Memory-Eater

```sh
vi oom.c
```

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

#define TEN_MB (10 * 1024 * 1024)

int main(int argc, char **argv) {
    int c = 0;
    while (1) {
        char *b = malloc(TEN_MB);
        if (b == NULL) {
            printf("malloc failed\n");
            return 1;
        }
        memset(b, 0, TEN_MB);   /* touch every page so RSS actually grows */
        printf("Allocated %d MB\n", (++c * 10));
    }
    return 0;
}
```

> **Why the `memset` matters.** Linux over-commits memory: `malloc` can succeed without any physical RAM being assigned, because pages are only backed when first written to. If you never touch the memory, RSS stays tiny and the OOM killer may never fire. `memset(b, 0, TEN_MB)` writes to every page, forcing the kernel to back it with real RAM. (Note the argument order — it's `memset(dest, value, size)`; putting the size in the value slot writes nothing meaningful and defeats the test.)

## 4. Compile

```sh
gcc -o oom oom.c
```

## 5. Run It

```sh
./oom
```

It prints a running total of allocated memory and keeps growing until the kernel intervenes.

## 6. Watch Memory and the Logs

In a second terminal, watch free memory shrink:

```sh
watch -n1 "free -m"
```

In a third, follow the kernel log:

```sh
tail -f /var/log/messages      # RHEL/CentOS
# or
journalctl -kf                 # systemd systems
```

### With the OOM killer (panic_on_oom=0)

You'll see the kernel choose and kill the process:

```text
kernel: Out of memory: Kill process 6089 (oom) score 629 or sacrifice child
kernel: Killed process 6089, UID 0, (oom) total-vm:1076838980kB, anon-rss:196516kB, file-rss:392kB
```

- **score** — the victim's OOM score; higher means more likely to be killed.
- **total-vm** — total virtual memory the process had mapped.
- **anon-rss** — anonymous resident memory (heap/stack) actually in RAM.
- **file-rss** — file-backed resident memory in RAM.

### With panic_on_oom=1

Instead of the messages above, the box kernel-panics and (thanks to `kernel.panic=10`) reboots after 10 seconds. You won't get a tidy log line on the running system; check the console or a crash-capture mechanism.

## Inspecting and Tuning OOM Behavior

Every process has an OOM score you can read, and a knob to bias the killer:

```sh
cat /proc/<pid>/oom_score        # current score (higher = killed sooner)
cat /proc/<pid>/oom_score_adj    # adjustment: -1000 (never) .. +1000 (first)

# Protect a critical process from the OOM killer
echo -1000 > /proc/<pid>/oom_score_adj
```

Related sysctls:

| Setting | Effect |
|---------|--------|
| `vm.panic_on_oom` | `0` = kill a process (default); `1` = kernel panic on OOM |
| `kernel.panic` | Seconds to wait after a panic before rebooting (`0` = never) |
| `vm.overcommit_memory` | `0` heuristic (default), `1` always overcommit, `2` strict accounting |
| `vm.overcommit_ratio` | Percent of RAM overcommitted when `overcommit_memory=2` |

## Cleanup

```sh
rm -f oom oom.c
# revert the OOM/panic settings if you want the defaults back
rm -f /etc/sysctl.d/oom.conf
sysctl vm.panic_on_oom=0
```

## Key Takeaways

- The OOM killer frees memory by killing a process chosen via an OOM score; `vm.panic_on_oom=1` makes the kernel panic (and reboot with `kernel.panic=N`) instead.
- Linux over-commits memory, so you must **write to** allocated pages (`memset`) to force real RAM use and trigger OOM — allocating alone isn't enough.
- Watch it with `free -m` and `tail -f /var/log/messages` (or `journalctl -kf`).
- Protect critical processes with `oom_score_adj` set toward `-1000`.
- Run this only on a throwaway VM.

## Quick Reference

```sh
# Make the kernel panic + reboot on OOM
echo -e 'vm.panic_on_oom=1\nkernel.panic=10' > /etc/sysctl.d/oom.conf
sysctl -p /etc/sysctl.d/oom.conf

# Build and run the memory-eater
gcc -o oom oom.c && ./oom

# Observe
watch -n1 "free -m"
tail -f /var/log/messages     # or: journalctl -kf

# Protect a process from the OOM killer
echo -1000 > /proc/<pid>/oom_score_adj
```

For related material, see [Linux Memory: RSS, VSZ, and Why RSS Alone Is Misleading](articles/linux-memory-rss-vsz.md), [Linux Swap Management](articles/linux-swap-management.md), and [Linux Kernel Panics](articles/linux-kernel-panics.md).
