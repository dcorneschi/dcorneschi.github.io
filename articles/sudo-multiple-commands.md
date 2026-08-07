# Running Multiple Commands with sudo

`sudo` elevates a single command to run with root (or another user's) privileges. When you need to run several commands as root, there are a few approaches — each with different trade-offs for readability, safety, and scope.

## The Problem

Running commands one at a time gets repetitive:

```bash
sudo apt update
sudo apt install nginx
sudo systemctl enable nginx
```

There are cleaner ways to handle this.

## Approach 1 — Semicolons or `&&` Inside a Subshell

```bash
sudo bash -c "apt update; apt install -y nginx; systemctl enable nginx"
```

Using `&&` instead stops execution on failure:

```bash
sudo bash -c "apt update && apt install -y nginx && systemctl enable nginx"
```

`bash -c` launches a subshell as root and runs the entire string inside it. All commands inherit root privileges for the lifetime of that subshell.

> **Note:** The quoting matters. Wrap the entire command string in double quotes. If any individual command itself uses double quotes, escape them (`\"`) or switch to single quotes for the outer wrapper.

## Approach 2 — Heredoc (Multiple Lines, Readable)

```bash
sudo bash <<'EOF'
apt update
apt install -y nginx
systemctl enable nginx
systemctl start nginx
EOF
```

The `<<'EOF'` (quoted heredoc) prevents variable expansion in the block, so `$variables` are passed literally to the subshell. Use `<<EOF` (unquoted) if you want variables to expand in the calling shell before being handed to sudo.

```bash
PACKAGE="nginx"

# Variables expand before sudo sees them
sudo bash <<EOF
apt install -y $PACKAGE
systemctl enable $PACKAGE
EOF
```

## Approach 3 — `sudo -s` (Interactive Root Shell)

```bash
sudo -s
```

Starts an interactive root shell using the `SHELL` environment variable. You run multiple commands manually and exit when done.

```bash
sudo -s
# Now running as root
apt update
apt install -y nginx
exit
```

> **Note:** `sudo -s` does not reset the environment. Use `sudo -i` if you need a full login shell with root's `PATH`, `HOME`, and profile scripts sourced.

## Approach 4 — `sudo -i` (Full Login Shell)

```bash
sudo -i
```

Starts a login shell as root, sourcing `/root/.profile` and `/root/.bashrc`. This is the closest equivalent to logging in directly as root.

Useful when environment variables like `PATH` or `HOME` pointing to root's directories are needed for the commands to work correctly.

## Approach 5 — Script File

For anything beyond a few commands, a script is cleaner:

```bash
#!/bin/bash
set -euo pipefail

apt update
apt install -y nginx
systemctl enable nginx
systemctl start nginx
```

```bash
sudo bash setup.sh
```

Or make the script executable and call sudo directly:

```bash
chmod +x setup.sh
sudo ./setup.sh
```

> **Note:** `sudo` looks at the file's owner, not the caller. The script executes as root because you prefixed it with `sudo`, not because of the script's own permissions.

## Approach 6 — Run as a Specific User

`sudo` is not limited to root. Use `-u` to run multiple commands as any user on the system:

```bash
sudo -u postgres -- sh -c 'pg_dumpall > /tmp/backup.sql && gzip /tmp/backup.sql'
sudo -u mysql -- sh -c '/home/mysql/backup.sh; /home/mysql/mirror.py'
```

The `--` signals the end of sudo's own option parsing, so anything after it is passed directly to the shell. This avoids ambiguity when the command string starts with something that looks like a flag.

You can use `--` with root as well:

```bash
sudo -- sh -c 'apt update && apt upgrade'
```

## Approach 7 — `sudo su`

```bash
sudo su
```

Switches to the root user by running `su` under sudo. Functionally similar to `sudo -i` but slightly different: `su` reads root's login shell from `/etc/passwd`, while `sudo -i` uses the `SHELL` variable or the sudoers-configured shell.

For scripting, prefer `sudo bash -c` or a script file. `sudo su` is more of an interactive convenience.

## Comparison

| Approach | Scope | Best for |
|----------|-------|----------|
| `sudo bash -c "..."` | Subshell | Short, inline multi-command sequences |
| `sudo bash <<'EOF'` | Subshell | Multi-line blocks without a separate file |
| `sudo -s` | Interactive shell | Quick interactive work as root |
| `sudo -i` | Login shell | Root work that depends on root's environment |
| `sudo bash script.sh` | Script file | Repeatable or complex automation |
| `sudo -u user -- sh -c '...'` | Subshell | Running as a specific non-root user |
| `sudo su` | Login shell | Interactive; equivalent to `sudo -i` in most cases |

## Pipes and Redirection

Pipes and redirections are interpreted by the shell before `sudo` runs, so the elevated privileges don't apply to them. This is a common source of unexpected "permission denied" errors.

### Why `sudo echo "text" >> file` Fails

```bash
# WRONG — only echo runs as root, >> runs as your user
sudo echo "some config" >> /etc/hosts
# Error: Permission denied
```

`sudo` only elevates the `echo` command. The shell processes `>>` before `sudo` even runs, using your unprivileged user. Your user can't write to `/etc/hosts`, so it fails.

### Solution 1 — `tee` (recommended)

`tee` runs as root and handles the file writing:

```bash
# Append to file
echo "some config" | sudo tee -a /etc/hosts

# Overwrite file
echo "some config" | sudo tee /etc/hosts

# Suppress stdout echo
echo "some config" | sudo tee -a /etc/hosts > /dev/null
```

### Solution 2 — `bash -c` with quoted redirection

Wrap the entire command including the redirection inside a root shell:

```bash
sudo bash -c 'echo "some config" >> /etc/hosts'

# Multi-line with heredoc
sudo bash -c 'cat >> /etc/hosts << EOF
192.168.1.100 myserver.local
192.168.1.101 database.local
EOF'
```

### Solution 3 — `sh -c`

Same as `bash -c`, using the system's `sh`:

```bash
sudo sh -c 'echo "some config" >> /etc/hosts'
```

### Quick Reference

| ❌ Wrong | ✅ Correct |
|-------|---------|
| `sudo echo "text" >> file` | `echo "text" \| sudo tee -a file` |
| `sudo echo "text" > file` | `echo "text" \| sudo tee file` |
| `sudo cat content >> file` | `sudo bash -c 'cat content >> file'` |
| `sudo command > file` | `sudo bash -c 'command > file'` |

### Best Practices

- Use `tee` for simple single-line appends
- Use `bash -c` for complex operations or multiple redirections
- Use heredocs for multi-line content
- Test with `echo` first to verify the command structure before writing to system files
- Always quote the content inside `bash -c`

### Why `tee` Is Preferred

1. **More portable** — works across different shells (bash, sh, zsh)
2. **Cleaner syntax** — easier to read and maintain
3. **Better error handling** — tee reports errors more clearly
4. **Standard practice** — widely used in automation scripts
5. **Append flag** — `-a` works like `>>` for appending

### Appending Multiple Lines with Brace Grouping

```bash
{
  echo "line1"
  echo "line2"
  echo "line3"
} | sudo tee -a /path/to/file

# Silent output (suppress tee echo)
{
  echo "[Service]"
  echo "Restart=always"
  echo "RestartSec=5"
} | sudo tee -a /etc/systemd/system/myapp.service > /dev/null
```

### Verifying the Write

Always confirm the file was written correctly:

```bash
# Check the last few lines
sudo tail -5 /etc/ssh/sshd_config

# Validate config syntax (for services that support it)
sudo sshd -t        # SSH
sudo nginx -t       # nginx
sudo visudo -c      # sudoers
sudo named-checkconf # BIND
```

### Real-World Examples

#### /etc/hosts

```bash
# Single entry
echo "192.168.1.100 myserver" | sudo tee -a /etc/hosts

# Multiple entries
sudo bash -c 'cat >> /etc/hosts << EOF
192.168.1.100 myserver.local myserver
192.168.1.101 database.local database
192.168.1.102 cache.local cache
EOF'
```

#### SSH configuration

```bash
echo "PasswordAuthentication yes" | sudo tee -a /etc/ssh/sshd_config
sudo bash -c 'echo "PermitRootLogin yes" >> /etc/ssh/sshd_config'
```

#### DNS — /etc/resolv.conf

```bash
echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf
sudo bash -c 'echo "nameserver 8.8.8.8" >> /etc/resolv.conf'
```

#### systemd-resolved

```bash
# Single line
echo "DNS=192.168.50.1" | sudo tee -a /etc/systemd/resolved.conf.d/custom.conf

# Full block
sudo bash -c 'cat > /etc/systemd/resolved.conf.d/custom.conf << EOF
[Resolve]
DNS=192.168.50.1
FallbackDNS=8.8.8.8
EOF'
```

#### Netplan

```bash
sudo bash -c 'cat > /etc/netplan/99-custom.yaml << EOF
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      nameservers:
        addresses: [192.168.50.1, 8.8.8.8]
EOF'
```

#### Docker daemon config

```bash
echo '{"registry-mirrors": ["https://mirror.local"]}' | sudo tee /etc/docker/daemon.json
```

#### Crontab

```bash
echo "0 2 * * * root /usr/bin/backup.sh" | sudo tee -a /etc/crontab
sudo bash -c 'echo "0 2 * * * root /usr/bin/backup.sh" >> /etc/crontab'
```

#### Kernel parameters — /etc/sysctl.conf

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo bash -c 'echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf'
```

#### Environment variables — /etc/environment

```bash
echo "JAVA_HOME=/usr/lib/jvm/java-11" | sudo tee -a /etc/environment
sudo bash -c 'echo "JAVA_HOME=/usr/lib/jvm/java-11" >> /etc/environment'
```

#### APT repository

```bash
echo "deb https://repo.example.com stable main" | sudo tee /etc/apt/sources.list.d/custom.list
sudo bash -c 'echo "deb https://repo.example.com stable main" > /etc/apt/sources.list.d/custom.list'
```

#### systemd service overrides

```bash
echo "[Service]" | sudo tee -a /etc/systemd/system/myapp.service.d/override.conf
echo "Restart=always" | sudo tee -a /etc/systemd/system/myapp.service.d/override.conf
```

#### Global shell aliases — /etc/bash.bashrc

```bash
echo "alias ll='ls -la'" | sudo tee -a /etc/bash.bashrc
```

### Packer / Terraform Provisioners

In HCL provisioners, the same rules apply. Use `tee` or `bash -c` inside inline shell commands:

```hcl
provisioner "shell" {
  inline = [
    "echo 'root:s3cureP@ss' | sudo chpasswd",
    "sudo passwd -u root",
    "sudo sed -i.bkp 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config",
    "echo \"ChallengeResponseAuthentication yes\" | sudo tee -a /etc/ssh/sshd_config",
    "echo \"PermitRootLogin yes\" | sudo tee -a /etc/ssh/sshd_config",
    "sudo systemctl restart ssh"
  ]
}
```

Alternative using `bash -c`:

```hcl
provisioner "shell" {
  inline = [
    "echo 'root:s3cureP@ss' | sudo chpasswd",
    "sudo passwd -u root",
    "sudo sed -i.bkp 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config",
    "sudo bash -c 'echo \"ChallengeResponseAuthentication yes\" >> /etc/ssh/sshd_config'",
    "sudo bash -c 'echo \"PermitRootLogin yes\" >> /etc/ssh/sshd_config'",
    "sudo systemctl restart ssh"
  ]
}
```

### Pipe inside a root shell

Wrapping everything in a subshell keeps both the command and the redirection under root:

```bash
sudo sh -c 'apt-get update && apt-get upgrade | tee /var/log/upgrade.log'
```

Here `tee` can write to the root-owned log file because the entire pipeline runs inside the elevated shell.

## Logical Operators

You're not limited to `&&` and `;`. The full set of shell logical operators works inside a subshell:

| Operator | Behaviour |
|----------|-----------|
| `;` | Run next command regardless of exit code |
| `&&` | Run next command only if previous succeeded (exit 0) |
| `\|\|` | Run next command only if previous failed (non-zero exit) |

```bash
# Stop on failure
sudo sh -c 'mkdir /backup && cp -r /home/user/* /backup'

# Fallback: try install, update repos if it fails
sudo sh -c 'apt-get install foo || apt-get update'

# Mix: update, then upgrade or log failure
sudo sh -c 'apt-get update && apt-get upgrade || echo "upgrade failed" >> /var/log/upgrade-errors.log'
```

You can also combine both operators in a single expression. The classic pattern is create-then-write-or-cleanup:

```bash
sudo sh -c 'touch test.txt && echo "hello" > test.txt || rm test.txt'
```

- If `touch` succeeds, `echo` runs to write the content
- If `touch` or `echo` fails, `rm` cleans up the partial file

## Running Commands in Parallel

Use `&` inside the subshell to background tasks and `wait` to block until they all finish:

```bash
sudo sh -c 'task1 & task2 & wait'
```

Useful for independent long-running operations, like running two backup jobs at the same time:

```bash
sudo sh -c '/opt/backup/db.sh & /opt/backup/files.sh & wait && echo "all done"'
```

> **Note:** Output from parallel tasks will interleave. Redirect each command to its own log file if the output matters.

## Preserving Environment Variables

By default, `sudo` strips most environment variables before running the command (controlled by `env_reset` in `/etc/sudoers`). If a command depends on variables set in your current shell, pass them explicitly:

```bash
# Pass a single variable
sudo MY_VAR="value" bash -c 'echo $MY_VAR'

# Preserve specific variables using -E flag (if allowed by sudoers)
sudo -E bash -c 'echo $MY_VAR'
```

> **Note:** `sudo -E` only works if the sudoers policy permits it (`!env_reset` or `env_keep`). Many distributions restrict this by default.

## Key Points

- `sudo bash -c "..."` is the quickest way to run multiple commands without opening a shell
- Use `&&` instead of `;` if you want the sequence to stop on failure
- Use `||` for fallback logic — the right side runs only when the left side fails
- Heredocs (`<<'EOF'`) are easier to read than long one-liners
- `sudo -i` gives a full root login shell; `sudo -s` keeps your current environment
- For anything more than a few commands, write a script and `sudo bash script.sh`
- Pipes and redirections run as your user, not root — use `tee` or wrap in a subshell
- Use `sudo -u username -- sh -c '...'` to run multiple commands as a specific non-root user
- `--` stops sudo from parsing its own flags; use it when the command string could be misread
