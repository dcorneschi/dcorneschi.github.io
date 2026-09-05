# ProxMenux Monitor: Guide, One-Liners, Tips & Tricks

[ProxMenux](https://github.com/MacRimi/ProxMenux) is a menu-driven toolkit for Proxmox VE
(post-install tweaks, backup/restore, and more). **ProxMenux Monitor** is its integrated
**web dashboard** — real-time visibility into a Proxmox host from any browser on your
network, no terminal required. It runs as a lightweight Flask app on **port 8008** and is
installed automatically with ProxMenux.

This guide covers installing and running the Monitor, the systemd service commands you'll
actually use, securing and reverse-proxying it, and a set of one-liners and tips for
day-to-day operation.

> Not to be confused with Proxmox's built-in web UI (port 8006). ProxMenux Monitor is a
> separate, read-focused dashboard on **8008** aimed at at-a-glance homelab health.

---

## What it shows

- **Real-time CPU, RAM, disk usage, and network traffic.**
- **VMs and LXC containers** with running/stopped status indicators.
- **Storage and SMART disk health** (including NVMe temperatures).
- **System logs**, filterable from the dashboard.
- **Backup status** (including PBS) in recent builds.
- **Health monitor** with a one-click Proxmox update trigger (recent builds).
- Works on desktop and mobile (installable as a PWA in newer versions).

## Install

Run ProxMenux's installer on the Proxmox host; the Monitor comes with it:

```bash
bash -c "$(wget -qLO - https://raw.githubusercontent.com/MacRimi/ProxMenux/main/install_proxmenux.sh)"
```

Then launch the menu any time with:

```bash
menu
```

Open the dashboard in a browser:

```
http://<your-proxmox-ip>:8008
```

Dependencies are installed automatically — notably `python3` + `python3-pip` (the Monitor
is a Flask web app), plus `dialog`, `curl`, `jq`, and `git` for the menu toolkit.

### Beta channel

To get the newest Monitor builds early (may be buggy), install from the `develop` branch:

```bash
bash -c "$(wget -qLO - https://raw.githubusercontent.com/MacRimi/ProxMenux/develop/install_proxmenux_beta.sh)"
```

## Managing the service

The Monitor runs as `proxmenux-monitor.service` and starts on boot. The essential commands:

```bash
# Is it running?
systemctl status proxmenux-monitor

# Recent logs (first place to look if the dashboard won't load)
journalctl -u proxmenux-monitor -n 50

# Follow logs live
journalctl -u proxmenux-monitor -f

# Restart after a config change or upgrade
systemctl restart proxmenux-monitor

# Enable / disable auto-start on boot
systemctl enable proxmenux-monitor
systemctl disable proxmenux-monitor
```

## One-liners

```bash
# Confirm the Monitor is listening on 8008
ss -tlnp | grep ':8008'

# Quick health probe of the dashboard (expect HTTP 200/302)
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8008

# Find the service unit file and see how it launches
systemctl cat proxmenux-monitor

# Watch CPU/RAM the Monitor process itself uses
ps -o pid,%cpu,%mem,cmd -C python3 | grep -i monitor

# Errors only, since the last boot
journalctl -u proxmenux-monitor -b -p err --no-pager

# Tail logs while you reproduce a dashboard issue
journalctl -u proxmenux-monitor -f &  # stop later with: kill %1

# Restart the Monitor and confirm it came back
systemctl restart proxmenux-monitor && sleep 2 && systemctl is-active proxmenux-monitor

# See which Proxmox host IP to browse to
hostname -I | awk '{print "http://"$1":8008"}'
```

## Securing access

The dashboard exposes host telemetry, so lock it down:

- **Enable login authentication** — ProxMenux Monitor supports a login to protect access.
  Turn it on rather than leaving the dashboard open on your LAN.
- **Enable 2FA (TOTP)** — the Monitor supports Two-Factor Authentication with an
  authenticator app. Recommended if the dashboard is reachable beyond a trusted network.
- **Don't expose port 8008 to the internet directly.** If you need remote access, put it
  behind a reverse proxy with TLS (see below) or reach it over a VPN/WireGuard/Tailscale.
- **Firewall it to your management network** if you only browse from a known subnet:

```bash
# Example: allow 8008 only from your LAN subnet (adjust to your network)
iptables -A INPUT -p tcp --dport 8008 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 8008 -j DROP
```

## Reverse proxy (TLS)

The Monitor supports running behind **Nginx** or **Traefik**, which lets you put it on a
hostname with a real certificate instead of `http://ip:8008`.

**Nginx example** — proxy `monitor.example.com` to the local Monitor:

```nginx
server {
    listen 443 ssl;
    server_name monitor.example.com;

    ssl_certificate     /etc/letsencrypt/live/monitor.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/monitor.example.com/privkey.pem;

    location / {
        proxy_pass         http://127.0.0.1:8008;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        # WebSocket upgrade for live metrics
        proxy_http_version 1.1;
        proxy_set_header   Upgrade    $http_upgrade;
        proxy_set_header   Connection "upgrade";
    }
}
```

**Traefik** (labels/dynamic config) — point a router at a service targeting
`http://<proxmox-ip>:8008`, attach your TLS resolver, and keep WebSocket passthrough
enabled so the live graphs update.

> Whichever proxy you use, forward the `Upgrade`/`Connection` headers — the dashboard uses
> WebSockets for real-time updates, and they'll stall without them.

## Tips & tricks

- **Do the SMART/NVMe temperature view early.** It surfaces failing or overheating disks
  before they take a VM down — one of the most useful panels for a homelab.
- **Filter the log view** instead of SSHing in for `journalctl`; it's often faster for a
  quick "what just happened" during an incident.
- **Install it as a PWA** on your phone (newer builds show an install prompt) for a
  quick home-screen health check.
- **Use the built-in Proxmox update trigger** (Health Monitor, recent builds) for a
  one-click `apt` update from the dashboard — but still reboot deliberately when a new
  kernel is installed (ProxMenux detects an unbooted kernel and prompts you).
- **After a ProxMenux upgrade**, if the UI looks stale, `systemctl restart
  proxmenux-monitor` and hard-refresh the browser (the frontend is cached).
- **If the dashboard won't load**, work the chain: `systemctl status proxmenux-monitor` →
  `ss -tlnp | grep 8008` (is it listening?) → `journalctl -u proxmenux-monitor -n 50`
  (why did it fail?). A Python traceback in the logs usually points at a missing dependency
  after an OS upgrade — reinstall/upgrade via `menu`.
- **Pair it with the ProxMenux menu**, not instead of it — the CLI does the changes
  (post-install tuning, backups), the Monitor gives you the read-only picture.
- **Keep it on a trusted network.** Even with login + 2FA, telemetry dashboards are best
  kept off the public internet unless fronted by a hardened reverse proxy.

## Troubleshooting quick table

| Symptom | Check | Likely fix |
|---------|-------|-----------|
| Dashboard won't open | `systemctl status proxmenux-monitor` | Start/restart the service |
| Service running but no page | `ss -tlnp \| grep 8008` | Confirm it's listening; check firewall |
| Live graphs frozen behind proxy | Proxy config | Forward `Upgrade`/`Connection` (WebSockets) |
| Python traceback in logs | `journalctl -u proxmenux-monitor -n 50` | Reinstall/upgrade via `menu` (missing dep) |
| Can't reach from another host | Firewall / subnet | Allow 8008 from your management subnet |
| Stale UI after upgrade | Browser cache | Restart service + hard-refresh |

---

### Sources

- [ProxMenux — GitHub (MacRimi)](https://github.com/MacRimi/ProxMenux)
- [ProxMenux documentation](https://proxmenux.com/en/docs/introduction)
- [ProxMenux changelog](https://proxmenux.com/en/changelog)
