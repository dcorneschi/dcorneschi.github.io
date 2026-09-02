# HP iLO CLI Cheatsheet (SSH / SMASH CLP)

Command reference for managing HP/HPE servers through the iLO (Integrated Lights-Out) baseboard management controller over SSH. iLO exposes a SMASH CLP command shell you reach by SSHing to the iLO's hostname or IP — useful for resetting the controller, recovering a stuck virtual serial port, controlling power, and pushing firmware, all without OS access.

iLO is HP/HPE's out-of-band management processor — the equivalent of Dell's iDRAC. The CLP tree is navigated like a filesystem (`cd`, `show`), with targets such as `/map1` (the management processor) and `/system1` (the host server).

## Connecting

```sh
ssh admin@<hostname-or-ip>
```

Once connected you're at the `</>` CLP prompt. Explore the target tree with `show`:

```sh
show            # list targets and properties at the current node
cd /system1     # move to the server (host) node
cd /map1        # move to the management processor (iLO) node
help            # list available verbs
exit            # close the session
```

## Reset the iLO Controller

Resets the management processor itself (not the host OS). Handy when the iLO web UI is unresponsive.

```sh
ssh admin@<hostname-or-ip>

cd /map1
reset
```

The SSH session drops when the iLO restarts; give it a minute or two to come back before reconnecting. The host server keeps running — resetting `/map1` does not power-cycle the OS.

## Fix a Stuck Virtual Serial Port (VSP)

If connecting to the serial console returns *"Virtual Serial Port is currently in use by another session,"* a previous session didn't release it. Stop and restart the VSP to clear it:

```sh
stop  /system1/oemhp_vsp1
start /system1/oemhp_vsp1
```

Then reconnect to the serial console:

```sh
vsp             # attach to the virtual serial port
# press Esc + ( to exit the VSP session back to the CLP prompt
```

The VSP relays the host's serial console over iLO — useful for watching boot/GRUB output or logging into a headless server.

## Power Control

The host power target lives under `/system1`:

```sh
power                       # show current power state
power on                    # power the server on
power off                   # graceful shutdown request
power off hard              # force power off
power reset                 # power-cycle the server
```

You can also drive it explicitly via the power target's verbs:

```sh
cd /system1
start                       # power on
stop                        # power off
reset                       # reset/reboot
```

## Firmware Upgrade

HP firmware for Linux ships as a self-extracting `.scexe`. Unpack it to get the raw `.bin` image, then load it into iLO from an HTTP source.

```sh
# On a Linux host: unpack the firmware bundle
chmod 755 CP022551.scexe
./CP022551.scexe --unpack=/tmp/iLO

# Host the extracted .bin on a reachable web server (e.g. 192.168.1.1),
# then from the iLO CLP session:
cd /map1/firmware1
load -source http://192.168.1.1/firmware/ilo3_170.bin
```

- Match the firmware image to your iLO generation (`ilo3_*` for iLO 3, `ilo4_*` for iLO 4, etc.). Loading the wrong generation's image will be rejected.
- The `.bin` must be served over HTTP/HTTPS from a host the iLO can reach on the management network.
- iLO restarts after a successful flash; the host OS is unaffected.

## Common CLP Targets

| Target | What it represents |
|--------|--------------------|
| `/map1` | The iLO management processor itself |
| `/map1/firmware1` | iLO firmware (load/update) |
| `/map1/config1` | iLO configuration |
| `/system1` | The host server (power, indicators) |
| `/system1/oemhp_vsp1` | Virtual Serial Port relay to the host console |
| `/system1/oemhp_power1` | Host power management/metering |

## Verification and Troubleshooting

```sh
# Confirm iLO is back after a reset
ssh admin@<hostname-or-ip> show /map1

# Check host power state
ssh admin@<hostname-or-ip> power

# Show iLO firmware version
show /map1/firmware1
```

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Virtual Serial Port is currently in use" | Prior VSP session not released | `stop` then `start /system1/oemhp_vsp1` |
| iLO web UI unresponsive, SSH still works | iLO processor hung | `cd /map1` then `reset` |
| `load` firmware rejected | Wrong iLO generation image or unreachable URL | Use the matching `ilo<N>` image; verify HTTP source reachability |
| Can't exit the VSP session | Escape sequence needed | Press `Esc` then `(` to return to CLP |
| SSH connection refused | iLO still restarting after reset | Wait 1–2 minutes and reconnect |

## iLO vs Dell iDRAC (Quick Mapping)

| Task | HP iLO (CLP) | Dell iDRAC (racadm) |
|------|--------------|---------------------|
| Reset the BMC | `cd /map1` → `reset` | `racadm racreset` |
| Power on/off/cycle | `power on` / `off` / `reset` | `racadm serveraction powerup/powerdown/powercycle` |
| Serial console | `vsp` (`oemhp_vsp1`) | `racadm connect com2` / SOL |
| Firmware update | `load -source http://.../ilo<N>.bin` | `racadm fwupdate` |

## References

- [HPE iLO documentation](https://www.hpe.com/info/ilo/docs) — official HPE docs
- [HPE iLO scripting and CLI guide](https://support.hpe.com/hpesc/public/home) — official HPE support portal
- [DMTF SMASH CLP specification](https://www.dmtf.org/standards/smash) — the standard iLO's CLI implements
