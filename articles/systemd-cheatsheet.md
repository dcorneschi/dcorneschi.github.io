# systemd Cheatsheet

Managing services, units, targets, and timers with `systemctl` — the control interface for the systemd init system.

## Runlevels / Targets

Targets are groups of units that define system states (like the old SysV runlevels).

| Sysvinit Runlevel | Systemd Target | Notes |
|:-:|---|---|
| 0 | `runlevel0.target`, `poweroff.target` | Halt the system |
| 1, s, single | `runlevel1.target`, `rescue.target` | Single user mode |
| 2, 4 | `runlevel2.target`, `runlevel4.target`, `multi-user.target` | User-defined/site-specific. By default, identical to 3 |
| 3 | `runlevel3.target`, `multi-user.target` | Multi-user, non-graphical. Login via console or network |
| 5 | `runlevel5.target`, `graphical.target` | Multi-user, graphical. All services of runlevel 3 plus graphical login |
| 6 | `runlevel6.target`, `reboot.target` | Reboot |
| emergency | `emergency.target` | Emergency shell — minimal, no services |

### Managing Targets

| Command | What It Does |
|---------|--------------|
| `systemctl get-default` | Show the default boot target |
| `systemctl set-default multi-user.target` | Boot to CLI (no GUI) |
| `systemctl set-default graphical.target` | Boot to GUI |
| `systemctl isolate multi-user.target` | Switch to a target now (stops unneeded units) |
| `systemctl isolate rescue.target` | Enter single-user/rescue mode |
| `systemctl isolate emergency.target` | Minimal emergency shell |

After `set-default`, the change applies at next boot. To switch immediately without rebooting, follow with `systemctl isolate <target>` or `systemctl start <target>`.

### SysVinit vs systemd

| SysVinit Command | systemd Equivalent |
|------------------|--------------------|
| `service httpd start` | `systemctl start httpd` |
| `service httpd stop` | `systemctl stop httpd` |
| `service httpd restart` | `systemctl restart httpd` |
| `service httpd reload` | `systemctl reload httpd` |
| `service httpd condrestart` | `systemctl try-restart httpd` |
| `service httpd status` | `systemctl status httpd` |
| `chkconfig httpd on` | `systemctl enable httpd` |
| `chkconfig httpd off` | `systemctl disable httpd` |
| `chkconfig httpd` | `systemctl is-enabled httpd` |
| `chkconfig --list` | `systemctl list-unit-files --type=service` |
| `chkconfig --list \| grep 5:on` | `systemctl list-dependencies graphical.target` |
| `ls /etc/rc.d/init.d/` | `systemctl list-unit-files --type=service` |
| `telinit 3` | `systemctl isolate multi-user.target` |
| `telinit 5` | `systemctl isolate graphical.target` |

Note: `service` and `chkconfig` still work on systemd systems — they are translated to native `systemctl` equivalents. The `.service` suffix is optional: `systemctl start httpd` is the same as `systemctl start httpd.service`.

## Service Lifecycle

| Command | What It Does |
|---------|--------------|
| `systemctl start nginx.service` | Start a service |
| `systemctl stop nginx.service` | Stop a service |
| `systemctl restart nginx.service` | Stop then start |
| `systemctl reload nginx.service` | Reload service config without restarting (e.g., nginx re-reads `nginx.conf`) |
| `systemctl try-restart nginx.service` | Restart only if already running |
| `systemctl reload-or-restart nginx.service` | Reload if supported, otherwise restart |
| `systemctl kill nginx.service` | Send a signal to service processes |
| `systemctl kill -s SIGHUP nginx.service` | Send a specific signal |

Note: `reload` reloads the *service's own* configuration (e.g., httpd.conf). It does not reload the unit file — that's `daemon-reload`.

## Enable and Disable (Boot Behavior)

| Command | What It Does |
|---------|--------------|
| `systemctl enable nginx.service` | Start at boot (creates symlinks from [Install] section) |
| `systemctl disable nginx.service` | Don't start at boot (removes symlinks) |
| `systemctl enable --now nginx.service` | Enable and start immediately |
| `systemctl disable --now nginx.service` | Disable and stop immediately |
| `systemctl reenable nginx.service` | Disable then enable (resets symlinks to defaults) |
| `systemctl mask nginx.service` | Completely prevent starting (links unit to `/dev/null`) |
| `systemctl unmask nginx.service` | Undo a mask |
| `systemctl preset nginx.service` | Reset to vendor-defined enable/disable policy |
| `systemctl preset-all` | Reset all units to vendor presets |

### Enable vs Start

Enabling and starting are orthogonal:
- **enable** = hooks the unit into the boot dependency tree
- **start** = activates it right now

A service can be enabled without being started, and started without being enabled.

## Inspecting Services

| Command | What It Does |
|---------|--------------|
| `systemctl status nginx.service` | Human-readable status, recent logs, PID, memory |
| `systemctl status nginx.service -l` | Full status output (don't truncate long lines) |
| `systemctl status nginx.service -n 20` | Status with last 20 journal lines (default is 10) |
| `systemctl status --all` | Status for all units |
| `systemctl status httpd -H root@webserver01` | Check service status on a remote host via SSH |
| `systemctl status <PID>` | Find which unit owns a process |
| `systemctl is-active nginx.service` | Exit 0 if running, non-zero if not |
| `systemctl is-enabled nginx.service` | Show enablement state (enabled, disabled, masked, static...) |
| `systemctl is-failed nginx.service` | Exit 0 if in failed state |
| `systemctl show nginx.service` | All properties (machine-readable) |
| `systemctl show -p MainPID nginx.service` | Show a specific property |
| `systemctl show -p MainPID --value nginx.service` | Property value only (no key=) |
| `systemctl cat nginx.service` | Print the unit file and all drop-ins |
| `systemctl list-dependencies nginx.service` | Show dependency tree |
| `systemctl list-dependencies --reverse nginx.service` | What depends on this unit |

## Listing Units

| Command | What It Does |
|---------|--------------|
| `systemctl` | All loaded units showing status (shorthand for `list-units`) |
| `systemctl list-units` | All loaded units (active, failed, pending) |
| `systemctl list-units --all` | Include inactive units |
| `systemctl list-units --type=service` | Only services |
| `systemctl --type=service` | Same (shorthand) |
| `systemctl list-units --type=service --state=active` | Loaded and active services (running + exited) |
| `systemctl list-units --type=service --state=running` | Only running services |
| `systemctl list-units --type service --all` | All services including inactive |
| `systemctl list-units --type=timer` | Only timers |
| `systemctl list-units --state=failed` | Only failed units |
| `systemctl --failed` | Shortcut for failed units |
| `systemctl list-unit-files` | All installed unit files with enablement state |
| `systemctl list-unit-files --type=service` | Only service unit files (installed, running or not) |
| `systemctl list-unit-files 'nginx*'` | Pattern matching |
| `systemctl list-timers` | Active timers with next/last run times |
| `systemctl list-timers --all` | Include inactive timers |
| `systemctl list-sockets` | Active sockets with listening addresses |

### Useful Alias

```bash
# Add to .bashrc for quick access
alias running_services='systemctl list-units --type=service --state=running'
```

## Daemon Management

| Command | What It Does |
|---------|--------------|
| `systemctl daemon-reload` | Reload all unit files (after editing `.service` files) |
| `systemctl daemon-reexec` | Re-execute the systemd manager (rare, for upgrades) |
| `systemctl reset-failed` | Clear failed state for all units |
| `systemctl reset-failed nginx.service` | Clear failed state for one unit |

**When to `daemon-reload`:** Every time you create, modify, or delete a unit file under `/etc/systemd/system/`. Without it, systemd keeps using the old cached version.

## Editing Unit Files

| Command | What It Does |
|---------|--------------|
| `systemctl edit nginx.service` | Create/edit a drop-in override (`override.conf`) |
| `systemctl edit --full nginx.service` | Edit the full unit file (replaces the original) |
| `systemctl edit --force nginx.service` | Create a new unit file from scratch |
| `systemctl edit --runtime nginx.service` | Temporary override (lost on reboot) |
| `systemctl revert nginx.service` | Remove all overrides, revert to vendor unit |

Drop-in files go to `/etc/systemd/system/nginx.service.d/override.conf`. They are merged on top of the base unit file.

## Power Management

| Command | What It Does |
|---------|--------------|
| `systemctl poweroff` | Shutdown and power off |
| `systemctl reboot` | Reboot |
| `systemctl suspend` | Suspend to RAM |
| `systemctl hibernate` | Suspend to disk |
| `systemctl hybrid-sleep` | Suspend to both RAM and disk |
| `systemctl suspend-then-hibernate` | Suspend, then hibernate after timeout |

## Unit File Structure

Unit files live in:
- `/usr/lib/systemd/system/` — Vendor-supplied (packages)
- `/etc/systemd/system/` — Admin overrides (highest priority)
- `/run/systemd/system/` — Runtime (generated, transient)

### Service Unit Example

```ini
[Unit]
Description=My Application
Documentation=https://example.com/docs
After=network.target
Wants=network.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
Environment=NODE_ENV=production
EnvironmentFile=-/etc/myapp/env

[Install]
WantedBy=multi-user.target
```

### Service Types

| Type | Behavior |
|------|----------|
| `simple` (default) | Process started by `ExecStart` is the main process |
| `exec` | Like simple, but service is "started" only after exec() succeeds |
| `forking` | Process forks; parent exits, child becomes main (traditional daemons) |
| `oneshot` | Process runs and exits; service is "active" only while running |
| `notify` | Like simple, but service sends `sd_notify()` when ready |
| `dbus` | Like simple, but service acquires a D-Bus name when ready |
| `idle` | Like simple, but delayed until all active jobs finish |

### Restart Policies

| Policy | Restarts on |
|--------|-------------|
| `no` | Never |
| `on-success` | Clean exit (code 0) only |
| `on-failure` | Non-zero exit, signal, timeout, watchdog |
| `on-abnormal` | Signal, timeout, watchdog |
| `on-abort` | Signal only |
| `on-watchdog` | Watchdog timeout only |
| `always` | Always, regardless of exit reason |

### Key Service Directives

| Directive | Purpose |
|-----------|---------|
| `ExecStart=` | Command to start the service |
| `ExecStartPre=` | Commands to run before start |
| `ExecStartPost=` | Commands to run after start |
| `ExecStop=` | Command to stop (default: SIGTERM) |
| `ExecReload=` | Command to reload config |
| `Restart=` | When to automatically restart |
| `RestartSec=` | Delay before restart |
| `TimeoutStartSec=` | Max time to start before failure |
| `TimeoutStopSec=` | Max time to stop before SIGKILL |
| `WatchdogSec=` | Watchdog timeout (service must ping systemd) |
| `User=` / `Group=` | Run as this user/group |
| `WorkingDirectory=` | Set the working directory |
| `Environment=` | Set environment variables |
| `EnvironmentFile=` | Read env vars from file (`-` prefix = ignore if missing) |
| `StandardOutput=` | Where stdout goes (`journal`, `file:/path`, `null`) |
| `StandardError=` | Where stderr goes |
| `LimitNOFILE=` | File descriptor limit |
| `MemoryMax=` | Memory cgroup limit |
| `CPUQuota=` | CPU cgroup limit |

### Key Unit Directives (common to all unit types)

| Directive | Purpose |
|-----------|---------|
| `Description=` | Human-readable name |
| `Documentation=` | URL(s) to docs |
| `After=` | Start after these units (ordering only, not dependency) |
| `Before=` | Start before these units |
| `Requires=` | Hard dependency — if it fails, this fails too |
| `Wants=` | Soft dependency — if it fails, this continues |
| `BindsTo=` | Like Requires, but also stops when dependency stops |
| `Conflicts=` | Cannot run at the same time as these units |
| `ConditionPathExists=` | Only start if path exists |
| `AssertPathExists=` | Fail if path doesn't exist |

### Install Section

| Directive | Purpose |
|-----------|---------|
| `WantedBy=` | Target(s) that pull this unit in (creates `.wants/` symlinks) |
| `RequiredBy=` | Target(s) that require this unit |
| `Alias=` | Alternative names for the unit |
| `Also=` | Additional units to enable/disable together |

## Timer Units

Timers are systemd's replacement for cron. A `.timer` unit triggers a matching `.service` unit.

### Timer Unit Example

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup timer

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=900

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Daily backup job

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

### Timer Types

| Directive | Trigger |
|-----------|---------|
| `OnCalendar=` | Absolute/calendar time (like cron) |
| `OnBootSec=` | Time after boot |
| `OnStartupSec=` | Time after systemd started |
| `OnActiveSec=` | Time after the timer itself was activated |
| `OnUnitActiveSec=` | Time after the triggered unit was last activated |
| `OnUnitInactiveSec=` | Time after the triggered unit became inactive |

### OnCalendar Syntax

| Expression | Meaning |
|------------|---------|
| `*-*-* 02:00:00` | Daily at 2am |
| `Mon *-*-* 09:00:00` | Every Monday at 9am |
| `*-*-01 00:00:00` | First of every month |
| `*-01,07-01 00:00:00` | Jan 1 and Jul 1 |
| `hourly` | Every hour (shorthand) |
| `daily` | Every day at midnight |
| `weekly` | Every Monday at midnight |
| `monthly` | First of month at midnight |
| `*:0/15` | Every 15 minutes |
| `*-*-* *:00,30:00` | Every 30 minutes |

Test calendar expressions:

```bash
systemd-analyze calendar "*-*-* 02:00:00"
systemd-analyze calendar "Mon *-*-* 09:00:00" --iterations=5
```

### Timer Options

| Directive | Purpose |
|-----------|---------|
| `Persistent=true` | Run immediately if a scheduled run was missed (e.g., system was off) |
| `RandomizedDelaySec=` | Add random jitter to prevent thundering herd |
| `AccuracySec=` | How precisely to fire (default 1min — set to `1s` for precision) |
| `Unit=` | Override which unit to trigger (default: same name `.service`) |

### Managing Timers

```bash
# Enable and start a timer
systemctl enable --now backup.timer

# List all timers with next/last fire times
systemctl list-timers --all

# Check when a timer will fire next
systemd-analyze calendar "daily" --iterations=3

# Manually trigger the associated service
systemctl start backup.service

# See timer logs
journalctl -u backup.timer
journalctl -u backup.service
```

## Runtime Properties

```bash
# Set a property at runtime (resets on restart)
systemctl set-property nginx.service CPUQuota=50%
systemctl set-property nginx.service MemoryMax=1G

# Make runtime property changes permanent
systemctl set-property nginx.service CPUQuota=50%  # writes to /etc/systemd/system/nginx.service.d/

# Show cgroup resource usage
systemd-cgtop
```

## Analyzing Boot

```bash
# Boot time breakdown
systemd-analyze

# Per-unit startup time
systemd-analyze blame

# Critical path (what's slowing boot)
systemd-analyze critical-chain

# Generate SVG boot chart
systemd-analyze plot > boot.svg

# Verify unit file syntax
systemd-analyze verify /etc/systemd/system/myapp.service

# Show security score for a service
systemd-analyze security nginx.service
```

## Recipes

### Create a new service from scratch

```bash
# Create the unit file
cat > /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=myapp
ExecStart=/opt/myapp/bin/server
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Reload, enable, and start
systemctl daemon-reload
systemctl enable --now myapp.service

# Verify it's running
systemctl status myapp.service
```

### Override a vendor service without editing the original

```bash
# Create a drop-in override
systemctl edit nginx.service
```

This opens an editor for `/etc/systemd/system/nginx.service.d/override.conf`. Example content:

```ini
[Service]
LimitNOFILE=65535
Environment=WORKER_CONNECTIONS=4096
```

After saving, systemd auto-reloads. Verify with `systemctl cat nginx.service`.

### Fix a service that hit its start rate limit

```bash
# Check the error
systemctl status myapp.service
# "Start request repeated too quickly"

# Reset the failure counter
systemctl reset-failed myapp.service

# Now start works again
systemctl start myapp.service
```

### Find why a service failed

```bash
# Status shows exit code and recent logs
systemctl status myapp.service

# Full logs for the service
journalctl -u myapp.service -b --no-pager

# Only errors
journalctl -u myapp.service -p err -b
```

### Run a one-shot command as a transient unit

```bash
# Run a command with systemd resource controls
systemd-run --unit=my-task --scope -p MemoryMax=512M /usr/local/bin/heavy-job

# Run a timer that fires once in 5 minutes
systemd-run --on-active=5m /usr/local/bin/cleanup.sh

# Run a timer on a calendar schedule
systemd-run --on-calendar="*:0/30" /usr/local/bin/check.sh
```

### Mask a unit to prevent any activation

```bash
# Completely prevent a service from starting (even manually)
systemctl mask bluetooth.service

# Verify
systemctl start bluetooth.service
# "Failed to start bluetooth.service: Unit bluetooth.service is masked."

# Undo
systemctl unmask bluetooth.service
```

### List all services ordered by memory usage

```bash
systemd-cgtop -m
```

### Check which unit owns a process

```bash
systemctl status <PID>
# or
systemctl whoami <PID>
```

### Replace cron with a systemd timer

```bash
# Old cron: */5 * * * * /usr/local/bin/check.sh

# Create the service
cat > /etc/systemd/system/check.service << 'EOF'
[Unit]
Description=Health check

[Service]
Type=oneshot
ExecStart=/usr/local/bin/check.sh
EOF

# Create the timer
cat > /etc/systemd/system/check.timer << 'EOF'
[Unit]
Description=Run health check every 5 minutes

[Timer]
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target
EOF

# Enable and start
systemctl daemon-reload
systemctl enable --now check.timer

# Verify
systemctl list-timers check.timer
```

### Show all overrides for a service

```bash
systemctl cat nginx.service
# Shows the base file + all drop-in files in order
```

### Temporarily change a property for debugging

```bash
# Set runtime-only (lost on reboot or service restart)
systemctl set-property --runtime nginx.service Environment=DEBUG=1

# Or edit with --runtime flag
systemctl edit --runtime nginx.service
```

### Create a drop-in override manually (without systemctl edit)

```bash
mkdir -p /etc/systemd/system/httpd.service.d
cat > /etc/systemd/system/httpd.service.d/50-custom.conf << 'EOF'
[Service]
LimitNOFILE=65535
TimeoutStartSec=90
EOF

systemctl daemon-reload
systemctl restart httpd.service
systemctl status httpd.service
```

### Debug systemd-journald

```bash
mkdir -p /etc/systemd/system/systemd-journald.service.d

cat > /etc/systemd/system/systemd-journald.service.d/10-debug.conf << 'EOF'
[Service]
Environment=SYSTEMD_LOG_LEVEL=debug
EOF

systemctl daemon-reload
systemctl restart systemd-journald
journalctl -b -u systemd-journald
```

### Monitor system resource usage by cgroup

```bash
systemd-cgtop
systemd-cgtop -m   # sort by memory
```

## Unit File Locations (Priority Order)

| Path | Purpose | Priority |
|------|---------|----------|
| `/etc/systemd/system/` | Admin customization | Highest |
| `/run/systemd/system/` | Runtime (generated, transient) | Medium |
| `/usr/lib/systemd/system/` | Vendor/package defaults | Lowest |

A file in `/etc/` overrides the same-named file in `/usr/lib/`. Drop-ins (`*.d/`) are merged in alphabetical order on top of the base file.

## Useful Flags

| Flag | Works With | Purpose |
|------|-----------|---------|
| `--now` | enable, disable, mask | Also start/stop the unit immediately |
| `--no-pager` | status, list-* | Print to stdout without paging |
| `--full` | list-units, status | Don't truncate long lines |
| `--all` | list-units, list-timers | Include inactive |
| `--type=` | list-units, list-unit-files | Filter by type (service, timer, socket, mount...) |
| `--state=` | list-units, list-unit-files | Filter by state (running, failed, enabled, disabled...) |
| `--property=` | show | Show specific property |
| `--value` | show | Print only the value (skip key=) |
| `--quiet` | is-active, is-enabled | Suppress output, just set exit code |
| `--runtime` | enable, edit, set-property | Changes are temporary (lost on reboot) |
| `--force` | enable, edit | Create new unit if it doesn't exist |
| `--plain` | list-dependencies | Flat list instead of tree |
| `--no-block` | start, stop, restart | Don't wait for the operation to complete |
| `--failed` | (standalone) | Shortcut for `--state=failed` |
| `-l` | status | Don't truncate long lines (same as `--full`) |
| `-n <N>` | status | Show last N journal lines (default 10) |
| `-H user@host` | status, start, stop, etc. | Execute on a remote host via SSH |

## Gotchas

- **`daemon-reload` after editing unit files.** If you create or modify a `.service` file and don't run `daemon-reload`, systemd uses the cached version. Symptoms: changes don't take effect, or you get "changed on disk" warnings.
- **`enable` does not `start`.** They're separate operations. Use `--now` to do both.
- **`mask` is stronger than `disable`.** A disabled service can still be started manually or pulled in by dependencies. A masked service cannot be started at all.
- **`reload` is not `daemon-reload`.** `reload` tells the service to re-read its *own* config (e.g., nginx re-reads nginx.conf). `daemon-reload` tells systemd to re-read *unit files*.
- **Type=forking needs PIDFile or GuessMainPID.** If your service forks (traditional daemons), set `Type=forking` and ideally `PIDFile=` so systemd can track the main process.
- **ExecStart must be an absolute path.** Relative paths, shell built-ins, and pipes don't work directly. Use `/bin/bash -c '...'` if you need shell features.
- **Environment variables in ExecStart need `$` escaping.** Use `${VAR}` or `$$` for a literal dollar sign. For complex commands, put the logic in a script.
- **Services may be killed by cgroup on stop.** After `ExecStop` runs, remaining processes in the cgroup are killed. Control this with `KillMode=` (default: `control-group`).
- **`Restart=always` doesn't survive `systemctl stop`.** Manual stops are not considered failures — the service won't restart. Only unexpected exits trigger restart.
- **Start rate limiting.** If a service fails too fast too many times (`StartLimitBurst`/`StartLimitIntervalSec`), systemd refuses to start it. Use `systemctl reset-failed` to clear.

## Kernel Boot Options

To boot into a specific target from GRUB, add to the kernel command line:

```
systemd.unit=rescue.target
systemd.unit=multi-user.target
systemd.unit=emergency.target
```

The `.target` extension is optional — `systemd.unit=rescue` works the same.

## See Also

- [journalctl Cheatsheet](articles/journalctl-cheatsheet.md) — querying logs for services managed by systemd
- [Cron Cheatsheet](articles/cron-cheatsheet.md) — traditional scheduling (systemd timers are the modern alternative)
- [Linux ulimit Guide](articles/linux-ulimit-guide.md) — resource limits that systemd's `Limit*` directives replace
