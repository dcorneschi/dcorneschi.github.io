# AIX Error Logging and System Logs Cheatsheet

Command reference for logging and diagnostics on IBM AIX — reading boot/console logs with `alog`, working with the error log (`errpt` reports, `errclear` cleanup, `errlogger` messages), the error daemon (`errdemon`/`errstop`), diagnostic results (`diagrpt`), and the LVM trace/log files.

> Most of these commands require `root`. The error log daemon (`errdemon`) reads entries from `/dev/error` and writes them to `/var/adm/ras/errlog`. Clearing the log (`errclear 0`) is irreversible — capture a report first if you need the history.

## alog — Boot and Console Logs

`alog` reads and writes fixed-size, wrap-around log files (such as the boot and console logs) and manages their configuration in the ODM.

```sh
# List the defined log types
alog -L

# Show the properties of the boot log file
alog -L -t boot

# View the boot log
alog -o -t boot

# View the console log
alog -o -t console

# Increase the size of the boot log (bytes)
alog -C -t boot -s 8192
```

| Option | Purpose |
|--------|---------|
| `-L` | List defined log types (add `-t <type>` for one type's attributes) |
| `-o` | Output (display) the contents of the log |
| `-t` | Select the log type (`boot`, `console`, `nim`, ...) |
| `-C -s <size>` | Change the log size (in bytes) |

## errpt — Error Report

`errpt` generates reports from the error log. With no options it prints a one-line-per-entry summary; add detail and filter flags to narrow the output.

```sh
# Summary report (one line per entry)
errpt

# Intermediate report
errpt -A

# Detailed report
errpt -a

# Detailed report for a specific error label
errpt -a -J ERRLOG_ON

# Summary report of all HARDWARE errors
errpt -d H

# Detailed report of all SOFTWARE errors
errpt -a -d S

# Concurrent ("real-time") error logging — follow new entries as they arrive
errpt -c

# Concurrent error logging directed to the console
errpt -c > /dev/console

# All errors for a resource (e.g. hdisk0)
errpt -N hdisk0

# All errors logged today
errpt -s `date +%m%d0000%y`

# Details for a specific error identifier
errpt -aDj 16F35C72

# Error-record templates for which logging is turned OFF
errpt -t -F log=0
```

| Option | Purpose |
|--------|---------|
| `-a` | Detailed (full) report |
| `-A` | Concise/intermediate report |
| `-c` | Concurrent (real-time) logging — stream entries as they occur |
| `-d <class>` | Filter by error class: `H` hardware, `S` software, `O` operator, `U` undetermined |
| `-J <label>` | Include only entries with the given error label |
| `-j <id>` | Include only entries with the given error identifier |
| `-N <resource>` | Filter by resource name (e.g. `hdisk0`) |
| `-s <date>` | Entries after `mmddhhmmyy` |
| `-D` | Report duplicate/aggregated entries |
| `-t` | Read from the error-record template repository |
| `-F <attr=val>` | Filter templates by attribute (e.g. `log=0`) |

### Error classes and types

Every error-log entry carries a **class**, a **type** (severity), and a description. These are what the `-d` (class) and reporting filters key on.

| Class | Meaning |
|-------|---------|
| `H` | Hardware error |
| `S` | Software error |
| `O` | Operator (informational messages via `errlogger`) |
| `U` | Undetermined |

| Type | Severity |
|------|----------|
| `PEND` | Loss of availability of a device or component is imminent |
| `PERF` | Performance of the device or component has degraded |
| `PERM` | Permanent error — hardware/software could not recover (most severe) |
| `TEMP` | Temporary error — recovered after several attempts |
| `INFO` | Informational, not an error |
| `UNKN` | Severity could not be determined |

## diagrpt — Diagnostic Results

```sh
# Display previous diagnostic results
diagrpt
```

## errclear — Purge Error Log Entries

`errclear` deletes entries older than a given number of days; `0` clears everything of the matched selection.

```sh
# Clear ALL entries
errclear 0

# Clear entries older than seven days
errclear 7

# Clear all entries from an alternate error-log file
errclear -i /var/adm/ras/errlog.alternate 0

# Clear all HARDWARE entries from an alternate error-log file
errclear -i /var/adm/ras/errlog.alternate -d H 0

# Delete all SOFTWARE-classified entries
errclear -d S 0
```

| Option | Purpose |
|--------|---------|
| `<days>` | Delete entries older than N days (`0` = all matched) |
| `-i <file>` | Operate on an alternate error-log file |
| `-d <class>` | Restrict to an error class (`H`, `S`, `O`, `U`) |
| `-j` / `-J` | Restrict by error identifier / label |

## errlogger — Write to the Error Log

```sh
# Send an operator message to the error log
errlogger "Your message here"
```

## errdemon / errstop — Error Log Daemon

The `errdemon` daemon collects error records and writes them to the error log. It is normally started automatically at boot.

```sh
# Change error-log characteristics through SMIT
smit errdemon

# Change the error log size to ~2 MB
/usr/lib/errdemon -s 2000000

# List details about the error log (path, size, buffer)
/usr/lib/errdemon -l

# Start the daemon
/usr/lib/errdemon

# Stop the daemon
/usr/lib/errstop
```

## LVM Log and Trace Files

The Logical Volume Manager keeps its own logs and traces, useful when debugging VG/LV operations.

| File | Path | Purpose |
|------|------|---------|
| `lvmcfg` | `/var/adm/ras/lvmcfg.log` | Records the high-level LVM commands that were run |
| `lvmt` | `/tmp/lvmt.log` | Trace for LVM commands and libraries |
| `lvmgs` | `/tmp/lvmgs.log` | Trace for the `gsclvmd` (concurrent LVM) daemon |

## Quick Reference

| Task | Command |
|------|---------|
| List log types | `alog -L` |
| View boot log | `alog -o -t boot` |
| View console log | `alog -o -t console` |
| Resize boot log | `alog -C -t boot -s 8192` |
| Summary report | `errpt` |
| Detailed report | `errpt -a` |
| Hardware errors | `errpt -d H` |
| Software errors (detailed) | `errpt -a -d S` |
| Real-time logging | `errpt -c` |
| Errors for a resource | `errpt -N hdisk0` |
| Details for an error ID | `errpt -aDj 16F35C72` |
| Diagnostic results | `diagrpt` |
| Clear all entries | `errclear 0` |
| Clear entries > 7 days | `errclear 7` |
| Log a message | `errlogger "message"` |
| Resize error log | `/usr/lib/errdemon -s 2000000` |
| Error log details | `/usr/lib/errdemon -l` |
| Start / stop daemon | `/usr/lib/errdemon` / `/usr/lib/errstop` |

## Related

- [AIX System Dump and Core File Cheatsheet](articles/aix-system-dump-core-cheatsheet.md) — crash dumps and core files, which surface as entries in the error log.
- [AIX Performance Monitoring Cheatsheet](articles/aix-performance-monitoring-cheatsheet.md) — correlating logged errors with CPU, memory, and disk activity.
- [AIX MPIO and Fibre Channel Cheatsheet](articles/aix-mpio-fibre-channel-cheatsheet.md) — many hardware (`H`) errors relate to FC adapters and MPIO paths.
