# AIX Users and Groups Cheatsheet

Command reference for managing users and groups on IBM AIX — creating/changing/removing accounts (`mkuser`/`chuser`/`rmuser`), groups (`mkgroup`/`chgroup`/`rmgroup`), passwords, login controls, and the security database files. AIX stores account data across several ODM-aware files under `/etc/security`, and every command supports a `-R` registry flag to target local files vs LDAP.

> User/group commands require `root`. Use the high-level `mkuser`/`chuser`/`lsuser` tools rather than hand-editing `/etc/passwd` — they keep the `/etc/security/*` files consistent. For LDAP-backed accounts, see the [AIX LDAP Cheatsheet](articles/aix-ldap-cheatsheet.md); for the menus, `smitty user` in the [AIX SMIT Cheatsheet](articles/aix-smit-cheatsheet.md).

## Key Files

| File | Purpose |
|------|---------|
| `/etc/passwd` | User names, UIDs, home dirs, shells (no hash). In the password field, `*` = invalid / not set, `!` = the user has a password (stored in `/etc/security/passwd`) |
| `/etc/security/passwd` | Password hashes and password flags |
| `/etc/security/user` | Per-user and default user attributes (login, expiry, limits) |
| `/etc/security/limits` | Per-user resource limits (ulimits) |
| `/etc/security/login.cfg` | Login and port/terminal policies |
| `/etc/security/.profile` | Default `.profile` copied into new users' home directories |
| `/etc/group` | Group names, GIDs, members |
| `/etc/security/group` | Group attributes (admin, adms) |
| `/etc/security/environ` | Per-user environment attributes |
| `/usr/lib/security/mkuser.default` | Default attributes applied to newly created users |

### /etc/security/passwd stanza fields

| Field | Meaning |
|-------|---------|
| `password` | The encrypted password; a lone `*` means the account is **locked** until a password is set |
| `lastupdate` | Seconds since the epoch (1 Jan 1970) when the password was last changed |
| `flags` | Password-change restrictions (see below) |

Password `flags`:

| Flag | Effect |
|------|--------|
| `ADMIN` | Only `root` can change the user's password |
| `ADMCHG` | The user is prompted to change the password at next login/su |
| `NOCHECK` | Additional restrictions in `/etc/security/user` are ignored |

## Users

### Create

```sh
mkuser jdoe                                  # create with system defaults
mkuser id=5001 pgrp=staff groups=staff,dba \
  home=/home/jdoe shell=/usr/bin/ksh gecos="John Doe" jdoe
mkuser -R LDAP jdoe                          # create in the LDAP registry

# Create an administrative user (only root/security can manage it)
mkuser -a admin=true jadmin
```

### Modify

```sh
chuser gecos="John Q Doe" jdoe               # change one attribute
chuser home=/home2/jdoe shell=/usr/bin/bash jdoe
chuser groups=staff,dba,web jdoe             # set secondary groups
chuser login=false jdoe                      # disable interactive login
chuser account_locked=true jdoe             # lock the account
chuser account_locked=false jdoe            # unlock
chuser expires=1231120026 jdoe               # account expiry MMDDhhmmyy (0 = never)
chuser core=1048576 jdoe                      # set the core-file ulimit
chuser histsize=0 jdoe                        # clear password history

# Interactive edit helpers
chsh jdoe                                    # change a user's login shell
chfn jdoe                                    # change GECOS (finger) info
passwd -f jdoe                               # change the finger information
```

A fully specified manual create (e.g. for a Kerberos account):

```sh
mkuser id=7117 pgrp=sap home=/home/paul gecos=Paul shell=/usr/bin/ksh \
  maxexpired=0 maxage=0 registry=KRB5Afiles SYSTEM=KRB5Afiles paul
```

### Query

```sh
lsuser jdoe                                  # all attributes of a user
lsuser ALL                                   # all users
lsuser -a ALL                                # all users (attribute form)
lsuser -a id home groups jdoe                # selected attributes
lsuser -a id home groups ALL                 # selected attributes for everyone
lsuser -f jdoe                               # attributes in stanza format
lsuser -c ALL                                # colon-separated (parseable) output
lsuser -c jdoe,asmith                        # compare two users
lsuser -c -a shell home jdoe,asmith          # compare specific fields
lsuser -R LDAP jdoe                          # from the LDAP registry
id jdoe                                       # UID/GID/groups

# Enumerate IDs known to the system
dispuid                                      # all valid user IDs
dispgid                                      # all valid group IDs
logins -p                                    # accounts with no password
logins -xl jdoe                              # extended info incl. last password change
```

### Remove

```sh
rmuser jdoe                                  # remove the account (keeps files/home)
rmuser -p jdoe                               # also remove security attributes
rmuser -R LDAP jdoe                          # remove from the LDAP registry
```

> `rmuser` does **not** delete the home directory — remove it separately (`rm -rf /home/jdoe`) if desired.

## Passwords

```sh
passwd jdoe                                  # set/change a user's password
passwd -R LDAP jdoe                          # in the LDAP registry

# pwdadm — administer password flags
pwdadm jdoe                                  # set a password (sets ADMCHG)
pwdadm -q jdoe                               # query password status
pwdadm -f ADMCHG jdoe                        # force a change at next login
pwdadm -f ADMIN jdoe                         # only root may change this password
pwdadm -c jdoe                               # clear the ADMCHG flag after changing the password
chuser  ADMCHG=true jdoe                      # equivalent of -f ADMCHG via attribute

# Set a password non-interactively
echo "jdoe:mypassword" | /bin/chpasswd -c    # -c = encrypt/apply

# Password aging / policy (per-user, via chuser)
chuser maxage=13 minage=1 histsize=8 minlen=8 maxexpired=2 jdoe
chuser maxage=0 jdoe                          # never expire (unexpire) the password

# Unexpire a password without setting a new one (bump lastupdate)
chsec -f /etc/security/passwd -s jdoe -a "lastupdate=$(perl -e 'print time')"
```

| Attribute | Meaning |
|-----------|---------|
| `maxage` | Max password age (weeks) before it must change |
| `minage` | Min weeks before a password can change again |
| `minlen` | Minimum password length |
| `histsize` | Number of previous passwords remembered |
| `maxexpired` | Weeks after `maxage` that a user can still change an expired password |
| `pwdwarntime` | Days before expiry to start warning |

System-wide defaults live in the `default:` stanza of `/etc/security/user`.

## Groups

```sh
mkgroup appteam                              # create a group
mkgroup id=6001 appteam                      # with a specific GID
mkgroup -a atcadmin                          # administrative group
mkgroup adms=jadmin appteam                  # user as group administrator at create
mkgroup -R LDAP appteam                      # in the LDAP registry

chgroup id=204 users=jdoe,asmith,bwong appteam   # change GID and set members
chgroup adms=jadmin appteam                  # set group administrators

# chgrpmem — manage membership without rewriting the whole list
chgrpmem appteam                             # list a group's members
chgrpmem -m + asmith appteam                 # add a member
chgrpmem -m - asmith appteam                 # remove a member

lsgroup appteam                              # group attributes
lsgroup ALL                                  # all groups
lsgroup -f appteam                           # attributes in stanza format
lsgroup -a users id appteam                  # selected attributes
lsgroup -a id ALL                            # one attribute for all groups
lsgroup -c appteam,atcadmin                  # parseable, compare groups
lsgroup -c -a id appteam,atcadmin            # parseable, specific field

rmgroup appteam                              # remove a group
```

## Login Controls and Limits

```sh
# Lock / unlock (see also account_locked above)
chuser login=false jdoe                       # deny interactive login
chuser rlogin=false jdoe                      # deny remote login (telnet/rlogin/ssh via login)
chuser su=false jdoe                          # disallow su to this account
chuser sugroups=staff jdoe                    # only these groups may su to it

# Resource limits (edit /etc/security/limits or via chuser)
chuser fsize=-1 data=-1 stack=-1 nofiles=10000 jdoe   # -1 = unlimited

# Failed-login lockout
chuser loginretries=5 jdoe                    # lock after N failed attempts
chsec -f /etc/security/lastlog -s jdoe -a "unsuccessful_login_count=0"   # reset the counter
```

### chsec — edit security stanzas safely

`chsec` is the supported way to change values in the `/etc/security/*` stanza files.

```sh
chsec -f /etc/security/user -s jdoe -a login=true
chsec -f /etc/security/login.cfg -s usw -a maxlogins=32
chsec -f /etc/security/user -s default -a maxage=13     # change a default
```

## Reporting and Auditing

```sh
who                                          # who is logged in
w                                            # who + what they're doing
last                                         # login history
last jdoe                                    # a user's login history
lsuser -a unsuccessful_login_count ALL       # failed-login counts
lssec -f /etc/security/user -s jdoe -a account_locked   # read one attribute
usrck -n ALL                                 # verify user definitions are consistent
grpck -n ALL                                 # verify group definitions
pwdck -n ALL                                 # verify password consistency
```

`usrck`/`grpck`/`pwdck` with `-y` attempt to fix problems; `-n` reports only; `-t` prompts.

## Roles (RBAC)

AIX supports Role-Based Access Control so non-root users can perform privileged tasks.

```sh
lsrole ALL                                   # list roles
lsrole -a authorizations RoleName            # a role's authorizations
mkrole authorizations=aix.fs.manage.mount NewRole
chuser roles=NewRole default_roles=NewRole jdoe   # assign a role to a user
swrole NewRole                               # switch to (activate) a role in a session
setkst                                       # update the kernel security tables after RBAC changes
```

## Registry, SYSTEM, and Name Limits

Every AIX user has a **`registry`** attribute (where the identity is administered) and a **`SYSTEM`** attribute (which authentication methods are used and how they combine); groups have only a `registry` value. Common values: `files` (local), `LDAP`, `KRB5files`/`KRB5Afiles` (Kerberos). To deny remote login, set `rlogin=false` in `/etc/security/user`.

```sh
echo $AUTHSTATE                              # which auth method was used for this login
```

**Name-length limits:** AIX 5.2 and below cap user/group names at **8 characters**; AIX 5.3+ allows up to **255**.

```sh
getconf LOGIN_NAME_MAX                        # current max name length from the kernel
chdev -l sys0 -a max_logname=30              # raise the limit (takes effect after reboot)
```

## Kerberos (NAS) Integration

AIX ships IBM Network Authentication Service (NAS) for Kerberos-based authentication.

```sh
# Configure a Kerberos server (creates /etc/krb5/krb5.conf, kdc.conf, kadm5.acl)
mkkrb5srv -r <realm> -s <servername> -d <domain>

# Configure a NAS Kerberos client
mkkrb5clnt -r <realm> -c <KDC_server> -s <kerberos_server> -d <domain> \
  -a admin/admin -A -i files -K -T

# Configure the client against a Microsoft AD KDC
config.krb5 -C -r <realm> -d <domain> -c <KDC_server> -s <kerberos_server>
unconfig.krb5                                 # unconfigure client/server

# Kerberos-backed AIX accounts
mkuser -R registry=KRB5files SYSTEM="KRB5files" <username>
passwd -R KRB5files <username>
```

## Quick Reference

| Task | Command |
|------|---------|
| Create a user | `mkuser <name>` |
| Create with attrs | `mkuser id=5001 pgrp=staff home=/home/x <name>` |
| Change an attribute | `chuser <attr>=<val> <name>` |
| Lock / unlock | `chuser account_locked=true\|false <name>` |
| List a user | `lsuser <name>` / `lsuser ALL` |
| Remove a user | `rmuser -p <name>` |
| Set a password | `passwd <name>` |
| Force change at login | `pwdadm -f ADMCHG <name>` |
| Create a group | `mkgroup <name>` |
| Set group members | `chgroup users=a,b,c <name>` |
| Edit a security stanza | `chsec -f <file> -s <stanza> -a <attr>=<val>` |
| Verify definitions | `usrck -n ALL` / `grpck -n ALL` / `pwdck -n ALL` |

## Related

- [AIX LDAP Cheatsheet](articles/aix-ldap-cheatsheet.md) — the `-R LDAP` registry and directory-backed accounts
- [AIX SMIT Cheatsheet](articles/aix-smit-cheatsheet.md) — `smitty user` / `smitty group` menus
- [AIX Filesystems Cheatsheet](articles/aix-filesystems-cheatsheet.md) — file ownership and permissions
