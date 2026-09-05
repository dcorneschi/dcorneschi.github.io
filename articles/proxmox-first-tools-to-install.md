# Setting Up a Brand-New Proxmox Server: First-Boot Configuration and Essential Tools

![Medieval-illuminated-manuscript illustration of a Proxmox castle: a stone keep flying an orange-and-white Proxmox VE banner, a blacksmith's workbench of labeled tools in the foreground, a knight guarding the gate](./images/proxmox-medieval-banner.png)

<!--
  IMAGE PLACEHOLDER — drop the generated file at ./images/proxmox-medieval-banner.png
  (create the images/ folder next to this .md). Adjust the path/filename above to match.

  Suggested image-generation prompt (Midjourney / DALL·E / Stable Diffusion):

  "A medieval illuminated-manuscript style illustration on aged parchment with gold-leaf
   accents and an ornate hand-drawn border. Center: a fortified stone castle keep flying a
   large banner bearing the Proxmox VE logo (orange chevron/sun on white). Foreground: a
   blacksmith's wooden workbench covered in neatly arranged tools, each tagged with a small
   parchment label reading 'lm-sensors', 'nvme-cli', 'smartmontools', 'iperf3', 'NUT',
   'sysstat'. A knight in armor stands guard at the castle gate, which bears a wooden sign
   painted ':8006'. Warm candle-lit tones, rich reds/blues/golds, intricate detail, wide
   banner/landscape aspect ratio (roughly 3:1). Style: medieval manuscript meets tech
   allegory, no modern objects, no photorealism."

  Aspect ratio tip: 3:1 (e.g. --ar 3:1 in Midjourney) reads well as a page header.
-->

A fresh Proxmox VE install is complete enough to run VMs and containers out of the box — you
won't be blocked by sticking with the defaults. But there's a handful of configuration steps
you should do *first* to make the node correct, reachable, secure, and recoverable, plus a
small set of packages that make it far easier to monitor and troubleshoot. This article
covers the first-boot configuration up front, then the tools I'd install.

Everything here is Debian-based (`apt`), run as root on the PVE host.

---

## Part 1 — First-boot configuration (do this first)

These steps make the node correct, reachable, secure, and recoverable. Several of them are
much harder to change later, so do them early — before you create a cluster or start running
real workloads.

### A. Fix the APT repositories

A fresh install points at the **enterprise** repo, which fails without a subscription. On a
home-lab node, disable it and enable the **no-subscription** repo so updates work.

```bash
# Disable the enterprise repos (comment them out)
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list 2>/dev/null
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/ceph.list 2>/dev/null   # if present

# Add the PVE no-subscription repo (adjust codename: bookworm, trixie, ...)
echo "deb http://download.proxmox.com/debian/pve $(. /etc/os-release; echo $VERSION_CODENAME) pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

apt update && apt full-upgrade -y
```

> Newer Proxmox versions use the **deb822** `.sources` format under
> `/etc/apt/sources.list.d/`; if you see `*.sources` files, edit those instead. The Helper-
> Scripts post-install (Part 2, item 8) automates all of this if you prefer.

### B. Remove the "No valid subscription" popup

The manual way (no external script). Re-apply after major PVE upgrades, as updates can
overwrite the file:

```bash
sed -Ezi.bak "s/(function\(orig_cmd\) \{)/\1\n\torig_cmd\(\); return;/g" \
  /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy.service
```

### C. Get hostname and networking right (before clustering)

Proxmox is sensitive to hostname resolution — a wrong `/etc/hosts` entry breaks the web UI
and, especially, cluster join. On a fresh node, confirm:

- A **static IP** on the management interface (`/etc/network/interfaces`).
- The hostname resolves to that IP in **`/etc/hosts`** (not `127.0.1.1`).

```bash
hostnamectl                          # confirm the hostname
hostname --ip-address                # should return your static management IP, not 127.x
grep "$(hostname)" /etc/hosts        # the FQDN + short name should map to the real IP
```

Fix this **before** creating or joining a cluster — changing a node's hostname/IP after the
fact is painful.

### D. Configure backups (do this before you need them)

Backups are the highest-value step here, and pair directly with the UPS tool (Part 2, item
6): a UPS helps prevent corruption, backups let you recover from it.

- **Quick start:** schedule `vzdump` backups of VMs/CTs to a storage target
  (Datacenter → Backup in the UI, or a `vzdump` cron/job). Choose a mode:
  `snapshot` (live, minimal downtime), `suspend`, or `stop`.
- **Better:** stand up a **Proxmox Backup Server (PBS)** for deduplicated, incremental,
  verifiable backups, and add it as a storage target. Test a **restore** — an untested
  backup isn't a backup.

```bash
# Ad-hoc backup of a single guest to local storage, as a sanity check
vzdump 100 --storage local --mode snapshot --compress zstd
```

### E. Harden SSH

If the host is reachable beyond your trusted LAN, move to **key-only** auth. This is a
two-part job: get your key onto the host, then disable password login.

**1. Generate a key (on your workstation, if you don't already have one).** Prefer Ed25519:

```bash
ssh-keygen -t ed25519 -C "you@workstation"
# creates ~/.ssh/id_ed25519 (private — never share) and id_ed25519.pub (public)
```

**2. Copy the public key to the Proxmox host:**

```bash
# Easiest — appends your pubkey to the server's authorized_keys:
ssh-copy-id root@pve1.lab.example.com

# Manual equivalent if ssh-copy-id isn't available:
cat ~/.ssh/id_ed25519.pub | ssh root@pve1 'mkdir -p ~/.ssh && chmod 700 ~/.ssh \
  && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

> Proxmox cluster note: on the PVE host, root's `~/.ssh/authorized_keys` is a **symlink** to
> `/etc/pve/priv/authorized_keys` (managed cluster-wide). Appending as above is fine; just
> don't replace the symlink with a regular file.

**3. Verify key login works, then disable passwords:**

```bash
# Confirm you can log in WITHOUT a password first, in a separate session:
ssh root@pve1 'echo key-login-ok'

# Then harden sshd:
sed -i 's/^#\?PasswordAuthentication .*/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
systemctl restart ssh
```

> Don't lock yourself out: keep the **second session open** while you test a fresh login.
> `PermitRootLogin prohibit-password` still allows root via key (needed for cluster/replication
> traffic) but blocks root password login.

### F. Proxmox web UI authentication (realms, non-root users, 2FA)

SSH keys protect the shell; the **web UI has its own authentication**, and it's worth
locking down separately. Proxmox authenticates against **realms**:

- **Linux PAM** — real system users (this is how `root@pam` logs in). Backed by
  `/etc/passwd`; use it for the initial admin.
- **Proxmox VE authentication server (`@pve`)** — users that exist **only in Proxmox**, not
  on the OS. Best for day-to-day admin and API accounts, because they can't SSH into the
  host.
- **LDAP / Active Directory / OpenID Connect** — for centralized identity / SSO.

**Recommended baseline:**

1. **Stop using `root@pam` for daily work.** Create a dedicated admin user in the PVE realm
   and grant it admin via a role, then use that:

   ```bash
   pveum user add admin@pve --comment "Daily admin"
   pveum passwd admin@pve
   pveum acl modify / --users admin@pve --roles Administrator
   ```

   (Or Datacenter → Permissions → Users / → add an ACL with role `Administrator`.) Reserve
   `root@pam` for break-glass.

2. **Enable Two-Factor Authentication.** Datacenter → Permissions → **Two Factor**, or
   per-user under Users → *TFA*. Proxmox supports **TOTP** (authenticator apps),
   **WebAuthn** (hardware keys / passkeys — set the WebAuthn origin to your UI hostname
   first), and **recovery keys**. Enable it on **`root@pam`** at minimum, and on any admin
   account.

3. **Use least-privilege roles for others.** Don't hand everyone `Administrator`. Built-in
   roles like `PVEVMUser`, `PVEAuditor`, or a custom role scoped to a pool/VM keep blast
   radius small. Grant via **ACLs** at the right path (`/`, `/vms/<id>`, a resource pool).

4. **API tokens for automation.** For scripts/Terraform/monitoring, create a scoped **API
   token** (Users → API Tokens) instead of using a password — it's revocable and can be
   privilege-separated.

> The web login and SSH are independent: disabling SSH password auth does **not** add 2FA to
> the web UI, and vice versa. Do both.

### G. Enable the firewall (baseline)

Proxmox ships a built-in firewall at datacenter, node, and guest levels. Turn it on with a
sane baseline — but **allow your management access first** (web UI `8006`, SSH `22`) so you
don't lock yourself out.

- Datacenter → Firewall → **Options**: enable, default inbound policy `DROP`.
- Add rules permitting `8006/tcp` and `22/tcp` from your admin network **before** enabling.
- Node → Firewall for per-node rules.

### H. Set up email notifications

Alerts are worthless if they don't reach you. Configure a notification target so **backup
failures**, replication errors, certificate-renewal problems, and health alerts get
delivered. Proxmox VE (8.1+) has a **notifications** framework at Datacenter →
**Notifications** with targets (endpoints) and **matchers** (which events go where).

**Option 1 — local mail via the built-in `sendmail` target (simplest).** Out of the box
Proxmox mails `root@`, and root's address is set at Datacenter → Options → *Email from
address* / per-user under Datacenter → Users. This works only if the host can actually send
mail, which usually means relaying through a real SMTP server.

**Option 2 — SMTP target (recommended; no local MTA needed).** Since PVE 8, you can add an
**SMTP endpoint** directly in the UI — no Postfix wrangling:

- Datacenter → Notifications → **Add → SMTP**.
- Fill in server, port (587 STARTTLS / 465 TLS), username, password/app-password, the
  **From** address, and one or more **To** recipients.
- Add a **Matcher** so the events you care about (or all) route to that endpoint.
- Use the target's **Test** button to send a probe email and confirm delivery.

> Gmail/M365 tip: use an **app password**, not your account password, and the provider's
> submission host (`smtp.gmail.com:587`, `smtp.office365.com:587`).

**Option 3 — relay via Postfix (older/advanced).** If you prefer a system MTA (e.g. to
relay all host mail through a smarthost), configure `/etc/postfix/main.cf` with a
`relayhost` and SASL auth, then `systemctl reload postfix`. The SMTP target above avoids
this for most home labs.

**Wire up what matters:**

- **Backup jobs:** Datacenter → Backup → edit the job → set a **Notification** target /
  "on failure" so a failed `vzdump`/PBS run reaches you.
- Confirm alerts for **replication**, **certificate renewal**, and node health route to a
  matcher too. You can also use **Gotify** or **ntfy** endpoints instead of (or alongside)
  email.

### I. Get a trusted TLS certificate with Let's Encrypt (ACME)

By default the PVE web UI uses a **self-signed** certificate, so browsers warn on every
visit. Proxmox has **built-in ACME** support to obtain and auto-renew free **Let's Encrypt**
certificates — configured entirely in the UI, no `certbot` needed.

**Step 1 — register an ACME account.** Datacenter → **ACME** → *Accounts* → **Add**: give it
a name, your email (used for expiry warnings), and accept the Let's Encrypt ToS. Use the
**staging** directory first to test without hitting rate limits, then switch to production.

**Step 2 — choose a challenge type:**

- **HTTP-01** — simplest, but requires **port 80 reachable from the internet** to that node.
  Fine for an exposed host; awkward for a purely internal lab.
- **DNS-01** — the better home-lab choice. Proxmox creates a `_acme-challenge` TXT record via
  your DNS provider's API, so **nothing needs to be internet-exposed** and you can even get
  certs for internal-only hostnames. Add your provider + API token under Datacenter → ACME →
  *Challenge Plugins*. (Proxmox supports many providers through its acme.sh integration —
  Cloudflare, Route 53, etc.)

**Step 3 — assign the domain and order the cert.** Node → **Certificates** → *ACME* → add
your FQDN (e.g. `pve1.lab.example.com`), pick the account and challenge, then **Order Now**.
Proxmox fetches the cert, installs it, and **auto-renews** it (~30 days before expiry) via a
built-in timer — reloading `pveproxy` automatically.

```bash
# CLI equivalents (UI is easier), for reference:
pvenode acme account register default you@example.com
pvenode config set --acme domains=pve1.lab.example.com
pvenode acme cert order          # order/renew now
```

> DNS caveat: the node's hostname must resolve to the FQDN you're requesting, and that name
> should be the one you (or your reverse proxy) actually use to reach the UI. If you
> terminate TLS at Traefik (section J), you can instead let the proxy handle Let's Encrypt
> and leave PVE on its self-signed cert internally — pick one place to own certs, not both.

### J. Reverse-proxy the web UI and dashboards

Rather than exposing raw ports (`8006` for PVE, `8008` for ProxMenux Monitor), front them
with your reverse proxy. In a Traefik home lab, route a hostname to the service and let
Traefik terminate TLS and enforce auth.

- Point Traefik at the PVE UI (`https://<node>:8006`, which uses a self-signed cert — set
  the backend to `insecureSkipVerify` or install a proper cert).
- Do the same for ProxMenux Monitor (`:8008`), and enable its **login + TOTP 2FA**.
- Keep the raw ports firewalled to your admin network (see G) so the proxy is the only
  public path.

---

## Part 2 — Tools to install

With the node configured, these packages give you the visibility and troubleshooting
firepower a home-lab host needs.

## 1. lm-sensors — hardware temperature visibility

Home labs rarely have data-center cooling, and mini PCs pack heat into tiny enclosures.
Temperature is a first-class metric: unexplained throttling is often heat, not an
underpowered CPU.

```bash
apt update
apt install lm-sensors -y
sensors-detect        # detect available sensors (usually also run post-install)
sensors               # read current temps
```

If you use **Pulse** for Proxmox monitoring, having `lm-sensors` installed lets Pulse pull
temps into the dashboard and **alert on thresholds** you define. Treat temperature
visibility as mandatory on compact nodes.

---

## 2. nvme-cli — NVMe drive info and SMART health

For any NVMe drives in the host, `nvme-cli` exposes device info and SMART data.

```bash
apt install nvme-cli -y
nvme list                       # list NVMe devices
nvme smart-log /dev/nvme0       # SMART: temp, % used, data written, power-on hours,
                                # unsafe shutdowns, media errors, warnings
```

Great for two things: **verifying used drives** bought off eBay (confirm the wear the seller
claimed), and **baselining** — a node running 15–20 °C hotter than its cluster peers likely
has an airflow or heatsink problem.

---

## 3. smartmontools — SMART for SATA SSDs and spinning disks

`nvme-cli` covers NVMe; `smartmontools` covers everything else — SATA SSDs, spindles, boot
drives, passthrough disks.

```bash
apt install smartmontools -y
smartctl -a /dev/sda            # full SMART: health, temp, hours, errors, attributes
smartctl -t short /dev/sda      # run a short self-test
smartctl -t long  /dev/sda      # run a long self-test
```

When **IO delay creeps up** or performance gets weird, a flaky drive is a prime suspect —
start here.

---

## 4. iperf3 — network throughput testing

Measures bandwidth between two hosts. Essential for validating you're actually getting the
2.5/10 GbE you think you are — which underpins live migration, cluster comms, and
iSCSI/NFS/Ceph storage.

```bash
apt install iperf3 -y

# On host A (the "server"):
iperf3 -s

# On host B (the "client"), point at A's IP:
iperf3 -c 10.10.10.20
```

If two nodes "feel" slow, an `iperf3` run quickly confirms whether the network is (or isn't)
the problem — especially valuable with **Ceph/HCI** storage.

---

## 5. ethtool — inspect NIC speed, duplex, and offloads

When you suspect a network-side problem, `ethtool` shows what the interface actually
negotiated.

```bash
apt install ethtool -y
ethtool nic3            # speed, duplex, auto-negotiation
ethtool -i nic3         # driver + firmware info
ethtool -k nic3         # offload settings (checksum, TCP, GSO, etc.)
```

Classic catch: a **10 GbE link that quietly negotiated down to 1 Gbps**. Don't randomly
disable offloads because a forum said so — but they're useful data when troubleshooting a
specific issue.

---

## 6. NUT — UPS monitoring and automated shutdown

If the host is on a UPS, you want monitored, **graceful** shutdown on power loss. Network UPS
Tools (NUT) provides it.

```bash
apt install nut -y
```

Why it matters more on Proxmox: the **blast radius** of a dirty shutdown is large — multiple
VMs, containers, DBs, and storage go down at once. Consumer NVMe often lacks power-loss
protection, so a sudden outage risks **data corruption**. Backups recover data *after* a
problem; a UPS + clean shutdown helps *prevent* the power-related problem in the first place.

---

## 7. ProxMenux (+ Monitor) — menu-driven management and a web dashboard

Unlike the diagnostic CLIs above, **ProxMenux** is an open-source, menu-driven toolkit for
Proxmox VE: an interactive CLI plus **ProxMenux Monitor**, an integrated **web dashboard**
for real-time health monitoring. It surfaces post-install config, disk/storage/share
managers, the PVE helper scripts, network management, security, and backup/restore.

### Installation

Run the installer in the Proxmox host shell. This installs the CLI **and** the Monitor
dashboard in one step:

```bash
bash -c "$(wget -qLO - https://raw.githubusercontent.com/MacRimi/ProxMenux/main/install_proxmenux.sh)"
```

> Safety note: this runs as root and pulls a script from the internet. **Review the source
> first** (`install_proxmenux.sh` in the MacRimi/ProxMenux repo) and always confirm the
> current command from the official site rather than trusting an old copy.

The installer pulls in its dependencies automatically (`dialog`, `curl`, `jq`, `git`,
`python3` + `python3-pip` — the last two power the Flask-based Monitor).

### Using the CLI

Launch the interactive menu any time with:

```bash
menu
```

### ProxMenux Monitor (web dashboard)

Monitor is installed **automatically** as part of the standard install and runs as a
**systemd service** (`proxmenux-monitor.service`) that starts on boot. Reach it from any
browser on your network:

```
http://<your-proxmox-ip>:8008
```

It shows real-time CPU/RAM/disk/network, running VMs and LXC containers, filterable logs,
PBS backup status, and NVMe/SMART disk temperatures. It supports login auth, TOTP 2FA, and
reverse-proxy setups (Nginx / Traefik).

Managing the service:

```bash
systemctl status proxmenux-monitor           # check it's running
journalctl -u proxmenux-monitor -n 50        # view recent logs
systemctl restart proxmenux-monitor          # restart the dashboard
```

Reasonable caution: many wouldn't put a bundled TUI/dashboard on a **production** host, but
it's convenient and actively maintained for home labs. If you expose the dashboard, put it
behind your reverse proxy and enable auth/2FA rather than leaving `:8008` open (see Part 1,
section J).

> Beta channel: to try pre-release Monitor builds, the project also ships a beta installer
> (`install_proxmenux_beta.sh` on the `develop` branch). Expect rough edges; use it only on
> non-critical nodes.

---

## 8. Proxmox VE Helper-Scripts — fast, repeatable setup

The community **Proxmox VE Helper-Scripts** (the well-known `tteck`-originated project) are
the fastest way to do common post-install chores and spin up LXC apps. The single most
useful one on a fresh node is the **post-install script**, which offers to configure the
correct APT repositories (disable the enterprise repo if you have no subscription, enable
the no-subscription repo), remove the subscription nag, and apply sane defaults — i.e. it
automates Part 1 sections A and B.

```bash
# Review the script first, then run the post-install helper from the PVE shell.
# Start at the project site and copy the current command:
#   https://community-scripts.github.io/ProxmoxVE/
```

> Safety note: these run as root and pull from the internet. **Read what a script does
> before running it**, and prefer the maintained community fork's current URL over an old
> copied command.

---

## 9. tmux + htop/btop — resilient sessions and live process view

Two small quality-of-life installs that pay off immediately during any real work or
troubleshooting session.

```bash
apt install tmux htop btop -y
```

- **tmux** keeps long-running work (migrations, `dd`, restores, `iperf3`) alive if your SSH
  session drops — reattach with `tmux attach`.
- **htop** / **btop** give a live, sortable view of CPU, memory, and IO pressure that's far
  more readable than `top` when a node is under load.

---

## 10. sysstat — historical CPU, memory, disk, and network stats

Where `htop`/`btop` show you *now*, **sysstat** records performance over time so you can
answer "what was this node doing at 3 a.m. when the alert fired?" It provides `sar` (the
historical reporter) plus point-in-time tools like `iostat`, `mpstat`, and `pidstat`.

```bash
apt install sysstat -y

# Enable collection (sysstat ships with the collector off by default on Debian)
sed -i 's/^ENABLED="false"/ENABLED="true"/' /etc/default/sysstat
systemctl enable --now sysstat
```

### Collect every 1 minute instead of the default 10

On Debian 12 / Proxmox VE 8, collection is driven by the **`sysstat-collect.timer`**
systemd unit, which fires every 10 minutes. For a homelab node you often want finer
granularity so a short-lived spike isn't averaged away. Override the timer to run every
minute — use a drop-in so a package update doesn't clobber your change:

```bash
# Create a drop-in override for the collector timer
systemctl edit sysstat-collect.timer
```

Add the following. The empty `OnCalendar=` line first **clears** the packaged 10-minute
schedule (otherwise both schedules apply), then sets a 1-minute cadence:

```ini
[Timer]
OnCalendar=
OnCalendar=*:00/1
AccuracySec=1sec
```

```bash
# Reload and confirm the new schedule
systemctl daemon-reload
systemctl restart sysstat-collect.timer
systemctl list-timers sysstat-collect.timer   # NEXT should be ~1 min away
```

> **Tradeoff — disk usage.** Sampling 10× more often makes today's binary log in
> `/var/log/sysstat/` (the `saDD` files) roughly 10× larger. It's still small (tens of MB
> per day on a typical node), but if you want to keep more days of this finer history,
> check and raise `HISTORY` in `/etc/sysstat/sysstat` (the retention in days; the packaged
> default varies by distro/version — Debian has shipped `7`, other distros `28`).
>
> **Older / cron-driven systems.** If your box drives collection from cron instead of the
> systemd timer, edit the `sysstat` entry in `/etc/cron.d/sysstat` and change the
> `5-55/10` minute field of the `sa1` line to `*` (every minute), e.g.
> `* * * * * root command -v debian-sa1 > /dev/null && debian-sa1 1 1`.

Once the collector has been running, you can read back history and probe live:

```bash
sar -u 1 5              # CPU utilization, 5 samples 1s apart (live)
sar -r                  # memory usage from today's collected history
sar -d -p               # per-disk I/O history (pretty device names)
sar -n DEV              # per-interface network throughput history
iostat -xz 1            # live extended per-device I/O (await, %util)
pidstat -d 1            # per-process disk I/O — find the noisy writer
```

This pairs well with `nvme-cli`/`smartmontools` (items 2–3) and `iperf3` (item 4): when IO
delay or "it was slow last night" comes up, `sar` gives you the timeline those point-in-time
tools can't.

---

## 11. unattended-upgrades or a patching habit — stay current safely

A new host should have a deliberate patching plan. For security updates specifically,
`unattended-upgrades` can apply them automatically.

```bash
apt install unattended-upgrades -y
dpkg-reconfigure -plow unattended-upgrades     # opt in to automatic security updates
```

Caveats for a hypervisor:

- Make sure APT is pointed at the **correct Proxmox repos first** (Part 1, section A) so
  you're not pulling from the enterprise repo without a subscription.
- **Kernel updates need a reboot** to take effect — schedule those, ideally after your VMs
  are safely migratable or backed up. Automatic *security* patching is fine; automatic
  reboots of a hypervisor are not something to enable blindly.

---

## Quick reference — first-boot configuration

| Step | What | Why it matters |
|------|------|----------------|
| A | Fix APT repos (disable enterprise, enable no-subscription) | Updates work without a subscription |
| B | Remove subscription popup | Cosmetic; re-apply after upgrades |
| C | Static IP + hostname/`/etc/hosts` sanity | Web UI and clustering break if wrong |
| D | Backups (`vzdump` / PBS) + test a restore | Recover from corruption/mistakes |
| E | SSH key-only auth (Ed25519 + `ssh-copy-id`) | Reduce shell exposure on reachable hosts |
| F | Web UI auth: non-root `@pve` admin + **2FA** + least-priv roles | Protect the UI/API separately from SSH |
| G | Enable built-in firewall (allow 8006/22 first) | Baseline network protection |
| H | Email/SMTP notifications (or Gotify/ntfy) + matchers | Backup/health failures actually reach you |
| I | Let's Encrypt cert via built-in ACME (HTTP-01 or DNS-01) | Trusted TLS on the UI, auto-renewed |
| J | Reverse-proxy the UI/dashboards (Traefik + TLS/auth) | Don't expose raw `8006`/`8008` |

## Quick reference — tools

| # | Tool | Install | Primary use |
|---|------|---------|-------------|
| 1 | lm-sensors | `apt install lm-sensors -y` | Host temperatures / throttling |
| 2 | nvme-cli | `apt install nvme-cli -y` | NVMe info + SMART |
| 3 | smartmontools | `apt install smartmontools -y` | SATA/HDD SMART + self-tests |
| 4 | iperf3 | `apt install iperf3 -y` | Network throughput testing |
| 5 | ethtool | `apt install ethtool -y` | NIC speed/duplex/offloads |
| 6 | NUT | `apt install nut -y` | UPS monitoring + safe shutdown |
| 7 | ProxMenux + Monitor | `bash -c "$(wget -qLO - .../install_proxmenux.sh)"` → `menu`; dashboard on `:8008` | Menu-driven mgmt + web monitoring |
| 8 | PVE Helper-Scripts | community-scripts site | Post-install + LXC app setup |
| 9 | tmux + htop/btop | `apt install tmux htop btop -y` | Durable sessions + live metrics |
| 10 | sysstat | `apt install sysstat -y` (enable collector) | Historical CPU/mem/disk/net (`sar`, `iostat`) |
| 11 | unattended-upgrades | `apt install unattended-upgrades -y` | Automatic security patching |

---

## Summary

- **Configure first:** fix the **APT repos**, get **hostname/networking** right before
  clustering, set up **backups** (and test a restore), harden **SSH** (Ed25519 key + disable
  passwords), lock down **web UI auth** (non-root `@pve` admin, **2FA**, least-privilege
  roles, API tokens), enable the **firewall** (allow `8006`/`22` first), wire up
  **email/SMTP notifications**, get a trusted **Let's Encrypt** cert via built-in **ACME**
  (DNS-01 is easiest for internal hosts), and **reverse-proxy** the web UI/dashboards instead
  of exposing raw ports. Several of these (storage layout, hostname/IP) are hard to change
  later — decide them early. **SSH and web-UI auth are independent — secure both.** And pick
  **one** place to own TLS — PVE's ACME **or** your reverse proxy, not both.
- **Then add tools — visibility first:** `lm-sensors` (temps), `nvme-cli` + `smartmontools`
  (drive health). On a compact home-lab node, temperature and disk SMART catch problems
  early.
- **Network toolkit:** `iperf3` (throughput) and `ethtool` (link speed/duplex/offloads) —
  the pair that quickly proves whether "slow" is the network or not.
- **Protect the host:** `NUT` for UPS-driven graceful shutdown, because a hypervisor's dirty
  shutdown blast radius is large and consumer NVMe corrupts easily.
- **Convenience + repeatability:** `ProxMenux` and the community **Helper-Scripts** for
  local management and fast, correct post-install setup.
- **Essentials:** `tmux` + `htop/btop` for resilient troubleshooting sessions, `sysstat`
  (`sar`/`iostat`) for historical performance data, and `unattended-upgrades` (with a
  deliberate reboot plan) to stay patched.

---

### References

- [Proxmox VE package repositories (official docs)](https://pve.proxmox.com/wiki/Package_Repositories)
- [Proxmox VE Community Helper-Scripts](https://community-scripts.github.io/ProxmoxVE/)
- [Network UPS Tools (NUT) documentation](https://networkupstools.org/)
- [ProxMenux (MacRimi/ProxMenux) — install, Monitor, and docs](https://github.com/MacRimi/ProxMenux)
- [Proxmox VE certificate management / ACME (Let's Encrypt)](https://pve.proxmox.com/wiki/Certificate_Management)
- [Proxmox VE notifications (SMTP and targets)](https://pve.proxmox.com/pve-docs/chapter-notifications.html)
