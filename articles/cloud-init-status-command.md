# cloud-init status: Understanding Status, Errors, and Failure Modes

## Overview

The `cloud-init status` command reports the current state of cloud-init — whether it's running, finished successfully, or encountered errors. Starting with version 23.4, cloud-init introduced a three-level exit code system that distinguishes between success, recoverable errors, and critical failures. Understanding these states is essential for automation scripts that wait on cloud-init and need to decide whether to proceed or abort.

## Basic Usage

```bash
# Simple status check — returns a one-word status
cloud-init status

# Wait for cloud-init to finish (blocks until complete)
cloud-init status --wait

# Detailed human-readable output
cloud-init status --long

# Machine-readable JSON output (most useful for scripting)
cloud-init status --format json
```

## Exit Codes

| Exit Code | Meaning | Action |
|-----------|---------|--------|
| `0` | Success — cloud-init completed without errors | Safe to proceed |
| `1` | Critical failure — cloud-init crashed or could not complete | Instance is likely broken, investigate immediately |
| `2` | Recoverable error — cloud-init completed but something went wrong | Investigate, but instance may be usable |

> **Note:** Exit code 2 was introduced in cloud-init 23.4. Older versions only returned 0 (success) or 1 (failure), making it impossible to distinguish between a clean run and one that logged errors but completed.

## Status Values

The `cloud-init status` command reports one of these statuses:

| Status | Description |
|--------|-------------|
| `not started` | cloud-init has not begun execution yet |
| `running` | cloud-init is currently executing |
| `done` | cloud-init completed successfully |
| `error - done` | cloud-init finished but experienced a critical (unrecoverable) error |
| `error - running` | cloud-init is still running but has already hit a critical error |
| `degraded done` | cloud-init finished with recoverable errors |
| `degraded running` | cloud-init is still running but has hit recoverable errors |
| `disabled` | cloud-init is disabled and will not run |

## Extended Status Output (JSON)

The `--format json` flag provides the most complete picture of what happened during cloud-init execution:

```bash
cloud-init status --format json
```

```json
{
  "boot_status_code": "enabled-by-generator",
  "datasource": "lxd",
  "detail": "DataSourceLXD",
  "errors": [],
  "extended_status": "degraded done",
  "init-local": {
    "errors": [],
    "finished": 1708550838.0196638,
    "recoverable_errors": {},
    "start": 1708550837.7719762
  },
  "init": {
    "errors": [],
    "finished": 1708550839.1837437,
    "recoverable_errors": {},
    "start": 1708550838.6881146
  },
  "modules-config": {
    "errors": [],
    "finished": 1708550843.8297973,
    "recoverable_errors": {
      "WARNING": [
        "Removing /etc/apt/sources.list to favor deb822 source format"
      ]
    },
    "start": 1708550843.7163966
  },
  "modules-final": {
    "errors": [],
    "finished": 1708550844.0884337,
    "recoverable_errors": {},
    "start": 1708550844.029698
  },
  "last_update": "Wed, 21 Feb 2024 21:27:24 +0000",
  "recoverable_errors": {
    "WARNING": [
      "Removing /etc/apt/sources.list to favor deb822 source format"
    ]
  },
  "stage": null,
  "status": "done"
}
```

### Key Fields

| Field | Description |
|-------|-------------|
| `status` | Overall status (`done`, `running`, `error`, `disabled`) |
| `extended_status` | Detailed status including degraded states |
| `errors` | List of critical (non-recoverable) errors |
| `recoverable_errors` | Errors grouped by severity level (`WARNING`, `DEPRECATED`, `ERROR`, `CRITICAL`) |
| `boot_status_code` | How cloud-init was enabled or disabled |
| `datasource` | Which cloud datasource was detected |
| `stage` | Currently executing stage (`null` if complete) |
| `init-local` | Per-stage errors and timing for the local stage |
| `init` | Per-stage errors and timing for the network/init stage |
| `modules-config` | Per-stage errors and timing for the config stage |
| `modules-final` | Per-stage errors and timing for the final stage |

## Critical Errors (Exit Code 1) — cloud-init Stops

Critical failures happen when cloud-init hits a condition it cannot safely handle. When this occurs, **cloud-init may not complete**, and subsequent modules or stages may never execute — meaning your `runcmd`, package installs, or service configurations will not run.

### What Causes Critical Failures

| Cause | Description | Impact |
|-------|-------------|--------|
| Datasource failure | Cannot contact or parse the metadata service (IMDS timeout, broken metadata) | cloud-init cannot determine instance identity, networking, or user-data |
| Corrupt or missing cloud-config | Malformed YAML that fails to parse entirely | No user configuration is applied |
| Broken cloud image | Missing core dependencies (Python libraries, `/var/lib/cloud` structure) | cloud-init cannot run at all |
| Permission errors on state directories | `/var/lib/cloud` or `/run/cloud-init` not writable | cloud-init cannot persist state or coordinate stages |
| Severe internal bug | Unhandled exception in core cloud-init code | Depends on where the crash occurs |
| Network stage timeout | Network never becomes available (broken DHCP, missing drivers) | Stages after `init-local` cannot proceed |
| Schema violation in critical modules | Some modules abort on invalid config (e.g., malformed `network:` config) | Networking may not be configured, blocking everything downstream |

### Example: Critical Error in JSON Output

```json
{
  "extended_status": "error - done",
  "errors": [
    "Traceback (most recent call last):\n  ...\nException: Failed to find datasource"
  ],
  "init-local": {
    "errors": [
      "Traceback (most recent call last):\n  ...\nException: Failed to find datasource"
    ],
    "recoverable_errors": {}
  },
  "status": "error"
}
```

### What Happens When a Critical Error Occurs

1. The current stage aborts
2. Depending on where the failure occurs, **subsequent stages may not run at all**
3. `cloud-init status` returns exit code 1
4. The instance may be inaccessible (no SSH keys deployed, no users created, no networking)

> **Important:** If cloud-init fails critically during `init-local` or `init`, modules like `users-groups`, `ssh`, `runcmd`, `packages`, and `write_files` will **not execute**. This means no SSH access, no custom users, and no software installed.

## Recoverable Errors (Exit Code 2) — cloud-init Continues

Recoverable errors are logged issues that did not prevent cloud-init from completing. The instance is likely functional, but some configuration may not have been applied.

### What Causes Recoverable Errors

| Cause | Description |
|-------|-------------|
| Invalid config for a non-critical module | A module's config doesn't match the expected schema, module is skipped |
| Missing templates | Template files referenced but not found on disk |
| Package installation failures | A package from `packages:` couldn't be installed (repo unreachable, package doesn't exist) |
| Failed runcmd commands | A command in `runcmd` exits non-zero |
| Deprecated configuration | Using deprecated keys that still work but generate warnings |
| apt/yum mirror timeouts | Package mirror temporarily unavailable |
| Partial write_files failures | Permission denied writing to a specific path |
| Failed phone_home | Callback URL unreachable |

### Recoverable Error Severity Levels

Recoverable errors are grouped by log level:

| Level | Meaning |
|-------|---------|
| `WARNING` | Something unexpected happened but is not necessarily a problem |
| `DEPRECATED` | Configuration uses deprecated syntax that will be removed in a future release |
| `ERROR` | Something definitively went wrong, but cloud-init continued |
| `CRITICAL` | A serious issue occurred, but cloud-init was able to finish other tasks |

### Example: Recoverable Errors in JSON Output

```json
{
  "extended_status": "degraded done",
  "errors": [],
  "recoverable_errors": {
    "WARNING": [
      "Failed at merging in cloud config part from p-01: empty cloud config",
      "No template found in /etc/cloud/templates for template sources.list"
    ],
    "ERROR": [
      "Failed to install package 'nonexistent-pkg'"
    ]
  },
  "status": "done"
}
```

## Enablement Status (boot_status_code)

The `boot_status_code` field tells you **why** cloud-init is enabled or disabled:

| Code | Meaning |
|------|---------|
| `enabled-by-generator` | ds-identify found a valid datasource |
| `enabled-by-kernel-cmdline` | Kernel cmdline contains `cloud-init=enabled` |
| `enabled-by-sysvinit` | Enabled by default in SysV init |
| `disabled-by-marker-file` | `/etc/cloud/cloud-init.disabled` exists |
| `disabled-by-generator` | ds-identify found no applicable datasource |
| `disabled-by-kernel-cmdline` | Kernel cmdline contains `cloud-init=disabled` |
| `disabled-by-environment-variable` | `KERNEL_CMDLINE` env var contains `cloud-init=disabled` |
| `unknown` | ds-identify hasn't run yet |

## Per-Stage Error Identification

The JSON output includes per-stage keys so you can pinpoint **when** an error occurred:

```bash
# Check which stage had errors
cloud-init status --format json | jq '{
  "init-local": .["init-local"].errors,
  "init": .init.errors,
  "modules-config": .["modules-config"].errors,
  "modules-final": .["modules-final"].errors
}'
```

The stages run in this order:

```
init-local      → Local stage (no network, applies network config)
init            → Network stage (bootcmd, write_files, users, ssh)
modules-config  → Config stage (apt, locale, ntp, runcmd written)
modules-final   → Final stage (packages, scripts, runcmd executed)
```

A critical error in `init-local` or `init` is far more damaging than one in `modules-final`, because the later stages depend on the earlier ones completing.

## Practical Scripting Patterns

### Wait and Check (Recommended)

```bash
#!/bin/bash
# Wait for cloud-init and check exit code
cloud-init status --wait
rc=$?

case $rc in
  0)
    echo "cloud-init completed successfully"
    ;;
  1)
    echo "CRITICAL: cloud-init crashed, instance may be broken"
    # Dump errors for debugging
    cloud-init status --format json | jq '.errors'
    exit 1
    ;;
  2)
    echo "WARNING: cloud-init completed with recoverable errors"
    cloud-init status --format json | jq '.recoverable_errors'
    # Decide whether to proceed based on your requirements
    ;;
esac
```

### Check Specific Stage for Errors

```bash
#!/bin/bash
# Check if the final stage had errors (runcmd, packages)
errors=$(cloud-init status --format json | jq -r '.["modules-final"].errors[]' 2>/dev/null)

if [ -n "$errors" ]; then
  echo "Final stage had critical errors:"
  echo "$errors"
  exit 1
fi
```

### Python Pattern

```python
import json
import subprocess
import sys

result = subprocess.run(
    ["cloud-init", "status", "--format", "json"],
    capture_output=True, text=True
)

status = json.loads(result.stdout)

if result.returncode == 1:
    print(f"CRITICAL: {status.get('errors')}", file=sys.stderr)
    sys.exit(1)
elif result.returncode == 2:
    print(f"Recoverable errors: {status.get('recoverable_errors')}", file=sys.stderr)
    # continue with caution

# Check specific stage
init_errors = status.get("init", {}).get("errors", [])
if init_errors:
    print(f"Init stage failed: {init_errors}", file=sys.stderr)
    sys.exit(1)
```

### Terraform / Packer Integration

When using `cloud-init status --wait` in provisioners:

```hcl
# In a Terraform null_resource or Packer shell provisioner
provisioner "remote-exec" {
  inline = [
    "cloud-init status --wait; rc=$?; if [ $rc -eq 1 ]; then echo 'cloud-init failed critically'; exit 1; fi"
  ]
}
```

## Debugging Failures

### Quick Triage Workflow

```bash
# 1. Get the overall status
cloud-init status --long

# 2. If errors, get the JSON for details
cloud-init status --format json | jq '.'

# 3. Check the result file (quick summary)
cat /run/cloud-init/result.json

# 4. Search for errors in logs
grep -E "(ERROR|CRITICAL|Traceback)" /var/log/cloud-init.log | tail -20

# 5. Check script output
tail -50 /var/log/cloud-init-output.log

# 6. Check which stage is currently running (if stuck)
cloud-init status --format json | jq '.stage'
```

### Common Critical Failure Scenarios

| Scenario | Symptom | How to Confirm |
|----------|---------|---------------|
| IMDS unreachable | `status: error`, datasource timeout in logs | `curl -s http://169.254.169.254/` fails |
| Broken user-data YAML | `status: error`, parse error in `init` stage | `cloud-init schema --system --annotate` |
| Missing Python deps | cloud-init service fails to start | `systemctl status cloud-init-network.service` |
| Disk full | Write errors across multiple stages | `df -h /var/lib/cloud` |
| SELinux denial | Permission errors in specific modules | `ausearch -m avc -ts recent` |
| Network driver missing | `init-local` completes but `init` hangs | `ip link show`, check for missing interfaces |

### Stuck cloud-init (Status: "running" Indefinitely)

```bash
# Check what stage it's stuck on
cloud-init status --format json | jq '.stage'

# Check for blocking processes
systemctl list-jobs

# Find what cloud-init is waiting on
pstree $(pgrep -f "cloud-init") 2>/dev/null

# Check if it's waiting for network
journalctl -u cloud-init-network.service --no-pager | tail -20
```

## Summary

The `cloud-init status` command is your primary tool for determining whether an instance was provisioned correctly. The key takeaways:

- **Exit code 0** = everything worked
- **Exit code 1** = critical failure, cloud-init may not have finished, subsequent commands likely did not run
- **Exit code 2** = completed with issues, investigate but the instance is likely usable
- **Always use `--format json`** in automation for parseable, per-stage error details
- **Critical errors in early stages** (`init-local`, `init`) are the most dangerous because they prevent later stages from running — meaning no users, no SSH, no packages, no `runcmd`
- **Recoverable errors** mean cloud-init finished but some module reported issues — the instance is usually functional but may be missing specific configurations
