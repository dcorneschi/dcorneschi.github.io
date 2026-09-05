# Kubernetes Distributions: K3s vs MicroK8s vs Minikube vs kubeadm and Others

A field guide to the ways you can stand up a Kubernetes cluster yourself — kubeadm, K3s, MicroK8s, Minikube, kind, k3d, k0s, and RKE2. What each one is for, how they differ, and how to pick.

These are the *self-managed* options: tools you run on your own machines. They're distinct from managed control planes like EKS, GKE, and AKS, where the provider runs the control plane for you.

## Overview

| | kubeadm | K3s | MicroK8s | Minikube | kind | k3d | k0s | RKE2 |
|---|---|---|---|---|---|---|---|---|
| **Category** | Upstream installer | Lightweight distro | Lightweight distro | Local dev | Local dev (in Docker) | Local dev (K3s in Docker) | Lightweight distro | Enterprise distro |
| **Maintainer** | Kubernetes project | SUSE (Rancher) | Canonical | Kubernetes project | Kubernetes SIG | Rancher community | Mirantis | SUSE (Rancher) |
| **Packaging** | Binaries + you assemble | Single binary | Snap package | Binary + driver | Docker images | Docker images | Single binary | Install script |
| **Typical min RAM** | ~2 GB | ~512 MB | ~540 MB | ~2 GB | ~4 GB (Docker) | ~1 GB | ~1 GB | ~4 GB |
| **Datastore** | etcd | SQLite (default) or etcd | dqlite | etcd | etcd | SQLite (K3s) | etcd (or others) | etcd |
| **Multi-node / HA** | Yes | Yes | Yes | Limited | Yes (in-Docker) | Yes (in-Docker) | Yes | Yes |
| **Production use** | Yes | Yes (edge, small prod) | Yes (edge, workstation) | No | No (CI/testing) | No (CI/testing) | Yes (edge, IoT) | Yes (enterprise) |
| **Best for** | On-prem prod, learning internals | Edge, IoT, CI, homelab | Edge, dev workstations | Learning, local dev | CI pipelines, testing | Fast local K3s clusters | Bare-metal edge, single binary | Regulated/enterprise prod |

RAM figures are rough minimums for a small node; real workloads need more. Facts checked against project docs and vendor comparisons as of early 2026; details change often. Content was rephrased for compliance with licensing restrictions.

## kubeadm

Not a distribution — a bootstrapping tool from the Kubernetes project that turns machines into a conformant cluster. It handles certificates, the control plane static pods, etcd, and node joins, but leaves the CNI, storage, and lifecycle to you.

- **Strengths:** upstream and vendor-neutral, closest to "vanilla" Kubernetes, full control over every component, and the best way to actually learn how a cluster fits together.
- **Trade-offs:** the most manual option. You assemble and maintain CNI, ingress, storage, and upgrades yourself. More moving parts means more that can break.

### Technical details

- **Architecture:** runs the control plane (`kube-apiserver`, `kube-controller-manager`, `kube-scheduler`) and `etcd` as static Pods managed by the kubelet from manifests in `/etc/kubernetes/manifests/`. The kubelet and container runtime run as host services, not containers.
- **Runtime:** runtime-agnostic via the CRI — commonly containerd or CRI-O. You install and configure it yourself before `kubeadm init`.
- **Networking:** ships no CNI. The cluster stays `NotReady` until you apply one (Calico, Cilium, Flannel, etc.). `kube-proxy` is installed as a DaemonSet for Service routing.
- **Datastore:** a real `etcd` — stacked (co-located with control plane nodes) or external. HA needs an odd number of control plane nodes (3 or 5) for etcd quorum, fronted by a load balancer on the API server.
- **PKI and tokens:** generates a full CA and component certs under `/etc/kubernetes/pki/`; nodes join with a bootstrap token plus the CA cert hash. Certs are time-limited (one year by default) and renewed with `kubeadm certs renew`.
- **Upgrades:** deliberate and version-skew-aware — `kubeadm upgrade plan`/`apply` on the first control plane node, then `kubeadm upgrade node` on the rest, draining and upgrading kubelets one node at a time.

```bash
# Control plane init (pick a pod CIDR that matches your CNI)
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
# Then apply a CNI, e.g. Calico/Flannel, before nodes go Ready
kubeadm token create --print-join-command   # prints the worker join command
```

Reach for kubeadm when you want a production on-prem cluster with full control, or when you want to understand the internals rather than have them abstracted away.

See also: [Kubernetes Cluster Setup with kubeadm](articles/kubeadm-cluster-setup.md) and [How kubeadm Creates a Control Plane](articles/kubeadm-control-plane-creation.md).

## K3s

A CNCF-certified lightweight distribution from SUSE/Rancher that packages the whole control plane into a single binary and process. It swaps some heavy defaults for lighter ones — SQLite as the default datastore instead of etcd, and a bundled set of components (containerd, flannel, Traefik, local-path storage, ServiceLB) — while staying fully Kubernetes-conformant.

- **Strengths:** installs with one command, tiny footprint (runs comfortably on a Raspberry Pi), production-ready for edge and small clusters, and great for CI. HA is supported by switching the datastore to embedded etcd or an external database.
- **Trade-offs:** the opinionated bundled components need extra steps to swap out, and the SQLite default isn't for HA. Less "vanilla" than kubeadm.

### Technical details

- **Architecture:** one `k3s` binary (~70 MB) runs everything. A **server** process wraps the API server, scheduler, and controller-manager; **agents** run the kubelet and kube-proxy. Both embed containerd, so there's no separate runtime to install.
- **Datastore via kine:** K3s uses **kine**, a shim that presents an etcd API to the API server while storing data in SQLite (default), or MySQL/MariaDB/Postgres. For HA it can instead run **embedded etcd** — needing an odd number of servers (3+) for quorum.
- **Bundled components:** flannel (VXLAN) for CNI, CoreDNS, Traefik as the ingress controller, **ServiceLB** (Klipper) as a bare-metal LoadBalancer, **local-path-provisioner** for storage, metrics-server, and **kube-router** for NetworkPolicy enforcement.
- **Disabling defaults:** any bundled add-on can be turned off with flags like `--disable=traefik` or `--disable=servicelb`, or replaced (e.g. Calico/Cilium instead of flannel via `--flannel-backend=none`).
- **Bootstrapping:** a node token (`/var/lib/rancher/k3s/server/node-token`) authenticates agents. Manifests dropped in `/var/lib/rancher/k3s/server/manifests/` are auto-applied, and HelmChart CRDs let you install charts declaratively.
- **Upgrades:** re-run the install script with a new `INSTALL_K3S_VERSION`, or automate rolling upgrades cluster-wide with the **system-upgrade-controller** and upgrade Plans.

```bash
# Server (control plane) — one command
curl -sfL https://get.k3s.io | sh -

# HA with embedded etcd (first server)
curl -sfL https://get.k3s.io | sh -s - server --cluster-init

# Add an agent (worker) node
curl -sfL https://get.k3s.io | K3S_URL=https://<server>:6443 K3S_TOKEN=<token> sh -

# Install without Traefik and the built-in LoadBalancer
curl -sfL https://get.k3s.io | sh -s - --disable=traefik --disable=servicelb
```

Reach for K3s when you want a real, low-overhead cluster fast — edge, IoT, CI, or a homelab.

See also: [hetzner-k3s Cheatsheet](articles/hetzner-k3s-cheatsheet.md).

## MicroK8s

Canonical's lightweight distribution, delivered as a snap package. It's a single-package install with add-ons (DNS, dashboard, storage, ingress, MetalLB, GPU) you enable with `microk8s enable`. It uses dqlite as its datastore and supports HA across multiple nodes.

- **Strengths:** simple install and upgrades on Ubuntu, a clean add-on system, strong on developer workstations and edge, and easy clustering.
- **Trade-offs:** snap-based, which not everyone likes and which is awkward on non-Ubuntu distros. The `microk8s.kubectl` wrapper and snap confinement can feel different from a standard setup.

### Technical details

- **Architecture:** the whole cluster ships inside one snap, with binaries under `/snap/microk8s/current/` and state under `/var/snap/microk8s/`. It runs its own containerd; components are wrapped as snap services (`microk8s.daemon-*`) rather than host packages.
- **Datastore (dqlite):** uses **dqlite** — SQLite plus Raft. A cluster of **three or more nodes automatically becomes HA**, with an elected leader holding the authoritative copy and two replicas. This differs from etcd-based distros and is easier to lose quorum on if you drop below three.
- **Networking:** default CNI is **Calico** with the VXLAN backend (Calico runs as pods in-cluster). `microk8s enable ingress` deploys an NGINX ingress controller; `metallb` provides bare-metal LoadBalancer IPs.
- **Add-on system:** `microk8s enable <addon>` / `disable <addon>` toggles curated components (dns/CoreDNS, hostpath-storage, dashboard, registry, gpu, cert-manager, observability). Add-ons are the intended way to extend the cluster.
- **Channels:** installs track a snap **channel** like `1.30/stable`; `snap refresh microk8s --channel=1.31/stable` moves between minor versions. Snap auto-refresh can upgrade the cluster unattended — pin or hold it in production.
- **CLI and access:** `microk8s kubectl` is the confined wrapper; `microk8s config` exports a standard kubeconfig for your own `kubectl`. `microk8s add-node` / `join` clusters machines.

```bash
sudo snap install microk8s --classic --channel=1.30/stable
sudo microk8s enable dns hostpath-storage ingress
microk8s status --wait-ready
microk8s config > ~/.kube/microk8s.yaml   # standard kubeconfig
# Hold auto-refresh so snap doesn't upgrade the cluster under you
sudo snap refresh --hold microk8s
```

Reach for MicroK8s when you're on Ubuntu, want a batteries-included cluster with toggleable add-ons, and prefer snap-managed upgrades.

See also: [Ingress for the Kubernetes Dashboard on MicroK8s](articles/ingress-kubernetes-dashboard-microk8s.md), [Ingress with MetalLB on MicroK8s](articles/ingress-metallb-microk8s-guide.md), and [NFS Storage for MicroK8s](articles/nfs-microk8s-installation.md).

## Minikube

The Kubernetes project's local-development tool. It runs a cluster in a VM or container via pluggable drivers (Docker, KVM, Hyperkit, VirtualBox, and more) and ships a large set of addons.

- **Strengths:** the friendliest way to learn Kubernetes and develop locally, real driver-based isolation, addons for common needs, and features like `minikube tunnel` and easy image loading.
- **Trade-offs:** heavier than the in-Docker options, primarily single-node in practice, and not meant for production.

### Technical details

- **Drivers:** abstracts the underlying isolation via drivers — `docker` and `podman` (container-based), plus VM drivers (`kvm2`, `hyperkit`, `hyperv`, `qemu`, `virtualbox`). VM drivers give a real kernel and stronger isolation; container drivers are faster but share the host kernel.
- **Under the hood:** provisions a node that runs kubeadm internally to bring up the control plane, with a bundled runtime (containerd, CRI-O, or Docker via `--container-runtime`). So it's genuinely kubeadm-based, just automated.
- **Multi-node and versions:** `--nodes N` creates multiple nodes and `--kubernetes-version` pins an exact version, which makes it handy for reproducing a specific cluster version locally. Profiles (`-p name`) let you run several independent clusters side by side.
- **Addons:** `minikube addons list/enable` manages a curated set (ingress-nginx, metrics-server, dashboard, registry, csi-hostpath, gvisor). These are Minikube-managed, separate from anything you install yourself.
- **Access helpers:** `minikube tunnel` routes LoadBalancer Service IPs to your host, `minikube service <svc>` opens a NodePort, and `minikube image load` / `--driver=docker` sidesteps registry pushes for local images.

```bash
minikube start --driver=docker --nodes=2 --kubernetes-version=v1.30.0
minikube addons enable ingress
minikube image load my-app:dev      # use a local image without a registry
minikube tunnel                     # expose LoadBalancer services on the host
kubectl get nodes
```

Reach for Minikube when you're learning Kubernetes or want an isolated local cluster with a rich addon set.

## kind (Kubernetes IN Docker)

A SIG-maintained tool that runs cluster nodes as Docker containers. It was built to test Kubernetes itself, which makes it excellent for CI: fast to create and tear down, and easy to spin up multi-node clusters on one host.

- **Strengths:** very fast, scriptable, multi-node clusters from a small config file, and the de facto standard for testing manifests and controllers in CI.
- **Trade-offs:** needs Docker, nodes are containers (not VMs) so isolation is weaker, and it's not for production. Loading local images requires `kind load`.

### Technical details

- **Node model:** each "node" is a Docker (or Podman) container running a purpose-built **node image** that bundles kubelet, containerd, and the Kubernetes components — Kubernetes running inside containers, with containerd nested inside each node ("containerd in containerd").
- **Bootstrapping:** brings the cluster up with kubeadm inside the node containers, so control-plane behavior matches upstream. Multi-node and multi-control-plane HA topologies come from a small config file.
- **Version pinning:** you select the Kubernetes version by pinning the node image by digest (`--image kindest/node:v1.30.0@sha256:...`), which makes CI matrices across versions straightforward.
- **Images and networking:** `kind load docker-image` / `load image-archive` injects local images into node containers (no registry needed). Port access uses `extraPortMappings`; ingress needs a controller plus those mappings since there's no cloud LB.
- **Ephemeral by design:** state lives in the containers, so `kind delete cluster` is a clean teardown — ideal for per-job CI clusters.

```yaml
# kind-config.yaml — one control plane, two workers
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --name dev --config kind-config.yaml
kind load docker-image my-app:dev --name dev
kubectl cluster-info --context kind-dev
```

Reach for kind when you need ephemeral clusters in CI pipelines or for testing changes locally.

## k3d

A community wrapper that runs **K3s** inside Docker. Think "kind, but for K3s" — you get K3s's light footprint with kind-style speed and multi-node convenience on a single host.

- **Strengths:** extremely fast create/delete, multi-node and multi-server clusters in seconds, built-in registry support, and the same K3s behavior you'd run at the edge.
- **Trade-offs:** needs Docker, containers-as-nodes (weaker isolation), and not for production.

### Technical details

- **Node model:** wraps the official K3s Docker image, running each server/agent as a container. So you get K3s internals (kine + SQLite, flannel, Traefik, ServiceLB) but with kind-style container nodes instead of hosts or VMs.
- **Load balancer front end:** creates a small **serverlb** container (a managed proxy) in front of the server nodes, so the API server and exposed ports have a single stable entrypoint even with multiple servers.
- **Registry integration:** `k3d registry create` spins up a local registry container and wires it into the cluster, avoiding image pushes to a remote registry during development.
- **Port exposure:** `-p "8080:80@loadbalancer"` maps host ports through the load balancer to Services/ingress — the usual way to reach workloads from the host.
- **Multi-server HA:** `--servers 3` runs embedded-etcd K3s across three server containers to rehearse HA behavior locally; `--agents N` adds workers.

```bash
k3d cluster create dev --servers 1 --agents 2 -p "8080:80@loadbalancer"
k3d registry create dev-registry --port 5000
k3d image import my-app:dev -c dev     # load a local image into the cluster
kubectl get nodes
```

Reach for k3d when you like K3s and want throwaway local clusters that match your edge setup.

## k0s

A single-binary distribution from Mirantis with zero host OS dependencies beyond the kernel — you copy one executable to each host and run it. It's CNCF-certified and aimed at bare-metal, on-prem, edge, and IoT, with a security-conscious posture.

- **Strengths:** truly dependency-free single binary, works on any Linux without extra packages, supports HA, and keeps the control plane isolated from workloads by default.
- **Trade-offs:** smaller community than K3s, and the "assemble it yourself" surface is larger than a batteries-included distro.

### Technical details

- **Single binary, no host deps:** the `k0s` binary bundles the API server, controller-manager, scheduler, kubelet, and containerd, and needs nothing from the host beyond the kernel — no snap, no package manager, no pre-installed runtime.
- **Controller/worker split:** `k0s controller` runs the control plane; `k0s worker` runs workloads. By default controllers do **not** run a kubelet or schedule workloads, keeping the control plane isolated from application pods (you opt in with `--enable-worker`).
- **Datastore:** embedded **etcd** for HA controllers (odd number for quorum), or kine-backed SQLite/MySQL/Postgres for lighter single-node setups.
- **Config-driven:** a declarative `k0s.yaml` defines CNI, storage, extensions, and API settings. Default CNI is **Kube-router** (Calico is also supported), and **Helm charts/manifests** can be declared in config as "extensions" so the cluster reconciles them on start.
- **Lifecycle:** `k0s install` registers a systemd service; **k0sctl** is a separate tool that provisions and upgrades multi-node clusters over SSH from an inventory file.

```bash
# Single-node controller that also runs workloads
sudo k0s install controller --single
sudo k0s start
sudo k0s kubectl get nodes
# Multi-node: define hosts in k0sctl.yaml, then
k0sctl apply --config k0sctl.yaml
```

Reach for k0s when you want a self-contained single binary for edge/bare-metal without pulling in snaps or scripts.

## RKE2

SUSE/Rancher's enterprise-focused distribution (sometimes called RKE Government). It keeps K3s-style simplicity but adds security and conformance layers — FIPS 140-2 compliance, CIS benchmark hardening, and DISA STIG support — with etcd as the datastore.

- **Strengths:** production and regulated-environment ready, hardened defaults, and a good fit for multi-cluster fleets managed by Rancher.
- **Trade-offs:** heavier than K3s and more than a homelab needs. The compliance focus adds complexity you don't want unless you need it.

### Technical details

- **K3s launch model, upstream internals:** RKE2 borrows K3s's simple install and supervisor design, but runs the control plane components as **static Pods** (like kubeadm) rather than in-process, and uses **etcd** as the datastore by default — closer to upstream for conformance.
- **Runtime and CNI:** ships its own containerd. Default CNI is **Canal** (Calico + Flannel); **Cilium** and **Calico** are selectable. Components are deployed and reconciled from manifests under `/var/lib/rancher/rke2/server/manifests/`.
- **Security posture:** built for regulated use — **FIPS 140-2** validated crypto, **CIS Kubernetes Benchmark** hardening (with a `profile: cis` mode and required kernel/sysctl settings), and **DISA STIG** coverage. Ships SELinux support and signed release artifacts.
- **Roles:** `rke2-server` (control plane + etcd) and `rke2-agent` (workers) run as systemd services; HA uses an odd number of servers for etcd quorum behind a fixed registration address.
- **Upgrades:** driven by the **system-upgrade-controller** with upgrade Plans, or through Rancher for fleet-wide managed rollouts.

```bash
# Server (control plane)
curl -sfL https://get.rke2.io | sh -
sudo systemctl enable --now rke2-server.service
# Harden to the CIS profile via config
echo 'profile: "cis"' | sudo tee -a /etc/rancher/rke2/config.yaml
# Agent joins with the server URL + token from /var/lib/rancher/rke2/server/node-token
```

Reach for RKE2 when you run enterprise or government workloads that require FIPS/CIS/STIG compliance and hardened defaults.

## Local Dev vs Production

A useful first cut is "am I learning/testing, or running real workloads?"

- **Local dev / CI / learning:** Minikube (isolation, addons, learning), kind (fast CI, testing), k3d (fast K3s-flavored clusters). None are meant for production.
- **Production / edge / on-prem:** kubeadm (full control), K3s (light, edge), MicroK8s (Ubuntu, add-ons), k0s (single binary, bare-metal), RKE2 (compliance-heavy enterprise).

kind and k3d run their "nodes" as Docker containers, so they share the host kernel and give weaker isolation than VM-based Minikube — fine for testing, not for multi-tenant production.

## Decision Guide

- **You want to learn how Kubernetes actually works** → kubeadm (or Minikube for a gentler start).
- **You want a real cluster fast for edge, IoT, or a homelab** → K3s.
- **You're on Ubuntu and want toggleable add-ons** → MicroK8s.
- **You're developing locally and want isolation plus addons** → Minikube.
- **You need ephemeral clusters in CI** → kind (or k3d if you prefer K3s).
- **You want a dependency-free single binary for bare-metal/edge** → k0s.
- **You need FIPS/CIS/STIG compliance for enterprise or government** → RKE2.
- **You don't want to run the control plane at all** → a managed service (EKS/GKE/AKS), which is outside this list.

## Summary

kubeadm is the upstream, do-it-yourself path and the best teacher of internals. K3s is the standout lightweight distribution for edge, CI, and homelabs, with k3d giving you the same thing as throwaway Docker clusters. MicroK8s is the Ubuntu-friendly, add-on-driven alternative. Minikube and kind cover local development and CI — Minikube for isolated, addon-rich learning, kind for fast ephemeral test clusters. k0s is the zero-dependency single binary for bare-metal edge, and RKE2 is the hardened, compliance-ready choice for enterprise and government. Match the tool to whether you're learning, testing in CI, or running production at the edge or in a datacenter.

For where each of these writes its logs, see [Kubernetes Log Locations by Distribution](articles/kubernetes-log-locations.md).
