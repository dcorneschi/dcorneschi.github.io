# lssh Cheatsheet

lssh is a terminal-native remote access suite for SSH workflows, cloud inventories, and provider-backed connectors. It provides interactive host selection, parallel execution, mux workspaces, file transfer, and monitoring.

## Installation

```bash
# Homebrew (macOS/Linux)
brew install blacknon/lssh/lssh

# Go install
go install github.com/blacknon/lssh/cmd/lssh@latest
```

## Quick Start

```bash
# Launch interactive host picker (reads ~/.ssh/config)
lssh

# Generate lssh config from existing SSH config
lssh --generate-lssh-conf > ~/.lssh.toml

# Connect to a specific host directly
lssh -H my-server

# Print server list from config
lssh -l
```

## TUI Navigation (Host Picker)

| Key | Action |
|-----|--------|
| Up / Down | Move cursor one line |
| Left / Right | Move between pages |
| Tab | Toggle current host selection and move to next |
| Ctrl+A | Select / unselect all visible hosts |
| Backspace | Delete one filter character |
| Space | Insert space into filter text |
| Enter | Confirm selection |
| Esc / Ctrl+C | Quit |
| Mouse click | Move cursor to clicked line |

Type to filter hosts by keyword. Use Tab to multi-select hosts for parallel or mux modes.

## Parallel Execution

```bash
# Run command in parallel across selected hosts
lssh -p <command>

# Example: tail logs on multiple servers
lssh -p tail -f /var/log/syslog

# Show server header in output
lssh -p -w <command>

# Hide server header in output
lssh -p -W <command>
```

## Mux Mode (Multi-Pane UI)

```bash
# Open mux pane workspace
lssh -P

# Run command in mux UI
lssh -P 'htop'

# Keep panes open after command exits
lssh -P --hold <command>

# Persistent mux sessions
lssh -P --mux-session ops              # create and attach
lssh -P --mux-session ops --mux-detach # create in background
lssh -P --mux-attach --mux-session ops # attach later
lssh -P --mux-list-sessions            # list sessions
lssh -P --mux-kill-session --mux-session ops  # kill session
```

### Mux Key Bindings

| Key | Action |
|-----|--------|
| Ctrl+A b | Toggle broadcast (type to all panes) |
| Ctrl+A c | New page |
| Ctrl+A s | New pane (opens host selector) |
| Ctrl+A " | Split pane horizontally |
| Ctrl+A % | Split pane vertically |
| Ctrl+A o | Next pane |
| Ctrl+A n | Next page |
| Ctrl+A p | Previous page |
| Ctrl+A w | Page list |
| Ctrl+A x | Close current pane |
| Ctrl+A f | File transfer |
| Ctrl+A d | Detach client |
| Ctrl+A & | Quit mux |

## Sending Commands to Multiple Hosts

| Method | Use Case |
|--------|----------|
| `lssh -p <cmd>` | Run one command across hosts, get combined output |
| `lssh -P '<cmd>'` | Each host gets a pane running the command |
| Ctrl+A b (in mux) | Broadcast live typing to all panes |
| `lsshell` | Persistent parallel interactive shell |

## Port Forwarding

```bash
# Local port forward
lssh -L 8080:localhost:80

# Remote port forward
lssh -R 80:localhost:8080

# Unix socket forwarding
lssh -L /tmp/local.sock:/tmp/remote.sock

# Dynamic (SOCKS5) forward
lssh -D 10080

# HTTP dynamic forward
lssh -d 18080

# HTTP reverse dynamic forward
lssh -r 18080

# Run in background after forwarding (like ssh -f)
lssh -f -L 8080:localhost:80

# Don't execute remote command (forwarding only)
lssh -N -L 8080:localhost:80
```

## NFS / SMB Forwarding

```bash
# NFS dynamic forward (mount remote path)
lssh -M 2049:/path/to/remote

# NFS reverse dynamic forward (expose local path)
lssh -m 2049:/path/to/local

# SMB dynamic forward
lssh -S 1445:/path/to/remote

# SMB reverse dynamic forward
lssh -s 1445:/path/to/local
```

## X11 Forwarding

```bash
# Standard X11 forwarding
lssh -X

# Trusted X11 forwarding
lssh -Y
```

## Tunnel Device

```bash
# Tunnel device (specify device numbers)
lssh --tunnel 0:0

# Tunnel device (auto-assign)
lssh --tunnel any:any
```

## Local Bashrc

```bash
# Use local bashrc on remote host (no files left behind)
lssh --localrc

# Disable local bashrc
lssh --not-localrc
```

## Connector Attach / Detach (AWS SSM)

```bash
# Attach to an existing connector session
lssh --attach session-1234567890abcdef

# Start a detached connector shell session
lssh --detach
```

## ControlMaster

```bash
# Temporarily enable ControlMaster
lssh --enable-control-master

# Temporarily disable ControlMaster
lssh --disable-control-master
```

## Piping

```bash
# Pipe stdin to remote command
echo "hello" | lssh cat

# Pipe local content to multiple hosts
cat script.sh | lssh -p bash
```

## File Transfer (Suite Tools)

### lscp (SCP-style)

```bash
# Copy local file to remote host
lscp /path/to/local/file.txt host:/tmp/

# Copy from remote to local
lscp host:/var/log/syslog /tmp/

# Remote-to-remote transfer
lscp host1:/path/file host2:/path/
```

### lsftp (Interactive SFTP)

```bash
# Open interactive SFTP session
lsftp
```

### lssync (Tree Sync)

```bash
# Sync directory to remote host
lssync /local/path host:/remote/path
```

## Configuration

Config file default: `~/.lssh.conf` (TOML or YAML). Override with `-F /path/to/config`.

### Load Hosts from SSH Config

```toml
[sshconfig.default]
path = "~/.ssh/config"
```

### Server Entry (TOML)

```toml
[server.myhost]
addr = "192.168.1.10"
port = "22"
user = "admin"
key  = "~/.ssh/id_rsa"
note = "Production web server"
```

### Server Entry (YAML)

```yaml
server:
  myhost:
    addr: "192.168.1.10"
    port: "22"
    user: "admin"
    key: "~/.ssh/id_rsa"
    note: "Production web server"
```

### SSH Agent

```toml
[server.myhost]
addr = "192.168.1.10"
user = "admin"
ssh_agent = true
```

### Pre/Post Commands

```toml
[server.themed]
addr = "192.168.100.10"
user = "demo"
pre_cmd = 'printf "\033]50;SetProfile=Remote\a"'
post_cmd = 'printf "\033]50;SetProfile=Default\a"'
```

### Port Forwarding in Config

```toml
# Multiple forwards
[server.multi]
port_forwards = [
    "L:8080:localhost:80",
    "R:80:localhost:8080",
]

# Single local forward
[server.fwd]
port_forward = "local"
port_forward_local = "8080"
port_forward_remote = "localhost:80"
```

### Local RC Files

```toml
[server.localrc]
addr = "192.168.100.104"
key  = "/path/to/private_key"
local_rc = "yes"
local_rc_compress = true
local_rc_file = [
    "~/dotfiles/.bashrc",
    "~/dotfiles/bash_prompt",
    "~/dotfiles/sh_alias",
]
```

### Ignoring / Hiding Hosts

```toml
# Hide a server from TUI unconditionally
[server.internal-jump]
addr = "10.0.0.1"
user = "admin"
ignore = true

# Hide conditionally (e.g., when outside a network)
[server.app.match.outside_office]
priority = 90
when.local_ip_not_in = ["192.168.100.0/24"]
ignore = true

# Ignore specific hosts imported from ~/.ssh/config
[sshconfig.default]
path = "~/.ssh/config"

[sshconfig.default.match.hide_jump]
name_in = ["jump-*", "bastion-*"]
ignore = true
```

### Prevent Duplicate Hosts (SSH Config Auto-Import)

```toml
# Option 1: Import but ignore all hosts from SSH config
[sshconfig.default]
path = "~/.ssh/config"

[sshconfig.default.match.ignore_all]
name_in = ["*"]
ignore = true

# Option 2: Point to empty path
[sshconfig.default]
path = "/dev/null"
```

### Match Condition Keys

| Key | Description |
|-----|-------------|
| `name_in` / `name_not_in` | Match/exclude by host name |
| `user_in` / `user_not_in` | Filter by username |
| `addr_in` / `addr_not_in` | Filter by address |
| `port_in` / `port_not_in` | Filter by port |

### Conditional `when.*` Keys

| Key | Description |
|-----|-------------|
| `local_ip_in` / `local_ip_not_in` | Match local IP / CIDR |
| `gateway_in` / `gateway_not_in` | Match default gateway |
| `username_in` / `username_not_in` | Match local username |
| `hostname_in` / `hostname_not_in` | Match local hostname |
| `os_in` / `os_not_in` | Match OS (darwin, linux, windows) |
| `term_in` / `term_not_in` | Match terminal (iterm2, tmux, etc.) |
| `env_in` / `env_not_in` | Check if env var exists |
| `env_value_in` / `env_value_not_in` | Match exact KEY=value pairs |

## Suite Tools

| Tool | Status | Purpose |
|------|--------|---------|
| `lssh` | Stable | Interactive SSH, parallel commands, forwarding |
| `lscp` | Stable | SCP-style copy over SSH/SFTP |
| `lsftp` | Stable | Interactive SFTP shell |
| `lsshell` | Beta | Parallel interactive shell with broadcast |
| `lsmux` | Beta | Pane-based SSH workspace |
| `lsmon` | Beta | Multi-host monitoring (no remote agents) |
| `lssync` | Beta | Tree sync over SSH/SFTP |
| `lsdiff` | Beta | Compare remote files in synchronized TUI |
| `lspipe` | Alpha | Persistent host sessions for pipelines |
| `lsshfs` | Beta | Mount remote directory via FUSE/NFS |

## Providers (Cloud Inventory)

| Provider | Description |
|----------|-------------|
| `provider-mixed-aws-ec2` | EC2 inventory + AWS SSM / EC2 Instance Connect |
| `provider-mixed-azure-compute` | Azure VM inventory |
| `provider-mixed-gcp-compute` | Google Compute Engine inventory |

## CLI Options Reference

```bash
--host, -H <name>         Connect to named host directly
--file, -F <path>         Config file path (default: ~/.lssh.conf)
--generate-lssh-conf      Generate lssh config from SSH config
-L <local:remote>         Local port forward
-R <remote:local>         Remote port forward
-D <port>                 SOCKS5 dynamic forward
-d <port>                 HTTP dynamic forward
-r <port>                 HTTP reverse dynamic forward
-M <port:/remote/path>    NFS dynamic forward
-m <port:/local/path>     NFS reverse dynamic forward
-S <port:/remote/path>    SMB dynamic forward
-s <port:/local/path>     SMB reverse dynamic forward
--tunnel <local:remote>   Tunnel device
-w                        Show server header in command mode
-W                        Hide server header in command mode
-N, --not-execute         Don't execute remote command
-X, --X11                 X11 forwarding
-Y                        Trusted X11 forwarding
-t, --term                Run command at terminal
-p, --parallel            Parallel execution
-P                        Mux UI mode
--hold                    Keep panes after command exits (with -P)
--mux-session <name>      Named persistent mux session
--mux-attach              Attach to existing mux session
--mux-detach              Create/keep session without attaching
--mux-list-sessions       List persistent mux sessions
--mux-kill-session        Kill named mux session
--attach <id>             Attach to connector session
--detach                  Start detached connector session
--localrc                 Use local bashrc
--not-localrc             Don't use local bashrc
--enable-transfer         Enable file transfer in mux
--disable-transfer        Disable file transfer in mux
--list, -l                Print server list
-f                        Run in background after connection
--enable-control-master   Enable ControlMaster temporarily
--disable-control-master  Disable ControlMaster temporarily
--help, -h                Show help
--version, -v             Show version
```

## Tips

- lssh reads `~/.ssh/config` by default — works out of the box
- Full-screen apps (vim, htop) work correctly over lssh sessions
- Use `--localrc` to bring your shell environment without leaving files behind
- Combine `-P --hold` to inspect output after parallel commands finish
- Use lssh on a bastion to centralize team SSH access
- `local_rc_compress = true` helps with large bashrc files
- Use Tab in the TUI to multi-select, then Enter to confirm

## Quick Reference

| Action | Command |
|--------|---------|
| Interactive picker | `lssh` |
| Direct connect | `lssh -H hostname` |
| Parallel command | `lssh -p 'command'` |
| Multi-pane mode | `lssh -P` |
| Broadcast mode | Ctrl+A b (inside mux) |
| Port forward | `lssh -L 8080:localhost:80` |
| SOCKS proxy | `lssh -D 10080` |
| Copy file | `lscp local host:/remote` |
| SFTP session | `lsftp` |
| Generate config | `lssh --generate-lssh-conf > ~/.lssh.toml` |
| List hosts | `lssh -l` |
| Mux session | `lssh -P --mux-session name` |
| Attach session | `lssh -P --mux-attach --mux-session name` |
