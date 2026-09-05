# LUKS Disk Encryption and NBDE (Tang/Clevis)

Encrypting disks at rest protects data if a drive is stolen or decommissioned. On Linux,
**LUKS** (Linux Unified Key Setup) is the standard for full-block-device encryption. The
catch: an encrypted volume needs a passphrase at boot, which is painful for headless
servers. **NBDE** (Network-Bound Disk Encryption) — using **Tang** servers and **Clevis**
clients — solves that by automatically unlocking volumes at boot as long as the machine can
reach a trusted server on the network.

This guide covers creating and managing LUKS volumes, persisting them, and setting up
automatic unlock with Tang/Clevis (including a redundant multi-server policy).

> Commands use the RHEL/CentOS/Fedora family (`yum`/`dnf`, `firewall-cmd`). The LUKS and
> Clevis parts apply to any modern Linux; the packaging commands differ on Debian/Ubuntu
> (`apt install cryptsetup clevis clevis-luks clevis-initramfs tang`).

---

## LUKS basics

LUKS stores its key material in a header on the block device. You unlock the device (which
creates a decrypted mapping under `/dev/mapper/<name>`), put a filesystem on that mapping,
and mount it — the underlying device stays encrypted.

### Encrypt a device after installation

```bash
# 1. Prepare a partition (example: a whole second disk)
parted -l                                         # inspect current layout
parted /dev/vdb mklabel msdos mkpart primary xfs 1M 1G
parted /dev/vdb print

# 2. Format it as LUKS (destroys existing data). Prompts for a passphrase,
#    or use --key-file for an unattended key.
cryptsetup luksFormat /dev/vdb1
# cryptsetup luksFormat /dev/vdb1 --key-file /path/to/keyfile

# 3. Inspect the LUKS header (key slots, cipher, UUID)
cryptsetup luksDump /dev/vdb1

# 4. Open it → creates /dev/mapper/example
cryptsetup luksOpen /dev/vdb1 example
ls -l /dev/mapper/example

# 5. Make a filesystem and mount the *mapping*, not the raw device
mkfs.xfs /dev/mapper/example
mount -t xfs /dev/mapper/example /encrypted

# ...later, to close it
umount /encrypted
cryptsetup luksClose example
```

### Managing keys

LUKS supports multiple key slots, so you can have several passphrases/keyfiles for one
device (e.g. a human passphrase plus an automation keyfile).

```bash
# Add a second key (you're prompted for an existing key first, then the new one)
cryptsetup luksAddKey --key-slot 1 /dev/vdb1

# Change an existing passphrase
cryptsetup luksChangeKey /dev/vdb1

# Remove a key slot
cryptsetup luksRemoveKey /dev/vdb1
```

### Encrypt at install time (Kickstart)

For automated RHEL installs, encrypt during partitioning:

```bash
# Encrypt everything with automatic LVM partitioning
autopart --type=lvm --encrypted --passphrase=PASSPHRASE

# Or per-partition / per-PV
part /home --fstype=ext4 --size=10000 --onpart=vda2 --encrypted --passphrase=PASSPHRASE
part pv.01 --size=10000 --encrypted --passphrase=PASSPHRASE
```

## Persistently mounting LUKS volumes

Two files work together at boot:

- **`/etc/crypttab`** — maps a LUKS device to a decrypted name (the `/dev/mapper/<name>`).
- **`/etc/fstab`** — mounts that decrypted mapping.

```bash
# /etc/crypttab  —  <name>  <device>  <key>  <options>
decrypted1  /dev/vdb1                                     none  _netdev
decrypted2  UUID=43d8995e-b876-4385-b124-7e402446d6c7     none  _netdev
```

```bash
# /etc/fstab  —  mount the mapper device, not the raw partition
/dev/mapper/decrypted1  /encrypted  xfs  _netdev  1 2
```

- `none` in the key field means "prompt for the passphrase at boot" (unless Clevis is
  bound — see below).
- **`_netdev`** marks the volume as network-dependent — important when unlock relies on a
  Tang server, so systemd waits for the network.
- Prefer `UUID=` over device names (`/dev/vdb1`) so reordering disks doesn't break boot.

## NBDE: automatic unlock with Tang and Clevis

NBDE unlocks LUKS volumes at boot **without a typed passphrase**, by binding the volume to
one or more **Tang** servers. **Clevis** (on the client) contacts Tang at boot; if the
server is reachable and returns its key material, the volume unlocks automatically. No
secrets are stored on the client, and Tang never sees the actual disk key — it's a
provable key-exchange, not a password vault.

Use it for: headless servers, remote/branch machines, and disk-theft protection where you
still want unattended reboots (as long as the box is on the trusted network).

### Set up a Tang server

```bash
# Install and enable Tang (binds to 80/TCP)
yum -y install tang
systemctl enable tangd.socket --now

# Open the firewall
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --reload
```

Tang generates its cryptographic keys on first start under `/var/db/tang`. If you ever need
to rotate keys manually:

```bash
cd /var/db/tang
jose jwk gen -i '{"alg":"ES512"}' -o signature.jwk      # new signing key
jose jwk gen -i '{"alg":"ECMR"}'  -o exchange.jwk       # new exchange key
# Retire old keys by prefixing their filenames with a dot (hidden = no longer advertised)
mv -v gxB7oqYiEu3zrLay.jwk .gxB7oqYiEu3zrLay.jwk
```

### Bind a client volume to Tang

```bash
# Install Clevis + the LUKS and dracut integrations
yum install clevis clevis-luks clevis-dracut

# Bind a LUKS device to a Tang server (you'll confirm the server's key thumbprint)
clevis luks bind -d /dev/vda1 tang '{"url":"http://tang.example.com"}'

# Verify Clevis metadata is now in the LUKS header
luksmeta show -d /dev/vda1

# Rebuild the initramfs so early boot can unlock the (root) device
dracut -f

# For a *non-root* filesystem, enable the unlock path unit
systemctl enable clevis-luks-askpass.path
```

After this, a reboot unlocks the bound volume automatically as long as the Tang server is
reachable. The original LUKS passphrase still works as a fallback (it's in a separate key
slot), so keep it somewhere safe for recovery.

### Redundant unlock with SSS (multiple Tang servers)

Relying on a single Tang server is a single point of failure. **Shamir's Secret Sharing
(SSS)** lets you bind to several servers and require a **threshold** (`t`) of them — e.g.
"any 2 of 3 must be reachable" — for the volume to unlock.

```bash
cfg='{"t":2,"pins":{"tang":[
  {"url":"http://tang1.example.com"},
  {"url":"http://tang2.example.com"},
  {"url":"http://tang3.example.com"}
]}}'

clevis luks bind -d /dev/vdb1 sss "$cfg"
```

That policy (threshold 2, three Tang servers) means the machine boots unattended as long
as **at least two** of the three Tang servers are up — tolerating one server outage without
falling back to a manual passphrase.

## Recovery and troubleshooting

- **Always keep a passphrase key slot.** Clevis binding adds a slot; it doesn't remove your
  passphrase. If Tang is unreachable and SSS threshold isn't met, you'll be prompted for
  the passphrase — make sure you still have it.
- **Inspect what's bound:** `cryptsetup luksDump /dev/vdX` shows key slots;
  `luksmeta show -d /dev/vdX` shows Clevis bindings.
- **Root won't auto-unlock after binding?** You likely forgot `dracut -f` to rebuild the
  initramfs, or the volume/mount isn't marked `_netdev` so the network wasn't up yet.
- **Test Tang reachability from the client** before rebooting a remote box:
  `curl -s http://tang.example.com/adv` should return the server's advertisement.
- **Rotate Tang keys carefully** — after rotating, re-bind clients (`clevis luks bind`
  again) or they'll fall back to passphrase once the old key is fully retired.

## Quick reference

| Task | Command |
|------|---------|
| Format device as LUKS | `cryptsetup luksFormat /dev/vdX` |
| Open (decrypt) | `cryptsetup luksOpen /dev/vdX name` |
| Close | `cryptsetup luksClose name` |
| Inspect header | `cryptsetup luksDump /dev/vdX` |
| Add / change key | `cryptsetup luksAddKey` / `luksChangeKey /dev/vdX` |
| Install Tang server | `yum -y install tang && systemctl enable tangd.socket --now` |
| Bind to one Tang | `clevis luks bind -d /dev/vdX tang '{"url":"http://tang..."}'` |
| Bind to N-of-M Tang | `clevis luks bind -d /dev/vdX sss "$cfg"` |
| Show Clevis bindings | `luksmeta show -d /dev/vdX` |
| Rebuild initramfs | `dracut -f` |

---

### Sources

- [Encrypting block devices using LUKS (Red Hat)](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/security_hardening/encrypting-block-devices-using-luks_security-hardening)
- [Configuring automated unlocking with NBDE (Red Hat)](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/security_hardening/configuring-automated-unlocking-of-encrypted-volumes-using-policy-based-decryption_security-hardening)
- [Clevis project](https://github.com/latchset/clevis)
- [Tang project](https://github.com/latchset/tang)
- [cryptsetup / LUKS documentation](https://gitlab.com/cryptsetup/cryptsetup/-/wikis/home)
