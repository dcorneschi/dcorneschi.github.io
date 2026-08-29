# AIX HMC Cheatsheet

Command reference for the IBM Power **Hardware Management Console (HMC)** CLI — the appliance that manages Power servers, their partitions (LPARs), profiles, and virtual/physical resources. You SSH into the HMC as `hscroot` (or another HMC user) and run a restricted set of commands. Managed servers are referenced by their **managed-system** name (`-m`), and partitions by name or ID.

> SSH in as `hscroot@<hmc>`. The HMC shell is restricted (a defined command set, not a general Unix shell). Most commands take `-m <managed-system>`; list systems with `lssyscfg -r sys`. Naming, DLPAR, and profile operations here complement the VIOS side — see the [AIX VIOS Cheatsheet](articles/aix-vios-cheatsheet.md).

## What the HMC Provides

- Virtual console windows to partitions
- LPAR configuration and operation management
- Capacity on Demand (CoD) management
- Service tools
- A PC-based console (custom Linux + Java application), remotely accessible, connecting to the service processor over a private or open network

### Remote access

- Pre-HMCv7: remote GUI via **Web-based System Manager (WebSM)** client on Windows, Linux, or AIX 5L/6 workstations.
- HMCv7+: a browser over **HTTPS** is enough for the GUI, plus extensive CLI access over **SSH**.
- Version/support notes: HMCv7 manages POWER5 and POWER6 systems; an HMC cannot simultaneously manage POWER4 and POWER5 systems. POWER5 HMCs (v6) can be upgraded to support POWER6, and HMCv7 supports both POWER5 and POWER6.

## Managed Systems

```sh
lssyscfg -r sys                               # list all managed systems
lssyscfg -r sys -F name,state                 # names + state
lssyscfg -r sys -F name:state                 # colon-separated
lssyscfg -r sys -m <system> -F name,state,type_model,serial_num
lssysconn -r all                              # IP connections to service processors / bulk power controllers

# System power control
chsysstate -r sys -m <system> -o on -f <sys_profile>          # power on
chsysstate -r sys -m <system> -o onstandby -f <sys_profile>   # power on to standby
chsysstate -r sys -m <system> -o off --immed                  # power off
chsysstate -r sys -m <system> -o off --immed --restart        # restart
chsysstate -r sys -m <system> -o recover                      # recover partition data
chsysstate -r sys -m <system> -o spfailover                   # service-processor failover
chsysstate -r sysprof -m <system> -n <sys_profile> --test     # validate/activate a system profile

# Rename a managed system / change service-processor passwords
chsyscfg -r sys -m pserver -i "new_name=pserver1"
hsyspwd -t {access|admin|general} -m <system> --passwd <current> --newpassword <new>
```

### HMC appliance info (lshmc)

```sh
lshmc -V                # version information
lshmc -v                # vital product data (VPD)
lshmc -b                # BIOS level
lshmc -n                # network settings
lshmc -r                # remote access settings
lshmc -l                # current locale
lshmc -L                # all supported locales
```

## Partitions (LPARs)

```sh
# List LPARs on a system (name + id + state + type)
lssyscfg -r lpar -m <system> -F name,lpar_id,state,lpar_env

# Details of one LPAR
lssyscfg -r lpar -m <system> --filter "lpar_names=<lpar>"

# Runtime state / reference codes (LED codes) for LPARs
lsrefcode -r lpar -m <system> --filter "lpar_names=<lpar>" -F lpar_name,refcode
```

`lpar_env` distinguishes `aixlinux` (AIX/Linux client) from `vioserver` (VIOS).

## Power Control (chsysstate)

```sh
# Power on an LPAR using a named profile
chsysstate -r lpar -m <system> -o on -n <lpar> -f <profile>

# Power on in SMS / OpenFirmware for boot troubleshooting
chsysstate -r lpar -m <system> -o on -n <lpar> -f <profile> -b sms
chsysstate -r lpar -m <system> -o on -n <lpar> -f <profile> -b of

# Soft (OS) shutdown vs immediate
chsysstate -r lpar -m <system> -o osshutdown -n <lpar>
chsysstate -r lpar -m <system> -o shutdown --immed -n <lpar>

# Restart an LPAR immediately
chsysstate -r lpar -m <system> -o shutdown --immed --restart -n <lpar>

# Power the whole managed system on/off
chsysstate -r sys -m <system> -o on
chsysstate -r sys -m <system> -o off
```

| `-o` operation | Effect |
|----------------|--------|
| `on` | Activate an LPAR (needs `-f <profile>`) |
| `osshutdown` | Graceful OS shutdown |
| `shutdown` | Partition shutdown (`--immed` for hard) |
| `--restart` | Reboot instead of stay-down |
| `dumprestart` | Force a dump then restart |

## Virtual Terminal / Console

```sh
vtmenu                              # interactive menu to pick a system + LPAR console
mkvterm -m <system> -p <lpar>      # open a console to an LPAR
mkvterm -m <system> --id <lpar_id>
rmvterm -m <system> -p <lpar>      # force-close an open console session
# Type  ~.  to exit a console session
```

## DLPAR: Dynamic Resource Changes (chhwres)

Add or remove CPU, memory, and I/O from a running LPAR without a reboot.

```sh
# Show current hardware resource assignment
lshwres -r proc  -m <system> --level lpar
lshwres -r mem   -m <system> --level lpar
lshwres -r virtualio -m <system> --rsubtype scsi   # virtual SCSI adapters
lshwres -r virtualio -m <system> --rsubtype fc     # virtual FC adapters

# CPU — add/remove processing units and virtual processors
chhwres -r proc -m <system> -o a -p <lpar> --procunits 0.5
chhwres -r proc -m <system> -o r -p <lpar> --procs 1

# Memory — add/remove (in MB)
chhwres -r mem  -m <system> -o a -p <lpar> -q 2048
chhwres -r mem  -m <system> -o r -p <lpar> -q 1024

# Virtual adapters — add a virtual SCSI/FC adapter dynamically
chhwres -r virtualio -m <system> -o a -p <lpar> --rsubtype scsi \
  -s <slot> -a "adapter_type=client,remote_lpar_name=<vios>,remote_slot_num=<n>"

# Move resources between LPARs
chhwres -r mem  -m <system> -o m -p <lpar_a> -t <lpar_b> -q 1024        # move memory
chhwres -r proc -m <system> -o m -p <lpar_a> -t <lpar_b> --procs 1      # move a dedicated CPU
chhwres -r proc -m <system> -o m -p <lpar_a> -t <lpar_b> --procunits 0.5

# Dedicated CPUs (add/remove)
chhwres -r proc -m <system> -o a -p <lpar> --procs 1
chhwres -r proc -m <system> -o r -p <lpar> --procs 1

# Physical I/O slot add / move (by DRC index)
chhwres -r io -m <system> --rsubtype slot -o a -p <lpar> -l 21030003 -w 0
chhwres -r io -m <system> -o m -p <src_lpar> -t <dst_lpar> -l <drc_index>

# The -w flag sets a DLPAR timeout (0 = wait indefinitely); some builds use -m <system>
chhwres -r mem -o a -p LPAR1 -m <system> -q 16 -w 0
```

| `-o` | Operation |
|------|-----------|
| `a` | Add resource |
| `r` | Remove resource |
| `m` | Move resource |
| `s` | Set (memory/proc levels) |

### Restore resources to profile values (rsthwres)

After DLPAR changes, `rsthwres` re-aligns a running LPAR's resources with its profile.

```sh
rsthwres -r mem  -m <system> -p <lpar>   # restore memory on one LPAR
rsthwres -r mem  -m <system>             # ... for all partitions
rsthwres -r proc -m <system> -p <lpar>   # restore processing resources
rsthwres -r io   -m <system> -p <lpar>   # restore physical I/O slots
```

## Profiles (mksyscfg / chsyscfg)

```sh
# List profiles for an LPAR
lssyscfg -r prof -m <system> --filter "lpar_names=<lpar>"

# Create a partition
mksyscfg -r lpar -m <system> \
  -i "name=<lpar>,profile_name=default,lpar_env=aixlinux,min_mem=1024,desired_mem=4096,max_mem=8192,proc_mode=shared,min_proc_units=0.2,desired_proc_units=1.0,max_proc_units=2.0,min_procs=1,desired_procs=2,max_procs=4"

# Create an LPAR/profile from a file of attributes
mksyscfg -r lpar -m <system> -f /tmp/profiles.txt

# Create/change a profile
mksyscfg -r prof -m <system> -i "name=newprof,lpar_name=<lpar>,..."
chsyscfg -r prof -m <system> -i "name=default,lpar_name=<lpar>,desired_mem=8192"

# Remove a partition or profile
rmsyscfg -r lpar -m <system> -n <lpar>
rmsyscfg -r prof -m <system> -n <profile> --filter "lpar_names=<lpar>"
```

> `-i` passes attributes inline as comma-separated `key=value`; use `-f <file>` to read them from a file. `save_sysprof`/`restore` (`bkprofdata`/`rstprofdata`) back up and restore the profile data.

## Reference Codes (LED)

```sh
lsrefcode -r sys -m <system>                          # current system reference code
lsrefcode -r sys -m <system> -n 10                    # last 10 system codes
lsrefcode -r lpar -m <system> -F lpar_name,time_stamp,refcode   # per-LPAR codes
lsrefcode -r lpar -m <system> --filter "lpar_names=<lpar>" -F lpar_name:refcode
lsrefcode -r lpar -m <system> -n 25 --filter "\"lpar_names=lpar-a,lpar-b\""  # last 25 for two LPARs
```

## Service Events

```sh
lssvcevents -t console                 # HMC console events
lssvcevents -t console -d 60           # console events in the past 60 days
lssvcevents -t hardware -m <system>    # hardware serviceable events
lssvcevents -t hardware -d 0           # serviceable events that occurred today

# DLPAR operation history (mined from console events)
lssvcevents -t console -d 300 | grep -i mem | grep DLPAR         # memory DLPAR ops, last 300 days
lssvcevents -t console -d 300 | grep -i processor | grep DLPAR   # processor DLPAR ops
```

## Frame / FSP

```sh
lshwres -r proc -m <system> --level sys    # system-level processor inventory
lshwres -r proc -m <system> --level pool   # shared processor pool
lshwinfo -r sys -m <system>                # (frame-managed) environmental info
chsysstate -m <system> -r sys -o rebuild   # rebuild the HMC's view after an FSP change
```

## HMC Users and Roles

```sh
lshmcusr                                   # list all HMC users
lshmcusr -F name:resourcerole              # user names + managed-resource roles

mkhmcusr -u <user> -a <role> -d "<description>" --passwd <pw> -M <expire_days>
rmhmcusr -u <user>                         # remove a user
chhmcusr -u <user> -t passwd -v <new_pw>   # change a user's password
chhmcusr -r <user> -t taskrole -v hmcoperator   # change a user's task role
```

Built-in task roles: `hmcsuperadmin`, `hmcoperator`, `hmcviewer`, `hmcpe`, `hmcservicerep` (plus any user-defined role).

### Custom access/task roles (lsaccfg / mkaccfg)

```sh
lsaccfg -t resource                        # all managed resource objects
lsaccfg -t resourcerole                    # all managed resource roles

mkaccfg -t resourcerole -f /tmp/fil1       # create a resource role from a file
mkaccfg -t taskrole -i "name=tr1,parent=hmcsuperadmin,\"resources=cec:chcod+lscod+lshwres,lpar:chssyscfg+lssyscfg+mksyscfg\""
chaccfg -t taskrole -i "name=tr1,\"resources=cec:chhwres+chsysstate,lpar:chssyscfg+chled+chhwres\""
rmaccfg -t taskrole -n tr1
```

## HMC Network Configuration (chhmc)

```sh
chhmc -c ssh   -s enable                   # enable SSH (-s disable to turn off)
chhmc -c xntp  -s enable                   # enable NTP
chhmc -c syslog -s add -a <IP_or_host>     # add a syslog target
chhmc -c xntp  -s add -a <IP_or_host>      # add an NTP server
chhmc -c netboot -s enable                 # network as a startup device
chhmc -s ssh -s add -a <IP_Addr>           # permit an IP to use a service (e.g. ssh)
chhmc -c network -s add -ns <DNS> -ds <domain_suffix>   # add DNS server / domain suffix
chhmc -c network -s add -g <gateway_ip>    # set the gateway
chhmc -c network -s modify -i <interface>  # change a specific interface
chhmc -c network -s modify -h <hostname> -d <domain> -g <gateway>
chhmc -c locale  -s modify -l <locale>     # change the HMC locale
```

## Backup and Restore

```sh
# HMC console/critical data
bkconsdata -r dvd                          # back up to DVD (/media/cdrom/backuphdr.tgz)
bkconsdata -r ftp -h <server> -u <user> --passwd <pw>
bkconsdata -r nfs -n <server> -l <mount_point>
lsmediadev                                 # list storage media devices
cat /var/hsc/log/backuphdr.log             # bkconsdata log

# Managed-system profile data (files under /var/hsc/profiles)
bkprofdata -m <system> -f backup_profiles_<server>.prof --force
rstprofdata -m <system> -l <restore_type> -f <file>
```

`rstprofdata` restore types: full restore from the backup; merge (backup priority); merge (current priority); or initialize (delete all partition and system profiles).

## HMC Updates

```sh
# GUI: boot from the HMC Recovery CD and upgrade (HMC_Recovery_* ISO files)

# Command line — apply a fix ISO from an FTP server (-r reboots after)
updhmc -t s -h <ftp_server> -u <user> -p <passwd> -f /software/server/hmc/fixes/MH01195.iso -r
updhmc -t s -h ftp.software.ibm.com -u anonymous -p ftp -f /software/server/hmc/fixes/MH01400.iso -r
getupgfiles -h <server> -u <user> -d <dir>    # fetch upgrade files

hmcshutdown -r -t now                      # reboot the HMC
```

## Quick Reference

| Task | Command |
|------|---------|
| List managed systems | `lssyscfg -r sys -F name,state` |
| List LPARs | `lssyscfg -r lpar -m <system> -F name,lpar_id,state` |
| Power on an LPAR | `chsysstate -r lpar -m <system> -o on -n <lpar> -f <profile>` |
| Boot to SMS | `chsysstate ... -o on ... -b sms` |
| OS shutdown | `chsysstate -r lpar -m <system> -o osshutdown -n <lpar>` |
| Open a console | `mkvterm -m <system> -p <lpar>` (`~.` to exit) |
| Close a console | `rmvterm -m <system> -p <lpar>` |
| Add CPU (DLPAR) | `chhwres -r proc -m <system> -o a -p <lpar> --procunits 0.5` |
| Add memory (DLPAR) | `chhwres -r mem -m <system> -o a -p <lpar> -q 2048` |
| Show resources | `lshwres -r proc -m <system> --level lpar` |
| Create an LPAR | `mksyscfg -r lpar -m <system> -i "..."` |
| Reference codes | `lsrefcode -r lpar -m <system> --filter "lpar_names=<lpar>"` |
| Service events | `lssvcevents -t hardware -m <system>` |
| HMC version | `lshmc -V` |
| Back up HMC data | `bkconsdata -r <target>` |

## RMC Connection (LPAR ↔ HMC)

DLPAR and dynamic operations depend on a healthy **RMC** (Resource Monitoring and Control) connection between each LPAR and the HMC, provided by RSCT.

### Check the connection

```sh
# On the LPAR — is the management server / control point known?
lsrsrc IBM.ManagementServer      # AIX 5.3 / 6.1
lsrsrc IBM.MCP                    # AIX 7.1
lssrc -g rsct_rm                  # status of the RMC resource managers (IBM.DRM etc.)

# On the LPAR — RMC domain/heartbeat status toward the HMC(s)
/usr/sbin/rsct/bin/rmcdomainstatus -s ctrmc
```

`rmcdomainstatus` output columns: `I` = partition is Up (active RMC heartbeat); `A` = no messages queued to the node; then the node ID, an internal node number, and the HMC IP.

```sh
# On the HMC — which LPARs have DLPAR capability?
lspartition -dlpar
```

In `lspartition -dlpar`, a **zero** `DCaps:` value means no DLPAR between the HMC and that LPAR; a **non-zero** value means DLPAR is available.

### Reset / recreate the connection (on the LPAR)

```sh
# Reset RMC
/usr/sbin/rsct/bin/rmcctrl -z        # stop
/usr/sbin/rsct/bin/rmcctrl -A        # add + start
/usr/sbin/rsct/bin/rmcctrl -p        # enable remote client connections

# Full recreate (last resort)
/usr/sbin/rsct/install/bin/recfgct

# Subsystem management
rmcctrl -a        # add the RMC subsystem (adds it + an /etc/inittab entry)
rmcctrl -s        # start
rmcctrl -k        # stop
rmcctrl -d        # delete (removes it + the /etc/inittab entry)
```

## Related

- [AIX VIOS Cheatsheet](articles/aix-vios-cheatsheet.md) — the VIOS side (create the vhost/vfchost adapters the HMC maps to clients)
- [AIX NIM Cheatsheet](articles/aix-nim-cheatsheet.md) — network installs to LPARs you activate from the HMC
- [AIX Boot and Init Cheatsheet](articles/aix-boot-init-cheatsheet.md) — bootlist/bosboot inside the LPAR once it powers on
