# Windows Tips and Commands

Useful commands, shortcuts, and troubleshooting tips for Windows system administration.

## System Health and Performance

### Generate System Health Report

```cmd
perfmon /report
```

Opens Performance Monitor and generates a 60-second diagnostic report.

### Generate Benchmark Report

```cmd
winsat formal
```

Runs a full Windows System Assessment Tool benchmark.

### Generate Energy Report

```cmd
powercfg -energy
```

Creates an energy efficiency report (saved to `C:\energy-report.html`).

## Disk and Storage

### Disk Cleanup

```cmd
cleanmgr
```

### Unable to Format Disk ("The system cannot find the file specified")

```cmd
diskpart

LIST DISK
SELECT DISK x
CLEAN
```

## Network

### Test a Remote Port (PowerShell)

```powershell
Test-NetConnection corneschi.ro -Port 443
Test-NetConnection 128.159.1.1 -Port 80
```

## Hosts File

### Edit Hosts File

Open Notepad as Administrator and edit:

```
C:\Windows\System32\drivers\etc\hosts
```

Or from an elevated command prompt:

```cmd
notepad C:\Windows\System32\drivers\etc\hosts
```

## Processes and Open Files

### List All Processes

```cmd
tasklist
```

### Find Which Process Holds an Open File

Open Resource Monitor:

```cmd
resmon.exe
```

1. Go to the **CPU** tab
2. Use the search field in the **Associated Handles** section
3. Type the filename to find which process has it locked

### View Who Is Connected to Shared Folders

```cmd
compmgmt.msc
```

Navigate to: System Tools → Shared Folders → Sessions

## Group Policy

### Force Group Policy Update

```cmd
gpupdate /force
```

## Java

### Clear Java Cache

```cmd
javaws -clearcache
```

### Java Configuration Paths

| File | Path |
|------|------|
| Security config | `C:\Program Files\Java\jre\lib\security\java.security` |
| Exception sites (system) | `C:\Windows\Sun\Java\Deployment\exception.sites` |
| Exception sites (user) | `C:\Users\{username}\AppData\LocalLow\Sun\Java\Deployment\security\exception.sites` |

## System Information

```cmd
systeminfo                         # Full system details (OS, hardware, hotfixes)
hostname                           # Computer name
whoami                             # Current user and domain
whoami /priv                       # Current user privileges
set                                # Show all environment variables
setx VAR "value"                   # Set persistent environment variable
```

## Network Commands

```cmd
ipconfig /all                      # All network adapter details
ipconfig /flushdns                 # Flush DNS resolver cache
ipconfig /release                  # Release DHCP lease
ipconfig /renew                    # Renew DHCP lease
nslookup example.com               # DNS lookup
netstat -an                        # All connections and listening ports
netstat -b                         # Show which process owns each connection
arp -a                             # Show ARP cache
route print                        # Show routing table
tracert example.com                # Trace route to host
pathping example.com               # Combined ping + tracert with stats
```

## Services

```cmd
sc query                           # List all services
sc query "ServiceName"             # Query specific service
sc start "ServiceName"             # Start a service
sc stop "ServiceName"              # Stop a service
net start                          # List running services
net start "ServiceName"            # Start a service
net stop "ServiceName"             # Stop a service
```

## Firewall

```cmd
netsh advfirewall show allprofiles                          # Show firewall status
netsh advfirewall set allprofiles state off                 # Disable firewall (all profiles)
netsh advfirewall set allprofiles state on                  # Enable firewall (all profiles)
netsh advfirewall firewall show rule name=all               # List all rules
netsh advfirewall firewall add rule name="Allow 8080" dir=in action=allow protocol=tcp localport=8080
```

## User and Computer Management

```cmd
net user                           # List local users
net user username                  # Show user details
net user username /active:yes      # Enable a user account
net localgroup administrators      # List local admins
lusrmgr.msc                        # Local Users and Groups snap-in
dsquery user -name "*admin*"       # Query AD for users (domain-joined)
dsquery computer                   # Query AD for computers
```

## System File Repair

```cmd
sfc /scannow                       # Scan and repair system files
DISM /Online /Cleanup-Image /CheckHealth    # Quick health check
DISM /Online /Cleanup-Image /ScanHealth     # Deeper scan
DISM /Online /Cleanup-Image /RestoreHealth  # Repair from Windows Update
```

## Shutdown and Restart

```cmd
shutdown /r /t 0                   # Restart immediately
shutdown /s /t 0                   # Shutdown immediately
shutdown /r /t 60 /c "Rebooting in 60s"  # Restart with delay and message
shutdown /a                        # Abort pending shutdown
shutdown /l                        # Log off current user
```

## Useful MMC Snap-ins

| Command | Opens |
|---------|-------|
| `services.msc` | Services |
| `devmgmt.msc` | Device Manager |
| `diskmgmt.msc` | Disk Management |
| `compmgmt.msc` | Computer Management |
| `eventvwr.msc` | Event Viewer |
| `lusrmgr.msc` | Local Users and Groups |
| `gpedit.msc` | Local Group Policy Editor |
| `ncpa.cpl` | Network Connections |
| `appwiz.cpl` | Programs and Features |
| `sysdm.cpl` | System Properties |
| `firewall.cpl` | Windows Firewall |
| `certmgr.msc` | Certificate Manager |

## Event Viewer

```cmd
eventvwr.msc                       # Open Event Viewer GUI

# Query events from command line (PowerShell)
Get-EventLog -LogName System -Newest 20
Get-EventLog -LogName Application -EntryType Error -Newest 10
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2} -MaxEvents 10
```

## Scheduled Tasks

```cmd
schtasks /query                    # List all scheduled tasks
schtasks /query /tn "TaskName"     # Query specific task
schtasks /run /tn "TaskName"       # Run a task immediately
schtasks /end /tn "TaskName"       # Stop a running task
```
