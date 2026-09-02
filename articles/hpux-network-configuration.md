# HP-UX Network Configuration

Configuring networking on HP-UX — discovering interfaces (`lanscan`/`ioscan`), link-layer management (`lanadmin` on 11i v1/v2, `nwmgr` on 11i v3), assigning IP addresses, IP multiplexing (aliases), routing, network tunables (`ndd`), the hostname, and making it all persistent in `/etc/rc.config.d/netconf`.

## How HP-UX Networking Fits Together

HP-UX networking is split into two layers that you configure separately, and understanding the split is the key to avoiding most configuration headaches:

- **Link layer (Layer 2)** — the physical NIC and its media settings: MAC address, speed, duplex, and auto-negotiation. This is where `lanadmin` (11i v1/v2) and `nwmgr` (11i v3) operate. The interface here is named `lanN` (e.g. `lan0`), which is a *logical* name mapped onto a physical hardware path.
- **Network layer (Layer 3)** — IP addresses, netmasks, and routes. This is where `ifconfig` and `route` operate, and the interface is referenced by the same `lanN` name.

Two more distinctions matter constantly on HP-UX:

- **Runtime vs. persistent.** Commands like `ifconfig`, `route`, `lanadmin`, and `ndd -set` change the *running* kernel only — those changes are lost at the next reboot. To survive a reboot you must edit the matching `/etc/rc.config.d/` file. The startup scripts read those files at boot and replay the equivalent commands.
- **Interface index vs. PPA.** Utilities such as `lanscan` show both the interface name (`lan0`) and the underlying "PPA" (Physical Point of Attachment) hardware number. `lanadmin` takes the PPA (the numeric portion), while `ifconfig`/`nwmgr` take the `lanN` name. On most systems these numbers align, but on systems where NICs were added and removed they can differ, so always confirm with `lanscan` before assuming `lan3` uses PPA `3`.

## Configuration Files

| File | Purpose |
|------|---------|
| `/etc/rc.config.d/netconf` | System IPv4 configuration (interfaces, routes, hostname) |
| `/etc/rc.config.d/netconf-ipv6` | System IPv6 configuration |
| `/etc/rc.config.d/nddconf` | Persistent `ndd` network tunables |

These files are plain shell scripts sourced by the boot-time RC scripts. Every variable is an *indexed array* (`IP_ADDRESS[0]`, `IP_ADDRESS[1]`, …) so a single file can describe many interfaces, aliases, and routes. The index ties related entries together: `INTERFACE_NAME[2]`, `IP_ADDRESS[2]`, and `SUBNET_MASK[2]` all describe the same interface. Keep the indices contiguous starting at `0` — a gap in the sequence can cause the startup script to stop processing entries early.

Apply changes made to these files without a full reboot:

```bash
/sbin/init.d/net start          # (re)apply netconf interfaces/routes
/sbin/init.d/hostname start     # apply the hostname
```

`net start` is idempotent — it re-reads `netconf` and brings the described interfaces and routes into the state the file requests. It does **not** tear down interfaces you removed from the file, so if you delete an entry and rerun `net start`, the old interface stays up until you `ifconfig ... down`/`unplumb` it or reboot. When in doubt about whether the running state matches the file, reboot to get a clean, reproducible result — that is also the best test that your persistent config is actually correct.

## Discovering Interfaces

```bash
ioscan -fC lan                  # LAN hardware
lanscan                         # LAN interfaces (name, MAC, state) — 11i v1/v2 style
netstat -i                      # configured interfaces with IP
```

Use `lanscan` (or `nwmgr` on 11i v3) to find the interface names (`lan0`, `lan1`, …) you'll reference elsewhere.

Each of these tools answers a different question, and knowing which to reach for saves time:

- `ioscan -fC lan` reports what the *kernel sees at the hardware level* — the hardware path, the driver claiming the card (`igelan`, `iether`, `btlan`, etc.), and whether the device is `CLAIMED` (driver bound) or `UNCLAIMED`/`NO_HW` (a problem). Use it first when a NIC seems entirely missing.
- `lanscan` maps hardware to the `lanN` logical interface, its **PPA**, station address (MAC), and hardware state (`UP`/`DOWN`). This is the bridge between "hardware" and "network config."
- `netstat -in` shows the Layer-3 view: which interfaces have IP addresses, plus packet and error counters.

A typical `lanscan` line looks like this:

```
Hardware  Station        Crd Hdw   Net-Interface  NM  MAC       HP-DLPI DLPI
Path      Address        In# State NamePPA         ID  Type      Support Mjr#
0/1/2/0   0x00306EF30D45 0   UP    lan0 snap0      1   ETHER     Yes     119
```

The `Crd In#` column is the PPA you pass to `lanadmin`; the `NamePPA` column gives the `lan0` name you pass to `ifconfig`. If `Hdw State` shows `DOWN`, the link is not up at the physical layer — check the cable, switch port, and speed/duplex before touching any IP settings.

Two more discovery helpers:

```bash
lanscan -v                      # verbose: driver, MTU, and encapsulation detail
lanscan -p                      # print just the PPA numbers, one per line
```

## Link-Layer Management

Link-layer settings are where most "the server can't reach the network" problems actually live. A mismatched speed/duplex between the NIC and the switch port is the classic culprit: the link comes *up* and pings may even work, but throughput collapses and the interface counters fill with `Ierrs`/`Oerrs`, late collisions, or FCS errors. Modern practice is to let both ends auto-negotiate; force fixed speed/duplex only when the switch side is also hard-set, and make sure both ends match exactly.

### lanadmin (11i v1 and v2)

Changes via `lanadmin` are **lost at reboot** — make them permanent by editing the card's `/etc/rc.config.d/` config. `lanadmin` addresses the card by its **PPA** number (the `Crd In#` from `lanscan`), not the `lanN` name.

```bash
lanadmin -a 0        # MAC address of lan0
lanadmin -x 0        # speed/duplex settings (lanadmin -s 0 on older cards)
lanadmin -r 0        # reset lan0 (force auto-negotiation)
lanadmin -g 0        # statistics (collisions, errors)
lanadmin -c 0        # clear the statistics registers to zero
```

The `-x` option both queries and sets media parameters. To force a fixed speed and duplex (only when the switch is hard-set to match):

```bash
lanadmin -X 100FD 0        # force 100 Mbit full-duplex on PPA 0
lanadmin -X AUTO_ON 0      # re-enable auto-negotiation on PPA 0
lanadmin -x 0              # confirm the resulting setting
```

Run `lanadmin` with no PPA to enter its interactive menu (LAN → then `display`, `reset`, `clear`, etc.). The interactive statistics screen is the fastest way to watch inbound/outbound errors climb in real time while you diagnose a flaky link. Because `-X` changes are runtime-only, record the chosen speed/duplex in the card's `/etc/rc.config.d/hpbtlanconf` (or the driver-specific conf file) so the setting is reapplied at boot.

### nwmgr (11i v3)

`lanscan` and `lanadmin` are deprecated (but still present) on 11i v3; use **`nwmgr`** for link-layer configuration. `nwmgr` unifies what used to be several separate tools (`lanadmin`, `lanscan`, `linkloop`) behind one command with a consistent verb/attribute grammar: a verb (`--get`, `--set`, `--reset`, `--diagnose`), what to act on (`--attribute`, `--stats`, `--st`), and which card (`-c lanN` or `-I <index>`). The big advantage over `lanadmin` is `--sa` ("save"), which writes the change straight into the persistent config so you don't have to hand-edit an RC file separately.

```bash
nwmgr                                            # list all LAN interfaces

# Query attributes / stats
nwmgr --get --attribute all -c lan0
nwmgr --get --attribute speed,mac -c lan0
nwmgr --get --stats all -c lan0

# Modify current parameters
nwmgr --set --attribute speed=100fd -c lan0
nwmgr --set --attribute mac=0x080009cccccc -c lan0
nwmgr --set --attribute all --from default -c lan0

# Make current settings permanent (--sa saves to config)
nwmgr --set --attribute all --sa --from current -c lan0

# Reset stats / reset the card
nwmgr --reset --st -c lan0
nwmgr --reset -c lan0

# Layer-2 connectivity test to a MAC
nwmgr --diagnose --attribute dest=0x00306ef30d45 -c lan0
```

## Assigning IP Addresses (ifconfig)

`ifconfig` changes are **temporary** (lost at reboot) — for persistence use `netconf` (below). Before an interface can carry IP traffic it must be *plumbed* (attached to the IP stack) and then assigned an address; a single `ifconfig lanN <ip> netmask <mask> up` does all three (plumb, address, bring up) in one step. The reverse operations are `down` (stop passing traffic but keep the address) and `unplumb` (detach from IP entirely). Use `down`/`up` for a quick bounce, and `unplumb` when you truly want the interface gone from the Layer-3 view.

```bash
ifconfig lan0 128.1.1.1 netmask 255.255.0.0 up
ifconfig lan0                     # show current config
ifconfig lan0 down                # bring down
ifconfig lan0 up                  # bring up
ifconfig lan0 unplumb             # remove the interface

# IPv6
ifconfig lan0 inet6 up
ifconfig lan0 inet6
```

### Persist IP Config in /etc/rc.config.d/netconf

```bash
INTERFACE_NAME[0]=lan0            # from lanscan/nwmgr
IP_ADDRESS[0]=128.1.1.1
SUBNET_MASK[0]=255.255.0.0
BROADCAST_ADDRESS[0]=""           # may be defaulted
INTERFACE_STATE[0]=""             # bring up at boot? default = up
DHCP_ENABLE[0]="0"                # "1" = DHCP sets the address
INTERFACE_MODULES[0]=""           # extra modules for the interface
```

```bash
/sbin/init.d/net start
```

A few things worth knowing about these arrays:

- `INTERFACE_NAME[n]` must match a real `lanN` (or `lanN:M` alias) from `lanscan`/`nwmgr`. A typo here is silently ignored at boot.
- Leaving `SUBNET_MASK` empty makes HP-UX fall back to the classful default for the address's class, which is almost never what you want on a subnetted network — always set it explicitly.
- `INTERFACE_STATE` defaults to `up`; set it to `down` to pre-stage an interface's config without bringing it online at boot.
- Set `DHCP_ENABLE[n]="1"` only when a DHCP server should assign the address; leave the static `IP_ADDRESS`/`SUBNET_MASK` blank in that case.

## IP Multiplexing (Interface Aliases)

Multiple IPs on one physical interface via `lan0:N` logical interfaces. This is how you host several IP addresses (for virtual hosts, service migration, or multiple subnets) on one NIC. The base interface is `lan0` (or, in `netconf`, `lan0:0`), and each additional address is `lan0:1`, `lan0:2`, and so on. All aliases share the physical link and MAC, so they all go up or down with the underlying `lan0`; bringing `lan0` down takes every alias with it. Aliases on the *same* subnet are common for service addresses; aliases on *different* subnets let one NIC participate in multiple networks (make sure the switch port is trunked/configured accordingly).

```bash
ifconfig lan0   129.1.1.1 netmask 255.255.0.0 up
ifconfig lan0:1 129.2.1.1 netmask 255.255.0.0 up
ifconfig lan0:2 129.3.1.1 netmask 255.255.0.0 up

ifconfig lan0:2                   # show
ifconfig lan0:1 up
ifconfig lan0:1 down
ifconfig lan0:2 0.0.0.0           # unconfigure a logical interface
```

Persist in `/etc/rc.config.d/netconf` with indexed arrays:

```bash
INTERFACE_NAME[0]=lan0:0
IP_ADDRESS[0]=129.1.1.1
SUBNET_MASK[0]=255.255.0.0

INTERFACE_NAME[1]=lan0:1
IP_ADDRESS[1]=129.2.1.1
SUBNET_MASK[1]=255.255.0.0

INTERFACE_NAME[2]=lan0:2
IP_ADDRESS[2]=129.3.1.1
SUBNET_MASK[2]=255.255.0.0
```

```bash
/sbin/init.d/net start
```

## Routing

```bash
# Host and network routes (last number = hop count/metric)
route add    host 129.1.1.1 128.1.0.1 1
route delete host 129.1.1.1 128.1.0.1
route add    net  129.1.0.0 netmask 255.255.0.0 128.1.0.1 1
route delete net  129.1.0.0 netmask 255.255.0.0 128.1.0.1

# Default route
route add    default 128.1.0.1 1
route delete default 128.1.0.1

# IPv6
route inet6 add 2345::1 4444::3
route inet6 add    net 2222::/64 4567::8 1
route inet6 delete net 2222::/64 4567::8 1

route -f                          # flush the routing table
netstat -rn                       # show the routing table (numeric)
```

The final number on each `route add` line is the *route metric* (hop count). Use `1` for a route whose gateway is on a directly attached subnet and `0` for an interface (directly connected) route; the value mainly matters when multiple routes could match and the kernel needs to prefer one. The kernel always selects the **most specific** matching route — a host route (`/32`) beats a network route, which beats the default route — so a `default` route is only consulted when nothing more specific matches.

A common ordering pitfall: you cannot add a route whose gateway isn't yet reachable. Bring the interface up (so its directly-connected network route exists), then add network/host routes through gateways on that network, and add `default` last. In `netconf`, the boot scripts process `ROUTE_*[n]` entries in index order, so list the default route after the specific ones if any depend on it.

`netstat -rn` **Flags**: `U` = up, `G` = uses a gateway, `H` = host route (vs network), `D` = created dynamically (redirect / PMTU), `M` = gateway route modified. `Refs` = active uses; `Interface` = the NIC; `Pmtu` = path MTU in bytes. The `-n` suppresses name resolution — always use it when troubleshooting so a slow/broken DNS server doesn't make `netstat` itself appear to hang.

### Persist Routes in /etc/rc.config.d/netconf

```bash
ROUTE_DESTINATION[0]="net 129.1.0.0"
ROUTE_MASK[0]="255.255.0.0"
ROUTE_GATEWAY[0]="128.1.0.1"
ROUTE_COUNT[0]="1"
ROUTE_ARGS[0]=""
ROUTE_SOURCE[0]=""

ROUTE_DESTINATION[1]="default"
ROUTE_MASK[1]=""
ROUTE_GATEWAY[1]="128.1.0.1"
ROUTE_COUNT[1]="1"
ROUTE_ARGS[1]=""
ROUTE_SOURCE[1]=""
```

```bash
/sbin/init.d/net start
```

## Network Tunables (ndd)

`ndd` tunes the TCP/IP stack itself — the `/dev/ip`, `/dev/tcp`, `/dev/udp`, `/dev/rawip`, and `/dev/arp` drivers. Each driver exposes named parameters that control behavior such as IP forwarding, default TTL, ARP timeouts, and TCP window/keepalive settings. These are runtime kernel knobs, not files, so a `-set` takes effect immediately but is lost on reboot unless recorded in `nddconf`.

```bash
ndd -h                            # list available tunables
ndd -h ip_forwarding              # describe a specific tunable
ndd -get /dev/ip ip_forwarding    # read a value
ndd -set /dev/ip ip_forwarding 1  # change a value (runtime)
ndd -c                            # re-read nddconf and apply to the running kernel
```

Frequently touched tunables:

```bash
ndd -get /dev/ip ip_forwarding          # 1 = act as a router between interfaces
ndd -get /dev/tcp tcp_keepalive_interval
ndd -get /dev/ip ip_ire_hash            # inspect route cache (diagnostics)
ndd -set /dev/arp arp_cleanup_interval 60000
```

Two cautions: setting `ip_forwarding` to `1` turns the host into a router, which is rarely desirable on an application server and can cause subtle asymmetric-routing problems — leave it `0` unless the box is intentionally a gateway. And always confirm the exact parameter name with `ndd -h`, because names and defaults vary across 11i v1/v2/v3.

Persist in `/etc/rc.config.d/nddconf`:

```bash
TRANSPORT_NAME[0]=ip
NDD_NAME[0]=ip_forwarding
NDD_VALUE[0]=0
```

```bash
/sbin/init.d/net start
```

## Hostname

The system hostname lives in the `HOSTNAME` variable of `/etc/rc.config.d/netconf` and is applied at boot by `/sbin/init.d/hostname`. HP-UX historically limited the *node name* (as returned by `uname -n` and `hostname`) to **8 characters**, while the fully qualified domain name in `/etc/hosts` and DNS can be longer. Keep the short name ≤ 8 characters to stay compatible with older tooling and licensing that keys off the node name.

```bash
vi /etc/rc.config.d/netconf       # set HOSTNAME
/sbin/init.d/hostname start
```

Setting the name at runtime with `hostname newname` changes only the running value and is lost at reboot, so always edit `netconf` for a permanent change. After changing the hostname, also update `/etc/hosts` and any name-service entries so forward and reverse lookups agree — a mismatch between the configured hostname and DNS is a frequent cause of slow logins and application startup delays. The order name resolution is attempted (files, DNS, NIS) is controlled by `/etc/nsswitch.conf`.

## Troubleshooting

```bash
lanscan                                       # interfaces / states
lanadmin                                      # link-layer stats (interactive)
netstat -in                                   # interface packet/error counts
netstat -rn                                   # routing table
linkloop -i 0 0x0060b007c179                  # layer-2 connectivity test (deprecated 11i v3)
nwmgr --diagnose --attribute dest=<mac> -c lan0   # 11i v3 layer-2 test
nslookup sanfran.example.com                  # DNS resolution
nsquery hosts sanfran.example.com             # name-service switch lookup
```

## Command Reference

| Task | 11i v1/v2 | 11i v3 |
|------|-----------|--------|
| List interfaces | `lanscan`, `ioscan -fC lan` | `nwmgr`, `ioscan -fC lan` |
| MAC / speed | `lanadmin -a`/`-x` | `nwmgr --get --attribute mac,speed` |
| Set speed/duplex | edit `netconf`, reboot | `nwmgr --set --attribute speed=... [--sa]` |
| Reset card | `lanadmin -r` | `nwmgr --reset -c lan0` |
| Stats | `lanadmin -g` | `nwmgr --get --stats all` |
| L2 connectivity test | `linkloop` | `nwmgr --diagnose` |
| Assign IP (temp) | `ifconfig` | `ifconfig` |
| Persist IP/route | `/etc/rc.config.d/netconf` | `/etc/rc.config.d/netconf` |
| Routing table | `netstat -rn` | `netstat -rn` |
| Tunables | `ndd` + `nddconf` | `ndd` + `nddconf` |
| Apply config | `/sbin/init.d/net start` | `/sbin/init.d/net start` |

## Related Articles

- [HP-UX Device Management and ioscan](articles/hpux-device-management-ioscan.md) — how the kernel discovers and claims the LAN hardware behind each `lanN`
- [HP-UX Startup and Services](articles/hpux-startup-and-services.md) — the RC scripts that read `netconf` and bring networking up at boot
- [HP-UX System Information](articles/hpux-system-information.md) — gathering hostname, interface, and general system detail
- [HP-UX NFS](articles/hpux-nfs.md) — a common consumer of correct network and name-resolution configuration
- [HP-UX Kernel Configuration](articles/hpux-kernel-configuration.md) — where persistent `ndd`/network tunables tie into kernel tuning
