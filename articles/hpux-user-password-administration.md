# HP-UX User and Password Administration

Managing users, groups, and password policy on HP-UX — the `useradd`/`usermod`/`groupmod` account commands, `passwd` aging and status, shadow-password conversion, and the **Trusted Computing Base (TCB)** tools (`getprpw`/`modprpw`) used on trusted systems. Covers HP-UX 11i v1–v3.

## Three Security Modes You Might Be In

The single most important thing to establish before touching accounts on an HP-UX box is *which security model it runs*, because the same policy (say, "expire this password in 90 days") is configured in completely different places in each:

1. **Standard (untrusted)** — the traditional UNIX model. Encrypted passwords sit in `/etc/passwd`, aging is stored in the password field, and you manage everything with `passwd` and the account commands.
2. **Trusted System (TCB)** — the box has been converted to the **Trusted Computing Base**. Per-user security attributes (password, aging, lockout, failed-login counts) move out of `/etc/passwd` into protected files under `/tcb/files/auth/`, and you manage them with `getprpw`/`modprpw`. This model predates shadow passwords and adds auditing and stricter policy.
3. **Shadow passwords (11i v3)** — a middle ground introduced later: encrypted passwords move to `/etc/shadow` (readable only by root) while `/etc/passwd` stays world-readable for name/UID lookups. You still use `passwd` and the account commands.

A system is either standard, trusted, **or** shadowed — they are alternative back-ends for the same account data. Running the wrong toolset (e.g. `passwd -x` on a trusted system) either fails or silently does nothing useful, so always check the mode first.

```bash
# Is the system trusted (TCB)? Non-error output means yes.
/usr/lbin/getprdef -r

# Is shadow in use? The file exists and is populated after pwconv.
grep -c . /etc/shadow 2>/dev/null
```

## Listing and Inspecting Users

```bash
# List all user accounts (system + regular)
logins

# Show the username-length setting: 64 = max 8-char names,
# 256 = long usernames enabled
lugadmin -l

# A user's group memberships
groups user1

# Password status: last change date and days until expiry
passwd -s root

# Status of all users
passwd -sa
```

`passwd -s` status codes:

| Code | Meaning |
|------|---------|
| `PS` | Passworded (has a password set) |
| `LK` | Locked |
| `NP` | No password |

## Editing Account Files Safely

```bash
# Lock /etc/passwd for safe editing (copies it to /etc/ptmp while you edit)
vipw
```

`vipw` serializes edits to `/etc/passwd` so two admins can't clobber each other.

## Shadow Passwords

HP-UX 11i v3 supports `/etc/shadow`; convert to and from it with:

```bash
pwconv      # move encrypted passwords into /etc/shadow
pwunconv    # revert to traditional non-shadowed passwords in /etc/passwd
```

The point of shadowing is that `/etc/passwd` must stay world-readable — countless tools resolve UIDs to names by reading it — yet a world-readable file is a terrible place to keep password hashes, since anyone can copy them and mount an offline dictionary attack. `pwconv` moves the hash into `/etc/shadow`, which is readable only by root, and leaves a placeholder `x` in the password field of `/etc/passwd`. The aging fields (last change, min/max/warn) move to shadow too. Note that **shadow and Trusted System are mutually exclusive** — you cannot convert to shadow on a system that is already trusted; you would first untrust it. After `pwconv`, keep managing accounts with the normal `passwd`/`useradd` commands; they are shadow-aware and write to the right file automatically.

## Creating and Modifying Users

```bash
# Create a user WITH a password (crypt the password inline).
# Without -p, the account is created but disabled (no valid password).
useradd -p "$(perl -e "print crypt('hp','xx')")" user1

# Change a user's primary group
usermod -g users user1

# Add supplementary (secondary) groups
usermod -G class,training user1
```

- `-p <encrypted>` takes an already-crypted password (not plaintext) — hence the `perl crypt()` trick.
- `-g` sets the **primary** group; `-G` sets the comma-separated **secondary** groups.

Two other `useradd` options matter in practice: **`-m`** creates the home directory (and copies the `/etc/skel` skeleton files into it), and **`-d`** sets its path. Without `-m` the account exists but has no home directory, so the user lands in `/` on login. A fuller, self-contained account creation looks like:

```bash
# Create user with home dir, shell, comment, primary + secondary groups, and a password
useradd -m -d /home/user1 -s /sbin/sh -c "Jane Roe" \
  -g users -G class,training \
  -p "$(perl -e "print crypt('hp','xx')")" user1

# Set the defaults useradd uses when options are omitted
useradd -D                       # show current defaults
useradd -D -b /home -s /sbin/sh  # change default base dir and shell
```

> **Why the `crypt()` trick?** `useradd -p` expects the value that goes *literally* into the password field — i.e. an already-hashed string, not a plaintext password. Passing plaintext would store the plaintext as if it were the hash, leaving the account effectively unloginnable and insecure. The `perl -e "print crypt(...)"` call produces a valid traditional DES/`crypt` hash. In many shops the cleaner path is to create the account **without** `-p` (which leaves it locked) and then set the password interactively with `passwd user1`, so the plaintext never appears on a command line or in shell history.

## Group Membership

```bash
# Add users to a group (append)
groupmod -a -l user1,user2 accounts

# Replace the entire member list of a group
groupmod -m -l user3,user4 accounts

# Temporarily switch your active (primary) GID for this session
newgrp sales

# Return to your normal primary group
newgrp
```

> Verify a user actually has group access by acting as them:
>
> ```bash
> su user23 -c "touch /home/project/f23"
> ```

## Password Policy and Aging

```bash
# Force the user to change password at next login
passwd -f user1

# Expire the password after 90 days
passwd -x 90 root

# Disable password aging (non-trusted system)
passwd -x -1 user_id

# Remove a user's password (blank — dangerous)
passwd -d user1

# Lock the account (replaces the password with *)
passwd -l user1
```

### System-Wide Policy Defaults

Per-account `passwd` flags set policy one user at a time. To set defaults for *new* accounts and system-wide minimums on a standard (non-trusted) system, edit `/etc/default/security`. Common knobs:

```bash
PASSWORD_MIN_LENGTH=8        # minimum password length
PASSWORD_HISTORY_DEPTH=5     # remember N old passwords, block reuse
PASSWORD_MAXDAYS=90          # max age before forced change
PASSWORD_MINDAYS=1           # min days between changes
PASSWORD_WARNDAYS=7          # start warning N days before expiry
MIN_PASSWORD_LENGTH=8        # alternate/legacy name on some releases
```

These defaults feed into what individual `passwd -x`/`-n`/`-w` settings then override per user. The distinction to keep straight: `/etc/default/security` establishes the *policy floor and defaults*, while per-user `passwd` aging fields (or TCB attributes on a trusted system) hold the *actual* dates and limits enforced at login.

> **Locking vs disabling — know the difference.** `passwd -l` prepends/replaces the hash so no password can match (the account is *locked* but still exists and can run cron jobs or receive files). `passwd -d` removes the password entirely, which on many configurations means the account can log in with *no* password — almost never what you want. To truly stop interactive login, lock the password **and** set the shell to `/sbin/false` or `/usr/bin/false`.

## Removing a User and Their Files

```bash
# Delete the user's files, then empty directories
find / -user user1 -type f -exec rm -i {} +
find / -user user1 -type d -exec rmdir {} +

# Or transfer ownership to another user instead of deleting
find / -user user1 -exec chown user2 {} +
```

Find files left behind by deleted accounts:

```bash
find / -nouser  -exec ll -d {} +     # files with no owning user
find / -nogroup -exec ll -d {} +     # files with no owning group
```

## Trusted Systems (TCB)

On a **trusted** HP-UX system (converted to the Trusted Computing Base), per-user security attributes live under `/tcb` and are managed with `getprpw`/`modprpw` rather than the standard files.

The Trusted System model was HP-UX's answer to stricter security requirements before shadow passwords existed. Converting a system to trusted (historically via SAM, now SMH, or `tsconvert`) does several things at once: it moves password hashes out of the world-readable `/etc/passwd` into protected per-user profiles, and it enables features standard mode lacks — **failed-login counting with automatic lockout**, **per-user password aging and lifetimes**, **time-of-day login restrictions**, and stronger auditing. The trade-off is that the familiar `passwd`/`useradd` aging flags no longer tell the whole story; the authoritative values live in the TCB profile, so you must use `getprpw` to read them and `modprpw` to change them. A frequent support scenario is a user locked out by too many bad login attempts — `getprpw`'s lockout field tells you *why*, and `modprpw -k` clears it.

```bash
# Convert to / from a trusted system (also doable via SMH)
/usr/lbin/tsconvert            # convert to trusted
/usr/lbin/tsconvert -r         # revert to standard
```

```bash
# Is TCB active on this system? (checks the trusted default profile)
/usr/lbin/getprdef -r

# Show a user's protected password/account attributes
/usr/lbin/getprpw useraccount
```

Managing accounts with `modprpw`:

```bash
# Unlock / enable / reactivate an account
/usr/lbin/modprpw -k useraccount

# Lock / expire the password
/usr/lbin/modprpw -e useraccount

# Force a password change at first login
/usr/lbin/modprpw -x useraccount

# Remove password expiry (no max lifetime)
/usr/lbin/modprpw -l -m exptm=0 username

# Disable all account/password expiry
/usr/lbin/modprpw -l -m mintm=0,exptm=0,expwarn=0,lftm=0 USER-ID

# Administratively lock a user
/usr/lbin/modprpw -m alock=YES USER

# Un-expire (reactivate) a password
/usr/lbin/modprpw -v -l user
```

### Interpreting `getprpw` lockout

The lockout reason is a bit string (`0` = condition absent, `1` = present). Read left to right:

| Position | Lockout reason |
|:--------:|----------------|
| 1 | Past password lifetime |
| 2 | Past last login time (inactive account) |
| 3 | Past absolute account lifetime |
| 4 | Exceeded unsuccessful login attempts |
| 5 | Password required but null |
| 6 | Admin lock |
| 7 | Password is `*` |

```bash
# Show just the lockout field for a user
/usr/lbin/getprpw -m lockout useraccount
```

## Account and Policy Files

| File | Purpose |
|------|---------|
| `/etc/passwd` | Account entries (login, UID, GID, GECOS, home, shell) |
| `/etc/shadow` | Encrypted passwords (11i v3, after `pwconv`) |
| `/etc/group` | Group definitions and membership |
| `/etc/default/security` | System-wide security/password policy defaults |
| `/etc/default/useradd` | Defaults used by `useradd` |
| `/tcb/files/auth/system/default` | Trusted-system default security profile |
| `/tcb/files/auth/<c>/<user>` | Per-user trusted profile (filed by first letter of the name) |

On a trusted system, per-user attributes are stored under `/tcb/files/auth/` in a directory named for the first character of the username (e.g. user `bob` → `/tcb/files/auth/b/bob`).

## Command Reference

| Task | Command |
|------|---------|
| List users | `logins` |
| Username-length mode | `lugadmin -l` |
| Safe passwd edit | `vipw` |
| Enable/disable shadow | `pwconv` / `pwunconv` |
| Create user (with password) | `useradd -p "$(...)" user` |
| Set primary / secondary groups | `usermod -g` / `usermod -G` |
| Add/replace group members | `groupmod -a -l` / `groupmod -m -l` |
| Switch active group | `newgrp <group>` / `newgrp` |
| Force password change | `passwd -f` / TCB `modprpw -x` |
| Expire in N days | `passwd -x <days>` |
| Disable aging | `passwd -x -1` |
| Lock / blank password | `passwd -l` / `passwd -d` |
| Password status | `passwd -s` / `passwd -sa` |
| Find orphaned files | `find / -nouser`, `find / -nogroup` |
| TCB active? | `/usr/lbin/getprdef -r` |
| TCB account info | `/usr/lbin/getprpw <user>` |
| TCB unlock / lock | `modprpw -k` / `modprpw -e` |
| Convert to/from trusted | `tsconvert` / `tsconvert -r` |
| System-wide policy | edit `/etc/default/security` |

## Related Articles

- [HP-UX Boot Process (PA-RISC and Integrity)](articles/hpux-boot-process.md) — recovering a lost root password by booting single-user
