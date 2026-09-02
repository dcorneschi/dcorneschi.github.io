# Solaris Zones

Solaris Zones are the OS-level virtualization (container) feature of Oracle Solaris — isolated runtime environments that share a single kernel. This guide covers the global vs non-global zone model, the `z*` command set, and the full lifecycle: configure, install, boot, log in, and remove a zone.

## What Zones Are

- **OS-level virtualization** — lightweight isolated environments sharing the host kernel, not full VMs.
- **Solaris only** — a zone can host an instance of Solaris, not another operating system.
- **Scale** — up to **8192** zones per Solaris host.
- **Program isolation** — run each service in its own zone (e.g. `zone1` = Apache, `zone2` = MySQL) so they can't interfere with each other.
- **Managed with `z*` commands** — `zonecfg`, `zoneadm`, `zlogin`, `zonename`.

## Global vs Non-Global Zones

Solaris always has one **global zone** (the host itself) and zero or more **non-global zones** (the containers).

| | Global zone | Non-global zone |
|---|-------------|-----------------|
| Boots | Solaris always boots (cold/warm) to it | Started by the global zone after host boot |
| Hardware | Knows about all attached devices | No direct device access by default |
| Visibility | Aware of all non-global zones | Cannot see or reach other non-global zones |
| Packages | Master package set | Derive/share packages from the global zone |
| Admin | Full control; can delegate zone admin | Delegated admin scope only |

### Global Zone

- Solaris **always boots to the global zone** (cold or warm boot).
- Knows about **all hardware devices** attached to the system.
- Knows about **all non-global zones** and manages their lifecycle.
- Has access to all zones; its admin can **delegate** administration of individual non-global zones.

### Non-Global Zones

- Installed at a **zone root path** on the global zone's filesystem, conventionally under `/export/home/zones/{zone1,zone2,...}`.
- **Share packages** with the global zone by default (sparse-root style), saving disk and simplifying patching.
- Maintain their own **hostname and system files** (distinct identity).
- **Cannot communicate with other non-global zones by default** — traffic must go over a network interface using the standard TCP/IP network API, just like separate hosts.

## Configure a Zone (`zonecfg`)

Create the parent directory, then define the zone's configuration — zone path and a network interface here:

```bash
mkdir -p /export/home/zones
```

```bash
zonecfg -z testzone1
```

```
zonecfg:testzone1> create
zonecfg:testzone1> set zonepath=/export/home/zones/testzone1
zonecfg:testzone1> add net
zonecfg:testzone1:net> set address=10.10.10.113
zonecfg:testzone1:net> set physical=e1000g0
zonecfg:testzone1:net> set defrouter=10.10.10.235
zonecfg:testzone1:net> info
zonecfg:testzone1:net> end
zonecfg:testzone1> verify
zonecfg:testzone1> commit
zonecfg:testzone1> exit
```

- `create` — start a new zone configuration.
- `set zonepath=...` — where the zone's root filesystem lives on the global zone.
- `add net` / `set address` / `set physical` / `set defrouter` — attach a network interface with an IP, physical NIC, and default router; `end` closes the resource.
- `verify` — check the config is valid; `commit` — save it; `exit` — leave `zonecfg`.

Review the resulting configuration:

```bash
zonecfg -z testzone1 info
```

### Shared-IP vs Exclusive-IP

A zone's network can be one of two IP types, set with `set ip-type`:

```
zonecfg:testzone1> set ip-type=shared        # default; shares the global zone's IP stack
zonecfg:testzone1> set ip-type=exclusive     # zone gets its own dedicated IP stack + NIC
```

| Type | Behavior | Use when |
|------|----------|----------|
| `shared` | Zone uses the global zone's TCP/IP stack; you assign an address on a shared NIC | Simple setups, many zones on one NIC |
| `exclusive` | Zone owns a dedicated data-link (physical or VNIC); manages its own `ipadm`/routing | Zone needs its own routing, DHCP, or firewall |

For an exclusive-IP zone you typically create a VNIC first and hand it to the zone:

```bash
# Global zone: create a virtual NIC over the physical link
dladm create-vnic -l e1000g0 vnic0
```

```
zonecfg:testzone1> set ip-type=exclusive
zonecfg:testzone1> add net
zonecfg:testzone1:net> set physical=vnic0
zonecfg:testzone1:net> end
```

### Resource Controls

Cap CPU and memory so one zone can't starve the others:

```
zonecfg:testzone1> add capped-cpu
zonecfg:testzone1:capped-cpu> set ncpus=2
zonecfg:testzone1:capped-cpu> end
zonecfg:testzone1> add capped-memory
zonecfg:testzone1:capped-memory> set physical=2g
zonecfg:testzone1:capped-memory> set swap=4g
zonecfg:testzone1:capped-memory> end
```

### Adding a Filesystem or Device

```
# Loopback-mount a global-zone directory into the zone (read-only example)
zonecfg:testzone1> add fs
zonecfg:testzone1:fs> set dir=/data
zonecfg:testzone1:fs> set special=/export/shared
zonecfg:testzone1:fs> set type=lofs
zonecfg:testzone1:fs> set options=[ro,nodevices]
zonecfg:testzone1:fs> end

# Delegate a raw device into the zone
zonecfg:testzone1> add device
zonecfg:testzone1:device> set match=/dev/rdsk/c0t3d0s0
zonecfg:testzone1:device> end
```

## Install and Boot (`zoneadm`)

```bash
# Install the zone (lays down its filesystem from the global zone's packages)
zoneadm -z testzone1 install

# Boot the zone
zoneadm -z testzone1 boot

# List zones with status and details
zoneadm list -iv
```

Sample `zoneadm list -iv` output:

```
  ID NAME             STATUS      PATH                          BRAND    IP
   0 global           running     /                             solaris  shared
   - testzone1        installed   /export/home/zones/testzone1  solaris  excl
```

Zone status values you'll see:

| Status | Meaning |
|--------|---------|
| `configured` | Config committed, not yet installed |
| `incomplete` | Install/uninstall in progress or interrupted |
| `installed` | Filesystem laid down, not running |
| `ready` | Kernel structures created, not yet booted |
| `running` | Booted and active |
| `down` / `shutting_down` | Halting |

Then complete first-boot configuration (hostname, root password, naming service, etc.) on the zone's console:

```bash
zlogin -C testzone1
```

## Manage a Running Zone

```bash
# Reboot a zone
zoneadm -z testzone1 reboot

# Shut down a zone (run shutdown inside it)
zlogin testzone1 shutdown

# Show physical and virtual network interfaces (global zone)
dladm show-link
```

### zlogin Modes

`zlogin` connects the global zone to a non-global zone in several ways:

| Mode | Command | Use |
|------|---------|-----|
| Interactive | `zlogin -l username zonename` | Log in as a specific user |
| Non-interactive | `zlogin zonename command` | Run one command in the zone |
| Console | `zlogin -C zonename` | Attach to the zone console (first-boot setup) |
| Safe | `zlogin -S zonename` | Minimal/safe login for recovery |

## Remove a Zone

Shut it down, uninstall its filesystem, then delete its configuration:

```bash
zoneadm -z testzone1 halt           # stop the zone (canonical, from the global zone)
zoneadm -z testzone1 uninstall -F   # -F = force, no prompt
zonecfg -z testzone1 delete -F
```

> `zoneadm -z NAME halt` is the standard way to stop a zone from the global zone. `zlogin NAME shutdown` (running `shutdown` inside the zone) works too but is a graceful in-zone shutdown rather than the direct halt.

## Remove a Network Interface from a Zone

Bring the interface down on the global zone, then remove it from the zone config:

```bash
# On the global zone — take the virtual interface down and unplumb it
ifconfig bge0:3 down
ifconfig bge0:3 unplumb
```

```bash
zonecfg -z zone1
```

```
zonecfg:zone1> remove net address=10.10.10.113
zonecfg:zone1> commit
zonecfg:zone1> exit
```

## Zone Lifecycle Summary

| Stage | Command | Resulting state |
|-------|---------|-----------------|
| Configure | `zonecfg -z NAME` … `commit` | Configured |
| Install | `zoneadm -z NAME install` | Installed |
| Boot | `zoneadm -z NAME boot` | Running |
| First-boot setup | `zlogin -C NAME` | Configured/ready |
| Reboot | `zoneadm -z NAME reboot` | Running |
| Shut down | `zoneadm -z NAME halt` (or `zlogin NAME shutdown`) | Installed (halted) |
| Uninstall | `zoneadm -z NAME uninstall -F` | Configured |
| Delete | `zonecfg -z NAME delete -F` | Removed |

## Command Reference

| Command | Purpose |
|---------|---------|
| `zonecfg -z NAME` | Create/edit a zone's configuration |
| `zonecfg -z NAME info` | Show a zone's configuration |
| `zoneadm -z NAME install` | Install the configured zone |
| `zoneadm -z NAME boot` | Boot the zone |
| `zoneadm -z NAME reboot` | Reboot the zone |
| `zoneadm list -iv` | List zones with status/details |
| `zlogin -C NAME` | Attach to the zone console |
| `zlogin NAME command` | Run a command in the zone |
| `zonename` | Print the current zone's name |
| `dladm show-link` | List physical/virtual network links |

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `zoneadm install` fails on zonepath | Permissions or wrong mode on zonepath | zonepath must be owned by root, mode `700`: `chmod 700 /export/home/zones/testzone1` |
| Zone stuck in `incomplete` | Interrupted install/uninstall | `zoneadm -z NAME uninstall -F` then reinstall |
| Zone won't boot, "IP already in use" | Address conflict with global or another host | Change the zone's `net` address; verify with `arp`/`ping` |
| `zlogin -C` shows nothing | Console not ready / zone not booted | Confirm `running`; press Enter; check `zoneadm list -iv` |
| Exclusive-IP zone has no network | VNIC not created or not assigned | `dladm show-vnic`; recreate VNIC and re-add to zone |
| Can't delete a zone | Still installed/running | Halt, `uninstall -F`, then `zonecfg ... delete -F` |

```bash
# Watch a zone's boot/console messages
zlogin -C testzone1
# detach from the console with: ~.   (tilde, dot)
```

## Notes

- **Exclusive-IP zones** were introduced in **Solaris 10 8/07** (update 4), not Solaris 11. Solaris 11 made exclusive-IP (over VNICs) the default and deepened ZFS integration (each zone gets its own ZFS dataset). The `zonepath` should sit on a ZFS filesystem.
- **Brands:** on Solaris 10, native zones show brand `native`; on Solaris 11 they show `solaris`. Solaris 11 also offers `solaris10` branded zones to run a Solaris 10 environment on an 11 host.
- Non-global zones are isolated by default — treat inter-zone traffic exactly like traffic between separate hosts (over TCP/IP), which is what makes the Apache-in-one-zone / MySQL-in-another pattern safe.

## References

- [Introduction to Oracle Solaris Zones](https://docs.oracle.com/cd/E37838_01/html/E61038/index.html) — official Oracle docs
- [Creating and Using Oracle Solaris Zones](https://docs.oracle.com/cd/E37838_01/html/E61039/index.html) — official Oracle docs
