# Proxmox ACME SSL with Hetzner DNS

Proxmox VE ships with a self-signed certificate, so browsers show a warning on the web UI. You can replace it with a trusted Let's Encrypt certificate using ACME. When your domain's DNS is hosted at Hetzner, the **DNS-01 challenge** with the Hetzner DNS plugin is the best option — it works without exposing port 80, and it's the only challenge type that can issue **wildcard** certificates.

This guide covers both the web UI flow and the equivalent `pvenode` CLI commands.

> DNS-01 proves domain ownership by creating a temporary `_acme-challenge` TXT record via the Hetzner DNS API, so your Proxmox host does not need to be reachable from the internet on port 80 (unlike the HTTP-01/standalone challenge).

## Prerequisites

- Proxmox VE with web UI access (and `root@pam` for the CLI steps)
- A domain whose DNS zone is managed by **Hetzner DNS**
- A **Hetzner DNS API token** (this is the DNS Console token, *not* a Hetzner Cloud API token)
- Outbound internet access from the Proxmox host to Let's Encrypt

## 1. Get a Hetzner DNS API Token

1. Log into the [Hetzner DNS Console](https://dns.hetzner.com/).
2. Open **API Tokens** (top-right account menu).
3. Create a new token with access to your DNS zones.
4. Copy the token — you'll paste it into Proxmox next.

> This is the **DNS** API token from dns.hetzner.com, which maps to the acme.sh `HETZNER_Token` variable. It is different from the Hetzner Cloud token used by the `hcloud` CLI.

## 2. Register an ACME Account

### Web UI

1. Go to **Datacenter → ACME**.
2. Under **Accounts**, click **Add**.
3. Fill in:
   - **Account Name** — e.g. `letsencrypt`
   - **E-Mail** — a valid address (used for expiry notices)
   - **ACME Directory** — `https://acme-v02.api.letsencrypt.org/directory` (production)
4. Accept the Terms of Service and click **Register**.

### CLI

```bash
pvenode acme account register letsencrypt you@example.com \
  --directory https://acme-v02.api.letsencrypt.org/directory
```

> Tip: test with the **staging** directory first (`https://acme-staging-v02.api.letsencrypt.org/directory`) to avoid hitting Let's Encrypt rate limits while you get the setup working, then re-register with production.

## 3. Add the Hetzner DNS Challenge Plugin

### Web UI

1. In **Datacenter → ACME**, open the **Challenge Plugins** tab.
2. Click **Add**.
3. Configure:
   - **Plugin ID** — e.g. `hetzner`
   - **DNS API** — select **Hetzner DNS**
   - **HETZNER_Token** — paste your Hetzner DNS API token
   - Optionally raise the **validation delay** if DNS propagation is slow (see note below)
4. Click **Add**.

### CLI

The CLI passes API credentials via a data file of `KEY=VALUE` pairs (matching the acme.sh variable names).

```bash
# Create the credentials file
cat > /root/hetzner-dns.txt <<'EOF'
HETZNER_Token=your-hetzner-dns-api-token
EOF
chmod 600 /root/hetzner-dns.txt

# Register the plugin (dns_hetzner is the acme.sh API name)
pvenode acme plugin add dns hetzner \
  --api hetzner \
  --data /root/hetzner-dns.txt \
  --validation-delay 60
```

> `--validation-delay` (default 30s) is how long Proxmox waits after writing the TXT record before asking Let's Encrypt to validate. Hetzner DNS can be slow to propagate — bump this to 60–120s if orders fail with authorization/verification errors.

## 4. Assign the Domain to the Node

Tell the node which domain(s) to request and which plugin validates them.

### Web UI

1. Select your node (e.g. **pve**) → **System → Certificates**.
2. Under **ACME**, click **Add**.
3. Configure:
   - **Challenge Type** — DNS
   - **Plugin** — your `hetzner` plugin
   - **Domain** — the node FQDN, e.g. `proxmox.example.com` (or `*.example.com` for a wildcard)
4. Click **Create**, then set the ACME **Account** in the ACME panel on the same page if prompted.

### CLI

```bash
# First domain uses acmedomain0, additional SANs use acmedomain1, acmedomain2, ...
pvenode config set --acmedomain0 proxmox.example.com,plugin=hetzner

# Wildcard example
pvenode config set --acmedomain0 '*.example.com,plugin=hetzner'
```

## 5. Order and Install the Certificate

### Web UI

1. In **System → Certificates**, click **Order Certificates Now**.
2. Wait for the DNS challenge to complete — Proxmox creates the TXT record, waits for the validation delay, then Let's Encrypt verifies it and issues the cert.
3. Proxmox installs the certificate and reloads `pveproxy` automatically.

### CLI

```bash
pvenode acme cert order

# Re-order / force renew
pvenode acme cert order --force
```

## 6. Verify

```bash
# Show the active certificate details
pvenode cert info

# Confirm pveproxy is healthy after the reload
systemctl status pveproxy
```

Then open the web UI over HTTPS and confirm the certificate is issued by Let's Encrypt (R3/E-series) and valid for your domain. If it doesn't update, reload the web proxy:

```bash
systemctl reload pveproxy    # preferred — no session drop
systemctl restart pveproxy   # if a reload isn't enough
```

## Auto-Renewal

Proxmox renews ACME certificates automatically via a daily `pve-daily-update` timer; certificates are renewed when they fall within ~30 days of expiry, and `pveproxy` is reloaded with the new cert. No cron job is needed. To confirm the renewal path is healthy, check that the ACME account is registered (`pvenode acme account list`) and that `pvenode cert info` shows the Let's Encrypt cert.

## Important Notes

- The domain's authoritative DNS **must** be Hetzner DNS for the plugin's TXT records to take effect.
- Wildcards (`*.example.com`) require DNS-01 — they cannot be issued via HTTP-01.
- Keep `/root/hetzner-dns.txt` at `chmod 600`; it holds an API token. Rotate the token periodically.
- Let's Encrypt production has rate limits (per registered domain, per week) — use the staging directory while testing.
- The same ACME setup works for the API/cluster; each node orders its own node certificate.

## Troubleshooting

| Symptom | Check / fix |
|---------|-------------|
| Order fails at DNS validation | Increase `--validation-delay` (60–120s); Hetzner propagation can lag |
| "authorization ... invalid" | Token lacks zone access, or the domain isn't actually hosted on Hetzner DNS |
| Wrong token type | Use the **DNS Console** token (`HETZNER_Token`), not a Hetzner Cloud (`hcloud`) token |
| Cert issued but UI still self-signed | `systemctl reload pveproxy`, then hard-refresh the browser |
| Want to watch it happen | Tail the logs during an order (below) |

```bash
# Watch pveproxy / ACME activity during an order
journalctl -u pveproxy -f
journalctl -u pvedaemon -f

# Web server access log
tail -f /var/log/pveproxy/access.log

# Manually verify the challenge TXT record propagated
dig +short TXT _acme-challenge.proxmox.example.com @hydrogen.ns.hetzner.com
```

For preparing cloud images and other Proxmox host tooling, see [Why Proxmox Needs libguestfs-tools](articles/proxmox-libguestfs-tools.md). To manage Hetzner resources from the command line, see the [hcloud CLI Cheatsheet](articles/hcloud-cheatsheet.md).
