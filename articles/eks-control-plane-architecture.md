# How the Amazon EKS Control Plane Is Created and How It Talks to the Outside

When you run `aws eks create-cluster`, you don't get a set of EC2 instances you can SSH
into. You get a **fully managed Kubernetes control plane** that AWS provisions, operates,
and isolates in an AWS-owned account, plus a single **API server endpoint** you talk to.
This article explains what actually gets built, how the pieces fit together, and — the
part that trips people up most — how a control plane running in AWS's account reaches into
*your* VPC and how clients and nodes reach *it*.

The mental model to hold throughout: **the control plane lives in an AWS-managed account;
your nodes and workloads live in your VPC/account; the two are stitched together by
network interfaces and a load-balanced endpoint, with IAM + RBAC gating access.**

---

## 1. What the control plane is made of

Every EKS cluster gets its **own dedicated, single-tenant control plane**. It is not
shared with other clusters or other AWS accounts. When you create a cluster, EKS stands up
the standard Kubernetes control-plane components, but runs and scales them for you:

- **kube-apiserver** — at least **two API server instances**, spread across **three
  Availability Zones** in the Region.
- **etcd** — **three etcd instances** across three AZs (the quorum store for all cluster
  state).
- **kube-controller-manager** and **kube-scheduler** — the control loops and scheduler,
  also AWS-operated.
- **cloud controller / EKS-managed controllers** — the AWS-specific glue.

Key properties of this managed setup:

- **Multi-AZ by design.** Two+ API servers and three etcd nodes across three AZs are what
  back the API server endpoint availability **SLA**.
- **Self-healing.** If a control plane instance degrades, EKS replaces it automatically,
  in a different AZ if needed.
- **Auto-scaling.** EKS monitors and resizes control plane instances to keep performance
  steady as load grows — you don't size the control plane.
- **Network-isolated.** EKS uses a VPC (in the managed account) to restrict traffic
  *between* a cluster's own control plane components. Components in one cluster can't see
  or receive traffic from another cluster or account, except where your Kubernetes RBAC
  explicitly allows it.

You never patch, scale, or back up these components. AWS owns that lifecycle, including
the one-minor-version-at-a-time Kubernetes upgrades you trigger.

---

## 2. What happens when you create a cluster

`CreateCluster` (via `aws eks create-cluster`, the console, eksctl, or Terraform) kicks
off roughly this sequence:

1. **You supply the inputs:** a name, a Kubernetes version, a **cluster IAM role** (the
   role EKS assumes to manage AWS resources on your behalf), and a **VPC config** — the
   subnets and security groups EKS will use to wire the control plane into your network.
2. **EKS provisions the control plane** in its managed account: the API servers, etcd,
   scheduler, and controllers described above, across three AZs.
3. **EKS creates the API server endpoint** — a unique, highly available URL fronted by a
   load balancer. Format depends on IP family and Region, e.g.:
   - IPv4: `eks-<cluster>.<region>.eks.amazonaws.com`
   - Dual-stack (IPv6 clusters after Oct 2024): `eks-<cluster>.<region>.api.aws`
4. **EKS injects network interfaces into your VPC** (Section 3) so the control plane can
   reach node-side components.
5. **EKS creates a cluster access entry** for itself and for you, and sets up the auth
   path (IAM → Kubernetes). The creating principal is granted cluster-admin.
6. The cluster transitions to **`ACTIVE`**. At that point the API server answers requests;
   you then attach compute (managed node groups, Fargate, Karpenter, Auto Mode, or hybrid
   nodes) separately.

Check progress with:

```bash
aws eks describe-cluster --name my-cluster --region us-west-2 \
  --query 'cluster.{status:status,version:version,endpoint:endpoint,
           roleArn:roleArn,vpc:resourcesVpcConfig}'
```

Note there is **no data plane yet** at `ACTIVE`. A brand-new cluster is a control plane and
an endpoint; nodes are added afterward and register *into* it.

---

## 3. How a control plane in AWS's account reaches into your VPC

This is the crux. The API server needs to initiate connections **to** node-side components
— to serve `kubectl exec`, `kubectl logs`, `kubectl port-forward`, and to call **admission
webhooks** and **extension API servers** running as pods. Those live in *your* VPC. So EKS
places network interfaces into your VPC on the control plane's behalf.

### Cross-account ENIs (X-ENIs)

Into the subnets you specified at create time, EKS provisions **elastic network interfaces
owned by the EKS managed account but living in your VPC** — commonly called
**cross-account ENIs (X-ENIs)**. These give the AWS-hosted control plane a presence
*inside* your network so it can open connections to kubelets and webhook pods.

- The **IPs of these control plane ENIs** are published in the `kubernetes` **`Endpoints`**
  object in the `default` namespace. As EKS adds or removes ENIs (scaling, AZ replacement),
  it keeps that object current.
- For **IPv6/dual-stack** clusters, these are dual-stack ENIs so control-plane↔data-plane
  traffic can flow over IPv6.
- Traffic on these ENIs is governed by the **cluster security group** EKS creates, plus any
  additional security groups you attached in the VPC config.

### The `kubernetes` Service — how pods reach the API server

The reverse direction (pods calling the API server) uses a well-known indirection:

- The `kubernetes` **Service** (type `ClusterIP`, `default` namespace) always takes the
  **first IP of the cluster's service CIDR** — e.g. `172.16.0.1` for a `172.16.0.0/16`
  service CIDR.
- A pod sends API requests to that service IP; **kube-proxy** sets up the translation that
  maps it to the actual control plane ENI IPs from the `Endpoints` object.
- This is how in-cluster clients (controllers, operators, and pods using in-cluster config)
  reach the API server without knowing the ENI IPs directly.

### Control plane egress routing (advanced)

By default EKS manages the network path from the control plane to your VPC resources. If
you need to control that path yourself (for example, to steer egress to webhook servers or
OIDC providers through specific routing), EKS supports **control plane egress routing**
(`controlPlaneEgressMode=CUSTOMER_ROUTED`). Most clusters don't need this; it's for
environments with strict egress-path requirements.

> Practical consequence: a **broken or unreachable admission webhook** can add latency to —
> or outright block — API requests, because the control plane (over the X-ENIs) has to call
> your webhook pod on the write path. This is a common cause of "the whole cluster stopped
> accepting changes" incidents.

---

## 4. How the outside world reaches the API server (endpoint access)

The API server endpoint created in Section 2 can be exposed three ways. This is the
**endpoint access** setting, and it controls *who can reach the API server*, orthogonally
to *who is authorized* (IAM + RBAC).

Every request is authenticated with **IAM** (via the AWS IAM Authenticator flow / cluster
access entries) and authorized with Kubernetes **RBAC** — regardless of which access mode
you pick. Endpoint access is purely a network-reachability control.

### The three modes

| Public | Private | Behavior |
|---|---|---|
| **Enabled** | **Disabled** (default) | API server reachable from the internet. Node→control-plane traffic leaves the VPC but stays on Amazon's network. You can restrict the public side with **public access CIDRs**. |
| **Enabled** | **Enabled** | Internet-reachable *and* in-VPC traffic uses the **private endpoint**. Node→control-plane traffic stays in the VPC; you gate it with the **cluster security group**. Public side can still be CIDR-restricted. |
| **Disabled** | **Enabled** | Fully private. All API traffic must originate from within the VPC or a connected network (VPN/Direct Connect/peering). No internet path to the API server. |

### How private access works under the hood

When you enable **private access**, EKS creates a **Route 53 private hosted zone**,
associated with your cluster VPC, that resolves the cluster endpoint to the **private ENI
IPs** instead of public ones. Requirements:

- The VPC must have `enableDnsHostnames` and `enableDnsSupport` set to `true`.
- The DHCP options set must include `AmazonProvidedDNS`.

This private hosted zone is EKS-managed and **won't appear** in your Route 53 resources.
Likewise the private endpoint is **not** a normal PrivateLink/VPC endpoint, so it doesn't
show up in the VPC console's endpoint list.

### Public access CIDRs

With the public endpoint enabled you can lock it down to a list of source **CIDR blocks**
(`publicAccessCidrs`). If you restrict it, make sure the CIDRs include wherever your nodes
and Fargate pods egress from to reach the public endpoint — or also enable the private
endpoint so node traffic uses the in-VPC path.

### Configure or change it

```bash
# At create time (excerpt)
aws eks create-cluster --name my-cluster --region us-west-2 \
  --resources-vpc-config \
    subnetIds=subnet-aaa,subnet-bbb,endpointPublicAccess=true,endpointPrivateAccess=true,publicAccessCidrs=203.0.113.0/24

# Change endpoint access on an existing cluster
aws eks update-cluster-config --name my-cluster --region us-west-2 \
  --resources-vpc-config endpointPublicAccess=false,endpointPrivateAccess=true
```

> Hybrid nodes caveat: because hybrid nodes run *outside* your VPC, they resolve the
> cluster endpoint to public IPs. AWS recommends using **either** public **or** private
> access for clusters with hybrid nodes, not both.

---

## 5. The full request path, end to end

Putting the directions together:

**You / CI running `kubectl`:**
`kubectl` → resolves the cluster endpoint (public IP, or private ENI IP via the EKS-managed
Route 53 zone) → hits the **load-balanced API server endpoint** → IAM authenticates →
RBAC authorizes → request served by one of the API server instances.

**A node's kubelet:**
kubelet → cluster endpoint (public or private per your config) → API server. Nodes register
and are then driven by the control plane.

**A pod using in-cluster config:**
pod → `kubernetes` Service IP (first IP of the service CIDR) → kube-proxy translates to a
control plane **ENI IP** → API server.

**The control plane calling back into your VPC (exec/logs/port-forward, webhooks):**
API server → over the **cross-account ENIs** in your subnets → kubelet on the node, or the
webhook/extension-API pod → response back to the control plane.

---

## 6. What you own vs what AWS owns

| Concern | AWS (managed account) | You (your account/VPC) |
|---|---|---|
| API servers, etcd, scheduler, controllers | ✅ provision, scale, patch, back up | — |
| API server endpoint + load balancer | ✅ create and operate | choose access mode |
| Cross-account ENIs (X-ENIs) | ✅ own and manage | provide the subnets + security groups |
| Route 53 private hosted zone | ✅ create and manage | provide a compliant VPC (DNS settings) |
| Kubernetes version upgrades | ✅ perform the control-plane upgrade | trigger it; prep workloads/nodes |
| Nodes / data plane | — (except Auto Mode / Fargate / hybrid offload) | ✅ manage (unless you use a managed option) |
| Auth | ✅ IAM authenticator plumbing | define RBAC + access entries |

---

## 7. Quick reference

| Goal | Command / fact |
|---|---|
| Create a cluster | `aws eks create-cluster --name <c> --role-arn <role> --resources-vpc-config ...` |
| Check status / endpoint / VPC config | `aws eks describe-cluster --name <c> --query 'cluster.{status:status,endpoint:endpoint,vpc:resourcesVpcConfig}'` |
| Control plane version | `aws eks describe-cluster --name <c> --query 'cluster.version'` |
| Change endpoint access | `aws eks update-cluster-config --name <c> --resources-vpc-config endpointPublicAccess=...,endpointPrivateAccess=...` |
| Control plane ENI IPs (in-cluster) | `kubectl -n default get endpoints kubernetes -o yaml` |
| API server Service IP | first IP of the service CIDR (e.g. `10.100.0.1`), via the `kubernetes` Service |
| Number of control plane instances | ≥2 API servers + 3 etcd across 3 AZs (AWS-managed) |

---

## Summary

- EKS gives every cluster a **dedicated, single-tenant, multi-AZ control plane** (≥2 API
  servers, 3 etcd across 3 AZs) that AWS provisions, scales, heals, and patches in its own
  account.
- `CreateCluster` builds the control plane, a **load-balanced API server endpoint**, and
  injects **cross-account ENIs** into the subnets you supply so the AWS-hosted control
  plane can reach node-side components in **your** VPC.
- Pods reach the API server via the `kubernetes` **Service** (first service-CIDR IP →
  kube-proxy → ENI IPs); the control plane reaches pods/kubelets back over the **X-ENIs**.
- **Endpoint access** (public / private / both) controls network reachability; **IAM +
  RBAC** always control authorization. Private access uses an EKS-managed **Route 53
  private hosted zone**.
- You own the VPC, subnets, security groups, nodes, and RBAC; AWS owns everything inside
  the managed control plane.

---

### Sources

- [Amazon EKS architecture (control plane, compute, capabilities)](https://docs.aws.amazon.com/eks/latest/userguide/eks-architecture.html)
- [Cluster API server endpoint — public and private access](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html)
- [EKS control plane in the VPC — control plane ENIs and the `kubernetes` Service](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-concepts-kubernetes.html)
- [Configuring control plane egress routing](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-egress.html)
- [EKS API Reference: CreateCluster](https://docs.aws.amazon.com/eks/latest/APIReference/API_CreateCluster.html)
- [Amazon EKS best practices: control plane](https://docs.aws.amazon.com/eks/latest/best-practices/control-plane.html)
