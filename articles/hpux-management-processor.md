# HP-UX Management Processor (MP / GSP / iLO)

The Management Processor is HP-UX's out-of-band service processor — the console, power control, and hardware management interface that stays available even when the OS is down. This guide covers the MP variants, its menus (console, VFP, logs, command menu), accessing nPar/vPar consoles, user accounts and access levels, and the reset/power commands. Relates to [nPartitions](articles/hpux-npars.md) and [Virtual Partitions](articles/hpux-vpars.md).

## Concepts: What the Management Processor Is

The Management Processor is a small, independent computer inside the server that runs whether or not HP-UX is running — it has its own processor, its own firmware, its own LAN connection, and (crucially) its own power domain fed from standby power. That independence is the whole point: it gives you **out-of-band** management. "Out-of-band" means your control path to the box does not depend on the operating system, the network stack HP-UX brings up, or even the main system power being on. When HP-UX has hung, panicked, or is sitting at a firmware prompt, the MP is still reachable and can show you the console, read the hardware logs, and cycle power.

Contrast this with **in-band** access (`ssh` to the running HP-UX): convenient, but it vanishes the moment the OS is unhealthy — exactly when you most need to get in. Every serious HP-UX deployment configures the MP LAN so that remote console and power control survive an OS outage.

A few ideas underpin everything below:

- **The MP owns the console.** On these systems there is no VGA monitor; the "console" is a serial/LAN stream the MP presents. That is why you reach the OS console *through* the MP, and why several people can watch the same console at once.
- **The MP logs hardware events itself.** Chassis/system logs are written by the MP independent of HP-UX, so they survive a crash and are the authoritative record for hardware faults (a failed fan, a memory error, a power event).
- **The MP can act on hardware without the OS.** Reset, TOC, and power commands go straight to the hardware, bypassing HP-UX — powerful and dangerous, which is why they carry data-corruption warnings.

## MP Variants

| Name | Used on |
|------|---------|
| **GSP** (Guardian Service Processor) | Older servers |
| **MP** (Management Processor) | Later servers |
| **iLO 2** | Latest entry-level servers (MP built on Integrated Lights-Out) |

The three names are the same idea evolving over hardware generations: an out-of-band service processor. **GSP** is the oldest branding, **MP** the mid-generation name, and **iLO 2** the Integrated Lights-Out implementation on later Integrity servers. The menu commands are largely consistent across them, with newer units adding the command menu (`cm`), DNS, SSH/SSL, and firmware update. Where a command differs by generation it is called out below.

**First-time setup:** use the local **serial console** to configure the MP LAN IP address. The predefined login is **Admin / Admin** — change it immediately.

Why serial first: out of the box the MP LAN has no usable address, so the only way in is the physical serial port. Connect a terminal (typically 9600 8N1), log in as `Admin/Admin`, then use `lc` to set a static IP, mask, and gateway. After that you can abandon the serial cable and manage everything over the LAN — but keep the serial port documented, because it is your fallback if the MP LAN config is ever lost or the `dc` (reset to defaults) command is run. Treat the MP LAN like any other management interface: put it on an isolated management VLAN, not the production data network.

## MP Menus (Older Systems)

At the `MP>` prompt:

```
co    console interface (exit with ^b)
vfp   virtual front panel — live system status / boot progress (q to quit)
cl    console log viewer (q)
sl    chassis (system) log viewer, written by the MP (q)
he    help menu (q)
x     terminate the MP session
```

The two log viewers are worth distinguishing. The **console log** (`cl`) is a scrollback buffer of what has appeared on the OS/firmware console — useful for reading boot messages or a panic you just missed. The **chassis/system log** (`sl`) is different in kind: it is written by the MP itself and records *hardware* events (fans, power, thermal, memory errors) with severity levels, independent of HP-UX. When diagnosing a hardware fault, `sl` is the authoritative source because it keeps logging even when the OS is down.

The **VFP** (`vfp`, virtual front panel) is a live status display — it mirrors what the physical front-panel LEDs and chassis codes would show, so you can watch boot progress, self-test codes, and current state in real time from a remote session. It is read-only and the quickest way to answer "where is this partition in its boot?" without taking console control.

**Shared console:** several users can view a console session at once, but only **one** has keyboard control. To take control of the keyboard, press **`^e`** then **`cf`**. This shared model is why you can safely join a colleague's troubleshooting session to watch, and why you should announce yourself (`te`) before grabbing keyboard control — snatching it mid-command from someone else is a good way to break a delicate recovery.

## MP Command Menu (Newer Systems)

Newer MPs add a **command menu** (`cm`) and firmware update (`fw`):

```
MP:CM> who        list currently logged-in MP users
MP:CM> sysrev     firmware revision
MP:CM> ls         display current LAN console port status
MP:CM> lc         modify the static LAN configuration (IP, etc.)
MP:CM> dns        modify the DNS configuration
MP:CM> sa         display current LAN console port settings
MP:CM> so         generate SSL/SSH certificates (security options)
MP:CM> uc         modify the user configuration (add/change/remove users)
MP:CM> te         send (tell) a message to other logged-in MP users
MP:CM> di         terminate all current remote logins
MP:CM> xd         restart the MP (needed after LAN config changes)
MP:CM> dc         reset the MP configuration to factory defaults
MP:CM> rs         reset a partition
MP:CM> tc         crash dump (TOC), then reset a partition
MP:CM> pc / pe    power the system / an entity up or down
MP:CM> ps         view power status of system components
```

After changing the LAN configuration with `lc`, run **`xd`** to restart the MP so the change takes effect. Restarting the MP does **not** affect the running OS or partitions — only management access blinks out briefly — so it is safe to do on a production box, though you will lose your MP session and have to reconnect.

A sensible hardening pass for a new system, all from the command menu:

- `so` — generate fresh SSL/SSH certificates and require SSH rather than telnet.
- `uc` — remove or rename the default `Admin` account after creating your own named admin users, and give each real person their own login (so `who` and the logs attribute actions to a person).
- `lc` / `dns` — pin the MP to a static address on the management VLAN and set DNS.
- `di` — a fast way to boot off any stale remote sessions before you begin sensitive work.

Be careful with `dc` (reset configuration to defaults): it wipes the LAN settings and user accounts, dropping you back to serial-only access with `Admin/Admin`. Only run it when you intend to re-provision from the serial console.

## Accessing an nPar Console

From `MP>`, `co` lists the partitions and lets you pick one:

```
MP> co

  Partitions available:
  Part#  Name
  -----  ----
     0)  Partition 0
     1)  Partition 1
     Q)  Quit
```

Once on a console, confirm which nPar you're on (node-partitionable systems only):

```bash
parstatus -w        # e.g. "The local partition number is 0."
```

## Accessing a vPar Console

On cell-based systems running vPars, each vPar has its own console. First enter the **nPar** console via the MP (`co`), then cycle between the vPar consoles inside it with **`^a`**:

```bash
parstatus -w        # which nPar console am I on?
# press ^a to cycle vPar consoles
vparstatus -w       # which vPar console am I on?
```

## MP User Accounts and Access

MP user accounts control who can reach the MP:

- At least one account is predefined: **Admin / Admin** — change its password ASAP.
- The Admin user (or any user with Administrator rights) can create up to **19** additional MP users — via **`uc`** on newer systems or **`so`** on older ones.
- Disable or remove accounts that are no longer needed.

### Access Privileges — Newer Systems

Each user gets any combination of four privileges:

| Privilege | Grants |
|:---------:|--------|
| `C` | Console interface and VFP access |
| `P` | Power the server on/off |
| `M` | MP configuration commands (ports, protocols, IP) |
| `U` | Add/modify/remove MP user logins |

### Access Levels — Older Systems

| Level | Scope |
|-------|-------|
| **Administrator** | All GSP/MP commands and all partitions |
| **Operator** | Access/manage all partitions, but can't change GSP/MP config |
| **Single-Partition** | Limited commands to access/boot/reboot one specified partition (cell-based only) |

Older systems manage users via `so` (security options) or `uc`, depending on model.

## Reset and Power Commands

> Always prefer a clean `shutdown`/`reboot` from within HP-UX. The MP `rs`/`tc`/`pc`/`pe` commands bypass the OS and can cause **data corruption**.

Think of these as a hierarchy of increasing bluntness. From within HP-UX, `shutdown -r` (reboot) or `shutdown -h` (halt) flushes buffers, unmounts filesystems, and stops services cleanly — always the first choice. The MP commands come into play only when the OS is unresponsive:

- `tc` is preferred over `rs` for a **hung** system, because it captures a crash dump first — that dump is often the only evidence of *why* it hung. Reach for `tc` when you want a diagnosis, `rs` when you just need the partition back and do not care why it stopped.
- `pc`/`pe` are the last resort — a hard power cycle with no state capture at all — used when even `rs`/`tc` do not respond.

The data-corruption warning is real: none of `rs`, `tc`, `pc`, `pe` gives HP-UX a chance to flush its buffer cache, so in-flight writes are lost and filesystems come up dirty (VxFS will replay its intent log on the next mount, which mitigates but does not eliminate the risk). Use them only when a clean shutdown is genuinely impossible.

### rs — reset

Immediate partition reset: halts all processing and I/O and restarts the partition, similar to power-cycling. The OS is **not** notified, so data corruption may result. On partitionable systems you're asked which partition to reset.

### tc — TOC (Transfer of Control) reset

Like `rs`, but first signals the processors to **dump state**, producing a crash dump under `/var/adm/crash/` for post-mortem analysis of a hang caused by an application or OS fault. Use with caution (possible data corruption). On partitionable systems you choose the partition to TOC.

### pc / pe — power control

- **`pc`** (power control) — toggle power for the **entire system** on non-partitionable systems.
- **`pe`** (power entity) — on cell-based systems, toggle power to individual **entities** (cell boards, I/O chassis, whole cabinets).
- Neither affects power to the **MP itself** (so you keep management access). Halt HP-UX first if possible.

### ps — power status

Reports power status of system components. On partitionable systems you select an entity for detailed status; on non-partitionable systems it shows a single-screen summary.

## Command Reference

| Task | Command |
|------|---------|
| Enter console | `MP> co` |
| Virtual front panel | `MP> vfp` |
| Console / chassis log | `MP> cl` / `MP> sl` |
| Take keyboard control | `^e` then `cf` |
| Exit console | `^b` |
| Command menu | `MP> cm` → `MP:CM>` |
| LAN config / restart MP | `MP:CM> lc` / `xd` |
| Firmware revision | `MP:CM> sysrev` |
| Manage users | `MP:CM> uc` (newer) / `so` (older) |
| Logged-in users / message | `MP:CM> who` / `te` |
| Reset partition | `MP:CM> rs` |
| TOC (dump + reset) | `MP:CM> tc` |
| Power up/down | `MP:CM> pc` / `pe` |
| Power status | `MP:CM> ps` |
| Which nPar / vPar | `parstatus -w` / `vparstatus -w` |
| Cycle vPar consoles | `^a` |

## Troubleshooting

### Cannot reach the MP over the LAN

If the MP does not answer on its IP, fall back to the serial console (it always works) and re-check the LAN settings. From the command menu:

```
MP:CM> lc     verify/repair IP, mask, gateway
MP:CM> ls     display current LAN status (link up? correct settings?)
MP:CM> xd     restart the MP if you changed anything
```

A dead MP LAN with a live link light is usually a wrong IP/gateway or a switch-port VLAN mismatch, not a failed MP. If `ls` shows the link down, it is physical (cable/switch port).

### "Console is in use" / no keyboard control

The console is shared and someone else holds keyboard control. See who is on and, if appropriate, coordinate before taking control:

```
MP:CM> who    list logged-in MP users
```

Then from the console, press **`^e`** then **`cf`** to take keyboard control. Use `te` to message the other user first, or `di` to disconnect stale sessions if a login is hung with no live person behind it.

### Stuck in a console session

If you are on the OS console (`co`) and need to get back to the `MP>` prompt without disturbing the OS, press **`^b`**. This detaches your view; it does not stop or reboot anything.

### Forgot the MP password / locked out

If all admin logins are lost, the only recovery is physical: connect to the serial port and, if even that is locked, use `dc` (reset configuration to defaults) which restores `Admin/Admin` — at the cost of erasing LAN and user config, so you must re-provision afterward.

## Related Articles

- [HP-UX nPartitions](articles/hpux-npars.md) — hardware partitions whose consoles you reach via the MP
- [HP-UX Virtual Partitions (vPars)](articles/hpux-vpars.md) — cycling between vPar consoles inside an nPar
- [HP-UX Boot Process](articles/hpux-boot-process.md) — ISL/BCH/EFI firmware you drive from the MP console
- [HP-UX Crash Dump Analysis](articles/hpux-crash-dump-analysis.md) — analyzing the dump a `tc` (TOC) produces
- [HP-UX Disaster Recovery](articles/hpux-disaster-recovery.md) — out-of-band access during recovery
- [HP-UX System Information](articles/hpux-system-information.md) — identifying partition and hardware layout
