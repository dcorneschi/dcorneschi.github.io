# AIX Login Auditing and Session Tracking Cheatsheet

Command reference for tracking logins and sessions on IBM AIX — reviewing failed and successful logins with `who` against the accounting files, auditing `su` usage, checking last-login history (`last`, `lastlog`), inspecting a user's environment, killing a user's processes, and controlling which groups may `su` to root.

> Reading the security and accounting files (`/etc/security/failedlogin`, `/var/adm/sulog`, `/etc/security/lastlog`) and changing `su` group membership require `root`. These files are the primary trail for investigating suspicious access — review them before making account changes.

## Login and Session History

`who` reads the accounting databases; point it at a specific file to review historical events instead of current sessions.

```sh
# All failed login attempts
who /etc/security/failedlogin

# All login and logout events (connect-time accounting)
who /var/adm/wtmp

# Last login for each user (from /etc/utmp)
who -a

# Show the current run level
who -r
```

| File | Contents |
|------|----------|
| `/etc/security/failedlogin` | Records of failed login attempts |
| `/var/adm/wtmp` | Login/logout (connect-time) history |
| `/etc/utmp` | Currently logged-in users |
| `/etc/security/lastlog` | Per-user last-login details |
| `/var/adm/sulog` | All `su` command invocations |

## su Auditing

```sh
# All invocations of the su command (success and failure)
cat /var/adm/sulog

# Detailed last-login information per user
cat /etc/security/lastlog
```

Each `/var/adm/sulog` line shows the date/time, success (`+`) or failure (`-`), the terminal, and the from/to users — useful for spotting unexpected privilege escalation.

## Last Login

```sh
# Last login time and session history for users
last

# Same idea, per-user detail via the security file
cat /etc/security/lastlog
```

## User Environment and Processes

```sh
# Display the exported environment (name=value pairs)
export -p

# Kill all of a user's processes (here: daniel)
kill -9 `ps -ef | awk '$1=="daniel" {print $2}'`
```

> The `kill` one-liner extracts PIDs owned by the user with `awk` and sends `SIGKILL`. Run `ps -ef | awk '$1=="daniel"'` first to review what will be killed — `-9` gives processes no chance to clean up.

## Controlling su to root

The `sugroups` attribute on an account lists the groups whose members are allowed to `su` to it.

```sh
# Show which groups may su to root
lsuser -a sugroups root

# Allow only the system and adm groups to su to root
chuser sugroups="system,adm" root
```

## Quick Reference

| Task | Command |
|------|---------|
| Failed logins | `who /etc/security/failedlogin` |
| Login/logout history | `who /var/adm/wtmp` |
| Last login per user | `who -a` |
| Current run level | `who -r` |
| su invocations | `cat /var/adm/sulog` |
| Last-login details | `cat /etc/security/lastlog` |
| Last login history | `last` |
| Show environment | `export -p` |
| Kill a user's processes | `kill -9 \`ps -ef \| awk '$1=="user"{print $2}'\`` |
| Who can su to root | `lsuser -a sugroups root` |
| Set groups allowed to su to root | `chuser sugroups="system,adm" root` |

## Related

- [AIX Users and Groups Cheatsheet](articles/aix-users-groups-cheatsheet.md) — creating and managing the accounts, passwords, and login controls audited here.
- [AIX Error Logging and System Logs Cheatsheet](articles/aix-error-logging-cheatsheet.md) — `errpt` and the error log for broader security and system event auditing.
