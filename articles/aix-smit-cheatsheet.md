# AIX SMIT Cheatsheet

SMIT (System Management Interface Tool) is IBM AIX's menu-driven administration interface. It builds and runs the underlying high-level commands for you — so you can learn the exact command being executed while using the menus. `smit` launches the Motif/X11 (GUI) version when a display is available; `smitty` forces the ASCII/curses version in a terminal. Both do the same work.

> SMIT never does anything the CLI can't — every panel maps to a real command. Press **F6** on any dialog to see the exact command SMIT will run, then use it directly in scripts.

## Launching SMIT

```sh
smit            # GUI (X11) if a display is set, otherwise ASCII
smitty          # always the ASCII/curses interface
smit -C         # force the ASCII (curses) interface
smit -M         # force the Motif (X11) interface
smit -x         # log the commands but do NOT run them (dry run)

# Redirect the log and script files
smit -s /u/team1/smit.script -l /u/team1/smit.log
```

> `smit -x` is ideal for discovery: walk through the menus and it records exactly what *would* run in `smit.script` without touching the system.

### Jump straight to a panel with a fast path

Every SMIT menu/dialog has a **fast path** — an identifier you can pass to jump directly there, skipping the menu hierarchy.

```sh
smitty <fastpath>          # e.g. smitty jfs2, smitty user, smitty tcpip
smit <fastpath>
```

Find a panel's fast path from inside SMIT by navigating to it and pressing **F8** — the current fast path is shown at the top of the screen.

## Common Fast Paths

| Fast path | Opens |
|-----------|-------|
| `smitty install` | Software installation and maintenance |
| `smitty install_latest` | Install/update software (installp) |
| `smitty update_all` | Update all installed filesets |
| `smitty commit` / `smitty reject` | Commit or reject applied updates |
| `smitty lslpp` | List installed software |
| `smitty user` | User administration (add/change/remove) |
| `smitty mkuser` / `smitty chuser` / `smitty rmuser` | Add / change / remove a user |
| `smitty passwd` | Change a password |
| `smitty group` | Group administration |
| `smitty jfs2` | JFS2 filesystems |
| `smitty crfs` / `smitty chfs` | Create / change a filesystem |
| `smitty lvm` | Logical Volume Manager menu |
| `smitty mkvg` / `smitty vg` | Create / manage volume groups |
| `smitty mklv` / `smitty lv` | Create / manage logical volumes |
| `smitty disk` / `smitty pv` | Disks / physical volumes |
| `smitty tcpip` | TCP/IP configuration |
| `smitty chinet` | Change a network interface |
| `smitty mktcpip` | Minimum TCP/IP config (hostname, IP, gateway, DNS) |
| `smitty nfs` | NFS configuration |
| `smitty mount` | Mount a filesystem |
| `smitty devices` | Device management |
| `smitty chgsys` | Change system run-time attributes |
| `smitty date` | Set date/time and timezone |
| `smitty nim` | NIM (Network Installation Management) |
| `smitty nim_mkmac` | Define a NIM client machine |
| `smitty nim_bosinst` | NIM BOS install on a client |
| `smitty chps` / `smitty pgsp` | Change / add paging space |
| `smitty shutdown` | Shut down the system |
| `smitty diag` | Diagnostics |
| `smitty tape` | Tape drive management |

## Navigation Keys

### ASCII (smitty) interface

On terminals without function keys, press **Esc** then the number — e.g. `Esc 3` = F3, `Esc 0` = F10.

**Menu / dialog control:**

| Key | Action |
|-----|--------|
| **Enter** | Select the highlighted item / run the dialog |
| **Arrow keys** | Move up/down through menu items |
| **F1** (Esc+1) | Help for the current field/panel |
| **F2** (Esc+2) | Refresh the screen |
| **F3** (Esc+3) | Cancel — go back to the previous screen |
| **F8** (Esc+8) | Show the current fast path (and image/screen name) |
| **F9** (Esc+9) | Escape to a shell |
| **F10** (Esc+0) | Exit SMIT |
| **F6** (Esc+6) | Show the command SMIT will run for the current dialog |
| **/** | Search within a list |

**Selecting and editing fields:**

| Key | Action |
|-----|--------|
| **F4** (Esc+4) | List valid choices for the field (a picker) |
| **Tab** | Toggle between fixed choices (e.g. true/false) |
| **F5** (Esc+5) | Reset the entry field to its original value |
| **F7** (Esc+7) | Select an item in a multi-select list; also view/edit a long field |
| **Ctrl-x** | Delete the next character |
| **Ctrl-k** | Delete to end of line |

**Scrolling (menus and command output):**

| Key | Action |
|-----|--------|
| **PageDown** / **Ctrl-V** | Next page |
| **PageUp** / **Esc-V** | Previous page |
| **Home** / **Esc <** | Top of the output |
| **End** / **Esc >** | Bottom of the output |

### Field markers

Symbols next to a field tell you what kind of input is expected:

| Symbol | Meaning |
|--------|---------|
| `*` | Required field |
| `#` | Numeric value required |
| `/` | Pathname required |
| `X` | Hexadecimal value required |
| `?` | Input is not displayed (e.g. a password) |
| `+` | A pop-up list or ring of valid values is available (F4) |

## Logging and Output

SMIT records what it does, which is useful for auditing and for copying the exact commands into scripts.

| File | Contents |
|------|----------|
| `$HOME/smit.log` | Every menu/dialog visited and each command SMIT ran, with output and return codes |
| `$HOME/smit.script` | Just the executable commands SMIT ran (a ready-to-reuse shell script) |
| `$HOME/smit.transaction` | Detailed transaction records (ODM/interaction level) |

```sh
# Send the logs somewhere else
smitty -l /tmp/mysmit.log -s /tmp/mysmit.script

# Review the commands SMIT ran this session
cat $HOME/smit.script
```

> `smit.script` is a great way to learn AIX commands: do a task through the menus once, then read the generated command to automate it next time.

## Tips

- Prefer `smitty` (ASCII) over remote X11 — it's faster and works over any SSH session.
- Press **F6** before committing any change to capture the exact command; paste it into runbooks/scripts.
- Use **F4** liberally — it lists valid devices, volume groups, filesets, etc., so you don't have to guess names.
- Fast paths are stable across releases; `smitty jfs2`, `smitty user`, `smitty tcpip` etc. save a lot of menu navigation.
- If a task fails, check `$HOME/smit.log` for the failed command and its return code.

## Related

- [AIX LVM Cheatsheet](articles/aix-lvm-cheatsheet.md) — the commands behind `smitty lvm` (VGs, LVs, PVs)
- [AIX Filesystems Cheatsheet](articles/aix-filesystems-cheatsheet.md) — the commands behind `smitty jfs2`/`crfs`/`chfs`
- [AIX NIM Cheatsheet](articles/aix-nim-cheatsheet.md) — the commands behind `smitty nim`
- [AIX Backup and Recovery Cheatsheet](articles/aix-backup-recovery-cheatsheet.md) — mksysb and related backup tooling
