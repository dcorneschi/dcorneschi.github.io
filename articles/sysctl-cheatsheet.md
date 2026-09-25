# sysctl Cheatsheet

`sysctl` reads and writes **kernel parameters** at runtime — the tunables exposed under `/proc/sys/`. Each parameter uses dotted notation (`net.ipv4.ip_forward`) that maps directly to a file path (`/proc/sys/net/ipv4/ip_forward`). Changes made with `sysctl -w` take effect immediately but are lost on reboot; to persist them, write to a config file under `/etc/sysctl.d/`.

Several tuning tasks lean on sysctl — see [Triggering and Testing the Linux OOM Killer](articles/linux-oom-killer-testing.md) and [Configuring kdump on RHEL 6–10](articles/kdump-configuration-rhel.md) for real examples.

## Reading Values

```sh
sysctl net.ipv4.ip_forward        # show one parameter
sysctl -a                         # show every parameter (long)
sysctl -a | grep somaxconn        # find a parameter by name
sysctl net.ipv4                   # show all keys under a prefix (some versions)
```

The parameter name and its `/proc/sys` path are interchangeable — `sysctl net.ipv4.ip_forward` reads the same value as `cat /proc/sys/net/ipv4/ip_forward` (dots become slashes).

## Changing Values (Runtime)

`sysctl -w` sets a value immediately; it does **not** survive a reboot.

```sh
sysctl -w net.ipv4.ip_forward=1               # enable IPv4 forwarding now
sysctl -w vm.swappiness=10                    # tune swap tendency now

# equivalent low-level write (same effect, not persistent)
echo 1 > /proc/sys/net/ipv4/ip_forward
```

## Persisting Values

Runtime changes are lost on reboot. To make them permanent, add them to a config file, then apply.

### Config file locations

- **`/etc/sysctl.conf`** — the traditional single file (still honored).
- **`/etc/sysctl.d/*.conf`** — drop-in directory; the modern, preferred place. Files are applied in lexical order, so a `99-tuning.conf` overrides earlier ones.

```ini
# /etc/sysctl.d/99-tuning.conf
net.ipv4.ip_forward = 1
vm.swappiness = 10
net.core.somaxconn = 1024
```

### Apply the config

```sh
sysctl -p                                  # reload /etc/sysctl.conf
sysctl -p /etc/sysctl.d/99-tuning.conf     # reload a specific file
sysctl --system                            # reload ALL standard config paths
```

> On systemd systems, `sysctl --system` reads every standard location (`/etc/sysctl.d/`, `/run/sysctl.d/`, `/usr/lib/sysctl.d/`, and `/etc/sysctl.conf`) in the correct precedence order — the most reliable "apply everything" command. `sysctl -p` alone only reads `/etc/sysctl.conf` unless you name a file.

## Options Reference

| Option | Meaning |
|--------|---------|
| `sysctl VAR` | Display the value of VAR |
| `sysctl -a` | Display all available parameters |
| `sysctl -w VAR=VALUE` | Set VAR at runtime (not persistent) |
| `sysctl -p [FILE]` | Load settings from FILE (default `/etc/sysctl.conf`) |
| `sysctl --system` | Load from all standard config directories |
| `sysctl -e` | Ignore errors about unknown keys |
| `sysctl -q` | Quiet — suppress normal output when setting |
| `sysctl -N` | Print only the names, not the values |
| `sysctl -b VAR` | Print the value with no newline (for scripting) |

## Commonly Tuned Parameters

A few frequently adjusted keys, for orientation (verify appropriateness for your workload):

| Parameter | Purpose |
|-----------|---------|
| `net.ipv4.ip_forward` | Enable routing/forwarding between interfaces |
| `vm.swappiness` | How aggressively the kernel swaps (0–100) |
| `vm.panic_on_oom` | Panic instead of invoking the OOM killer |
| `kernel.panic` | Seconds to wait after a panic before rebooting |
| `net.core.somaxconn` | Max pending connections queue length |
| `fs.file-max` | System-wide max open file descriptors |
| `kernel.pid_max` | Maximum process ID value |

## Notes and Gotchas

- **`-w` is temporary.** Runtime changes vanish on reboot; put anything you want to keep in `/etc/sysctl.d/`.
- **Prefer `/etc/sysctl.d/` over editing `/etc/sysctl.conf`** on modern systems — drop-ins are cleaner and package-friendly.
- **`sysctl -p` ≠ `sysctl --system`.** Plain `-p` only reads `/etc/sysctl.conf`; use `--system` (or name the file) to apply drop-ins.
- **Unknown-key errors** on `-p` usually mean a module isn't loaded yet (e.g. a `net.bridge.*` key before `br_netfilter` is loaded). Load the module first, or use `-e` to ignore.
- **Names map to paths:** `a.b.c` ⇔ `/proc/sys/a/b/c`. Reading `/proc/sys` and using `sysctl` are equivalent.
- **Boot-time timing.** `/etc/sysctl.d/` is applied early in boot; parameters depending on later-loaded modules may need a module load or a udev rule.

## Quick Reference

```sh
sysctl VAR                         # read one value
sysctl -a                          # read all values
sysctl -a | grep KEY               # find a parameter
sysctl -w VAR=VALUE                # set now (temporary)
echo VALUE > /proc/sys/PATH        # equivalent low-level write

# persist:
echo 'VAR = VALUE' >> /etc/sysctl.d/99-tuning.conf
sysctl --system                    # apply all config (systemd)
sysctl -p /etc/sysctl.d/99-tuning.conf   # apply one file
```

For related material, see [Triggering and Testing the Linux OOM Killer](articles/linux-oom-killer-testing.md) and [Configuring kdump on RHEL 6–10](articles/kdump-configuration-rhel.md).
