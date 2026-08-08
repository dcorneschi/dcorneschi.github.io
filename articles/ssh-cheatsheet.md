<h1 align="center">SSH Cheatsheet</h1>

## Basic Connections

```bash
# Connect to a host
ssh user@host

# Connect on a specific port
ssh -p 2222 user@host

# Connect with a specific identity (key)
ssh -i ~/.ssh/id_ed25519 user@host

# Connect with verbose output (debugging)
ssh -v user@host
ssh -vv user@host      # more verbose
ssh -vvv user@host     # maximum verbosity

# Connect and run a command
ssh user@host 'ls -la /tmp'

# Connect with a pseudo-terminal (needed for interactive commands)
ssh -t user@host 'top'

# Force pseudo-terminal allocation (even when stdin is not a terminal)
ssh -tt user@host 'sudo command'

# Connect with X11 forwarding (GUI applications)
ssh -X user@host

# Trusted X11 forwarding (less secure, full access to display)
ssh -Y user@host

# Connect without executing a remote command (port forwarding only)
ssh -N user@host

# Go to background after authentication
ssh -f user@host -L 8080:localhost:80

# Go to background and don't execute a command
ssh -fN user@host

# Disable pseudo-terminal allocation
ssh -T user@host

# Connect using IPv4 only
ssh -4 user@host

# Connect using IPv6 only
ssh -6 user@host

# Use a specific config file
ssh -F ~/.ssh/config_alt user@host

# Suppress all warnings and diagnostic messages
ssh -q user@host

# Connection timeout (fail fast)
ssh -o ConnectTimeout=10 user@host

# Keep connection alive
ssh -o ServerAliveInterval=60 user@host

# Test SSH connectivity (e.g. GitHub)
ssh -T git@github.com

# Show SSH client version
ssh -V
```

## Remote Command Execution

```bash
# Run a single command
ssh user@host 'uptime'

# Run multiple commands
ssh user@host 'df -h && free -m'

# Run a command that needs a TTY (sudo, top, etc.)
ssh -t user@host 'sudo systemctl restart nginx'

# Run a local script on the remote server
ssh user@host 'bash -s' < local_script.sh

# Run a local script with arguments
ssh user@host 'bash -s' -- arg1 arg2 < local_script.sh

# Compress and pull remote files (tar over SSH)
ssh user@host "tar czf - /var/log/nginx/" | tar xzf - -C /tmp/

# Push local directory to remote (tar over SSH)
tar czf - /local/dir/ | ssh user@host "tar xzf - -C /remote/path/"

# Pipe remote output through local commands
ssh user@host "cat /var/log/syslog" | gzip > remote-syslog.gz

# Database backup over SSH
pg_dumpall | ssh user@backup-server "cat > /backups/db-$(date +%F).sql"

# Copy remote output to local file
ssh user@host 'cat /etc/nginx/nginx.conf' > nginx.conf.bak
```

## SSH Config File

Location: `~/.ssh/config` (user) or `/etc/ssh/ssh_config` (system)

```
Host dev
    HostName 192.168.1.100
    User admin
    Port 2222
    IdentityFile ~/.ssh/id_ed25519

# Multiple hosts with same settings
Host server1 server2 server3
    User admin
    Port 22
    IdentityFile ~/.ssh/admin_key

# Wildcard host patterns
Host *.example.com
    User deploy
    IdentityFile ~/.ssh/deploy_key
    ForwardAgent no

# IP range wildcard
Host 192.168.1.*
    StrictHostKeyChecking no
    Port 22
    UserKnownHostsFile /dev/null
    IdentityFile ~/.ssh/id_rsa

# Jump host (bastion/proxy)
Host target
    HostName 10.0.1.100
    User myuser
    ProxyJump bastion

Host bastion
    HostName bastion.example.com
    User jumpuser

# Alternative jump host syntax (legacy)
Host target-legacy
    HostName 10.0.1.100
    User myuser
    ProxyCommand ssh -W %h:%p bastion

# Connection multiplexing
Host *
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600

# Forward SSH agent (use carefully)
Host trusted-server
    ForwardAgent yes

# Disable host key checking (testing only!)
Host testserver
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

# Global defaults
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes
```

### Common Config Options

| Option | Description |
|--------|-------------|
| `Host` | Pattern to match hostnames |
| `HostName` | Actual hostname or IP to connect to |
| `User` | Default username |
| `Port` | Default port |
| `IdentityFile` | Path to private key |
| `IdentitiesOnly` | Only use specified keys (ignore agent) |
| `ForwardAgent` | Forward SSH agent to remote host |
| `ForwardX11` | Forward X11 display |
| `ProxyJump` | Jump through a bastion host |
| `ServerAliveInterval` | Keepalive interval in seconds |
| `ServerAliveCountMax` | Max missed keepalives before disconnect |
| `StrictHostKeyChecking` | Host key verification policy |
| `UserKnownHostsFile` | Path to known_hosts file |
| `LogLevel` | Logging verbosity |
| `Compression` | Enable compression |
| `RequestTTY` | TTY allocation policy (no/yes/force/auto) |
| `AddKeysToAgent` | Auto-add keys to agent |

## Port Forwarding

### Local Forward (-L)

Forward a local port to a remote destination through the SSH tunnel:

```bash
# Access remote_host:3306 via localhost:3306
ssh -L 3306:localhost:3306 user@host

# Bind to a specific local address
ssh -L 127.0.0.1:8080:internal-server:80 user@bastion

# Forward to a third host through the SSH server
ssh -L 9090:database.internal:5432 user@jumphost

# Multiple forwards in one connection
ssh -L 8080:web:80 -L 8443:web:443 user@host

# Background tunnel (no shell)
ssh -N -L 8080:localhost:80 user@host

# Background tunnel forked to background
ssh -f -N -L 8080:localhost:80 user@host
```

```
Local Machine          SSH Server            Target
localhost:8080 ──────▶ host ──────────────▶ web:80
```

### Remote Forward (-R)

Forward a remote port back to a local destination:

```bash
# Make local port 3000 accessible on remote as port 9090
ssh -R 9090:localhost:3000 user@host

# Bind to all interfaces on the remote (requires GatewayPorts yes on server)
ssh -R 0.0.0.0:9090:localhost:3000 user@host

# Forward to a third host
ssh -R 8080:internal-service:80 user@host

# Reverse tunnel for remote access to local SSH
# (access your machine from remote via port 40099)
ssh -R 40099:localhost:22 user@remote-server
# Then from remote-server: ssh -p 40099 localhost
```

```
Remote Machine         SSH Server            Local Machine
host:9090 ◀────────── tunnel ◀──────────── localhost:3000
```

### Dynamic Forward (-D) — SOCKS Proxy

```bash
# Create a SOCKS5 proxy on localhost:1080
ssh -D 1080 user@host

# Bind to a specific address
ssh -D 127.0.0.1:1080 user@host
```

Then configure your browser/application to use `localhost:1080` as a SOCKS5 proxy.

### Forward Options

| Flag | Description |
|------|-------------|
| `-L [bind:]port:host:hostport` | Local forward |
| `-R [bind:]port:host:hostport` | Remote forward |
| `-D [bind:]port` | Dynamic (SOCKS) forward |
| `-g` | Allow remote hosts to connect to local forwarded ports |
| `-N` | No remote command (just forward) |
| `-f` | Background after authentication |

## Jump Hosts / Bastion

```bash
# ProxyJump (preferred, OpenSSH 7.3+)
ssh -J bastion user@internal-host

# Multiple jumps
ssh -J bastion1,bastion2 user@internal-host

# Equivalent in config
# Host internal
#     ProxyJump bastion

# Legacy: ProxyCommand with nc
ssh -o ProxyCommand="ssh -W %h:%p bastion" user@internal-host
```

## Key Management

### Generate Keys

```bash
# Ed25519 (recommended)
ssh-keygen -t ed25519 -C "user@host"

# Ed25519 with custom filename
ssh-keygen -t ed25519 -f ~/.ssh/id_project -C "project key"

# RSA 4096-bit (wider compatibility)
ssh-keygen -t rsa -b 4096 -C "user@host"

# ECDSA
ssh-keygen -t ecdsa -b 521 -C "user@host"

# Generate without passphrase (automation only)
ssh-keygen -t ed25519 -f ~/.ssh/id_auto -N ""
```

### Key Operations

```bash
# Change passphrase on existing key
ssh-keygen -p -f ~/.ssh/id_ed25519

# View key fingerprint
ssh-keygen -lf ~/.ssh/id_ed25519.pub

# View fingerprint in different format
ssh-keygen -lf ~/.ssh/id_ed25519.pub -E md5

# Convert between key formats
ssh-keygen -e -f ~/.ssh/id_rsa.pub -m RFC4716    # OpenSSH → RFC4716
ssh-keygen -i -f key.pub -m RFC4716              # RFC4716 → OpenSSH

# Extract public key from private key
ssh-keygen -y -f ~/.ssh/id_ed25519 > ~/.ssh/id_ed25519.pub

# Remove a host from known_hosts
ssh-keygen -R hostname
ssh-keygen -R [hostname]:port

# Search for a host in known_hosts
ssh-keygen -F hostname

# Generate a visual fingerprint (ASCII art)
ssh-keygen -lv -f ~/.ssh/id_ed25519.pub
```

### Copy Public Key to Remote Host

```bash
# Using ssh-copy-id (preferred)
ssh-copy-id user@host
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host
ssh-copy-id -p 2222 user@host

# Manual method
cat ~/.ssh/id_ed25519.pub | ssh user@host 'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

### Copy Public Key to Clipboard

```bash
# macOS
cat ~/.ssh/id_ed25519.pub | pbcopy

# Linux (xclip)
cat ~/.ssh/id_ed25519.pub | xclip -selection clipboard

# Linux (xsel)
cat ~/.ssh/id_ed25519.pub | xsel --clipboard

# Windows (Git Bash / WSL)
cat ~/.ssh/id_ed25519.pub | clip
```

### View and List Keys

```bash
# View your public key
cat ~/.ssh/id_ed25519.pub

# List all SSH key files
ls -la ~/.ssh/

# List all public keys
ls ~/.ssh/*.pub
```

### Copy Keys Between Machines

```bash
# Copy entire .ssh directory to another machine
scp -r ~/.ssh/ user@host:~/

# Copy specific key files
scp ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub user@host:~/.ssh/

# Remember to fix permissions on the target
ssh user@host 'chmod 700 ~/.ssh && chmod 600 ~/.ssh/id_ed25519 && chmod 644 ~/.ssh/id_ed25519.pub'
```

## SSH Agent

```bash
# Start the agent (Bash/Zsh)
eval "$(ssh-agent -s)"

# Start the agent (Fish)
eval (ssh-agent -c)

# Add a key
ssh-add ~/.ssh/id_ed25519

# Add with expiry (auto-remove after time)
ssh-add -t 3600 ~/.ssh/id_ed25519     # 1 hour

# List loaded keys (fingerprints)
ssh-add -l

# List loaded keys (full public key)
ssh-add -L

# Remove a specific key
ssh-add -d ~/.ssh/id_ed25519

# Remove all keys
ssh-add -D

# Lock the agent (requires password to unlock)
ssh-add -x

# Unlock the agent
ssh-add -X

# Kill the agent
ssh-agent -k

# Check if agent is running
echo $SSH_AUTH_SOCK
ps aux | grep ssh-agent

# Set SSH agent socket manually (if needed)
export SSH_AUTH_SOCK=/path/to/agent/socket
```

### Agent Forwarding

```bash
# Forward agent to remote host
ssh -A user@host

# Check if agent is forwarded (on remote)
echo $SSH_AUTH_SOCK
ssh-add -l
```

> **Security warning:** Agent forwarding allows anyone with root access on the remote host to use your agent. Prefer `ProxyJump` over agent forwarding when possible.

## File Transfer

### SCP

```bash
# Copy file to remote
scp file.txt user@host:/remote/path/

# Copy file from remote
scp user@host:/remote/file.txt ./local/

# Copy directory recursively
scp -r ./local-dir user@host:/remote/path/

# Use specific port
scp -P 2222 file.txt user@host:/tmp/

# Preserve modification times and permissions
scp -p file.txt user@host:/tmp/

# Use specific key
scp -i ~/.ssh/id_ed25519 file.txt user@host:/tmp/

# Limit bandwidth (in Kbit/s)
scp -l 1000 large-file.tar.gz user@host:/tmp/

# Enable compression during transfer
scp -C large-file.tar.gz user@host:/tmp/

# Copy between two remote hosts
scp user1@host1:/file user2@host2:/path/
```

### Rsync over SSH

More efficient than scp for large or incremental transfers:

```bash
# Sync local directory to remote
rsync -avz -e ssh local_dir/ user@host:/remote/dir/

# Sync remote to local
rsync -avz -e ssh user@host:/remote/dir/ local_dir/

# Use specific port
rsync -avz -e "ssh -p 2222" local_dir/ user@host:/remote/dir/

# Use specific key
rsync -avz -e "ssh -i ~/.ssh/id_ed25519" local_dir/ user@host:/remote/dir/

# Dry run (show what would be transferred)
rsync -avzn -e ssh local_dir/ user@host:/remote/dir/

# Delete remote files that don't exist locally
rsync -avz --delete -e ssh local_dir/ user@host:/remote/dir/

# Exclude patterns
rsync -avz --exclude='*.log' --exclude='.git' -e ssh local_dir/ user@host:/remote/dir/

# Limit bandwidth (in KB/s)
rsync -avz --bwlimit=1000 -e ssh local_dir/ user@host:/remote/dir/
```

### SFTP

```bash
# Start interactive SFTP session
sftp user@host

# Use specific port
sftp -P 2222 user@host

# Start in specific remote directory
sftp user@host:/var/log

# Batch mode (non-interactive)
sftp -b commands.txt user@host
```

Common SFTP commands:

| Command | Description |
|---------|-------------|
| `ls`, `lls` | List remote / local files |
| `cd`, `lcd` | Change remote / local directory |
| `pwd`, `lpwd` | Print remote / local working directory |
| `get file` | Download file |
| `get -r dir` | Download directory recursively |
| `put file` | Upload file |
| `put -r dir` | Upload directory recursively |
| `mkdir dir` | Create remote directory |
| `rm file` | Delete remote file |
| `rmdir dir` | Delete remote directory |
| `chmod 644 file` | Change remote file permissions |
| `chown uid file` | Change remote file owner |
| `rename old new` | Rename remote file |
| `df -h` | Remote disk usage |
| `!command` | Run local shell command |
| `exit` / `bye` | Quit SFTP |

## Escape Sequences

While in an SSH session, press `Enter` then the escape character (default `~`):

| Sequence | Description |
|----------|-------------|
| `~.` | Disconnect (kill frozen session) |
| `~^Z` | Suspend SSH (background the client) |
| `~#` | List forwarded connections |
| `~&` | Background SSH when waiting for connections to close |
| `~?` | Show available escape sequences |
| `~B` | Send BREAK to remote |
| `~C` | Open command line (add forwards on the fly) |
| `~R` | Request rekeying |
| `~V` / `~v` | Increase / decrease verbosity |
| `~~` | Send a literal tilde character |

### Adding Forwards on the Fly

Press `Enter`, then `~C` to open the SSH command line:

```
ssh> -L 8080:localhost:80      # add local forward
ssh> -R 9090:localhost:3000    # add remote forward
ssh> -D 1080                   # add dynamic forward
ssh> -KL 8080                  # cancel local forward on port 8080
ssh> -KR 9090                  # cancel remote forward on port 9090
```

## Known Hosts

```bash
# Location
~/.ssh/known_hosts           # user-level
/etc/ssh/ssh_known_hosts     # system-level

# Remove a host entry
ssh-keygen -R hostname

# Remove host on non-standard port
ssh-keygen -R [hostname]:2222

# Hash all hostnames in known_hosts (privacy)
ssh-keygen -H -f ~/.ssh/known_hosts

# Connect without checking host key (NOT recommended for production)
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null user@host

# Accept new keys automatically but reject changed keys
ssh -o StrictHostKeyChecking=accept-new user@host

# View server's host key before connecting
ssh-keyscan -t ed25519 hostname
ssh-keyscan -p 2222 hostname

# Add a host key to known_hosts without connecting
ssh-keyscan -H hostname >> ~/.ssh/known_hosts
```

## Tunneling and Proxying

### SSH as a SOCKS Proxy

```bash
# Create SOCKS proxy
ssh -D 1080 -fN user@host

# Use with curl
curl --socks5 localhost:1080 http://example.com

# Use with git
git -c http.proxy=socks5://localhost:1080 clone https://repo.example.com/project.git
```

### Tunnel a TCP Connection

```bash
# Direct TCP connection through SSH (OpenSSH 5.4+)
ssh -W target-host:22 user@bastion

# In config (equivalent to ProxyJump)
# Host target
#     ProxyCommand ssh -W %h:%p bastion
```

### VPN over SSH (tun/tap)

```bash
# Requires root on both sides and PermitTunnel in sshd_config
sudo ssh -w 0:0 root@remote-host

# Point-to-point (layer 3)
ssh -o Tunnel=point-to-point -w 0:0 root@host

# Ethernet (layer 2)
ssh -o Tunnel=ethernet -w 0:0 root@host
```

### SSH Through an HTTP Proxy

```bash
# Connect through an HTTP CONNECT proxy using netcat
ssh -o ProxyCommand="nc -X connect -x proxyhost:8080 %h %p" user@host

# In config
# Host target
#     ProxyCommand nc -X connect -x proxyhost:8080 %h %p
```

### VNC Tunnel

```bash
# Forward VNC (port 5901) through SSH for encrypted remote desktop
ssh -L 5901:localhost:5901 -N -f user@host

# Then connect VNC client to localhost:5901
```

## Multiplexing (ControlMaster)

```bash
# Start a master connection in the background
ssh -fN -o ControlMaster=yes -o ControlPath=~/.ssh/master-%r@%h:%p user@host

# Reuse existing connection
ssh -o ControlMaster=auto -o ControlPath=~/.ssh/master-%r@%h:%p user@host

# Check master status
ssh -O check -o ControlPath=~/.ssh/master-%r@%h:%p user@host

# Gracefully exit master
ssh -O exit -o ControlPath=~/.ssh/master-%r@%h:%p user@host

# Stop accepting new sessions
ssh -O stop -o ControlPath=~/.ssh/master-%r@%h:%p user@host
```

See [SSH ControlMaster](articles/ssh-controlmaster.md) for a detailed guide.

## Connection Management

```bash
# Kill backgrounded SSH tunnel
jobs           # list background jobs
fg %1          # bring job 1 to foreground
kill %1        # kill job 1

# Find and kill SSH tunnels
ps aux | grep "ssh -f"
kill <pid>

# List all SSH connections
ss -tnp | grep ssh
```

## Security and Debugging

```bash
# Debug connection issues (increasing verbosity)
ssh -v user@host
ssh -o LogLevel=DEBUG3 user@host

# Test with specific cipher
ssh -c aes256-gcm@openssh.com user@host

# Test with specific key exchange algorithm
ssh -o KexAlgorithms=curve25519-sha256 user@host

# Force only public key authentication
ssh -o PreferredAuthentications=publickey user@host

# Disable password authentication for this connection
ssh -o PasswordAuthentication=no user@host

# Check remote server's host key fingerprint
ssh-keyscan hostname
ssh-keyscan -t ed25519 hostname
ssh-keyscan -p 2222 hostname

# Check supported algorithms in local sshd
sshd -T | grep ^"\(ciphers\|macs\|kexalgorithms\|hostkey\)"

# Scan remote server's supported algorithms
nmap --script ssh2-enum-algos -sV -p 22 192.168.50.2
```

### Determine Key Strength

```bash
# Check strength of all private keys
for keyfile in ~/.ssh/id_*; do ssh-keygen -lf "${keyfile}"; done | uniq

# Check strength of all authorized public keys
ssh-keygen -lf ~/.ssh/authorized_keys
```

### Compare SSH Configuration Between Servers

```bash
# Side-by-side comparison of supported algorithms on two servers
sdiff -bW <(nmap --script ssh2-enum-algos -sV -p 22 192.168.50.2) <(nmap --script ssh2-enum-algos -sV -p 22 192.168.50.3)
```

### Compare Remote and Local Files

```bash
# Diff a remote file against a local file
ssh user@host "cat /etc/fstab" | diff - /etc/fstab

# Diff two remote files
diff <(ssh user@host1 "cat /etc/hosts") <(ssh user@host2 "cat /etc/hosts")
```

## Useful SSH Utilities

```bash
# autossh — auto-reconnecting SSH tunnels
# Monitors the connection and restarts it if it drops
autossh -M 20000 -L 8080:localhost:80 user@host
autossh -M 0 -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -fN -L 8080:localhost:80 user@host

# sshpass — SSH with password (scripting/automation only)
sshpass -p 'password' ssh -o StrictHostKeyChecking=no user@host
sshpass -p 'password' scp file.txt user@host:/tmp/
# Install: apt-get install sshpass / brew install sshpass

# sshuttle — VPN-like tunnel over SSH (no server config needed)
sshuttle -r user@host 10.0.0.0/8 192.168.0.0/16
sshuttle -r user@host 0/0    # route ALL traffic through the tunnel

# sshfs — mount remote directory as local folder
sshfs user@host:/remote/path /local/mountpoint
sshfs -p 2222 user@host:/remote/path /local/mountpoint
fusermount -u /local/mountpoint    # unmount (Linux)
umount /local/mountpoint           # unmount (macOS)

# ngrok — expose local port to the internet (reverse tunnel service)
ngrok http 8080

# SSH with screen/tmux (attach or create session)
ssh -t user@host 'screen -DR'
ssh -t user@host 'tmux attach || tmux new'

# SSH with sudo
ssh -t user@host 'sudo command'

# Batch SSH operations (parallel-ssh / pssh)
parallel-ssh -h hosts.txt -l username "uptime"
parallel-ssh -h hosts.txt -l username -i "df -h"    # inline output

# Mute "Warning: Permanently added" message
ssh -o LogLevel=error user@host

# Avoid trying all identity files (use only specified key)
ssh -o IdentitiesOnly=yes -i ~/.ssh/specific_key user@host

# Generate key pair non-interactively (no prompts)
ssh-keygen -t rsa -b 4096 -f /tmp/sshkey -N "" -q
```

### expect Script for Automated Password Input

```bash
#!/usr/bin/expect
set timeout 20
set command "cat /etc/hosts"
set user "admin"
set password "secret"
set ip "192.168.50.10"
spawn ssh -o StrictHostKeyChecking=no $user@$ip "$command"
expect "*password:*"
send "$password\r"
expect eof;
```

## Server-Side (sshd)

### Common sshd_config Options

```bash
# /etc/ssh/sshd_config

Port 22
ListenAddress 0.0.0.0

# Authentication
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3

# Security
PermitEmptyPasswords no
X11Forwarding no
AllowTcpForwarding yes
GatewayPorts no
MaxSessions 10
LoginGraceTime 30

# Keepalive
ClientAliveInterval 300
ClientAliveCountMax 2

# Access control
AllowUsers deploy admin
AllowGroups sshusers
DenyUsers guest

# Logging
LogLevel INFO
SyslogFacility AUTH

# Subsystem
Subsystem sftp /usr/lib/openssh/sftp-server
```

### sshd Management

```bash
# Test config syntax
sshd -t

# Test config and print effective configuration
sshd -T

# Reload config without restarting (systemd)
sudo systemctl reload sshd

# Restart SSH service
sudo systemctl restart sshd     # systemd (RHEL/CentOS/modern)
sudo service ssh restart        # SysVinit (older Debian/Ubuntu)

# View active connections
ss -tnp | grep :22
who

# View auth logs
journalctl -u sshd -f
sudo tail -f /var/log/auth.log      # Debian/Ubuntu
sudo tail -f /var/log/secure        # RHEL/CentOS

# Disable password auth via sed (quick hardening)
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config

# Disable root login via sed
sudo sed -i 's/^PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
```

### Parse SSH Log Files

```bash
# SSH service started
grep -R "sshd.*Server listening" /var/log/auth.log

# SSH service stopped
grep -R "ssh.*Received signal 15" /var/log/auth.log

# Successful login by public key
grep -R "sshd.*Accepted publickey for" /var/log/auth.log

# Successful login by password
grep -R "sshd.*Accepted password for" /var/log/auth.log

# Failed login attempts
grep -R "sshd.*Failed password for invalid user" /var/log/auth.log

# Possible break-in attempts
grep -R "sshd.*POSSIBLE BREAK-IN ATTEMPT!" /var/log/auth.log

# Port scanning attempts
grep -R "sshd.*Bad protocol version identification" /var/log/auth.log

# Session closed (logout)
grep -R "sshd.*pam_unix(sshd:session): session closed for" /var/log/auth.log
```

### Brute Force Protection

```bash
# Install fail2ban
sudo apt-get install fail2ban       # Debian/Ubuntu
sudo yum install fail2ban           # RHEL/CentOS

# Check fail2ban status for SSH
sudo fail2ban-client status sshd

# Unban an IP
sudo fail2ban-client set sshd unbanip 192.168.1.100
```

### SELinux Port Change (RHEL/Rocky/CentOS)

When changing the SSH port on SELinux-enabled systems:

```bash
# Allow the new port in SELinux
sudo semanage port -a -t ssh_port_t -p tcp 33000

# Verify
sudo semanage port -l | grep ssh
```

## SSH Certificate Authentication

SSH certificates scale better than individual keys. Instead of distributing public keys to every server, sign user keys with a Certificate Authority (CA) and configure servers to trust that CA.

```bash
# Generate a CA key pair
ssh-keygen -t ed25519 -f ~/.ssh/ssh_ca -N "" -C "SSH CA"

# Sign a user's public key (valid for 52 weeks, principals: admin,deploy)
ssh-keygen -s ~/.ssh/ssh_ca -I "user-cert" -n admin,deploy -V +52w ~/.ssh/id_ed25519.pub

# Inspect a certificate
ssh-keygen -Lf ~/.ssh/id_ed25519-cert.pub

# Sign a host key (for server identity)
ssh-keygen -s ~/.ssh/ssh_ca -I "host-cert" -h -n server.example.com -V +52w /etc/ssh/ssh_host_ed25519_key.pub
```

Server-side configuration to trust the CA:

```bash
# Copy CA public key to the server
sudo cp ssh_ca.pub /etc/ssh/ca.pub

# Add to /etc/ssh/sshd_config
TrustedUserCAKeys /etc/ssh/ca.pub

# Restart sshd
sudo systemctl restart sshd
```

Any user presenting a key signed by this CA can now log in without their public key being in `authorized_keys`.

## Permissions Reference

SSH is strict about file permissions. Incorrect permissions will cause silent auth failures.

| Path | Permission | Description |
|------|-----------|-------------|
| `~/.ssh/` | `700` | SSH directory |
| `~/.ssh/config` | `600` | Client config |
| `~/.ssh/id_*` | `600` | Private keys |
| `~/.ssh/id_*.pub` | `644` | Public keys |
| `~/.ssh/authorized_keys` | `600` | Authorized keys |
| `~/.ssh/known_hosts` | `600` | Known hosts |
| `~` (home dir) | `755` or stricter | Home directory (must not be group/world-writable) |

```bash
# Fix permissions in one go
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_* ~/.ssh/authorized_keys ~/.ssh/config 2>/dev/null
chmod 644 ~/.ssh/*.pub 2>/dev/null
```

## Common Flags Reference

| Flag | Description |
|------|-------------|
| `-4` / `-6` | Force IPv4 / IPv6 |
| `-A` / `-a` | Enable / disable agent forwarding |
| `-C` | Enable compression |
| `-D [bind:]port` | Dynamic (SOCKS) forward |
| `-E log_file` | Append debug logs to file |
| `-F configfile` | Use alternate config file |
| `-f` | Background after auth |
| `-G` | Print config and exit |
| `-g` | Allow remote hosts to connect to forwarded ports |
| `-i identity_file` | Specify private key |
| `-J destination` | ProxyJump (jump host) |
| `-K` | Enable GSSAPI auth and forwarding |
| `-L [bind:]port:host:port` | Local forward |
| `-l login_name` | Specify username |
| `-M` | Master mode for multiplexing |
| `-N` | No remote command |
| `-n` | Redirect stdin from /dev/null |
| `-O ctl_cmd` | Control multiplexing master (check/exit/stop/forward/cancel) |
| `-o option` | Pass config option |
| `-p port` | Specify port |
| `-q` | Quiet mode |
| `-R [bind:]port:host:port` | Remote forward |
| `-S ctl_path` | Specify control socket path |
| `-s` | Request subsystem invocation |
| `-T` | Disable pseudo-terminal |
| `-t` | Force pseudo-terminal allocation |
| `-V` | Print version and exit |
| `-v` | Verbose mode (up to `-vvv`) |
| `-W host:port` | Direct TCP forwarding |
| `-w local_tun:remote_tun` | Request tunnel device forwarding |
| `-X` / `-x` | Enable / disable X11 forwarding |
| `-Y` | Trusted X11 forwarding |

## Troubleshooting

### Connection Refused

```bash
# Check if sshd is running
systemctl status sshd

# Check if port is open
ss -tlnp | grep :22

# Check firewall
sudo iptables -L -n | grep 22
sudo ufw status
```

### Permission Denied (publickey)

```bash
# Verbose connect to see which keys are tried
ssh -vvv user@host

# Check authorized_keys permissions on remote
ls -la ~/.ssh/
ls -la ~/.ssh/authorized_keys

# Check if key is loaded in agent
ssh-add -l

# Check sshd logs for details
journalctl -u sshd --since "5 minutes ago"
```

### Host Key Changed

```bash
# Remove old host key
ssh-keygen -R hostname

# If port is non-standard
ssh-keygen -R [hostname]:port
```

### Broken Pipe / Connection Dropped

Add keepalives in `~/.ssh/config`:

```
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    TCPKeepAlive yes
```

### Frozen Session

Press `Enter` then `~.` to force disconnect a hung session.

## Tips

- Use SSH config file (`~/.ssh/config`) to save connection details and avoid typing long commands
- Always use strong SSH keys (Ed25519 preferred, RSA 4096-bit for compatibility)
- Enable SSH agent forwarding carefully — prefer `ProxyJump` for accessing hosts behind bastions
- Use connection multiplexing (`ControlMaster`) to speed up repeated connections to the same host
- Regularly rotate SSH keys and remove unused keys from `authorized_keys`
- Monitor SSH logs (`/var/log/auth.log` or `journalctl -u sshd`) for unauthorized access attempts
- Consider using SSH certificates for large deployments instead of managing individual keys
- Set `PermitRootLogin no` and `PasswordAuthentication no` on servers exposed to the internet
- Use `autossh` for persistent tunnels that need to survive network interruptions
- Use `~.` to escape from frozen sessions instead of closing the terminal

## Links

- [sshcheck.com](https://sshcheck.com) — online SSH server security scanner
- [OpenSSH Wikibook](https://en.wikibooks.org/wiki/OpenSSH)
