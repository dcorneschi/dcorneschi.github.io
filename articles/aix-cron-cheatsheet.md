# AIX Cron and Job Scheduling Cheatsheet

Command reference for scheduling jobs on IBM AIX — recurring jobs with `crontab` (managed by the `cron` daemon), one-off jobs with `at` and `batch`, access control, and the files/logs involved. AIX cron behaves like other Unix cron implementations, with a few AIX-specific paths and access-control files.

> Per-user crontabs live in `/var/spool/cron/crontabs/<user>`. Always edit them with `crontab -e` (which validates and reloads) rather than editing the spool files directly. The `cron` daemon is started from `/etc/inittab` and re-reads changes automatically.

## crontab — Manage a User's Jobs

```sh
crontab -e                 # edit your crontab (uses $EDITOR/$VISUAL; validated on save)
crontab -l                 # list your crontab
crontab -r                 # remove your crontab entirely
crontab -v                 # show the submission time of your crontab (AIX-specific)

# Manage another user's crontab (as root)
crontab -e <user>          # edit user's crontab
crontab -l <user>          # list user's crontab
crontab -r <user>          # remove user's crontab

# Install a crontab from a file (replaces the current one)
crontab mycron.txt
crontab -l > mycron.bak    # back up before changing
```

> `crontab -r` deletes the whole crontab with no confirmation — back up with `crontab -l > file` first.

## Crontab Entry Format

Each line has five time fields plus the command:

```
* * * * *  command to run
| | | | |
| | | | +-- day of week   (0-6, Sunday=0; or 0-7 with 7=Sunday)
| | | +---- month         (1-12)
| | +------ day of month  (1-31)
| +-------- hour          (0-23)
+---------- minute        (0-59)
```

Field syntax: `*` (any), lists `1,15,30`, ranges `1-5`, and steps `*/10` (AIX supports steps).

### Examples

```
0 2 * * *        /usr/local/bin/backup.sh          # daily at 02:00
30 6 * * 1-5     /usr/local/bin/report.sh          # 06:30 Mon-Fri
*/15 * * * *     /usr/local/bin/poll.sh            # every 15 minutes
0 0 1 * *        /usr/local/bin/monthly.sh         # midnight on the 1st
0 22 * * 0       /usr/local/bin/weekly.sh          # Sundays at 22:00
0 3 1,15 * *     /usr/local/bin/twice-monthly.sh   # 03:00 on the 1st and 15th
```

> cron runs jobs with a minimal environment (a limited `PATH`, no login profile). Use **absolute paths**, set variables explicitly, and redirect output — otherwise cron mails stdout/stderr to the job owner.

```
0 1 * * *  /usr/local/bin/job.sh >/var/log/job.log 2>&1
```

## Access Control

Who may use `crontab` is governed by two files under `/var/adm/cron`:

| File | Effect |
|------|--------|
| `/var/adm/cron/cron.allow` | If it exists, **only** listed users may use cron |
| `/var/adm/cron/cron.deny` | If `cron.allow` is absent, listed users are **denied**; everyone else allowed |

- If **neither** file exists, on AIX only `root` may use cron (behavior differs from some Linux defaults).
- One username per line.

```sh
cat /var/adm/cron/cron.allow
echo appuser >> /var/adm/cron/cron.allow   # let a user schedule jobs
```

The equivalent files for `at`/`batch` are `/var/adm/cron/at.allow` and `/var/adm/cron/at.deny`.

## One-Off Jobs: at and batch

`at` runs a command once at a specified time; `batch` runs it when system load permits.

```sh
at 22:00                       # then type commands, end with Ctrl-D
at now + 1 hour                # relative time
at 8am tomorrow
at -f script.sh 23:30          # run a script file
echo "/usr/local/bin/x.sh" | at midnight

at -l                          # list pending at jobs (same as atq)
atq                            # list the at queue
at -r <job>                    # remove a job (same as atrm)
atrm <job>
batch                          # run when load average allows; commands then Ctrl-D
```

## Files and Daemon

| Path | Purpose |
|------|---------|
| `/var/spool/cron/crontabs/<user>` | Per-user crontab files |
| `/var/spool/cron/atjobs/` | Queued `at` jobs |
| `/var/adm/cron/cron.allow` / `cron.deny` | crontab access control |
| `/var/adm/cron/at.allow` / `at.deny` | at/batch access control |
| `/var/adm/cron/log` | cron activity log (job runs, start/stop) |
| `/etc/cronlog.conf` | cron logging configuration — log file location, rotation size, and number of rotated copies |
| `/etc/inittab` | Starts the `cron` daemon at boot (`cron` entry) |

```sh
# The cron daemon
ps -ef | grep cron            # is cron running?
lssrc -s cron 2>/dev/null || ps -ef | grep '[c]ron'

# Restart cron if needed (it re-reads crontabs automatically, but if it's wedged):
kill -9 $(ps -ef | grep '[c]ron' | awk '{print $2}')   # inittab respawns it
# or via inittab:
telinit q                     # re-examine /etc/inittab

# Review what cron did
tail -f /var/adm/cron/log
```

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Job didn't run | `/var/adm/cron/log` for the entry; is `cron` running? |
| "you are not authorized to use cron" | Add the user to `cron.allow` (or remove from `cron.deny`) |
| Runs manually but not in cron | Minimal environment — use absolute paths, set `PATH`, source needed vars |
| No output / silent failure | Redirect `>log 2>&1`; unredirected output is mailed to the owner |
| Wrong time | Check the system timezone (`echo $TZ`, `/etc/environment`) and clock |
| Edits not taking effect | Use `crontab -e` (not direct file edits); confirm you edited the right user |

```sh
# Test a job with cron's minimal environment
env -i /bin/sh -c '/usr/local/bin/job.sh'
```

## Quick Reference

| Task | Command |
|------|---------|
| Edit your crontab | `crontab -e` |
| List your crontab | `crontab -l` |
| Remove your crontab | `crontab -r` |
| Edit another user's | `crontab -e <user>` |
| Back up a crontab | `crontab -l > file` |
| Install from a file | `crontab <file>` |
| Allow a user | `echo <user> >> /var/adm/cron/cron.allow` |
| One-off job | `at <time>` / `at -f <script> <time>` |
| List at jobs | `atq` (or `at -l`) |
| Remove at job | `atrm <job>` |
| cron log | `tail -f /var/adm/cron/log` |

## Related

- [AIX Boot and Init Cheatsheet](articles/aix-boot-init-cheatsheet.md) — `/etc/inittab` and `telinit`, which start/reload the cron daemon
- [AIX Users and Groups Cheatsheet](articles/aix-users-groups-cheatsheet.md) — user accounts that own crontabs and cron.allow entries
- [AIX Filesystems Cheatsheet](articles/aix-filesystems-cheatsheet.md) — where spool and log files live
