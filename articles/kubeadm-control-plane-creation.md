# How kubeadm Creates a Kubernetes Control Plane (the Self-Managed Path)

On Amazon EKS, AWS builds and operates the control plane for you — you never touch etcd or
the API server process. **`kubeadm` is the opposite end of that spectrum:** it's the
official upstream tool for bootstrapping a conformant Kubernetes cluster on machines *you*
own (bare metal, VMs, a homelab, on-prem). You run the control-plane components yourself,
so understanding what kubeadm does is the best way to see what a managed service like EKS
is doing under the hood.

This article walks through what `kubeadm init` actually creates, how the pieces connect,
how nodes join, and how the control plane talks to the outside world — deliberately
mirroring the structure of the EKS control-plane doc so you can compare the two models.

> Scope note: `kubeadm` bootstraps and upgrades the cluster. It intentionally does **not**
> install a CNI network plugin, provision infrastructure, or manage nodes long-term —
> those are your responsibility. It gets you a working, best-practice control plane and a
> join mechanism, then gets out of the way.

---

## 1. Prerequisites (what must be true before `kubeadm init`)

kubeadm assumes the host is already prepared. On each machine you need:

- A supported Linux host, with **swap disabled** (or kubelet configured to tolerate it).
- A **container runtime** that speaks CRI — containerd, CRI-O, etc. — with a matching
  cgroup driver (`systemd` on modern distros).
- The three packages installed and the right version: **`kubeadm`, `kubelet`, `kubectl`**.
- Required **kernel modules and sysctls** (e.g. `br_netfilter`,
  `net.bridge.bridge-nf-call-iptables=1`, `net.ipv4.ip_forward=1`).
- **Ports open**: API server `6443`, etcd `2379-2380`, kubelet `10250`, scheduler/
  controller-manager `10259`/`10257`, and your CNI's ports.
- Unique hostname, MAC, and `product_uuid` per node.

The one long-running daemon on every node is the **kubelet**. kubeadm configures it, but
the kubelet is what actually runs the control-plane containers as **static pods**.

---

## 2. What `kubeadm init` does, phase by phase

`kubeadm init` runs a sequence of **phases**. You can run them all at once, or invoke any
phase individually (`kubeadm init phase <name>`) for customization. The main phases:

1. **preflight** — Validates the host: runtime reachable, swap off, ports free, versions
   compatible, required kernel settings present. Fails fast if something's wrong.
2. **certs** — Creates the cluster **PKI** under `/etc/kubernetes/pki`: a self-signed
   cluster CA, plus certs/keys for the API server, the API server↔kubelet client, the
   API server↔etcd client, the front-proxy CA, and the **etcd CA** and its serving/peer
   certs. Also the **service account** signing key pair.
3. **kubeconfig** — Writes admin/controller-manager/scheduler/kubelet kubeconfig files to
   `/etc/kubernetes/*.conf`, each with embedded client certs. `admin.conf` is the one you
   copy to `~/.kube/config`.
4. **kubelet-start** — Writes the kubelet config and starts the kubelet service.
5. **control-plane** — Writes **static Pod manifests** to
   `/etc/kubernetes/manifests/` for `kube-apiserver`, `kube-controller-manager`, and
   `kube-scheduler`. The kubelet watches that directory and starts them as static pods.
6. **etcd** — Writes a static Pod manifest for a **local etcd** (stacked topology). Skipped
   if you point kubeadm at an external etcd cluster.
7. **wait-control-plane / upload-config** — Waits for the API server to come healthy, then
   stores the cluster and kubelet configuration as ConfigMaps (`kubeadm-config`,
   `kubelet-config`) in the `kube-system` namespace.
8. **mark-control-plane** — Labels and taints the node so ordinary workloads don't schedule
   on it (`node-role.kubernetes.io/control-plane:NoSchedule`).
9. **bootstrap-token** — Creates a **bootstrap token** used by nodes to join securely.
10. **addon** — Deploys the two required add-ons: **CoreDNS** and **kube-proxy**.

At the end, `kubeadm init` prints a `kubeadm join ...` command (with the token and CA hash)
that you run on other nodes.

```bash
# Minimal control-plane bootstrap. --pod-network-cidr must match your CNI's expectation.
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --control-plane-endpoint="k8s-api.example.com:6443"   # use a stable name for HA

# Then set up kubectl access
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> `--control-plane-endpoint` matters: set it to a **stable DNS name / load balancer** even
> for a single control-plane node if you might ever go HA. You cannot cleanly add it later.

---

## 3. What you end up with: static pods, not Deployments

The defining trait of a kubeadm control plane is that the core components run as **static
pods**, managed directly by the kubelet from manifests on disk — *not* by the API server as
Deployments. This is the bootstrapping chicken-and-egg solution: the kubelet can start the
API server before the API server exists to schedule anything.

- Manifests live in `/etc/kubernetes/manifests/`:
  - `kube-apiserver.yaml`
  - `kube-controller-manager.yaml`
  - `kube-scheduler.yaml`
  - `etcd.yaml` (stacked etcd)
- The kubelet **watches that directory**. Edit a manifest and the kubelet restarts that
  pod. Delete it and the pod stops. There is no `kubectl scale` for these.
- They appear in `kubectl get pods -n kube-system` as **mirror pods** (read-only
  reflections), with the node name as a suffix, e.g. `kube-apiserver-cp1`.

```bash
# Inspect the static-pod manifests the kubelet is running
ls /etc/kubernetes/manifests/
sudo less /etc/kubernetes/manifests/kube-apiserver.yaml

# See them reflected as mirror pods
kubectl get pods -n kube-system -o wide | grep -E 'apiserver|controller|scheduler|etcd'
```

To change an API server flag, you **edit the manifest file on disk**, not a Kubernetes
object. This is exactly what a managed provider hides from you.

---

## 4. etcd topology: stacked vs external

kubeadm supports two etcd layouts:

- **Stacked (default):** etcd runs as a static pod *on the same node(s)* as the other
  control-plane components. Simpler, fewer machines. Losing a control-plane node loses that
  node's etcd member too, so HA requires an **odd number (3 or 5)** of control-plane nodes
  for quorum.
- **External:** etcd runs on its own dedicated hosts/cluster, and you pass its endpoints
  and certs to kubeadm. Decouples etcd failures from control-plane failures; more machines
  to operate. Better for larger or stricter-SLA clusters.

Either way, **etcd is the single source of truth** for all cluster state. Back it up:

```bash
# Snapshot etcd (stacked). Point at the etcd static-pod cert paths.
sudo ETCDCTL_API=3 etcdctl snapshot save /var/backups/etcd-$(date +%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

## 5. How nodes join

Joining is a **token + TLS-bootstrap** flow, secured so a new node can trust the control
plane and vice versa without pre-sharing full credentials.

- **Worker nodes** run `kubeadm join` with a **bootstrap token** and the **CA cert hash**
  (`--discovery-token-ca-cert-hash sha256:...`). The token authenticates the join and lets
  the node fetch the cluster-info; the CA hash lets the node verify it's talking to the
  real control plane (defends against MITM).
- The joining kubelet then does **TLS bootstrapping**: it submits a Certificate Signing
  Request; kubeadm's auto-approver signs it, and the kubelet gets its own client cert. No
  long-lived shared secret ends up on the node.

```bash
# Worker join (printed by kubeadm init; regenerate a token if it expired)
sudo kubeadm join k8s-api.example.com:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:<hash>

# Additional control-plane node (HA) — adds the --control-plane flag + cert key
sudo kubeadm join k8s-api.example.com:6443 \
  --token <token> --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <cert-key>

# Tokens expire after 24h by default; mint a fresh join command
kubeadm token create --print-join-command
```

---

## 6. How the control plane talks to the outside world

Unlike EKS — where AWS injects cross-account ENIs and a managed load-balanced endpoint —
with kubeadm **you own every network path**:

- **The API server endpoint is whatever you exposed.** It's `kube-apiserver` listening on
  `:6443` on the control-plane host(s). Clients (`kubectl`, kubelets) connect to the
  address in their kubeconfig — either a node IP or, for HA, the `--control-plane-endpoint`
  DNS name in front of a **load balancer you run** (HAProxy, an external LB, keepalived +
  VIP, etc.). kubeadm does **not** create a load balancer.
- **Pods reach the API server via the `kubernetes` Service** — same mechanism as anywhere:
  a `ClusterIP` Service in the `default` namespace, taking the **first IP of the service
  CIDR** (e.g. `10.96.0.1`), with kube-proxy translating it to the real API server
  endpoint(s). This part is identical to EKS.
- **Control plane → kubelet** (for `kubectl exec`/`logs`/`port-forward`) goes over the node
  network to kubelet port `10250`, secured by the certs kubeadm generated. There's no
  cross-account ENI abstraction; it's just routable IPs on your network plus firewall
  rules you manage.
- **CNI is your job.** kubeadm sets up kube-proxy and the service CIDR, but pod-to-pod
  networking doesn't work until *you* install a CNI plugin (Calico, Cilium, Flannel, etc.).
  Nodes stay `NotReady` until the CNI is up.

```bash
# You must install a CNI before nodes go Ready — example (choose one plugin):
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml

# Confirm the api server endpoint the kubernetes Service points at
kubectl -n default get endpoints kubernetes -o yaml
```

---

## 7. Upgrades and certificate management (your responsibility)

- **Upgrades** are driven by `kubeadm upgrade`, one minor version at a time, control plane
  first, then kubelets — the same n-1 discipline as EKS, but you run each step:

  ```bash
  kubeadm upgrade plan                      # shows available target versions
  sudo kubeadm upgrade apply v1.35.0        # upgrade the control plane
  # then drain each node, upgrade kubelet/kubeadm packages, uncordon
  ```

- **Certificates** kubeadm generates are valid for **1 year** and are **renewed
  automatically on `kubeadm upgrade`**. If you don't upgrade within a year, renew manually
  or the control plane stops trusting its own certs:

  ```bash
  kubeadm certs check-expiration      # see expiry dates
  sudo kubeadm certs renew all        # renew everything (then restart control-plane pods)
  ```

This is precisely the toil EKS removes: version upgrades, cert rotation, etcd backups,
and endpoint HA are all on you with kubeadm.

---

## 8. kubeadm vs a managed control plane (EKS)

| Concern | kubeadm (self-managed) | Amazon EKS (managed) |
|---|---|---|
| Who runs API server / etcd / scheduler | You, as static pods on your hosts | AWS, in a managed account |
| HA / multi-AZ | You design it (odd # CP nodes, your LB) | Built in (≥2 API servers, 3 etcd, 3 AZs) |
| API endpoint | `:6443` on your host / your load balancer | AWS-managed load-balanced endpoint |
| Control plane → pod networking | Routable IPs + your firewall rules | Cross-account ENIs (X-ENIs) in your VPC |
| CNI | You install it | Managed VPC CNI add-on available |
| Upgrades | `kubeadm upgrade`, you run every step | You trigger; AWS performs CP upgrade |
| Certificates | You renew (auto on upgrade, else manual) | AWS-managed |
| etcd backups | Your job | AWS-managed |
| Cost model | Just your machines | Per-cluster control plane fee + compute |

The `kubernetes` Service indirection and the one-minor-at-a-time upgrade rule are the same
in both. Everything else that EKS "just handles," kubeadm makes explicit.

---

## 9. Quick reference

| Goal | Command |
|---|---|
| Bootstrap a control plane | `sudo kubeadm init --pod-network-cidr=<cidr> --control-plane-endpoint=<name:6443>` |
| Set up kubectl | `cp /etc/kubernetes/admin.conf ~/.kube/config` |
| Join a worker | `sudo kubeadm join <ep> --token <t> --discovery-token-ca-cert-hash sha256:<h>` |
| New join command / token | `kubeadm token create --print-join-command` |
| See control-plane static pods | `ls /etc/kubernetes/manifests/` |
| Change an API server flag | edit `/etc/kubernetes/manifests/kube-apiserver.yaml` |
| Check cert expiry | `kubeadm certs check-expiration` |
| Renew certs | `sudo kubeadm certs renew all` |
| Plan / apply upgrade | `kubeadm upgrade plan` / `sudo kubeadm upgrade apply vX.Y.Z` |
| Reset a node (undo) | `sudo kubeadm reset` |
| Snapshot etcd | `etcdctl snapshot save ... --cacert ... --cert ... --key ...` |

---

## Summary

- **`kubeadm init`** runs an ordered set of phases: preflight → PKI/certs → kubeconfigs →
  kubelet → control-plane static pods → etcd → config upload → taints → bootstrap token →
  CoreDNS/kube-proxy.
- The control plane runs as **static pods** managed by the kubelet from
  `/etc/kubernetes/manifests/`; you configure components by editing those files.
- **etcd** is stacked by default (needs an odd number of CP nodes for HA) or external.
- **Nodes join** via a bootstrap token + CA-cert-hash, then TLS-bootstrap their own certs.
- **You own all networking** (endpoint/LB, firewall, and the CNI plugin) and all day-2 ops
  (upgrades, cert renewal, etcd backups) — the exact responsibilities a managed service
  like EKS takes over.

---

### Sources

- [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [kubeadm init](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)
- [kubeadm join](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/)
- [Options for Highly Available topology (stacked vs external etcd)](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)
- [Certificate Management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)
- [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
