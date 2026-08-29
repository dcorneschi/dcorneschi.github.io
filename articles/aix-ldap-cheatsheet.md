# AIX LDAP Cheatsheet

Command reference for LDAP-based authentication on IBM AIX — configuring the LDAP client with `mksecldap`, managing the client daemon (`secldapclntd`), directing users/groups to the LDAP registry, and the underlying config and diagnostic tools. AIX uses IBM Security Directory Server (formerly Tivoli Directory Server / IDS) as its native LDAP server, and the client integrates through the `LDAP` loadable authentication module.

> Most commands require `root`. The client configuration lives in `/etc/security/ldap/ldap.cfg`, and the daemon `secldapclntd` caches directory lookups. For local user administration context, see the [AIX SMIT Cheatsheet](articles/aix-smit-cheatsheet.md) (`smitty ldap`).

## Key Files

| File | Purpose |
|------|---------|
| `/etc/security/ldap/ldap.cfg` | LDAP client configuration (server, bind DN, base DNs, cache/TLS settings) |
| `/etc/security/ldap/*.map` | Attribute maps between AIX attributes and LDAP schema |
| `/usr/lib/security/methods.cfg` | Defines the `LDAP` loadable auth module |
| `/etc/security/user` | Per-user/default `SYSTEM` and `registry` attributes |
| `/etc/group` | Local group definitions (LDAP groups resolve via the registry) |
| `/var/adm/ras/secldapclntd.log` | Client daemon log |

## Configure the LDAP Client (mksecldap)

`mksecldap` sets up the client (or server). On the client side it writes `ldap.cfg`, defines the `LDAP` module in `methods.cfg`, and starts `secldapclntd`.

```sh
# Configure this host as an LDAP client
mksecldap -c -h ldap1.example.com \
  -a cn=admin,dc=example,dc=com -p 'adminpassword' \
  -d ou=aix,dc=example,dc=com

# Multiple/failover LDAP servers (comma-separated after -h)
mksecldap -c -h "ldap1.example.com,ldap2.example.com" \
  -a cn=admin,dc=example,dc=com -p 'adminpassword' \
  -d ou=aix,dc=example,dc=com

# Use SSL/TLS to the directory (requires a key database)
mksecldap -c -h ldap1.example.com \
  -a cn=admin,dc=example,dc=com -p 'adminpassword' \
  -d ou=aix,dc=example,dc=com \
  -k /etc/security/ldap/key.kdb -w 'keypassword'
```

| Flag | Meaning |
|------|---------|
| `-c` | Configure as a **client** (`-s` configures a server) |
| `-h` | LDAP server host(s) — comma-separated for failover |
| `-a` | Bind DN (admin) used to read the directory |
| `-p` | Bind password |
| `-d` | Base DN (suffix) under which AIX user/group data lives |
| `-k` / `-w` | SSL key database and its password (for TLS) |
| `-u` | Restrict which users are allowed on this client (`-u ALL` for all) |

> To reconfigure or undo: rerun `mksecldap -c ...`, or remove the client with `rmsecldap` / by editing `ldap.cfg` and stopping the daemon.

## Client Daemon (secldapclntd)

`secldapclntd` is the caching client daemon. The management helpers live in `/usr/sbin` (they're normally on `PATH`, so the bare names work too).

```sh
# Show daemon status: connected server, port, caching status, entries cached
/usr/sbin/ls-secldapclntd

# Lifecycle
/usr/sbin/start-secldapclntd     # start (same as running secldapclntd directly)
/usr/sbin/stop-secldapclntd      # stop
/usr/sbin/restart-secldapclntd   # restart (acts like start if not running)

# Clear the client cache (force fresh lookups)
/usr/sbin/flush-secldapclntd
```

`ls-secldapclntd` is the quickest way to confirm the client is actually talking to a server and to see cache effectiveness.

### Query the directory via the client (lsldap)

`lsldap` lists entries from the configured LDAP server using the client's own config — handy to confirm the client can read users/groups without crafting a full `ldapsearch`.

```sh
lsldap                 # list naming contexts / entries
lsldap -a passwd       # all user (posixAccount) entries
lsldap passwd jdoe     # a specific user
lsldap -a group        # all groups
lsldap hosts           # host entries
```

### Check connectivity to the LDAP server

```sh
# Verify the TLS listener on the secure LDAP port (636) responds
openssl s_client -host <ldap_server> -port 636

# Then confirm the client daemon and its view of the server
ls-secldapclntd
lsldap
restart-secldapclntd
flush-secldapclntd
```

## Directing Users and Groups to LDAP

AIX decides where a user is defined via the `SYSTEM` and `registry` attributes. `LDAP` means "authenticate against LDAP"; `files` means local.

```sh
# Create a user stored in the LDAP registry
mkuser -R LDAP id=5001 pgrp=staff home=/home/jdoe jdoe

# Move an existing local user to LDAP (change registry)
chuser -R LDAP registry=LDAP SYSTEM=LDAP jdoe

# Make a user authenticate against LDAP but fall back to files
chuser SYSTEM="LDAP or compat" registry=LDAP jdoe

# List / change / remove in the LDAP registry
lsuser -R LDAP jdoe
lsuser -R LDAP ALL
chuser -R LDAP gecos="John Doe" jdoe
rmuser -R LDAP jdoe

# Groups work the same way
mkgroup -R LDAP id=6001 appteam
lsgroup -R LDAP appteam
rmgroup -R LDAP appteam

# Set/reset a password in the LDAP registry
passwd -R LDAP jdoe
```

> `-R LDAP` targets the LDAP registry for that one command; `-R files` targets the local files. Without `-R`, AIX uses the user's configured registry.

## Verify Authentication

```sh
# Confirm which registry/module resolves a user
lsuser -a registry SYSTEM jdoe

# Test that credentials actually authenticate (prompts for password)
# Returns 0 on success
/usr/bin/lssec -f /etc/security/user -s jdoe -a SYSTEM
```

## IBM Directory Server (server side)

When the host is also the directory server (IBM Security Directory Server / IDS), these tools manage the instance and its data.

```sh
# Start / stop the directory server instance (idsinst = instance owner)
ibmslapd -I idsinst           # start
ibmdirctl -D cn=admin -w passwd stop   # stop via the control daemon

# Import / export directory data (LDIF)
idsldif2db -i /tmp/data.ldif  # bulk load
idsdb2ldif -o /tmp/dump.ldif  # export

# Generic LDAP client queries (from any client with ldap.client installed)
ldapsearch -h ldap1.example.com -D cn=admin,dc=example,dc=com -w passwd \
  -b ou=aix,dc=example,dc=com "(objectclass=posixAccount)" uid

ldapadd    -h ldap1.example.com -D cn=admin,dc=example,dc=com -w passwd -f new.ldif
ldapmodify -h ldap1.example.com -D cn=admin,dc=example,dc=com -w passwd -f change.ldif
ldapdelete -h ldap1.example.com -D cn=admin,dc=example,dc=com -w passwd "uid=jdoe,ou=aix,dc=example,dc=com"
```

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Logins hang or fail | `ls-secldapclntd` — is the daemon up and connected to a server? |
| Stale user/group data | `flush-secldapclntd` to clear the cache |
| Users resolve locally instead of LDAP | `lsuser -a registry SYSTEM <user>`; fix with `chuser -R LDAP registry=LDAP SYSTEM=LDAP <user>` |
| TLS/bind errors | Verify `-k`/`-w` key DB in `ldap.cfg`; check `/var/adm/ras/secldapclntd.log` |
| Can't reach the directory | `ldapsearch` from the client to isolate network vs config |

```sh
# Watch the client daemon log
tail -f /var/adm/ras/secldapclntd.log

# Review the active client config
cat /etc/security/ldap/ldap.cfg
```

## Quick Reference

| Task | Command |
|------|---------|
| Configure LDAP client | `mksecldap -c -h <server> -a <bindDN> -p <pw> -d <baseDN>` |
| Daemon status | `ls-secldapclntd` |
| Restart daemon | `restart-secldapclntd` |
| Flush cache | `flush-secldapclntd` |
| Query directory via client | `lsldap -a passwd` |
| Test TLS to server | `openssl s_client -host <server> -port 636` |
| Create LDAP user | `mkuser -R LDAP <name>` |
| Move user to LDAP | `chuser -R LDAP registry=LDAP SYSTEM=LDAP <name>` |
| List LDAP users | `lsuser -R LDAP ALL` |
| Set LDAP password | `passwd -R LDAP <name>` |
| Query the directory | `ldapsearch -h <server> -D <bindDN> -w <pw> -b <baseDN> "<filter>"` |

## Related

- [AIX SMIT Cheatsheet](articles/aix-smit-cheatsheet.md) — `smitty ldap` menus for the same tasks
- [AIX Filesystems Cheatsheet](articles/aix-filesystems-cheatsheet.md) — file ownership and permissions context
- [AIX Backup and Recovery Cheatsheet](articles/aix-backup-recovery-cheatsheet.md) — mksysb and system recovery
