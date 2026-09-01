# AIX Networking Cheatsheet

Command reference for TCP/IP networking on IBM AIX — interface and IP configuration (`ifconfig` for temporary, `chdev` for persistent via the ODM), hostnames and `/etc/hosts`, routing (`route`, `chdev -l inet0`), network tunables (`no`), device and interface listing (`lsdev`/`lsattr`), statistics (`entstat`, `netstat`), name resolution, and packet tracing (`iptrace`/`ipreport`).

> On AIX, `ifconfig` and `route add` changes are **temporary** (lost at reboot); persistent configuration lives in the ODM and is set with `chdev` against the interface (`enX`) or `inet0`. Most of these commands require `root`. Changing an interface state or IP on the link you're connected through can disconnect your session — use the console when in doubt.

## Interface and IP Configuration

```sh
# List the ODM IP configuration for an interface
lsattr -El en0

# Temporarily assign an address to an interface (lost at reboot)
ifconfig en0 192.168.1.2 netmask 255.255.255.0

# Temporarily add an alias to an interface
ifconfig en0 192.168.1.3 netmask 255.255.255.0 alias

# Permanently add an IP address to an interface (persists in the ODM)
chdev -l en0 -a netaddr=1.1.1.1 -a netmask=255.255.255.0 -a state=up

# Permanently add an alias
chdev -l en0 -a alias4=1.1.1.2,255.255.255.0

# Remove a permanently added alias
chdev -l en0 -a delalias4=1.1.1.2,255.255.255.0
```

### Interface state

```sh
# Disable an interface permanently (-P = persist in ODM)
chdev -l en0 -a state=down -P

# Detach a device
chdev -l en2 -a state=detach

# Bring an interface back up
chdev -l en2 -a state=up
```

### Remove an IP address from a NIC

```sh
chdev -l en1 -a state=down
chdev -l en1 -a netaddr=''
chdev -l en1 -a netmask=''
```

### Remove an interface entirely

```sh
ifconfig en0 down detach
rmdev -dl en0
cfgmgr
```

### Rename a device

```sh
# Rename ent0 to ent99
rendev -l ent0 -n ent99
```

## Hostname and /etc/hosts

```sh
# Permanently set the hostname
chdev -l inet0 -a hostname=server.domeniu.ro

# Add an entry to /etc/hosts (host with aliases)
hostent -a 192.100.201.7 -h "alpha bravo charlie"
```

## Routing

```sh
# View the kernel route table
netstat -r

# View the route table from the ODM
lsattr -EHl inet0 -a route

# Temporarily add a default route (lost at reboot)
route add default 192.168.1.1

# Temporarily add a static route
route add -net 192.168.2.0 -netmask 255.255.255.0 192.168.0.1

# Persistent default gateway via the ODM
chdev -l inet0 -a route=net,-hopcount,0,,0,192.168.1.1

# Persistent route via the ODM
chdev -l inet0 -a route="net,-hopcount,0,-netmask,255.255.255.224,,,,,158.226.253.160"

# Remove a persistent route from the ODM
chdev -l inet0 -a delroute="net,-hopcount,0,-netmask,255.255.255.224,,,,,158.226.253.160"

# Show which interface/gateway is used to reach an IP
route get 10.10.7/24
```

## Network Tunables (no)

The `no` command manages network options (kernel tunables).

```sh
# Turn on IP forwarding (routing)
no -o ipforwarding=1

# List all networking tunables
no -a

# Set a tunable temporarily (until reboot)
no -o use_isno=1

# Set a tunable to apply at the next reboot
no -r -o use_isno=1

# List all settings
no -L

# Reset all networking tunables to defaults
no -D

# Show a description of a tunable
no -h use_isno

# Set a tunable in both current and reboot values (persistent)
no -p -o use_isno=1
```

> A plain `no -o <tunable>=<value>` changes only the running value and is **lost at the next reboot**. Use `-r` to change only the reboot value, or `-p` to change both the current and reboot (permanent) values. Some tunables are reboot-only and require `bosboot` + reboot to take effect.

## Devices, Interfaces, and Attributes

```sh
# List TCP/IP networking devices
lsdev -Cc tcpip

# List network interfaces
lsdev -Cc if

# List inet0 attributes
lsattr -El inet0

# List routes (from ODM)
lsattr -El inet0 -a route

# Physical-layer (adapter) attributes of ent0
lsattr -El ent0

# Networking-layer (interface) attributes of en0
lsattr -El en0

# Show the media speed of an adapter
lsattr -El ent0 -a media_speed

# Set ent0 to 1G full duplex (persist with -P)
chdev -l ent0 -a media_speed=1000_Full_Duplex -P

# Show EtherChannel / pseudo adapters
lsdev -Cc adapter -s pseudo
lsattr -El ent8
```

> `entX` is the physical adapter (layer 2); `enX` is the corresponding IP interface (layer 3). Configure IP on `enX`, adapter attributes like `media_speed` on `entX`.

## Statistics and Connections

```sh
# Detailed statistics for an adapter
entstat -d ent0

# Similar statistics via netstat
netstat -v ent0

# All open/in-use TCP and UDP ports
netstat -anf inet

# All LISTENing TCP ports
netstat -na | grep LISTEN

# Traffic across an interface in 2-second intervals
netstat -I en0 2
```

## TCP/IP Configuration Management

```sh
# Remove all TCP/IP configuration from a host
rmtcpip

# Update the routing table / interface from the ODM
mkdev -l inet0
```

## Name Resolution

```sh
# Service lookup order for name services
cat /etc/netsvc.conf

# Resolver configuration (managed via namerslv / *namsv commands)
cat /etc/resolv.conf

# Flush the netcd DNS cache
netcdctrl -t dns -e hosts -f
```

## Connectivity Testing

```sh
# Reachability / round-trip test (Ctrl-C to stop, or -c to bound the count)
ping <host>
ping -c 4 <host>

# Trace the path to a host
traceroute <host>

# ARP cache (IP-to-MAC) for the local segment
arp -a

# Remove a stale ARP entry
arp -d <host>

# Resolve a name / reverse-resolve an address
host <name-or-ip>
```

## Troubleshooting

### Find the process behind a socket

Use `netstat` to find the socket address, then map it to a PID:

```sh
# Show the PID owning a specific socket (address from netstat -Aan)
rmsock f100050001717bb0 tcpcb
```

> Despite its name, `rmsock` does **not** remove the socket if it is in use — it reports the process holding it.

### Packet capture with iptrace / ipreport

```sh
# Capture UDP port 69 traffic
startsrc -s iptrace -a "-a -p 69 /tmp/udp.port"

# Capture traffic to a destination address
startsrc -s iptrace -a "-a -d <destination_address> /tmp/filename.ipt"

# Stop the capture
stopsrc -s iptrace

# Format the raw trace into a readable report
ipreport -rnsC /tmp/udp.port > /tmp/udp.port.out
```

## Quick Reference

| Task | Command |
|------|---------|
| View route table | `netstat -r` |
| Temp IP on interface | `ifconfig en0 <ip> netmask <mask>` |
| Persistent IP | `chdev -l en0 -a netaddr=<ip> -a netmask=<mask> -a state=up` |
| Persistent alias | `chdev -l en0 -a alias4=<ip>,<mask>` |
| Set hostname | `chdev -l inet0 -a hostname=<fqdn>` |
| Add /etc/hosts entry | `hostent -a <ip> -h "name aliases"` |
| Temp default route | `route add default <gw>` |
| Persistent route | `chdev -l inet0 -a route="net,...,<net>"` |
| Enable IP forwarding | `no -o ipforwarding=1` |
| List tunables | `no -a` |
| List interfaces | `lsdev -Cc if` |
| Adapter attributes | `lsattr -El ent0` |
| Set media speed | `chdev -l ent0 -a media_speed=1000_Full_Duplex -P` |
| Adapter statistics | `entstat -d ent0` |
| Listening ports | `netstat -na \| grep LISTEN` |
| Interface traffic | `netstat -I en0 2` |
| Socket → PID | `rmsock <addr> tcpcb` |
| Start packet trace | `startsrc -s iptrace -a "..."` |
| Format trace | `ipreport -rnsC <file>` |

## Related

- [AIX System Resource Controller (SRC) Cheatsheet](articles/aix-src-services-cheatsheet.md) — starting/stopping the TCP/IP daemons and `inetd` subservers referenced here.
