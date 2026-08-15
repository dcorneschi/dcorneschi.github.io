# MySQL LDAP Authentication with PAM and SSSD

This guide covers installing MySQL from a commercial tarball on RHEL 7+, configuring SSSD for LDAP authentication, and setting up PAM-based LDAP authentication for MySQL users.

## Overview

MySQL Enterprise supports PAM-based authentication, allowing users to authenticate against external directories (LDAP, Active Directory) via SSSD and PAM modules. This avoids managing passwords in MySQL directly.

| Component | Purpose |
|-----------|---------|
| MySQL PAM plugin | `authentication_pam` — delegates auth to the OS PAM stack |
| SSSD | Caches LDAP identities and handles PAM auth requests |
| PAM (`pam_sss.so`) | Bridges MySQL's PAM call to SSSD |
| LDAP server | Central directory holding user credentials |

### Authentication Flow

```
MySQL client → MySQL server (authentication_pam) → PAM (mysql-ldap) → pam_sss.so → SSSD → LDAP server
```

---

## Install MySQL (Commercial Tarball)

### Prerequisites

```bash
yum install libaio mysql
```

### Create MySQL User and Group

```bash
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
```

### Extract and Set Up

```bash
cd /usr/local
tar zxvf mysql-commercial-8.0.30-el7-x86_64.tar.gz
ln -s mysql-commercial-8.0.30-el7-x86_64 mysql
cd /usr/local/mysql
mkdir mysql-files
chown mysql:mysql mysql-files
chmod 750 mysql-files
```

### Initialize and Start

```bash
# Initialize the data directory (note the temporary root password in the output)
/usr/local/mysql/bin/mysqld --initialize --user=mysql

# Generate SSL/RSA certificates
/usr/local/mysql/bin/mysql_ssl_rsa_setup

# Start MySQL
/usr/local/mysql/bin/mysqld_safe --user=mysql &
```

### Install Init Script

```bash
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql

mkdir /var/lib/mysql
chown mysql:mysql /var/lib/mysql

/etc/init.d/mysql restart
```

---

## Create MySQL Users

### Set Root Password

Connect with the temporary password shown during `--initialize`:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'secret_password';
```

### Create PAM-Authenticated User

```sql
CREATE USER 'username'@'localhost' IDENTIFIED WITH authentication_pam AS 'mysql-ldap';
```

The `'mysql-ldap'` string references the PAM service file `/etc/pam.d/mysql-ldap`.

### Test Connection

```bash
mysql -u username -p --enable-cleartext-plugin
```

The `--enable-cleartext-plugin` flag is required for PAM authentication — it sends the password in cleartext to the server (use SSL/TLS to protect the connection).

---

## Configure SSSD

### Install Packages

```bash
yum install sssd sssd-tools oddjob-mkhomedir
```

### Configure /etc/sssd/sssd.conf

```ini
[sssd]
config_file_version = 2
services = nss, pam, autofs
domains = ldap
debug_level = 9

[nss]
homedir_substring = /home
debug_level = 9

[pam]
debug_level = 9

[domain/ldap]
id_provider = ldap
autofs_provider = ldap
auth_provider = ldap
chpass_provider = ldap
ldap_uri = ldaps://192.168.50.30
ldap_search_base = dc=homelab,dc=local
ldap_id_use_start_tls = False
ldap_tls_cacertdir = /etc/openldap/cacerts
cache_credentials = True
ldap_tls_reqcert = allow
debug_level = 9
```

### Set Permissions and Start

```bash
chmod 600 /etc/sssd/sssd.conf
systemctl enable --now sssd
```

---

## Add PAM Module for MySQL

Create `/etc/pam.d/mysql-ldap`:

```
auth    required pam_sss.so domains=ldap
account required pam_sss.so domains=ldap
```

This tells MySQL's PAM plugin to authenticate and authorize via SSSD using the `ldap` domain.

---

## TLS Certificate Setup

### Verify LDAP Server Certificate

```bash
openssl s_client -connect 192.168.50.30:636 -showcerts < /dev/null | openssl x509 -text
```

### Verify Local CA Certificates

```bash
for cert in /etc/openldap/cacerts/*.crt; do openssl verify -show_chain $cert; done
```

### Extract Certificates from LDAP Server

```bash
openssl s_client -showcerts -verify 5 -connect 192.168.50.30:636 < /dev/null | \
  awk '/BEGIN/,/END/{ if(/BEGIN/) {a++}; out="ldap_server.crt"a".pem"; print >out}'
```

### Add CA Certificate to System Trust Store

```bash
cp /etc/openldap/cacerts/ldap_server.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust extract
```

---

## Useful Commands

### SSSD Diagnostics

```bash
# List configured domains
sssctl domain-list

# Check domain status
sssctl domain-status ldap

# List available authselect features
authselect list-features sssd

# Validate SSSD configuration
sssctl config-check
```

### LDAP Queries

```bash
# Search for a user
ldapsearch -x -H "ldaps://192.168.50.30" -b "dc=homelab,dc=local" -s sub "(uid=username)"

# Debug mode (-d 1)
ldapsearch -d 1 -x -H "ldaps://192.168.50.30" -b "dc=homelab,dc=local" -s sub "(uid=username)"
```

### SSSD Logs

```bash
# Domain log (identity lookups)
less /var/log/sssd/sssd_ldap.log

# PAM log (authentication)
less /var/log/sssd/sssd_pam.log

# Main SSSD log
less /var/log/sssd/sssd.log
```

---

## MySQL Proxy User Mapping

If all LDAP users in a group should share the same MySQL privileges, use proxy authentication:

```sql
-- Create the proxied account with actual privileges
CREATE USER 'app_readonly'@'localhost' IDENTIFIED WITH mysql_no_login;
GRANT SELECT ON mydb.* TO 'app_readonly'@'localhost';

-- Create the LDAP-authenticated user
CREATE USER 'ldapuser'@'localhost' IDENTIFIED WITH authentication_pam AS 'mysql-ldap';

-- Grant proxy privilege
GRANT PROXY ON 'app_readonly'@'localhost' TO 'ldapuser'@'localhost';
```

When `ldapuser` connects via PAM/LDAP, they inherit the privileges of `app_readonly`. This lets you manage access by group without creating individual MySQL grants.

To map an entire group (using PAM group mapping in `mysql-ldap`):

```sql
-- Any user authenticating through PAM maps to this account
CREATE USER ''@'' IDENTIFIED WITH authentication_pam AS 'mysql-ldap';
GRANT PROXY ON 'app_readonly'@'localhost' TO ''@'';
```

---

## authselect Setup (RHEL 8+)

On RHEL 8+ and Fedora, `authselect` manages NSS/PAM configuration. Enable SSSD system-wide:

```bash
# Select SSSD profile with home directory creation
authselect select sssd with-mkhomedir --force

# Verify current profile
authselect current

# List available features for SSSD profile
authselect list-features sssd

# Enable additional features
authselect enable-feature with-pamaccess
```

Enable the oddjobd service for automatic home directory creation:

```bash
systemctl enable --now oddjobd
```

---

## Systemd Service for MySQL

For RHEL 7+ deployments, create a proper systemd unit file instead of the legacy init script:

Create `/etc/systemd/system/mysqld.service`:

```ini
[Unit]
Description=MySQL Server
After=network.target

[Service]
Type=forking
User=mysql
Group=mysql
PIDFile=/usr/local/mysql/data/mysqld.pid
ExecStart=/usr/local/mysql/bin/mysqld_safe --user=mysql
ExecStop=/usr/local/mysql/bin/mysqladmin -u root -p shutdown
TimeoutSec=300
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
systemctl daemon-reload
systemctl enable --now mysqld
systemctl status mysqld
```

---

## SSSD with Bind User (Non-Anonymous LDAP)

If your LDAP server requires authentication for searches (anonymous bind disabled), add bind credentials to the domain configuration in `/etc/sssd/sssd.conf`:

```ini
[domain/ldap]
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap
ldap_uri = ldaps://192.168.50.30
ldap_search_base = dc=homelab,dc=local

# Bind user for LDAP searches
ldap_default_bind_dn = cn=svc-sssd,ou=Service Accounts,dc=homelab,dc=local
ldap_default_authtok_type = password
ldap_default_authtok = bind_password_here

ldap_tls_cacertdir = /etc/openldap/cacerts
ldap_tls_reqcert = demand
cache_credentials = True
```

The bind user only needs read access to the directory — never use a privileged account.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Access denied` in MySQL | PAM service file missing | Create `/etc/pam.d/mysql-ldap` |
| `Access denied` in MySQL | User not in LDAP | Verify with `ldapsearch` |
| SSSD won't start | Bad permissions on config | `chmod 600 /etc/sssd/sssd.conf` |
| TLS handshake failure | CA cert not trusted | Add cert to `/etc/pki/ca-trust/source/anchors/` and run `update-ca-trust extract` |
| `--enable-cleartext-plugin` required | Expected behavior for PAM auth | Use SSL/TLS connection to protect credentials |
| SSSD cache stale | Old credentials cached | `sss_cache -E` to flush all caches |
| MySQL can't find PAM plugin | Plugin not loaded | `INSTALL PLUGIN authentication_pam SONAME 'authentication_pam.so';` |

---

## Security Considerations

- Always use `ldaps://` or STARTTLS — never send credentials over plaintext LDAP
- Protect the MySQL connection with TLS when using `--enable-cleartext-plugin`
- Set `debug_level = 0` in production (9 is for troubleshooting only, logs contain sensitive data)
- Restrict `/etc/sssd/sssd.conf` to root only (`chmod 600`)
- Use `ldap_tls_reqcert = demand` in production (instead of `allow`) to enforce certificate validation
