# Solaris 11: Configure a Static IP with ipadm and dladm

Oracle Solaris 11 manages networking through `dladm`/`ipadm` and SMF services rather than the legacy `/etc/hostname.<if>` files used on Solaris 10. By default a fresh install uses **Network Auto-Magic (NWAM)**, which is DHCP/automatic. This guide switches to a **fixed (static) configuration**: selecting the DefaultFixed profile, assigning a static address, setting the default route, and configuring DNS and name-service switch via SMF.

> For the Solaris 10 / legacy file-based approach (`/etc/hostname.<if>`, `/etc/defaultrouter`, etc.), see [Solaris Network Configuration Files](articles/solaris-network-configuration.md).

## From DHCP/NWAM to a Static IP

Solaris 11 has two network configuration profiles: **Automatic** (NWAM, used for DHCP/laptops) and **DefaultFixed** (manual, used for servers). Switch to DefaultFixed first, then configure the interface.

```bash
# List available network configuration profiles (NCPs)
netadm list

# Switch to the fixed (manual) profile
netadm enable -p ncp DefaultFixed

# Show physical data links (find your NIC, e.g. net0)
dladm show-phys

# Create the IP interface on the data link
ipadm create-ip net0

# Assign a static IPv4 address (CIDR notation)
ipadm create-addr -T static -a local=192.168.1.141/24 net0/v4

# Verify the address and links
ipadm show-addr
dladm show-link

# Add a persistent default route (-p survives reboot)
route -p add default 192.168.1.1
```

- `netadm enable -p ncp DefaultFixed` — activates the manual profile; NWAM stops managing interfaces.
- `dladm show-phys` — lists physical NICs and their link state; `net0` is the typical first interface name.
- `ipadm create-ip <link>` — creates the IP interface object over the data link.
- `ipadm create-addr -T static -a local=<ip/prefix> <if>/v4` — assigns the static address; the `/v4` label names this address object.
- `route -p add default <gw>` — `-p` makes the route persistent across reboots (stored in the persistent route DB).

Sample `dladm show-phys`:

```
LINK              MEDIA                STATE      SPEED  DUPLEX    DEVICE
net0              Ethernet             up         1000   full      e1000g0
net1              Ethernet             unknown    0      unknown   e1000g1
```

Sample `ipadm show-addr` after configuration:

```
ADDROBJ           TYPE     STATE        ADDR
lo0/v4            static   ok           127.0.0.1/8
net0/v4           static   ok           192.168.1.141/24
lo0/v6            static   ok           ::1/128
```

The `ADDROBJ` (`net0/v4`) is the address-object name you use to manage or delete that address later (`ipadm delete-addr net0/v4`).

### DHCP Instead of Static

To make an interface use DHCP rather than a static address:

```bash
ipadm create-ip net0
ipadm create-addr -T dhcp net0/dhcp
```

### VNICs and Link Aggregation

```bash
# Virtual NIC over a physical link (used for exclusive-IP zones)
dladm create-vnic -l net0 vnic0
dladm show-vnic

# Link aggregation (bonding) across two NICs
dladm create-aggr -l net0 -l net1 aggr0
ipadm create-ip aggr0
ipadm create-addr -T static -a 192.168.1.141/24 aggr0/v4
```

## DNS Client (SMF)

DNS resolver settings live in the `network/dns/client` SMF service, not directly in `/etc/resolv.conf` (SMF generates that file):

```bash
# Set the nameserver(s)
svccfg -s network/dns/client setprop config/nameserver = 192.168.1.1

# Review the current config
svccfg -s network/dns/client listprop config

# Apply and restart the service
svcadm refresh /network/dns/client
svcadm restart /network/dns/client
```

To set multiple nameservers or a search domain, use the list/net_address forms, e.g.:

```bash
svccfg -s network/dns/client setprop config/nameserver = net_address: "(192.168.1.1 8.8.8.8)"
svccfg -s network/dns/client setprop config/search = astring: "(example.com)"
svcadm refresh network/dns/client
```

## Name Service Switch (nsswitch)

On Solaris 11 the `nsswitch.conf` content is managed through the `name-service/switch` SMF service. Set the `host` lookup order to consult local files then DNS:

```bash
# Set host resolution order to "files dns"
svccfg -s name-service/switch setprop config/host = astring: '("files dns")'

# Review
svccfg -s name-service/switch listprop config

# Apply
svcadm refresh name-service/switch
svcadm restart name-service/switch
```

After `svcadm refresh`, SMF regenerates `/etc/nsswitch.conf` from these properties — don't hand-edit that file on Solaris 11, as changes get overwritten.

## Verify the Configuration

```bash
ipadm show-addr                     # address objects and states
ipadm show-if                       # interface states
netstat -rn                         # routing table (confirm default route)
cat /etc/resolv.conf                # SMF-generated DNS config
getent hosts www.oracle.com         # end-to-end name resolution test
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `ipadm create-addr` fails "interface disabled" | Still on the Automatic (NWAM) profile | `netadm enable -p ncp DefaultFixed` first |
| Address disappears after reboot | Created in a temporary manner | Recreate without `-t`; confirm with `ipadm show-addr` (STATE `ok`) |
| DNS not resolving | dns/client not refreshed, or nsswitch lacks dns | `svcadm refresh network/dns/client`; set `config/host = "files dns"` |
| `/etc/resolv.conf` reverts | Hand-edited instead of SMF | Configure via `svccfg network/dns/client` |
| Default route missing after reboot | Added without `-p` | `route -p add default <gw>` |
| Duplicate address error | Address already on another host | Change the address; check with `arp`/`ping` |

```bash
# Confirm the active network profile
netadm list -p ncp
```

## Command Reference

| Task | Command |
|------|---------|
| List NCPs | `netadm list` |
| Switch to manual profile | `netadm enable -p ncp DefaultFixed` |
| Show physical links | `dladm show-phys` |
| Show data links | `dladm show-link` |
| Create IP interface | `ipadm create-ip net0` |
| Assign static address | `ipadm create-addr -T static -a local=IP/CIDR net0/v4` |
| Show addresses | `ipadm show-addr` |
| Persistent default route | `route -p add default GW` |
| Set DNS nameserver | `svccfg -s network/dns/client setprop config/nameserver = IP` |
| Apply DNS | `svcadm refresh/restart network/dns/client` |
| Set nsswitch host order | `svccfg -s name-service/switch setprop config/host = astring: '("files dns")'` |
| Apply nsswitch | `svcadm refresh/restart name-service/switch` |

## ipadm/dladm vs Legacy Files

| Legacy (Solaris 10) | Solaris 11 (this article) |
|---------------------|---------------------------|
| `/etc/hostname.<if>` | `ipadm create-ip` + `ipadm create-addr` |
| `/etc/defaultrouter` | `route -p add default` |
| `/etc/inet/netmasks` | CIDR prefix in `create-addr` |
| Edit `/etc/resolv.conf` | `svccfg network/dns/client` |
| Edit `/etc/nsswitch.conf` | `svccfg name-service/switch` |
| `svcadm restart network/physical` | `netadm` profile + SMF refresh |

## References

- [Configuring and Administering Network Components in Oracle Solaris 11](https://docs.oracle.com/cd/E37838_01/html/E60988/index.html) — official Oracle docs
- [ipadm(1M) man page](https://docs.oracle.com/cd/E36784_01/html/E36871/ipadm-1m.html) — official Oracle docs
- [dladm(1M) man page](https://docs.oracle.com/cd/E36784_01/html/E36871/dladm-1m.html) — official Oracle docs
