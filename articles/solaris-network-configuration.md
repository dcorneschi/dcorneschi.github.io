# Solaris Network Configuration Files

Network configuration on Oracle Solaris (10 and the "classic"/legacy path on 11) is driven by a set of files under `/etc`. This guide maps each file to what it controls, shows the commands to apply and inspect changes, and notes how a static IPv4 host is wired together.

> This covers the traditional file-based configuration used on Solaris 10 and Solaris 11 in *legacy/DefaultFixed* mode. Solaris 11 default installs use the **Network Auto-Magic (NWAM)** / `ipadm` + `dladm` framework; the files below still apply once you switch to the fixed network profile.

## Configuration Files

| Setting | File | Notes |
|---------|------|-------|
| IP address | `/etc/hostname.<interface>` | e.g. `/etc/hostname.e1000g0`; contains the hostname or IP to plumb on that NIC at boot |
| Domain name | `/etc/defaultdomain` | NIS/DNS domain name |
| Netmask | `/etc/inet/netmasks` | Network-number → netmask mappings |
| Hosts database | `/etc/hosts` (symlink to `/etc/inet/hosts`) | Static hostname/IP entries |
| DNS resolver | `/etc/resolv.conf` | `nameserver`, `domain`, `search` |
| Default gateway | `/etc/defaultrouter` | Static default route(s) |
| Node name | `/etc/nodename` | The system's hostname |
| Name service order | `/etc/nsswitch.conf` | Resolution order (files, dns, nis, ldap) |

### `/etc/hostname.<interface>`

The interface name is encoded in the filename — `/etc/hostname.e1000g0` configures `e1000g0`. It typically contains the hostname (resolved via `/etc/hosts`) or a literal IP address that Solaris plumbs onto that interface at boot. The presence of this file is what brings the interface up automatically.

```bash
# Example: /etc/hostname.e1000g0
server01
# ...or a literal address
192.168.1.10
```

### `/etc/inet/netmasks`

Maps a network number to its netmask so the address in `hostname.<interface>` gets the right prefix:

```
# network        netmask
192.168.1.0      255.255.255.0
```

### `/etc/hosts` and `/etc/inet/hosts`

`/etc/hosts` is a symlink to `/etc/inet/hosts` — the static host database:

```
127.0.0.1        localhost
192.168.1.10     server01.example.com  server01
```

### `/etc/resolv.conf`

The DNS client resolver configuration:

```
domain example.com
search example.com
nameserver 192.168.1.1
nameserver 8.8.8.8
```

### `/etc/defaultrouter`

One default gateway IP per line; read at boot to install the default route:

```
192.168.1.1
```

### `/etc/nodename`

A single line with the system's node (host) name:

```
server01
```

### `/etc/nsswitch.conf`

Controls the order sources are consulted for each database. For DNS name resolution, the `hosts:` line must include `dns`:

```
hosts:      files dns
```

## Commands

```bash
# Restart networking to apply file changes
svcadm restart network/physical
svcadm restart svc:/network/physical:default

# Check whether the machine is configured via DHCP
netstat -D

# Reset the system to an unconfigured state (re-runs setup at next boot)
/usr/sbin/sys-unconfig
```

- `svcadm restart network/physical` — reloads interface configuration under SMF (the Solaris service manager) after editing the files above.
- `netstat -D` — shows DHCP status for interfaces (whether an address is DHCP-assigned and its lease).
- `sys-unconfig` — **destructive to config:** unconfigures hostname, network, naming service, timezone, and root password, then halts. On next boot the system runs the initial setup interview again. Useful for cloning/templating a system, but it wipes the current network identity.

### Inspecting Interfaces with `ifconfig` (Solaris 10)

```bash
# Show all interfaces (including down)
ifconfig -a
```

Sample output:

```
lo0: flags=2001000849<UP,LOOPBACK,RUNNING,MULTICAST,IPv4,VIRTUAL> mtu 8232 index 1
        inet 127.0.0.1 netmask ff000000
e1000g0: flags=1000843<UP,BROADCAST,RUNNING,MULTICAST,IPv4> mtu 1500 index 2
        inet 192.168.1.10 netmask ffffff00 broadcast 192.168.1.255
        ether 8:0:27:ab:cd:ef
```

Temporary (non-persistent) changes with `ifconfig` — lost on reboot; use the files above for persistence:

```bash
# Plumb, address, and bring an interface up manually
ifconfig e1000g0 plumb
ifconfig e1000g0 192.168.1.10 netmask 255.255.255.0 up
ifconfig e1000g0 down       # take it down
ifconfig e1000g0 unplumb    # remove it
```

> On Solaris 11 use `ipadm`/`dladm` instead of `ifconfig` — see [Solaris 11: Configure a Static IP with ipadm and dladm](articles/solaris11-static-ip-ipadm.md).

## Bringing a Static IPv4 Host Together

To statically configure `e1000g0` the traditional way, the files work in concert:

1. `/etc/nodename` and `/etc/hostname.e1000g0` → the hostname / interface address
2. `/etc/inet/hosts` → resolve that hostname to the IP
3. `/etc/inet/netmasks` → the netmask for the network
4. `/etc/defaultrouter` → the default gateway
5. `/etc/resolv.conf` + `/etc/nsswitch.conf` (`hosts: files dns`) → DNS resolution
6. `svcadm restart network/physical` → apply

## Solaris 11: ipadm / dladm

On modern Solaris 11 with the fixed network profile, the same outcomes are configured with commands (which persist to the underlying config), rather than hand-editing every file:

```bash
# Data-link and IP configuration
dladm show-link
ipadm create-ip e1000g0
ipadm create-addr -T static -a 192.168.1.10/24 e1000g0/v4

# Default route and DNS
route -p add default 192.168.1.1
svccfg -s dns/client setprop config/nameserver = net_address: 192.168.1.1
svcadm refresh dns/client
```

`/etc/hosts`, `/etc/nsswitch.conf`, and `/etc/resolv.conf` remain relevant on Solaris 11; the per-interface `hostname.<if>`, `defaultrouter`, and `netmasks` files are the Solaris 10 / legacy mechanism.

## Diagnostics

```bash
# Routing table (confirm the default route)
netstat -rn

# Per-interface statistics / errors
netstat -i

# Test reachability and name resolution
ping -s 192.168.1.1
getent hosts server01

# Capture packets on an interface (Solaris tcpdump equivalent)
snoop -d e1000g0 host 192.168.1.1
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Interface not up after reboot | Missing/empty `/etc/hostname.<if>` | Create it with the hostname or IP; `svcadm restart network/physical` |
| No default route | `/etc/defaultrouter` missing/wrong | Add the gateway IP; `route add default <gw>` to test now |
| Name resolution fails | `nsswitch.conf` lacks `dns`, or bad `resolv.conf` | `hosts: files dns`; verify `resolv.conf` nameservers |
| Wrong netmask | `/etc/inet/netmasks` missing entry | Add `network netmask` line; re-plumb the interface |
| DHCP when you wanted static | Leftover DHCP config | `netstat -D` to confirm; remove DHCP, set static files |
| Changes don't persist | Set with `ifconfig` only | Put config in the `/etc` files (or use `ipadm` on S11) |

## Command and File Reference

| Task | File / Command |
|------|----------------|
| Interface IP | `/etc/hostname.<interface>` |
| Hostname | `/etc/nodename` |
| Netmask | `/etc/inet/netmasks` |
| Static hosts | `/etc/inet/hosts` (`/etc/hosts`) |
| DNS servers | `/etc/resolv.conf` |
| Default gateway | `/etc/defaultrouter` |
| Domain | `/etc/defaultdomain` |
| Resolution order | `/etc/nsswitch.conf` |
| Apply changes | `svcadm restart network/physical` |
| DHCP status | `netstat -D` |
| Unconfigure system | `/usr/sbin/sys-unconfig` |

## References

- [Configuring and Administering Oracle Solaris Networks](https://docs.oracle.com/cd/E37838_01/html/E60988/index.html) — official Oracle docs
- [nsswitch.conf(4) man page](https://docs.oracle.com/cd/E23824_01/html/821-1473/nsswitch.conf-4.html) — official Oracle docs
