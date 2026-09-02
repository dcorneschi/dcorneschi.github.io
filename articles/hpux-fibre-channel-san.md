# HP-UX Fibre Channel and SAN Storage

Managing Fibre Channel HBAs and SAN LUNs on HP-UX — the `fcmsutil` HBA tool, scanning for and removing LUNs, PowerPath vs native multipathing, setting the load-balancing policy with `scsimgr`, and I/O statistics. Covers HP-UX 11i v1–v3. For general device discovery and DSFs, see [HP-UX Device Management (ioscan, scsimgr, DSFs)](articles/hpux-device-management-ioscan.md).

## Concepts: Fibre Channel and SAN Building Blocks

A SAN presents block storage to a host over a Fibre Channel fabric rather than a local bus. A handful of terms explain almost everything you do here:

- **HBA (Host Bus Adapter)** — the FC card in the server. Each port has a globally unique **WWN** (World Wide Name): a **WWNN** identifies the node (the adapter) and a **WWPN** identifies each individual port. Zoning and array masking are done by WWPN, so knowing a port's WWPN (`fcmsutil`) is step one of any SAN provisioning task.
- **Fabric and switches** — the FC switches between host and array. **Zoning** is configured on the switch and controls which WWPNs are allowed to see each other; it is the SAN equivalent of a firewall rule.
- **LUN (Logical Unit Number)** — a slice of storage the array publishes. **LUN masking** on the array decides which host WWPNs may access which LUNs. A LUN only appears to HP-UX after it is both *zoned* (switch) and *masked/presented* (array).
- **Multipathing** — because a host usually has two HBAs into two fabrics for redundancy, the same LUN is reachable by multiple physical **paths** (lunpaths). Multipathing software presents those paths as one logical device and spreads or fails over I/O across them.

The practical consequence: when a new LUN "does not show up," the fault is almost always one layer down — a missing zone or an incomplete array presentation — not HP-UX. Verify the WWPN is zoned and masked before scanning repeatedly on the host.

### Agile vs legacy device model (11i v3)

HP-UX 11i v3 introduced the **agile** naming model. In the older (legacy) model a device file encoded the hardware path — controller/target/LUN — so a LUN seen down two paths appeared as two different `c#t#d#` devices and moving a cable renamed the device. The agile model gives each LUN a single persistent identifier (`/dev/disk/diskN` for the block DSF, `/dev/rdisk/diskN` raw) regardless of how many paths reach it, and it survives cabling changes. Native multipathing is built into this agile stack, which is why on 11i v3 you rarely need add-on multipath software. Both naming models can coexist; `ioscan` has separate legacy and agile (`-N`) views.

## Fibre Channel HBAs (fcmsutil)

```bash
# List FC adapters
ioscan -funC fc

# Enable / disable an HBA (by its device file)
fcmsutil /dev/fcd1 enable
fcmsutil /dev/fcd1 -f disable

# Link statistics
fcmsutil /dev/fcd2 stat -s              # summary
/opt/fcms/bin/fcmsutil /dev/fclp0 stat -s
/opt/fcms/bin/fcmsutil /dev/fclp0 clear_stat    # zero the counters

# Devices attached to all HBAs
tdlist
```

`fcmsutil` (in `/opt/fcms/bin`) is the Fibre Channel Mass Storage utility — it reports the HBA's WWN, link state, topology, speed, and driver stats, and lets you enable/disable a port. Disabling an HBA is useful before pulling a cable or doing fabric maintenance so I/O fails over to the other path.

```bash
fcmsutil /dev/fcd0                      # full HBA info (WWN, state, speed, topology)
```

Reading the output is a skill in itself. The fields you use most:

| Field | What it tells you |
|-------|-------------------|
| `N_Port Port World Wide Name` | The port's **WWPN** — the value to give the SAN/array team for zoning and masking |
| `Node World Wide Name` | The adapter's **WWNN** |
| `Topology` | `PTTOPT_FABRIC` (switched fabric) vs private/arbitrated loop — fabric is normal in a modern SAN |
| `Link Speed` | Negotiated speed (2/4/8 Gb). A port stuck below its rated speed hints at a bad SFP or cable |
| `Local N_Port_id` | The fabric address (FCID) assigned by the switch — non-zero means the port logged into the fabric |

A quick health rule: if `fcmsutil` shows no `N_Port_id` and topology is unknown, the port never logged into the fabric — suspect the cable, SFP, or an unconfigured switch port before touching HP-UX. The link statistics (`stat`) counters for loss-of-sync, loss-of-signal, and CRC errors are the first place to look for a flaky physical link; clear them with `clear_stat` and watch whether they climb again.

## Scanning for New LUNs

After presenting a LUN on the array and zoning it, make HP-UX discover it:

```bash
ioscan -C disk                          # rescan the disk class
insf -eC disk                           # create device special files for new disks
powermt config                          # (if EMC PowerPath is installed) build pseudo devices
```

Confirm the new LUN and its paths (see the device-management article for `ioscan -m lun` / health):

```bash
ioscan -NfC disk                        # agile view of disks/LUNs
ioscan -m lun                           # LUNs and lunpaths
ioscan -P health -C lunpath             # per-path health (online/offline/unusable)
```

The order of operations matters. `ioscan` triggers the driver to probe the bus and find hardware; `insf -e` then (re)creates the device special files so the new hardware is actually usable. Running `insf` without a preceding `ioscan` on a genuinely new LUN finds nothing, which is a common source of "I scanned and it's still not there." On 11i v3, `ioscan -NfC disk` gives the agile per-LUN view while `ioscan -m lun` shows how many lunpaths back each LUN — you want to see *all* expected paths (typically two) before you build a volume group, otherwise you have provisioned a single point of failure.

If a LUN still does not appear after scanning, stop and verify zoning/masking rather than re-scanning: capture the HBA WWPN with `fcmsutil /dev/fcdN` and confirm with the storage team that this exact WWPN is zoned to the array and the LUN is presented to it.

## Identifying a Disk / LUN

```bash
ioscan -mdsf /dev/rdisk/disk69          # map a DSF to its hardware path(s)
```

## Multipath Load Balancing (native, scsimgr)

On 11i v3 the native mass-storage stack multipaths LUNs automatically; control the per-LUN policy with `scsimgr`:

```bash
# Check the current load-balancing policy
scsimgr get_attr -D /dev/rdisk/disk69 -a load_bal_policy

# Change it (e.g. round-robin across paths)
scsimgr set_attr -D /dev/rdisk/disk69 -a load_bal_policy=round_robin
```

Common `load_bal_policy` values:

| Policy | Behavior |
|--------|----------|
| `round_robin` | Rotate I/O evenly across all active paths — the usual default for symmetric active/active arrays |
| `least_cmd_load` | Send each I/O down the path with the fewest outstanding commands — adapts to uneven path speeds |
| `cl_round_robin` | Cell-local round robin — on cell-based servers, prefer paths through the local cell to reduce cross-cell traffic |
| `preferred_path` | Pin I/O to one nominated path; fail over only if it dies |
| `path_lockdown` | Restrict to a fixed path set (used for controlled/asymmetric configurations) |

Match the policy to the array. For an **active/active** array where any controller can service any LUN, `round_robin` or `least_cmd_load` maximizes throughput. For an **active/passive** (ALUA) array where a LUN is owned by one controller at a time, blindly round-robining across both controllers causes constant, expensive LUN trespass; there the native stack honors the array's optimized paths, and you generally leave it on the default rather than forcing `round_robin`. You can set a policy per-LUN, so it is fine to tune a hot database LUN differently from bulk storage.

```bash
# See all path-related attributes for a LUN at once
scsimgr get_attr -D /dev/rdisk/disk69
```

## I/O Statistics

```bash
iostat -L                               # per-active-lunpath I/O statistics
iostat 2 5                              # overall device I/O, every 2s
```

`iostat -L` breaks the numbers down per **lunpath**, which is how you confirm I/O is actually spread across paths under a round-robin policy.

## Removing LUNs

Work top-down so nothing is using the device when you remove it. This example uses EMC PowerPath (`powermt`):

```bash
# 1. Stop all I/O and unmount filesystems on the device
umount /data

# 2. Inspect and remove the PowerPath pseudo device
powermt display dev=all
powermt remove dev=emcpowerN
powermt release          # required — otherwise the pseudo device lingers in /dev

# 3. Remove/unpresent the LUN on the storage array (array admin tools)

# 4. Clean up stale HP-UX device files (11i v3)
ioscan -NfC disk
rmsf -x                  # remove DSFs for stale (now-absent) devices
```

> Skipping `powermt release` leaves the pseudo device visible even after the LUN is gone — a common cause of confusing leftover `emcpower*` entries.

## PowerPath vs Native Multipathing

| | Native (11i v3) | EMC PowerPath |
|---|-----------------|---------------|
| Multipathing | Built into the mass-storage stack (agile view) | Add-on product, creates `emcpower*` pseudo devices |
| Load-balance control | `scsimgr ... load_bal_policy` | `powermt set policy` |
| LUN removal | `rmsf -x` after unpresenting | `powermt remove` + `powermt release` |
| Path status | `ioscan -P health -C lunpath` | `powermt display` |

On 11i v3 the native stack multipaths automatically, so PowerPath is often unnecessary; PowerPath is common on older releases or where an EMC array standard requires it.

## Command Reference

| Task | Command |
|------|---------|
| List FC HBAs | `ioscan -funC fc` |
| HBA info | `fcmsutil /dev/fcdN` |
| Enable / disable HBA | `fcmsutil /dev/fcdN enable` / `-f disable` |
| HBA stats / clear | `fcmsutil /dev/fclp0 stat -s` / `clear_stat` |
| Devices on all HBAs | `tdlist` |
| Scan for new LUNs | `ioscan -C disk` + `insf -eC disk` |
| PowerPath rescan | `powermt config` |
| Map DSF → hwpath | `ioscan -mdsf <dsf>` |
| Get/set balance policy | `scsimgr get_attr\|set_attr -D <dsf> -a load_bal_policy` |
| Per-path I/O stats | `iostat -L` |
| Remove PowerPath LUN | `powermt remove dev=` + `powermt release` |
| Clean stale DSFs | `rmsf -x` |

## Troubleshooting

### New LUN does not appear after scanning

Confirm the physical link and fabric login first, then check that the array actually presents the LUN to this host's WWPN:

```bash
fcmsutil /dev/fcd0                      # is the port logged in? note the WWPN
ioscan -fnC fc                          # HBA claimed and CLAIMED, not NO_HW
ioscan -NfC disk ; insf -e              # rescan disks, then create DSFs
```

If the HBA is healthy and logged into the fabric but no disk shows, the problem is upstream: zoning on the switch or masking on the array. Re-scanning HP-UX will not fix a zoning gap.

### A path is offline / LUN has fewer paths than expected

```bash
ioscan -P health -C lunpath             # which lunpath is offline?
scsimgr -f get_attr -D /dev/rdisk/disk69 -a state
```

An offline path usually means a dead cable/SFP, a disabled switch port, or a disabled HBA (`fcmsutil ... disable` left set). Re-enable the HBA with `fcmsutil /dev/fcdN enable` and recheck. After the physical fix, the native stack normally brings the path back automatically; if not, a rescan re-probes it.

### Stale device files after removing a LUN

If old `disk`/`lunpath` entries linger after a LUN is unpresented, they show as `NO_HW` in `ioscan`. Clean them so they do not clutter future scans:

```bash
ioscan -NfC disk                        # look for NO_HW entries
rmsf -x                                 # remove DSFs for absent (stale) devices
```

Do this only *after* the LUN is genuinely unpresented on the array — removing DSFs for a LUN that is merely temporarily offline just recreates work when it returns.

### Queue depth and throughput

If a busy LUN shows high service times under load, the per-device queue depth may be the bottleneck. Inspect and tune it with `scsimgr` (raise cautiously and in line with array vendor guidance — too deep can overrun the array's port):

```bash
scsimgr get_attr -D /dev/rdisk/disk69 -a max_q_depth
scsimgr set_attr -D /dev/rdisk/disk69 -a max_q_depth=16
```

## Related Articles

- [HP-UX Device Management (ioscan, scsimgr, DSFs)](articles/hpux-device-management-ioscan.md) — device discovery, the agile model, and DSFs in depth
- [HP-UX LVM](articles/hpux-lvm.md) — building volume groups on top of SAN LUNs
- [HP-UX Filesystem Management (HFS, JFS/VxFS)](articles/hpux-filesystem-management.md) — filesystems on those volumes
- [HP-UX Performance Monitoring](articles/hpux-performance-monitoring.md) — reading iostat and diagnosing I/O bottlenecks
- [HP-UX Disaster Recovery](articles/hpux-disaster-recovery.md) — SAN-attached storage in recovery planning
