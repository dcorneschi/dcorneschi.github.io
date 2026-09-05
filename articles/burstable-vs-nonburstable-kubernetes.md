# Burstable vs. Non-Burstable: Which AWS Instance Type Is a Better Pick for Kubernetes?

When you pick the EC2 instance type for a Kubernetes node group, one of the first forks in
the road is **burstable (T-family: `t3`, `t3a`, `t4g`) vs. non-burstable (M/C/R and
friends)**. Burstable instances are cheaper and look attractive on paper, but they behave
very differently under sustained load — and Kubernetes has its own opinions about CPU that
interact badly with CPU credits if you're not careful.

This article explains how each type behaves, where burstable nodes bite you on a cluster,
and a practical decision framework for choosing between them.

> **TL;DR:** Burstable nodes are great for small, spiky, or dev/test clusters and idle
> system workloads. For production data planes running steady or latency-sensitive
> services, non-burstable instances give you predictable performance and are usually worth
> the extra cost. The danger with burstable is **CPU credit exhaustion** — a node that
> silently throttles to its baseline mid-incident.

---

## How burstable (T-family) instances work

Burstable instances provide a **low baseline CPU allocation** and earn **CPU credits** when
they run below baseline. When the workload needs more, they spend credits to burst up
toward 100% of a vCPU. Run hard for long enough and the credits run out.

- **Baseline is a fraction of a vCPU**, and it varies by size — roughly **30% for
  `t3.large`, 40% for `t3.xlarge`/`t3.2xlarge`** (per vCPU, and lower for smaller sizes
  like `t3.medium`). The rest is only available while you have credits.
- **One CPU credit = one vCPU-minute of full performance.** Concretely, a credit buys
  100% of a core for 1 minute, or 25% of a core for 4 minutes, or 10% for 10 minutes, and
  so on.
- **Credits accrue over time** at a fixed rate and cap at a maximum balance. Larger T
  instances earn credits faster.
- **Stop/start and reboot cost you credits.** A stopped **T2** instance **loses its whole
  credit balance immediately**; **T3/T4** retain accrued credits for **7 days** before
  losing them. This matters for autoscaled/recycled nodes that come and go.
- **Two modes when credits run out:**
  - **Standard mode:** the instance is **throttled back to baseline**. CPU-bound work
    slows down, latency climbs.
  - **Unlimited mode** (default on many T3 launches): the instance keeps bursting and you
    pay a **surcharge** for sustained CPU above baseline. This can quietly erase the cost
    savings — a T instance in unlimited mode running hot can cost *more* than an equivalent
    M instance.

The key trap: performance is a function of **credit balance**, which is a function of
*recent* history. Two identical nodes can perform very differently depending on what they
did five minutes ago.

## How non-burstable instances work

M (general purpose), C (compute optimized), R (memory optimized), and similar families
deliver their **full rated vCPU performance continuously**. No credits, no baseline, no
throttling surprise. You pay a flat hourly rate for consistent capacity.

- Predictable CPU under sustained load.
- No hidden surcharge for running hot.
- Higher baseline cost than a same-size T instance.

## Why this matters more on Kubernetes than on a single VM

A Kubernetes node isn't running one workload — it's a shared, densely-packed host, and the
scheduler and kubelet make decisions based on capacity that **burstable credits don't
model.**

- **The scheduler sees vCPUs, not credits.** Kubernetes schedules Pods based on CPU
  **requests** against the node's *nominal* allocatable CPU (e.g. 2 vCPUs on a
  `t3.medium`). It has **no idea** the node can only sustain ~0.4 vCPU once credits are
  gone. You can pack a node "correctly" by requests and still have it throttle hard.
- **Bin-packing concentrates load.** Cluster Autoscaler / Karpenter and the scheduler try
  to fill nodes efficiently. A well-packed burstable node is *more* likely to run above
  baseline continuously and burn credits fast.
- **System components need CPU too.** kubelet, the CNI (e.g. `aws-node`/VPC CNI),
  kube-proxy, CoreDNS, and log/metric agents all consume CPU. On a credit-starved node,
  even these can be starved, which shows up as **`NotReady` nodes, failing health checks,
  and DNS timeouts** — not just slow app Pods.
- **Noisy-neighbor blast radius.** One Pod burning credits degrades **every** Pod on that
  node, because the throttle is applied at the instance level.

### The CPU-limits interaction

Kubernetes CPU **limits** are enforced by the Linux CFS quota at the cgroup level, which
throttles a container to its limit regardless of spare CPU. Stack that on top of
instance-level credit throttling and you get **two independent throttles** — a container
can be CFS-throttled *and* the whole node can be credit-throttled at the same time. This
makes performance debugging genuinely confusing. (See the companion article on CPU
throttling with available CPU for the cgroup side.)

## Symptoms of credit exhaustion on a cluster

Watch for these — they often appear together during a traffic spike:

- Latency and error rates climb on a node while CPU **utilization looks capped** (pinned at
  baseline, not 100%).
- `CPUCreditBalance` in CloudWatch trending toward **zero**.
- `CPUSurplusCreditBalance` growing (you're in unlimited mode and being charged).
- Nodes flapping `Ready`/`NotReady`, kubelet PLEG warnings, CoreDNS timeouts.
- The same deployment performing inconsistently across nodes of the same type.

Monitor `CPUCreditBalance` and `CPUCreditUsage` per node and alarm when the balance drops
below a threshold — this is the single most important signal for a burstable data plane.

## When burstable (T-family) is the better pick

- **Dev, test, and CI clusters** with intermittent activity.
- **Small clusters / control-plane-adjacent nodes** running mostly idle system services.
- **Spiky, low-average workloads** — bursty web front ends, cron-driven jobs, batch that
  runs briefly then idles, giving credits time to recharge.
- **Cost-sensitive, non-critical** environments where occasional throttling is acceptable.
- **Learning / homelab-style** EKS clusters.

Rule of thumb: if **average** CPU is well below baseline and bursts are short, T-family in
**standard** mode is a genuine bargain.

## When non-burstable is the better pick

- **Production data planes** serving steady or latency-sensitive traffic.
- **CPU-bound workloads** (encoding, compression, ML inference, big JVM/GC apps) that run
  near capacity for long stretches.
- **Databases and stateful services** where consistent I/O and CPU matter.
- **Anything running > baseline CPU for sustained periods** — do the math; unlimited-mode
  surcharges often make an M instance cheaper *and* faster.
- **Latency-SLO workloads** where a credit-exhaustion throttle would breach the SLO.

Rule of thumb: if **average** CPU approaches or exceeds the T baseline, or the workload is
latency-critical, choose M/C/R.

## A quick decision framework

| Question | Lean burstable (T) | Lean non-burstable (M/C/R) |
|----------|--------------------|-----------------------------|
| Average CPU vs. baseline | Well below baseline | Near or above baseline |
| Traffic shape | Spiky with idle gaps | Steady / sustained |
| Latency sensitivity | Tolerant | Strict SLO |
| Environment | Dev/test/CI/homelab | Production |
| Cost priority vs. predictability | Cost first | Predictability first |
| Workload type | Idle system services, light web | Databases, CPU-bound, stateful |

**Cost sanity check:** estimate sustained CPU %. If a T instance would spend most of its
time above baseline (paying unlimited-mode surcharges), price the equivalent M instance —
it's frequently cheaper at that duty cycle, with none of the throttling risk. Concrete
data points to calibrate on (AWS list pricing, Linux): unlimited-mode surplus credits cost
about **$0.05 per vCPU-hour**, and a published
[CAST AI analysis](https://cast.ai/blog/burstable-vs-non-burstable-which-aws-instance-type-is-a-better-pick-for-kubernetes/)
found that **above ~42.5% sustained CPU, `m6i.large` is cheaper than `t3.large`**, and that
leaving `t3.large` in unlimited mode running hot cost roughly **48% more than an equivalent
`m5.large`**. In other words, the "15% cheaper" headline for T instances only holds while
you stay under baseline. *Content was rephrased for compliance with licensing
restrictions.*

Also worth knowing: idle credit reserves drain fast under real load. A T instance that sat
idle for a day can burn through its accumulated credits in a few hours once a workload
starts pushing it above baseline.

## Practical guidance for EKS node groups

- **Separate node groups by workload class.** Put idle system add-ons and dev workloads on
  a T-family node group; put production services on M/C/R node groups. Use
  labels/taints/`nodeSelector` (or Karpenter NodePools) to steer Pods.
- **Always set CPU `requests`** so the scheduler bin-packs sanely — but remember requests
  don't protect you from credit exhaustion on T nodes.
- **Be cautious with CPU `limits` on burstable nodes.** Combining CFS throttling with
  credit throttling amplifies latency; many teams set requests and omit CPU limits for
  latency-sensitive services.
- **Decide standard vs. unlimited deliberately.** Unlimited protects performance but can
  surprise you on cost; standard protects cost but throttles under sustained load. Pick per
  node group, and monitor `CPUSurplusCreditBalance` if you use unlimited.
- **Alarm on `CPUCreditBalance`** per node (or use node-level dashboards) so you see
  exhaustion before users do.
- **Consider Graviton (`t4g`, and `m7g`/`c7g`/`r7g`)** for price/performance regardless of
  burstable vs. not — often the bigger cost lever than the T-vs-M decision itself.
- **Watch Karpenter's defaults.** If a NodePool doesn't constrain instance types,
  Karpenter can select **T-family (e.g. `t3`) instances** as cheap options — landing
  burstable nodes in your data plane unintentionally. Constrain the NodePool
  `requirements` (e.g. `karpenter.k8s.aws/instance-category In [m, c, r]`) for production
  pools so you don't get surprise burstable capacity.

## Key takeaways

- Burstable = cheap baseline + credits; **throttles to baseline** (standard) or **charges a
  surcharge** (unlimited) when credits run out.
- Kubernetes schedules on **nominal vCPUs and requests**, which **do not model CPU
  credits** — so a "correctly packed" burstable node can still throttle hard.
- Credit exhaustion has a **node-wide blast radius**: slow Pods, `NotReady` nodes, DNS
  timeouts.
- Use T-family for spiky/dev/idle workloads; use M/C/R for steady, CPU-bound, or
  latency-sensitive production.
- Split node groups by workload class, monitor `CPUCreditBalance`, and price out unlimited
  mode before assuming burstable is cheaper.

---

### Sources

- [Burstable performance instances (Amazon EC2)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances.html)
- [Key concepts and definitions for burstable performance instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances-concepts.html)
- [Unlimited mode for burstable performance instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances-unlimited-mode.html)
- [CloudWatch metrics for burstable instances (CPU credits)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances-monitoring-cpu-credits.html)
- [Kubernetes: CPU management and CFS quota](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/)
