# Solaris User and Password Administration

Managing user accounts and password policy on Oracle Solaris — the `passwd` and `useradd` commands for account/password control, the `/etc` files that store account data and policy, and Solaris's reserved UID ranges.

## Password and Account Commands

```bash
# Show password attributes (status, aging) for a login
passwd -s username

# Lock / unlock an account's password
passwd -l username        # lock
passwd -u username        # unlock

# Force the user to change their password at next login
passwd -f username

# Turn off password aging but let the user keep the current password
passwd -x -1 username

# Show the default values used when creating a new account
useradd -D
```

| Command | Purpose |
|---------|---------|
| `passwd -s username` | Show password status/aging attributes |
| `passwd -l username` | Lock the account's password |
| `passwd -u username` | Unlock a locked password |
| `passwd -f username` | Force a password change at next login |
| `passwd -x -1 username` | Disable aging, keep current password |
| `useradd -D` | Display (or set) account-creation defaults |

- `passwd -s` output shows whether the password is passworded (`PS`), locked (`LK`), or empty (`NP`), plus aging fields (last change, min, max, warn).
- `passwd -x -1` sets the maximum-days field to `-1`, which turns aging off. To *force* a change at next login use `passwd -f` (not `-x`).
- `useradd -D` with no other options prints the defaults; with options it updates them (default group, base dir, skel, shell, etc.).

Sample `passwd -s alice` output:

```
alice  PS  06/01/24   7  90   14
#      │    │          │  │    └ warn days before expiry
#      │    │          │  └───── max days between changes
#      │    │          └──────── min days between changes
#      │    └─────────────────── last change date
#      └──────────────────────── status: PS=passworded, LK=locked, NP=no password
```

## Account Lifecycle (useradd / usermod / userdel)

```bash
# Create a user with home dir, shell, primary + secondary groups
useradd -m -d /export/home/alice -s /usr/bin/bash \
        -g staff -G webadmin -c "Alice Smith" alice
passwd alice                       # set the initial password

# Modify an existing account
usermod -G webadmin,dba alice      # set secondary groups
usermod -s /usr/bin/ksh alice      # change shell
usermod -l alice2 alice            # rename login
usermod -u 1234 alice              # change UID

# Delete an account (and its home directory with -r)
userdel -r alice
```

- `-m -d <dir>` — create the home directory (from `/etc/skel`).
- `-g` primary group, `-G` comma-separated secondary groups, `-c` GECOS/comment.
- On Solaris 11, `useradd` defaults to creating a ZFS dataset for the home directory when `-m` is used with an auto-home setup.

## Group Management

```bash
groupadd -g 200 webadmin           # create a group with a specific GID
groupmod -n webteam webadmin       # rename a group
groupdel webteam                   # delete a group
groups alice                       # show a user's group membership
```

## Account and Policy Files

| File | Contents |
|------|----------|
| `/etc/passwd` | Account entries (login, UID, GID, GECOS, home, shell) |
| `/etc/shadow` | Encrypted passwords and aging fields |
| `/etc/group` | Group definitions and membership |
| `/etc/default/passwd` | Default password policy (length, aging, complexity) |
| `/etc/default/login` | Login defaults, including the `RETRIES` parameter |
| `/etc/security/policy.conf` | System-wide security policy (e.g. account lockout) |
| `/etc/user_attr` | Associates users/roles with authorizations and profiles (RBAC) |

### `/etc/default/passwd`

Default password policy applied when passwords are set — minimum/maximum age, minimum length, and complexity rules (e.g. `PASSLENGTH`, `MAXWEEKS`, `MINWEEKS`, `WARNWEEKS`).

### `/etc/shadow`

Holds the encrypted password and per-user aging fields (last change, min, max, warn, inactive, expire). This is what `passwd -s` summarizes.

### Account Lockout

Locking an account after repeated failed logins takes two settings working together:

```bash
# 1. Enable lockout system-wide in /etc/security/policy.conf
LOCK_AFTER_RETRIES=YES

# 2. Set the retry threshold in /etc/default/login
RETRIES=5
```

- `LOCK_AFTER_RETRIES=YES` in `/etc/security/policy.conf` turns on automatic account locking.
- `RETRIES` in `/etc/default/login` defines how many consecutive failures trigger it.
- Per-user override is possible via `lock_after_retries` in `/etc/user_attr`.

### `/etc/user_attr` (RBAC)

Maps users and roles to authorizations and rights profiles — the basis of Solaris Role-Based Access Control (RBAC). It's also where a per-user `lock_after_retries` keyword can be set.

RBAC lets you grant admin capabilities without sharing root. Instead of logging in as root, users assume a **role** (via `su`) that carries a **rights profile**:

```bash
# Create a role and assign a rights profile to it
roleadd -m -d /export/home/netadmin -P "Network Management" netadmin
passwd netadmin

# Allow a user to assume the role
usermod -R netadmin alice

# List profiles/authorizations available to the current user
profiles
auths

# Run a single command with a profile's privileges
pfexec dladm show-link

# Assume a role
su - netadmin
```

- On Solaris 11, `root` itself is often a **role**, not a login — you log in as a normal user and `su` to root. Check with `roles <user>`.
- Profiles are defined in `/etc/security/prof_attr` and `/etc/security/exec_attr`.

## UID Conventions

Solaris reserves specific UID ranges and IDs:

| UID | Reserved for |
|-----|--------------|
| `0`–`99` | System accounts (root is 0; reserved for the OS/vendor) |
| `100`+ | Regular user accounts (typical start) |
| `60001` | `nobody` account |
| `60002` | `noaccess` account |
| `65534` | `nobody4` (historical, 16-bit `nobody`) |

Keep regular users at UID `100` and above to avoid colliding with system accounts and the reserved special IDs.

## Common Workflows

```bash
# Check an account's status and aging
passwd -s alice

# Lock an account (e.g. employee leaving), then unlock later
passwd -l alice
passwd -u alice

# Force a password reset at next login (e.g. after admin sets a temp password)
passwd alice
passwd -f alice

# Inspect account-creation defaults before adding users
useradd -D
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| User can't log in, account "locked" | Too many failed logins (lockout) | `passwd -u <user>` to unlock; review `RETRIES` |
| Login rejected immediately | Password status `LK`/`NP` | `passwd -s` to check; set a password or unlock |
| Password change refused "too soon" | `MINWEEKS`/min-days aging | Wait, or admin resets aging with `passwd -x` |
| New user has no home dir | Created without `-m` | `usermod -m -d <dir> <user>` or recreate with `-m` |
| Can't run admin command | Missing rights profile/role | Assign via `usermod -P`/`-R`; use `pfexec` |
| Root login refused on Solaris 11 | root is a role, not a login | Log in as a user, then `su - root` |

```bash
# Verify a user's roles, profiles, and authorizations
roles alice ; profiles alice ; auths alice
```

## Command and File Reference

| Task | Command / File |
|------|----------------|
| Show password status | `passwd -s username` |
| Lock / unlock | `passwd -l` / `passwd -u username` |
| Force change at login | `passwd -f username` |
| Disable aging | `passwd -x -1 username` |
| Account-creation defaults | `useradd -D` |
| Encrypted passwords | `/etc/shadow` |
| Password policy defaults | `/etc/default/passwd` |
| Login retries | `/etc/default/login` (`RETRIES`) |
| Enable lockout | `/etc/security/policy.conf` (`LOCK_AFTER_RETRIES=YES`) |
| RBAC user attributes | `/etc/user_attr` |
| Groups | `/etc/group` |

## References

- [Managing User Accounts and Groups in Oracle Solaris](https://docs.oracle.com/cd/E37838_01/html/E56528/index.html) — official Oracle docs
- [passwd(1) man page](https://docs.oracle.com/cd/E23824_01/html/821-1461/passwd-1.html) — official Oracle docs
- [Securing Users and Processes in Oracle Solaris (RBAC)](https://docs.oracle.com/cd/E37838_01/html/E54830/index.html) — official Oracle docs
