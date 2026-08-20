# Installing Cacti from Source on RHEL and Ubuntu

Cacti is an open-source, RRDTool-based network monitoring and graphing solution. This guide covers installing Cacti from source with Apache, MariaDB, and PHP on RHEL 9 and Ubuntu 22.04/24.04.

## Prerequisites Overview

Cacti requires:
- Web server (Apache)
- PHP 8.1+ with required modules
- MariaDB or MySQL
- RRDTool
- SNMP
- PHP Composer (since Cacti 1.2.31)

## Install on RHEL 9 / CentOS Stream 9

### Install Required Packages

```bash
# Enable EPEL and CRB repositories
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled crb

# Install Apache, PHP, MariaDB
sudo dnf install -y httpd mariadb-server mariadb php php-mysqlnd php-snmp \
    php-xml php-mbstring php-json php-gd php-gmp php-posix php-ldap \
    php-intl php-pdo php-opcache php-cli php-common php-zip

# Install RRDTool and SNMP
sudo dnf install -y rrdtool net-snmp net-snmp-utils net-snmp-libs

# Install development tools (for Spine)
sudo dnf install -y gcc automake autoconf libtool mariadb-devel \
    net-snmp-devel help2man dos2unix wget git

# Install Composer
sudo dnf install -y php-composer-installers
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

### Enable and Start Services

```bash
sudo systemctl enable --now httpd mariadb

# Verify
systemctl status httpd mariadb
```

### Secure MariaDB

```bash
sudo mysql_secure_installation
# Set root password, remove anonymous users, disallow remote root login, remove test DB
```

### Configure PHP

```bash
# Edit PHP settings
sudo vi /etc/php.ini
```

Set the following values:

```ini
date.timezone = America/New_York
memory_limit = 512M
max_execution_time = 60
max_input_vars = 5000
```

```bash
# Restart Apache to apply
sudo systemctl restart httpd
```

### Configure MariaDB for Cacti

```bash
# Edit MariaDB configuration
sudo vi /etc/my.cnf.d/mariadb-server.cnf
```

Add under `[mysqld]`:

```ini
[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
max_heap_table_size=256M
tmp_table_size=256M
join_buffer_size=128M
innodb_file_format=Barracuda
innodb_large_prefix=1
innodb_buffer_pool_size=1G
innodb_buffer_pool_instances=10
innodb_flush_log_at_timeout=3
innodb_read_io_threads=32
innodb_write_io_threads=16
innodb_io_capacity=5000
innodb_io_capacity_max=10000
innodb_doublewrite=OFF
log_error=/var/log/mariadb/mariadb.log
```

```bash
# Restart MariaDB
sudo systemctl restart mariadb

# Load timezone data
mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -u root -p mysql
```

### Create Cacti Database and User

```bash
mysql -u root -p
```

```sql
CREATE DATABASE cacti DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'cacti'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON cacti.* TO 'cacti'@'localhost';
GRANT SELECT ON mysql.time_zone_name TO 'cacti'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Download and Install Cacti

```bash
cd /tmp

# Download latest Cacti release
wget https://www.cacti.net/downloads/cacti-latest.tar.gz

# Extract to web root
tar xzf cacti-latest.tar.gz
sudo mv cacti-1* /var/www/html/cacti

# Import database schema
mysql -u cacti -p cacti < /var/www/html/cacti/cacti.sql

# Configure database connection
sudo vi /var/www/html/cacti/include/config.php
```

Set database credentials in `config.php`:

```php
$database_type     = 'mysql';
$database_default  = 'cacti';
$database_hostname = 'localhost';
$database_username = 'cacti';
$database_password = 'your_secure_password';
$database_port     = '3306';
$database_retries  = 5;
$database_ssl      = false;
$database_ssl_key  = '';
$database_ssl_cert = '';
$database_ssl_ca   = '';
```

### Install PHP Dependencies with Composer

```bash
cd /var/www/html/cacti
sudo composer install --no-dev
```

### Set Permissions

```bash
# Create cacti user
sudo useradd -r -s /sbin/nologin cacti

# Set ownership
sudo chown -R cacti:apache /var/www/html/cacti/rra/
sudo chown -R cacti:apache /var/www/html/cacti/log/
sudo chown -R cacti:apache /var/www/html/cacti/cache/
sudo chown -R cacti:apache /var/www/html/cacti/scripts/
sudo chown -R cacti:apache /var/www/html/cacti/resource/
sudo chmod -R 775 /var/www/html/cacti/rra/ /var/www/html/cacti/log/ /var/www/html/cacti/cache/
```

### Configure Apache

```bash
cat << 'EOF' | sudo tee /etc/httpd/conf.d/cacti.conf
Alias /cacti /var/www/html/cacti

<Directory /var/www/html/cacti>
    Options -Indexes +FollowSymLinks
    AllowOverride All
    Require all granted

    <IfModule mod_php.c>
        php_flag magic_quotes_gpc Off
        php_flag short_open_tag On
        php_flag register_globals Off
        php_flag register_argc_argv On
        php_value max_execution_time 300
        php_value memory_limit 512M
        php_value max_input_vars 5000
    </IfModule>

    DirectoryIndex index.php
</Directory>
EOF

sudo systemctl restart httpd
```

### Configure Firewall

```bash
sudo firewall-cmd --zone=public --add-service=http --permanent
sudo firewall-cmd --zone=public --add-service=https --permanent
sudo firewall-cmd --reload
```

### Set Up Poller Cron Job

```bash
# Create cron entry
echo '*/5 * * * * cacti php /var/www/html/cacti/poller.php &>/dev/null' | sudo tee /etc/cron.d/cacti
```

### Configure SELinux (If Enabled)

```bash
# Allow Apache to connect to the network
sudo setsebool -P httpd_can_network_connect on

# Allow Apache to write to Cacti directories
sudo chcon -R -t httpd_sys_rw_content_t /var/www/html/cacti/rra/
sudo chcon -R -t httpd_sys_rw_content_t /var/www/html/cacti/log/
sudo chcon -R -t httpd_sys_rw_content_t /var/www/html/cacti/cache/
```

## Install on Ubuntu 22.04 / 24.04

### Install Required Packages

```bash
sudo apt update

# Install Apache, PHP, MariaDB
sudo apt install -y apache2 mariadb-server mariadb-client \
    php php-mysql php-snmp php-xml php-mbstring php-json php-gd \
    php-gmp php-posix php-ldap php-intl php-pdo php-opcache \
    php-cli php-common php-zip php-curl libapache2-mod-php

# Install RRDTool and SNMP
sudo apt install -y rrdtool snmp snmpd libsnmp-dev

# Install development tools (for Spine)
sudo apt install -y gcc make automake autoconf libtool \
    libmariadb-dev libsnmp-dev help2man dos2unix wget git

# Install Composer
sudo apt install -y composer
```

### Enable and Start Services

```bash
sudo systemctl enable --now apache2 mariadb

# Enable Apache modules
sudo a2enmod rewrite
sudo systemctl restart apache2
```

### Secure MariaDB

```bash
sudo mysql_secure_installation
```

### Configure PHP

```bash
# Find the correct php.ini
php --ini | grep "Loaded Configuration"

# Edit PHP settings (adjust path for your PHP version)
sudo vi /etc/php/8.1/apache2/php.ini
sudo vi /etc/php/8.1/cli/php.ini
```

Set the following values in both files:

```ini
date.timezone = America/New_York
memory_limit = 512M
max_execution_time = 60
max_input_vars = 5000
```

```bash
sudo systemctl restart apache2
```

### Configure MariaDB for Cacti

```bash
sudo vi /etc/mysql/mariadb.conf.d/50-server.cnf
```

Add under `[mysqld]`:

```ini
[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
max_heap_table_size=256M
tmp_table_size=256M
join_buffer_size=128M
innodb_buffer_pool_size=1G
innodb_buffer_pool_instances=10
innodb_flush_log_at_timeout=3
innodb_read_io_threads=32
innodb_write_io_threads=16
innodb_io_capacity=5000
innodb_io_capacity_max=10000
innodb_doublewrite=OFF
```

```bash
sudo systemctl restart mariadb

# Load timezone data
mysql_tzinfo_to_sql /usr/share/zoneinfo | sudo mysql -u root mysql
```

### Create Cacti Database and User

```bash
sudo mysql
```

```sql
CREATE DATABASE cacti DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'cacti'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON cacti.* TO 'cacti'@'localhost';
GRANT SELECT ON mysql.time_zone_name TO 'cacti'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Download and Install Cacti

```bash
cd /tmp

# Download latest Cacti
wget https://www.cacti.net/downloads/cacti-latest.tar.gz

# Extract to web root
tar xzf cacti-latest.tar.gz
sudo mv cacti-1* /var/www/html/cacti

# Import database schema
mysql -u cacti -p cacti < /var/www/html/cacti/cacti.sql

# Configure database connection
sudo vi /var/www/html/cacti/include/config.php
# Set $database_username, $database_password, etc. (same as RHEL section)
```

### Install PHP Dependencies with Composer

```bash
cd /var/www/html/cacti
sudo composer install --no-dev
```

### Set Permissions

```bash
sudo useradd -r -s /usr/sbin/nologin cacti

sudo chown -R cacti:www-data /var/www/html/cacti/rra/
sudo chown -R cacti:www-data /var/www/html/cacti/log/
sudo chown -R cacti:www-data /var/www/html/cacti/cache/
sudo chown -R cacti:www-data /var/www/html/cacti/scripts/
sudo chown -R cacti:www-data /var/www/html/cacti/resource/
sudo chmod -R 775 /var/www/html/cacti/rra/ /var/www/html/cacti/log/ /var/www/html/cacti/cache/
```

### Configure Apache

```bash
cat << 'EOF' | sudo tee /etc/apache2/sites-available/cacti.conf
Alias /cacti /var/www/html/cacti

<Directory /var/www/html/cacti>
    Options -Indexes +FollowSymLinks
    AllowOverride All
    Require all granted

    <IfModule mod_php.c>
        php_flag magic_quotes_gpc Off
        php_flag short_open_tag On
        php_flag register_globals Off
        php_flag register_argc_argv On
        php_value max_execution_time 300
        php_value memory_limit 512M
        php_value max_input_vars 5000
    </IfModule>

    DirectoryIndex index.php
</Directory>
EOF

sudo a2ensite cacti.conf
sudo systemctl restart apache2
```

### Configure Firewall

```bash
sudo ufw allow 'Apache Full'
sudo ufw reload
```

### Set Up Poller Cron Job

```bash
echo '*/5 * * * * cacti php /var/www/html/cacti/poller.php &>/dev/null' | sudo tee /etc/cron.d/cacti
```

## Install Spine (Optional, Recommended)

Spine is a fast C-based poller replacement for the default PHP poller.

### Compile Spine on RHEL 9

```bash
cd /tmp
wget https://www.cacti.net/downloads/spine/cacti-spine-latest.tar.gz
tar xzf cacti-spine-latest.tar.gz
cd cacti-spine-*

./bootstrap
./configure
make
sudo make install

# Configure Spine
sudo cp /usr/local/spine/etc/spine.conf.dist /usr/local/spine/etc/spine.conf
sudo vi /usr/local/spine/etc/spine.conf
```

### Compile Spine on Ubuntu

```bash
cd /tmp
wget https://www.cacti.net/downloads/spine/cacti-spine-latest.tar.gz
tar xzf cacti-spine-latest.tar.gz
cd cacti-spine-*

./bootstrap
./configure
make
sudo make install

sudo cp /usr/local/spine/etc/spine.conf.dist /usr/local/spine/etc/spine.conf
sudo vi /usr/local/spine/etc/spine.conf
```

### Configure Spine

```ini
DB_Host     localhost
DB_Database cacti
DB_User     cacti
DB_Pass     your_secure_password
DB_Port     3306
```

After installation, set Spine as the poller in the Cacti web UI under **Configuration → Settings → Poller → Poller Type → spine**.

## Complete the Web Installation

1. Open your browser: `http://<server-ip>/cacti`
2. Login with default credentials: `admin` / `admin`
3. You will be prompted to change the password
4. Follow the installation wizard:
   - Accept the license
   - Verify PHP module requirements
   - Verify directory permissions
   - Configure database connection paths
   - Select installation type (New Install)
   - Review and confirm settings

## Post-Installation

### Verify SNMP

```bash
# Test local SNMP
snmpwalk -v2c -c public localhost system

# Configure SNMP community
sudo vi /etc/snmp/snmpd.conf
# rocommunity public 127.0.0.1

sudo systemctl enable --now snmpd
```

### Enable Systemd Poller (Alternative to Cron)

```bash
# Copy the systemd service file
sudo cp /var/www/html/cacti/service/cactid.service /etc/systemd/system/

# Edit paths if needed
sudo vi /etc/systemd/system/cactid.service

# Create environment file
sudo touch /etc/sysconfig/cactid    # RHEL
# or
sudo touch /etc/default/cactid      # Ubuntu

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable --now cactid
sudo systemctl status cactid
```

## Troubleshooting

### PHP Modules Missing

```bash
# Check installed PHP modules
php -m

# RHEL — install missing module
sudo dnf install php-<module>

# Ubuntu — install missing module
sudo apt install php-<module>

# Restart web server after installing modules
sudo systemctl restart httpd    # RHEL
sudo systemctl restart apache2  # Ubuntu
```

### Database Connection Failed

```bash
# Test database connectivity
mysql -u cacti -p -h localhost cacti

# Check MariaDB is running
systemctl status mariadb

# Check config.php credentials match
grep database_ /var/www/html/cacti/include/config.php
```

### Graphs Not Generating

```bash
# Run poller manually to see errors
sudo -u cacti php /var/www/html/cacti/poller.php --force

# Check Cacti log
tail -50 /var/www/html/cacti/log/cacti.log

# Verify RRDTool is installed
which rrdtool
rrdtool --version

# Check cron is running
systemctl status crond    # RHEL
systemctl status cron     # Ubuntu
```

### Permission Errors

```bash
# Fix ownership
sudo chown -R cacti:apache /var/www/html/cacti/rra/ /var/www/html/cacti/log/ /var/www/html/cacti/cache/    # RHEL
sudo chown -R cacti:www-data /var/www/html/cacti/rra/ /var/www/html/cacti/log/ /var/www/html/cacti/cache/  # Ubuntu

# Fix permissions
sudo chmod -R 775 /var/www/html/cacti/rra/ /var/www/html/cacti/log/ /var/www/html/cacti/cache/
```

## Directory Structure

| Path | Purpose |
|------|---------|
| `/var/www/html/cacti/` | Web root |
| `/var/www/html/cacti/include/config.php` | Database configuration |
| `/var/www/html/cacti/rra/` | RRD data files |
| `/var/www/html/cacti/log/` | Application logs |
| `/var/www/html/cacti/cache/` | Cache files |
| `/var/www/html/cacti/scripts/` | Data collection scripts |
| `/var/www/html/cacti/resource/` | SNMP query XML files |
| `/usr/local/spine/etc/spine.conf` | Spine configuration |

## Quick Reference

| Action | Command |
|--------|---------|
| Run poller manually | `sudo -u cacti php /var/www/html/cacti/poller.php --force` |
| Check logs | `tail -f /var/www/html/cacti/log/cacti.log` |
| Rebuild poller cache | `sudo -u cacti php /var/www/html/cacti/cli/rebuild_poller_cache.php` |
| Test SNMP | `snmpwalk -v2c -c public localhost system` |
| Verify PHP modules | `php -m` |
| Restart services | `systemctl restart httpd mariadb` (RHEL) |
| Web interface | `http://<server-ip>/cacti` |
| Default login | `admin` / `admin` |
