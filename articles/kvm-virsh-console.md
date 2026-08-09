# Enable virsh console on KVM

`virsh console` provides text-based serial access to a VM — useful for headless servers, boot troubleshooting, or when networking is unavailable. By default, most guests don't have a serial console configured, so `virsh console` connects but shows a blank screen.

## The Problem

```bash
virsh console myvm
Connected to domain 'myvm'
Escape character is ^]
# Nothing happens — blank screen
```

This happens because:
1. The VM has no serial device defined, or
2. The guest OS isn't configured to output to the serial console

Both need to be set up.

## Step 1: Add Serial Device to VM (Host Side)

### Check If Serial Device Exists

```bash
virsh dumpxml myvm | grep -A5 serial
virsh dumpxml myvm | grep -A5 console
```

If no `<serial>` or `<console>` device is present, add one.

### Add via virsh edit

```bash
virsh edit myvm
```

Add inside `<devices>`:

```xml
<serial type='pty'>
  <target type='isa-serial' port='0'>
    <model name='isa-serial'/>
  </target>
</serial>
<console type='pty'>
  <target type='serial' port='0'/>
</console>
```

### Add at VM Creation (virt-install)

```bash
virt-install \
    --name myvm \
    --ram 2048 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/myvm.qcow2,size=20 \
    --os-variant rocky9.0 \
    --network network=default \
    --graphics none \
    --console pty,target_type=serial \
    --extra-args='console=ttyS0,115200n8 serial'
```

The key flags:
- `--graphics none` — no VNC/SPICE
- `--console pty,target_type=serial` — allocate serial console
- `--extra-args='console=ttyS0,115200n8'` — tell the kernel to use serial

## Step 2: Configure the Guest OS

### RHEL / CentOS / Rocky / AlmaLinux (GRUB2 + systemd)

#### RHEL 6 (GRUB Legacy)

Both methods require a reboot:

```bash
# Method 1: grubby (all kernels at once)
grubby --update-kernel=ALL --args="console=ttyS0"

# Method 2: Edit grub.conf manually
vi /boot/grub/grub.conf
# Add console=ttyS0 at the end of the kernel line:
# kernel /vmlinuz-2.6.32-754.el6.x86_64 ... rhgb quiet console=ttyS0
```

#### RHEL 7/8/9 — Enable getty on ttyS0 (No Reboot Required)

```bash
systemctl enable serial-getty@ttyS0.service
systemctl start serial-getty@ttyS0.service
```

#### Configure GRUB to Output to Serial

```bash
# Method 1: grubby (quick, all kernels — requires reboot)
grubby --update-kernel=ALL --args="console=ttyS0,115200n8"

# Method 2: Edit /etc/default/grub (full control — requires reboot)
vi /etc/default/grub
```

Add or modify:

```bash
GRUB_TERMINAL_OUTPUT="serial console"
GRUB_TERMINAL_INPUT="serial console"
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
GRUB_CMDLINE_LINUX="... console=tty0 console=ttyS0,115200n8"
```

> **Important:** The last `console=` in the kernel cmdline becomes the primary console. Put `ttyS0` last if you want serial as primary.

Rebuild GRUB config:

```bash
# BIOS
grub2-mkconfig -o /boot/grub2/grub.cfg

# UEFI
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
# or for Rocky/Alma:
grub2-mkconfig -o /boot/efi/EFI/rocky/grub.cfg
```

### RHEL 7 Specific

```bash
# Enable getty
systemctl enable serial-getty@ttyS0.service
systemctl start serial-getty@ttyS0.service

# Edit GRUB
vi /etc/default/grub
```

```bash
GRUB_TERMINAL="serial console"
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=rhel/root console=tty0 console=ttyS0,115200n8"
```

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

### Ubuntu / Debian

#### Enable getty on ttyS0

```bash
systemctl enable serial-getty@ttyS0.service
systemctl start serial-getty@ttyS0.service
```

#### Configure GRUB

```bash
vi /etc/default/grub
```

```bash
GRUB_TERMINAL="serial console"
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
GRUB_CMDLINE_LINUX_DEFAULT=""
GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8"
```

```bash
update-grub
```

### Ubuntu (Pre-systemd or Cloud Images)

For Ubuntu cloud images, the serial console is usually pre-configured. If not:

```bash
# /etc/init/ttyS0.conf (Upstart — older Ubuntu)
start on stopped rc RUNLEVEL=[2345]
stop on runlevel [!2345]
respawn
exec /sbin/getty -L 115200 ttyS0 vt102
```

### Verify Guest Configuration

```bash
# Check kernel cmdline (from inside the guest)
cat /proc/cmdline
# Should contain: console=ttyS0,115200n8

# Check if getty is running on ttyS0
systemctl status serial-getty@ttyS0.service

# Check if ttyS0 exists
ls -la /dev/ttyS0
dmesg | grep ttyS
```

## Step 3: Connect with virsh console

```bash
# Connect
virsh console myvm

# Connect to a specific serial device (if multiple)
virsh console myvm --devname serial0

# Force connect (disconnect existing sessions)
virsh console myvm --force

# Safe mode (exclusive access)
virsh console myvm --safe
```

### Disconnect

Press `Ctrl+]` (or `Ctrl+5` on some keyboards) to escape from the console.

### Successful Connection Example

```
[root@host ~]# virsh console myvm
Connected to domain 'myvm'
Escape character is ^]

CentOS Linux 7 (Core)
Kernel 3.10.0-1062.el7.x86_64 on an x86_64

localhost login:
```

## Quick One-Liner (Existing VM, Already Running)

If the VM already has a serial device but the guest isn't configured:

```bash
# SSH into the VM first, then:
sudo systemctl enable --now serial-getty@ttyS0.service
sudo sed -i 's/GRUB_CMDLINE_LINUX="/GRUB_CMDLINE_LINUX="console=ttyS0,115200n8 /' /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg   # RHEL
# or
sudo update-grub                                # Ubuntu
```

Then reboot for full GRUB + kernel output over serial.

## Without Reboot (Temporary)

If you just need console access NOW without rebooting:

```bash
# Inside the guest (via SSH):
sudo systemctl start serial-getty@ttyS0.service
```

Then from the host:

```bash
virsh console myvm
```

You'll get a login prompt (but no boot messages until GRUB is configured).

## Troubleshooting

### Console Connects But Shows Nothing

```bash
# Check serial device exists in VM XML
virsh dumpxml myvm | grep -A5 '<serial'

# If missing, add it and reboot the VM
virsh edit myvm
# Add <serial> and <console> elements
virsh reboot myvm

# Check getty is running inside guest
virsh qemu-agent-command myvm '{"execute":"guest-exec","arguments":{"path":"/bin/systemctl","arg":["status","serial-getty@ttyS0"]}}' 2>/dev/null
```

### "error: operation failed: Active console session exists for this domain"

Another session is already connected:

```bash
# Use --force to take over
virsh console myvm --force
```

### Console Shows Garbled Text

Baud rate mismatch between host and guest. Ensure both use 115200:

- Guest GRUB: `console=ttyS0,115200n8`
- VM XML serial device defaults to 115200

### Cannot Type After Connecting

The console might be connected to the wrong device. Check:

```bash
virsh dumpxml myvm | grep -B2 -A5 console
```

Ensure `<console type='pty'>` targets `serial` port `0`.

### UEFI VMs (OVMF)

UEFI firmware may not output to serial by default. Some OVMF builds include serial output, but you may also need:

```xml
<os>
  <type arch='x86_64' machine='q35'>hvm</type>
  <loader readonly='yes' type='pflash'>/usr/share/edk2/ovmf/OVMF_CODE.fd</loader>
  <nvram>/var/lib/libvirt/qemu/nvram/myvm_VARS.fd</nvram>
</os>
```

For UEFI serial output, add to the GRUB config inside the guest:

```bash
GRUB_TERMINAL_OUTPUT="serial console"
```

## Summary

| Step | Where | What |
|------|-------|------|
| 1 | Host (VM XML) | Add `<serial>` and `<console>` devices |
| 2 | Guest (GRUB) | Add `console=ttyS0,115200n8` to kernel cmdline |
| 3 | Guest (systemd) | Enable `serial-getty@ttyS0.service` |
| 4 | Host | Connect with `virsh console myvm` |
| | | Disconnect with `Ctrl+]` |
