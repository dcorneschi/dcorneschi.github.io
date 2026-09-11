# Configure Automatic Updates with unattended-upgrades

Install and configure `unattended-upgrades` on Ubuntu and Debian to apply security updates automatically. This guide covers scheduling, allowed repositories, package exclusions, email reports, controlled reboots, dry runs, logs, and troubleshooting.

> **Important:** Automatic patching reduces exposure to known vulnerabilities but does not replace monitoring, backups, maintenance windows, or application testing. Test repository and reboot policies before enabling them on production systems.

## Install unattended-upgrades

Refresh the package index and install the package:

```bash
sudo apt update
sudo apt install unattended-upgrades
```

Enable the default automatic security-update configuration interactively:

```bash
sudo dpkg-reconfigure -plow unattended-upgrades
```

Check the package and relevant systemd units:

```bash
dpkg-query -W unattended-upgrades
systemctl status unattended-upgrades --no-pager -l
systemctl status apt-daily.timer apt-daily-upgrade.timer --no-pager
systemctl list-timers 'apt-daily*'
```

The actual scheduled APT work is normally launched by `apt-daily.timer` and `apt-daily-upgrade.timer`. The `unattended-upgrades.service` unit also participates in shutdown handling, so its status alone does not show the complete schedule.

## Configure Ubuntu Desktop from the GUI

Ubuntu Desktop users can enable automatic security updates without editing APT configuration files directly:

1. Open **Software & Updates**.
2. Select the **Updates** tab.
3. Set **Automatically check for updates** to the desired interval, normally **Daily**.
4. Set **When there are security updates** to **Download and install automatically**.
5. Authenticate when prompted and close the application.

Option labels can vary slightly between Ubuntu releases. The GUI updates the same underlying APT configuration, so verify the resulting policy and timers from a terminal:

```bash
apt-config dump | grep -iE 'APT::Periodic|Unattended-Upgrade'
systemctl list-timers 'apt-daily*' --all
sudo unattended-upgrade --dry-run --debug
```

The desktop interface does not expose every setting in `50unattended-upgrades`. Use the configuration files for package exclusions, repository-origin rules, mail reports, automatic reboot policy, bandwidth limits, and syslog integration.

Do not schedule a separate `apt update && apt upgrade -y` script after enabling automatic updates in the GUI. It bypasses unattended-upgrades policy controls and can overlap the APT systemd timers.

## Understand the Configuration Files

APT configuration fragments are stored under `/etc/apt/apt.conf.d/` and processed in lexical order. The two main unattended-upgrade files are:

| File | Purpose |
|------|---------|
| `/etc/apt/apt.conf.d/20auto-upgrades` | Enables periodic package-list refreshes and unattended upgrades |
| `/etc/apt/apt.conf.d/50unattended-upgrades` | Controls repositories, package exclusions, notifications, cleanup, and reboots |

Back up the files before modifying them:

```bash
sudo cp -a /etc/apt/apt.conf.d/20auto-upgrades \
  /etc/apt/apt.conf.d/20auto-upgrades.bak

sudo cp -a /etc/apt/apt.conf.d/50unattended-upgrades \
  /etc/apt/apt.conf.d/50unattended-upgrades.bak
```

Inspect the effective merged configuration with:

```bash
apt-config dump | grep -iE 'APT::Periodic|Unattended-Upgrade'
```

## Configure `20auto-upgrades`

Edit `/etc/apt/apt.conf.d/20auto-upgrades`:

```bash
sudo vi /etc/apt/apt.conf.d/20auto-upgrades
```

A daily configuration contains:

```text
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

Optional periodic download and cache-cleanup settings are:

```text
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
```

The directives mean:

| Directive | Purpose |
|-----------|---------|
| `APT::Periodic::Update-Package-Lists` | Refresh package indexes, similar to `apt update` |
| `APT::Periodic::Download-Upgradeable-Packages` | Download eligible package files before installation without installing them |
| `APT::Periodic::Unattended-Upgrade` | Run the unattended upgrade process |
| `APT::Periodic::AutocleanInterval` | Remove obsolete cached package files at the configured interval |

The numeric value is an interval in days:

| Value | Behavior |
|-------|----------|
| `"0"` | Disabled |
| `"1"` | Run daily when due |
| `"2"` | Run when at least two days have elapsed |
| `"7"` | Run when at least seven days have elapsed |

The interval does not guarantee an exact clock time. Systemd timers may introduce randomized delays to prevent many systems from contacting repositories simultaneously. Pre-downloading consumes disk space and network bandwidth before the installation window, so enable it only when that behavior is desirable.

To disable periodic unattended installation, set `APT::Periodic::Unattended-Upgrade` to `"0"`. Enabling or disabling only `unattended-upgrades.service` does not control the complete APT timer schedule.

## Configure `50unattended-upgrades`

Edit the policy file:

```bash
sudo vi /etc/apt/apt.conf.d/50unattended-upgrades
```

The package installs a distribution-specific configuration. Preserve the existing repository labels and variables rather than replacing the entire file with an example from another Ubuntu or Debian release.

### Recover Interrupted dpkg Configuration

`AutoFixInterruptedDpkg` allows unattended-upgrades to attempt recovery when an earlier package operation left packages unpacked but not fully configured:

```text
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
```

This behavior is commonly enabled by default. It helps complete interrupted package configuration before applying new upgrades, but it cannot repair every dependency, maintainer-script, or filesystem failure. Review the unattended-upgrades and `dpkg` logs whenever automatic recovery is triggered.

Confirm the effective value installed on the system:

```bash
apt-config dump | grep -i AutoFixInterruptedDpkg
```

### Select Allowed Repositories

The allowed origins section commonly resembles:

```text
Unattended-Upgrade::Allowed-Origins {
        "${distro_id}:${distro_codename}";
        "${distro_id}:${distro_codename}-security";
        // "${distro_id}:${distro_codename}-updates";
};
```

Security repositories are normally enabled by default. To include regular stable updates, uncomment the matching `-updates` line:

```text
        "${distro_id}:${distro_codename}-updates";
```

> **Caution:** Enabling regular updates increases the scope beyond security fixes and can introduce larger behavioral changes. Validate this policy against application support requirements and maintenance procedures.

Ubuntu releases may also include Ubuntu Pro or Expanded Security Maintenance origins. Debian may use `Origins-Pattern` or different archive labels. Modify the file installed on the system instead of assuming origin names are interchangeable.

Display configured repositories and package candidates before changing the policy:

```bash
apt-cache policy
apt list --upgradable
```

### Exclude Packages

Add package names or regular expressions under `Unattended-Upgrade::Package-Blacklist`:

```text
Unattended-Upgrade::Package-Blacklist {
        "docker-ce";
        "postgresql-.*";
        "example-package";
};
```

Patterns are matched against package names. Keep exclusions narrow and document why each package requires a separate maintenance process.

To hold a package from all APT upgrades, not only unattended upgrades, use `apt-mark`:

```bash
sudo apt-mark hold example-package
apt-mark showhold
sudo apt-mark unhold example-package
```

A package hold is broader than the unattended-upgrades blacklist.

## Configure Email Notifications

A local mail transport agent or SMTP relay must be able to deliver system mail. Set a recipient in `/etc/apt/apt.conf.d/50unattended-upgrades`:

```text
Unattended-Upgrade::Mail "admin@example.com";
```

On versions that support `MailReport`, select when reports are sent:

```text
Unattended-Upgrade::MailReport "on-change";
```

Common policies include `always`, `only-on-error`, and `on-change`; available values can vary by package version. Review the comments in the installed configuration and the local manual page:

```bash
man unattended-upgrade
```

Older releases may instead use:

```text
Unattended-Upgrade::MailOnlyOnError "true";
```

Test mail delivery independently before relying on update notifications.

## Configure Automatic Reboots

Some updates, particularly kernel and core-library updates, require a reboot. To reboot automatically when `/var/run/reboot-required` exists, configure:

```text
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "false";
Unattended-Upgrade::Automatic-Reboot-Time "02:00";
```

These settings mean:

| Directive | Behavior |
|-----------|----------|
| `Automatic-Reboot` | Permit a reboot when one is required |
| `Automatic-Reboot-WithUsers` | When `false`, do not reboot while users are logged in |
| `Automatic-Reboot-Time` | Schedule the required reboot for the specified local time |

> **Production systems:** Coordinate automatic reboots with clustering, maintenance windows, backups, monitoring silences, and workload-draining procedures. A fixed reboot time does not make an uncoordinated reboot safe.

To leave rebooting as a manual operation:

```text
Unattended-Upgrade::Automatic-Reboot "false";
```

Check whether the system currently requires a reboot:

```bash
if [ -f /var/run/reboot-required ]; then
    echo 'Reboot required'
    cat /var/run/reboot-required.pkgs 2>/dev/null
else
    echo 'No reboot required'
fi
```

## Configure Cleanup Behavior

Optional cleanup directives may be present in `50unattended-upgrades`:

```text
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Dependencies "false";
```

Review removals carefully before enabling automatic dependency cleanup. Packages incorrectly marked as automatically installed can otherwise be removed unexpectedly.

Preview unused packages with:

```bash
sudo apt autoremove --dry-run
```

## Validate the Configuration

Check APT's parsed configuration:

```bash
apt-config dump | grep -iE 'APT::Periodic|Unattended-Upgrade'
```

Run a dry run with debug output:

```bash
sudo unattended-upgrade --dry-run
sudo unattended-upgrade --dry-run --debug
```

The package is named `unattended-upgrades`, but the command is normally `unattended-upgrade` in the singular.

A dry run does not install packages. Review:

- Which repositories are allowed
- Which packages would be upgraded
- Which packages are excluded or held
- Whether package dependencies can be resolved
- Whether a reboot would be required

To list packages that are currently upgradeable:

```bash
apt list --upgradable
```

## Apply and Trigger Updates

Configuration files are read each time the job runs, so a service restart is not normally required. If operational procedures require refreshing the helper service, use:

```bash
sudo systemctl restart unattended-upgrades
sudo systemctl status unattended-upgrades --no-pager -l
```

To trigger the systemd upgrade job:

```bash
sudo systemctl start apt-daily-upgrade.service
```

To run unattended-upgrades directly and apply eligible updates:

```bash
sudo unattended-upgrade --debug
```

> **Warning:** Unlike `--dry-run`, a direct run can install packages. Run it only during an approved maintenance window and do not start it while another APT or `dpkg` process holds the package locks.

## Review Logs and Activity

The primary logs are stored in `/var/log/unattended-upgrades/`:

| Log | Contents |
|-----|----------|
| `unattended-upgrades.log` | Selected packages, decisions, results, and errors |
| `unattended-upgrades-dpkg.log` | Package installation output from `dpkg` |
| `unattended-upgrades-shutdown.log` | Shutdown-related activity, when present |

Review recent activity:

```bash
sudo tail -n 50 /var/log/unattended-upgrades/unattended-upgrades.log
sudo tail -n 50 /var/log/unattended-upgrades/unattended-upgrades-dpkg.log
```

Review systemd journal messages:

```bash
journalctl -u unattended-upgrades --since '-24 hours' --no-pager
journalctl -u apt-daily-upgrade.service --since '-24 hours' --no-pager
```

Review APT and `dpkg` history:

```bash
sudo tail -n 100 /var/log/apt/history.log
sudo grep -E ' upgrade | install ' /var/log/dpkg.log | tail -n 50
```

## Check the Schedule

Inspect when the timers last ran and when they will run next:

```bash
systemctl list-timers 'apt-daily*' --all
systemctl cat apt-daily.timer
systemctl cat apt-daily-upgrade.timer
```

The timers use settings such as `OnCalendar`, `RandomizedDelaySec`, and `Persistent`. A persistent timer can run shortly after boot if the machine was powered off at the scheduled time.

Do not modify vendor timer units directly under `/lib/systemd/system` or `/usr/lib/systemd/system`. Use a systemd drop-in if the schedule must be customized:

```bash
sudo systemctl edit apt-daily-upgrade.timer
```

After changing a timer override:

```bash
sudo systemctl daemon-reload
sudo systemctl restart apt-daily-upgrade.timer
systemctl list-timers 'apt-daily*' --all
```

## Additional Repository and Delivery Controls

### Treat proposed and backports as opt-in

Ubuntu configurations may contain commented `-proposed` and `-backports` origins:

```text
// "${distro_id}:${distro_codename}-proposed";
// "${distro_id}:${distro_codename}-backports";
```

Do not enable `-proposed` for routine unattended upgrades on production systems. It contains packages undergoing validation and can introduce regressions. Backports provide newer packages rebuilt for an older release, but they should normally be enabled only for deliberately selected packages rather than applied automatically across the system.

### Limit download bandwidth

APT can limit HTTP download bandwidth when updates must not saturate a constrained link:

```text
Acquire::http::Dl-Limit "70";
```

The value is expressed in kilobytes per second. APT may not use parallel downloads while this limit is active, so enable it only when bandwidth control is necessary.

### Send messages to syslog

Enable unattended-upgrade messages through syslog when a central log collector consumes the system journal or syslog stream:

```text
Unattended-Upgrade::SyslogEnable "true";
Unattended-Upgrade::SyslogFacility "daemon";
```

Verify the effective behavior with the local package version, then query recent messages:

```bash
journalctl -t unattended-upgrade --since '-24 hours' --no-pager
journalctl -u apt-daily-upgrade.service --since '-24 hours' --no-pager
```

### Avoid duplicate schedulers

Do not add a cron entry for `/usr/bin/unattended-upgrade` while `apt-daily-upgrade.timer` is enabled. Two independent schedulers can overlap, contend for APT locks, produce duplicate notifications, and make run times harder to diagnose.

Prefer the existing systemd timer and customize it with `systemctl edit apt-daily-upgrade.timer`. If an external patch-management platform must become the only scheduler, document the change and deliberately disable the superseded schedule rather than layering another scheduler on top.

## Troubleshooting

| Symptom | Likely cause | Check or fix |
|---------|--------------|--------------|
| No packages are upgraded | Only security origins are allowed or no eligible updates exist | Run the debug dry run and inspect allowed origins |
| `unattended-upgrades: command not found` | Package name was used as the executable | Run `unattended-upgrade` in the singular |
| APT lock error | Another APT, `dpkg`, or unattended job is active | Inspect processes and systemd units; wait for the current job |
| Timer appears inactive | Timer disabled or masked | Run `systemctl status apt-daily-upgrade.timer` |
| Email never arrives | No working local mail transport or relay | Test mail delivery independently and inspect mail logs |
| System did not reboot | Reboot not required, users logged in, or reboot policy disabled | Check `/var/run/reboot-required` and configured directives |
| System rebooted unexpectedly | Automatic reboot enabled without an appropriate window | Disable `Automatic-Reboot` and review the logs |
| Package is repeatedly skipped | Package blacklist, APT hold, pinning, dependency conflict, or phased update | Check blacklist, `apt-mark showhold`, `apt-cache policy`, and debug output |
| Configuration is ignored | Syntax error or overridden by a later APT fragment | Run `apt-config dump` and inspect `/etc/apt/apt.conf.d/` ordering |

Check for active package-management processes without killing them:

```bash
ps aux | grep -E '(apt|dpkg|unattended-upgrade)' | grep -v grep
systemctl is-active unattended-upgrades apt-daily.service apt-daily-upgrade.service
```

Never delete APT lock files while a package manager is active. Interrupting `dpkg` can leave the package database inconsistent.

If package configuration was interrupted, first confirm that no package manager is running, then repair it:

```bash
sudo dpkg --configure -a
sudo apt --fix-broken install
```

## Recommended Baseline

For a typical server where only security updates should be automatic:

```text
// /etc/apt/apt.conf.d/20auto-upgrades
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

In `/etc/apt/apt.conf.d/50unattended-upgrades`:

1. Keep the distribution's security origins enabled.
2. Leave the regular `-updates` origin disabled unless it is approved.
3. Add only documented package exclusions.
4. Configure a tested email recipient or external monitoring.
5. Set `Automatic-Reboot` according to the maintenance policy.
6. Use `Automatic-Reboot-WithUsers "false"` when automatic rebooting is enabled.
7. Validate with `unattended-upgrade --dry-run --debug`.
8. Monitor logs and timer execution.

## Quick Reference

```bash
# Install and enable the default policy
sudo apt update
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# Inspect configuration and schedule
apt-config dump | grep -iE 'APT::Periodic|Unattended-Upgrade'
systemctl list-timers 'apt-daily*' --all

# Test without installing
sudo unattended-upgrade --dry-run --debug

# Check status and logs
systemctl status unattended-upgrades --no-pager -l
sudo tail -n 50 /var/log/unattended-upgrades/unattended-upgrades.log
journalctl -u apt-daily-upgrade.service --since '-24 hours' --no-pager

# Check whether a reboot is required
test -f /var/run/reboot-required && cat /var/run/reboot-required
```

## See Also

- [apt Cheatsheet](articles/apt-cheatsheet.md) — package installation, updates, simulation, and maintenance
- [Fixing apt Lock Held Errors](articles/apt-lock-held-fix.md) — diagnose package-manager lock contention safely
- [Postfix Gmail SMTP Relay Setup](articles/postfix-gmail-relay.md) — configure outbound email for update reports
- [systemd Cheatsheet](articles/systemd-cheatsheet.md) — inspect and customize services and timers
- [journalctl Cheatsheet](articles/journalctl-cheatsheet.md) — query unattended-upgrade and APT service logs
