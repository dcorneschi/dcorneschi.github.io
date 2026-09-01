# AIX System Dump and Core File Cheatsheet

Command reference for system crash dumps and process core files on IBM AIX — configuring the dump device (`sysdumpdev`), checking dump size requirements (`dumpcheck`), controlling crash-reboot and full-core behavior, generating and analysing core files (`gencore`, `snapcore`, `check_core`), examining dumps with `kdb`/`crash`/`mdmprpt`, and the System Health / Shell (`shconf`) settings.

> A dedicated dump logical volume `lg_dumplv` is created automatically on systems with at least 4 GB of real memory. Most of these commands require `root`. Changing the dump device or reboot behavior affects how the system recovers from a crash — verify device names with `lsvg -l rootvg` before repointing the dump.

## System Dump Device (sysdumpdev)

`sysdumpdev` displays and configures the primary and secondary dump devices used to capture a kernel crash dump.

```sh
# Show where the system dump is currently located
sysdumpdev -l

# Change the PRIMARY dump device (-P persists it, -p sets the device)
sysdumpdev -P -p /dev/hd9

# Estimate the size of the current dump
sysdumpdev -e

# Display statistics about the last dump
sysdumpdev -L
```

| Option | Purpose |
|--------|---------|
| `-l` | List current primary/secondary dump devices |
| `-P` | Make the change permanent (persist across reboots) |
| `-p` | Set the primary dump device |
| `-s` | Set the secondary dump device |
| `-e` | Estimate current dump size |
| `-L` | Show information about the most recent dump |

Configure the dump interactively through SMIT:

```sh
# Access dump configuration through SMIT
smitty dump
```

## Dump Size Check (dumpcheck)

`/usr/lib/ras/dumpcheck` verifies that the dump device and the copy directory have enough free space, and normally runs from cron.

```sh
# Print the estimated dump size AIX requires
/usr/lib/ras/dumpcheck -p

# Remove the dumpcheck crontab entry
/usr/lib/ras/dumpcheck -r
```

## Crash and Reboot Behavior

```sh
# Automatically reboot after a crash (default: false)
chdev -l sys0 -a autorestart=true

# Enable full core dumps (capture the full data section)
chdev -l sys0 -a fullcore=true
```

> AIX exposes both `autorestart` and, on some levels, `autostart` on `sys0`; use `lsattr -El sys0` to confirm the exact attribute name before changing it. Enabling `fullcore` produces larger core files but preserves more state for analysis.

## Starting a Dump Manually

```sh
# Force/start a system dump
sysdumpstart
```

## Dump Copy Directory

On reboot after a crash, AIX copies the dump from the dump device into the **copy directory** (default `/var/adm/ras`), producing files like `vmcore.0`. The device and copy directory are shown by `sysdumpdev -l`.

```sh
# Show the current copy directory (copydir) and dump devices
sysdumpdev -l

# Set the copy directory (persist with -P)
sysdumpdev -P -d /var/adm/ras

# Set copy directory, but if space is short, continue booting
# instead of prompting (ignore) — use -D to prompt instead
sysdumpdev -P -D /var/adm/ras

# Copy a dump from the dump device to a directory / media after boot
savecore -d /var/adm/ras
```

> If the copy directory lacks space for the dump, the `-d` option ignores the copy (system boots), while `-D` prompts at the console. Always confirm free space in the copy directory with `sysdumpdev -e` (estimated size) beforehand.

## Analysing a System Dump

AIX writes the vmcore file (e.g. `/var/adm/ras/vmcore.0`) which you analyse with `kdb`, `crash`, or `mdmprpt`.

```sh
# Open a dump with the kernel debugger against the running kernel image
kdb /var/adm/ras/vmcore.0 /unix

# Inside kdb, useful subcommands:
#   trace -k      kernel stack trace
#   thread -r     display thread/run state
#   status 0      status of processor 0

# Analyse a dump non-interactively with crash
echo "stat\n status\n t -m" | crash /var/adm/ras/vmcore.0

# Read a minidump from the error log
mdmprpt -i /var/adm/ras/errlog
```

| Tool | Use |
|------|-----|
| `kdb` | Interactive kernel debugger for live system or vmcore |
| `crash` | Scriptable dump analysis (feed subcommands via stdin) |
| `mdmprpt` | Format and display a minidump stored in the error log |

## Core Files (Process Dumps)

### Where core files are written (syscorepath)

`syscorepath` sets a single repository directory for all process core files instead of scattering them in each process's working directory.

```sh
# Set /core as the repository for core files
syscorepath -p /core

# Display the current core-file repository
syscorepath -g

# Unset (clear) the configured core-file repository
syscorepath -c
```

### Generating and collecting cores

```sh
# Generate a core dump of a running/hung process by PID
gencore 188538 /tmp/core

# Collect a core plus its binary and shared libraries
# (default output dir is /tmp/snapcore)
snapcore -d /tmp/snapcore2 core.xx /sbin/vxconfigd.5.3
```

### Inspecting core files

```sh
# List the program that produced a core file and the libraries it used
/usr/lib/ras/check_core core.xx

# Show which program generated the core file
strings core | grep _=

# View the contents of the snapcore pax archive
uncompress -c snapcore_32811.pax.Z | pax
```

## System Health / Shell Daemon (shconf)

`shconf` and the `shdaemon` control System Health monitoring behavior.

```sh
# Display whether the settings are enabled or not
shconf -d

# Display the current configuration of shdaemon (priority)
shconf -El prio
```

## Quick Reference

| Task | Command |
|------|---------|
| Show dump location | `sysdumpdev -l` |
| Set primary dump device | `sysdumpdev -P -p /dev/hd9` |
| Estimate dump size | `sysdumpdev -e` |
| Last dump info | `sysdumpdev -L` |
| SMIT dump config | `smitty dump` |
| Required dump size | `/usr/lib/ras/dumpcheck -p` |
| Remove dumpcheck cron | `/usr/lib/ras/dumpcheck -r` |
| Auto-reboot after crash | `chdev -l sys0 -a autorestart=true` |
| Enable full core | `chdev -l sys0 -a fullcore=true` |
| Start a dump | `sysdumpstart` |
| Analyse vmcore | `kdb /var/adm/ras/vmcore.0 /unix` |
| Read minidump | `mdmprpt -i /var/adm/ras/errlog` |
| Set core repository | `syscorepath -p /core` |
| Show core repository | `syscorepath -g` |
| Generate process core | `gencore <PID> /tmp/core` |
| Collect core + libs | `snapcore -d <dir> core.xx <binary>` |
| Identify core producer | `/usr/lib/ras/check_core core.xx` |
| Health settings status | `shconf -d` |

## Related

- [AIX Error Logging and System Logs Cheatsheet](articles/aix-error-logging-cheatsheet.md) — `errpt`/`errdemon` and the error log where dump and RAS events are recorded.
- [AIX Power Systems, LPAR, and Boot Concepts](articles/aix-power-lpar-boot-concepts.md) — the boot process, BLV, and `bosboot`/KDB context behind kernel dumps.
- [AIX Performance Monitoring Cheatsheet](articles/aix-performance-monitoring-cheatsheet.md) — `svmon` and memory analysis for the processes that produce cores.
