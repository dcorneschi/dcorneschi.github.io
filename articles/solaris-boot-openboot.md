# Solaris Boot Management: OpenBoot, eeprom, and bootadm

Booting and boot configuration on Oracle Solaris spans two worlds: **SPARC** systems use the OpenBoot PROM (OBP) firmware with its `ok` prompt and NVRAM parameters, while **x86** systems use GRUB and the boot archive. This guide covers OpenBoot boot commands, managing NVRAM parameters (from the `ok` prompt and from a running system with `eeprom`), and the x86 boot archive with `bootadm`.

## Platform Overview

| Area | SPARC | x86 |
|------|-------|-----|
| Firmware / bootloader | OpenBoot PROM (`ok` prompt) | BIOS/UEFI → GRUB |
| Boot parameters | NVRAM (via OBP or `eeprom`) | `eeprom` (maps to bootenv.rc) + GRUB |
| Boot config file | NVRAM parameters | GRUB `menu.lst` |
| RAM boot image | — | boot archive (`/platform/i86pc/boot_archive`) |

`eeprom` works on both platforms; on SPARC it reads/writes real NVRAM, on x86 it stores parameters in `bootenv.rc`.

## OpenBoot Boot Commands (SPARC `ok` Prompt)

You reach the `ok` prompt by halting the OS (`init 0`), or via `Stop-A` / a console break.

```text
ok boot -s          # boot to single-user mode (prompts for the root password)
ok boot cdrom -s    # boot single-user from CD-ROM/DVD (recovery/install)
ok boot -a          # boot interactively (prompts for config file paths, etc.)
ok boot -r          # reconfiguration boot (rebuild device tree for new hardware)
```

| Command | Purpose |
|---------|---------|
| `boot -s` | Single-user mode; asks for the root password |
| `boot cdrom -s` | Single-user from removable media — recovery |
| `boot -a` | Interactive boot, prompting for boot parameters |
| `boot -r` | Reconfiguration boot — detect newly attached hardware |

A **reconfiguration boot** (`boot -r`) is the classic way to make Solaris pick up hardware added while powered off; on a running system, `devfsadm` does the equivalent.

### Reaching the `ok` Prompt

```bash
# From a running system — halt to the ok prompt
init 0
# or
halt
```

- **Console break:** `Stop-A` (`L1-A`) on a Sun keyboard, or `Ctrl-]` then `send brk` over a telnet/console session, or `~#` on an LOM/ALOM serial console.
- Beware: on a system with `auto-boot?=true`, a `reset-all` at the `ok` prompt reboots straight back into the OS.

### Run Levels and Shutdown

Solaris uses SVR4 run levels; `init`/`shutdown` move between them:

| Run level | State |
|-----------|-------|
| `0` | Down to the OpenBoot `ok` prompt / firmware |
| `S` / `s` | Single-user (maintenance) |
| `1` | Single-user, all filesystems mounted |
| `2` | Multi-user, no NFS shares |
| `3` | Multi-user with NFS (default) |
| `5` | Power off the machine |
| `6` | Reboot (to default run level) |

```bash
# Graceful shutdown with a grace period and a target run level
shutdown -y -g0 -i6      # reboot now, no confirm
shutdown -y -g60 -i5     # power off after 60s
init 6                   # reboot
init 5                   # power off
```

## OpenBoot Help and Device Inspection

```text
ok help             # help on the main OpenBoot firmware categories
ok show-devs        # view the entire device tree
ok devalias         # list device aliases and the current boot device alias
```

- `show-devs` lists the full hardware device path tree.
- `devalias` maps friendly names (e.g. `disk`, `disk2`, `net`) to full device paths — these aliases are what you pass to `boot` and set as `boot-device`.

## NVRAM Parameters from OpenBoot

```text
ok printenv                     # list all NVRAM parameters and values
ok setenv auto-boot? false      # change a parameter (disable auto-boot here)
ok set-defaults                 # reset ALL parameters to factory defaults
ok set-default diag-level       # reset a SINGLE parameter to its default
```

- `printenv` shows every parameter, its current value, and its default.
- `setenv <param> <value>` changes one parameter.
- `set-defaults` (plural) resets everything; `set-default <param>` (singular) resets just one.
- Common parameters: `auto-boot?` (boot automatically at power-on), `boot-device` (device/alias to boot from), `diag-level`, `boot-file`.

Sample `printenv` output (abbreviated):

```
Parameter Name        Value            Default Value
auto-boot?            true             true
boot-device          disk net          disk net
boot-file            (empty)           (empty)
diag-switch?         false             false
```

Key parameters worth knowing:

| Parameter | Controls |
|-----------|----------|
| `auto-boot?` | Whether the system boots automatically at power-on |
| `boot-device` | Ordered list of devices/aliases to try booting |
| `boot-file` | Kernel/boot file and flags (e.g. `-s`) |
| `diag-switch?` | Run POST diagnostics at boot |
| `diag-level` | Depth of diagnostics (`min`/`max`) |
| `local-mac-address?` | Use each NIC's own MAC (needed for some network setups) |
| `nvramrc` / `use-nvramrc?` | Firmware startup script (holds custom `devalias` entries) |

### Making a Boot Device Alias Persistent

`devalias` set at the prompt is temporary. To persist a custom alias across resets, use `nvalias` (which writes it into `nvramrc`):

```text
ok nvalias mydisk /pci@1f,0/pci@1/scsi@8/disk@1,0
ok setenv boot-device mydisk
ok reset-all
```

## NVRAM Parameters from a Running System (`eeprom`)

`eeprom` lets you read and set the same parameters without dropping to the `ok` prompt — handy for scripting and remote administration.

```bash
eeprom                          # list all parameters with current values
eeprom boot-device              # show a single parameter
eeprom boot-device=disk2        # set the default boot device to disk2
eeprom 'auto-boot?'=true        # names containing '?' must be single-quoted
```

> The `?` in parameter names like `auto-boot?` is a shell metacharacter, so wrap the whole name in single quotes: `eeprom 'auto-boot?'=true`.

## x86 Boot Archive and GRUB (`bootadm`)

On x86, Solaris boots via GRUB and loads a **boot archive** — a RAM filesystem image containing the kernel modules and data needed to bring the system up before the root filesystem is mounted.

```bash
# Show the active GRUB menu.lst location
bootadm list-menu

# The primary boot archive image
ls -l /platform/i86pc/boot_archive

# List the contents of the primary boot archive
bootadm list-archive
```

- `bootadm list-menu` reports the active boot menu and its entries.
- **GRUB version differs by release:** Solaris 10 and Solaris 11 11/11 use **GRUB Legacy** with a `menu.lst` file (`/rpool/boot/grub/menu.lst`). Solaris 11.1 and later switched to **GRUB 2**, where the menu is generated from the ZFS boot configuration and `bootadm` is the supported way to manage it (don't hand-edit a `menu.lst` on GRUB 2). `bootadm list-menu` works on both.
- The **boot archive** at `/platform/i86pc/boot_archive` is loaded by GRUB early in boot.
- After changing drivers or `/etc` boot config, rebuild the archive so it stays consistent:

  ```bash
  bootadm update-archive
  ```

  A stale or corrupt boot archive is a common cause of x86 Solaris boot failures — `bootadm update-archive` (or a reconfiguration reboot) regenerates it.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Boots to `ok` and stops | `auto-boot?=false` | `setenv auto-boot? true`, or type `boot` |
| Boots the wrong disk | `boot-device` order | `setenv boot-device <alias>`; make it persistent with `nvalias` |
| x86 won't boot after driver/patch change | Stale boot archive | Boot install media → `bootadm update-archive -R /a` on the mounted root |
| "boot archive out of date" warning | Archive not regenerated | `bootadm update-archive` then reboot |
| New hardware not seen | Device tree not rebuilt | `boot -r` (SPARC) or `reboot -- -r` / `devfsadm` |
| Custom `devalias` lost after reset | Set with `devalias`, not `nvalias` | Recreate with `nvalias` (persists in `nvramrc`) |

```bash
# Repair a broken x86 boot archive from install media (root mounted at /a)
bootadm update-archive -R /a
```

## Command Reference

| Task | SPARC (OpenBoot) | x86 / running system |
|------|------------------|----------------------|
| Single-user boot | `boot -s` | GRUB entry / `boot -s` equivalent |
| Recovery boot | `boot cdrom -s` | boot from install media |
| Reconfiguration boot | `boot -r` | `boot -r` / `devfsadm` |
| List boot params | `printenv` | `eeprom` |
| Set boot param | `setenv name value` | `eeprom name=value` |
| Reset one/all params | `set-default` / `set-defaults` | edit via `eeprom` |
| Set boot device | `setenv boot-device disk2` | `eeprom boot-device=disk2` |
| Device tree / aliases | `show-devs` / `devalias` | `prtconf` / `bootadm list-menu` |
| Boot archive | — | `bootadm list-archive` / `update-archive` |

## References

- [Booting and Shutting Down Oracle Solaris on SPARC](https://docs.oracle.com/cd/E37838_01/html/E52182/index.html) — official Oracle docs
- [Booting and Shutting Down Oracle Solaris on x86](https://docs.oracle.com/cd/E37838_01/html/E52181/index.html) — official Oracle docs
- [OpenBoot Command Reference](https://docs.oracle.com/cd/E19455-01/806-1377-10/index.html) — official Oracle docs
