# Fixing k3d TLS Certificate SAN Errors

## Problem

```text
tls: failed to verify certificate: x509: certificate is valid for 0.0.0.0, 10.43.0.1, 127.0.0.1, 172.18.0.2, ::1, not <VM_IP>
```

The API server certificate doesn't include your VM's IP address (`<VM_IP>`), so
`kubectl` from another machine refuses to trust it. The fix is to either add the
IP to the certificate's Subject Alternative Names (SANs), tunnel to an address
the cert already covers, or (for testing only) skip verification.

## Solution Options

### Option 1: Skip TLS verification (quick fix)

On your laptop:

```bash
kubectl config set-cluster k3d-mycluster --insecure-skip-tls-verify=true
```

Test the connection:

```bash
kubectl get nodes
```

**Pros:** Quick and easy.
**Cons:** Less secure — not recommended for production.

### Option 2: Recreate k3d with the correct IP (recommended)

On the VM:

```bash
# Delete the existing cluster
k3d cluster delete mycluster

# Recreate with the VM IP included in the certificate
k3d cluster create mycluster \
  --servers 1 \
  --agents 2 \
  --api-port 6443 \
  --k3s-arg "--tls-san=<VM_IP>@server:0"
```

Then re-export the config and point it at the VM's IP:

```bash
# On the VM
k3d kubeconfig get mycluster > ~/k3d-mycluster.yaml
VM_IP=$(hostname -I | awk '{print $1}')
sed -i "s/0.0.0.0/$VM_IP/g" ~/k3d-mycluster.yaml
sed -i "s/127.0.0.1/$VM_IP/g" ~/k3d-mycluster.yaml
```

Transfer the file to your laptop and merge it into your kubeconfig (see
[Cleaning Up Kubernetes Clusters from .kube/config](articles/kubeconfig-cleanup-guide.md)
for merge steps).

**Pros:** Proper solution with valid certificates.
**Cons:** Requires recreating the cluster.

### Option 3: Use SSH port forwarding

On your laptop, create an SSH tunnel to the VM:

```bash
ssh -L 6443:localhost:6443 user@<VM_IP> -N
```

Keep this terminal running. In another terminal, point the kubeconfig at
`127.0.0.1` (which the certificate already covers):

```bash
kubectl config set-cluster k3d-mycluster --server=https://127.0.0.1:6443
```

Test:

```bash
kubectl get nodes
```

**Pros:** No need to recreate the cluster, and TLS verification still works.
**Cons:** Requires keeping the SSH tunnel running.

### Option 4: Add multiple IPs to the certificate

To access from multiple networks, recreate with all the IPs you need:

```bash
k3d cluster delete mycluster

k3d cluster create mycluster \
  --servers 1 \
  --agents 2 \
  --api-port 6443 \
  --k3s-arg "--tls-san=<VM_IP>@server:0" \
  --k3s-arg "--tls-san=<SECOND_IP>@server:0"
```

Add as many `--tls-san` flags as needed for different IPs.

## Recommended Approach

- **For quick testing:** Option 1
- **For a proper setup:** Option 2
- **If you can't recreate the cluster:** Option 3

## Verify Certificate SANs

To check which IPs are in the certificate:

```bash
# On the VM
docker exec k3d-mycluster-server-0 cat /var/lib/rancher/k3s/server/tls/serving-kube-apiserver.crt | \
  openssl x509 -noout -text | grep -A1 "Subject Alternative Name"
```

## Skills Practiced

- Reading an x509 SAN mismatch error and identifying the missing IP
- Adding IPs to the API server cert with `--tls-san` when creating a k3d cluster
- Working around cert mismatches via SSH tunneling or skipping verification
- Inspecting certificate SANs with `openssl x509`
