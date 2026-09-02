# Oracle Solaris 11 Installation Methods

Oracle Solaris 11 can be installed **interactively** (a person answers prompts) or via **automated installation** (hands-off, template-driven). This guide summarizes the available installers, when to use each, the media per platform, and where to find the installation log.

## Interactive vs Automated

| Approach | How it works | Best for |
|----------|--------------|----------|
| Interactive | You answer prompts at the console/GUI | One-off installs, workstations, learning |
| Automated (AI) | Network install driven by a manifest, no prompts | Fleets, repeatable/hands-off provisioning |

## Interactive Installation

Interactive installs come in two forms:

### Live Media (x86 only)

- A bootable GNOME desktop environment for **x86-based systems**.
- Boot the Live Media, evaluate the environment, then launch the installer from the desktop.
- Good for workstations and trying Solaris before committing.
- **Not available for SPARC.**

### Text Installer (x86 and SPARC)

- A text-based (curses) installer that runs on **both x86 and SPARC** machines.
- Menu-driven prompts for disk, network, user, and time zone — no desktop required.
- The typical choice for servers and headless systems.

| Installer | Platform | Interface |
|-----------|----------|-----------|
| Live Media | x86 only | Graphical (GNOME desktop) |
| Text installer | x86 and SPARC | Text/curses |

## Automated Installation (AI)

For installing many systems consistently, Solaris 11 uses the **Automated Installer (AI)** — the successor to Solaris 10 JumpStart:

- An **AI install server** hosts install services and boots clients over the network (PXE on x86, WAN boot/network on SPARC).
- Client configuration comes from an **AI manifest** (XML) describing disk layout, packages, and publishers.
- **System configuration profiles** (SMF `sysconfig` profiles) supply hostname, network, users, time zone, etc.
- Clients are matched to manifests by criteria such as architecture, MAC, or memory.

This is the right path when you need repeatable, unattended installs across a fleet.

### Setting Up an AI Server (`installadm`)

```bash
# Install the AI server package
pkg install install/installadm

# Create an install service from a downloaded AI ISO
installadm create-service -n s11-x86 -s /repo/sol-11_4-ai-x86.iso -i 192.168.1.50

# List services, manifests, and clients
installadm list
installadm list -m -n s11-x86           # manifests for a service
installadm list -c                       # associated clients

# Associate a specific client (by MAC) with a service
installadm create-client -e 08:00:27:ab:cd:ef -n s11-x86
```

### AI Manifest and sysconfig Profile

An **AI manifest** (XML) describes the target disk, publishers, and packages:

```xml
<!-- excerpt of an AI manifest -->
<auto_install>
  <ai_instance name="default">
    <target>
      <logical>
        <zpool name="rpool" is_root="true"/>
      </logical>
    </target>
    <software type="IPS">
      <source>
        <publisher name="solaris">
          <origin name="http://192.168.1.40/"/>
        </publisher>
      </source>
      <software_data action="install">
        <name>pkg:/entire</name>
        <name>pkg:/group/system/solaris-large-server</name>
      </software_data>
    </software>
  </ai_instance>
</auto_install>
```

A **sysconfig profile** supplies identity (hostname, root/user accounts, network, timezone). Generate an interactive template with:

```bash
sysconfig create-profile -o /tmp/sc_profile.xml
# Add it to a service:
installadm create-profile -n s11-x86 -f /tmp/sc_profile.xml
```

Clients are matched to a manifest/profile by criteria (architecture, MAC, memory, IP range) so different hardware can get different configurations from one server.

## Choosing an Installer

- **Single x86 workstation, want a GUI** → Live Media.
- **Server, or any SPARC system** → Text installer.
- **Many systems, repeatable/unattended** → Automated Installer.

## Installation Log

The installer records its actions to a log you can review afterward (or consult if an install fails):

```bash
/var/log/install/install_log
```

- Review it after installation to confirm what was installed and catch warnings.
- On a failed or partial install, this is the first place to look for the cause.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| AI client won't PXE boot | DHCP/AI service not serving that client | Check `installadm list -c`; verify DHCP and `create-client` MAC |
| Install picks wrong manifest | Criteria too broad/overlapping | Refine manifest criteria (arch/MAC/mem); `installadm list -m` |
| Text installer can't find a disk | Unsupported controller / no label | Check firmware; label the disk; verify `format` sees it |
| Install fails pulling packages | Publisher unreachable from manifest | Confirm the IPS origin URL in the manifest is reachable |
| Need to review a failed install | — | Read `/var/log/install/install_log` on the target |

## Post-Installation

After the base OS is installed, common next steps are covered in the related Solaris articles:

- [Solaris 11: Configure a Static IP with ipadm and dladm](articles/solaris11-static-ip-ipadm.md) — network setup
- [Solaris 11 IPS: Local Package Repository and pkg Management](articles/solaris-ips-pkg-repository.md) — package repositories and updates
- [Solaris User and Password Administration](articles/solaris-user-password-administration.md) — accounts and password policy

## References

- [Installing Oracle Solaris 11 Systems](https://docs.oracle.com/cd/E23824_01/html/E21798/index.html) — official Oracle docs
- [Automatically Installing Oracle Solaris 11 Systems](https://docs.oracle.com/cd/E23824_01/html/E21798/useraai.html) — official Oracle docs
