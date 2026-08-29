# AIX CDE and X Window System Cheatsheet

Command reference for managing the graphical desktop on IBM AIX — enabling or disabling the CDE (Common Desktop Environment) login manager with `dtconfig`, starting `dtlogin` manually, and bringing up X11/CDE sessions with `xinit`.

> These commands require `root`. `dtconfig` changes take effect on the next reboot (it edits the inittab entry for the desktop), while `dtlogin`/`xinit` start a session in the current boot.

## dtconfig — Enable/Disable Desktop Autostart

`dtconfig` controls whether the CDE login manager (`dtlogin`) starts automatically at boot, by managing its `/etc/inittab` entry.

```sh
# Disable desktop (graphical) logins at boot
/usr/dt/bin/dtconfig -d

# Enable desktop logins at boot
/usr/dt/bin/dtconfig -e
```

| Flag | Effect |
|------|--------|
| `-e` | Enable autostart of the CDE login manager |
| `-d` | Disable autostart (console/CLI login only) |

> After `dtconfig -d`/`-e`, the change applies on the next reboot. To manage the inittab entry directly, see the [AIX Boot and Init Cheatsheet](articles/aix-boot-init-cheatsheet.md).

## dtlogin — Start the Login Manager Manually

```sh
# Start the CDE graphical login manager now (without rebooting)
/usr/dt/bin/dtlogin -daemon
```

Use this to bring up the graphical login screen in the current session when autostart is disabled or has been stopped.

## xinit — Start an X / CDE Session from the CLI

`xinit` initializes the X Window System and optionally runs a session/startup script. Run it from a text console to launch a desktop directly (bypassing `dtlogin`).

```sh
# Initialize a bare X session
xinit

# Start the CDE desktop using the site-customized session script
xinit /etc/dt/Xsession

# Start the CDE desktop using the default session script
xinit /usr/dt/bin/Xsession
```

> `/etc/dt/Xsession` holds site/customized session settings and takes precedence when present; `/usr/dt/bin/Xsession` is the shipped default. (Some environments spell the directory `Xsessions` — check what exists on your host.)

## Quick Reference

| Task | Command |
|------|---------|
| Disable desktop autostart | `/usr/dt/bin/dtconfig -d` |
| Enable desktop autostart | `/usr/dt/bin/dtconfig -e` |
| Start login manager now | `/usr/dt/bin/dtlogin -daemon` |
| Bare X session | `xinit` |
| CDE (customized) | `xinit /etc/dt/Xsession` |
| CDE (default) | `xinit /usr/dt/bin/Xsession` |

## Related

- [AIX Boot and Init Cheatsheet](articles/aix-boot-init-cheatsheet.md) — inittab and run-level control (how the desktop entry is started at boot)
- [AIX Backup and Recovery Cheatsheet](articles/aix-backup-recovery-cheatsheet.md) — mksysb, volume group, and file-level backups
