# Installing Nagios Core from Source on RHEL and Ubuntu

Step-by-step guide for installing Nagios Core and Nagios Plugins from source on RHEL 9 / CentOS Stream 9 and Ubuntu 22.04 / 24.04.

## Prerequisites

### Disable SELinux (or Set Permissive)

```bash
# Set to permissive immediately
sudo setenforce 0

# Disable permanently (requires reboot)
sudo sed -i 's/SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
```

### Install Required Packages

```bash
# Nagios Core prerequisites
sudo dnf install -y gcc glibc glibc-common perl httpd php wget gd gd-devel \
    s-nail postfix openssl-devel

# Update system
sudo dnf update -y
```

## Install Nagios Core

### Download Source

```bash
cd /tmp

# Download latest Nagios Core release
wget --output-document="nagioscore.tar.gz" \
    $(wget -q -O - https://api.github.com/repos/NagiosEnterprises/nagioscore/releases/latest \
    | grep '"browser_download_url":' | grep -o 'https://[^"]*')

# Extract
tar xzf nagioscore.tar.gz
```

### Compile

```bash
cd /tmp/nagios-*
./configure
make all
```

### Create User and Group

```bash
# Creates nagios user/group and adds apache to nagios group
make install-groups-users
usermod -a -G nagios apache
```

### Install Binaries

```bash
# Installs binary files, CGIs, and HTML files
make install
```

### Install Service/Daemon

```bash
# Install systemd service and enable Apache
make install-daemoninit
systemctl enable httpd.service
```

### Install Command Mode

```bash
# Installs and configures the external command file
make install-commandmode
```

### Install Configuration Files

```bash
# Installs sample configuration files
make install-config
```

### Install Apache Config

```bash
# Installs Apache web server configuration for Nagios
make install-webconf
```

### Configure Firewall

```bash
# Allow HTTP through firewall
sudo firewall-cmd --zone=public --add-port=80/tcp
sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
```

### Create nagiosadmin User

```bash
# Create web interface user (you'll be prompted for a password)
htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin

# IMPORTANT: When adding more users later, remove -c flag:
# htpasswd /usr/local/nagios/etc/htpasswd.users newuser
```

### Start Services

```bash
# Start Apache
systemctl start httpd.service

# Start Nagios
systemctl start nagios.service

# Verify
systemctl status nagios.service
systemctl status httpd.service
```

### Test Nagios

Open your browser and navigate to:

```
http://<your-server-ip>/nagios
```

Login with username `nagiosadmin` and the password you set. You'll see errors about missing plugins — that's expected until the next section.

## Install Nagios Plugins

### Plugin Prerequisites

```bash
# Install EPEL repository
sudo dnf -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm

# Enable CodeReady Builder (CRB) repository
sudo subscription-manager repos --enable codeready-builder-for-rhel-9-x86_64-rpms

# Install plugin dependencies
sudo dnf install -y gcc glibc glibc-common make gettext automake autoconf \
    wget openssl-devel net-snmp net-snmp-utils perl-Net-SNMP
```

### Download Plugin Source

```bash
cd /tmp

# Download latest Nagios Plugins release
wget --output-document="nagios-plugins.tar.gz" \
    $(wget -q -O - https://api.github.com/repos/nagios-plugins/nagios-plugins/releases/latest \
    | grep '"browser_download_url":' | grep -o 'https://[^"]*')

# Extract
tar zxf nagios-plugins.tar.gz
```

### Compile and Install Plugins

```bash
cd /tmp/nagios-plugins-*
./configure
make
make install
```

### Verify Plugin Installation

Plugins are now located at:

```bash
ls /usr/local/nagios/libexec/
```

Go back to the Nagios web interface, navigate to a host or service, and click "Re-schedule the next check" under Commands. The plugin errors should now be resolved.

## Service Management

```bash
# Start
systemctl start nagios.service

# Stop
systemctl stop nagios.service

# Restart
systemctl restart nagios.service

# Status
systemctl status nagios.service

# Enable on boot
systemctl enable nagios.service
```

## Verify Configuration

```bash
# Check Nagios config for errors before restarting
/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
```

## Directory Structure

| Path | Purpose |
|------|---------|
| `/usr/local/nagios/bin/` | Nagios binaries |
| `/usr/local/nagios/etc/` | Configuration files |
| `/usr/local/nagios/libexec/` | Plugins |
| `/usr/local/nagios/share/` | Web interface files |
| `/usr/local/nagios/var/` | Log and state files |
| `/usr/local/nagios/var/rw/` | External command pipe |

## Post-Installation Configuration

### Add a Host to Monitor

```bash
# Create a host definition file
cat << 'EOF' | sudo tee /usr/local/nagios/etc/objects/myhost.cfg
define host {
    use                     linux-server
    host_name               webserver01
    alias                   Web Server 01
    address                 192.168.1.100
    max_check_attempts      5
    check_period            24x7
    notification_interval   30
    notification_period     24x7
}

define service {
    use                     generic-service
    host_name               webserver01
    service_description     PING
    check_command           check_ping!100.0,20%!500.0,60%
}

define service {
    use                     generic-service
    host_name               webserver01
    service_description     SSH
    check_command           check_ssh
}
EOF

# Add the config file to nagios.cfg
echo 'cfg_file=/usr/local/nagios/etc/objects/myhost.cfg' >> /usr/local/nagios/etc/nagios.cfg

# Verify config
/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg

# Restart Nagios
systemctl restart nagios.service
```

### Enable Email Notifications

```bash
# Ensure postfix is running
systemctl enable --now postfix

# Edit contacts.cfg to set your email
vi /usr/local/nagios/etc/objects/contacts.cfg
# Change: email  nagios@localhost  →  your-email@example.com
```

### Enable HTTPS

```bash
# Install mod_ssl
sudo dnf install -y mod_ssl

# Generate self-signed certificate (or use your own)
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/pki/tls/private/nagios.key \
    -out /etc/pki/tls/certs/nagios.crt

# Restart Apache
systemctl restart httpd.service

# Update firewall
sudo firewall-cmd --zone=public --add-port=443/tcp --permanent
sudo firewall-cmd --reload
```

## Upgrading Nagios Core

```bash
cd /tmp

# Download new version
wget --output-document="nagioscore.tar.gz" \
    $(wget -q -O - https://api.github.com/repos/NagiosEnterprises/nagioscore/releases/latest \
    | grep '"browser_download_url":' | grep -o 'https://[^"]*')

tar xzf nagioscore.tar.gz
cd /tmp/nagios-*

# Compile and install (overwrites binaries, keeps config)
./configure
make all
make install

# Restart
systemctl restart nagios.service

# Verify version
/usr/local/nagios/bin/nagios --version
```

## Troubleshooting

### "No output on stdout" Errors

```bash
# Plugins not installed — follow the "Install Nagios Plugins" section above
ls /usr/local/nagios/libexec/

# If plugins exist but still failing, check permissions
ls -la /usr/local/nagios/libexec/
chown -R nagios:nagios /usr/local/nagios/libexec/
```

### Cannot Access Web Interface

```bash
# Check Apache is running
systemctl status httpd.service

# Check firewall
firewall-cmd --list-ports

# Check Apache error log
tail -20 /var/log/httpd/error_log

# Check Nagios Apache config exists
ls -la /etc/httpd/conf.d/nagios.conf
```

### Permission Denied on Command Pipe

```bash
# Fix command pipe permissions
chmod 660 /usr/local/nagios/var/rw/nagios.cmd
chown nagios:apache /usr/local/nagios/var/rw/nagios.cmd

# Verify command mode
ls -la /usr/local/nagios/var/rw/
```

### Config Verification Failed

```bash
# Always verify before restarting
/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg

# Common issues:
# - Missing semicolons or brackets in object definitions
# - Referencing undefined commands, timeperiods, or contacts
# - Duplicate object definitions
```

## Install on Ubuntu 22.04 / 24.04

### Disable SELinux (If Installed)

SELinux is not enabled by default on Ubuntu. Verify with:

```bash
sudo dpkg -l selinux*
```

### Install Required Packages

```bash
sudo apt update
sudo apt install -y autoconf gcc libc6 make wget unzip apache2 apache2-utils \
    php libapache2-mod-php libgd-dev openssl libssl-dev
```

### Download Source

```bash
cd /tmp

# Download latest Nagios Core release
wget -O nagioscore.tar.gz \
    $(wget -q -O - https://api.github.com/repos/NagiosEnterprises/nagioscore/releases/latest \
    | grep '"browser_download_url":' | grep -o 'https://[^"]*')

# Extract
tar xzf nagioscore.tar.gz
```

### Compile

```bash
cd /tmp/nagios-*
sudo ./configure --with-httpd-conf=/etc/apache2/sites-enabled
sudo make all
```

### Create User and Group

```bash
sudo make install-groups-users
sudo usermod -a -G nagios www-data
```

### Install Binaries

```bash
sudo make install
```

### Install Service/Daemon

```bash
sudo make install-daemoninit
```

### Install Command Mode

```bash
sudo make install-commandmode
```

### Install Configuration Files

```bash
sudo make install-config
```

### Install Apache Config

```bash
sudo make install-webconf
sudo a2enmod rewrite
sudo a2enmod cgi
```

### Configure Firewall

```bash
sudo ufw allow Apache
sudo ufw reload
```

### Create nagiosadmin User

```bash
sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
```

### Start Services

```bash
sudo systemctl restart apache2.service
sudo systemctl start nagios.service
sudo systemctl enable nagios.service

# Verify
sudo systemctl status nagios.service
```

### Test

Open `http://<your-server-ip>/nagios` and login with `nagiosadmin`.

## Install Nagios Plugins on Ubuntu

### Plugin Prerequisites

```bash
sudo apt install -y autoconf gcc libc6 libmcrypt-dev make libssl-dev wget bc \
    gawk dc build-essential snmp libnet-snmp-perl gettext iputils-ping
```

### Download Plugin Source

```bash
cd /tmp

wget -O nagios-plugins.tar.gz \
    $(wget -q -O - https://api.github.com/repos/nagios-plugins/nagios-plugins/releases/latest \
    | grep '"browser_download_url":' | grep -o 'https://[^"]*')

tar zxf nagios-plugins.tar.gz
```

### Compile and Install Plugins

```bash
cd /tmp/nagios-plugins-*
sudo ./configure
sudo make
sudo make install
```

### Service Management (Ubuntu)

```bash
sudo systemctl start nagios.service
sudo systemctl stop nagios.service
sudo systemctl restart nagios.service
sudo systemctl status nagios.service
```

## Quick Reference

| Action | Command |
|--------|---------|
| Start Nagios | `systemctl start nagios.service` |
| Stop Nagios | `systemctl stop nagios.service` |
| Restart Nagios | `systemctl restart nagios.service` |
| Verify config | `/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg` |
| Check version | `/usr/local/nagios/bin/nagios --version` |
| Web interface | `http://<server-ip>/nagios` |
| Plugins directory | `/usr/local/nagios/libexec/` |
| Config directory | `/usr/local/nagios/etc/` |
| Logs | `/usr/local/nagios/var/nagios.log` |
