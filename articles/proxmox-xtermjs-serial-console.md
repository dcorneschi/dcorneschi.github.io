# Proxmox xterm.js Serial Console

xterm.js is a web-based **serial** terminal in the Proxmox UI. Unlike the default noVNC console — which emulates a monitor/display — xterm.js connects to the VM through a serial port, behaving like a real hardware serial cable plugged into a server's COM port. That makes it the better tool for catching the GRUB menu, working with headless VMs, and reliably copying log output.

> These steps involve a change on the **Proxmox host** (adding a serial device) and changes **inside the guest** (kernel/GRUB console config). Both are required — the host device alone gives you a blank terminal.

## noVNC vs xterm.js

| noVNC (default) | xterm.js (serial) |
|-----------------|-------------------|
| Emulates a monitor/display | Direct serial text console |
| Graphical, but keystrokes can lag | Pure text, responsive input |
| Hard to catch GRUB (delayed key events) | Easy to interrupt GRUB |
| Copy/paste unreliable | Full copy/paste |
| No scrollback | Scroll buffer available |
| Needs a VGA/display driver in the guest | Works without display drivers |
| Heavier on the browser | Lightweight |

### Key use cases

- Catching the GRUB menu on fast-booting VMs (RHEL 10, Ubuntu cloud images)
- Troubleshooting VMs with no network (can't SSH)
- Headless VMs and cloud-init templates with no display
- Reliable copy/paste of log output from a broken VM
- Automation and scripting over the serial port

## Step 1: Add a Serial Device to the VM

On the Proxmox host:

```bash
qm set <vmid> --serial0 socket
```

Or in the UI: select the VM → **Hardware** → **Add** → **Serial Port**, set the number to `0`, leave type as `socket`.

## Step 2: Configure the Guest to Use the Serial Console

The guest must send its console output to the serial port. Without this, xterm.js shows a blank screen.

### RHEL / Rocky / AlmaLinux / CentOS

```bash
# Add the serial console to the kernel command line
grubby --update-kernel=ALL --args="console=tty0 console=ttyS0,115200n8"

# Enable the serial login prompt
systemctl enable --now serial-getty@ttyS0.service
```

### Ubuntu / Debian

```bash
# Set the kernel command line in GRUB defaults
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT="console=tty0 console=ttyS0,115200n8"/' /etc/default/grub
update-grub

# Enable the serial login prompt
systemctl enable --now serial-getty@ttyS0.service
```

### For cloud-init templates (offline, before templating)

Bake the config into the image with `virt-customize` (from [libguestfs-tools](articles/proxmox-libguestfs-tools.md)):

```bash
# RHEL-based image
virt-customize -a /path/to/image.qcow2 \
    --run-command "grubby --update-kernel=ALL --args='console=tty0 console=ttyS0,115200n8'" \
    --run-command "systemctl enable serial-getty@ttyS0.service"

# Debian-based image
virt-customize -a /path/to/image.qcow2 \
    --run-command "sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT=\"console=tty0 console=ttyS0,115200n8\"/' /etc/default/grub" \
    --run-command "update-grub" \
    --run-command "systemctl enable serial-getty@ttyS0.service"
```

## Step 3: Show GRUB on Serial (to catch the boot menu)

To interrupt or read the GRUB menu over xterm.js, tell GRUB itself to use the serial terminal. Add to `/etc/default/grub`:

```ini
GRUB_TERMINAL="console serial"
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
GRUB_TIMEOUT=5
```

Then regenerate the GRUB config:

```bash
# RHEL / Rocky
grub2-mkconfig -o /boot/grub2/grub.cfg

# Ubuntu / Debian
update-grub
```

Offline with `virt-customize`:

```bash
virt-customize -a /path/to/image.qcow2 \
    --run-command "echo 'GRUB_TERMINAL=\"console serial\"' >> /etc/default/grub" \
    --run-command "echo 'GRUB_SERIAL_COMMAND=\"serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1\"' >> /etc/default/grub" \
    --run-command "sed -i 's/GRUB_TIMEOUT=.*/GRUB_TIMEOUT=5/' /etc/default/grub" \
    --run-command "grub2-mkconfig -o /boot/grub2/grub.cfg"
```

## Accessing xterm.js

From the Proxmox UI:

1. Select the VM in the sidebar.
2. Open the **Console** dropdown (top-right of the console panel).
3. Choose **xterm.js**.

### Make xterm.js the default console

```bash
qm set <vmid> --vga serial0
```

This makes the serial console the VM's primary display — noVNC will no longer show graphical output.

> With `--vga serial0` the VM has **no graphical display at all**. Use it only for headless/server VMs.

### Keep both options

To switch between noVNC and xterm.js from the dropdown, **do not** set `--vga serial0`. Keep the default VGA and just add `--serial0 socket`.

## The Console Parameter Explained

```
console=tty0 console=ttyS0,115200n8
```

| Part | Meaning |
|------|---------|
| `console=tty0` | Keep output on the regular display (noVNC still works) |
| `console=ttyS0` | Also send output to the first serial port |
| `115200` | Baud rate |
| `n` | No parity |
| `8` | 8 data bits |

Order matters: the **last** `console=` becomes the primary console where the login prompt appears. Putting `ttyS0` last gives the serial console the login prompt.

## Troubleshooting

### Blank screen in xterm.js

- Guest is missing `console=ttyS0` in its kernel parameters
- `serial-getty@ttyS0.service` is not enabled in the guest
- No serial device on the VM — run `qm set <vmid> --serial0 socket`

### Boot messages appear but no login prompt

- `serial-getty@ttyS0.service` is not running
- Check inside the guest: `systemctl status serial-getty@ttyS0.service`

### GRUB still not visible

- `/etc/default/grub` needs `GRUB_TERMINAL="console serial"`
- Regenerate the GRUB config after changes
- EFI VMs may also need the EFI grub.cfg updated (e.g. `/boot/efi/EFI/redhat/grub.cfg`)

### Garbled or missing characters

- Baud rate mismatch between the GRUB config and the kernel parameter
- Use `115200` consistently everywhere

## Baking It Into a Template Build

Add serial-console support to a template build script so every clone has it ready:

```bash
# Alongside qm create — add the serial device
qm set "$template_id" --serial0 socket

# In the virt-customize step (RHEL-based)
virt-customize -a "$image_name" \
    --run-command "grubby --update-kernel=ALL --args='console=tty0 console=ttyS0,115200n8'" \
    --run-command "systemctl enable serial-getty@ttyS0.service" \
    --run-command "echo 'GRUB_TERMINAL=\"console serial\"' >> /etc/default/grub" \
    --run-command "echo 'GRUB_SERIAL_COMMAND=\"serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1\"' >> /etc/default/grub" \
    --run-command "sed -i 's/GRUB_TIMEOUT=.*/GRUB_TIMEOUT=5/' /etc/default/grub" \
    --run-command "grub2-mkconfig -o /boot/grub2/grub.cfg"
```

Every VM cloned from the template will then have a working serial console with a catchable GRUB menu over xterm.js. For more on preparing images offline, see [Why Proxmox Needs libguestfs-tools](articles/proxmox-libguestfs-tools.md).
