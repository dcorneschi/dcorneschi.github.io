# Cron Cheatsheet

## What is Cron?

Cron is a time-based job scheduler in Unix-like operating systems. It allows users to schedule commands or scripts to run automatically at specified intervals.

System wide crontab entries are found in `/etc/cron{tab,.d,.daily,.hourly,.monthly,.weekly}` and individual users crontab jobs are found in `/var/spool/cron/` directory.

- RHEL 6.x, a non-root user's home directory must be created for the cron job to run.
- RHEL 5.x uses vixie-cron which does not check for the existence of a users home directory while cronie used.

## Installation

```sh
yum install cronie
```

## Cron Expression Format

A cron expression consists of five fields (six for some implementations that include seconds):

```console
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed
```

## Special Characters

| Character | Meaning | Example |
|-----------|---------|---------|
| `*` | Any value | `* * * * *` (every minute) |
| `,` | Value list separator | `1,15,30 * * * *` (at minute 1, 15, and 30) |
| `-` | Range of values | `1-5 * * * *` (minutes 1 through 5) |
| `/` | Step values | `*/15 * * * *` (every 15 minutes) |
| `@` | Predefined schedules | `@daily` (once a day at midnight) |

## Common Schedule Examples

| Expression | Description |
|-----------|-------------|
| `* * * * *` | Every minute |
| `*/5 * * * *` | Every 5 minutes |
| `*/15 * * * *` | Every 15 minutes |
| `*/30 * * * *` | Every 30 minutes |
| `0 * * * *` | Every hour (at minute 0) |
| `0 */2 * * *` | Every 2 hours |
| `0 */6 * * *` | Every 6 hours |
| `0 */12 * * *` | Every 12 hours |
| `4 0 * * *` | Once a day (at 00:04) |
| `0 0 * * *` | Every day at midnight |
| `0 6 * * *` | Every day at 6:00 AM |
| `30 8 * * *` | Every day at 8:30 AM |
| `0 10,22 * * *` | Twice a day (at 10:00 AM and 10:00 PM) |
| `4 0 * * 0` | Once a week (Sunday at 00:04) |
| `0 0 * * 0` | Every Sunday at midnight |
| `0 1 * * sun` | Every Sunday at 1:00 AM |
| `0 9 * * 1-5` | Every weekday at 9:00 AM |
| `0 13 * * tue,thu` | Every Tuesday and Thursday at 1:00 PM |
| `0 0 * * 1,3,5` | Every Monday, Wednesday, and Friday at midnight |
| `0 22 * * 1-5` | Every weekday at 10:00 PM |
| `0 8-17 * * 1-5` | Every hour from 8 AM to 5 PM on weekdays |
| `4 0 1 * *` | Once a month (1st day at 00:04) |
| `0 0 1 * *` | First day of every month at midnight |
| `0 11 15 * *` | 15th of every month at 11:00 AM |
| `0 0 1,15 * *` | 1st and 15th of every month at midnight |
| `0 0 * jan,apr,jun *` | First day of January, April, and June at midnight |
| `0 0 1 1 *` | Once a year (January 1st at midnight) |
| `5 4 * * 0` | Every Sunday at 4:05 AM |
| `0 0 */2 * *` | Every alternate day at midnight |
| `0 9 * * 6,0` | Weekends only at 9:00 AM |
| `0 0,6,12,18 * * *` | Every 6 hours starting at midnight |
| `0 0 1 1,4,7,10 *` | Every quarter (Jan, Apr, Jul, Oct) on the 1st at midnight |

## Advanced Time Patterns

```bash
# Every 30 minutes between 9 AM and 5 PM, Monday to Friday
*/30 9-17 * * 1-5 /path/to/script.sh

# At 2:30 PM on the 15th of every month
30 14 15 * * /path/to/script.sh

# Last day of every month at 11:59 PM
59 23 28-31 * * [ "$(date +\%d -d tomorrow)" = "01" ] && /path/to/script.sh

# First Monday of every month at 8 AM
0 8 1-7 * 1 /path/to/script.sh

# Every 10 seconds (using sleep in a minute loop)
* * * * * for i in {0..5}; do /path/to/script.sh && sleep 10; done

# Business days excluding holidays (with external check)
0 9 * * 1-5 /path/to/check-holiday.sh && /path/to/script.sh
```

## Predefined Schedules (Shortcuts)

| Shortcut | Equivalent | Description |
|----------|-----------|-------------|
| `@reboot` | — | Run once at startup |
| `@yearly` / `@annually` | `0 0 1 1 *` | Once a year (Jan 1st) |
| `@monthly` | `0 0 1 * *` | Once a month (1st day) |
| `@weekly` | `0 0 * * 0` | Once a week (Sunday) |
| `@daily` / `@midnight` | `0 0 * * *` | Once a day (midnight) |
| `@hourly` | `0 * * * *` | Once an hour |

## Crontab Commands

```bash
# Edit the current user's crontab
crontab -e

# Edit another user's crontab (requires root)
crontab -e -u username

# List the current user's crontab
crontab -l

# List another user's crontab
crontab -l -u username

# Remove the current user's crontab
crontab -r

# Remove crontab for specific user (as root)
crontab -u username -r

# Remove crontab with confirmation prompt (some systems)
crontab -i -r

# Install a crontab from a file
crontab mycronfile.txt

# Test cron syntax (some systems)
crontab -T filename

# Check cron service status
systemctl status cron     # Debian/Ubuntu
systemctl status crond    # CentOS/RHEL
service cron status       # older systems

# Start/stop/restart cron service
systemctl start cron
systemctl stop cron
systemctl restart cron
```

## Crontab File Format

Each line in a crontab file follows this format:

```bash
# Environment variables (optional)
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=user@example.com
HOME=/

# m   h   dom mon dow   command
  0   5   *   *   *     /usr/local/bin/backup.sh
*/10  *   *   *   *     /usr/local/bin/healthcheck.sh
  30  2   1   *   *     /usr/local/bin/monthly-report.sh
```

### Send Email in Crontab

```sh
MAILTO="dcorneschi"
1 1 * * * /path/to/script.sh
```

### Change Shell in Cron

```sh
SHELL=/bin/sh
1 1 * * * /path/to/script.sh
```

### Environmental Variables in Cron

```sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
1 1 * * * /path/to/script.sh
```

### Set Home for Cron

```sh
HOME=/
1 1 * * * /path/to/script.sh
```

## System Crontab (/etc/crontab)

The system crontab has an extra field for the user:

```bash
# /etc/crontab includes user field
# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
```

## Cron Directories

Drop scripts into these directories to run on a fixed schedule (no cron expression needed):

| Directory | Schedule |
|-----------|----------|
| `/etc/cron.d/` | Custom schedules (crontab format files) |
| `/etc/cron.hourly/` | Every hour |
| `/etc/cron.daily/` | Once a day |
| `/etc/cron.weekly/` | Once a week |
| `/etc/cron.monthly/` | Once a month |

Scripts in `cron.hourly`, `cron.daily`, `cron.weekly`, and `cron.monthly` must be executable and should not have a file extension.

```bash
# Check what's scheduled in cron directories
run-parts --test /etc/cron.daily
run-parts --test /etc/cron.hourly
```

## Output Handling

The output of a successfully executed cron job or script will be sent via email to your local mail account. To stop receiving output for the executed cronjob you need to add one of the following strings at the end of the cron job command: `>/dev/null 2>&1` or `&> /dev/null`. To stop the emails for all cron jobs you can also use `MAILTO=""`.

```bash
# Discard all output (stdout and stderr)
0 * * * * /path/to/script.sh > /dev/null 2>&1

# Log output to a file
0 * * * * /path/to/script.sh >> /var/log/myscript.log 2>&1

# Save output to a file
0 0 * * * /some/path/to/a/file.php > $HOME/cron.log 2>&1

# Separate stdout and stderr
0 2 * * * /path/to/script.sh >> /var/log/script.log 2>> /var/log/script_errors.log

# Discard stdout only, keep stderr (still emailed)
0 * * * * /path/to/script.sh > /dev/null

# Send output to a specific email
MAILTO=admin@example.com
0 * * * * /path/to/script.sh

# Disable email entirely
MAILTO=""
0 * * * * /path/to/script.sh

# Log with timestamp
0 2 * * * echo "$(date): Starting backup" >> /var/log/backup.log && /path/to/backup.sh >> /var/log/backup.log 2>&1

# Rotate logs (keep last 1000 lines)
0 2 * * * /path/to/script.sh >> /var/log/script.log 2>&1 && tail -n 1000 /var/log/script.log > /tmp/script.log && mv /tmp/script.log /var/log/script.log

# Use logger for syslog
0 2 * * * /path/to/script.sh 2>&1 | logger -t myscript

# Advanced logging with date-based filenames
0 2 * * * /path/to/script.sh >> /var/log/script_$(date +\%Y\%m\%d).log 2>&1
```

## Environment Variables

Cron runs with a minimal environment. Key variables you can set in the crontab:

| Variable | Default | Description |
|----------|---------|-------------|
| `SHELL` | `/bin/sh` | Shell used to run commands |
| `PATH` | `/usr/bin:/bin` | Search path for commands |
| `MAILTO` | crontab owner | Email recipient for job output |
| `HOME` | user's home | Working directory for commands |
| `LOGNAME` | crontab owner | Login name |
| `TZ` | system timezone | Timezone for cron schedule |

```bash
# Set a custom PATH at the top of your crontab
PATH=/usr/local/bin:/usr/bin:/bin:/home/user/scripts

# Set timezone
TZ=America/New_York

# Custom environment variables
DATABASE_URL=postgresql://user:pass@localhost/db
API_KEY=your_api_key_here

# Use full paths in commands to avoid PATH issues
0 * * * * /usr/local/bin/python3 /home/user/scripts/task.py

# Full crontab example with environment variables
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
SHELL=/bin/bash
HOME=/home/user
MAILTO=admin@example.com
TZ=UTC

0 2 * * * /home/user/backup.sh
```

## Cron Logging

```bash
# View cron logs (Debian/Ubuntu)
grep CRON /var/log/syslog

# View cron logs (RHEL/CentOS)
cat /var/log/cron

# Follow cron logs in real time
tail -f /var/log/syslog | grep CRON     # Debian/Ubuntu
tail -f /var/log/cron                    # RHEL/CentOS
tail -f /var/log/cron.log               # Debian/Ubuntu alternative
journalctl -f -u cron                   # systemd systems

# Check for specific user's cron jobs in logs
grep "username" /var/log/cron

# Check if cron daemon is running
systemctl status cron        # Debian/Ubuntu
systemctl status crond       # RHEL/CentOS
ps aux | grep cron
```

## Cron Access Control

Control which users can use cron:

| File | Purpose |
|------|---------|
| `/etc/cron.allow` | If exists, only listed users can use cron |
| `/etc/cron.deny` | If exists (and no cron.allow), listed users are denied |

- If both files exist, `cron.allow` takes precedence.
- If neither file exists, behavior varies by distribution (often only root can use cron, or all users can).

## Common Pitfalls

### 1. PATH Issues

Cron uses a minimal PATH (`/usr/bin:/bin`). Always use full paths:

```bash
# WRONG - may not find the binary
0 * * * * my-script.sh

# RIGHT - use absolute paths
0 * * * * /home/user/scripts/my-script.sh
```

### 2. Percent Signs (%)

From `man 5 crontab`:

The "sixth" field (the rest of the line) specifies the command to be run. The entire command portion of the line, up to a newline or % character, will be executed by /bin/sh or by the shell specified in the SHELL variable of the cronfile. Percent-signs (%) in the command, unless escaped with backslash (\), will be changed into newline characters, and all data after the first % will be sent to the command as standard input.

```bash
# WRONG - cron interprets % as a newline
0 0 * * * /usr/bin/date +%Y-%m-%d

# RIGHT - escape percent signs
0 0 * * * /usr/bin/date +\%Y-\%m-\%d

# Execute date inside of a cron tab job
0 * * * * echo hello >> ~/cron-logs/hourly/test`date "+\%d"`.log

# Or put the command in a script and call the script instead
0 0 * * * /home/user/scripts/datestamp.sh
```

### 3. Environment Differences

Your interactive shell has a full environment; cron does not:

```bash
# Source your profile if needed
0 * * * * . /home/user/.profile && /home/user/scripts/task.sh

# Or set variables directly in crontab
PATH=/usr/local/bin:/usr/bin:/bin
0 * * * * my-command
```

### 4. Working Directory

Cron jobs start in the user's home directory by default:

```bash
# Use cd to change to the expected directory
0 * * * * cd /opt/myapp && ./run.sh

# Or use absolute paths throughout
0 * * * * /opt/myapp/run.sh
```

### 5. Scripts Not Executable

```bash
# Make your script executable
chmod +x /home/user/scripts/backup.sh

# Or invoke the interpreter directly
0 * * * * /bin/bash /home/user/scripts/backup.sh
```

### 6. GUI Applications Don't Work

```bash
# Set DISPLAY variable
0 2 * * * DISPLAY=:0 /path/to/gui_app
```

### 7. Time Zone Problems

```bash
# Set TZ variable in crontab
TZ=America/New_York
0 2 * * * /path/to/script.sh
```

## Locking (Prevent Overlapping Jobs)

Use `flock` to prevent a job from running if the previous instance is still active:

```bash
# Skip if already running
*/5 * * * * flock -n /tmp/myjob.lock /home/user/scripts/long-task.sh

# Wait up to 60 seconds for the lock
*/5 * * * * flock -w 60 /tmp/myjob.lock /home/user/scripts/long-task.sh
```

### Lock File Wrapper Script

```bash
#!/bin/bash
# wrapper.sh
LOCKFILE=/tmp/myscript.lock
if [ -e ${LOCKFILE} ] && kill -0 `cat ${LOCKFILE}`; then
    echo "Script already running"
    exit
fi
echo $$ > ${LOCKFILE}
# Remove lockfile when script finishes
trap "rm -f ${LOCKFILE}" EXIT
/path/to/actual_script.sh

# Use in crontab
*/5 * * * * /path/to/wrapper.sh
```

## Testing Cron Jobs

```bash
# Simulate the cron environment to test your command
env -i SHELL=/bin/sh PATH=/usr/bin:/bin HOME=$HOME /bin/sh -c 'your-command-here'

# Run cron job manually as specific user
sudo -u username /path/to/script.sh

# Run the command manually to check for errors
/home/user/scripts/backup.sh

# Temporarily run every minute to verify it works
* * * * * /home/user/scripts/backup.sh >> /tmp/cron-test.log 2>&1
# (Remember to change it back after testing!)

# Create test environment script
#!/bin/bash
# test-cron-env.sh
env > /tmp/cron-env
echo "PATH: $PATH" >> /tmp/cron-env
echo "PWD: $PWD" >> /tmp/cron-env
echo "USER: $USER" >> /tmp/cron-env
```

## Anacron

Anacron is designed for machines that are not running 24/7. It ensures jobs run even if the system was off at the scheduled time.

```bash
# /etc/anacrontab format:
# period  delay  job-id   command
1         5      daily-backup   /home/user/scripts/backup.sh
7         10     weekly-report  /home/user/scripts/report.sh
30        15     monthly-clean  /home/user/scripts/cleanup.sh
```

| Field | Description |
|-------|-------------|
| period | Run every N days |
| delay | Wait N minutes after boot before running |
| job-id | Unique identifier for the job |
| command | Command to execute |

## at

The `at` command schedules a one-time job to run at a specific time (unlike cron which is recurring).

```bash
# Run a command in 1 minute
echo "ping -c 10 www.google.com > ping.out" | at now + 1 minute

# Run a command at 05:24 and send an email with the output
echo "ping -c 10 www.google.com" | at -m 05:24 today

# Schedule a script
echo "sh /root/script.sh > /root/script.out" | at 2:00 AM Mar 25

# List the scheduled jobs
at -l
atq

# Remove scheduled job
atrm 6
at -d 6

# Check the content of scheduled job
at -c 18
```

## Conditional Execution

```bash
# Run only if file exists
0 2 * * * [ -f /tmp/run_backup ] && /path/to/backup.sh

# Run only if file doesn't exist
0 2 * * * [ ! -f /tmp/maintenance_mode ] && /path/to/script.sh

# Run only if another process is not running
*/5 * * * * pgrep -f "my_process" > /dev/null || /path/to/my_process.sh

# Run with lock file to prevent overlapping
*/5 * * * * flock -n /tmp/script.lock /path/to/script.sh

# Chain commands with conditions
0 2 * * * /path/to/backup.sh && /path/to/cleanup.sh || echo "Backup failed" | mail -s "Backup Error" admin@example.com

# Run only on specific servers
0 2 * * * [ "$(hostname)" = "server1" ] && /path/to/script.sh

# Run based on load average
*/5 * * * * [ "$(uptime | awk '{print $10}' | cut -d, -f1)" \< "2.0" ] && /path/to/cpu_intensive_script.sh
```

## Retry Logic

```bash
# Retry script up to 3 times
0 2 * * * for i in {1..3}; do /path/to/script.sh && break || sleep 60; done
```

## Resource Monitoring

```bash
# Run only if disk space is available
0 2 * * * [ $(df /var | tail -1 | awk '{print $5}' | sed 's/%//') -lt 90 ] && /path/to/script.sh

# Run only if memory usage is below threshold
*/10 * * * * [ $(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100.0}') -lt 80 ] && /path/to/memory_intensive_script.sh
```

## Best Practices and Security

```bash
# Always use absolute paths
0 2 * * * /usr/bin/php /var/www/html/script.php

# Set strict permissions on scripts
chmod 700 /path/to/script.sh
chown root:root /path/to/script.sh

# Validate input in scripts
#!/bin/bash
if [[ ! "$1" =~ ^[a-zA-Z0-9_-]+$ ]]; then
    echo "Invalid input"
    exit 1
fi

# Use specific user accounts
# In /etc/cron.d/myapp
0 2 * * * myappuser /path/to/script.sh

# Limit resource usage
0 2 * * * nice -n 19 ionice -c3 /path/to/resource_intensive_script.sh
```

### Error Handling Wrapper

```bash
#!/bin/bash
# cron-wrapper.sh
SCRIPT="$1"
LOGFILE="$2"
MAILTO="admin@example.com"

if [ -z "$SCRIPT" ] || [ -z "$LOGFILE" ]; then
    echo "Usage: $0 <script> <logfile>"
    exit 1
fi

{
    echo "=== $(date) ==="
    echo "Starting: $SCRIPT"
    
    if "$SCRIPT"; then
        echo "SUCCESS: $SCRIPT completed"
    else
        EXITCODE=$?
        echo "ERROR: $SCRIPT failed with exit code $EXITCODE"
        echo "$SCRIPT failed at $(date)" | mail -s "Cron Job Failed" "$MAILTO"
    fi
    echo "=== $(date) ==="
    echo
} >> "$LOGFILE" 2>&1
```

## Performance and Optimization

```bash
# Stagger similar jobs to avoid resource conflicts
0 2 * * * /path/to/backup1.sh
5 2 * * * /path/to/backup2.sh
10 2 * * * /path/to/backup3.sh

# Use random delays for distributed systems
*/5 * * * * sleep $(( RANDOM \% 60 )) && /path/to/script.sh

# Limit concurrent executions
*/5 * * * * [ $(pgrep -c -f "my_script.sh") -lt 3 ] && /path/to/my_script.sh

# Use at command for one-time scheduling
0 2 * * * echo "/path/to/script.sh" | at now + 1 hour

# Monitor and alert on long-running jobs
0 * * * * timeout 30m /path/to/script.sh || echo "Script timeout" | mail -s "Cron Timeout" admin@example.com
```

## Common Patterns and Examples

```bash
# Database backup with rotation
0 2 * * * pg_dump mydb > /backup/mydb_$(date +\%Y\%m\%d).sql && find /backup -name "mydb_*.sql" -mtime +7 -delete

# Log rotation
0 0 * * * gzip /var/log/app.log && mv /var/log/app.log.gz /var/log/archive/app_$(date +\%Y\%m\%d).log.gz && touch /var/log/app.log

# Health check with notification
*/5 * * * * curl -f http://localhost:8080/health > /dev/null 2>&1 || echo "Service down" | mail -s "Health Check Failed" admin@example.com

# Cleanup old files
0 3 * * * find /tmp -type f -mtime +7 -delete
0 3 * * 0 find /var/log -name "*.log" -mtime +30 -delete

# Update system packages (with caution)
0 4 * * 1 apt update && apt upgrade -y >> /var/log/system-update.log 2>&1

# Generate reports
0 6 1 * * /path/to/generate_monthly_report.sh
0 6 * * 1 /path/to/generate_weekly_report.sh

# Sync files
*/30 * * * * rsync -av /local/path/ user@remote:/remote/path/

# Monitor SSL certificates
0 9 * * * openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates | grep "notAfter" | awk -F= '{print $2}' | xargs -I {} date -d "{}" +\%s | awk '{if ($1 < systime() + 604800) print "SSL cert expires soon"}' | [ -s /dev/stdin ] && mail -s "SSL Warning" admin@example.com

# Run only on the first Monday of the month
0 9 1-7 * 1 /path/to/script.sh

# Run every 5 minutes during business hours only
*/5 9-17 * * 1-5 /path/to/healthcheck.sh

# Chain multiple commands
0 3 * * * /usr/local/bin/backup.sh && /usr/local/bin/verify-backup.sh

# Run with a timeout (kill if running longer than 60 seconds)
*/5 * * * * timeout 60 /path/to/script.sh
```

## Script Examples with Redirections and Emails

### Backup Script with Logging

```bash
#!/bin/bash
# /usr/local/bin/backup.sh
# Called from crontab: 0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

BACKUP_DIR="/backup/$(date +%Y%m%d)"
SOURCE="/var/www/html"
LOGFILE="/var/log/backup.log"

echo "[$(date '+%Y-%m-%d %H:%M:%S')] Starting backup..."

mkdir -p "$BACKUP_DIR"

if tar czf "$BACKUP_DIR/www-backup.tar.gz" "$SOURCE" 2>> "$LOGFILE"; then
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] Backup completed successfully"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] Size: $(du -h "$BACKUP_DIR/www-backup.tar.gz" | cut -f1)"
else
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR: Backup failed!" >&2
    exit 1
fi
```

Crontab entry:

```bash
# Log both stdout and stderr to file
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# Separate success and error logs
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>> /var/log/backup-errors.log

# Discard stdout, only capture errors
0 2 * * * /usr/local/bin/backup.sh > /dev/null 2>> /var/log/backup-errors.log
```

### Script with Email Notification on Failure

```bash
#!/bin/bash
# /usr/local/bin/db-backup.sh
# Called from crontab: 0 3 * * * /usr/local/bin/db-backup.sh

MAILTO="admin@example.com"
HOSTNAME=$(hostname)
BACKUP_FILE="/backup/db_$(date +%Y%m%d_%H%M%S).sql.gz"

pg_dump production_db | gzip > "$BACKUP_FILE"

if [ ${PIPESTATUS[0]} -ne 0 ]; then
    echo "Database backup failed on $HOSTNAME at $(date)" | \
        mail -s "[$HOSTNAME] DB Backup FAILED" "$MAILTO"
    exit 1
fi

# Verify backup is not empty
if [ ! -s "$BACKUP_FILE" ]; then
    echo "Backup file is empty on $HOSTNAME" | \
        mail -s "[$HOSTNAME] DB Backup Empty" "$MAILTO"
    exit 1
fi

# Send success summary
echo "Backup completed: $BACKUP_FILE ($(du -h "$BACKUP_FILE" | cut -f1))" | \
    mail -s "[$HOSTNAME] DB Backup OK" "$MAILTO"
```

Crontab entry:

```bash
# Let the script handle its own email, silence cron output
0 3 * * * /usr/local/bin/db-backup.sh > /dev/null 2>&1
```

### Script with Email Containing Attachment

```bash
#!/bin/bash
# /usr/local/bin/daily-report.sh
# Called from crontab: 0 7 * * * /usr/local/bin/daily-report.sh

MAILTO="team@example.com"
REPORT="/tmp/daily-report-$(date +%Y%m%d).txt"

{
    echo "=== Daily System Report - $(date '+%Y-%m-%d') ==="
    echo ""
    echo "--- Disk Usage ---"
    df -h
    echo ""
    echo "--- Memory ---"
    free -h
    echo ""
    echo "--- Top Processes ---"
    ps aux --sort=-%mem | head -10
    echo ""
    echo "--- Recent Failed Logins ---"
    lastb 2>/dev/null | head -10
} > "$REPORT"

# Send report as email body
mail -s "Daily Report - $(date '+%Y-%m-%d')" "$MAILTO" < "$REPORT"

# Cleanup
rm -f "$REPORT"
```

Crontab entry:

```bash
# Script handles email internally, log errors only
0 7 * * * /usr/local/bin/daily-report.sh 2>> /var/log/report-errors.log
```

### Script with Conditional Email (Alert Only on Problems)

```bash
#!/bin/bash
# /usr/local/bin/disk-check.sh
# Called from crontab: */30 * * * * /usr/local/bin/disk-check.sh

THRESHOLD=85
MAILTO="sysadmin@example.com"
HOSTNAME=$(hostname)
ALERT=""

while read -r line; do
    usage=$(echo "$line" | awk '{print $5}' | sed 's/%//')
    mount=$(echo "$line" | awk '{print $6}')
    if [ "$usage" -ge "$THRESHOLD" ]; then
        ALERT="${ALERT}WARNING: $mount is ${usage}% full\n"
    fi
done < <(df -h | grep -E '^/dev/' )

if [ -n "$ALERT" ]; then
    echo -e "$ALERT" | mail -s "[$HOSTNAME] Disk Space Alert" "$MAILTO"
fi
```

Crontab entry:

```bash
# No output unless there's an error in the script itself
*/30 * * * * /usr/local/bin/disk-check.sh 2>> /var/log/disk-check-errors.log
```

### Script Redirecting to Multiple Destinations

```bash
#!/bin/bash
# /usr/local/bin/deploy.sh
# Called from crontab: 0 4 * * * /usr/local/bin/deploy.sh

LOGFILE="/var/log/deploy/deploy-$(date +%Y%m%d).log"
MAILTO="devops@example.com"

mkdir -p /var/log/deploy

# Redirect all output to log file AND capture for email
exec > >(tee -a "$LOGFILE") 2>&1

echo "=== Deploy started at $(date) ==="

cd /opt/myapp || exit 1
git pull origin main
npm install --production
systemctl restart myapp

if systemctl is-active --quiet myapp; then
    echo "=== Deploy successful at $(date) ==="
else
    echo "=== Deploy FAILED at $(date) ==="
    mail -s "Deploy Failed" "$MAILTO" < "$LOGFILE"
    exit 1
fi
```

Crontab entry:

```bash
# Script manages its own logging with tee, discard cron output
0 4 * * * /usr/local/bin/deploy.sh > /dev/null 2>&1
```

### Script Using MAILTO with Different Recipients per Job

```bash
# Different email recipients for different jobs
MAILTO="dba@example.com"
0 2 * * * /usr/local/bin/db-backup.sh
0 3 * * * /usr/local/bin/db-optimize.sh

MAILTO="sysadmin@example.com"
0 4 * * * /usr/local/bin/system-cleanup.sh
*/10 * * * * /usr/local/bin/healthcheck.sh

# Disable email for noisy jobs
MAILTO=""
* * * * * /usr/local/bin/metrics-collector.sh

# Re-enable for important jobs
MAILTO="oncall@example.com"
*/5 * * * * /usr/local/bin/critical-monitor.sh
```

### Script with Output Piped to Logger (syslog)

```bash
#!/bin/bash
# /usr/local/bin/cleanup.sh
# All output goes to syslog via logger

# Redirect stdout and stderr to logger
exec 1> >(logger -t "cron-cleanup" -p local0.info)
exec 2> >(logger -t "cron-cleanup" -p local0.error)

echo "Starting cleanup at $(date)"
find /tmp -type f -mtime +7 -delete
find /var/log -name "*.gz" -mtime +30 -delete
echo "Cleanup finished at $(date)"
```

Crontab entry:

```bash
# All output handled by logger inside the script
0 3 * * * /usr/local/bin/cleanup.sh
```

### Inline Cron Redirections Cheatsheet

```bash
# Redirect stdout only to file (stderr still emailed)
0 * * * * /path/to/script.sh > /var/log/output.log

# Redirect stderr only to file (stdout still emailed)
0 * * * * /path/to/script.sh 2> /var/log/errors.log

# Redirect stdout to file, stderr to same file
0 * * * * /path/to/script.sh > /var/log/all.log 2>&1

# Append stdout to file, stderr to same file
0 * * * * /path/to/script.sh >> /var/log/all.log 2>&1

# Redirect stdout to one file, stderr to another
0 * * * * /path/to/script.sh >> /var/log/out.log 2>> /var/log/err.log

# Discard stdout, redirect stderr to file
0 * * * * /path/to/script.sh > /dev/null 2>> /var/log/errors.log

# Discard everything
0 * * * * /path/to/script.sh > /dev/null 2>&1

# Pipe output to mail command
0 * * * * /path/to/script.sh 2>&1 | mail -s "Cron Output" user@example.com

# Pipe output to logger for syslog
0 * * * * /path/to/script.sh 2>&1 | logger -t mycron

# Tee to both file and stdout (stdout still emailed by cron)
0 * * * * /path/to/script.sh 2>&1 | tee -a /var/log/output.log
```
