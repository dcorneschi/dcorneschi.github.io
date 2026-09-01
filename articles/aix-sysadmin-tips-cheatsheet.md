# AIX System Administration Tips Cheatsheet

A grab-bag of useful IBM AIX system administration commands — device management (`mkdev`/`rmdev`), software media and TOC (`gencopy`/`inutoc`), filesystem maintenance and superblock recovery (`defragfs`/`fsck`/`lquerypv`/`dd`), CD/DVD/ISO mounting, terminal and console control, process inspection (`procfiles`/`procflags`/`procstack`), CPU/kernel info, time/NTP, archiving, shutdown, and more.

> Many of these require `root` and some are genuinely destructive — the superblock `dd` recovery, `mkboot -c`, and `rmdev -d` can make a disk or system unbootable if run against the wrong device. Double-check device names and take a backup before running recovery commands.

## Devices

```sh
# Make a defined device available (configure it)
mkdev -l rmt0
mkdev -l hdisk2

# Move an available device back to defined (unconfigure)
rmdev -l rmt0
rmdev -l hdisk2

# Permanently remove a device from the ODM
rmdev -dl rmt0

# Stop the device (S) and unconfigure its children (R)
rmdev -l rmt0 -SR

# Bus resource attributes of a device
lsresource -l rmt0

# List valid values for a device attribute
lsattr -Rl fcs0 -a init_link

# List devices in a device class / all classes
listdgrp disk
getdgrp

# Vital Product Data (VPD) for FRUs
lsvpd

# Detect newly added devices
lsdev -C > lsd.pre; cfgmgr; lsdev -C > lsd.post; diff lsd.pre lsd.post | grep -v '^>'
```

## Software Media and TOC

```sh
# Copy all software from CD to disk for later installation
gencopy -d /dev/cd0 -t /usr/sys/inst.images all

# List install packages on the media
gencopy -L -d /dev/cd0

# Copy specific images (I: installp, R: RPM) from CD
gencopy -d /dev/cd0 I:bos.perf R:cdrecord

# Create/refresh the .toc for an image directory
inutoc                    # /usr/sys/inst.images
inutoc .                  # current directory
inutoc /tmp/images        # a specific directory
```

## Filesystems and Superblocks

```sh
# Reorganize logical volumes within a VG
reorgvg vg3 lv04 lv07

# Report file system fragmentation state
defragfs -q /             # reports the current state
defragfs -r /             # current state + state after a defrag
defragfs -s /             # reports fragmentation (passes through metadata)

# List supported filesystem types
cat /etc/vfs

# Check whether the ODM and VGDA are in sync
getlvodm -u <vg>
```

### Superblock locations and recovery

```sh
# JFS2 primary superblock at 32 KB (0x8000) into the LV
lquerypv -h /dev/fslv05 8000 100

# JFS2 secondary superblock at 60 KB (0xF000) into the LV
lquerypv -h /dev/fslv05 F000 100

# Recover superblock from the backup copy — JFS
dd count=1 bs=4k skip=31 seek=1 if=/dev/lv00 of=/dev/lv00

# Recover superblock from the backup copy — JFS2
dd count=1 bs=4k skip=15 seek=8 if=/dev/lv00 of=/dev/lv00
```

> The `dd` superblock recovery overwrites the primary superblock with the secondary copy. Run it only on an unmounted filesystem and confirm the device name — a mistake here destroys the filesystem.

### fsck logs (JFS2)

```sh
# View the fsck log for a device
/sbin/helpers/jfs2/fscklog /dev/oracle

# Print the prior log
/sbin/helpers/jfs2/fscklog -p /dev/oracle
```

## CD / DVD / ISO

```sh
# Mount a CD (CDRFS, read-only)
mount -V cdrfs -o ro /dev/cd0 /cdrom
mount -rv cdrfs /dev/cd0 /mnt
mount -v udfs -o ro /dev/cd0 /mnt        # UDFS (DVD)

# Mount an ISO image via loopback
loopmount -i dvd_aix.iso -o "-V cdrfs -o ro" -m /mnt

# Start the cdromd automount daemon (config: /etc/cdromd.conf)
startsrc -s cdromd

# cdromd helpers
cdmount                  # mount a CD
cdeject cd0              # eject
cdumount cd0             # unmount
cdcheck -m cd0           # is a CD mounted?
```

## Terminal and Console

```sh
# Show this terminal's tty/pty number
tty

# Set a tty's terminal type
chdev -l tty1 -a term=vt100

# Terminal geometry / type
termdef -c               # columns
termdef -l               # lines
termdef -t               # terminal type

# Redirect the system console to a file
chcons /tmp/console.out

# Temporarily change console output
swcons /cons.out

# List displays / default display
lsdisp
```

## Fonts and Keyboard

```sh
# Fonts
lsfont                   # list available fonts
chfont                   # change the default boot font
mkfont                   # add a font
mkfontdir                # build fonts.dir from a directory

# Keyboard maps
lskbd                    # list available maps
chkbd                    # change the default map
```

## Processes and Loaded Objects

```sh
# Files referred to by a process's file descriptors
procfiles -n <pid>

# A process's tracing flags / current stack
procflags <pid>
procstack <pid>

# Loaded objects per running process
genld -l

# Kernel extensions currently loaded
genkex

# Free unused kernel/library memory modules
slibclean
```

## CPU, Kernel, and Hardware Info

```sh
# Is SMT enabled?
smtctl

# Number of active virtual processors
echo vpm | kdb

# CPU architecture
mach

# CPU frequency / current speed
pmcycles -dm
pmcycles -dMm

# Power Status Register
machstat

# Firmware level of the system
invscout

# Queue depth of a disk (hex) via kdb
echo scsidisk hdisk6 | kdb | grep queue_depth
```

## Files, Programs, and Archives

```sh
# Full path of a program
whence program
whereis program

# Determine a file's type
file <path>

# Contents of an executable (loader/section info)
dump -nTv binary

# Compress a file
compress -c file.txt > file.Z

# Create / extract a gzip tarball
tar -cvf - ./bin/* | gzip -c - > bin.tgz
gzip -dc bin.tgz | tar -xvf -

# Split a tarball into 500 MB pieces and reassemble
tar -cvf - /some/directory/path/ | gzip -c | split -b 500m - file.tgz.
cat file.tgz.a* | gzip -dc - | tar -xvf -

# Copy files preserving permissions/links recursively
cp -pPr /usr/sap/* /mnt/sap
cp -Rph . /oracle/P4S_new
```

## Locale, Time, and NTP

```sh
# Change timezone / language in /etc/environment
chtz 'CST-6CDT1'
chlang en_US

# Sync the clock with an NTP server
setclock <ntp_server>

# NTP daemon status
xntpdc -l                # server list
xntpdc -s                # peers and their state
```

## Users, Limits, and Auth

```sh
# Show a user's ulimit settings
su - bb "-c ulimit -a"

# Remove the file-size limit for the current shell
ulimit -f unlimited

# Create a user with explicit attributes
mkuser home='/ctsdata/DP2/ARCHIV' rlogin='false' maxage='0' \
  maxexpired='0' shell='/usr/bin/ksh' gecos='FTP user' saparc

# List authentication methods
lsauthent

# Change your encryption key / NIS key
chkey
```

## Cron

```sh
# Check crontab submission time
crontab -v

# Submit a crontab file
crontab mycronfile
```

## Diagnostics and Support

```sh
# Create a support snapshot (default /tmp/ibmsupt)
snap -ad <directory>

# Remove old snap files (also clears /tmp/ibmsupt)
snap -r
```

## Boot and Shutdown

```sh
# Clear the boot record on a PV
mkboot -c -d /dev/hdisk0

# Fast shutdown and reboot
shutdown -Fr

# Down to maintenance (single-user) after 1 minute
# (root password, then telinit 2 to return to multiuser)
shutdown -m +1

# Log shutdown output to /etc/shutdown.log
shutdown -l

# Flush buffers and reboot
sync; sync; sync; reboot
```

## Quick Reference

| Task | Command |
|------|---------|
| Make device available | `mkdev -l <dev>` |
| Remove device (permanent) | `rmdev -dl <dev>` |
| Copy media to disk | `gencopy -d /dev/cd0 -t /usr/sys/inst.images all` |
| Build install TOC | `inutoc <dir>` |
| Query fragmentation | `defragfs -q /` |
| Read a superblock | `lquerypv -h /dev/<lv> 8000 100` |
| Recover JFS2 superblock | `dd count=1 bs=4k skip=15 seek=8 if=/dev/<lv> of=/dev/<lv>` |
| Mount ISO | `loopmount -i <iso> -o "-V cdrfs -o ro" -m /mnt` |
| SMT status | `smtctl` |
| Active virtual CPUs | `echo vpm \| kdb` |
| CPU frequency | `pmcycles -dm` |
| Files open by PID | `procfiles -n <pid>` |
| Loaded kernel extensions | `genkex` |
| Free unused modules | `slibclean` |
| Firmware level | `invscout` |
| NTP peer state | `xntpdc -s` |
| Support snapshot | `snap -ad <dir>` |
| Clear boot record | `mkboot -c -d /dev/hdisk0` |
| Fast reboot | `shutdown -Fr` |
