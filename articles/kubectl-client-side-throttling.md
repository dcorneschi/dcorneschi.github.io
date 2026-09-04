# "Waited for ... due to client-side throttling, not priority and fairness"

If you have run `kubectl` against a cluster with a lot of resource types (CRDs, many API
groups) or a slow/overloaded API server, you have probably seen something like this:

```
I0903 12:14:07.482913   54210 request.go:697] Waited for 1.184s due to client-side throttling, not priority and fairness, request: GET:https://<api-server>/apis/apps/v1?timeout=32s
```

It looks alarming because it mentions "throttling," but the key phrase is **"not priority
and fairness."** This is the client (`kubectl` / `client-go`) rate-limiting *itself*
before the request ever leaves your machine. The API server is not rejecting you. This
article explains what the message means, why it happens, and how to make it go away.

---

## What the message actually says

The message is emitted by `client-go`'s request logic (`request.go`) and has two halves:

- **"Waited for 1.184s"** — the client held the request in a local queue for that long
  before sending it.
- **"due to client-side throttling, not priority and fairness"** — the delay came from
  the client's own rate limiter, *not* from the server-side **API Priority and Fairness**
  (APF) mechanism.

That second clause exists specifically to disambiguate the two throttling sources. Both
can slow down requests, but they live in completely different places and are fixed in
completely different ways.

| | Client-side throttling | API Priority and Fairness (server-side) |
|---|---|---|
| Where it runs | In your `kubectl` / controller process | In the kube-apiserver |
| What controls it | `--qps` / `--burst` (client rate limiter) | `FlowSchema` / `PriorityLevelConfiguration` |
| How you know | Log says **"not priority and fairness"** | HTTP `429 Too Many Requests` with `Retry-After`, APF metrics |
| Who to fix it | The caller (you) | The cluster operator |

If your log line contains "not priority and fairness," you are looking at the left
column. No server config change will help.

---

## Why it happens

`client-go` ships with a **client-side rate limiter** that defaults to a low ceiling:

- **QPS = 5** (sustained requests per second)
- **Burst = 10** (short-term bucket size)

These defaults are intentionally conservative to protect the API server from a stampede
of clients. The problem is that a single `kubectl` command can fire *many* requests in a
short window, most commonly during **API discovery** — the step where `kubectl` learns
what resource types exist so it can map `get pods` to the right REST path.

Discovery scales with the number of API groups/versions in the cluster. On a cluster with
lots of CRDs (service meshes, operators, cloud controllers, monitoring stacks), discovery
can easily need dozens of GET requests. At 5 QPS, the client queues them and you get a
burst of these log lines, often adding a few seconds to an otherwise trivial command.

Common triggers:

- Many CRDs / API groups installed.
- A cold or expired discovery cache (`~/.kube/cache/discovery/`), forcing a full refresh.
- Custom controllers/operators using the `client-go` defaults under load.
- A genuinely slow API server, which makes each queued request take longer and the
  queuing more visible.
- **Nodes being recycled for patching** — a burst of cluster churn that hammers the API
  from many clients at once (see below).

---

## A very common trigger: nodes being recycled for patching

This message shows up reliably during **node recycling / rolling patch operations** —
managed node group upgrades, Karpenter/Cluster Autoscaler drift replacement, `kured`
reboots, or any rolling-replace of nodes for OS patching. It is usually a symptom of the
churn, not a problem with the patching itself.

Why recycling specifically sets it off:

- **Drain/evict storms.** Cordoning and draining a node evicts every pod on it. The
  scheduler, controllers (Deployment/ReplicaSet/StatefulSet), and any operators watching
  those objects all react at once, each issuing `list`/`get`/`update`/`patch` calls.
- **Endpoint and topology churn.** As pods terminate and reschedule, `EndpointSlice`,
  Service, and readiness updates cascade, generating another wave of API traffic.
- **Reconcile amplification.** Controllers re-list and re-sync affected resources.
  Several controllers hitting the API concurrently means each one is independently
  bumping into its own `client-go` limiter (QPS=5 / Burst=10 by default).
- **Repeated discovery.** Automation that shells out to fresh `kubectl` processes per node
  (or per pod) pays the discovery cost each time, especially if the discovery cache is not
  shared or is cold.

The tell-tale sign is that the `Waited for ... not priority and fairness` lines appear
**in bursts that line up with each node rotation** and quiet down once the roll finishes.
That timing correlation is your confirmation it is churn-driven client throttling, not a
steady-state problem.

What to do about it:

- **If the logs come from your own controller/operator/automation:** raise its
  `config.QPS` / `config.Burst` (see the code fixes below) so it can keep up with the
  burst. This is the most effective fix, since these clients are the ones doing the work
  during a roll.
- **If it is your CI/automation shelling out to `kubectl`:** share/warm the discovery
  cache and raise `--kube-api-qps` / `--kube-api-burst`, or use `--watch` instead of
  polling loops that re-list on every tick.
- **Slow the roll down.** Reducing surge / max-unavailable (or Karpenter consolidation
  aggressiveness) spreads the churn over time so no single client is saturated.
- **Confirm it is harmless.** During a controlled patch window, these info logs are
  expected and self-resolving. Only escalate to tuning if the throttling is delaying
  drains enough to slow the roll or trip eviction timeouts.

> Watch for it turning into the *server* side. If many clients all raise their QPS/burst
> at once to power through a roll, you can push the API server into real APF throttling
> and start seeing HTTP `429`s. Tune client limits to match the roll, not far beyond it.

---

## How to confirm it is client-side

A few quick checks:

- **Read the log line.** The phrase "not priority and fairness" is definitive — it is
  client-side.
- **It is a `Waited for ...` info log, not a `429`.** Server-side APF throttling surfaces
  as HTTP `429 Too Many Requests`, usually with a `Retry-After` header, not as this
  message.
- **Raise `kubectl` verbosity** to see the queuing decisions and the requests involved:

  ```bash
  kubectl get pods -v=6
  ```

  At `-v=6` and above you will see the individual discovery requests and the throttle
  waits attributed to the local rate limiter.

---

## Fixes

### 1. Refresh / warm the discovery cache (quick, no config)

Much of the pain comes from discovery. `kubectl` caches discovery results for 10 minutes
under `~/.kube/cache/discovery/`. If the cache is stale or missing, the next command pays
the full discovery cost. Simply re-running the command once the cache is warm often makes
the messages disappear.

If the cache is corrupt or you want a clean refresh:

```bash
rm -rf ~/.kube/cache/discovery
kubectl get pods    # repopulates the cache
```

### 2. Raise the client QPS/burst for kubectl

`kubectl` lets you override the client rate limiter per invocation:

```bash
kubectl get pods --request-timeout=0 \
  --kubeconfig ~/.kube/config \
  --v=2
```

The relevant flags are:

```bash
kubectl <command> --kube-api-qps=50 --kube-api-burst=100
```

> Note: flag availability varies by `kubectl` version. Older builds may not expose
> `--kube-api-qps` / `--kube-api-burst` on every subcommand. If the flag is rejected,
> upgrade `kubectl` or use the controller-side fix below. In all cases, raise these values
> deliberately — the low defaults exist to protect the API server.

### 3. Raise QPS/burst in your own controllers and clients

If the messages come from a controller, operator, or a Go program you own, set the values
on the `rest.Config` before building the clientset:

```go
config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
if err != nil {
    return err
}

// Defaults are QPS=5, Burst=10. Raise for discovery-heavy or high-throughput clients.
config.QPS = 50
config.Burst = 100

clientset, err := kubernetes.NewForConfig(config)
```

For controllers built with `controller-runtime`, set it on the manager options:

```go
cfg := ctrl.GetConfigOrDie()
cfg.QPS = 50
cfg.Burst = 100

mgr, err := ctrl.NewManager(cfg, ctrl.Options{ /* ... */ })
```

### 4. Reduce the number of requests

Sometimes the right fix is to stop making so many calls rather than raising the limit:

- Prune unused CRDs / API groups so discovery has less to enumerate.
- Cache and reuse a discovery/REST mapper client instead of rebuilding it per request.
- Batch or `list`+`watch` instead of polling many individual `get`s.

---

## When you should *not* just raise the limit

The client-side limiter is a safety valve. Cranking QPS/burst very high across many
clients can shift the load onto the API server and turn a harmless client-side info log
into real server-side pressure — at which point you *will* see APF `429`s (the "priority
and fairness" side). Raise the values to match legitimate need, and if you are hitting
walls at the server, that is a cluster-capacity / APF-tuning conversation, not a client
rate-limit one.

---

## TL;DR

- The message is `client-go` throttling itself, **before** the request is sent.
- "not priority and fairness" explicitly rules out the server-side APF mechanism.
- Defaults are **QPS=5, Burst=10**; discovery on CRD-heavy clusters blows past them.
- Fix by warming the discovery cache, raising `--kube-api-qps` / `--kube-api-burst`
  (or `config.QPS` / `config.Burst` in code), or making fewer requests.
- It is an informational log, not an error — but persistent delays are worth tuning.
