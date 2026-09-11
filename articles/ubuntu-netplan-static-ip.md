# Configure a Static IP with Netplan on Ubuntu

Change an Ubuntu network interface from DHCP to a persistent static IPv4 address using Netplan. This guide includes safe remote application, automatic rollback, cloud-init considerations, and troubleshooting.

> **Warning:** Changing network settings can immediately disconnect an SSH session. Use `netplan try`, keep console or hypervisor access available, and confirm that the address, prefix, gateway, and DNS servers are correct before applying the configuration.

## How Netplan Configuration Works

Netplan reads YAML files from:

```text
/etc/netplan/*.yaml
```

Configuration filenames can be chosen freely but normally begin with a number and end in `.yaml`, such as:

```text
01-network-manager-all.yaml
50-cloud-init.yaml
99-static-ip.yaml
```

Files are processed in lexicographic order. Later files can override matching settings from earlier files, so `99-static-ip.yaml` is useful for a local override.

YAML indentation is significant:

- Use spaces, not tabs.
- Keep child keys consistently indented.
- Include the CIDR prefix in a static address, such as `/24`.
- Restrict configuration-file permissions to prevent Netplan warnings and protect sensitive settings.

## Collect the Current Network Settings

Identify the interface name, current address, default gateway, and DNS configuration:

```bash
ip -br link
ip -br address
ip route show
resolvectl status
ls -la /etc/netplan/
```

Ubuntu may use a predictable interface name such as `ens18`, `ens160`, or `enp1s0` instead of `eth0`. Use the actual name shown by `ip -br link`.

Before assigning an address, confirm that:

1. The address is reserved for this host and is not in use.
2. The CIDR prefix matches the subnet.
3. The gateway is reachable from that subnet.
4. The DNS servers are reachable.
5. The address is outside the DHCP pool or reserved in DHCP.

The examples below use:

| Setting | Example value |
|---------|---------------|
| Interface | `eth0` |
| Static address | `192.168.50.209/24` |
| Default gateway | `192.168.50.1` |
| Primary DNS | `192.168.50.1` |
| Secondary DNS | `1.1.1.1` |

Replace these values with the settings for the target network.

## Back Up the Existing Configuration

Create a protected backup before editing Netplan:

```bash
sudo install -d -m 700 /root/netplan-backup
sudo cp -a /etc/netplan/. /root/netplan-backup/
```

Review all existing files because Netplan merges their contents:

```bash
sudo netplan get
sudo ls -la /etc/netplan/
```

## Create the Static IPv4 Configuration

Create `/etc/netplan/99-static-ip.yaml`:

```bash
sudo vi /etc/netplan/99-static-ip.yaml
```

Add the static configuration:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.50.209/24
      routes:
        - to: default
          via: 192.168.50.1
      nameservers:
        addresses:
          - 192.168.50.1
          - 1.1.1.1
```

Set restrictive permissions:

```bash
sudo chown root:root /etc/netplan/99-static-ip.yaml
sudo chmod 600 /etc/netplan/99-static-ip.yaml
```

`routes` is preferred over the deprecated `gateway4` key. The default route is equivalent to `0.0.0.0/0` through the specified gateway.

If IPv6 DHCP must also be disabled, add this beneath `dhcp4`:

```yaml
      dhcp6: false
```

Do not disable IPv6 unless the network design requires it.

## Validate the YAML

Generate the backend configuration without activating it:

```bash
sudo netplan --debug generate
```

If the command reports an error, correct it before continuing. Common causes include tabs, inconsistent indentation, an incorrect interface name, and malformed addresses.

Display the merged configuration:

```bash
sudo netplan get
```

Confirm that the intended interface has `dhcp4: false`, the static address, the default route, and the expected DNS servers.

## Apply Safely with Automatic Rollback

Use `netplan try` instead of applying the change directly:

```bash
sudo netplan try --timeout 120
```

Netplan temporarily applies the configuration and asks for confirmation. If the connection fails or the change is not confirmed before the timeout, Netplan attempts to restore the previous configuration.

From a second terminal or console, verify connectivity while the confirmation prompt remains open:

```bash
ip -br address show eth0
ip route show
ping -c 3 192.168.50.1
getent hosts ubuntu.com
```

Confirm the configuration only after local and remote connectivity work. Then make sure the final configuration is active:

```bash
sudo netplan apply
```

> **Remote systems:** Automatic rollback reduces risk but does not replace console access. A malformed configuration, renderer failure, or unavailable gateway can still leave the server unreachable.

## Verify the Static Address

Check the address, route, DNS service, and external connectivity:

```bash
ip -br address show eth0
ip route show default
resolvectl status eth0
ping -c 3 192.168.50.1
getent ahosts ubuntu.com
```

Expected results include:

- `192.168.50.209/24` assigned to `eth0`
- A default route through `192.168.50.1`
- The configured DNS servers associated with `eth0`
- Successful gateway connectivity and DNS resolution

Reboot only after these checks pass, then verify again:

```bash
sudo reboot
```

After reconnecting:

```bash
ip -br address show eth0
ip route show default
resolvectl status eth0
```

## Handle `50-cloud-init.yaml`

Cloud images commonly contain `/etc/netplan/50-cloud-init.yaml`. Its header may state that cloud-init generated the file and that manual changes may not persist.

Prefer creating `/etc/netplan/99-static-ip.yaml` rather than directly editing the generated file. If the cloud platform must no longer manage networking, disable cloud-init network generation by creating `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`:

```bash
sudo vi /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

```yaml
network:
  config: disabled
```

Keep a valid Netplan file under `/etc/netplan/` before disabling cloud-init networking. Then validate and apply it:

```bash
sudo netplan --debug generate
sudo netplan try --timeout 120
```

> **Cloud platforms:** Provider metadata, DHCP reservations, floating IPs, or generated routes may be required for connectivity. Confirm the platform's network model before replacing its cloud-init configuration.

## Configure NetworkManager or systemd-networkd

Netplan renders configuration for a backend service. Ubuntu Server typically uses `systemd-networkd`, while Ubuntu Desktop commonly uses NetworkManager.

Check the active renderer:

```bash
sudo netplan get | grep -i renderer
systemctl is-active systemd-networkd
systemctl is-active NetworkManager
```

Most installations can omit `renderer` and retain the existing default. If it must be specified explicitly, place it under `network`:

```yaml
network:
  version: 2
  renderer: networkd
```

For NetworkManager:

```yaml
network:
  version: 2
  renderer: NetworkManager
```

Do not change the renderer merely to assign a static address.

## Add Multiple Addresses or Static Routes

Assign multiple addresses to one interface:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.50.209/24
        - 192.168.50.210/24
      routes:
        - to: default
          via: 192.168.50.1
      nameservers:
        addresses:
          - 192.168.50.1
```

Add a route to another private network:

```yaml
      routes:
        - to: default
          via: 192.168.50.1
        - to: 10.20.0.0/16
          via: 192.168.50.254
```

Only one interface should normally provide the preferred default route. Multiple defaults require route metrics or policy routing.

## Roll Back Manually

If console access is available and the new configuration fails, remove the override and restore the backup:

```bash
sudo rm /etc/netplan/99-static-ip.yaml
sudo cp -a /root/netplan-backup/. /etc/netplan/
sudo netplan --debug generate
sudo netplan apply
```

If the backup originally contained no `99-static-ip.yaml`, removing that file prevents it from continuing to override the restored configuration.

To temporarily return one interface to DHCP, use:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
```

Apply it first with `netplan try`.

## Troubleshooting

| Symptom | Likely cause | Check or fix |
|---------|--------------|--------------|
| YAML parse error | Incorrect indentation or tabs | Run `netplan --debug generate`; replace tabs with spaces |
| Interface has no address | Interface name does not match | Check `ip -br link` and update the YAML key |
| Two IPv4 addresses appear | DHCP remains enabled in another merged definition | Inspect `netplan get` and all `/etc/netplan/*.yaml` files |
| No default route | Missing or invalid `routes` entry | Check `ip route`; verify gateway and subnet |
| `gateway4` warning | Deprecated configuration key | Use a default route under `routes` |
| DNS fails but IP connectivity works | Wrong DNS servers or resolver association | Check `resolvectl status` and `getent hosts ubuntu.com` |
| Configuration changes after reboot | cloud-init regenerated networking | Review `50-cloud-init.yaml` and cloud-init settings |
| Netplan warns about permissions | YAML file is accessible by other users | Run `chmod 600 /etc/netplan/*.yaml` |
| SSH disconnects after apply | New address, route, or firewall path is unreachable | Wait for `netplan try` rollback or use console access |
| Duplicate default routes | More than one interface supplies a default | Remove the unwanted route or configure route metrics |

Inspect logs for the active backend:

```bash
journalctl -u systemd-networkd --since '-10 minutes'
journalctl -u NetworkManager --since '-10 minutes'
networkctl status eth0
```

Use the commands relevant to the active renderer.

## Quick Procedure

```bash
# 1. Identify the interface and existing route
ip -br link
ip route show

# 2. Back up Netplan
sudo install -d -m 700 /root/netplan-backup
sudo cp -a /etc/netplan/. /root/netplan-backup/

# 3. Create /etc/netplan/99-static-ip.yaml with the static settings
sudo vi /etc/netplan/99-static-ip.yaml
sudo chmod 600 /etc/netplan/99-static-ip.yaml

# 4. Validate without applying
sudo netplan --debug generate
sudo netplan get

# 5. Apply temporarily and confirm connectivity
sudo netplan try --timeout 120

# 6. Verify
ip -br address show eth0
ip route show default
resolvectl status eth0
```

## See Also

- [ip Command Cheatsheet](articles/ip-command-cheatsheet.md) — inspect addresses, links, and routes
- [resolvectl Cheatsheet](articles/resolvectl-cheatsheet.md) — inspect and troubleshoot DNS resolver state
- [cloud-init Cheatsheet](articles/cloud-init-cheatsheet.md) — cloud image and network configuration
- [nmcli Cheatsheet](articles/nmcli-cheatsheet.md) — persistent NetworkManager configuration on RHEL and other systems
