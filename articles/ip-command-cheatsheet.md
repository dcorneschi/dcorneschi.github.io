# ip Command Cheatsheet

The `ip` command (from the `iproute2` package) is the modern replacement for legacy `ifconfig`, `route`, `arp`, and `netstat` tools on Linux. It manages network interfaces, addresses, routes, neighbours, and multicast.

## IP Queries

### addr — Display IP Addresses

```bash
ip addr                               # Show information for all addresses
ip addr show dev em1                  # Display information only for device em1
```

### link — Manage and Display Network Interfaces

```bash
ip link                               # Show information for all interfaces
ip link show dev em1                  # Display information only for device em1
ip -s link                            # Display interface statistics
```

### route — Display the Routing Table

```bash
ip route                              # List all route entries in the kernel
```

### maddr — Display Multicast Addresses

```bash
ip maddr                              # Display multicast information for all devices
ip maddr show dev em1                 # Display multicast information for device em1
```

### neigh — Show Neighbour Objects (ARP Table)

```bash
ip neigh                              # Display neighbour objects
ip neigh show dev em1                 # Show the ARP cache for device em1
```

### help — Display Commands and Arguments

```bash
ip help                               # Display ip commands and arguments
ip addr help                          # Display address commands and arguments
ip link help                          # Display link commands and arguments
ip neigh help                         # Display neighbour commands and arguments
```

---

## Modifying Address and Link Properties

### addr add — Add an Address

```bash
ip addr add 192.168.1.1/24 dev em1    # Add address 192.168.1.1 with netmask 24 to device em1
```

### addr del — Delete an Address

```bash
ip addr del 192.168.1.1/24 dev em1    # Remove address 192.168.1.1/24 from device em1
```

### link set — Alter Interface Status

```bash
ip link set em1 up                    # Bring em1 online
ip link set em1 down                  # Bring em1 offline
ip link set em1 mtu 9000             # Set the MTU on em1 to 9000
ip link set em1 promisc on           # Enable promiscuous mode for em1
```

---

## Adjusting and Viewing Routes

### route add — Add a Route

```bash
ip route add default via 192.168.1.1 dev em1       # Add default route via gateway 192.168.1.1 on em1
ip route add 192.168.1.0/24 via 192.168.1.1        # Add route to 192.168.1.0/24 via gateway 192.168.1.1
ip route add 192.168.1.0/24 dev em1                # Add route to 192.168.1.0/24 reachable on em1
```

### route delete — Delete a Route

```bash
ip route delete 192.168.1.0/24 via 192.168.1.1     # Delete route for 192.168.1.0/24 via gateway 192.168.1.1
```

### route replace — Replace or Add a Route

```bash
ip route replace 192.168.1.0/24 dev em1            # Replace the route for 192.168.1.0/24 to use device em1
```

### route get — Display the Route for an Address

```bash
ip route get 192.168.1.5             # Display the route taken for IP 192.168.1.5
```

---

## Managing the ARP Table

### neigh add — Add an ARP Entry

```bash
ip neigh add 192.168.1.1 lladdr 1:2:3:4:5:6 dev em1   # Add address 192.168.1.1 with MAC 1:2:3:4:5:6 to em1
```

### neigh del — Invalidate an Entry

```bash
ip neigh del 192.168.1.1 dev em1     # Invalidate the entry for 192.168.1.1 on em1
```

### neigh replace — Replace or Add an Entry

```bash
ip neigh replace 192.168.1.1 lladdr 1:2:3:4:5:6 dev em1   # Replace entry for 192.168.1.1 to use MAC 1:2:3:4:5:6 on em1
```

---

## Multicast Addressing

### maddr add — Add a Multicast Address

```bash
ip maddr add 33:33:00:00:00:01 dev em1   # Add multicast address 33:33:00:00:00:01 to em1
```

### maddr del — Delete a Multicast Address

```bash
ip maddr del 33:33:00:00:00:01 dev em1   # Delete address 33:33:00:00:00:01 from em1
```

---

## Useful Networking Commands

### arping — Send ARP Requests

```bash
arping -I eth0 192.168.1.1           # Send ARP request to 192.168.1.1 via interface eth0
arping -D -I eth0 192.168.1.1       # Check for duplicate MAC addresses at 192.168.1.1 on eth0
```

### ethtool — Query Network Driver and Hardware

```bash
ethtool -g eth0                       # Display ring buffer for eth0
ethtool -i eth0                       # Display driver information for eth0
ethtool -p eth0                       # Identify eth0 by sight (blink LEDs on the network port)
ethtool -S eth0                       # Display network and driver statistics for eth0
```

### ss — Display Socket Statistics

```bash
ss -a                                 # Show all sockets (listening and non-listening)
ss -e                                 # Show detailed socket information
ss -o                                 # Show timer information
ss -n                                 # Do not resolve addresses
ss -p                                 # Show process using the socket
```

---

## Comparing net-tools vs iproute2

| net-tools Command | iproute2 Equivalent |
|-------------------|---------------------|
| `arp -a` | `ip neigh` |
| `arp -v` | `ip -s neigh` |
| `arp -s 192.168.1.1 1:2:3:4:5:6` | `ip neigh add 192.168.1.1 lladdr 1:2:3:4:5:6 dev eth1` |
| `arp -i eth1 -d 192.168.1.1` | `ip neigh del 192.168.1.1 dev eth1` |
| `ifconfig -a` | `ip addr` |
| `ifconfig eth0 down` | `ip link set eth0 down` |
| `ifconfig eth0 up` | `ip link set eth0 up` |
| `ifconfig eth0 192.168.1.1` | `ip addr add 192.168.1.1/24 dev eth0` |
| `ifconfig eth0 netmask 255.255.255.0` | `ip addr add 192.168.1.1/24 dev eth0` |
| `ifconfig eth0 mtu 9000` | `ip link set eth0 mtu 9000` |
| `ifconfig eth0:0 192.168.1.2` | `ip addr add 192.168.1.2/24 dev eth0` |
| `netstat` | `ss` |
| `netstat -neopa` | `ss -neopa` |
| `netstat -g` | `ip maddr` |
| `route` | `ip route` |
| `route add -net 192.168.1.0 netmask 255.255.255.0 dev eth0` | `ip route add 192.168.1.0/24 dev eth0` |
| `route add default gw 192.168.1.1` | `ip route add default via 192.168.1.1` |

---

## Advanced Usage

### VLANs

```bash
ip link add link eth0 name eth0.100 type vlan id 100   # Create VLAN 100 on eth0
ip link set eth0.100 up                                # Bring the VLAN interface up
ip link delete eth0.100                                # Remove the VLAN interface
```

### Bridge Interfaces

```bash
ip link add br0 type bridge                            # Create a bridge
ip link set eth0 master br0                            # Add eth0 to bridge br0
ip link set eth0 nomaster                              # Remove eth0 from bridge
ip link set br0 up                                     # Bring bridge up
ip link delete br0 type bridge                         # Delete the bridge
```

### Bonding

```bash
ip link add bond0 type bond mode active-backup         # Create bond in active-backup mode
ip link set eth0 master bond0                          # Add eth0 to bond
ip link set eth1 master bond0                          # Add eth1 to bond
ip link set bond0 up                                   # Bring bond up
```

### Tunnel Interfaces

```bash
ip tunnel add gre1 mode gre remote 10.0.0.1 local 10.0.0.2   # Create GRE tunnel
ip link set gre1 up
ip addr add 172.16.0.1/30 dev gre1

ip link add vxlan0 type vxlan id 42 remote 10.0.0.1 dstport 4789   # Create VXLAN
ip link set vxlan0 up
```

### Network Namespaces

```bash
ip netns add mynamespace                               # Create a network namespace
ip netns list                                          # List all namespaces
ip netns exec mynamespace ip addr                     # Run command inside namespace
ip link set eth0 netns mynamespace                    # Move interface to namespace
ip netns del mynamespace                              # Delete namespace

# Create a veth pair connecting two namespaces
ip link add veth0 type veth peer name veth1
ip link set veth1 netns mynamespace
ip addr add 10.0.0.1/24 dev veth0
ip netns exec mynamespace ip addr add 10.0.0.2/24 dev veth1
ip link set veth0 up
ip netns exec mynamespace ip link set veth1 up
```

### Policy Routing (Multiple Routing Tables)

```bash
ip rule list                                           # Show all routing rules
ip rule add from 192.168.1.0/24 table 100             # Route traffic from subnet via table 100
ip route add default via 10.0.0.1 table 100           # Add default route in table 100
ip rule add fwmark 1 table 100                        # Route marked packets via table 100
ip rule del from 192.168.1.0/24 table 100             # Remove a rule
```

### Traffic Control Basics

```bash
ip link set eth0 txqueuelen 10000                     # Set transmit queue length
ip -s -d link show eth0                               # Show detailed stats with drops/errors
```

---

## Output Formatting

```bash
ip -br addr                                           # Brief output — one line per interface
ip -br link                                           # Brief link status
ip -4 addr                                            # Show only IPv4 addresses
ip -6 addr                                            # Show only IPv6 addresses
ip -j route                                           # JSON output (useful for scripting)
ip -j -p addr                                         # JSON pretty-printed
ip -c addr                                            # Colourised output
ip -o addr                                            # One-line per address (parseable)
ip -d link show eth0                                  # Detailed link info (driver, features)
ip -ts neigh                                          # Show timestamps on neighbour entries
```

---

## Useful One-Liners

```bash
# Show primary IP of default interface
ip route get 1.1.1.1 | awk '{print $7; exit}'

# Get the default gateway
ip route | awk '/default/ {print $3}'

# Get the interface used for default route
ip route | awk '/default/ {print $5}'

# List all IPs on the system (one per line)
ip -o -4 addr show | awk '{print $4}' | cut -d/ -f1

# List all interfaces with their MAC addresses
ip -o link show | awk '{print $2, $(NF-2)}'

# Flush all addresses on an interface
ip addr flush dev eth0

# Flush the routing table
ip route flush table main

# Flush the ARP cache
ip neigh flush all

# Flush ARP cache for a specific interface
ip neigh flush dev eth0

# Watch for neighbour (ARP) changes in real time
ip monitor neigh

# Monitor all network events (links, addresses, routes)
ip monitor all

# Monitor only route changes
ip monitor route

# Show interfaces that are UP
ip -br link show up

# Show interfaces that are DOWN
ip -br link show type ether | grep DOWN

# Add multiple IPs to one interface
for i in $(seq 1 10); do ip addr add 192.168.1.$i/24 dev eth0; done

# Check if an interface exists
ip link show dev eth0 &>/dev/null && echo "exists" || echo "missing"

# Show the MAC address of a specific interface
ip link show eth0 | awk '/ether/ {print $2}'

# Display routing cache statistics
ip -s route show cache

# Show all routes for a specific subnet
ip route show match 10.0.0.0/8

# List routes learned via a specific protocol
ip route show proto static
ip route show proto dhcp

# Add a blackhole route (drop traffic)
ip route add blackhole 10.10.10.0/24

# Add an unreachable route (return ICMP unreachable)
ip route add unreachable 10.10.10.0/24

# Temporarily add an IP (survives until reboot or flush)
ip addr add 10.0.0.100/24 dev eth0 valid_lft 3600 preferred_lft 1800
```

---

## Persistent Configuration

The `ip` command makes **temporary** changes that don't survive a reboot. To persist:

| Distribution | Method |
|-------------|--------|
| RHEL 7 | `/etc/sysconfig/network-scripts/ifcfg-*` |
| RHEL 8+ | `nmcli` / NetworkManager connection files |
| Ubuntu (netplan) | `/etc/netplan/*.yaml` then `netplan apply` |
| Debian (ifupdown) | `/etc/network/interfaces` |
| systemd-networkd | `/etc/systemd/network/*.network` |

### nmcli equivalents for common ip commands

```bash
# Add IP address
nmcli con mod "System eth0" +ipv4.addresses 192.168.1.1/24

# Set default gateway
nmcli con mod "System eth0" ipv4.gateway 192.168.1.254

# Add static route
nmcli con mod "System eth0" +ipv4.routes "10.0.0.0/8 192.168.1.1"

# Apply changes
nmcli con up "System eth0"
```

---

## Tips and Best Practices

1. **Use `ip -br` for quick status checks** — cleaner than parsing full `ip addr` output
2. **Always use `ip -c`** for colour in interactive sessions — makes errors and states obvious
3. **Prefer `ip -j`** for scripting — stable JSON output is easier to parse than text
4. **Use `ip monitor`** for debugging — watch real-time changes to links, addresses, routes, and neighbours
5. **Remember changes are ephemeral** — `ip` commands don't persist across reboots unless saved via NetworkManager, netplan, or ifupdown
6. **Use `ip route get`** to troubleshoot routing — shows exactly which route and interface a packet will use
7. **Flush before reconfiguring** — `ip addr flush dev eth0` avoids stale addresses stacking up
8. **Use `-o` for scripting** — one-line output is reliable for `awk`/`cut` parsing
9. **Check with `ip route get`** after adding routes — confirms the kernel accepted and prefers your route
10. **Use network namespaces for testing** — isolate experiments without touching production interfaces
