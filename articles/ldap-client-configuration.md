# Configuring LDAP Client Authentication

This guide covers configuring Linux systems to authenticate users against an LDAP directory (OpenLDAP, Active Directory, FreeIPA). It covers RHEL 5–10 and Ubuntu 22.04/24.04, including SSSD and legacy configurations.

## Overview

LDAP client authentication allows centralizing user accounts in a directory server. The client system queries the LDAP server for user information (via NSS) and verifies credentials (via PAM).

| Component | Purpose |
|-----------|---------|
| NSS (Name Service Switch) | Resolves user/group information from LDAP |
| PAM (Pluggable Authentication Modules) | Handles authentication against LDAP |
| SSSD (System Security Services Daemon) | Modern caching daemon for identity and auth (recommended) |
| nslcd | Lightweight NSS/PAM LDAP daemon (legacy alternative) |

### LDAP Ports

| Protocol | Port |
|----------|------|
| Conventional LDAP | 389 |
| LDAP over SSL | 636 |

## RHEL 5

### Install the Required Packages

```sh
yum install openldap nss_ldap nscd
```

### Configure with authconfig

```sh
authconfig --enableldap --enableldapauth --ldapserver="ldap://192.168.50.10" --ldapbasedn="dc=homelab,dc=local" --enablemkhomedir --update
authconfig --enableldaptls --update
```

Or use the text-based UI:

```sh
authconfig-tui
```

## RHEL 6

RHEL 6 uses `authconfig` and `nss-pam-ldapd` (nslcd) or direct `pam_ldap`/`nss_ldap`.

Client side configuration can be done either using:

- nslcd - `/etc/nslcd.conf` & `/etc/pam_ldap.conf` file (authconfig updates both automatically)
- sssd - `/etc/sssd/sssd.conf` (does not support authentication over an unencrypted channel)

In both cases, make sure that the LDAP server name is resolvable (in `/etc/hosts`, DNS or other) on the client, otherwise authentication would fail even if the LDAP search succeeds.

### Configuring LDAP Client Using SSSD (Recommended)

```bash
yum install -y sssd sssd-client
```

```bash
authconfig --enablesssd --enablesssdauth --ldapserver="ldap://ldap.example.com" --ldapbasedn="dc=example,dc=com" --enableldaptls --update
```

> **Note:** SSSD does not support authentication over an unencrypted channel. If LDAP authentication is enabled, either TLS/SSL or LDAPS is required. If the LDAP server is used only as an identity provider, an encrypted channel is not needed.

### Configuring LDAP Client Using nslcd

Install packages:

```bash
yum install -y openldap-clients nss-pam-ldapd pam_ldap authconfig
```

### Configure with authconfig

Authconfig will try to use SSSD by default. To configure nslcd, enable the FORCELEGACY option:

```bash
authconfig --enableldap --enableldapauth --ldapserver="ldap://192.168.50.10" --ldapbasedn="dc=homelab,dc=local" --enableforcelegacy --enablemkhomedir --update
```

Or enable FORCELEGACY by editing `/etc/sysconfig/authconfig` directly:

```sh
FORCELEGACY=yes
```

Then apply:

```bash
authconfig --enableforcelegacy --update
```

Using TLS for LDAP authentication is not mandatory for nslcd. Add `--enableldaptls` if you wish to enable TLS:

```bash
authconfig --enableldaptls --update
```

- `--update` — Only the files affected by the configuration changes are overwritten
- `--updateall` — All configuration files are written

For LDAPS (TLS):

```bash
authconfig \
  --enableldap \
  --enableldapauth \
  --ldapserver=ldaps://ldap.example.com \
  --ldapbasedn="dc=example,dc=com" \
  --enableldaptls \
  --ldaploadcacert=http://ca.example.com/ca.crt \
  --enablemkhomedir \
  --update
```

### PAM Configuration

```sh
vi /etc/pam.d/system-auth
```

```sh
auth  sufficient  pam_ldap.so use_first_pass
session  optional  pam_mkhomedir.so skel=/etc/skel umask=0022
```

```sh
/etc/init.d/nslcd restart
```

### Key Configuration Files

| File | Purpose |
|------|---------|
| `/etc/nslcd.conf` | nslcd daemon configuration |
| `/etc/openldap/ldap.conf` | OpenLDAP client defaults |
| `/etc/pam_ldap.conf` | PAM LDAP configuration |
| `/etc/nsswitch.conf` | Name service switch order |

### Manual /etc/nslcd.conf

```bash
uid nslcd
gid ldap
uri ldap://ldap.example.com
base dc=example,dc=com
ssl start_tls
tls_cacertfile /etc/openldap/certs/ca-bundle.crt
```

### Manual /etc/nsswitch.conf

```bash
passwd:     files ldap
shadow:     files ldap
group:      files ldap
```

### Enable and Start Services

```bash
chkconfig nslcd on
service nslcd start

# Enable home directory creation on login
authconfig --enablemkhomedir --update
```

### Verify

```bash
getent passwd ldapuser
id ldapuser
```

### Configuring a Netgroup Backend

> **Note:** SSSD version 1.2 shipped with RHEL 6.0 does not have support for netgroups. Use `pam_ldap` and `nss-pam-ldapd` (which replaces `nss_ldap` in RHEL 6) to get netgroups working. In RHEL 6, `nss_ldap` is split into `nss-pam-ldapd` (lookup service) and `pam_ldap` (authentication module).

To disable SSSD for lookup and use nss-pam-ldapd instead, edit `/etc/nsswitch.conf` and change occurrences of `sss` to `ldap`.

#### /etc/nslcd.conf (with netgroup support)

`/etc/nslcd.conf` is used for lookup:

```sh
uid nslcd
gid ldap
timelimit 120
bind_timelimit 120
idle_timelimit 3600
filter passwd objectclass=posixAccount
uri ldap://ldap.example.com
base dc=example,dc=com
base   netgroup ou=Netgroup,dc=example,dc=com
nss_initgroups_ignoreusers root,ldap,named,avahi,haldaemon,dbus,radvd,tomcat,radiusd,news,mailman
scope one
ssl no
tls_cacertdir /etc/openldap/cacerts
```

#### /etc/pam_ldap.conf

`/etc/pam_ldap.conf` is used by the `pam_ldap` module for authentication:

```sh
base dc=example,dc=com
uri ldap://ldap.example.com
ssl no
tls_cacertdir /etc/openldap/cacerts
pam_password md5
```

#### Method 1: NSS compat mode

```sh
vi /etc/nsswitch.conf
```

```sh
passwd:     	compat
passwd_compat:  ldap
shadow:     	files ldap
group:      	files ldap
netgroup:   	ldap
```

```sh
vipw
```

```sh
+@family
```

```sh
getent netgroup family
```

#### Method 2: pam_access with LDAP netgroups

Edit `/etc/nsswitch.conf`:

```sh
vi /etc/nsswitch.conf
```

```sh
netgroup: files ldap
```

Edit `/etc/security/access.conf`:

```sh
vi /etc/security/access.conf
```

```sh
+:root:LOCAL
+:@family:ALL
-:ALL:ALL
```

Enable `pam_access.so` module:

```sh
authconfig --enablepamaccess --update
```

#### Verify Netgroups

```sh
getent netgroup QAUsers
QAUsers               ( , idmuser1, example.com) ( , idmuser2, example.com)
```

## RHEL 7

RHEL 7 introduces SSSD as the recommended approach, though `authconfig` is still available.

### Install Packages (SSSD)

```bash
yum install -y sssd sssd-clients openldap-clients oddjob-mkhomedir
```

### Configure with authconfig (SSSD)

```bash
authconfig --enableldap --enableldapauth --ldapserver="ldap://192.168.50.10" --ldapbasedn="dc=homelab,dc=local" --enablemkhomedir --enablesssd --enablesssdauth --update
```

With TLS and CA certificate:

```bash
authconfig --enableldap --ldapserver="ldap://192.168.50.10" --enableldapauth --ldapbasedn="dc=homelab,dc=local" --enableldaptls --ldaploadcacert=/path/to/slapd-cert.crt --update
```

### Alternative: nslcd (Legacy)

```bash
yum install -y openldap openldap-clients nss-pam-ldapd
```

```bash
authconfig --enableldap --enableldapauth --ldapserver="ldap://192.168.50.10" --ldapbasedn="dc=homelab,dc=local" --enablemkhomedir --enableforcelegacy --update
```

- `--enableldaptls` — for SSL/TLS
- `--ldaploadcacert` — downloaded through authconfig command
- `--enableforcelegacy` — sssd daemon stopped and nslcd daemon started

### Configuration Files and Directories

| Path | Purpose |
|------|---------|
| `/etc/sssd/sssd.conf` | SSSD configuration file |
| `/etc/openldap/cacerts` | SSL certificates |
| `/etc/sysconfig/authconfig` | Authconfig settings |

### Manual /etc/sssd/sssd.conf

```ini
[sssd]
config_file_version = 2
services = nss, pam
domains = example.com

[nss]
filter_groups = root
filter_users = root

[pam]

[domain/example.com]
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap

ldap_uri = ldap://ldap.example.com
ldap_search_base = dc=example,dc=com
ldap_id_use_start_tls = True
ldap_tls_cacert = /etc/openldap/certs/ca-bundle.crt

# User/group schema mapping
ldap_user_object_class = posixAccount
ldap_user_name = uid
ldap_group_object_class = posixGroup
ldap_group_name = cn

# Cache settings
cache_credentials = True
entry_cache_timeout = 600

# Access control
access_provider = ldap
ldap_access_filter = (memberOf=cn=linux-users,ou=Groups,dc=example,dc=com)
```

### Set Permissions and Start

```bash
chmod 600 /etc/sssd/sssd.conf
systemctl enable sssd
systemctl start sssd
systemctl enable oddjobd
systemctl start oddjobd
```

### Enable mkhomedir

```bash
authconfig --enablemkhomedir --update
```

### Verify

```bash
getent passwd ldapuser
id ldapuser
su - ldapuser
```

## RHEL 8

RHEL 8 replaces `authconfig` with `authselect`. SSSD is the standard.

Authselect command when used to create an SSSD profile, will basically modify these files. This command will succeed only if you have removed the files below or use `--force` parameter. Default directory is `/etc/authselect`.

- `/etc/pam.d/system-auth`
- `/etc/pam.d/password-auth`
- `/etc/pam.d/fingerprint-auth`
- `/etc/pam.d/smartcard-auth`
- `/etc/pam.d/postlogin`
- `/etc/nsswitch.conf`

### Authselect Features

| Feature Name | Description |
|---|---|
| with-faillock | Lock the account after too many authentication failures. |
| with-mkhomedir | Create home directory on user's first log in. |
| with-ecryptfs | Enable automatic per-user ecryptfs. |
| with-smartcard | Authenticate smart cards through SSSD. |
| with-smartcard-lock-on-removal | Lock the screen when the smart card is removed. Requires that with-smartcard is also enabled. |
| with-smartcard-required | Only smart card authentication is operative; others, including password, are disabled. Requires that with-smartcard is also enabled. |
| with-fingerprint | Authenticate through fingerprint reader. |
| with-silent-lastlog | Disable generation of pam_lastlog messages during login. |
| with-sudo | Enable sudo to use SSSD for rules besides /etc/sudoers. |
| with-pamaccess | Refer to /etc/access.conf for account authorization. |
| without-nullock | Do not add the nullock parameter to pam_unix. |

### Authselect Commands

- Display profile: `authselect current`
- List available profiles: `authselect list`
- List all features available in given profile: `authselect list-features sssd`

### Install Packages

```bash
dnf install -y sssd sssd-tools oddjob-mkhomedir openldap-clients openssl-perl
systemctl enable --now oddjobd
```

### Edit /etc/openldap/ldap.conf

```sh
vi /etc/openldap/ldap.conf
```

```sh
BASE    dc=homelab,dc=local
URI     ldap://192.168.50.30
TLS_CACERTDIR  /etc/openldap/certs
```

### /etc/sssd/sssd.conf

For communication over **StartTLS** (port 389, upgrade to TLS):

```ini
[sssd]
config_file_version = 2
services = nss, pam, autofs
domains = ldap

[nss]
homedir_substring = /home

[pam]

[domain/ldap]
id_provider = ldap
autofs_provider = ldap
auth_provider = ldap
chpass_provider = ldap
ldap_uri = ldap://192.168.50.30
ldap_search_base = dc=homelab,dc=local
ldap_id_use_start_tls = True
ldap_tls_cacertdir = /etc/openldap/certs
cache_credentials = True
ldap_tls_reqcert = allow
```

For communication over **SSL** (LDAPS, port 636):

```ini
[domain/ldap]
id_provider = ldap
autofs_provider = ldap
auth_provider = ldap
chpass_provider = ldap
ldap_uri = ldaps://192.168.50.30:636
ldap_chpass_uri = ldaps://192.168.50.30:636
ldap_search_base = dc=homelab,dc=local
ldap_id_use_start_tls = False
ldap_tls_cacertdir = /etc/openldap/certs
cache_credentials = True
ldap_tls_reqcert = demand
entry_cache_timeout = 600
ldap_network_timeout = 3
ldap_connection_expire_timeout = 60
```

If the LDAP server is not allowing anonymous bind, add the bind details under the domain section:

```ini
ldap_default_bind_dn = uid=binduser,ou=users,dc=homelab,dc=local
ldap_default_authtok_type = password
ldap_default_authtok = <YourPassword>
```

### Configure with authselect

```bash
chmod 600 /etc/sssd/sssd.conf
chown root:root /etc/sssd/sssd.conf
authselect select sssd with-mkhomedir --force
```

### Configure Certificate

Place CA certificate file in `/etc/openldap/certs/`, then rehash:

```sh
openssl rehash /etc/openldap/certs
```

There are 2 ways to obtain the certificate:

1. Copy the certificate from LDAP server to `/etc/openldap/certs/`
2. Extract the certificate with openssl:

```sh
openssl s_client -connect <ldap_server>:636 -showcerts < /dev/null | openssl x509 -text
```

Save the certificate:

```sh
vi /etc/openldap/certs/ldap_server.crt
```

```
-----BEGIN CERTIFICATE-----
MIIDkDCCAngCCQCwM9rQBpwrhDANBgkqhkiG9w0BAQ0FADCBiTELMAkGA1UEBhMC
Uk8xDjAMBgNVBAgMBVRpbWlzMRIwEAYDVQQHDAlUaW1pc29hcmExDTALBgNVBAoM
BEhvbWUxDTALBgNVBAsMBEhvbWUxGTAXBgNVBAMMEHd3dy5jb3JuZXNjaGkucm8x
........
-----END CERTIFICATE-----
```

Rehash after placing the certificate:

```sh
openssl rehash /etc/openldap/certs
```

### Start SSSD

```bash
systemctl enable --now sssd
```

### Verify

```bash
authselect current
getent passwd ldapuser
id ldapuser
sssctl domain-list
sssctl user-checks ldapuser
```

## RHEL 9

Same approach as RHEL 8 with `authselect` and SSSD.

### Install Packages

```bash
dnf install -y openldap-clients sssd sssd-ldap oddjob-mkhomedir openssl-perl
```

### Configure with authselect

```bash
authselect select sssd with-mkhomedir --force
systemctl enable --now oddjobd
```

### /etc/sssd/sssd.conf

```ini
[sssd]
config_file_version = 2
services = nss, pam
domains = example.com

[nss]
filter_groups = root
filter_users = root

[pam]

[domain/example.com]
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap

ldap_uri = ldaps://ldap.example.com
ldap_search_base = dc=example,dc=com
ldap_tls_reqcert = demand
ldap_tls_cacert = /etc/pki/tls/certs/ca-bundle.crt

ldap_user_object_class = posixAccount
ldap_user_name = uid
ldap_group_object_class = posixGroup
ldap_group_name = cn

cache_credentials = True
entry_cache_timeout = 600
enumerate = False
```

### Set Permissions and Start

```bash
chmod 600 /etc/sssd/sssd.conf
systemctl enable --now sssd
```

### Verify

```bash
getent passwd ldapuser
id ldapuser
sssctl domain-list
sssctl user-checks ldapuser
```

## RHEL 10

RHEL 10 continues the SSSD + authselect pattern. The crypto-policies may affect TLS requirements.

### Install Packages

```bash
dnf install -y sssd sssd-ldap openldap-clients oddjob-mkhomedir
```

### Configure with authselect

```bash
authselect select sssd with-mkhomedir --force
systemctl enable --now oddjobd
```

### /etc/sssd/sssd.conf

```ini
[sssd]
config_file_version = 2
services = nss, pam
domains = example.com

[nss]
filter_groups = root
filter_users = root

[pam]

[domain/example.com]
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap

ldap_uri = ldaps://ldap.example.com
ldap_search_base = dc=example,dc=com
ldap_tls_reqcert = demand
ldap_tls_cacert = /etc/pki/tls/certs/ca-bundle.crt

ldap_user_object_class = posixAccount
ldap_user_name = uid
ldap_group_object_class = posixGroup
ldap_group_name = cn

cache_credentials = True
entry_cache_timeout = 600
enumerate = False

# RHEL 10: min TLS version (crypto-policies may enforce this)
ldap_tls_cipher_suite = DEFAULT:!NULL
```

### Set Permissions and Start

```bash
chmod 600 /etc/sssd/sssd.conf
systemctl enable --now sssd
```

### Crypto Policies Note

RHEL 10 enforces stricter crypto policies by default. If connecting to legacy LDAP servers:

```bash
# Check current policy
update-crypto-policies --show

# Temporarily allow legacy connections (not recommended for production)
update-crypto-policies --set LEGACY
```

### Verify

```bash
getent passwd ldapuser
id ldapuser
sssctl domain-list
sssctl user-checks ldapuser
```

## Ubuntu 22.04

Ubuntu uses `sssd` with manual configuration or `pam-auth-update`.

### Install Packages

```bash
apt update
apt install -y sssd sssd-ldap ldap-utils libpam-sss libnss-sss
```

### /etc/sssd/sssd.conf

```ini
[sssd]
config_file_version = 2
services = nss, pam
domains = example.com

[nss]
filter_groups = root
filter_users = root

[pam]

[domain/example.com]
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap

ldap_uri = ldaps://ldap.example.com
ldap_search_base = dc=example,dc=com
ldap_tls_reqcert = demand
ldap_tls_cacert = /etc/ssl/certs/ca-certificates.crt

ldap_user_object_class = posixAccount
ldap_user_name = uid
ldap_group_object_class = posixGroup
ldap_group_name = cn

cache_credentials = True
entry_cache_timeout = 600
enumerate = False
```

### Set Permissions and Start

```bash
chmod 600 /etc/sssd/sssd.conf
systemctl enable --now sssd
```

### Enable mkhomedir

```bash
# Method 1: pam-auth-update
pam-auth-update --enable mkhomedir

# Method 2: Manual PAM configuration
# Add to /etc/pam.d/common-session:
# session required pam_mkhomedir.so skel=/etc/skel umask=0077
```

### Configure /etc/nsswitch.conf

Verify that `sss` is listed:

```bash
passwd:         files systemd sss
group:          files systemd sss
shadow:         files sss
```

### Verify

```bash
getent passwd ldapuser
id ldapuser
pam-auth-update --list
```

## Ubuntu 24.04

Same approach as Ubuntu 22.04. SSSD is the standard.

### Install Packages

```bash
apt update
apt install -y sssd sssd-ldap ldap-utils libpam-sss libnss-sss
```

### /etc/sssd/sssd.conf

```ini
[sssd]
config_file_version = 2
services = nss, pam
domains = example.com

[nss]
filter_groups = root
filter_users = root

[pam]

[domain/example.com]
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap

ldap_uri = ldaps://ldap.example.com
ldap_search_base = dc=example,dc=com
ldap_tls_reqcert = demand
ldap_tls_cacert = /etc/ssl/certs/ca-certificates.crt

ldap_user_object_class = posixAccount
ldap_user_name = uid
ldap_group_object_class = posixGroup
ldap_group_name = cn

cache_credentials = True
entry_cache_timeout = 600
enumerate = False
```

### Set Permissions and Start

```bash
chmod 600 /etc/sssd/sssd.conf
systemctl enable --now sssd
```

### Enable mkhomedir

```bash
pam-auth-update --enable mkhomedir
```

### Configure /etc/nsswitch.conf

```bash
passwd:         files systemd sss
group:          files systemd sss
shadow:         files sss
```

### Verify

```bash
getent passwd ldapuser
id ldapuser
```

## Active Directory Integration

For authenticating against Active Directory with SSSD, the `id_provider` changes to `ad` and you join the domain with `realm`.

### Install (RHEL 8+)

```bash
dnf install -y sssd realmd adcli oddjob-mkhomedir samba-common-tools
```

### Install (Ubuntu)

```bash
apt install -y sssd realmd adcli oddjob-mkhomedir samba-common-bin
```

### Discover and Join Domain

```bash
# Discover the domain
realm discover example.com

# Join the domain
realm join --user=admin example.com

# Verify
realm list
```

### /etc/sssd/sssd.conf (Auto-generated by realm join)

```ini
[sssd]
config_file_version = 2
services = nss, pam
domains = example.com

[domain/example.com]
id_provider = ad
auth_provider = ad
access_provider = ad

ad_domain = example.com
krb5_realm = EXAMPLE.COM

realmd_tags = manages-system joined-with-adcli

cache_credentials = True
krb5_store_password_if_offline = True

# Use short names (user instead of user@example.com)
use_fully_qualified_names = False

# Default shell and home directory
fallback_homedir = /home/%u
default_shell = /bin/bash

# Access control: allow specific groups
ad_access_filter = (memberOf=CN=LinuxAdmins,OU=Groups,DC=example,DC=com)
```

### Allow/Deny Access

```bash
# Allow specific users
realm permit user1@example.com user2@example.com

# Allow specific groups
realm permit -g LinuxAdmins

# Allow all domain users
realm permit --all

# Deny all (default after join on some systems)
realm deny --all
```

## TLS/SSL Configuration

### CA Certificate Locations

| Distribution | Default CA Bundle |
|-------------|-------------------|
| RHEL 6–7 | `/etc/openldap/certs/ca-bundle.crt` |
| RHEL 8–10 | `/etc/pki/tls/certs/ca-bundle.crt` |
| Ubuntu | `/etc/ssl/certs/ca-certificates.crt` |

### Install Custom CA Certificate

**RHEL 8+:**

```bash
cp custom-ca.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust
```

**RHEL 6–7:**

```bash
cp custom-ca.crt /etc/openldap/certs/
# Or append to the bundle
cat custom-ca.crt >> /etc/openldap/certs/ca-bundle.crt
```

**Ubuntu:**

```bash
cp custom-ca.crt /usr/local/share/ca-certificates/
update-ca-certificates
```

### TLS Options in sssd.conf

```ini
# Require valid certificate (recommended)
ldap_tls_reqcert = demand

# Options: never, allow, try, demand, hard
# never  - never request or check certificates
# allow  - request cert, proceed even if not provided
# try    - request cert, proceed if not provided, abort if invalid
# demand - require valid certificate (recommended)
# hard   - same as demand

# Specify CA certificate
ldap_tls_cacert = /etc/pki/tls/certs/ca-bundle.crt

# Client certificate authentication (mutual TLS)
ldap_tls_cert = /etc/sssd/client.crt
ldap_tls_key = /etc/sssd/client.key
```

### STARTTLS vs LDAPS

| Method | URI | Port | Description |
|--------|-----|------|-------------|
| STARTTLS | `ldap://` | 389 | Upgrades plain connection to TLS |
| LDAPS | `ldaps://` | 636 | TLS from the start |

```ini
# STARTTLS (port 389, upgrade to TLS)
ldap_uri = ldap://ldap.example.com
ldap_id_use_start_tls = True

# LDAPS (port 636, native TLS)
ldap_uri = ldaps://ldap.example.com
```

## SSSD Configuration Reference

### Common Options

| Option | Description |
|--------|-------------|
| `ldap_uri` | LDAP server URI(s), space-separated for failover |
| `ldap_search_base` | Base DN for searches |
| `ldap_default_bind_dn` | Bind DN for non-anonymous searches |
| `ldap_default_authtok` | Bind password |
| `ldap_tls_reqcert` | TLS certificate validation level |
| `ldap_tls_cacert` | Path to CA certificate bundle |
| `cache_credentials` | Cache credentials for offline login |
| `entry_cache_timeout` | Cache entry lifetime in seconds |
| `enumerate` | List all users/groups (False for large directories) |
| `access_provider` | Access control method (ldap, ad, simple, permit) |
| `ldap_access_filter` | LDAP filter to restrict login access |

### Multiple LDAP Servers (Failover)

```ini
ldap_uri = ldaps://ldap1.example.com, ldaps://ldap2.example.com, ldaps://ldap3.example.com
ldap_backup_uri = ldaps://ldap-backup.example.com
```

### Bind DN (Non-anonymous Searches)

```ini
ldap_default_bind_dn = cn=sssd-bind,ou=ServiceAccounts,dc=example,dc=com
ldap_default_authtok_type = password
ldap_default_authtok = S3cur3P@ss
```

### Restrict Login Access

```ini
# Allow only members of a specific group
access_provider = ldap
ldap_access_filter = (memberOf=cn=linux-users,ou=Groups,dc=example,dc=com)

# Or use simple provider with allow list
access_provider = simple
simple_allow_groups = linux-users, developers
simple_allow_users = admin, service-user
```

### Sudo LDAP Integration

```ini
[sssd]
services = nss, pam, sudo

[domain/example.com]
sudo_provider = ldap
ldap_sudo_search_base = ou=SUDOers,dc=example,dc=com
```

## Troubleshooting

### Common Commands

```bash
# Check SSSD status
systemctl status sssd

# Clear SSSD cache (force re-read from server)
sss_cache -E

# Remove cache files entirely
systemctl stop sssd
rm -rf /var/lib/sss/db/*
systemctl start sssd

# Test LDAP connectivity
ldapsearch -x -H ldap://ldap.example.com -b "dc=example,dc=com" "(uid=testuser)"

# Test LDAPS connectivity
ldapsearch -x -H ldaps://ldap.example.com -b "dc=example,dc=com" "(uid=testuser)"

# Test with bind DN
ldapsearch -x -H ldaps://ldap.example.com \
  -D "cn=sssd-bind,ou=ServiceAccounts,dc=example,dc=com" \
  -W -b "dc=example,dc=com" "(uid=testuser)"

# Check NSS resolution
getent passwd ldapuser
getent group ldapgroup

# Check user details
id ldapuser

# SSSD domain status (RHEL 8+)
sssctl domain-list
sssctl domain-status example.com
sssctl user-checks ldapuser

# Check authentication
su - ldapuser
ssh ldapuser@localhost
```

### Enable Debug Logging

Add to `/etc/sssd/sssd.conf`:

```ini
[sssd]
debug_level = 9

[domain/example.com]
debug_level = 9

[nss]
debug_level = 9

[pam]
debug_level = 9
```

```bash
# Restart and watch logs
systemctl restart sssd
journalctl -u sssd -f

# Log files location
/var/log/sssd/sssd.log
/var/log/sssd/sssd_example.com.log
/var/log/sssd/sssd_nss.log
/var/log/sssd/sssd_pam.log
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `getent passwd` returns nothing | SSSD not running or misconfigured | Check `systemctl status sssd`, verify sssd.conf |
| TLS handshake failure | CA certificate not trusted | Install CA cert, check `ldap_tls_cacert` path |
| Login slow or hanging | DNS resolution issues | Verify DNS, add server IPs to `/etc/hosts` |
| `Permission denied` on sssd.conf | File permissions too open | `chmod 600 /etc/sssd/sssd.conf` |
| Cached stale data | Old entries in SSSD cache | `sss_cache -E` or clear `/var/lib/sss/db/` |
| `No such user` after server change | Enumerate off + cache | Clear cache and restart SSSD |
| Home directory not created | mkhomedir not enabled | Enable with `authselect` or `pam-auth-update` |
| Unable to connect to LDAP | Firewall blocking ports | Open port 389 (LDAP) or 636 (LDAPS) |

### Firewall Rules

```bash
# RHEL (firewalld)
firewall-cmd --permanent --add-service=ldap
firewall-cmd --permanent --add-service=ldaps
firewall-cmd --reload

# Ubuntu (ufw) - outbound is typically allowed by default
ufw allow out 389/tcp
ufw allow out 636/tcp
```

## Useful Commands

```bash
# Disable pam_access.so module
authconfig --disablepamaccess --update

# Print configuration settings
authconfig --test
authconfig --enableldap --enableldapauth --ldapserver="ldap://192.168.50.10" --ldapbasedn="dc=homelab,dc=local" --enablemkhomedir --test

# Print active configuration for authconfig
grep -v "=no" /etc/sysconfig/authconfig

# Save all files which authconfig modifies
authconfig --savebackup=/tmp/ldap_backup

# Restore the last configuration
authconfig --restorelastbackup
authconfig --restorebackup=/tmp/ldap_backup

# Install OpenLDAP tools
yum install openldap-clients

# List all LDAP users
ldapsearch -H ldap://192.168.50.10 -x -LLL uid=*
ldapsearch -W -D "cn=Manager,dc=homelab,dc=local" -h 192.168.50.10 -b dc=homelab,dc=local uid=daniel

# List a LDAP netgroup
ldapsearch -x -LLL -H ldap://localhost -b "dc=homelab,dc=local" "(cn=family)"

# Delete an LDAP user
ldapdelete -x -W -D "cn=Manager,dc=homelab,dc=local" "uid=daniel,ou=People,dc=homelab,dc=local"

# Delete an LDAP group
ldapdelete -x -W -D "cn=Manager,dc=homelab,dc=local" "cn=daniel,ou=Groups,dc=homelab,dc=local"

# Delete an LDAP netgroup
ldapdelete -x -W -D "cn=Manager,dc=homelab,dc=local" "cn=family,ou=Netgroup,dc=homelab,dc=local"

# Change password for a user
ldappasswd -H ldap://localhost -x -W -S -D "cn=Manager,dc=homelab,dc=local" "uid=daniel,ou=People,dc=homelab,dc=local"

# Generate a password for "userPassword"
slappasswd

# Read the manuals
man nslcd
man nslcd.conf

# Shows LDAP users on a system
getent passwd

# Shows LDAP groups on a system
getent group

# Shows users from a netgroup on a system
getent netgroup family
```

## Quick Reference: authconfig vs authselect

| | authconfig (RHEL 6–7) | authselect (RHEL 8+) |
|-|------------------------|----------------------|
| Install | `yum install authconfig` | Pre-installed |
| Enable SSSD | `authconfig --enablesssd --enablesssdauth --update` | `authselect select sssd` |
| Enable mkhomedir | `authconfig --enablemkhomedir --update` | `authselect select sssd with-mkhomedir` |
| Backup | `authconfig --savebackup=mybackup` | `authselect backup-create mybackup` |
| Restore | `authconfig --restorebackup=mybackup` | `authselect backup-restore mybackup` |
| Check current | `authconfig --test` | `authselect current` |

## Summary of Steps

1. Install packages (`sssd`, `sssd-ldap`, LDAP client tools)
2. Configure `/etc/sssd/sssd.conf` with server URI, base DN, and TLS settings
3. Set permissions: `chmod 600 /etc/sssd/sssd.conf`
4. Enable home directory creation (authselect/pam-auth-update/authconfig)
5. Start and enable SSSD: `systemctl enable --now sssd`
6. Verify with `getent passwd ldapuser` and `id ldapuser`
7. Test login with `su - ldapuser` or SSH

## LDAP Commands Reference

### Basic LDAP Search Commands

```bash
# Basic search
ldapsearch -x -H ldap://server.example.com -b "dc=example,dc=com"

# Search with authentication
ldapsearch -x -D "cn=admin,dc=example,dc=com" -W -H ldap://server.example.com -b "dc=example,dc=com"

# Search specific entry
ldapsearch -x -H ldap://server.example.com -b "dc=example,dc=com" "uid=john"

# Return only specific attributes
ldapsearch -x -H ldap://server.example.com -b "dc=example,dc=com" "uid=john" cn mail

# Return all attributes
ldapsearch -x -H ldap://server.example.com -b "dc=example,dc=com" "uid=john" "*"

# Return operational attributes
ldapsearch -x -H ldap://server.example.com -b "dc=example,dc=com" "uid=john" "+"
```

### Authentication

```bash
# Simple bind with username/password
ldapsearch -x -D "cn=admin,dc=example,dc=com" -w password

# Prompt for password
ldapsearch -x -D "cn=admin,dc=example,dc=com" -W

# Anonymous bind
ldapsearch -x

# SASL GSSAPI (Kerberos)
ldapsearch -Y GSSAPI -H ldap://server.example.com

# SASL PLAIN
ldapsearch -Y PLAIN -U username -W -H ldap://server.example.com
```

### Search Scope

```bash
# Base scope (single entry)
ldapsearch -x -s base -b "uid=john,ou=people,dc=example,dc=com"

# One level scope (immediate children)
ldapsearch -x -s one -b "ou=people,dc=example,dc=com"

# Subtree scope (default - all descendants)
ldapsearch -x -s sub -b "dc=example,dc=com"
```

### Advanced Search Examples

```bash
# Search for users in specific OU
ldapsearch -x -b "ou=people,dc=example,dc=com" "objectClass=person"

# Search with multiple filters
ldapsearch -x -b "dc=example,dc=com" "(&(objectClass=person)(cn=John*))"

# Case-insensitive search
ldapsearch -x -b "dc=example,dc=com" "cn~=john"

# Search for entries modified after specific date
ldapsearch -x -b "dc=example,dc=com" "modifyTimestamp>=20231201000000Z"
```

### Modify Operations

```bash
# Modify using LDIF file
ldapmodify -x -D "cn=admin,dc=example,dc=com" -W -f modify.ldif

# Modify from command line
cat << EOF | ldapmodify -x -D "cn=admin,dc=example,dc=com" -W
dn: uid=john,ou=people,dc=example,dc=com
changetype: modify
replace: mail
mail: john.new@example.com
EOF
```

### LDIF Modify Examples

```ldif
# Add attribute
dn: uid=john,ou=people,dc=example,dc=com
changetype: modify
add: telephoneNumber
telephoneNumber: +1-555-123-4567

# Replace attribute
dn: uid=john,ou=people,dc=example,dc=com
changetype: modify
replace: mail
mail: john.doe@example.com

# Delete specific attribute value
dn: uid=john,ou=people,dc=example,dc=com
changetype: modify
delete: telephoneNumber
telephoneNumber: +1-555-123-4567

# Delete all values of an attribute
dn: uid=john,ou=people,dc=example,dc=com
changetype: modify
delete: telephoneNumber
```

### Add Operations

```bash
# Add using LDIF file
ldapadd -x -D "cn=admin,dc=example,dc=com" -W -f newuser.ldif
```

Sample user LDIF:

```ldif
dn: uid=jane,ou=people,dc=example,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
cn: Jane Smith
sn: Smith
givenName: Jane
uid: jane
mail: jane.smith@example.com
userPassword: {SSHA}hashedpassword
```

Sample organizational unit LDIF:

```ldif
dn: ou=groups,dc=example,dc=com
objectClass: top
objectClass: organizationalUnit
ou: groups
description: Container for groups
```

### Delete Operations

```bash
# Delete specific entry
ldapdelete -x -D "cn=admin,dc=example,dc=com" -W "uid=john,ou=people,dc=example,dc=com"

# Delete using LDIF
cat << EOF | ldapmodify -x -D "cn=admin,dc=example,dc=com" -W
dn: uid=john,ou=people,dc=example,dc=com
changetype: delete
EOF

# Delete entry and all children (recursive)
ldapdelete -x -r -D "cn=admin,dc=example,dc=com" -W "ou=temp,dc=example,dc=com"
```

### Password Operations

```bash
# Change user password (prompt for new password)
ldappasswd -x -D "cn=admin,dc=example,dc=com" -W -S "uid=john,ou=people,dc=example,dc=com"

# Set specific password
ldappasswd -x -D "cn=admin,dc=example,dc=com" -w adminpass -s newpassword "uid=john,ou=people,dc=example,dc=com"
```

### Compare and Who Am I

```bash
# Compare attribute value
ldapcompare -x -H ldap://server.example.com "uid=john,ou=people,dc=example,dc=com" "mail:john@example.com"

# Check current authentication
ldapwhoami -x -D "cn=admin,dc=example,dc=com" -W -H ldap://server.example.com
```

### Common LDAP Filters

```bash
# Equality
"cn=John Doe"

# Wildcard
"cn=John*"
"cn=*Doe"
"cn=*John*"

# Present (attribute exists)
"mail=*"

# Not present (attribute doesn't exist)
"(!(mail=*))"

# AND
"(&(objectClass=person)(cn=John*))"

# OR
"(|(cn=John*)(cn=Jane*))"

# NOT
"(!(cn=John*))"

# Complex combination
"(&(objectClass=person)(|(cn=John*)(mail=*@example.com))(!(ou=disabled)))"

# Greater than or equal
"uidNumber>=1000"

# Less than or equal
"uidNumber<=9999"

# Approximate match
"cn~=Jon"
```

### Common Object Classes

| Object Class | Description |
|---|---|
| `top` | Base object class |
| `person` | Basic person information |
| `organizationalPerson` | Organizational person |
| `inetOrgPerson` | Internet organizational person |
| `posixAccount` | POSIX account |
| `groupOfNames` | Group with member DNs |
| `organizationalUnit` | Organizational unit |

### Common Attributes

| Attribute | Description |
|---|---|
| `cn` | Common name |
| `sn` | Surname |
| `givenName` | First name |
| `uid` | User ID |
| `mail` | Email address |
| `telephoneNumber` | Phone number |
| `userPassword` | Password |
| `uidNumber` | POSIX user ID |
| `gidNumber` | POSIX group ID |
| `homeDirectory` | Home directory path |
| `loginShell` | Login shell |

### Connection Options

```bash
# LDAP (unencrypted)
-H ldap://server.example.com:389

# LDAPS (SSL/TLS)
-H ldaps://server.example.com:636

# StartTLS
-H ldap://server.example.com -Z
```

### Common Command-Line Options

```bash
-x          # Simple authentication
-D          # Bind DN
-w          # Password (inline)
-W          # Prompt for password
-b          # Base DN for search
-s          # Search scope (base, one, sub)
-f          # Read from LDIF file
-v          # Verbose output
-LLL        # LDIF output without comments
-o          # Set options (e.g., -o ldif-wrap=no)
```

### Useful One-Liners

```bash
# Find all users
ldapsearch -x -LLL -b "dc=example,dc=com" "objectClass=person" cn mail

# Find all groups
ldapsearch -x -LLL -b "dc=example,dc=com" "objectClass=groupOfNames" cn member

# Find user's groups
ldapsearch -x -LLL -b "dc=example,dc=com" "member=uid=john,ou=people,dc=example,dc=com" cn

# Export entire directory
ldapsearch -x -LLL -b "dc=example,dc=com" > backup.ldif

# Count entries
ldapsearch -x -b "dc=example,dc=com" "objectClass=person" | grep -c "^dn:"

# Find user by email
ldapsearch -x -b "dc=example,dc=com" "mail=user@example.com"

# Find users in specific department
ldapsearch -x -b "dc=example,dc=com" "departmentNumber=IT"

# Find disabled users (Active Directory)
ldapsearch -x -b "dc=example,dc=com" "userAccountControl=514"

# Find empty groups
ldapsearch -x -b "dc=example,dc=com" "(&(objectClass=group)(!(member=*)))"

# Find groups user belongs to
ldapsearch -x -b "dc=example,dc=com" "member=uid=john,ou=people,dc=example,dc=com"
```

### System Administration

```bash
# Check replication status
ldapsearch -x -s base -b "" "objectClass=*" contextCSN

# Monitor connections
ldapsearch -x -s base -b "cn=monitor" "objectClass=*"

# Check schema
ldapsearch -x -s base -b "cn=schema,cn=config" "objectClass=*"
```

### Debug Options

```bash
# Enable debug output
ldapsearch -d 1 -x -b "dc=example,dc=com"

# Network debugging
ldapsearch -d 2 -x -b "dc=example,dc=com"

# Filter debugging
ldapsearch -d 32 -x -b "dc=example,dc=com"

# Certificate issues with LDAPS
export LDAPTLS_REQCERT=never

# Timeout issues
ldapsearch -o nettimeout=30 -o ldif-wrap=no

# Large result sets (no size or time limits)
ldapsearch -z 0 -l 0
```

### Quick Reference Card

| Operation | Command Pattern |
|-----------|----------------|
| Search | `ldapsearch -x -b "base_dn" "filter"` |
| Add | `ldapadd -x -D "bind_dn" -W -f file.ldif` |
| Modify | `ldapmodify -x -D "bind_dn" -W -f file.ldif` |
| Delete | `ldapdelete -x -D "bind_dn" -W "entry_dn"` |
| Password | `ldappasswd -x -D "bind_dn" -W -S "user_dn"` |
| Compare | `ldapcompare -x "entry_dn" "attr:value"` |
