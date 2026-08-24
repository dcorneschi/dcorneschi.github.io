# Kubelet Privilege and Capability Check

How to inspect what Linux capabilities and privileges the kubelet process is using on a Kubernetes node.

## 1. Check Linux Capabilities

Inspect the actual capabilities kubelet is using:

```bash
# Get kubelet PID
KUBELET_PID=$(pgrep kubelet)

# View capabilities
sudo cat /proc/$KUBELET_PID/status | grep Cap

# Decode capabilities to human-readable format
sudo capsh --decode=$(grep CapEff /proc/$KUBELET_PID/status | awk '{print $2}')
```

## 2. Use getpcaps

```bash
KUBELET_PID=$(pgrep kubelet)
sudo getpcaps $KUBELET_PID
```

## 3. Audit System Calls with strace

Monitor what system calls kubelet makes during startup:

```bash
# Attach to running kubelet
sudo strace -p $(pgrep kubelet) -f -e trace=all 2>&1 | tee kubelet-syscalls.log

# Or start kubelet under strace
sudo systemctl stop kubelet
sudo strace -f -o kubelet-trace.log /usr/bin/kubelet <flags>
```

## 4. Check Kubelet Service File

Review the systemd service configuration:

```bash
systemctl cat kubelet
```

Look for privilege-related settings like:
- `CapabilityBoundingSet`
- `AmbientCapabilities`
- `User=`
- `PermissionsStartOnly`

## 5. Use auditd to Monitor Operations

Enable audit logging for kubelet:

```bash
# Check if auditd is running
sudo systemctl status auditd

# Add audit rule for kubelet
sudo auditctl -w /usr/bin/kubelet -p x -k kubelet_exec

# Monitor specific syscalls
sudo auditctl -a always,exit -F arch=b64 -S mount,umount2,setns,unshare -F exe=/usr/bin/kubelet -k kubelet_privops

# View audit logs
sudo ausearch -k kubelet_privops
```

## 6. Check SELinux/AppArmor Context

```bash
# SELinux
ps -eZ | grep kubelet
ls -Z /usr/bin/kubelet

# AppArmor
sudo aa-status | grep kubelet
```

## 7. Review Kubelet Documentation

Check the kubelet flags and requirements:

```bash
kubelet --help | grep -i secur
kubelet --help | grep -i cap
```

## Common Privileges Kubelet Requires

| Capability | Purpose |
|------------|---------|
| `CAP_SYS_ADMIN` | Mount filesystems, set namespaces |
| `CAP_NET_ADMIN` | Network configuration |
| `CAP_NET_BIND_SERVICE` | Bind to privileged ports |
| `CAP_SYS_RESOURCE` | Override resource limits |
| `CAP_SYS_PTRACE` | Debug/trace processes |
| `CAP_DAC_OVERRIDE` | Bypass file permission checks |
| `CAP_FOWNER` | Bypass permission checks for file operations |
| `CAP_SETUID` / `CAP_SETGID` | Set process UIDs/GIDs |
| `CAP_KILL` | Send signals to processes |
| `CAP_CHOWN` | Change file ownership |

## Complete Capability Check Script

```bash
#!/bin/bash

echo "=== Kubelet Privilege Check ==="
echo

# Check if kubelet is running
if ! pgrep kubelet > /dev/null; then
    echo "ERROR: kubelet is not running"
    exit 1
fi

KUBELET_PID=$(pgrep kubelet)
echo "Kubelet PID: $KUBELET_PID"
echo

# Get capabilities
echo "--- Capabilities (Raw) ---"
sudo cat /proc/$KUBELET_PID/status | grep Cap
echo

# Get effective capabilities decoded
echo "--- Effective Capabilities (Decoded) ---"
CAP_EFF=$(grep CapEff /proc/$KUBELET_PID/status | awk '{print $2}')
sudo capsh --decode=$CAP_EFF
echo

# Check user/group
echo "--- Process User/Group ---"
ps -o pid,user,group,comm -p $KUBELET_PID
echo

# Check SELinux context (if available)
if command -v getenforce &> /dev/null; then
    echo "--- SELinux Context ---"
    ps -eZ | grep kubelet
    echo
fi

# Check systemd service capabilities
echo "--- Systemd Service Configuration ---"
systemctl show kubelet.service | grep -i cap
echo

echo "=== Check Complete ==="
```

Save this script and run it on your node to get a complete overview of kubelet privileges.
