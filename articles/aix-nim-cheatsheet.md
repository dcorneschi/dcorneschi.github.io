# AIX NIM Cheatsheet

Reference for IBM AIX Network Installation Management (NIM) — the resources involved (lpp_source, SPOT, mksysb, and helpers), the network ports and files it uses, master and client commands, standing up a NIM master, and common install/migration tasks including `nimadm`.

> NIM lets a **master** install and maintain AIX on **standalone clients** over the network. Commands run as `root` on the master (`nim`, `lsnim`, `nimconfig`) or on the client (`nimclient`, `niminit`). For local backup/boot/LVM context, see the [AIX Backup and Recovery](articles/aix-backup-recovery-cheatsheet.md), [Boot and Init](articles/aix-boot-init-cheatsheet.md), and [LVM](articles/aix-lvm-cheatsheet.md) cheatsheets.

## Level Pairing (Important)

Keep the **lpp_source** and **SPOT** at matching versions/levels where possible:

- You may use a SPOT at a **lower** level than the lpp_source for a new install or upgrade/migration, but always check software levels first.
- **Never** use an lpp_source at a lower level than the SPOT.
- For a mksysb install, the SPOT must be at the **same or higher** level than the mksysb. After the mksysb install, the system is brought up to the SPOT's level using the (also-allocated) lpp_source.

## Resources

| Resource | What it is |
|----------|------------|
| **lpp_source** | Directory of AIX software install images (BFF images from the install CD/DVD). One per OS version; keep 32-bit and 64-bit sets separate. |
| **SPOT** | Shared Product Object Tree — a directory built from an lpp_source, providing boot images and install scripts (like the base install CD). Multiple SPOTs may be needed for different ML/versions. |
| **mksysb** | A system backup image; the master can install a client from an lpp_source or from a mksysb, then run a customization script. |

Helper resources:

- **scripts** — run during BOS install to tailor the OS (security, third-party software, paging/dump space).
- **bosinst_data** — a *file* with answers for an unattended install (default disk, install type, etc.).
- **image_data** — a *file* with OS image info (filesystems, mirroring, etc.).
- **installp_bundles** — files listing extra software to install after AIX (e.g. a DB2 bundle vs a web-server bundle).

## Files

| File | Purpose |
|------|---------|
| `/etc/bootptab` | BOOTP client entries |
| `/tftpboot/<client>.info` | Per-client boot info (e.g. `/tftpboot/aiiics02_priv.info`) |
| `/var/adm/ras/nimsh.log` | nimsh log — check here for client connection problems |
| `/var/adm/ras/nimlog` | General NIM log; read failed operations with `alog -f /var/adm/ras/nimlog -o` |

## Ports

| Service | Port(s) |
|---------|---------|
| nimsh | 3901–3902 |
| rsh* | 513–1023** |
| rlogin* | 513 |
| shell* | 514 |
| bootp | 67–68 |
| tftp | 69 and 32768–65535 |
| nfs | 2049 |
| mountd | 32768–65535 (or chosen) |
| portmapper | 111 |
| NIM | 1058–1059 |

(*) Required for rsh operation.
(**) For each NIM rsh communication, leave five ports open starting at 1023 and decreasing.

## Master Commands

```sh
lsnim                              # list all NIM objects
lsnim -l bosinstdata-61            # detailed info about an object
lsnim -p                           # list NIM object classes
lsnim -c machines                  # all machines
lsnim -c networks                  # all networks
lsnim -t mksysb                    # all mksysb resources
lsnim -t lpp_source                # all lpp_source resources
lsnim -O aiiics02_priv             # valid operations for an object
lsnim -a lpp_source|spot|mksysb    # resources allocated to clients

nim -o remove <machine>                              # delete a machine
nim -Fo reset <object>                               # reset object state
nim -Fo deallocate -a subclass=all host-01           # deallocate everything for a client
nim -o change -a <attribute>="<value>" <object>      # change an attribute
nimconfig -r                                         # rebuild /etc/niminfo on the master

nim -o maint_boot -a spot=spot_7100_00_01 aiiics02_priv   # maintenance-mode boot
nim -o showlog -a log_type=niminst aiiics02_priv          # review the installp log
nim -o allocate   -a lpp_source=spot_7100_00_01 aiiics02_priv
nim -o deallocate -a spot=spot_7100_00_01 aiiics02_priv
nim -Fo deallocate -a subclass=all aiiics02_priv          # deallocate all resources
nim -o showres lpp_61-04-03 | grep nim                    # packages in an lpp_source
nim -o check spot_5300_05                                 # check a SPOT
nim -o lslpp aiiics01                                     # test nimsh connectivity
nim -Fo change -a cpuid=<id> ClientName                   # change client CPUID
nim -o define -t mksysb -a server=master -a location=<file>   # define a mksysb resource
```

## Client Commands

```sh
nimclient -l -L aiiics02_priv                    # resources available to the client
nimclient -l -c resources aiiics02_priv          # resources allocated to the client
nimclient -o allocate -a lpp_source=lpp_7100_00_01
nimclient -o allocate -a spot=spot_7100_00_01
nimclient -Fo deallocate -a spot=spot_7100_00_01
nimclient -o maint_boot -a spot=spot_7100_00_01  # maintenance-mode boot from the client

# Update the client from the NIM master
nimclient -o cust -a lpp_source=lpp_7100_00_01 -a fixes=update_all
nimclient -o cust -a lpp_source=$OSLPPSOURCE -a fixes=update_all -a installp_flags="acgNXY"

# Recreate /etc/niminfo on the client
rm /etc/niminfo
niminit -v -a master=aiiics01_priv -a name=aiiics02_priv -a connect=nimsh
```

## Installing a NIM Master

### 1. Prepare the LPAR (move the CD/DVD device)

```sh
lsdev -Cl cd0 -F parent        # find the parent (e.g. ide0)
lsslot -c slot                 # map slots to adapters
rmdev -dl pci2 -R              # remove the adapter chain before moving it
```

Then in the HMC: **Dynamic Logical Partitioning → Physical Adapters → Move or Remove**, select the "Other Mass Storage Controller" adapter and move it to the target LPAR. Back on the LPAR:

```sh
cfgmgr
lsdev -Cl cd0
lsparent -Cl cd0
lscfg -vpl sata0
```

### 2. Install NIM packages

```sh
mount -V cdrfs -o ro /dev/cd0 /dvdrom
cd /dvdrom
installp -aXYg -d . bos.sysmgt.nim.master
installp -aXYg -d . bos.sysmgt.nim.spot
```

### 3. Configure NIM

```sh
smitty nimconfig
# or non-interactively:
nimconfig -a netname=net_155_17_18_0 -a pif_name=en0 \
  -a netboot_kernel=mp -a cable_type=tp -a client_reg=no

lsnim               # should now show master, boot, nim_script, and the network
lsnim -l master
```

### 4. Enable bootpd and tftpd

Uncomment in `/etc/inetd.conf`:

```
bootps dgram udp   wait  root   /usr/sbin/bootpd bootpd /etc/bootptab
tftp   dgram udp6  SRC   nobody /usr/sbin/tftpd  tftpd -n
```

```sh
refresh -s inetd
lssrc -ls inetd
```

### 5. Test tftp

```sh
touch /tftpboot/test2
tftp -o - 0 /tftpboot/ms-lpar01.company.local.info
```

### 6. Export the mksysb location

Add to `/etc/exports`:

```
/nim/mksysb -sec=sys:krb5p:krb5i:krb5:dh,rw
```

```sh
chmod 777 /nim/mksysb
```

## Resource Tasks

### Define an lpp_source (from CD/DVD)

```sh
nim -o define -t lpp_source -a server=master \
  -a location=/nim/lpp_source/lpp_7100_00_01 -a source=/dev/cd0 lpp_7100_00_01
# or: smitty nim_mkres  ->  choose lpp_source
```

### Update an lpp_source

```sh
cd /nim/lpp_source
cp -pr lpp_7100_00_01 lpp_7100_01_03
nim -o define -t lpp_source -a server=master \
  -a location=/nim/lpp_source/lpp_7100_01_03 lpp_7100_01_03
inutoc /kit/7100-01-03-1207
nim -o update -a packages=all -a source=/kit/7100-01-03-1207 lpp_7100_01_03
nim -o check lpp_7100_01_03
nim -o lppmgr -a lppmgr_flags=-rbux lpp_7100_01_03   # prune duplicates/superseded
nim -o check lpp_7100_01_03
```

### Create a SPOT

```sh
nim -o define -t spot -a server=master -a location=/nim/spot/ \
  -a source=lpp_7100_00_01 -a installp_flags=-aQg spot_7100_00_01
# or: smitty nim_mkres  ->  choose spot
```

### Update a SPOT

```sh
nim -o define -t spot -a server=master -a location=/nim/spot/ \
  -a source=lpp_7100_01_03 -a installp_flags=-aQg spot_7100_01_03
```

### Define a client

```sh
smitty nim_mkmac
# or:
nim -o define -t standalone \
  -a if1="net_192_168_5_0 aiiics02.example.net 0 ent2" aiiics02
```

### Create a mksysb image of a client

```sh
smitty nim_mkres   # -> mksysb
nim -o define -t mksysb -a server=master -a source=aiiics02_priv \
  -a mk_image=yes -a location=/nim/mksysb/mksysb-aiiics02 mksysb-aiiics02
lsmksysb -l -f mksysb-aiiics02
```

## Installation Tasks

### Install the OS (from lpp_source + SPOT)

```sh
nim -o bos_inst -a spot=spot_7100_00_01 -a lpp_source=lpp_7100_00_01 \
  -a bosinst_data=bosinst_data_7100_00_01 -a no_client_boot=yes aiiics02_priv
# no_client_boot=yes -> NIM won't reboot the LPAR over rsh
```

### Install from a mksysb

```sh
nim -o bos_inst -a source=mksysb -a spot=spot_7100_00_01 \
  -a lpp_source=lpp_7100_00_01 -a mksysb=mksysb-aiiics02 \
  -a no_client_boot=yes aiiics02_priv
```

### Handy operations

```sh
nim_update_all -l -s -d -u -B                    # update SPOT + lpp_source to latest
smit nim_mkmac                                   # add machines
smit nim_bosinst                                 # BOS install on a machine
nim -o maint_boot -a spot=<spot> <client>        # provide a maintenance boot image
nim -o showlog -a log_type=boot LPAR2            # watch install/first-boot progress
loopmount -i aixv7-base.iso -m /aix -o "-V cdrfs -o ro"   # mount an ISO
/usr/lpp/bos.sysmgt/nim/methods/m_chattr -a net_addr=192.168.5.0 net_192_168_5_0
nim -o unconfig master                           # unconfigure the NIM master
```

### Install software on a client (from the master)

```sh
nim -o cust -a lpp_source=lpp_Generic610_0-current \
  -a installp_flags="agXY" -a filesets="erac.oracle.utilities.rte" server
```

### Install a fileset from the client

```sh
nimclient -o cust -a filesets='erac.asm_utils.rte' \
  -a lpp_source=lpp_Generic610_0-current -a installp_flags="acgNXY"
```

## Migrating with nimadm

`nimadm` migrates a client to a new AIX level on an **alternate** root volume group, minimizing downtime (the client keeps running until you reboot onto the migrated disk).

```sh
# AIX 5.3 -> 7.1
nimadm -j nimvadmg -c lparaix53 -s spotaix710104 -l lpp_sourceaix710104 -d "hdisk5" -Y

# AIX 6.1 -> 7.1
nimadm -j nimadmvg -c lparaix61 -s spotaix710104 -l lpp_sourceaix710104 -d "hdisk5" -Y

# With a customization script and bundle, logging to a file
nimadm -c tstapl26 -s spot_AIX71-current -l lpp_AIX711_0-current -d "hdisk1" -Y \
  -z script_migration_AIX711_0-current \
  -b bnd_BaseOS_AIX711_0-current 2>&1 | tee -a /tmp/tstapl26.migrate &
```

| Flag | Meaning |
|------|---------|
| `-j` | VG on the master used for the migration |
| `-c` | client name |
| `-s` | SPOT name |
| `-l` | lpp_source name |
| `-d` | hdisk for the alternate root VG (`altinst_rootvg`) |
| `-Y` | accept software license agreements |

A `post_migration` script (found under `/usr/lpp/bos` after migration) verifies the result. All migration activity is logged on the master under `/var/adm/ras/alt_mig`.

### Post-migration tunable checks

```sh
tuncheck -p -f /etc/tunables/nextboot     # validate current tunables
vmo -p -d maxperm%                        # reset one tunable to default
vmo -r -D                                 # reset all vmo tunables (needs bosboot + reboot)
tundefault -r                             # run ioo/vmo/schedo/no/nfso/raso with -D
tuncheck -p -f /etc/tunables/nextboot     # re-verify, expect no errors
```

### Back out (boot the original rootvg)

```sh
lspv | grep old_rootvg
bootlist -m normal -o hdisk0 blv=hd5
shutdown -Fr
```

## Related

- [AIX Backup and Recovery Cheatsheet](articles/aix-backup-recovery-cheatsheet.md) — mksysb, savevg/restvg, alt_disk_install
- [AIX Boot and Init Cheatsheet](articles/aix-boot-init-cheatsheet.md) — bootlist, bosboot, inittab
- [AIX LVM Cheatsheet](articles/aix-lvm-cheatsheet.md) — volume groups, logical volumes, mirroring
