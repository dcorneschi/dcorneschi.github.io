# Postfix Gmail SMTP Relay Setup

Configure Postfix as a send-only relay through Gmail's SMTP server. This is ideal for homelab servers, cron jobs, monitoring tools, and system notifications that need to send mail (like `logwatch`, `mdadm`, or `unattended-upgrades`) without running a full mail server.

Gmail requires an **App Password** for SMTP authentication — your regular account password will not work when 2-Step Verification is enabled (and 2FA is required to generate App Passwords).

## Prerequisites

- A Gmail account with **2-Step Verification enabled**
- A **16-character App Password** generated at [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
- Root or `sudo` access on the server

> Gmail relay limits apply: roughly 500 recipients per day for free accounts and 2,000 for Google Workspace. For higher volume, use a dedicated transactional provider (SES, SendGrid, Mailgun).

## Install Required Packages

```bash
sudo apt update
sudo apt install -y libsasl2-modules sasl2-bin mailutils
```

| Package | Purpose |
|---------|---------|
| `postfix` | The MTA (installed as a dependency of `mailutils`, or install explicitly) |
| `libsasl2-modules` | SASL authentication plugins — required for Postfix to authenticate to Gmail |
| `sasl2-bin` | SASL utilities and helper binaries |
| `mailutils` | Provides the `mail`/`mailx` command for testing |

During installation, if prompted for a Postfix configuration type, choose **Internet Site** (or **Satellite system** for a pure relay). You can reconfigure later with `sudo dpkg-reconfigure postfix`.

## Back Up the Current Config

Always keep a copy before editing so you can roll back.

```bash
sudo cp /etc/postfix/main.cf /etc/postfix/main.cf.backup
```

## Configure main.cf

Edit the main Postfix configuration:

```bash
sudo vi /etc/postfix/main.cf
```

Add (or update) the following relay settings:

```ini
# Gmail SMTP Configuration
relayhost = [smtp.gmail.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_use_tls = yes
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
mydestination =
```

### What each directive does

| Directive | Meaning |
|-----------|---------|
| `relayhost = [smtp.gmail.com]:587` | Route all outbound mail through Gmail on the submission port. Brackets `[]` skip an MX lookup and connect directly to the host. |
| `smtp_sasl_auth_enable = yes` | Enable SASL authentication for the SMTP client. |
| `smtp_sasl_password_maps` | Path to the lookup table holding the Gmail credentials. |
| `smtp_sasl_security_options = noanonymous` | Disallow anonymous auth — credentials are always sent. |
| `smtp_use_tls = yes` | Encrypt the connection with STARTTLS (required by Gmail). |
| `smtp_tls_CAfile` | CA bundle used to verify Gmail's server certificate. |
| `mydestination =` | Empty — this host accepts no local mail delivery; everything is relayed out. |

> Port 587 uses STARTTLS (upgrade to TLS after connecting). Gmail also supports port 465 for implicit TLS; if you use it, set `smtp_tls_wrappermode = yes` and `relayhost = [smtp.gmail.com]:465`.

## Create the Credentials File

Create `/etc/postfix/sasl_passwd` with your Gmail address and App Password:

```bash
sudo vi /etc/postfix/sasl_passwd
```

Add a single line in the format `host:port user:password`:

```
[smtp.gmail.com]:587 YOUR_GMAIL@gmail.com:YOUR_16_CHAR_APP_PASSWORD
```

Enter the App Password **without spaces** (Google displays it as four groups of four, e.g. `abcd efgh ijkl mnop` → `abcdefghijklmnop`).

## Secure and Compile the Credentials

The file contains a plaintext secret, so restrict permissions before compiling it into a Postfix lookup database.

```bash
sudo chmod 600 /etc/postfix/sasl_passwd
sudo chown root:root /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
```

`postmap` generates `/etc/postfix/sasl_passwd.db` — the hashed database Postfix actually reads. Re-run `postmap` any time you change the source file. For safety, lock down the compiled DB too:

```bash
sudo chmod 600 /etc/postfix/sasl_passwd.db
```

## Restart Postfix

```bash
sudo systemctl restart postfix
sudo systemctl status postfix
```

Enable it on boot if it is not already:

```bash
sudo systemctl enable postfix
```

## Test It

Send yourself a test message:

```bash
echo "Postfix relay test from $(hostname)" | mail -s "Test $(date)" you@example.com
```

Watch the mail log to confirm delivery:

```bash
sudo tail -f /var/log/mail.log        # Debian/Ubuntu
sudo journalctl -u postfix -f
```

A successful relay shows a line like:

```
status=sent (250 2.0.0 OK ... - gsmtp)
```

## Troubleshooting

| Symptom | Likely cause / fix |
|---------|-------------------|
| `SASL authentication failed` / `535 5.7.8 Username and Password not accepted` | Wrong App Password, spaces left in it, or 2FA not enabled. Regenerate the App Password. |
| `warning: SASL authentication failure: No worthy mechs found` | `libsasl2-modules` not installed. Install it and restart Postfix. |
| `Connection timed out` to port 587 | Outbound port 587 blocked by firewall/ISP. Try port 465, or open the port. |
| `Cannot start TLS: handshake failure` | Missing CA bundle. Ensure `ca-certificates` is installed and `smtp_tls_CAfile` points to a valid file. |
| Mail stuck in queue | Inspect with `mailq`; flush with `sudo postqueue -f`; delete all with `sudo postsuper -d ALL`. |
| Changes not taking effect | Re-run `sudo postmap /etc/postfix/sasl_passwd` and `sudo systemctl reload postfix`. |

### Useful commands

```bash
mailq                      # view the mail queue
sudo postqueue -f          # attempt to flush the queue now
sudo postsuper -d ALL      # delete all queued mail
sudo postconf -n           # show non-default settings
sudo postfix check         # validate configuration
```

## Optional: Rewrite the Sender Address

Gmail rewrites the `From` header to your authenticated address anyway, but you can normalize local sender addresses (e.g. `root@server`) so they look consistent:

```ini
# in main.cf
sender_canonical_maps = regexp:/etc/postfix/sender_canonical
```

```bash
# /etc/postfix/sender_canonical
/.+/    YOUR_GMAIL@gmail.com
```

Then reload:

```bash
sudo systemctl reload postfix
```

## Security Notes

- The App Password lives in plaintext in `sasl_passwd`; keep it `chmod 600`, root-owned, and out of version control and backups that leave the host.
- Revoke the App Password from your Google account if the server is decommissioned or compromised.
- This setup is **send-only**. Postfix is not listening for inbound mail from the internet with `mydestination` empty and no public listener configured.
