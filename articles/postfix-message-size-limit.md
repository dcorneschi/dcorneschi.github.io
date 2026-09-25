# Increasing the Postfix Message Size Limit

When a message or attachment exceeds Postfix's configured maximum, the mail is rejected outright and the sender sees a "message file too big" error. The fix is to raise `message_size_limit` (and keep `mailbox_size_limit` in step with it). This guide covers diagnosing the error, changing the limits safely with `postconf`, the MIME-encoding overhead to account for, and how to verify the new value.

For sending mail through an external relay, see [Postfix Gmail SMTP Relay Setup](articles/postfix-gmail-relay.md).

## Diagnose the Error

An oversized message shows up in the mail log (`/var/log/maillog` on RHEL/CentOS, `/var/log/mail.log` on Debian/Ubuntu):

```sh
less /var/log/maillog
```

```text
postfix/postdrop[27766]: warning: uid=0: File too large
postfix/sendmail[27765]: fatal: root(0): message file too big
```

`message file too big` means the submitted message exceeded `message_size_limit`. The mail is not queued — it's rejected at submission.

## Check the Current Limit

`postconf` reads the effective configuration, including compiled-in defaults that aren't written in `main.cf`:

```sh
postconf message_size_limit
```

```text
message_size_limit = 10240000
```

The value is in **bytes**. The default of `10240000` is roughly 10 MB.

## Account for MIME Encoding Overhead

The limit applies to the **encoded** message, not the raw file. Attachments are MIME/base64 encoded for transport, which inflates their size by about **33%**. So a 36 MB attachment becomes roughly 48 MB on the wire, plus headers.

Size the limit for the encoded payload, not the original file. A rough rule: multiply the largest attachment you want to allow by 1.37 and round up. For a ~36 MB attachment, 50 MB is a safe limit.

## Raise the Limit

Use `postconf -e` to edit `main.cf` in place. This example raises the limit from 10 MB to 50 MB (52428800 bytes):

```sh
postconf -e message_size_limit=52428800
```

> `postconf -e` rewrites `/etc/postfix/main.cf` and the change is picked up without a manual restart. If you instead edit `main.cf` by hand, reload Postfix with `systemctl reload postfix` (or `/etc/init.d/postfix restart` on older init systems) to apply it.

### Keep mailbox_size_limit in Step

`mailbox_size_limit` (default `51200000`) caps the size of a local mailbox file, and **it must not be smaller than `message_size_limit`**. If it is, local delivery fails:

```text
postfix/local[2105]: fatal: main.cf configuration error: mailbox_size_limit is smaller than message_size_limit
```

Raise it to at least the message limit (here, match it at 50 MB):

```sh
postconf -e mailbox_size_limit=52428800
```

A value of `0` disables the mailbox size check entirely, but matching or exceeding `message_size_limit` is the safer choice. Leaving `mailbox_size_limit` too low relative to the message limit also produces warnings in the mail log.

## Verify the New Limit

```sh
postconf message_size_limit
```

```text
message_size_limit = 52428800
```

Check both values at once:

```sh
postconf message_size_limit mailbox_size_limit
```

## Test With a Real Attachment

Create a file just under the new limit and mail it to a local user. Remember the MIME overhead — a 35 MB file encodes to about 47 MB, comfortably under 50 MB:

```sh
truncate -s 35M test.txt
echo "test attach" | mailx -s "subject" -a test.txt root@localhost
```

Then confirm delivery succeeded rather than being rejected:

```sh
tail -f /var/log/maillog
```

A successful send shows a `status=sent` line; a rejection repeats the `message file too big` fatal.

## Key Takeaways

- `message file too big` in the mail log means the message exceeded `message_size_limit` (default ~10 MB).
- Change it with `postconf -e message_size_limit=<bytes>`; values are in **bytes** and `postconf -e` needs no restart.
- Always keep `mailbox_size_limit` **≥** `message_size_limit`, or local delivery fails with a fatal config error.
- MIME/base64 encoding adds ~33% overhead, so size the limit for the encoded payload, not the raw file.
- Verify with `postconf message_size_limit mailbox_size_limit` and test with a real attachment.

## Quick Reference

```sh
# Inspect current limits
postconf message_size_limit mailbox_size_limit

# Raise to 50 MB (bytes) — no restart needed with postconf -e
postconf -e message_size_limit=52428800
postconf -e mailbox_size_limit=52428800

# Test with a ~35 MB attachment (encodes to ~47 MB)
truncate -s 35M test.txt
echo "test attach" | mailx -s "subject" -a test.txt root@localhost

# Watch delivery
tail -f /var/log/maillog     # RHEL/CentOS
tail -f /var/log/mail.log    # Debian/Ubuntu
```

For related material, see [Postfix Gmail SMTP Relay Setup](articles/postfix-gmail-relay.md).
