# Understanding the Windows Battery Report

Windows includes a built-in tool that generates a detailed HTML report about your battery's health, usage patterns, and estimated life. It's one of the best ways to check if your battery is degrading or if something is draining it abnormally.

---

## Generating the Report

```powershell
# Run in an elevated (admin) or regular PowerShell/CMD
powercfg /batteryreport

# Output goes to: C:\Users\<username>\battery-report.html
```

You can also specify a custom output path:

```powershell
powercfg /batteryreport /output "D:\Reports\battery.html"
```

Open the resulting HTML file in any browser to view it.

---

## Report Sections Explained

### 1. System Information

The header shows:

| Field | What It Means |
|-------|---------------|
| Computer Name | Machine hostname |
| System Product Name | Manufacturer and model |
| BIOS | Firmware version and date |
| OS Build | Windows build string |
| Platform Role | Mobile = laptop, Desktop = no battery |
| Connected Standby | Whether Modern Standby is supported |

### 2. Installed Batteries

Key fields:

| Field | What It Means |
|-------|---------------|
| **Design Capacity** | The battery's original rated capacity (mWh) |
| **Full Charge Capacity** | What the battery can actually hold today (mWh) |
| **Cycle Count** | Number of full charge/discharge cycles (may show "-" on some hardware) |
| Chemistry | Battery type — LiP (Lithium Polymer), Li-I (Lithium Ion) |

**How to assess battery health:**

```
Health % = (Full Charge Capacity / Design Capacity) × 100
```

- 95-100% → Excellent, like new
- 80-95% → Normal wear
- 60-80% → Noticeable degradation, consider replacement
- Below 60% → Battery is worn out

### 3. Recent Usage

A log of power state transitions over the last 7 days. Each row shows:

| Column | Meaning |
|--------|---------|
| Start Time | When the state began |
| State | Active, Suspended, Connected Standby, or Battery Changed |
| Source | AC (plugged in) or Battery |
| Capacity Remaining | Percentage and mWh at that moment |

**States explained:**

- **Active** — screen on, system running
- **Connected Standby** — Modern Standby (low-power state, can still receive notifications/updates)
- **Suspended** — traditional sleep (S3), very low power
- **Battery Changed** — system detected a change in battery (plug/unplug or driver event)

### 4. Battery Usage (Drain Graph)

A visual graph showing battery percentage over time with a table below it. The table shows:

| Column | Meaning |
|--------|---------|
| Start Time | When this drain period started |
| State | Active or Connected Standby |
| Duration | How long this period lasted |
| Energy Drained | Percentage and mWh consumed |

Use this to identify:
- Which active sessions drain the most
- Whether Connected Standby drain is reasonable (should be very low)
- Time periods with unexpectedly high consumption

### 5. Usage History

Aggregated daily totals showing how long the system spent:
- On battery (active vs. connected standby)
- On AC power (active vs. connected standby)

Useful for understanding your daily usage pattern over weeks.

### 6. Battery Capacity History

Tracks **Full Charge Capacity** over time compared to **Design Capacity**. This is the key section for monitoring long-term battery degradation.

A healthy battery loses roughly 1-2% capacity per year under normal use. If you see rapid drops, something may be wrong (excessive heat, deep discharges, faulty charging circuitry).

### 7. Battery Life Estimates

Estimated runtime based on observed drain rates:

| Column | Meaning |
|--------|---------|
| At Full Charge — Active | Estimated hours of active use on a full charge |
| At Full Charge — Connected Standby | Estimated hours of standby |
| At Design Capacity — Active | What the estimate would be if the battery were new |
| At Design Capacity — Connected Standby | Same, for standby |

The "Since OS install" row at the bottom is the most reliable long-term estimate, as it averages all observed data.

---

## Interpreting Drain Rates

### Active Use

Typical drain rates vary by workload:

| Workload | Typical Drain Rate |
|----------|-------------------|
| Idle / light browsing | 3-5% per hour |
| Office work / coding | 5-8% per hour |
| Video playback | 7-10% per hour |
| Heavy compilation / gaming | 15-30% per hour |

### Connected Standby

- **Good:** < 1% per hour (< 5% overnight)
- **Acceptable:** 1-2% per hour
- **Problematic:** > 2% per hour — something is waking the system

If standby drain is too high, investigate with:

```powershell
# Show what can wake the system
powercfg /waketimers

# Show standby/sleep transitions
powercfg /sleepstudy

# Show devices that can wake the system
powercfg /devicequery wake_armed
```

---

## Common Issues and What to Look For

### Battery degrading faster than expected

Check the "Battery Capacity History" section. If full charge capacity drops more than 5% in a few months:
- Avoid leaving the laptop plugged in at 100% constantly
- Avoid deep discharges below 10%
- Check for excessive heat (throttling, blocked vents)
- Some Lenovo/Dell/HP laptops have a "charge threshold" setting (e.g., stop charging at 80%)

### High drain in Connected Standby

The "Battery Usage" table shows drain per standby session. If Connected Standby drains significant energy:

```powershell
# Generate a sleep study report
powercfg /sleepstudy
```

Common culprits: Wi-Fi staying active, background app refresh, Windows Update, USB devices preventing deep sleep.

### Battery life much shorter than expected

Compare your "Active" drain rate to the estimates. If estimated life is 8 hours but you get 4:
- Check which processes are consuming power (Task Manager → power usage column)
- Review display brightness (biggest single consumer)
- Check for runaway background processes
- Verify power plan settings: `powercfg /getactivescheme`

---

## Useful Related Commands

```powershell
# Full battery report
powercfg /batteryreport

# Sleep/standby analysis (Modern Standby systems)
powercfg /sleepstudy

# Energy efficiency report (finds power policy issues)
powercfg /energy

# Show current power plan details
powercfg /getactivescheme
powercfg /query

# List all wake timers
powercfg /waketimers

# Show available sleep states
powercfg /availablesleepstates

# Show last wake source
powercfg /lastwake
```

---

## Quick Health Check Workflow

1. Generate the report: `powercfg /batteryreport`
2. Open the HTML file
3. Check **Full Charge Capacity vs Design Capacity** → battery wear
4. Check **Battery Life Estimates → Active** → expected runtime
5. Check **Battery Usage** table → any abnormally high drains in standby?
6. If standby drain is high → run `powercfg /sleepstudy` for details

---

## Tips for Battery Longevity

- Keep charge between 20-80% when possible (use manufacturer charge thresholds if available)
- Avoid sustained high temperatures (keep vents clear, use a laptop stand)
- If docked/plugged in most of the time, enable a charge limit (Lenovo Vantage, Dell Power Manager, etc.)
- Update BIOS/EC firmware — battery management improvements are common in firmware updates
- Hibernate instead of sleep for long idle periods if Connected Standby drain is high
