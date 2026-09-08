# Using systemctl to Debug EKS Nodes

When debugging EKS nodes, you often need `systemctl` to check and manage system
services like kubelet, containerd, and other components. Because a debug
container has its own PID 1 (not systemd), you reach the host's services with
`kubectl debug node/...` and `chroot /host`.

## Why systemctl Matters for EKS Debugging

Key node services that are managed by systemd:

- **kubelet** — Kubernetes node agent
- **containerd** — container runtime (default on modern EKS)
- **docker** — only on older Docker-runtime nodes
- **amazon-ssm-agent** — Systems Manager agent
- **chronyd / ntp** — time synchronization
- **systemd-networkd / NetworkManager** — network management

Note that `kube-proxy` and the VPC CNI (`aws-node`) run as DaemonSet pods, not
systemd services — check those with `kubectl`, not `systemctl`.

## Container Images with systemctl Support

The debug container doesn't run systemd itself; it just needs the `systemctl`
binary and shell tools, which you use against the host via `chroot /host`.

### Ubuntu (recommended)

```bash
kubectl debug node/<node-name> -it --image=ubuntu

# Inside the container — systemctl runs against the host
chroot /host systemctl status kubelet
chroot /host systemctl status containerd
```

Full systemd tooling, large package repo, well documented. Slightly larger
image and more surface area than a minimal base.

### Amazon Linux 2

```bash
kubectl debug node/<node-name> -it --image=amazonlinux:2

chroot /host systemctl status kubelet
chroot /host systemctl show kubelet
```

AWS-native and closest to the EKS node OS. Smaller package selection than Ubuntu.

### CentOS / RHEL-based

```bash
kubectl debug node/<node-name> -it --image=centos:8

chroot /host systemctl status kubelet
chroot /host systemctl is-active containerd
```

Good systemd integration and Red Hat tooling. Larger image, and CentOS 8 is
end-of-life — prefer a maintained base for anything recurring.

### Fedora

```bash
kubectl debug node/<node-name> -it --image=fedora

chroot /host systemctl status kubelet
chroot /host journalctl -u kubelet
```

Latest systemd features; changes frequently, so less predictable for routine
debugging.

### Minimal images (alpine, busybox)

These have **no systemd** and no `systemctl`. They're fine for quick filesystem
or network checks, but you can't manage services with them.

## Using systemctl in Debug Containers

```bash
kubectl debug node/<node-name> -it --image=ubuntu

# Kubelet
chroot /host systemctl status kubelet
chroot /host systemctl is-active kubelet
chroot /host systemctl is-enabled kubelet

# Container runtime
chroot /host systemctl status containerd
chroot /host systemctl status docker   # only if Docker runtime

# Logs
chroot /host journalctl -u kubelet -n 50 --no-pager
chroot /host journalctl -u containerd -n 50 --no-pager

# Restart (be careful — see the warning below)
chroot /host systemctl restart kubelet
chroot /host systemctl restart containerd
```

Advanced inspection:

```bash
chroot /host systemctl list-dependencies kubelet
chroot /host systemctl --failed
chroot /host systemctl show kubelet
chroot /host systemctl cat kubelet
chroot /host systemd-analyze blame
chroot /host systemd-analyze critical-chain
```

## Debugging Workflow

### Quick health check

```bash
kubectl debug node/<node-name> -it --image=ubuntu -- bash -c '
echo "=== EKS Node Health Check ==="
date
echo
echo "=== Kubelet ==="
chroot /host systemctl is-active kubelet
chroot /host systemctl is-enabled kubelet
echo "=== Container Runtime ==="
chroot /host systemctl is-active containerd
chroot /host systemctl is-active docker 2>/dev/null || echo "Docker not active"
echo "=== Load ===" ; uptime
echo "=== Disk ===" ; df -h /host
echo "=== Memory ===" ; free -h
'
```

### Detailed service analysis

```bash
kubectl debug node/<node-name> -it --image=ubuntu

chroot /host systemctl status kubelet -l --no-pager
chroot /host journalctl -u kubelet -n 100 --no-pager
chroot /host systemctl show kubelet | grep -E "ExecStart|Environment|WorkingDirectory"
chroot /host systemctl status containerd -l --no-pager
chroot /host systemctl --failed --no-pager
```

### Service management

```bash
# Restart kubelet, then confirm
kubectl debug node/<node-name> -it --image=ubuntu -- bash -c '
chroot /host systemctl restart kubelet
sleep 5
chroot /host systemctl status kubelet --no-pager
'

# Reload unit files after editing them
kubectl debug node/<node-name> -it --image=ubuntu -- bash -c '
chroot /host systemctl daemon-reload
chroot /host systemctl status kubelet --no-pager
'
```

## EKS-Specific Service Commands

### Kubelet

```bash
chroot /host systemctl status kubelet
chroot /host systemctl cat kubelet
cat /host/etc/systemd/system/kubelet.service
cat /host/etc/systemd/system/kubelet.service.d/*.conf
chroot /host systemctl show kubelet | grep Environment
```

### Container runtime

```bash
chroot /host systemctl status containerd
chroot /host systemctl restart containerd

# Only if the node still uses Docker
chroot /host systemctl status docker

# Inspect runtime config
cat /host/etc/containerd/config.toml
cat /host/var/lib/kubelet/config.yaml
```

### AWS agents

```bash
chroot /host systemctl status amazon-ssm-agent
chroot /host systemctl status amazon-cloudwatch-agent   # if installed
```

## Troubleshooting Common Issues

### Kubelet won't start

```bash
chroot /host systemctl status kubelet
chroot /host journalctl -u kubelet -n 50 --no-pager

# Common checks
df -h /host                              # disk space
cat /host/var/lib/kubelet/config.yaml    # config
ls -la /host/var/lib/kubelet/pki/        # certificates

chroot /host systemctl restart kubelet
```

### Container runtime issues

```bash
chroot /host systemctl status containerd
chroot /host journalctl -u containerd -n 50 --no-pager
cat /host/etc/containerd/config.toml

# Restarting containerd restarts ALL containers on the node
chroot /host systemctl restart containerd
```

### Node NotReady

```bash
for service in kubelet containerd docker; do
  echo "--- $service ---"
  chroot /host systemctl is-active "$service" 2>/dev/null || echo "not active/not found"
done

free -h
df -h /host
uptime
```

## Pro Tips

### Debugging aliases

```bash
alias kstatus='chroot /host systemctl status'
alias jlogs='chroot /host journalctl -u'
alias srestart='chroot /host systemctl restart'
```

### Service check script

```bash
#!/bin/bash
services=("kubelet" "containerd" "docker" "amazon-ssm-agent")

echo "=== EKS Node Services Status ==="
for service in "${services[@]}"; do
    echo "--- $service ---"
    if chroot /host systemctl is-active "$service" >/dev/null 2>&1; then
        echo "Status: ACTIVE"
        chroot /host systemctl status "$service" --no-pager -l | head -5
    else
        echo "Status: INACTIVE/FAILED"
        chroot /host systemctl status "$service" --no-pager -l | head -10
    fi
    echo
done
```

### Recent-errors script

```bash
#!/bin/bash
for service in kubelet containerd docker; do
    echo "=== Recent errors in $service ==="
    chroot /host journalctl -u "$service" -p err -n 20 --no-pager
    echo
done
```

## Image Comparison

| Image | Approx. size | systemctl (via chroot) | Best for |
|-------|-------------:|:----------------------:|----------|
| ubuntu | ~70 MB | Yes | General debugging |
| amazonlinux:2 | ~165 MB | Yes | AWS-native debugging |
| centos:8 | ~200 MB | Yes | Enterprise environments |
| fedora | ~190 MB | Yes | Latest systemd features |
| alpine | ~5 MB | No (no systemd) | Quick checks only |
| busybox | ~1 MB | No (no systemd) | Basic tasks only |

## Important Notes

**Always use `chroot /host`.** `systemctl` in the container talks to the
container's init, not the node's. Prefix host commands with `chroot /host`:

```bash
chroot /host systemctl status kubelet   # correct
systemctl status kubelet                # wrong — not the host's systemd
```

**Be careful with restarts.** Restarting `kubelet` briefly disrupts the node;
restarting `containerd` restarts every container on it. Check state first, then
restart only if needed:

```bash
chroot /host systemctl is-active kubelet
chroot /host systemctl status kubelet --no-pager
chroot /host systemctl restart kubelet
```

**`kubectl debug node/...` creates a privileged pod** on the target node that
mounts the host filesystem at `/host` and shares host namespaces. Clean up the
debug pod when finished.

## Custom Debug Image (Optional)

For frequent debugging, bake the tools into an image:

```dockerfile
FROM amazonlinux:2

RUN yum update -y && yum install -y \
    systemd curl wget vim htop iotop tcpdump nc bind-utils \
    iputils strace lsof procps tree jq \
    && yum clean all

RUN echo 'alias kstatus="chroot /host systemctl status"' >> /root/.bashrc && \
    echo 'alias jlogs="chroot /host journalctl -u"'       >> /root/.bashrc && \
    echo 'alias srestart="chroot /host systemctl restart"' >> /root/.bashrc

WORKDIR /
CMD ["/bin/bash"]
```

```bash
docker build -t my-registry/eks-debug:latest .
docker push my-registry/eks-debug:latest

kubectl debug node/<node-name> -it --image=my-registry/eks-debug:latest
```

## Quick Reference

| Task | Command |
|------|---------|
| Kubelet status | `chroot /host systemctl status kubelet` |
| Kubelet logs | `chroot /host journalctl -u kubelet -n 50` |
| Restart kubelet | `chroot /host systemctl restart kubelet` |
| Containerd status | `chroot /host systemctl status containerd` |
| Failed services | `chroot /host systemctl --failed` |
| Service dependencies | `chroot /host systemctl list-dependencies kubelet` |
| Reload systemd config | `chroot /host systemctl daemon-reload` |

## Related

- [Seeing Network Traffic on EKS Nodes](articles/eks-node-network-traffic-debugging.md)
- [EKS Node Troubleshooting Guide](articles/eks-node-troubleshooting-guide.md)
- [kubectl Cheatsheet](articles/kubectl-cheatsheet.md)

## Skills Practiced

- Reaching host systemd from a debug container with `kubectl debug` + `chroot /host`
- Inspecting and managing kubelet/containerd and AWS agents on EKS nodes
- Reading service logs and unit config, and diagnosing NotReady/kubelet failures
- Choosing an appropriate debug image and understanding restart blast radius
