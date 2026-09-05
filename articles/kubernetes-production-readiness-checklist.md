# Kubernetes Production Readiness: A Practical Checklist

Getting a Pod to *start* is easy. Getting a workload to survive rollouts, node drains, zone
failures, traffic spikes, and 3 a.m. incidents is a different problem. This article
distills a production-readiness checklist into five areas — **your application, your
manifests, your security, scaling, and going live** — with the reasoning behind each check
and the concrete Kubernetes fields involved.

It's organized as a checklist you can walk top to bottom. Nothing here is exotic; the value
is in doing all of it *before* production rather than discovering the gaps under load.

---

## 1. Your application

The runtime contract your app must satisfy inside a container. If these are wrong,
Kubernetes will still start the Pod — but updates, replacements, and scaling will be
fragile.

### Application behavior

- **Log to stdout/stderr.** Use *passive* logging: the app writes to standard streams and
  the platform (node agents → central store) collects, enriches, and forwards them. Avoid
  *active* logging (app pushing directly to Elasticsearch/third parties) — it couples your
  app to the logging system's availability. Make logs **structured** (JSON), not plain
  text, so they're searchable and correlatable with metrics/traces. `kubectl logs` is for
  debugging, not long-term storage.
- **Read config from env vars or files**, not baked into the image. Use **ConfigMaps** for
  non-sensitive config (env vars for scalars via `configMapKeyRef`/`envFrom`; mounted files
  when the app expects a config file), and **Secrets** for sensitive values. Remember a
  single ConfigMap/Secret is capped at **~1 MiB** (etcd limit) — it's not file storage.
- **Handle SIGTERM and shut down gracefully.** On termination the kubelet sends the stop
  signal to PID 1, then waits `terminationGracePeriodSeconds` before SIGKILL. The app
  should: stop accepting new requests → drain in-flight work → close keep-alive connections
  → exit before the grace period. Use the **exec form** of `CMD`/`ENTRYPOINT`
  (`CMD ["node","server.js"]`) so signals reach the process; if you use a wrapper script,
  `exec` into the app. A `preStop` hook helps with small predictable actions but doesn't
  replace signal handling (and its time counts against the same grace period).
- **Expose health signals.** Give the kubelet something to check — an HTTP endpoint
  (status 200–399 = success; body ignored; kubelet reads at most 10 KiB), TCP listener,
  exec command, or gRPC health protocol. Match the signal to the decision: *readiness* =
  "should I get traffic now?", *liveness* = "am I stuck and unrecoverable?", *startup* =
  "have I finished initializing?".
- **Don't store state on local disk.** Container local storage is **ephemeral** — gone when
  the container stops or the Pod moves. Local disk is fine for scratch/cache; permanent data
  belongs in a PersistentVolume or external store (DB, object store). For true stateful
  workloads with stable identity/storage, use a **StatefulSet**. Open TCP connections are
  lost on replacement too.
- **Handle long-lived connections.** gRPC, WebSockets, HTTP keep-alive, HTTP/2, and DB
  pools stick to one Pod, so scaling out won't rebalance existing connections. Clients
  should reconnect safely; servers should drain connections cleanly on shutdown (traffic
  can still reach terminating Pods briefly via EndpointSlice `terminating` state).

### Container image

- **Include only what's needed to run** — use multi-stage builds so build tools, package
  managers, and test files don't ship in the runtime image.
- **Use stable tags; avoid `:latest`.** Pin a clear version tag, or a full
  `sha256` **digest** (`myimage@sha256:...`) when you need an immutable reference. `:latest`
  makes "what's running?" and rollback ambiguous.

---

## 2. Your Kubernetes manifests

### Runtime contract

- **Define readiness, liveness, and startup probes.** Startup gates the other two for
  slow-starting apps. Readiness failure removes the Pod from Service endpoints (no restart);
  liveness failure restarts the container. Be conservative with liveness — too-strict
  settings restart Pods that would have recovered. Tune `initialDelaySeconds`,
  `periodSeconds`, `timeoutSeconds`, `failureThreshold`.
- **Set resource requests and limits.** **Requests** are a *scheduling* input (where the
  Pod fits); **limits** are *runtime* enforcement. Over-CPU throttles; over-memory gets you
  OOM-killed. So: **CPU + memory requests, and a memory limit** on every container (CPU
  limit is workload-dependent). Requests+limits determine the **QoS class** (Guaranteed /
  Burstable / BestEffort), which drives eviction order. QoS ≠ **PriorityClass** (a separate
  scheduling/eviction signal).
- **Bound ephemeral storage.** The writable layer, logs, and disk-backed `emptyDir` count
  as local ephemeral storage; runaway usage gets Pods evicted. Set
  `ephemeral-storage` requests/limits, and give writable paths an `emptyDir` with a
  `sizeLimit` — pairs well with `readOnlyRootFilesystem: true`.

### Rollouts and configuration

- **Make rolling-update settings explicit.** Set `maxUnavailable`, `maxSurge`,
  `minReadySeconds`, `progressDeadlineSeconds`, `revisionHistoryLimit` rather than relying
  on defaults. These lean on readiness probes to know when a new Pod can take traffic.
- **Tolerate old + new Pods running together.** `RollingUpdate` (default) runs both
  versions behind one Service — your data formats and APIs must be forward/backward
  compatible during the window. If they can't coexist, use `Recreate` (accepts downtime to
  avoid mixed versions).
- **Have a ConfigMap/Secret reload strategy.** Env-var-injected config is read at start —
  changes need a Pod restart (versioned names or a template annotation to force a new
  ReplicaSet). Volume-mounted config updates live (with kubelet sync delay), except
  `subPath` mounts. Watch out: projected files update via **symlinks**, which naive file
  watchers miss. Use immutable ConfigMaps/Secrets when config should never change.

### Placement and disruption

- **Run least-privilege containers.** `runAsNonRoot: true` (and a non-zero `runAsUser`),
  `readOnlyRootFilesystem: true` (mount an `emptyDir` for writable paths),
  `allowPrivilegeEscalation: false`, `capabilities.drop: ["ALL"]` (add back only what's
  needed), and `seccompProfile.type: RuntimeDefault`. User namespaces
  (`hostUsers: false`) add isolation but aren't a license to run as root.
- **Define a PodDisruptionBudget.** A PDB protects against **voluntary** disruptions (node
  drains) via `minAvailable`/`maxUnavailable`. It does **not** stop node crashes, and does
  **not** limit Deployment/HPA replica changes. Don't set it so strict that drains can never
  make progress.
- **Spread Pods across nodes and zones** with `topologySpreadConstraints`
  (`topologyKey`, `maxSkew`, `whenUnsatisfiable`). A strict `DoNotSchedule` can leave Pods
  `Pending` if a zone lacks capacity. Verify the `topologyKey` exists on eligible nodes and
  the `labelSelector` matches the Pod template — and test under failure/scale-out.

### Secrets and metadata

- **Mount Secrets as volumes, not env vars.** (Env vars are easier to leak via logs/child
  processes and don't update live.)
- **Give resources meaningful labels** (owner, app, version) for selection and ops.
- **Use API versions the target cluster supports** — scan manifests before upgrades (ties
  into deprecated-API checks).

---

## 3. Your security

### Runtime access controls

- **Enforce Pod Security Standards at the namespace level.** Pod Security Admission (which
  replaced PodSecurityPolicy, removed in **1.25**) offers three profiles: `privileged`
  (system components only), `baseline` (blocks the obvious escalations; most apps pass),
  and `restricted` (the production target — non-root, seccomp, tight capabilities). Roll it
  out safely: start with `warn`/`audit: restricted` while `enforce: baseline`, fix
  findings, then move `enforce` to `restricted`.
- **Dedicated ServiceAccount per workload, minimal RBAC.** Don't share the namespace
  `default` SA. Most app Pods need **zero** API access — a SA with no RoleBindings. Start
  from nothing and add permissions one at a time. Disable token auto-mount
  (`automountServiceAccountToken: false`) when the workload doesn't call the API.
- **Restrict traffic with NetworkPolicy.** The default flat network is too open for prod.
  Define ingress (who can reach this workload) and egress (what it can reach), and consider
  **default-deny** — but remember to allow DNS. Policies are additive, and only work if the
  **CNI enforces them** (and they don't understand domain names, only selectors/IP blocks).

### Supply chain and admission control

- **Scan images and pull from trusted registries.** Scan at build and continuously in the
  registry (Trivy, Grype, or cloud-native scanners as backstop). Decide up front which CVE
  severity blocks a release. Pull only from registries you control/approve.
- **Validate every manifest at admission.** RBAC and PSS can't answer rules like "images
  must come from our registry" or "every Deployment needs an owner label." Use native
  **ValidatingAdmissionPolicy** (CEL) for object-local checks; reach for **Kyverno**
  (YAML policies, also mutation/generation/image-verification) or **OPA/Gatekeeper** (Rego,
  cross-resource and reusable) when you need more than local validation.

### Cloud access and secrets

- **Use workload identity for cloud resources**, not fixed access keys — **IRSA / EKS Pod
  Identity** (AWS), **Workload Identity** (GCP), **Entra Workload ID** (Azure). The Pod gets
  a short-lived, scoped credential tied to its SA; a leak expires in minutes.
- **Keep secrets in an external secret store.** Kubernetes Secrets are a delivery
  mechanism, not a secret manager (base64 ≠ encryption; anyone with `get secrets` can read
  them). Keep the source of truth in Vault / AWS Secrets Manager / GCP Secret Manager /
  Azure Key Vault, and bridge in with **External Secrets Operator** (syncs into K8s Secrets)
  or the **Secrets Store CSI Driver** (mounts directly, no K8s Secret object). The workload
  authenticates to the store via its workload identity.

---

## 4. Scaling

### Scaling model

- **Prefer horizontal scaling** for stateless apps — **HPA** (CPU/memory/request rate) or
  **KEDA** for event-driven signals (queue length, Kafka lag, and it can scale to zero,
  which HPA can't). Before scaling out, confirm the app: doesn't store local state, handles
  long-lived connections, and shuts down gracefully.
- **Set explicit autoscaler bounds and scale-down behavior.** `minReplicas` protects
  baseline availability/cold-start; `maxReplicas` protects downstream deps, capacity, and
  cost. Use `behavior.scaleDown.stabilizationWindowSeconds` (e.g. 300s) to avoid flapping.
  For KEDA `ScaledObject`, set `minReplicaCount`/`maxReplicaCount`/`pollingInterval`/
  `cooldownPeriod`.
- **Treat vertical scaling as a targeted option, not a default.** Right for single-replica
  stateful DBs, big-heap JVMs, or over-provisioned workloads. Historically disruptive
  (VPA recreated Pods), but **In-Place Pod Resize** (GA in **1.35**) plus **VPA 1.4+
  `InPlaceOrRecreate`** (beta in 1.35) can resize live. Rule: **VPA and HPA must not control
  the same metric** — e.g. VPA on memory, HPA on CPU/custom.

### Resource pressure

- **Base requests on real usage data.** Deploy with a best guess → run under representative
  load → observe a few days (Prometheus, `kubectl top`, VPA in **Off** mode) → update. Size
  for the **peak**, not the average — especially memory, where a spike above the request
  gets you OOM-killed.
- **Express survival with PriorityClass.** Under node pressure the kubelet evicts by QoS +
  age by default, which isn't what you want. Define classes (e.g. `production-critical` high
  value, `batch` low) and set `priorityClassName` so customer-facing workloads outlive batch
  jobs.

### Traffic and validation

- **Scale-down must drain cleanly.** HPA scale-down terminates Pods like a drain — SIGTERM,
  grace period, kill. A weak SIGTERM handler drops requests, magnified when many Pods stop
  during a spike. Ensure a short post-SIGTERM serving window, clean connection close, and a
  `terminationGracePeriodSeconds` long enough for the slowest in-flight request. **Don't**
  rely on a PDB to slow HPA scale-down (PDBs cover voluntary evictions, not replica
  changes).
- **Load-test the scaling path.** Autoscaling is a control loop with lag — HPA checks every
  ~15s, metrics take time, new Pods need to start and become ready, so ~30–90s from spike to
  serving. Ramp traffic realistically, measure p99/errors through the ramp, confirm scaling
  actually happens and scale-down drains. If traffic outruns the autoscaler: pre-scale
  before known spikes, lower the target utilization, or use a faster signal (KEDA on queue
  length).

---

## 5. Going live

### Visibility

- **Make application health visible.** Three layers: **metrics** (start here — Prometheus /
  OpenTelemetry; **USE** for infra, **RED** for services), **logs** (ship off-node with
  Fluent Bit / Vector / Alloy before the node dies; collect app *and* cluster logs; enable
  the **audit log** early), **traces** (find where latency goes).
- **Collect Kubernetes Events.** They're often the fastest explanation for "why isn't this
  running?" (failed scheduling, `ImagePullBackOff`, probe failures, mount failures,
  evictions, stalled rollouts) — but the API server keeps them only **~1 hour** by default.
  Forward them to your log/metric store (kubernetes-event-exporter, cloud integrations).
  The runbook should say where to find recent Events per Deployment/Pod/HPA/Service.

### Recovery

- **Decide rollback vs. roll-forward up front.** Rollback (`kubectl rollout undo`, or a Git
  revert under GitOps) is fast/safe for config or code bugs with no data changes.
  Roll-forward is the *only* option when the bad version did something irreversible (schema
  change, new data format, queue writes). Keep a running workload traceable to its source
  revision (commit-SHA tags, `app.kubernetes.io/version`, GitOps). `revisionHistoryLimit`
  defaults to 10.
- **Know what happens when a Pod crashes or a node dies — and test it.** Verify: >1 replica
  actually landing on different nodes/zones (three replicas on one node = one replica's
  worth of protection); the PDB is tight enough to protect the minimum but loose enough to
  drain; OOMKills/non-zero exits restart *and show up in metrics/logs* (silent crash loops
  are the worst); lost-node Pods reschedule within the expected window. Test with real
  failures — delete a busy Pod, drain a node, block a zone.

### Runbooks and cost

- **Write a troubleshooting runbook.** Cover the common Pod states (`Pending`,
  `CrashLoopBackOff`, `ImagePullBackOff`, `OOMKilled`, `Error`, `Completed`) and what to
  check for each; where logs are (`kubectl logs --previous` for the crashed container);
  `kubectl describe` (events at the bottom first); `kubectl exec` / `kubectl debug`;
  rollback/roll-forward steps; and escalation.
- **Review cost and right-size.** After a week or two, compare actual CPU/memory to
  requests (using ~20% of request → too high), peak traffic to replica count, and cluster
  node utilization (<50% most of the time → something's over-provisioned). VPA in **Off**
  mode gives continuous recommendations; FinOps tools (OpenCost, cloud cost explorer)
  translate usage to dollars.

---

## Condensed checklist

**Application:** stdout/stderr + structured logs · config in ConfigMap/Secret · SIGTERM
graceful shutdown (exec-form entrypoint) · readiness/liveness/startup signals · no local
state · long-lived connection handling · minimal image · pinned tag/digest.

**Manifests:** three probes · CPU+mem requests + mem limit (QoS aware) · bounded ephemeral
storage · explicit rolling-update fields · mixed-version tolerance · config reload strategy
· non-root / read-only FS / dropped caps / seccomp · PodDisruptionBudget ·
topologySpreadConstraints · Secrets as volumes · labels · supported API versions.

**Security:** Pod Security Standards (`restricted`) · dedicated SA + minimal RBAC + no token
automount · NetworkPolicy (default-deny + DNS) · image scanning + trusted registry ·
admission policies (CEL/Kyverno/OPA) · workload identity · external secret store.

**Scaling:** HPA/KEDA for stateless · explicit bounds + scale-down window · VPA only where
it fits (never same metric as HPA) · requests from real usage (size for peak) ·
PriorityClass · clean scale-down drain · load-tested scaling path.

**Going live:** metrics/logs/traces + audit log · forward Events · rollback vs roll-forward
posture · tested Pod/node/zone failure · troubleshooting runbook · cost review /
right-sizing.

---

### Sources

- [Pod Security Standards (Kubernetes docs)](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Configure Liveness, Readiness and Startup Probes (Kubernetes docs)](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Horizontal Pod Autoscaling (Kubernetes docs)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Pod Topology Spread Constraints (Kubernetes docs)](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
