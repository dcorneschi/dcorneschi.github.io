# nmcli Cheatsheet

`nmcli` is the command-line client for NetworkManager. It controls network connections, devices, Wi-Fi, VPNs, and DNS on RHEL 7+, CentOS, Fedora, and Ubuntu systems running NetworkManager.

## Packages and History

```bash
# nmcli is part of the NetworkManager package
# nmtui (text UI) is in NetworkManager-tui
sudo yum install NetworkManager NetworkManager-tui    # RHEL 7
sudo dnf install NetworkManager NetworkManager-tui    # RHEL 8+

sudo systemctl enable --now NetworkManager
```

| RHEL Version | NetworkManager Status |
|--------------|----------------------|
| RHEL 6 | Introduced (optional) |
| RHEL 7 | Optional (network-scripts still default) |
| RHEL 8 | Default network management |
| RHEL 9+ | Only option (ifcfg deprecated, keyfile format) |

Three ways to manage connections:
- `nmcli` — command line (scriptable, this cheatsheet)
- `nmtui` — text UI (interactive, menu-driven)
- `nm-connection-editor` — graphical UI

## General Status

```bash
# Overall NetworkManager status
nmcli general status

# Show hostname
nmcli general hostname

# Set hostname
sudo nmcli general hostname server01.example.com

# Show all connections and devices in one view
nmcli

# Check if NetworkManager is running
nmcli -t -f RUNNING general

# Show NetworkManager version
nmcli --version

# Show permissions
nmcli general permissions

# Show logging level
nmcli general logging
```

## Connections

### List Connections

```bash
# Show all connections (active and inactive)
nmcli connection show

# Show only active connections
nmcli connection show --active

# Show details of a specific connection
nmcli connection show "Wired connection 1"
nmcli connection show ens192

# Terse output (scriptable, colon-separated)
nmcli -t connection show

# Show specific fields only
nmcli -t -f NAME,UUID,TYPE,DEVICE connection show
```

### Create Connections

```bash
# Create a static IPv4 connection
sudo nmcli connection add \
  con-name "eth0-static" \
  ifname eth0 \
  type ethernet \
  ipv4.addresses 192.168.1.10/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8,8.8.4.4" \
  ipv4.dns-search "example.com" \
  ipv4.method manual

# Create a DHCP connection
sudo nmcli connection add \
  con-name "eth0-dhcp" \
  ifname eth0 \
  type ethernet \
  ipv4.method auto

# Create a connection with IPv4 and IPv6
sudo nmcli connection add \
  con-name "dual-stack" \
  ifname eth0 \
  type ethernet \
  ipv4.addresses 192.168.1.10/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.method manual \
  ipv6.addresses "fd00::10/64" \
  ipv6.method manual

# Create a VLAN connection
sudo nmcli connection add \
  con-name "vlan100" \
  ifname vlan100 \
  type vlan \
  vlan.parent eth0 \
  vlan.id 100 \
  ipv4.addresses 10.100.0.10/24 \
  ipv4.method manual

# Create a bridge connection
sudo nmcli connection add \
  con-name "br0" \
  ifname br0 \
  type bridge \
  ipv4.addresses 192.168.1.10/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.method manual

# Add an interface to the bridge
sudo nmcli connection add \
  con-name "br0-port1" \
  ifname eth0 \
  type bridge-slave \
  master br0

# Create a bond connection
sudo nmcli connection add \
  con-name "bond0" \
  ifname bond0 \
  type bond \
  bond.options "mode=active-backup,miimon=100" \
  ipv4.addresses 192.168.1.10/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.method manual

# Add slave interfaces to the bond
sudo nmcli connection add con-name "bond0-slave1" ifname eth0 type bond-slave master bond0
sudo nmcli connection add con-name "bond0-slave2" ifname eth1 type bond-slave master bond0

# Create a team connection
sudo nmcli connection add \
  con-name "team0" \
  ifname team0 \
  type team \
  team.runner activebackup \
  ipv4.addresses 192.168.1.10/24 \
  ipv4.method manual

# Add slave interfaces to the team
sudo nmcli connection add con-name "team0-port1" ifname eth0 type team-slave master team0
sudo nmcli connection add con-name "team0-port2" ifname eth1 type team-slave master team0
```

### Inspect Teams and Bridges

```bash
# View teaming port status
teamnl team0 ports

# View teaming state (active runner, link watches)
teamdctl team0 state dump

# View teaming configuration (JSON)
teamdctl team0 config dump

# Show bridge configuration via nmcli
nmcli -f bridge connection show br0

# Show bridge status (brief)
brctl show br0

# Show bridge forwarding table
brctl showmacs br0
```

### Modify Connections

```bash
# Rename a connection
sudo nmcli connection modify "Wired connection 3" con-name ens3

# Change IP address
sudo nmcli connection modify ens192 ipv4.addresses 192.168.1.20/24

# Add a secondary IP address
sudo nmcli connection modify ens192 +ipv4.addresses 192.168.1.21/24

# Remove a secondary IP address
sudo nmcli connection modify ens192 -ipv4.addresses 192.168.1.21/24

# Change gateway
sudo nmcli connection modify ens192 ipv4.gateway 192.168.1.254

# Change DNS servers
sudo nmcli connection modify ens192 ipv4.dns "8.8.8.8,1.1.1.1"

# Add a DNS server
sudo nmcli connection modify ens192 +ipv4.dns "9.9.9.9"

# Remove a DNS server
sudo nmcli connection modify ens192 -ipv4.dns "9.9.9.9"

# Set DNS search domain
sudo nmcli connection modify ens192 ipv4.dns-search "example.com,internal.lan"

# Set DNS options (rotate servers, timeout)
sudo nmcli connection modify ens192 ipv4.dns-options "rotate,timeout:1"

# Switch from DHCP to static
sudo nmcli connection modify ens192 ipv4.method manual ipv4.addresses 192.168.1.10/24 ipv4.gateway 192.168.1.1

# Switch from static to DHCP
sudo nmcli connection modify ens192 ipv4.method auto
sudo nmcli connection modify ens192 ipv4.addresses "" ipv4.gateway ""

# Disable IPv6
sudo nmcli connection modify ens192 ipv6.method disabled

# Set IPv6 to link-local only
sudo nmcli connection modify ens192 ipv6.method link-local

# Set connection to autoconnect on boot
sudo nmcli connection modify ens192 connection.autoconnect yes

# Disable autoconnect
sudo nmcli connection modify ens192 connection.autoconnect no

# Set MTU
sudo nmcli connection modify ens192 ethernet.mtu 9000
# Alternative property name (same effect)
sudo nmcli connection modify ens192 802-3-ethernet.mtu 9000

# Set MAC address (cloning)
sudo nmcli connection modify ens192 ethernet.cloned-mac-address "AA:BB:CC:DD:EE:FF"

# Add static route
sudo nmcli connection modify ens192 +ipv4.routes "10.0.0.0/8 192.168.1.254"

# Add static route with metric
sudo nmcli connection modify ens192 +ipv4.routes "10.0.0.0/8 192.168.1.254 100"

# Remove a static route
sudo nmcli connection modify ens192 -ipv4.routes "10.0.0.0/8 192.168.1.254"

# Set connection priority (lower = preferred)
sudo nmcli connection modify ens192 ipv4.route-metric 100

# Never use this connection as default route
sudo nmcli connection modify ens192 ipv4.never-default yes
```

### Activate and Deactivate

```bash
# Bring up a connection
sudo nmcli connection up ens192

# Bring up by UUID
sudo nmcli connection up uuid 12345678-abcd-efgh-ijkl-123456789012

# Bring down a connection
sudo nmcli connection down ens192

# Reload connection files from disk (after manual edits)
sudo nmcli connection reload

# Load a specific connection file
sudo nmcli connection load /etc/NetworkManager/system-connections/ens192.nmconnection

# Delete a connection
sudo nmcli connection delete "eth0-static"
sudo nmcli connection delete uuid 12345678-abcd-efgh-ijkl-123456789012
```

## Devices

```bash
# List all network devices
nmcli device status

# Show detailed info for a device
nmcli device show eth0

# Show all device details
nmcli device show

# Connect a device (auto-activate best connection)
sudo nmcli device connect eth0

# Disconnect a device
sudo nmcli device disconnect eth0

# Set a device as managed by NetworkManager
sudo nmcli device set eth0 managed yes

# Set a device as unmanaged
sudo nmcli device set eth0 managed no

# Reapply connection settings without down/up
sudo nmcli device reapply eth0

# Delete all device connections
sudo nmcli device delete br0
```

## Wi-Fi

```bash
# List available Wi-Fi networks
nmcli device wifi list

# Rescan for Wi-Fi networks
nmcli device wifi rescan

# Connect to a Wi-Fi network
sudo nmcli device wifi connect "MyNetwork" password "MyPassword"

# Connect to a hidden network
sudo nmcli device wifi connect "HiddenSSID" password "MyPassword" hidden yes

# Connect to a specific BSSID
sudo nmcli device wifi connect "MyNetwork" password "MyPassword" bssid AA:BB:CC:DD:EE:FF

# Show Wi-Fi status
nmcli radio wifi

# Enable Wi-Fi
sudo nmcli radio wifi on

# Disable Wi-Fi
sudo nmcli radio wifi off

# Show saved Wi-Fi connections
nmcli connection show | grep wifi

# Show Wi-Fi password for a saved connection
sudo nmcli -s connection show "MyNetwork" | grep psk
```

## DNS

```bash
# Show current DNS configuration
nmcli device show | grep DNS

# Show DNS for a specific connection
nmcli -f IP4.DNS connection show ens192

# Set DNS servers
sudo nmcli connection modify ens192 ipv4.dns "8.8.8.8 1.1.1.1"

# Prevent DHCP from overwriting DNS (ignore DHCP-provided DNS)
sudo nmcli connection modify ens192 ipv4.ignore-auto-dns yes

# Re-enable DHCP DNS
sudo nmcli connection modify ens192 ipv4.ignore-auto-dns no
```

## Proxy

```bash
# Set proxy via PAC URL
sudo nmcli connection modify ens192 proxy.method auto
sudo nmcli connection modify ens192 proxy.pac-url "http://proxy.example.com/proxy.pac"

# Set proxy via inline PAC script
sudo nmcli connection modify ens192 proxy.method auto
sudo nmcli connection modify ens192 proxy.pac-script "function FindProxyForURL(url,host) { return \"PROXY proxy.example.com:8080\"; }"

# Remove proxy settings
sudo nmcli connection modify ens192 proxy.method none
```

Note: NetworkManager proxy support is limited to PAC (Proxy Auto-Configuration). For manual HTTP/HTTPS proxy, configure via environment variables or `/etc/environment` instead.

## Output Formatting

```bash
# Terse output (for scripting — colon-separated fields)
nmcli -t connection show

# Specific fields
nmcli -t -f NAME,DEVICE,STATE connection show

# Tabular output (default)
nmcli connection show

# Multiline output (key: value pairs)
nmcli -m multiline connection show ens192

# Show specific property of a connection
nmcli -g ipv4.addresses connection show ens192
nmcli -g ipv4.dns connection show ens192

# Colors off (useful for piping)
nmcli -c no connection show

# Pretty output with all fields
nmcli -p connection show ens192
```

## Connection Files

```bash
# RHEL 9+ / Fedora (keyfile format)
ls /etc/NetworkManager/system-connections/
cat /etc/NetworkManager/system-connections/ens192.nmconnection

# RHEL 7-8 (ifcfg format — deprecated in RHEL 9)
ls /etc/sysconfig/network-scripts/
cat /etc/sysconfig/network-scripts/ifcfg-ens192

# Migrate ifcfg to keyfile format (RHEL 9)
sudo nmcli connection migrate

# Reload after manual file edits
sudo nmcli connection reload
```

### Keyfile Format Example (RHEL 9+)

```ini
# /etc/NetworkManager/system-connections/ens192.nmconnection
[connection]
id=ens192
type=ethernet
interface-name=ens192
autoconnect=true

[ipv4]
method=manual
addresses=192.168.1.10/24
gateway=192.168.1.1
dns=8.8.8.8;1.1.1.1;
dns-search=example.com;

[ipv6]
method=disabled
```

## Troubleshooting

```bash
# Restart NetworkManager
sudo systemctl restart NetworkManager

# Check NetworkManager status
systemctl status NetworkManager

# View NetworkManager logs
journalctl -u NetworkManager --since "10 minutes ago"
journalctl -u NetworkManager -f    # follow live

# Increase logging level temporarily
sudo nmcli general logging level DEBUG domain ALL
# Reset to default
sudo nmcli general logging level INFO domain ALL

# Check if a device is managed
nmcli device status | grep eth0

# Check connection file syntax
sudo nmcli connection load /etc/NetworkManager/system-connections/ens192.nmconnection

# Force connection re-read
sudo nmcli connection reload && sudo nmcli connection up ens192
```

## One-Liners

```bash
# Get IP address of a specific interface
nmcli -g IP4.ADDRESS device show eth0

# Get default gateway
nmcli -g IP4.GATEWAY device show eth0

# Get DNS servers for all connections
nmcli -t -f IP4.DNS device show

# List all interface IPs (terse, scriptable)
nmcli -t -f DEVICE,IP4.ADDRESS device show | grep -v "^$"

# Quick static IP setup (one command)
sudo nmcli connection modify ens192 ipv4.addresses 192.168.1.10/24 ipv4.gateway 192.168.1.1 ipv4.dns "8.8.8.8" ipv4.method manual && sudo nmcli connection up ens192

# Check if interface has link
nmcli -t -f DEVICE,STATE device status | grep eth0

# Get connection UUID for scripting
nmcli -t -f NAME,UUID connection show | grep ens192 | cut -d: -f2

# Show all IPs across all interfaces
nmcli -t -f IP4.ADDRESS device show | grep -v "^$" | sort

# Find which connection is using which device
nmcli -t -f NAME,DEVICE connection show --active

# Export connection as a single line (for documentation)
nmcli -t connection show ens192 | grep -E "ipv4\.(addresses|gateway|dns|method)"

# Batch modify multiple connections
for conn in $(nmcli -t -f NAME connection show | grep eth); do
  sudo nmcli connection modify "$conn" ipv4.dns "8.8.8.8,1.1.1.1"
done

# Wait for a connection to be active (useful in scripts)
nmcli connection monitor ens192 &
sudo nmcli connection up ens192
wait

# Show connection speed/duplex
nmcli -f GENERAL.SPEED device show eth0
```

## Tips

- Use `nmcli connection up <name>` after `modify` — changes don't take effect until the connection is reactivated
- Use `+` prefix to add values and `-` prefix to remove values from list properties (addresses, DNS, routes)
- `nmcli -t` (terse) is your friend for scripting — colon-separated, no headers
- `nmcli -g` (get) returns just the value of a specific field — cleanest for single-value lookups
- Tab completion works with nmcli — type `nmcli con mod <tab>` to see connection names
- On RHEL 9+, connection files are keyfile format (`.nmconnection`) — ifcfg is deprecated
- `nmcli device reapply` applies changes without full disconnect/reconnect (minimizes downtime)
- `nmcli connection reload` is needed only after manual file edits — `nmcli modify` saves automatically
- Use `connection.autoconnect-priority` to control which connection activates when multiple match one device
- NetworkManager ignores interfaces listed in `/etc/NetworkManager/conf.d/` with `unmanaged-devices`
