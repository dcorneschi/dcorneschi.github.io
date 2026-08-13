# Fibre Channel Error Statistics on Linux

Each Fibre Channel HBA exposes error counters through sysfs at `/sys/class/fc_host/host<N>/statistics/`. These are hardware-level error conditions detected by the FC port. Understanding what each counter means helps diagnose cabling, SFP, switch, and link problems before they cause I/O failures or data corruption.

## Where the Counters Live

```bash
# List all FC hosts
ls /sys/class/fc_host/

# View all statistics for a specific host
ls /sys/class/fc_host/host0/statistics/

# Read a specific counter (values are in hex)
cat /sys/class/fc_host/host0/statistics/invalid_crc_count
```

> **Note:** Counter values are reported in hexadecimal (e.g., `0x3a`). Use `printf '%d\n' $(cat file)` to convert to decimal.

## Counter Reset Behavior

All counters reset to zero when the driver loads (typically at boot). The `seconds_since_last_reset` file shows how long the counters have been accumulating. Use it to calculate error rates:

```bash
# Seconds since counters were reset
cat /sys/class/fc_host/host0/statistics/seconds_since_last_reset

# Calculate errors per hour
ERRORS=$(printf '%d' $(cat /sys/class/fc_host/host0/statistics/invalid_crc_count))
SECONDS=$(printf '%d' $(cat /sys/class/fc_host/host0/statistics/seconds_since_last_reset))
[ "$SECONDS" -gt 0 ] && echo "scale=4; $ERRORS / $SECONDS * 3600" | bc
```

> **Tip:** When comparing error rates between hosts or across reboots, always normalize by `seconds_since_last_reset`.

## Critical Error Counters

These three counters are the most important for diagnosing FC connectivity problems. If any of these are non-zero, investigate physical layer issues (cables, SFPs, switch ports).

### invalid_crc_count

**What it means:** The HBA received a frame that failed its CRC (Cyclic Redundancy Check) integrity check. The data was corrupted in transit.

**Common causes:**
- Bad or damaged fibre optic cable (bent, kinked, dirty connectors)
- Faulty or degraded SFP/GBIC transceiver
- Incompatible SFP (wrong wavelength, wrong speed)
- Switch port hardware failure
- Excessive cable length or signal attenuation

**Significance:** High. CRC errors indicate data corruption on the wire. Even though FC has retry mechanisms, persistent CRC errors increase latency and can lead to I/O timeouts.

### invalid_tx_word_count

**What it means:** The HBA received a transmission word (the smallest FC transmission unit — 4 characters, 40 bits) that could not be decoded. This includes code violations, invalid special code alignment, and disparity errors.

**Common causes:**
- Problem on the local link between the HBA and the directly-connected switch port
- Faulty SFP on either end
- Cable or connector damage
- Speed negotiation mismatch (e.g., 8Gbps HBA forced to 16Gbps link)
- EMI interference on copper FC links

**Significance:** High. Indicates a physical layer encoding problem on the immediate link segment. An elevated count points to the cable/SFP between the HBA and the first switch.

### error_frames

**What it means:** A frame was received that is not consistent with Fibre Channel protocol — malformed, unexpected delimiters, or violating the frame structure (SOF/EOF framing errors).

**Common causes:**
- General link instability
- Faulty switch ASIC or port buffer issues
- Overloaded switch causing frame drops and recovery
- Combination of the issues above

**Significance:** High. Indicates protocol-level errors that may trigger error recovery sequences on the link.

## Secondary Error Counters

These counters track events that can indicate problems but are also expected in small quantities during normal operations (reboots, cable swaps, fabric reconfigurations).

### link_failure_count

**What it means:** FC connectivity with the port was completely broken. The link went down and required re-initialization (NOS — Not Operational Sequence).

**Common causes:**
- Physical cable disconnection
- Switch port going offline (admin action, switch reboot, or failure)
- HBA or SFP failure
- Persistent loss_of_sync or loss_of_signal that wasn't recovered

**Significance:** Medium-High. A small number after a maintenance window is expected. Incrementing during normal operations indicates link instability.

### loss_of_signal_count

**What it means:** The HBA's receiver detected a complete absence of signal on the fibre — no light (or signal on copper). The physical layer link is interrupted.

**Common causes:**
- Cable physically disconnected (even momentarily)
- SFP pulled or failing
- Dirty fibre connectors (light not passing through)
- Cable break or severe bend causing total signal loss
- Switch port powered down

**Significance:** Medium. Expected during cable swaps or maintenance. If persistent during normal operations, check physical cabling and SFPs. Can escalate to link_failure_count.

### loss_of_sync_count

**What it means:** The HBA lost bit-level synchronization with the remote end. In FC, the receiver must lock onto the sender's transmission pattern — loss of sync means it can no longer reliably identify word boundaries in the data stream.

**Common causes:**
- Marginal signal quality (dirty connectors, aging SFP, long cable runs)
- Brief interruptions or jitter on the link
- Cable or SFP starting to degrade
- Speed mismatch during negotiation

**Significance:** Medium. A few during initialization or fabric changes is normal. Sustained incrementing during operations indicates degrading physical layer. Often precedes loss_of_signal or link_failure.

### prim_seq_protocol_err_count

**What it means:** An unexpected primitive sequence was received. Primitive sequences are special ordered sets transmitted repeatedly to signal port states (e.g., idle, link reset, offline). Receiving one out of context indicates a protocol state mismatch between the two ends.

**Common causes:**
- Firmware bugs on HBA or switch
- Link recovery gone wrong
- Timing issues during fabric login

**Significance:** Low-Medium. Rarely seen in isolation. Usually accompanies other error types.

### lip_count

**What it means:** Loop Initialization Primitive count. A LIP is essentially a bus reset on an FC-AL (Arbitrated Loop) topology. It forces all devices on the loop to re-initialize.

**Common causes:**
- Device added or removed from a loop
- Device or HBA reset
- Loop instability (bad device disrupting the loop)
- SCSI scan commands triggering LIP (`echo "1" > /sys/class/fc_host/hostX/issue_lip`)

**Significance:** Low on point-to-point (fabric) topologies (LIPs don't apply). On Arbitrated Loop topologies, frequent LIPs indicate instability. Each LIP causes a brief I/O pause for all devices on the loop.

> **Note:** Modern SAN environments use fabric (switched) topologies almost exclusively. LIP counters are mostly relevant for legacy FC-AL configurations.

### dumped_frames

**What it means:** Frames that were received but discarded (dumped) because the host couldn't process them in time — typically due to a lack of host buffers.

**Common causes:**
- Host under extreme memory pressure
- HBA driver buffer pools exhausted
- Very high I/O burst rates exceeding the HBA's buffering capacity
- Driver or firmware bugs

**Significance:** Low-Medium. In small numbers, indicates temporary resource pressure. Sustained dumped_frames suggest the host or HBA can't keep up with the workload.

## Viewing Statistics

### Quick View — All Hosts

```bash
# Show all non-zero error counters across all FC hosts
for host in /sys/class/fc_host/host*/statistics; do
    echo "=== $(basename $(dirname $host)) ==="
    for f in "$host"/*; do
        val=$(cat "$f" 2>/dev/null)
        name=$(basename "$f")
        # Skip counters that are -1 (unsupported) or 0x0 (zero)
        if [ "$val" != "0xffffffffffffffff" ] && [ "$val" != "0x0" ] && [ "$name" != "seconds_since_last_reset" ]; then
            printf "  %-35s %s (%d)\n" "$name" "$val" "$(printf '%d' $val)"
        fi
    done
done
```

### One-Liners

```bash
# Show critical counters for all hosts (decimal)
for f in /sys/class/fc_host/host*/statistics/{invalid_crc_count,invalid_tx_word_count,error_frames}; do
    [ -f "$f" ] && printf "%-70s %d\n" "$f" "$(printf '%d' $(cat $f))"
done

# Show all counters for host0
grep -r . /sys/class/fc_host/host0/statistics/ 2>/dev/null

# Check uptime of counters (how long since last reset)
for h in /sys/class/fc_host/host*/statistics/seconds_since_last_reset; do
    printf "%-60s %d seconds (%d days)\n" "$h" "$(printf '%d' $(cat $h))" "$(($(printf '%d' $(cat $h)) / 86400))"
done

# Convert all hex counters to decimal for a host
for f in /sys/class/fc_host/host0/statistics/*; do
    printf "%-40s %d\n" "$(basename $f)" "$(printf '%d' $(cat $f) 2>/dev/null)" 2>/dev/null
done
```

### HBA and Port Information

```bash
# Show port state (Online, Linkdown, etc.)
cat /sys/class/fc_host/host*/port_state

# Show port speed
cat /sys/class/fc_host/host*/speed

# Show port WWPN (World Wide Port Name)
cat /sys/class/fc_host/host*/port_name

# Show port WWNN (World Wide Node Name)
cat /sys/class/fc_host/host*/node_name

# Show supported speeds
cat /sys/class/fc_host/host*/supported_speeds

# Show port type (fabric, point-to-point, loop)
cat /sys/class/fc_host/host*/port_type

# Show remote port states
cat /sys/class/fc_remote_ports/rport-*/port_state

# All FC host info at a glance
for host in /sys/class/fc_host/host*; do
    echo "=== $(basename $host) ==="
    echo "  Port Name:  $(cat $host/port_name)"
    echo "  Node Name:  $(cat $host/node_name)"
    echo "  Port State: $(cat $host/port_state)"
    echo "  Speed:      $(cat $host/speed)"
    echo "  Port Type:  $(cat $host/port_type)"
done
```

### Using systool

```bash
# Requires sysfsutils package
sudo systool -c fc_host -v

# Show specific host
sudo systool -c fc_host -v host0
```

## Monitoring Script

Capture FC statistics over time to correlate with I/O issues:

```bash
#!/bin/bash
# fc-stats-monitor.sh — capture FC error stats every N seconds
# Usage: ./fc-stats-monitor.sh [interval_seconds] > fc-stats.log 2>&1 &

INTERVAL=${1:-5}

echo "# Hostname: $(hostname)"
echo "# Started: $(date)"
echo "# Interval: ${INTERVAL}s"
echo ""

while true; do
    echo "# $(date +%F_%T) uptime: $(uptime -p)"
    for f in /sys/class/fc_host/host*/statistics/{error_frames,invalid_crc_count,invalid_tx_word_count,link_failure_count,loss_of_signal_count,loss_of_sync_count}; do
        [ -f "$f" ] && printf "  %-70s %d\n" "$f" "$(printf '%d' $(cat $f) 2>/dev/null)"
    done
    echo ""
    sleep "$INTERVAL"
done
```

### Monitoring with a Rate Calculation

```bash
#!/bin/bash
# fc-error-rate.sh — show errors per second for critical counters

declare -A PREV

while true; do
    for host in /sys/class/fc_host/host*/; do
        hname=$(basename "$host")
        for counter in invalid_crc_count invalid_tx_word_count error_frames; do
            file="$host/statistics/$counter"
            [ -f "$file" ] || continue
            val=$(printf '%d' $(cat "$file") 2>/dev/null)
            key="${hname}_${counter}"
            if [ -n "${PREV[$key]}" ]; then
                diff=$((val - ${PREV[$key]}))
                if [ "$diff" -gt 0 ]; then
                    echo "$(date +%T) $hname $counter: +$diff (total: $val)"
                fi
            fi
            PREV[$key]=$val
        done
    done
    sleep 5
done
```

## Driver Support

Not all HBA drivers expose every counter. If a file contains `0xffffffffffffffff` (all bits set, -1 signed), the counter is not supported by that driver.

| Driver | Card Vendor | Counter Support |
|--------|-------------|-----------------|
| `lpfc` | Broadcom/Emulex | Most complete — supports nearly all counters |
| `qla2xxx` | Marvell/QLogic | Partial — some counters always read `0xffffffffffffffff` |
| `bfa` | Brocade | Good support for most counters |
| `zfcp` | IBM (System z) | Partial support |

```bash
# Check which driver an FC host uses
ls -la /sys/class/fc_host/host0/device/driver
# or
basename $(readlink -f /sys/class/fc_host/host0/device/driver)

# Check HBA model
cat /sys/class/fc_host/host0/symbolic_name
```

## Troubleshooting Workflow

### Step 1: Identify Non-Zero Counters

```bash
for f in /sys/class/fc_host/host*/statistics/{invalid_crc_count,invalid_tx_word_count,error_frames,link_failure_count,loss_of_signal_count,loss_of_sync_count}; do
    val=$(printf '%d' $(cat "$f" 2>/dev/null) 2>/dev/null)
    [ "$val" -gt 0 ] 2>/dev/null && echo "$f = $val"
done
```

### Step 2: Calculate Error Rate

A non-zero counter alone isn't necessarily a problem — it depends on how fast it's growing relative to uptime. A handful of errors over months of uptime is different from hundreds per hour.

```bash
# Errors per day (example for invalid_crc_count on host0)
CRC=$(printf '%d' $(cat /sys/class/fc_host/host0/statistics/invalid_crc_count))
UPTIME=$(printf '%d' $(cat /sys/class/fc_host/host0/statistics/seconds_since_last_reset))
DAYS=$((UPTIME / 86400))
echo "CRC errors: $CRC over $DAYS days = $(echo "scale=2; $CRC / $DAYS" | bc) errors/day"
```

### Step 3: Correlate With I/O Errors

```bash
# Check kernel messages for SCSI/FC errors
dmesg | grep -iE "scsi|fc|lpfc|qla2xxx|link down|rport"

# Check multipath path states
multipath -ll | grep -E "failed|faulty"

# Check remote port states
cat /sys/class/fc_remote_ports/rport-*/port_state
```

### Step 4: Check Physical Layer

```bash
# Port state should be "Online"
cat /sys/class/fc_host/host*/port_state

# Link speed — verify it matches expectations
cat /sys/class/fc_host/host*/speed

# Issue a LIP to force re-login (caution: briefly disrupts I/O)
echo 1 > /sys/class/fc_host/host0/issue_lip
```

### Step 5: Engage Storage/Network Team

If errors persist after reseating cables and cleaning connectors:
- Check switch port error counters (`porterrshow` on Brocade, `show interface` on Cisco MDS)
- Swap SFPs
- Try a different switch port
- Test with a known-good cable
- Check for firmware updates on HBA and switch

## Error Counter Summary

| Counter | Severity | Indicates |
|---------|----------|-----------|
| `invalid_crc_count` | **High** | Data corruption on the wire — bad cable, SFP, or switch port |
| `invalid_tx_word_count` | **High** | Encoding errors on the local link — cable/SFP between HBA and switch |
| `error_frames` | **High** | Protocol-level framing errors — general link instability |
| `link_failure_count` | Medium-High | Link went completely down — cable pull, switch issue, or SFP failure |
| `loss_of_signal_count` | Medium | Complete signal loss — cable disconnected or SFP dark |
| `loss_of_sync_count` | Medium | Bit-sync lost — degrading signal quality, aging components |
| `prim_seq_protocol_err_count` | Low-Medium | Unexpected primitive sequence — firmware or timing issue |
| `lip_count` | Low | Loop initialization — only relevant for FC-AL topologies |
| `dumped_frames` | Low-Medium | Frames dropped due to host buffer exhaustion |

## Key sysfs Paths

| Path | Purpose |
|------|---------|
| `/sys/class/fc_host/hostN/` | FC host adapter attributes |
| `/sys/class/fc_host/hostN/statistics/` | Error and traffic counters |
| `/sys/class/fc_host/hostN/port_state` | Online, Linkdown, etc. |
| `/sys/class/fc_host/hostN/port_name` | WWPN of the local port |
| `/sys/class/fc_host/hostN/speed` | Negotiated link speed |
| `/sys/class/fc_host/hostN/issue_lip` | Write `1` to trigger LIP |
| `/sys/class/fc_remote_ports/rport-H:B-R/` | Remote (target) port attributes |
| `/sys/class/fc_remote_ports/rport-H:B-R/port_state` | Remote port state |
| `/sys/class/fc_remote_ports/rport-H:B-R/dev_loss_tmo` | Seconds before device removal |
| `/sys/class/fc_remote_ports/rport-H:B-R/fast_io_fail_tmo` | Seconds before I/O fails |

## Best Practices

1. **Baseline after fresh boot** — record all counters shortly after boot as your reference point
2. **Monitor trends, not absolutes** — a few errors over months is normal; rapid growth is not
3. **Normalize by time** — always compare error rates (errors/hour) not raw totals
4. **Check both ends** — if the HBA shows errors, check the switch port counters too (`porterrshow` on Brocade)
5. **Clean connectors** — dirty fibre optic connectors are the most common cause of CRC errors
6. **Replace SFPs in pairs** — if one end has a degraded SFP, the other may be marginal too
7. **Correlate with I/O** — FC errors matter when they coincide with SCSI errors, path failures, or application timeouts
8. **Automate collection** — run a periodic stats capture script and retain logs for post-incident analysis
9. **Watch for cascading** — `loss_of_sync → loss_of_signal → link_failure` is a typical degradation sequence
10. **Check after maintenance** — always verify counters are stable after any physical SAN changes
