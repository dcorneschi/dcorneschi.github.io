# SSH ControlMaster

SSH ControlMaster allows multiple SSH sessions to share a single network connection. Once the first connection to a host is established, subsequent sessions reuse that connection — skipping the TCP handshake and authentication overhead entirely.

This benefits all SSH-based tools: `ssh`, `scp`, `rsync`, `git` (over SSH), and automation tools like Ansible.

## How It Works

Every SSH connection goes through a handshake: TCP three-way handshake, version exchange, key exchange (Diffie-Hellman or ECDH), host key verification, and user authentication. Even on a fast network, this takes 150–500ms. On high-latency links, it can exceed a second.

With ControlMaster enabled:

1. The first SSH connection to a given user@host:port becomes the **master**
2. It opens a **Unix domain socket** at the `ControlPath` location
3. Every subsequent SSH connection to the same target finds that socket, skips the handshake entirely, and multiplexes a new channel inside the existing session

The result: subsequent connections drop from ~400ms to ~20ms.

### Without ControlMaster

```
Session 1:  Client ──TCP──▶ Server ──KeyExchange──▶ Auth ──▶ Shell
Session 2:  Client ──TCP──▶ Server ──KeyExchange──▶ Auth ──▶ Shell
Session 3:  Client ──TCP──▶ Server ──KeyExchange──▶ Auth ──▶ Shell
            ───────────────────────────────────────────────────────
            Each session: ~400ms setup overhead
```

### With ControlMaster

```
Session 1 (master):
  Client ──TCP──▶ Server ──KeyExchange──▶ Auth ──▶ Shell
       │
       └──creates──▶ Unix Socket (~/.ssh/master-user@host:22)

Session 2 (slave):
  Client ──socket──▶ Master Process ──mux channel──▶ Shell
                     (no TCP, no auth)              ~20ms

Session 3 (slave):
  Client ──socket──▶ Master Process ──mux channel──▶ Shell
                     (no TCP, no auth)              ~20ms
```

### Connection Lifecycle

```
┌─────────────────────────────────────────────────────────┐
│                     CLIENT MACHINE                      │
│                                                         │
│  ┌──────────┐     ┌──────────────────┐                  │
│  │ ssh (1)  │────▶│  Master Process  │──── TCP ────▶ Server
│  └──────────┘     │  (holds socket)  │                  │
│                   └────────▲─────────┘                  │
│  ┌──────────┐              │                            │
│  │ ssh (2)  │──── Unix ────┘                            │
│  └──────────┘     Socket                                │
│  ┌──────────┐              │                            │
│  │ scp (3)  │──── Unix ────┘                            │
│  └──────────┘                                           │
│  ┌──────────┐              │                            │
│  │ rsync(4) │──── Unix ────┘                            │
│  └──────────┘                                           │
└─────────────────────────────────────────────────────────┘
```

### ControlPersist Behavior

```
Timeline:
─────────────────────────────────────────────────────────▶ time

├── ssh session 1 (master) ──────┤
                    ├── ssh session 2 ──┤
                                        ├── ssh session 3 ──┤
                                                            │
                                                    last session closes
                                                            │
                                                            ├─── ControlPersist=10m ───┤
                                                            │                          │
                                                            │    master stays alive    master exits
                                                            │    (accepts new sessions)
                                                            │
                                                    If new session connects
                                                    during this window:
                                                    timer resets ─────────────────────┤
```

## Configuration

Edit `~/.ssh/config`:

```
Host *
    ControlMaster auto
    ControlPath ~/.ssh/master-%r@%h:%p
    ControlPersist 10m
```

### Options

| Option | Description |
|--------|-------------|
| `ControlMaster auto` | Automatically use an existing master connection or create one if none exists |
| `ControlPath` | Path to the Unix socket used for connection sharing |
| `ControlPersist` | Keep the master connection alive in the background after the initial session closes |

### ControlPath Tokens

| Token | Meaning |
|-------|---------|
| `%r` | Remote login name |
| `%h` | Remote host name |
| `%p` | Port |
| `%L` | Local hostname (first component) |
| `%l` | Local hostname (full) |
| `%C` | Hash of `%l%h%p%r%j` (useful to shorten long paths) |

A common alternative ControlPath that avoids socket path length issues:

```
ControlPath ~/.ssh/master-%C
```

### ControlMaster Values

| Value | Behavior |
|-------|----------|
| `auto` | Use existing master if available, otherwise create one |
| `yes` | This connection acts as master (fails if one already exists) |
| `no` | Do not share connections (default) |
| `ask` | Ask for confirmation before reusing a master connection |
| `autoask` | Like `auto`, but ask before creating a new master |

### ControlPersist Values

| Value | Behavior |
|-------|----------|
| `yes` | Keep master alive indefinitely after last session disconnects |
| `no` | Close master when the last session exits (default) |
| `10m` | Keep master alive for 10 minutes after last session exits |
| `3600` | Keep alive for 3600 seconds |

## Manually Managing Connections

Check the status of a master connection:

```bash
ssh -O check user@host
```

Request the master to exit gracefully:

```bash
ssh -O exit user@host
```

Stop accepting new multiplexed sessions but keep existing ones alive:

```bash
ssh -O stop user@host
```

Forward a new port over an existing master:

```bash
ssh -O forward -L 8080:localhost:80 user@host
```

Cancel a port forward:

```bash
ssh -O cancel -L 8080:localhost:80 user@host
```

List active sockets:

```bash
lsof -U | grep master
```

## Background Master Connection

Instead of relying on an interactive session as the master, start a dedicated background master that stays idle:

```bash
ssh -CX -o ServerAliveInterval=30 -fN user@host
```

| Flag | Purpose |
|------|---------|
| `-f` | Go to background after authentication |
| `-N` | Do not execute a remote command (just hold the connection) |
| `-C` | Enable compression |
| `-X` | Enable X11 forwarding |
| `-o ServerAliveInterval=30` | Send keepalive every 30 seconds to prevent idle disconnects |

This is useful when you need the master to outlive any single terminal session. All subsequent `ssh user@host` commands piggyback off it automatically.

## Per-Host Configuration

Apply ControlMaster only for specific hosts:

```
Host dev-*
    ControlMaster auto
    ControlPath ~/.ssh/master-%r@%h:%p
    ControlPersist 10m

Host production-*
    ControlMaster no
```

### Single Sign-On Pattern

Useful for clusters or environments requiring MFA — authenticate once, reuse for all subsequent sessions:

```
Host mycluster
    User myusername
    HostName login.cluster.example.com
    ControlMaster auto
    ControlPath ~/.ssh/%r@%h:%p
    ControlPersist 4h
```

After the first `ssh mycluster` (where you enter password + MFA), all subsequent sessions skip authentication entirely.

## Use with ProxyJump and Bastion Hosts

ControlMaster works with `ProxyJump`, but the master holds the proxied connection open through the bastion:

```
┌────────┐       ┌──────────┐       ┌──────────────┐
│ Client │──TCP──│ Bastion  │──TCP──│ Internal Host│
└────┬───┘       └──────────┘       └──────────────┘
     │
     ├── Master socket A (client ↔ bastion)
     │
     └── Master socket B (client ↔ internal, via bastion)
         Subsequent sessions to internal host
         reuse BOTH connections (no new TCP at all)
```

```
Host bastion
    HostName bastion.example.com
    User admin
    ControlMaster auto
    ControlPath ~/.ssh/master-%r@%h:%p
    ControlPersist 5m

Host internal-*
    ProxyJump bastion
    ControlMaster auto
    ControlPath ~/.ssh/master-%r@%h:%p
    ControlPersist 5m
```

> **Warning:** If your bastion enforces short session timeouts, the master may die mid-task. In that case, disable multiplexing for that host or increase the bastion's idle timeout.

## Use with Automation Tools

### Ansible

Ansible opens an SSH connection for every task on every host. With ControlMaster, subsequent tasks reuse the first connection:

```ini
# ansible.cfg
[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o ControlPath=~/.ssh/ansible-%r@%h:%p
```

For a playbook with 30 tasks against 20 hosts, this eliminates up to 600 redundant SSH handshakes.

### rsync and scp

Both `rsync` and `scp` run over SSH and automatically benefit from an existing master connection — no additional configuration needed:

```bash
# These reuse the master connection if one exists
rsync -avz ./files/ user@host:/remote/path/
scp file.txt user@host:/tmp/
```

### Git over SSH

Git push/pull/fetch operations over SSH also reuse the multiplexed connection:

```bash
# All of these benefit from ControlMaster
git clone git@github.com:user/repo.git
git push origin main
git fetch --all
```

## CI/CD Environments

In CI pipelines, `~/.ssh` may be read-only or non-existent. Use `/dev/shm` (tmpfs) for the socket:

```
Host *
    ControlMaster auto
    ControlPath /dev/shm/ssh-%r@%h:%p
    ControlPersist 60
```

Benefits of `/dev/shm`:
- Lives in memory, no disk I/O
- Automatically cleaned up when the container/runner terminates
- Writable in most container configurations

## Performance Comparison

Typical latency with and without multiplexing:

| Scenario | Without ControlMaster | With ControlMaster |
|----------|---------------------- |--------------------|
| Local network | ~150ms | ~20ms |
| Same region (cloud) | ~200–400ms | ~20ms |
| Transatlantic | ~800–1200ms | ~30ms |
| Through bastion | ~1–3s | ~40ms |

Real-world deploy example (multiple SSH commands to 3 hosts):

| Task | Without multiplexing | With multiplexing |
|------|----------------------|-------------------|
| Code sync | 28.4s | 4.1s |
| Run migrations | 3.8s | 2.2s |
| Swap symlink | 1.2s | 0.3s |
| Cleanup | 2.3s | 0.4s |

## Security Considerations

- **Socket permissions**: The Unix socket file is owned by your user and has `0600` permissions by default. Only processes running as your user can connect to it.
- **Shared access risk**: Anyone who can access the socket file can multiplex sessions without authenticating. Ensure `~/.ssh` has `0700` permissions.
- **Port forwarding is fixed**: All port forwarding (including X11) must be set up by the initial master connection. Subsequent sessions cannot add new forwards unless you use `ssh -O forward`.
- **Master compromise**: If the master process is compromised, all multiplexed sessions are affected. For high-security hosts, consider `ControlMaster no`.

## Tips and Tricks

### Verify multiplexing is active

```bash
ssh -O check user@host
# Master running (pid=12345)
```

### Quick speed test

```bash
# First connection (full handshake)
time ssh user@host true

# Second connection (multiplexed)
time ssh user@host true
```

### Automatically clean stale sockets on login

Add to `~/.bashrc` or `~/.zshrc`:

```bash
find ~/.ssh -name 'master-*' -type s -exec ssh -O check {} \; 2>/dev/null
```

### Disable for a single connection

Override without changing your config:

```bash
ssh -o ControlMaster=no -o ControlPath=none user@host
```

### Use with screen/tmux on the remote host

Start a background master, then attach to remote tmux:

```bash
ssh -fN user@host
ssh user@host -t 'tmux attach || tmux new'
```

### Socket directory with restricted permissions

Create a dedicated directory for sockets:

```bash
mkdir -p ~/.ssh/sockets
chmod 700 ~/.ssh/sockets
```

Then configure:

```
ControlPath ~/.ssh/sockets/%r@%h:%p
```

### Keepalive to prevent idle disconnects

Pair ControlMaster with keepalive settings:

```
Host *
    ControlMaster auto
    ControlPath ~/.ssh/master-%r@%h:%p
    ControlPersist 10m
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

This sends a keepalive packet every 60 seconds and disconnects after 3 missed responses.

## Caveats and Gotchas

### ControlPersist without keepalives

If you use `ControlPersist yes` (indefinite) without `ServerAliveInterval`, NAT gateways and stateful firewalls will silently drop the idle TCP connection after their timeout (often 5–15 minutes). The socket file remains, but the master is effectively dead. Always pair long-lived persistence with keepalives.

### File transfer performance

Multiplexing dramatically helps with many small `scp`/`rsync` operations (eliminating per-transfer handshakes). For a single large file transfer, it makes no meaningful difference — the bottleneck is bandwidth, not connection setup.

### macOS socket quirks

Apple's bundled OpenSSH on macOS Ventura and later occasionally has issues with `ControlPersist` and backgrounding. If you encounter unexpected socket errors or master processes that refuse to persist, try installing OpenSSH via Homebrew (`brew install openssh`) and ensure your `PATH` picks it up before `/usr/bin/ssh`.

### NFS or network-mounted home directories

Unix domain sockets do not work over NFS or other network filesystems. If `~/.ssh` is on a network mount, the socket will fail silently. Point `ControlPath` to a local filesystem:

```
ControlPath /tmp/ssh-master-%r@%h:%p
```

Or use the XDG runtime directory if available:

```
ControlPath /run/user/%i/ssh-master-%r@%h:%p
```

(`%i` expands to the local user's UID.)

### MaxSessions server-side limit

OpenSSH server has a `MaxSessions` setting (default: 10) that limits the number of multiplexed channels per connection. If you exceed this, new sessions will fail with "channel open failed". Increase it in the server's `sshd_config`:

```
# /etc/ssh/sshd_config
MaxSessions 20
```

Then reload sshd: `systemctl reload sshd`

### All sessions die when the master dies

If the master connection drops (network outage, laptop sleep, kill -9), all multiplexed sessions die immediately. This is the fundamental trade-off of ControlMaster. Mitigations:
- Use `tmux` or `screen` on the remote host to persist work
- Use a background master (`ssh -fN`) separate from your interactive sessions
- Set reasonable `ControlPersist` timeouts rather than `yes` (indefinite)

## Troubleshooting

### Socket path too long

Unix sockets have a maximum path length (108 bytes on Linux, 104 on macOS). If you get errors like `ControlPath too long`, use `%C` instead:

```
ControlPath ~/.ssh/master-%C
```

Or use a shorter base path:

```
ControlPath /tmp/ssh-%C
```

### Stale socket files

If a master connection was not cleaned up properly (kill -9, OOM, container restart):

```bash
rm ~/.ssh/master-user@host:22
```

Or find and remove all stale sockets:

```bash
find ~/.ssh -name 'master-*' -type s -delete
```

### "mux_client_request_session: read from master failed"

This means the master process is gone but the socket file remains. With `ControlMaster=auto`, OpenSSH detects this and creates a new master on the next attempt. If the error persists, remove the socket file manually.

### Connection hangs after network change

When a network change (WiFi switch, VPN toggle) makes the master connection stale, new multiplexed sessions will hang. Force disconnect:

```bash
ssh -O exit user@host
```

If that doesn't respond, remove the socket file manually.

### X11 forwarding not working on subsequent sessions

X11 forwarding must be established on the initial (master) connection. If you forgot `-X` on the first connection, exit the master and reconnect:

```bash
ssh -O exit user@host
ssh -X user@host
```

### Debugging

Enable verbose output to see multiplexing status:

```bash
ssh -v user@host
```

Look for lines like:

```
debug1: auto-mux: Trying existing master
debug1: mux_client_request_session: master session id: 2
```

If multiplexing is not being used, look for:

```
debug1: auto-mux: Trying existing master
debug1: mux_client_hello_exchange: master refused mux request
```

