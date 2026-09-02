# HP-UX Startup, Run Levels, and Network Services

How HP-UX boots to a running state and how its services are configured — the `init`/run-level model, the `/sbin/rc*.d` + `/etc/rc.config.d` startup framework, the `ch_rc` utility, custom startup scripts, and configuration of the common network services (inetd, AutoFS, CIFS, NTP, SSH, Serviceguard, and SMH). Covers HP-UX 11i v1–v3.

## From Kernel to init

The early boot stages just find and load the kernel (see the [HP-UX Boot Process](articles/hpux-boot-process.md) article for PDC → ISL → kernel loader detail). Once loaded:

1. The kernel does a sanity check on the root filesystem, then starts the **`init`** daemon.
2. `init` checks the filesystems in `/etc/fstab` for corruption, initializes the system console, and runs the tasks defined in **`/etc/inittab`**.
3. `init` brings the system to its default run level, calling `/sbin/rc` to start the necessary services.

### Why /etc/inittab Matters

`/etc/inittab` is the table `init` consults to decide what to run and at which run levels. Each line has the form `id:runlevels:action:process`. The `initdefault` line names the run level `init` targets on a normal boot (conventionally `3`), and `respawn` entries — such as the getty processes that provide login prompts — are restarted automatically by `init` whenever they exit. This is why a mistyped `initdefault` (for example set to `0` or `6`) can leave a machine that immediately halts or loops on every boot; recovery then requires interrupting boot and coming up in single-user mode to fix the file.

```bash
grep initdefault /etc/inittab      # confirm the default run level
```

After editing `/etc/inittab` you can make `init` re-read it without a reboot:

```bash
init q                              # re-read /etc/inittab (a.k.a. init Q)
```

## Run Levels

| Run level | Meaning |
|-----------|---------|
| `0` | System shutdown — stop all processes and halt |
| `s` | Single-user (admin); only the physical console has OS access. `/sbin/shutdown` goes here |
| `S` | Like `s`, but the console capability moves to your logged-in terminal (virtual console) |
| `1` | Single-user, but filesystems mounted and `syncer` running; admin tasks |
| `2` | Multiuser — all users can access the system |
| `3` | Networked multiuser — NFS filesystems exported; HP CDE active |
| `4`–`6` | Additional/defined levels |

```bash
who -r                 # show the current run level
init 3                 # change to run level 3
shutdown               # go to single-user (s)
```

HP-UX run levels are **hierarchical and additive**: reaching level 3 means the system passed through and started everything for levels 1 and 2 first. When you move *up* (say from 2 to 3), `init` runs only the start (`S`) scripts for the newly entered level; when you move *down* (from 3 to 2), it runs the kill (`K`) scripts for the level you are leaving. This is why services are placed at the lowest level where their prerequisites exist — networking comes up at level 2 so that network services layered on top can start at level 2 or 3.

Two important nuances:

- `who -r` shows the current level **and** the previous one, which is handy when diagnosing an incomplete transition.
- `shutdown` and `reboot` are the correct way to change to halt/reboot states; calling `init 0` or `init 6` directly skips the graceful `shutdown` notification and grace period. Prefer `shutdown -h now` (halt) and `shutdown -r now` (reboot).

## The /sbin/rc Startup Framework

At each run level, `init` calls **`/sbin/rc`**, which starts/stops services by consulting the `/sbin/rc*.d` directories:

- **`/sbin/init.d/`** — the actual service scripts (start/stop logic).
- **`/sbin/rc<N>.d/`** — per-run-level symlinks to the init.d scripts: `S###name` = start, `K###name` = kill, run in numeric order.
- **`/etc/rc.config.d/`** — a configuration file per service holding control variables and tunables.
- **`/etc/rc.log`** — records each service's startup status.

Most init.d scripts check a **control variable** in their `/etc/rc.config.d` file:

- Control variable **= 1** → the script runs at startup/shutdown.
- Control variable **= 0** → the script does not run.

So the pattern for the common network services is: they become available at **run level 2** (shortly after LAN cards configure), are started/stopped by scripts in `/sbin/init.d/`, and are configured via files in `/etc/rc.config.d/`.

### Anatomy of the Naming Convention

The `S###name` / `K###name` symlink names encode both *what* to do and *when*. The leading letter selects the action (`S` = start, `K` = kill), and the three-digit number sets the ordering within that run level — lower numbers run first on start, and the kill sequence is designed to run in the reverse order so dependencies unwind cleanly. A service that must start late (because it depends on others) therefore gets a high start number and, correspondingly, a low kill number in the next level down. This is exactly the `S900myservice` / `K100myservice` pairing shown later.

Every `/sbin/init.d` script is invoked by `/sbin/rc` with a single argument — `start`, `start_msg`, `stop`, or `stop_msg`:

- `start` / `stop` do the actual work.
- `start_msg` / `stop_msg` just print the one-line description `/sbin/rc` shows on the console and logs to `/etc/rc.log`.

### Reading /etc/rc.log

When a service fails to come up at boot, `/etc/rc.log` is the first place to look. It records each script's name, its start message, and the exit status. A well-behaved script returns one of three values that `/sbin/rc` interprets:

| Exit value | Meaning |
|------------|---------|
| `0` | Script succeeded |
| `1` | Script failed (an error is flagged in `/etc/rc.log`) |
| `2` | Script did nothing (for example the control variable was `0`) — treated as success, not an error |

Because a control variable of `0` yields exit `2` (not an error), a service that is simply *disabled* looks different in the log from one that *tried and failed* — a useful distinction when triaging a boot.

## Managing rc.config.d Variables with ch_rc

`ch_rc` views and edits `/etc/rc.config.d/` variables. If you don't give a filename, it searches the directory for the file containing the variable.

```bash
# View (-l) a parameter's current value
ch_rc -l -p CRON

# Add/modify (-a) a parameter — found automatically if it already exists
ch_rc -a -p CRON=0

# If the parameter doesn't exist yet, name the file explicitly
ch_rc -a -p CRON=0 /etc/rc.config.d/cron

# Query / modify a parameter in a specific file
ch_rc -l -p NIS_MASTER_SERVER /etc/rc.config.d/namesvrs
ch_rc -a -p NIS_MASTER_SERVER=1 /etc/rc.config.d/namesvrs

# Find duplicate entries; work with arrays; remove a variable
ch_rc -v -l -p NIS_MASTER_SERVER
ch_rc -A -lv -p SUBNET_MASK
ch_rc -r -p NIS_MASTER_SERVER /etc/rc.config.d/namesvrs
```

Why use `ch_rc` at all rather than just editing the file in `vi`? The `/etc/rc.config.d` files are shell scripts that are *sourced* at boot, and it is easy to introduce a syntax error (an unquoted value, a stray character, a duplicated variable) that silently breaks the whole file and every service that reads it. `ch_rc` parses and rewrites entries safely, refuses malformed input, and can locate the correct file for a variable automatically — so it is the safer choice for scripted or repeated changes. Hand-editing is fine for a quick one-off, but validate afterward.

> **Gotcha:** Setting a control variable to `1` with `ch_rc` enables a service *at the next reboot*; it does **not** start the daemon immediately. To start it now as well, run the matching `/sbin/init.d/<service> start`. Likewise, setting a variable to `0` does not stop a currently running daemon — stop it explicitly.

Common flags at a glance:

| Flag | Purpose |
|------|---------|
| `-l` | List (show) the current value of a parameter |
| `-a` | Add or modify a parameter |
| `-r` | Remove a parameter |
| `-p` | Specifies the parameter name (and, with `-a`, `=value`) |
| `-v` | Verbose; with `-l` reveals duplicate definitions |
| `-A` | Treat the parameter as an array |

## Creating a Custom Startup Script

HP-UX ships a template in `/sbin/init.d/`. Copy it, add a matching config file, and link it into the start/kill directories:

```bash
cp /sbin/init.d/template /sbin/init.d/myservice
vi /sbin/init.d/myservice                 # implement start/stop/restart
vi /etc/rc.config.d/myservice             # control variable + tunables

# Start at run level 3, kill at run level 2 (numbers set order)
ln -s /sbin/init.d/myservice /sbin/rc3.d/S900myservice
ln -s /sbin/init.d/myservice /sbin/rc2.d/K100myservice

# Find the config file that controls an existing service
grep -il sendmail /etc/rc.config.d/*
```

Test the script directly before trusting it to a reboot — this catches path, permission, and syntax problems while you can still see the output:

```bash
/sbin/init.d/myservice start
/sbin/init.d/myservice stop
```

A robust `/sbin/init.d` script follows a few conventions the template already sketches out:

- Source its config file (`. /etc/rc.config.d/myservice`) and honor the control variable — exit `2` if it is `0`, so a disabled service is not logged as a failure.
- Provide `start`, `stop`, `start_msg`, and `stop_msg` cases, and return the correct exit value (`0`/`1`/`2`) so `/etc/rc.log` is meaningful.
- Use absolute paths for every command, because `/sbin/rc` runs with a minimal environment and no guaranteed `PATH`.

## inetd (Internet Services Daemon)

```bash
swlist -l product InternetSrvcs        # verify installed
```

Files: `/etc/inetd.conf` (services), `/etc/rc.config.d/netdaemons` (enable/args), `/var/adm/inetd.sec` (access control), `/sbin/init.d/inetd` (script).

`inetd` is enabled by default. Disable or add logging via `/etc/rc.config.d/netdaemons`:

```bash
vi /etc/rc.config.d/netdaemons
export INETD=0                # disable inetd (variable added in 11i v2)
export INETD_ARGS="-l"        # enable connection logging to /var/adm/syslog/syslog.log
```

Reload after editing `/etc/inetd.conf` (no full restart needed):

```bash
inetd -c                      # re-read inetd.conf
```

### inetd Access Control (/var/adm/inetd.sec)

An HP-UX-specific file controlling which hosts may reach each inetd service; editable without restarting inetd:

```
ftp    deny  128.1.1.1
telnet deny  128.1.*.*
shell  allow 192.1.1.* 192.1.3.*
login  allow 192.1.1-3.* host1 host2
```

> `/etc/hosts.equiv` does **not** grant password-free root access on HP-UX — that requires a `.rhosts` in root's home. To disable `.rhosts` for the Berkeley r-services, append `-l` to each of their lines in `/etc/inetd.conf`.

## AutoFS (Automounter)

```bash
swlist -l product NFS
```

Files: `/etc/rc.config.d/nfsconf`, `/etc/auto_master`.

```bash
# Start/stop (script name differs by release)
/sbin/init.d/nfs.client start      # 11i v1 and v2
/sbin/init.d/autofs   start        # 11i v3
```

`/etc/auto_master` maps mount points to maps:

```
/net   -hosts          -soft    # auto soft-mount filesystems under /net
/home  /etc/auto.home            # use the /etc/auto.home map
/data  /etc/auto.data
/tools /etc/auto.tools
```

After editing `/etc/auto_master` or a direct map, apply changes:

```bash
automount
```

## CIFS (Samba)

### Server

```bash
swlist -l product CIFS-Server
```

```bash
# Enable the Samba daemons at boot
vi /etc/rc.config.d/samba          # set the control variable to 1

# Share definitions
vi /etc/opt/samba/smb.conf
/opt/samba/bin/testparm            # check smb.conf for syntax errors

# Samba password file
touch /var/opt/samba/private/smbpasswd
chmod 500 /var/opt/samba/private
chmod 600 /var/opt/samba/private/smbpasswd
/opt/samba/bin/smbpasswd -a user1  # add a UNIX user to Samba

# Start and verify
/sbin/init.d/samba start
/opt/samba/bin/smbclient -L localhost -U%
```

### Client

```bash
swlist -l product CIFS-Client
```

```bash
# Set the workgroup/domain
vi /etc/opt/cifsclient/cifsclient.cfg    # domain = "WORKGROUP"

# Start the client (no /sbin/init.d script is shipped)
/opt/cifsclient/bin/cifsclient start

# Mount a share via /etc/fstab
mkdir /homes
vi /etc/fstab                            # server:/homes /homes cifs defaults 0 0
mount -aF cifs

# Authenticate / list / logout
/opt/cifsclient/bin/cifslogin  server user1
cifslist -A
/opt/cifsclient/bin/cifslogout server
```

## NTP (xntpd)

```bash
swlist -l product InternetSrvcs        # 11i v1/v2
swlist -l product NTP                  # 11i v3
```

One-shot sync (before starting the daemon):

```bash
ntpdate -b server1 server2 server3
```

Enable and configure the daemon in `/etc/rc.config.d/netdaemons`:

```bash
vi /etc/rc.config.d/netdaemons
export NTPDATE_SERVER="192.1.1.1 192.1.1.2"
export XNTPD=1
export XNTPD_ARGS=
```

Configure servers in `/etc/ntp.conf`:

```
server 192.1.1.1
server 192.1.1.2
driftfile /etc/ntp.drift
```

```bash
/sbin/init.d/xntpd start        # (stop to restart)
ps -ef | grep xntpd
ntpq -p                         # show peers / sync status
tail /var/adm/syslog/syslog.log
```

## SSH (Secure Shell)

Install bundle **T1471AA**, then:

```bash
grep SSHD_START /etc/rc.config.d/sshd   # 1 = daemon restarts at every reboot
vi /etc/opt/ssh/sshd_config             # server config
vi /etc/opt/ssh/ssh_config              # client config

/sbin/init.d/secsh start
ps -ef | grep /opt/ssh/sbin/sshd
netstat -an | grep 22
```

Note the init.d script is named `secsh` (Secure Shell) while the control variable lives in `/etc/rc.config.d/sshd` — a naming mismatch worth remembering when you search for either one. HP-UX Secure Shell is HP's build of OpenSSH; its config file syntax matches upstream OpenSSH, so standard `sshd_config` hardening (disabling root login, restricting protocols/ciphers) applies directly.

## Serviceguard (Cluster)

```bash
cmruncl -v                              # start the entire cluster
cmhaltcl                                # stop the entire cluster
cmviewcl                                # cluster status
cmrunnode -v nodename                   # start one node
cmhaltnode -f -v nodename               # stop a node
cmgetconf -C config_name                # get current configuration
cmrunpkg -n nodename package_name       # start a package on a node
cmmodpkg -e package_name                # enable package switching
cmhaltpkg package_name                  # stop a package
```

## SMH (System Management Homepage)

SMH provides a web admin interface via a dedicated Apache daemon. Default mode is **autostart**:

- At boot, `/sbin/init.d/hpsmh` launches a lightweight `smhstartd` that listens on **http://servername:2301/**.
- On a connection, `smhstartd` launches the SSL Apache/SMH daemon and redirects the client to **https://servername:2381/**.
- A `timeoutmonitor` script stops the Apache/SMH daemon after 30 minutes of inactivity (configurable via `TIMEOUT_SMH` in `/opt/hpsmh/conf/timeout.conf`).

```bash
swlist SysMgmtWeb                       # SMH version

smhstartconfig -a on  -b off            # enable autostart mode
smhstartconfig -a off -b on             # enable start-on-boot mode
smhstartconfig                          # verify (no options = show state)

/sbin/init.d/hpsmh start                # re-run after editing /etc/rc.config.d/hpsmh
```

`smhstartconfig` just edits variables in `/etc/rc.config.d/hpsmh` (read by the `hpsmh` startup script); you can edit that file directly and re-run the script.

## Command Reference

| Task | Command |
|------|---------|
| Current run level | `who -r` |
| Change run level | `init <n>` |
| View rc.config.d var | `ch_rc -l -p VAR` |
| Set rc.config.d var | `ch_rc -a -p VAR=value [file]` |
| Remove var | `ch_rc -r -p VAR file` |
| Find a service's config | `grep -il <svc> /etc/rc.config.d/*` |
| Custom service | copy `/sbin/init.d/template`, link into `rc<N>.d` |
| Reload inetd | `inetd -c` |
| Apply automount maps | `automount` |
| NTP peers | `ntpq -p` |
| Cluster status | `cmviewcl` |
| SMH mode | `smhstartconfig` |
| Service start/stop | `/sbin/init.d/<service> start\|stop` |
| Re-read /etc/inittab | `init q` |
| Graceful halt / reboot | `shutdown -h now` / `shutdown -r now` |
| Reload inetd config | `inetd -c` |
| Test a custom script | `/sbin/init.d/<service> start` |

## Troubleshooting

**A service didn't start at boot.** Check `/etc/rc.log` first — it names the script and its exit status. Exit `1` means the script ran and failed; exit `2` means it deliberately did nothing (its control variable was `0`). If the log shows exit `2`, the fix is to set the control variable to `1` in `/etc/rc.config.d/<service>`, then start it by hand.

**I set the control variable to 1 but the daemon isn't running.** `ch_rc`/editing the config only affects the *next* boot. Start the daemon now with `/sbin/init.d/<service> start`.

**The whole `/etc/rc.config.d` file seems to be ignored.** A syntax error in a sourced config file can abort processing of that file. Sanity-check it (`sh -n /etc/rc.config.d/<file>`) and prefer `ch_rc` for edits to avoid this class of failure.

**Editing `/etc/inetd.conf` had no effect.** `inetd` reads that file only at start; run `inetd -c` to make it re-read without restarting. For access changes in `/var/adm/inetd.sec`, no reload is needed.

**The system boots to the wrong run level or loops.** Check the `initdefault` entry in `/etc/inittab`. Recover by interrupting the boot and coming up single-user, then correct the file. See the [HP-UX Boot Process](articles/hpux-boot-process.md) article for interrupting boot at the firmware prompt.

## Related Articles

- [HP-UX Boot Process](articles/hpux-boot-process.md)
- [HP-UX System Information and Initial Configuration](articles/hpux-system-information.md)
- [HP-UX Kernel Configuration](articles/hpux-kernel-configuration.md)
- [HP-UX Device Management (ioscan)](articles/hpux-device-management-ioscan.md)
- [HP-UX Software Distribution (SD-UX): Depots and swinstall](articles/hpux-software-depots-swinstall.md)
