# SSH ProxyJump vs ProxyCommand

Both `ProxyJump` and `ProxyCommand` allow SSH to reach hosts that are not directly accessible — typically servers behind a bastion or jump host. They solve the same problem but work differently.

```
You ──▶ Bastion ──▶ Server
```

`ProxyJump` (the `-J` flag) was introduced in OpenSSH 7.3. Use `ProxyJump` instead of `ProxyCommand` whenever possible — it's simpler, safer, and supports chaining natively.

## Quick Comparison

| | ProxyJump (`-J`) | ProxyCommand |
|--|------------------|--------------|
| Available since | OpenSSH 7.3 (2016) | OpenSSH 2.0+ (all versions) |
| Command-line flag | `-J user@bastion` | `-o ProxyCommand="..."` |
| Config directive | `ProxyJump bastion` | `ProxyCommand ssh -W %h:%p bastion` |
| Encryption | End-to-end (double encrypted) | Depends on the command used |
| Multiple hops | Built-in (`-J hop1,hop2,hop3`) | Manual nesting required |
| External tools needed | None | May require `nc`, `socat`, etc. |
| Config inheritance | Jump host reads its own `~/.ssh/config` | Full control over the command |
| Flexibility | SSH-only jump hosts | Any transport (HTTP proxy, custom scripts) |
| Tokens supported | `%%`, `%h`, `%n`, `%p`, `%r` | `%%`, `%h`, `%n`, `%p`, `%r` |

## How They Work

### ProxyJump

```
Client ═══SSH═══▶ Bastion ───TCP forward───▶ Target
         (encrypted end-to-end)
```

ProxyJump tells the SSH client to first establish an SSH connection to the jump host, then request a TCP forwarding channel to the final destination. The client then runs the full SSH handshake to the target **through** this channel.

The result is two nested SSH sessions — the traffic between the client and target is encrypted independently of the bastion. The bastion only sees an opaque TCP stream; it cannot inspect the inner session.

### ProxyCommand

```
Client ═══SSH═══▶ [stdout/stdin of command] ═══▶ Target
```

ProxyCommand runs an arbitrary command whose stdin/stdout become the transport for the SSH connection. The command is responsible for delivering bytes between the client and the target's SSH port.

The most common ProxyCommand usage (`ssh -W %h:%p bastion`) produces the same result as ProxyJump. But ProxyCommand can use any program — `nc`, `socat`, `connect-proxy`, `corkscrew`, or a custom script.

## ProxyJump

### Command Line

```bash
# Single jump host
ssh -J bastion user@target

# With explicit user and port on the jump host
ssh -J admin@bastion:2222 user@target

# Multiple jump hosts (chained left to right)
ssh -J bastion1,bastion2,bastion3 user@target
```

### Config File

```
Host bastion
    HostName bastion.example.com
    User jumpuser
    Port 22
    IdentityFile ~/.ssh/id_bastion

Host target
    HostName 10.0.1.100
    User deploy
    ProxyJump bastion
```

After configuring:

```bash
ssh target    # automatically jumps through bastion
scp file.txt target:/tmp/    # also works
rsync -avz ./dir/ target:/path/    # also works
```

### Multiple Hops in Config

```
Host bastion
    HostName bastion.example.com
    User jumpuser

Host middleware
    HostName 10.0.1.50
    User admin
    ProxyJump bastion

Host deep-target
    HostName 172.16.0.10
    User app
    ProxyJump middleware
```

This chains: `client → bastion → middleware → deep-target`

### Disable ProxyJump for Specific Host

```
Host direct-server
    HostName server.example.com
    ProxyJump none
```

## ProxyCommand

### Command Line

```bash
# Using ssh -W (most common, equivalent to ProxyJump)
ssh -o ProxyCommand="ssh -W %h:%p bastion" user@target

# With -q to suppress warnings from the proxy hop
ssh -o ProxyCommand="ssh -q -W %h:%p bastion" user@target

# Using netcat
ssh -o ProxyCommand="ssh bastion nc %h %p" user@target

# Using netcat with timeout (120 seconds)
ssh -o ProxyCommand="ssh bastion nc -w 120 %h %p" user@target

# Through an HTTP CONNECT proxy
ssh -o ProxyCommand="nc -X connect -x proxy.corp.com:8080 %h %p" user@target

# Using socat
ssh -o ProxyCommand="socat - PROXY:proxy.corp.com:%h:%p,proxyport=8080" user@target

# Using corkscrew (HTTP proxy tunneling)
ssh -o ProxyCommand="corkscrew proxy.corp.com 8080 %h %p" user@target

# Windows: must specify full path to ssh.exe
# ProxyCommand C:\Windows\System32\OpenSSH\ssh.exe -W %h:%p bastion
```

### Config File

```
Host target
    HostName 10.0.1.100
    User deploy
    ProxyCommand ssh -W %h:%p bastion
```

### Multiple Hops with ProxyCommand

Nesting is more verbose than ProxyJump:

```
Host bastion
    HostName bastion.example.com
    User jumpuser

Host middleware
    HostName 10.0.1.50
    User admin
    ProxyCommand ssh -W %h:%p bastion

Host deep-target
    HostName 172.16.0.10
    User app
    ProxyCommand ssh -W %h:%p middleware
```

### ProxyCommand Tokens

| Token | Meaning |
|-------|---------|
| `%h` | Target hostname |
| `%p` | Target port |
| `%r` | Target username |
| `%n` | Original hostname as typed on the command line |
| `%%` | Literal `%` |

### Disable ProxyCommand for Specific Host

```
Host direct-server
    ProxyCommand none
```

## When to Use Which

### Use ProxyJump when:

- You're jumping through one or more SSH bastion hosts
- You want simple, readable config
- You need multiple chained hops
- The jump host runs a standard sshd
- You're on OpenSSH 7.3+

### Use ProxyCommand when:

- You need to go through an HTTP/SOCKS proxy (corporate firewall)
- The intermediate host doesn't run sshd (or runs a non-standard version)
- You need to run custom connection logic (scripts, special tunneling tools)
- You're on an older OpenSSH version (< 7.3)
- You need to use `nc`, `socat`, `corkscrew`, or similar tools

## Precedence Rules

From the ssh_config man page:

> ProxyJump and ProxyCommand compete — whichever is specified first will prevent later instances of the other from taking effect.

In practice:
- Command-line options override config file settings
- In the config file, the **first matching** directive wins (SSH processes top to bottom)
- `ProxyJump none` and `ProxyCommand none` explicitly disable the respective option

## Interaction with Other Features

### ControlMaster (Multiplexing)

Both ProxyJump and ProxyCommand work with ControlMaster. The master connection holds the proxy path open, and subsequent sessions reuse it:

```
Host *
    ControlMaster auto
    ControlPath ~/.ssh/control-%h-%p-%r
    ControlPersist 300

Host target
    ProxyJump bastion
```

With multiplexing active, subsequent connections through the same proxy reuse the existing tunnel — significantly speeding up repeated operations like deploy scripts.

### Agent Forwarding vs ProxyJump

With ProxyJump, your SSH agent is used on the **client** to authenticate to both the jump host and the target. Keys never leave your machine. This is inherently safer than agent forwarding (`-A`), where the bastion could potentially hijack your agent.

```
# Safe: ProxyJump (keys never exposed to bastion)
ssh -J bastion target

# Less safe: Agent forwarding (bastion can use your agent)
ssh -A bastion
ssh target    # from bastion
```

### Port Forwarding Through a Jump Host

```bash
# Local forward through a bastion
ssh -J bastion -L 3306:db.internal:3306 user@target

# In config
Host target
    ProxyJump bastion
    LocalForward 3306 db.internal:3306
```

### SCP and Rsync

Both tools work transparently with ProxyJump config:

```bash
# SCP through jump host (uses config)
scp file.txt target:/tmp/

# Rsync through jump host
rsync -avz -e "ssh -J bastion" ./dir/ user@target:/path/
```

## Security Comparison

| Concern | ProxyJump | ProxyCommand (ssh -W) | ProxyCommand (nc) |
|---------|-----------|----------------------|-------------------|
| End-to-end encryption | Yes | Yes | Depends on target sshd |
| Bastion can read traffic | No | No | No (if target uses SSH) |
| Bastion auth required | Yes (SSH) | Yes (SSH) | No (just TCP relay) |
| Client keys exposed to bastion | No | No | N/A |
| Host key verification | Both hops verified | Both hops verified | Only target verified |

## Common Patterns

### Corporate Environment (HTTP Proxy + Bastion)

```
# First, get through the corporate HTTP proxy to reach the bastion
Host bastion
    HostName bastion.example.com
    User admin
    ProxyCommand nc -X connect -x corpproxy.internal:8080 %h %p

# Then jump through the bastion to internal hosts
Host internal-*
    ProxyJump bastion
```

### Cloud Environment (Direct ProxyJump)

```
Host bastion
    HostName 52.1.2.3
    User ec2-user
    IdentityFile ~/.ssh/aws-key.pem

Host 10.0.*
    User ec2-user
    IdentityFile ~/.ssh/aws-key.pem
    ProxyJump bastion
```

### Dynamic Jump Host (Script-based)

```
Host dynamic-target
    ProxyCommand ./scripts/get-jump-host.sh %h %p
```

### Fallback: Try Direct, Then Jump

SSH doesn't natively support fallback. A workaround:

```bash
ssh target 2>/dev/null || ssh -J bastion target
```

## Troubleshooting

### "ProxyJump: unsupported option"

Your SSH client is older than 7.3. Upgrade or use ProxyCommand:

```bash
ssh -V    # check version

# Fallback to ProxyCommand
ssh -o ProxyCommand="ssh -W %h:%p bastion" user@target
```

### "channel 0: open failed: connect failed"

The jump host cannot reach the target (wrong IP, firewall, target sshd not running):

```bash
# Test from the bastion directly
ssh bastion "nc -zv target-ip 22"
```

### Verbose debugging through jump hosts

```bash
# Debug the full chain
ssh -vvv -J bastion user@target

# Debug only the jump host connection
ssh -vvv bastion "nc -zv target-ip 22"

# Print resolved SSH config without connecting (verify your config)
ssh -G target
```

### Config not applying to jump host

SSH config for the **destination** is not applied to jump hosts. If your bastion needs a specific key or port, define it explicitly:

```
Host bastion
    HostName bastion.example.com
    User jumpuser
    Port 2222
    IdentityFile ~/.ssh/bastion_key

Host target
    HostName 10.0.1.100
    ProxyJump bastion
    # These settings apply to target, NOT to bastion
    User deploy
    IdentityFile ~/.ssh/deploy_key
```

### IdentitiesOnly with Many Keys

When the SSH agent has many keys loaded, the server may reject the connection after `MaxAuthTries` (default: 6) failed attempts before the right key is tried. Use `IdentitiesOnly yes` to ensure only the specified key is offered:

```
Host bastion
    HostName bastion.example.com
    User jumpuser
    IdentityFile ~/.ssh/bastion_key
    IdentitiesOnly yes

Host target
    HostName 10.0.1.100
    ProxyJump bastion
    User deploy
    IdentityFile ~/.ssh/deploy_key
    IdentitiesOnly yes
```

### HostKeyAlias

When the target hostname used by the client doesn't match what's stored in `known_hosts` (common with ProxyCommand and internal DNS), use `HostKeyAlias` to explicitly name which host key to expect:

```
Host internal-server
    HostName 10.0.1.100
    ProxyCommand ssh -W %h:%p bastion
    HostKeyAlias internal-server.lan
```

### DNS Resolution Through Jump Hosts

ProxyJump requires the **client** to resolve the target hostname. If the target uses internal DNS that only the jump host can resolve, ProxyCommand handles this because the jump host's SSH client does the resolution:

```bash
# ProxyJump will FAIL if client can't resolve "internal.lan"
ssh -J bastion user@internal.lan    # NXDOMAIN on client

# ProxyCommand works — the jump host resolves the name
ssh -o ProxyCommand="ssh -W internal.lan:22 bastion" user@internal.lan
```

In config:

```
Host internal
    HostName internal.lan
    ProxyCommand ssh -W %h:%p bastion
    HostKeyAlias internal.lan
```

## Conditional ProxyJump with Match exec

Use `Match exec` to auto-detect whether a jump host is needed — useful for laptops that move between office (direct access) and remote (needs bastion):

```
# Only jump when target is NOT directly reachable
Match host 10.0.1.* !exec "nc -z -w 1 %h %p"
    ProxyJump bastion.example.com

# IPv6-only host: jump when no IPv6 route exists
Match host ipv6only.example.org !exec "route -n get -inet6 %h"
    ProxyJump dualstack.example.org
```

This sends connections through the jump host only when the destination cannot be reached directly. Exclude the jump host itself to avoid loops:

```
Match host 192.168.1.* !host bastion.example.org !exec "nc -z -w 1 %h %p"
    ProxyJump bastion.example.org
```

### Conditional Based on Local Network

Jump only when you're NOT on the office network (check local IP on interface):

```
# macOS: jump when not on the office network
Match host server1 !exec "ifconfig en0 | grep 192.168.100.10"
    ProxyJump user@bastion.example.com

Host server1
    HostName 192.168.100.20
```

## SOCKS Proxy Through a Jump Host

Create a SOCKS proxy that exits through a server behind a bastion:

```bash
# Modern way (OpenSSH 7.3+)
ssh -D 1080 -J bastion user@server

# All browser traffic now routes: client → bastion → server → internet
```

## SSH Over Tor

### Client Through Tor

Route SSH connections through the Tor network to hide the client's origin:

```bash
# Using torsocks (simplest)
torsocks ssh user@host

# Using netcat through Tor's SOCKS5 proxy (port 9050)
ssh -o ProxyCommand="nc -X 5 -x localhost:9050 %h %p" user@host

# Important: disable DNS leaks
ssh -o VerifyHostKeyDNS=no -o ProxyCommand="nc -X 5 -x localhost:9050 %h %p" user@host
```

### Auto-Route .onion Addresses Through Tor

```
Host *.onion
    VerifyHostKeyDNS no
    ProxyCommand nc -x localhost:9050 -X 5 %h %p
```

## Security Best Practices

- Use **different SSH keys** for different security zones (bastion vs internal hosts)
- Restrict bastion access with firewall rules to only necessary source IPs and ports
- Disable password authentication on the bastion — require keys only
- Prefer `ProxyJump` over agent forwarding (`-A`) — keys never leave your machine
- Set `IdentitiesOnly yes` to prevent the agent from trying unrelated keys
- Monitor bastion logs for unusual connection patterns
- Use short-lived SSH certificates instead of static keys for large environments
- Set `MaxAuthTries 3` and `LoginGraceTime 30` on the bastion's sshd

## Netcat with sudo

When netcat isn't available to regular users on the jump host due to restricted permissions:

```
Host target
    ProxyCommand ssh bastion sudo nc -w 120 %h %p
```

This runs nc with elevated privileges on the jump host. Requires passwordless sudo for nc on the bastion.

## Legacy Methods (Pre-7.3)

For environments stuck on old OpenSSH versions where `-J` / `ProxyJump` is unavailable.

### ssh -W (OpenSSH 5.4–7.2)

stdio forwarding — equivalent to ProxyJump but more verbose:

```bash
ssh -o ProxyCommand="ssh -W %h:%p bastion" user@target
```

### ssh -tt Chaining (Pre-5.4)

Force TTY allocation and chain connections. Less secure — intermediate hosts can see the session:

```bash
# Single hop with -t (allocates TTY for interactive access)
ssh -t bastion ssh target

# Two hops with -tt (force TTY even without terminal)
ssh -tt bastion ssh -tt target

# Run a command through the chain
ssh -tt bastion ssh -tt target "uptime"

# Reattach to a remote screen session
ssh -tt bastion ssh -tt target screen -dR
```

> **Warning:** With `-tt` chaining, the intermediate host has full access to the session content. Use only when no better option exists.

### Bash /dev/tcp Pseudo-Device (No netcat Available)

When the jump host has Bash but lacks `nc`:

```bash
ssh -o HostKeyAlias=target.example.com \
    -o ProxyCommand="ssh user@bastion 'exec 3<>/dev/tcp/target.example.com/22; cat <&3 & cat >&3; kill \$!'" \
    user@target
```

> **Note:** Only works with Bash on Linux. Does not work with sh, ksh, zsh, or on BSDs/macOS even if Bash is installed.

### ProxyCommand with Routing Tables (rdomain)

When the jump host has interfaces on different routing tables:

```bash
ssh -o ProxyCommand="ssh user@bastion route -T 2 exec nc %h %p" user@target
```

In config:

```
Host target
    HostName 10.100.200.44
    User fred
    ProxyCommand ssh bastion route -T 2 exec nc %h %p
```
