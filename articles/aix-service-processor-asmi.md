# AIX / Power Service Processor and ASMI

Reference notes for the IBM Power Systems **service processor (SP)** and the **Advanced System Management Interface (ASMI)** — what the service processor does, how it connects to the HMC and serial ports, SPCN power control, launching ASMI, and the Power firmware naming codes.

> The service processor is independent of the main processors and stays operational even when the system's CPUs are down, which is what makes remote diagnosis and power control possible. ASMI settings affect low-level system behavior (power, boot, surveillance) — change them deliberately.

## What the Service Processor Does

The service processor is an embedded controller (based on a PowerPC 405GP running its own internal operating system with programs and device drivers) that provides Reliability, Availability, and Serviceability (RAS) functions. Its main roles:

- **System initialization** — brings the hardware up before the main processors take over.
- **Connection to the HMC** — communicates with the Hardware Management Console over the network.
- **Web-based ASMI** — the Advanced System Management Interface for setting system flags.
- **Hardware error detection** — senses and reports faults.

Because it runs independently of the main processors, the SP can diagnose, check status, and sense operational conditions of a remote system **even when the main processor is inoperable**. It provides:

- Firmware and operating system surveillance
- Several remote power controls
- Environmental monitoring (only critical errors are supported under Linux)
- Reset and boot features
- Remote maintenance and diagnostics, including console mirroring
- The ability to **call home** to report surveillance failures, critical environmental faults, and critical processing faults

## Connectivity

### HMC over the network

The service processor connects to the HMC over the network and performs the vital RAS functions listed above on its behalf.

### SPCN ports

There are two **SPCN** (System Power Control Network) ports that control the power of the attached I/O subsystem. The SPCN control software runs on the service processor alongside the service processor software.

### Serial port access

When **no HMC** is connected to the service processor, two serial ports provide terminal access:

| Serial port | Access |
|-------------|--------|
| Port 1 | Advanced System Management Interface (ASMI) |
| Port 2 | Receives boot sequence information only |

When the **HMC is connected, the serial ports are disabled**.

## Launching ASMI

ASMI is the web-based (and CLI-accessible) interface to service processor settings. Default credentials in many environments are `admin`/`admin`; connect to the HMC as `hscroot` first.

```sh
# From the HMC command line (ssh -X hscroot@<hmc>), launch the ASM menu for a system
asmmenu --ip 10.128.128.251
```

From the HMC GUI:

```
Select Tasks > Operations > Advanced System Management (ASM)
```

## Power Firmware Naming Codes

IBM Power system microcode (firmware) uses a naming convention that encodes the processor generation and market segment.

### POWER5

- **SF** — "Squadron Firmware"

### POWER6

| Code | Segment |
|------|---------|
| `EH` | Enterprise High-End |
| `EM` | Enterprise Mid-Range (formerly Intermediate-High) |
| `EL` | Enterprise Low-End |

### POWER7

| Code | Models |
|------|--------|
| `AL` | 750, 755 |
| `AM` | 770, 780 |
| `AH` | 775, 790, 795 (ultra high-end) |

See also the [AIX Power Systems, LPAR, and Boot Concepts](articles/aix-power-lpar-boot-concepts.md) article for firmware update commands (`lsmcode`, `invscout`, `update_flash`) and broader Power/LPAR background.

## Quick Reference

| Item | Detail |
|------|--------|
| SP base | Embedded PowerPC 405GP controller with its own OS |
| SP roles | Init, HMC connection, ASMI, hardware error detection |
| SPCN ports | Two ports; control power of the attached I/O subsystem |
| Serial port 1 | ASMI terminal access (HMC not connected) |
| Serial port 2 | Boot sequence information only |
| Serial ports + HMC | Disabled when the HMC is connected |
| Launch ASMI (HMC CLI) | `asmmenu --ip <system-ip>` |
| Launch ASMI (HMC GUI) | Tasks > Operations > Advanced System Management (ASM) |

## Related

- [AIX Power Systems, LPAR, and Boot Concepts](articles/aix-power-lpar-boot-concepts.md) — the boot process the service processor kicks off, plus firmware update commands.
- [AIX System Dump and Core File Cheatsheet](articles/aix-system-dump-core-cheatsheet.md) — RAS diagnostics that complement the service processor's error detection.
