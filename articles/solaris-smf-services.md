# Solaris SMF (Service Management Facility) and Cron

The Service Management Facility (SMF) is how Oracle Solaris 10 and 11 manage services — the equivalent of systemd/init on Linux. This guide covers the SMF commands, service identifiers (FMRIs), instance states, the configuration repository, and log locations, plus Solaris cron access control and files.

## SMF Commands

```bash
# List service instances and their states
svcs

# List a specific service (verbose) and why it's in its state
svcs -l svc:/network/ssh:default
svcs -xv                       # explain services that are failing/offline

# List services that depend on / are depended on by a service
svcs -d svc:/network/ssh:default    # dependencies
svcs -D svc:/network/ssh:default    # dependents
```

Enable, disable, and restart services with `svcadm`; view and change properties with `svccfg`:

```bash
# Enable/disable (persist across reboot); -t = temporary (this boot only)
svcadm enable  svc:/network/ssh:default
svcadm disable svc:/network/ssh:default
svcadm enable -t svc:/network/ssh:default

# Restart / refresh (reread config)
svcadm restart svc:/network/ssh:default
svcadm refresh svc:/network/ssh:default

# Clear a service out of the maintenance state after fixing it
svcadm clear   svc:/network/ssh:default

# Inspect/change service configuration properties
svccfg -s svc:/network/ssh:default listprop
```

Sample `svcs` output (STATE, when it last changed, and the FMRI):

```
STATE          STIME    FMRI
legacy_run     15:22:01 lrc:/etc/rc2_d/S47pppd
online         15:21:54 svc:/system/filesystem/local:default
online         15:22:03 svc:/network/ssh:default
offline        15:22:00 svc:/application/pkg/server:default
maintenance    15:23:11 svc:/network/http:apache24
disabled       15:21:40 svc:/network/telnet:default
```

Sample `svcs -xv` (explains only the services that are broken and points to the log):

```
svc:/network/http:apache24 (Apache 2.4 HTTP server)
 State: maintenance since Mon Jun 03 15:23:11 2024
Reason: Start method exited with $SMF_EXIT_ERR_FATAL.
   See: http://support.oracle.com/msg/SMF-8000-KS
   See: /var/svc/log/network-http:apache24.log
Impact: This service is not running.
```

### Changing Service Properties (svccfg)

```bash
# Set a config property, then refresh so the service re-reads it
svccfg -s svc:/network/ssh:default setprop config/listen_port = 2222
svcadm refresh svc:/network/ssh:default
svcadm restart svc:/network/ssh:default

# Show a single property value
svcprop -p config/listen_port svc:/network/ssh:default
```

`setprop` writes to the repository; `svcadm refresh` moves it into the running service's snapshot; a `restart` applies it if the daemon reads it only at startup.

## The SMF Repository

SMF stores all service configuration in a disk-based database, not scattered config files:

```bash
# Location of the disk-based configuration database
ls -l /etc/svc/repository.db

# SMF service log files (one per service)
ls -l /var/svc/log

# Repair or restore a corrupt repository
/lib/svc/bin/restore_repository
```

- `/etc/svc/repository.db` — the SMF repository (an SQLite database) holding every service's configuration.
- `/var/svc/log` — per-service logs; the first place to look when a service won't start (`svcs -xv` points you here).
- `/lib/svc/bin/restore_repository` — interactive tool to roll the repository back to a known-good backup if it becomes corrupt (a boot-blocking situation).

## Service Identifiers (FMRI Categories)

Services are named by a **Fault Management Resource Identifier (FMRI)**, e.g. `svc:/network/ssh:default`. The path segment groups services by category:

| Identifier | Scope |
|------------|-------|
| `milestone` | Synthetic services used for clean dependency grouping (like run levels) |
| `device` | General device services |
| `system` | Host-centric, non-networked capabilities |
| `system/security` | Low-level host-centric security facilities |
| `network` | Host-centric network infrastructure capabilities |
| `application` | General software services |
| `application/management` | Management facilities |
| `application/security` | High-level security facilities |
| `site` | Site-specific software |
| `platform` | Platform-specific software |

An FMRI has the form `svc:/<category>/<service>:<instance>` — `:default` is the common instance name.

## Service Instance States

Every service instance is in one of these states (shown by `svcs`):

| State | Meaning |
|-------|---------|
| `online` | Enabled and running normally |
| `offline` | Enabled but not running (e.g. waiting on an unmet dependency) |
| `disabled` | Turned off; won't start |
| `legacy_run` | A legacy `/etc/rc*.d` script service (not fully SMF-managed) |
| `uninitialized` | Started but hasn't yet read its configuration |
| `maintenance` | Failed / needs admin intervention (`svcadm clear` after fixing) |
| `degraded` | Running but at reduced capacity/functionality |

Typical troubleshooting: a service in `maintenance` → run `svcs -xv` to see why and find its log in `/var/svc/log`, fix the issue, then `svcadm clear`.

## Milestones (the SMF Equivalent of Run Levels)

SMF **milestones** are synthetic services that group dependencies, roughly mapping to the old SVR4 run levels:

| Milestone | Rough run-level equivalent |
|-----------|----------------------------|
| `milestone/single-user` | S / single-user |
| `milestone/multi-user` | 2 / multi-user, no NFS |
| `milestone/multi-user-server` | 3 / multi-user with services |
| `milestone/all` | all services (the default) |

```bash
# Boot to (or drop to) a milestone temporarily
svcadm milestone milestone/single-user
svcadm milestone all                       # back to full service

# Boot to single-user for maintenance (persistent until changed)
svcadm milestone -d milestone/single-user
```

`svcadm milestone -d` changes the *default* boot milestone; without `-d` the change lasts until the next reboot.

## Cron Access Control and Files

Cron on Solaris is managed as an SMF service (`svc:/system/cron:default`), with these access-control and data files:

| File | Purpose |
|------|---------|
| `/etc/cron.d/cron.allow` | Usernames allowed to submit cron jobs |
| `/etc/cron.d/cron.deny` | Usernames denied from submitting cron jobs |
| `/var/cron/log` | Cron activity log |
| `/var/spool/cron/crontabs` | Per-user crontab files |

Access-control precedence:

- If `cron.allow` exists, **only** users listed in it can use `crontab`.
- If `cron.allow` doesn't exist, everyone **except** users in `cron.deny` may use `crontab`.
- If neither file exists, typically only the superuser can use cron.

```bash
# Edit / list / remove the current user's crontab
crontab -e
crontab -l
crontab -r

# Confirm the cron service is online
svcs svc:/system/cron:default
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Service in `maintenance` | Start method failed | `svcs -xv`; read `/var/svc/log/<fmri>.log`; fix; `svcadm clear <fmri>` |
| Service `offline` and won't start | Unmet dependency | `svcs -d <fmri>` to find the blocker; fix that first |
| Config change ignored | Not refreshed/restarted | `svcadm refresh` then `svcadm restart <fmri>` |
| Service re-enters maintenance immediately | Repeated rapid failures (retry limit) | Fix root cause, `svcadm clear`; check the log for the real error |
| System won't boot past a milestone | A required service stuck | Boot `-m milestone=none` or single-user, fix, then `svcadm milestone all` |
| Repository corrupt (boot fails) | Damaged `repository.db` | `/lib/svc/bin/restore_repository` from maintenance mode |

```bash
# Boot bypassing SMF services for emergency repair (at boot: -m milestone=none),
# then once in, bring services up manually:
svcadm milestone all
```

## Command Reference

| Task | Command |
|------|---------|
| List services/states | `svcs` |
| Explain failing services | `svcs -xv` |
| Service detail | `svcs -l <fmri>` |
| Enable / disable | `svcadm enable` / `disable <fmri>` |
| Restart / refresh | `svcadm restart` / `refresh <fmri>` |
| Clear maintenance | `svcadm clear <fmri>` |
| View properties | `svccfg -s <fmri> listprop` |
| Repository DB | `/etc/svc/repository.db` |
| Service logs | `/var/svc/log` |
| Restore repository | `/lib/svc/bin/restore_repository` |
| Cron allow/deny | `/etc/cron.d/cron.allow`, `cron.deny` |
| Crontab files | `/var/spool/cron/crontabs` |

## References

- [Managing System Services in Oracle Solaris (SMF)](https://docs.oracle.com/cd/E37838_01/html/E60998/index.html) — official Oracle docs
- [svcs(1) man page](https://docs.oracle.com/cd/E23824_01/html/821-1462/svcs-1.html) — official Oracle docs
- [svcadm(1M) man page](https://docs.oracle.com/cd/E23824_01/html/821-1462/svcadm-1m.html) — official Oracle docs
