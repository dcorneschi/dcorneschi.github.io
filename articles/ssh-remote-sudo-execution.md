# SSH Remote Sudo Execution

Running privileged commands on remote servers via SSH without an interactive session. The `sudo -S` flag reads the password from stdin, making it useful for scripting and automation.

## Basic Syntax

```bash
ssh user@server sudo -S command
```

The `-S` flag tells sudo to read the password from standard input instead of the terminal device.

## Common Examples

### System Information

```bash
# Check disk usage
ssh user@server sudo -S df -h

# View system logs
ssh user@server sudo -S tail -f /var/log/syslog

# Check running processes
ssh user@server sudo -S ps aux | grep nginx

# Check memory usage
ssh user@server sudo -S free -h

# View system uptime
ssh user@server sudo -S uptime
```

### Service Management

```bash
# Restart a service
ssh user@server sudo -S systemctl restart nginx

# Check service status
ssh user@server sudo -S systemctl status docker

# Start/stop services
ssh user@server sudo -S service apache2 start
ssh user@server sudo -S service apache2 stop

# Enable/disable services
ssh user@server sudo -S systemctl enable docker
ssh user@server sudo -S systemctl disable nginx
```

### File Operations

```bash
# Read protected files
ssh user@server sudo -S cat /etc/shadow

# Change file permissions
ssh user@server sudo -S chmod 644 /etc/nginx/nginx.conf

# Change file ownership
ssh user@server sudo -S chown www-data:www-data /var/www/html

# Create directories with proper permissions
ssh user@server sudo -S mkdir -p /opt/myapp
```

### Package Management

```bash
# Update packages (Ubuntu/Debian)
ssh user@server 'sudo -S apt update && sudo -S apt upgrade -y'

# Install packages
ssh user@server sudo -S apt install -y docker.io
ssh user@server sudo -S apt install -y nginx

# Remove packages
ssh user@server sudo -S apt remove nginx

# Clean package cache
ssh user@server sudo -S apt autoremove
ssh user@server sudo -S apt autoclean
```

### Network Operations

```bash
# Check network connections
ssh user@server sudo -S netstat -tulpn

# View iptables rules
ssh user@server sudo -S iptables -L

# Restart networking
ssh user@server sudo -S systemctl restart networking

# Check open ports
ssh user@server sudo -S ss -tulpn
```

## Password Automation

### Using echo (less secure)

```bash
echo "password" | ssh user@server sudo -S command
```

### Using here-string

```bash
ssh user@server 'echo "password" | sudo -S command'
```

### Using environment variable

```bash
export SUDO_PASS="your_password"
echo "$SUDO_PASS" | ssh user@server sudo -S command
```

## More Secure Approaches

### SSH Key Authentication + NOPASSWD sudo

Set up passwordless sudo for specific commands on the remote server:

```bash
# In /etc/sudoers on the remote server:
user ALL=(ALL) NOPASSWD: /usr/bin/systemctl, /usr/bin/apt, /usr/bin/docker
```

Then run without password prompts:

```bash
ssh user@server sudo systemctl restart nginx
ssh user@server sudo apt update
```

### Using expect for interactive password entry

```bash
#!/usr/bin/expect -f
spawn ssh user@server sudo -S command
expect "password:"
send "your_password\r"
interact
```

### SSH Config for connection multiplexing

```bash
# In ~/.ssh/config
Host myserver
    HostName server.example.com
    User myuser
    ControlMaster auto
    ControlPath ~/.ssh/control-%h-%p-%r
    ControlPersist 10m
```

## Kubernetes and Container Examples

```bash
# Check Docker containers
ssh user@k8s-node sudo -S docker ps

# View kubelet logs
ssh user@k8s-node sudo -S journalctl -u kubelet -f

# Restart kubelet service
ssh user@k8s-node sudo -S systemctl restart kubelet

# Check Docker daemon status
ssh user@k8s-node sudo -S systemctl status docker

# View container logs
ssh user@k8s-node sudo -S docker logs container_name

# Clean up Docker resources
ssh user@k8s-node sudo -S docker system prune -f
```

## Advanced Examples

### Multiple commands in sequence

```bash
ssh user@server 'sudo -S systemctl stop nginx && sudo -S systemctl start nginx'
```

### Command with output redirection

```bash
ssh user@server sudo -S dmesg > /tmp/remote_dmesg.log
```

### Running scripts remotely

```bash
ssh user@server 'sudo -S bash -s' < local_script.sh
```

### Checking system resources

```bash
# CPU usage
ssh user@server sudo -S top -bn1 | head -20

# Disk I/O
ssh user@server sudo -S iotop -bn1

# Network traffic
ssh user@server sudo -S iftop -t -s 10
```

## Best Practices

1. Use SSH keys instead of passwords when possible
2. Configure NOPASSWD sudo for specific commands you run frequently
3. Use connection multiplexing to avoid repeated authentication
4. Combine multiple commands to reduce SSH overhead
5. Use specific sudo rules instead of blanket NOPASSWD access
6. Log sudo activities for security auditing
7. Use jump hosts for accessing internal servers securely

## Security Considerations

- Never hardcode passwords in scripts
- Use SSH key-based authentication
- Limit sudo privileges to specific commands
- Enable sudo logging: `Defaults logfile=/var/log/sudo.log`
- Use SSH connection timeouts and limits
- Consider using tools like Ansible for complex remote operations

## Troubleshooting

| Issue | Fix |
|-------|-----|
| sudo prompts for password despite `-S` | Suppress the prompt: `sudo -S -p "" command` |
| No TTY available | Force pseudo-terminal allocation: `ssh -t user@server sudo command` |
| Connection debugging | Use verbose mode: `ssh -v user@server sudo -S command` |
| Password not being read | Ensure stdin is connected — pipe or here-string required with `-S` |
| `sudo: no tty present` | Use `-S` flag or allocate a TTY with `ssh -t` |
