# Solaris Tips and Tricks

A grab-bag of small but useful fixes for day-to-day work on Oracle Solaris — correcting the backspace key, setting the terminal type per shell, and installing VirtualBox Guest Additions in a Solaris guest.

## Fix the Backspace Key (erase character)

On some terminals the Backspace key prints `^H` instead of deleting. Map the erase character to fix it. Add it to your shell startup file (`~/.profile` for sh/ksh, `~/.bashrc` for bash) so it persists:

```bash
vi ~/.profile
```

```bash
# Make Backspace erase (^H is entered literally as Ctrl-V then Backspace)
stty erase ^H
```

- Type the `^H` by pressing **Ctrl-V** then **Backspace** in the editor, so a real control character is stored (not the two literal characters `^` and `H`).
- If Backspace sends Delete instead, use `stty erase ^?` (Ctrl-V then Delete).
- Apply immediately in the current session by running the `stty` line at the prompt.

## Change the Terminal Type (TERM)

Misbehaving screen redraws, `vi`, or `clear` usually mean `TERM` is wrong. Set it to match your terminal (e.g. `vt100`, `xterm`, `dtterm`). The syntax differs by shell:

```bash
# Bourne shell (sh)
TERM=vt100; export TERM

# ksh, bash, or zsh
export TERM=vt100

# csh or tcsh
setenv TERM vt100
```

| Shell | Command |
|-------|---------|
| sh | `TERM=vt100; export TERM` |
| ksh / bash / zsh | `export TERM=vt100` |
| csh / tcsh | `setenv TERM vt100` |

Put the appropriate line in your shell startup file to make it stick (`~/.profile` for sh/ksh, `~/.bashrc` for bash, `~/.cshrc` for csh/tcsh). Common values: `vt100`, `xterm`, `xterm-256color`, `dtterm` (CDE), `sun-color` (Solaris console).

## Install VirtualBox Guest Additions (Solaris guest)

Running Solaris as a VirtualBox VM? The Guest Additions improve display, mouse integration, and shared folders. After inserting the Guest Additions CD (VirtualBox menu: *Devices → Insert Guest Additions CD image*), install the Solaris package:

```bash
# The mount path/version will match your VirtualBox version
cd /cdrom/vboxadditions_4.3.18_96516
pkgadd -G -d ./VBoxSolarisAdditions.pkg
```

- The directory name includes the VirtualBox version — `cd /cdrom/VBox*` or check `ls /cdrom` to find the exact path.
- `pkgadd -G` installs the package in the **current (global) zone only**, not propagated to non-global zones — appropriate for guest additions.
- `-d ./VBoxSolarisAdditions.pkg` installs from the package file on the mounted CD.
- Reboot afterward so the additions' drivers load: `init 6`.

If the CD isn't auto-mounted, restart the volume manager or mount it manually (see the disk/media notes in [Solaris SVR4 Package Management](articles/solaris-svr4-package-management.md)).

## Use the GNU/Modern Tools in /usr/gnu and /usr/bin

Solaris ships classic SVR4 versions of many commands whose options differ from GNU/Linux. On Solaris 11, GNU variants live under `/usr/gnu/bin`, and there's a more capable `xpg4` version of some tools:

```bash
# Put GNU tools ahead of the legacy ones for Linux-like behavior
export PATH=/usr/gnu/bin:/usr/bin:$PATH

# Examples where the legacy version surprises Linux admins:
/usr/xpg4/bin/grep -E 'foo|bar' file      # -E works reliably here
/usr/gnu/bin/tar xzf archive.tgz          # GNU tar handles -z; legacy tar does not
```

- Legacy `/usr/bin/grep` lacks some GNU extensions — use `/usr/xpg4/bin/grep` or `egrep`.
- Legacy `tar` can't gunzip on the fly (`z` flag) — use `gtar`/`/usr/gnu/bin/tar`, or pipe through `gzip`.
- Legacy `awk` is old; use `/usr/bin/nawk` or `/usr/gnu/bin/gawk`.

## Set a User's Default Shell

Solaris root's default shell is often `sh`/`ksh`, not bash. Change a login shell with:

```bash
# Per user (updates /etc/passwd) — the reliable method
usermod -s /usr/bin/bash username

# Interactive alternative: passwd -e prompts for the new shell (no -s argument)
passwd -e username

# Just for the current session
exec /usr/bin/bash
```

Available shells are listed in `/etc/shells`; common Solaris paths are `/usr/bin/bash`, `/usr/bin/ksh93`, `/usr/bin/csh`.

## Find Files and Commands

```bash
# Which binary will run, and where a command lives
which nawk
type ls

# Find files (legacy find is standard; GNU find under /usr/gnu/bin)
find / -name core -mtime +7

# Locate the package that delivers a command
pkgchk -l -p /usr/bin/ls        # Solaris 10 SVR4
pkg search -l -o pkg.name '/usr/bin/ls'   # Solaris 11 IPS
```

## Notes for Linux Admins

| Linux habit | Solaris equivalent |
|-------------|--------------------|
| `free -h` | `swap -s`, `echo ::memstat \| mdb -k` |
| `systemctl status foo` | `svcs -xv foo` |
| `top` | `prstat` |
| `apt/yum install` | `pkg install` (11) / `pkgadd` (10) |
| `useradd` defaults in `/etc/default/useradd` | `useradd -D` |
| `ip addr` | `ipadm show-addr` (11) / `ifconfig -a` (10) |
| `tar xzf` | `gtar xzf` or `/usr/gnu/bin/tar xzf` |

## Quick Reference

| Task | Command |
|------|---------|
| Fix Backspace (erase) | `stty erase ^H` (Ctrl-V, Backspace) |
| Backspace sends Delete | `stty erase ^?` |
| Set TERM (sh) | `TERM=vt100; export TERM` |
| Set TERM (bash/ksh/zsh) | `export TERM=vt100` |
| Set TERM (csh/tcsh) | `setenv TERM vt100` |
| Install VBox additions | `pkgadd -G -d ./VBoxSolarisAdditions.pkg` |

## References

- [stty(1) man page](https://docs.oracle.com/cd/E23824_01/html/821-1461/stty-1.html) — official Oracle docs
- [VirtualBox Guest Additions for Solaris](https://www.virtualbox.org/manual/ch04.html) — official VirtualBox docs
