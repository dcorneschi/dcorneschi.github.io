# How Kubernetes Watches Work — Informers, List-Watch, and the Shared Cache

The internal mechanism that powers every Kubernetes controller — how components observe cluster state changes in real-time without polling, using the list-watch pattern, informers, and shared caches.

## Why Watches Exist

Every controller (Deployment controller, scheduler, kubelet) needs to know the current state of objects and react to changes. Polling the API server on a timer would be:
- Wasteful (most polls return "nothing changed")
- Slow (changes detected only at poll interval)
- Expensive (API server handles N controllers × M objects × poll frequency)

Instead, Kubernetes uses **watches** — long-lived HTTP connections that stream change events in real-time.

## The List-Watch Pattern

Every Kubernetes client follows the same pattern:

```
┌───────────────┐                          ┌───────────────┐
│   Controller  │                          │   API Server  │
│               │                          │               │
│  1. LIST ─────┼──── GET /api/v1/pods ───▶│               │
│               │◀─── Full list + RV ──────┤               │
│               │                          │               │
│  2. WATCH ────┼──── GET /api/v1/pods?    │               │
│               │     watch=true&          │               │
│               │     resourceVersion=RV ─▶│               │
│               │                          │               │
│               │◀─── Event stream ────────┤               │
│               │     (ADDED, MODIFIED,    │               │
│               │      DELETED)            │               │
└───────────────┘                          └───────────────┘
```

### Step 1: LIST

The initial LIST retrieves all objects matching a selector and returns a `resourceVersion` (RV):

```json
{
  "kind": "PodList",
  "metadata": {
    "resourceVersion": "12345"
  },
  "items": [ ... all pods ... ]
}
```

The `resourceVersion` is an opaque string (etcd revision number) that marks "I've seen everything up to this point."

### Step 2: WATCH

The WATCH request opens a long-lived HTTP/2 stream starting from the given resourceVersion:

```
GET /api/v1/pods?watch=true&resourceVersion=12345

Response (chunked transfer encoding, never-ending):
{"type":"ADDED","object":{...pod spec...}}
{"type":"MODIFIED","object":{...updated pod...}}
{"type":"DELETED","object":{...deleted pod...}}
{"type":"MODIFIED","object":{...another update...}}
...
```

The connection stays open indefinitely. Each line is a JSON event.

### Watch Event Types

| Type | Meaning | When Fired |
|------|---------|------------|
| `ADDED` | New object created | Object didn't exist before this RV |
| `MODIFIED` | Object updated | Any field changed (spec, status, labels, annotations) |
| `DELETED` | Object removed from etcd | Object was deleted |
| `BOOKMARK` | No data change — just an RV checkpoint | Periodically, to prevent RV from going stale |
| `ERROR` | Watch failed | Usually 410 Gone (RV too old) |

## resourceVersion and the 410 Gone Problem

etcd keeps a limited history of changes (controlled by `--etcd-compaction-interval`, default 5 minutes). If a client's resourceVersion is older than what etcd remembers:

```
Controller reconnects with resourceVersion=500
etcd has compacted everything before RV=10000

API Server returns: 410 Gone
Message: "resourceVersion 500 is too old"
```

The client must fall back to a full LIST to re-sync, then start a new WATCH from the fresh RV.

```
┌───────────────┐                       ┌───────────────┐
│   Controller  │                       │   API Server  │
│               │                       │               │
│  WATCH(rv=500)┼──────────────────────▶│               │
│               │◀── 410 Gone ──────────│               │
│               │                       │               │
│  LIST ────────┼──────────────────────▶│               │
│               │◀─ Full list (rv=15000)│               │
│               │                       │               │
│  WATCH(15000) ┼──────────────────────▶│               │
│               │◀── Event stream ──────│               │
└───────────────┘                       └───────────────┘
```

## The Informer Architecture

Raw list-watch is low-level. In practice, every Go controller uses **informers** — a higher-level abstraction from `client-go`:

```
┌─────────────────────────────────────────────────────────────────┐
│  Informer Architecture                                          │
│                                                                 │
│  ┌──────────┐     ┌───────────┐     ┌───────────┐     ┌───────┐ │
│  │ Reflector│────▶│ DeltaFIFO │────▶│  Indexer  │────▶│ Store │ │
│  │          │     │ (queue)   │     │  (cache)  │     │(thread│ │
│  │ List +   │     │           │     │           │     │ safe) │ │
│  │ Watch    │     │Deduplicates     │ In-memory │     │       │ │
│  │          │     │ + orders  │     │ indexed   │     │       │ │
│  └──────────┘     └───────────┘     └───────────┘     └───────┘ │
│       │                │                                        │
│       │                ▼                                        │
│       │         ┌──────────────┐                                │
│       │         │  Event       │                                │
│       │         │  Handlers    │                                │
│       │         │  (callbacks) │                                │
│       │         │              │                                │
│       │         │ OnAdd()      │                                │
│       │         │ OnUpdate()   │                                │
│       │         │ OnDelete()   │                                │
│       │         └──────────────┘                                │
│       │                │                                        │
│       │                ▼                                        │
│       │         ┌──────────────┐                                │
│       │         │  Work Queue  │                                │
│       │         │  (rate-      │                                │
│       │         │   limited)   │                                │
│       │         └──────────────┘                                │
│       │                │                                        │
│       │                ▼                                        │
│       │         ┌──────────────┐                                │
│       │         │  Worker      │                                │
│       │         │  (reconcile  │                                │
│       │         │   loop)      │                                │
│       │         └──────────────┘                                │
└─────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Role |
|-----------|------|
| **Reflector** | Performs LIST + WATCH against the API server. Pushes events into DeltaFIFO. Handles reconnection on errors. |
| **DeltaFIFO** | Ordered queue of deltas (Add, Update, Delete, Sync). Deduplicates — if an object is updated 3 times before processing, only the latest state matters. |
| **Indexer / Store** | Thread-safe in-memory cache of all objects. Indexed by namespace/name (and custom indexes). Controllers read from here, NOT from the API server. |
| **Event Handlers** | Callbacks registered by the controller: `OnAdd`, `OnUpdate`, `OnDelete`. Typically just enqueue the object key into the work queue. |
| **Work Queue** | Rate-limited queue of object keys (namespace/name). Handles retries with exponential backoff. Prevents duplicate processing. |
| **Worker** | Goroutine(s) that dequeue keys and call the reconcile function. The actual business logic lives here. |

## Shared Informers — Why Sharing Matters

Multiple controllers often watch the same resource (e.g., both the Deployment controller and HPA watch Pods). Without sharing:

```
Deployment controller → LIST pods + WATCH pods  (own connection)
ReplicaSet controller → LIST pods + WATCH pods  (own connection)
Scheduler             → LIST pods + WATCH pods  (own connection)
Endpoint controller   → LIST pods + WATCH pods  (own connection)

= 4 separate LIST calls + 4 watch connections for the same data
```

**SharedInformerFactory** solves this:

```
┌────────────────────────────────────┐
│  SharedInformerFactory             │
│                                    │
│  Pod Informer (single connection)  │
│    ├── Event Handler: Deployment   │
│    ├── Event Handler: ReplicaSet   │
│    ├── Event Handler: Scheduler    │
│    └── Event Handler: Endpoints    │
│                                    │
│  = 1 LIST + 1 WATCH + 1 cache      │
│    shared across all controllers   │
└────────────────────────────────────┘
```

One connection to the API server, one in-memory cache, multiple consumers.

## How Controllers Use the Cache

Controllers NEVER read directly from the API server in their reconcile loop. They read from the local informer cache:

```
┌──────────────────────────────────────────────────────┐
│  Reconcile Loop (Deployment Controller)              │
│                                                      │
│  func Reconcile(key string):                         │
│    // Read from LOCAL CACHE (not API server)         │
│    deployment = informer.Lister().Get(key)           │
│    replicaSets = rsInformer.Lister().List(selector)  │
│                                                      │
│    // Compute desired state vs actual state          │
│    // ...                                            │
│                                                      │
│    // WRITE to API server (creates/updates/deletes)  │
│    client.AppsV1().ReplicaSets().Create(newRS)       │
│    client.AppsV1().ReplicaSets().Scale(oldRS, 0)     │
└──────────────────────────────────────────────────────┘
```

**Reads**: Always from local cache (fast, no API server load)  
**Writes**: Always to API server (which then persists to etcd)

This means the cache can be slightly stale (milliseconds), but controllers are designed to be **level-triggered** (reconcile the full desired state), not **edge-triggered** (react to individual events).

## Level-Triggered vs Edge-Triggered

```
Edge-triggered (fragile):
  "A pod was added" → do ONE thing
  Problem: if you miss an event, state drifts

Level-triggered (resilient):
  "There should be 3 pods, I see 2" → create 1 pod
  Problem: none! Works even if events were missed
```

Kubernetes controllers are level-triggered. The event handlers just trigger reconciliation — the reconcile function examines the FULL desired vs actual state every time.

## Watch Bookmarks

To prevent resourceVersion from going stale during quiet periods, the API server sends bookmark events:

```json
{"type":"BOOKMARK","object":{"metadata":{"resourceVersion":"50000"}}}
```

This tells the client: "Nothing happened, but your RV is now 50000." If the client disconnects and reconnects, it can use RV=50000 instead of having to do a full re-LIST.

## Watch Connection Lifecycle

```
┌────────────────────────────────────────────────────────────┐
│  Reflector Connection Management                           │
│                                                            │
│  1. Initial LIST → populate cache                          │
│  2. Start WATCH from LIST's resourceVersion                │
│  3. Process events until:                                  │
│     a. Connection drops → reconnect WATCH from last RV     │
│     b. 410 Gone → full re-LIST, new WATCH                  │
│     c. Timeout (server-side, ~5-10min) → reconnect         │
│     d. Context cancelled → stop                            │
│                                                            │
│  Retry with backoff on transient errors                    │
│  Never give up unless context is cancelled                 │
└────────────────────────────────────────────────────────────┘
```

The API server may close watch connections after a timeout (configurable, typically 5-10 minutes). This is normal — the reflector reconnects transparently.

## API Server Watch Implementation

On the server side, watches are backed by etcd's watch mechanism:

```
┌───────────────┐     ┌───────────────┐     ┌───────────┐
│   Controller  │────▶│   API Server  │────▶│   etcd    │
│  (WATCH req)  │     │               │     │           │
│               │     │ Watch Cache   │     │  Watch on │
│               │◀────│ (ring buffer) │◀────│ /registry │
│               │     │               │     │  prefix   │
└───────────────┘     └───────────────┘     └───────────┘
```

The API server maintains a **watch cache** (in-memory ring buffer) per resource type:
- Buffers recent events so multiple watchers can share them
- Serves watch requests from cache, not directly from etcd
- Reduces etcd load significantly in large clusters

```bash
# API server watch cache configuration:
# --watch-cache=true (default)
# --watch-cache-sizes (per-resource cache size)
# --default-watch-cache-size=100 (default events buffered)
```

## Observing Watches in Action

```bash
# Watch pods yourself (same mechanism controllers use):
kubectl get pods -w

# See raw watch events with verbosity:
kubectl get pods -w -v=8

# Watch with output format (see each event):
kubectl get pods -w -o json

# Count active watchers on the API server:
kubectl get --raw /metrics | grep apiserver_current_inflight_requests

# Watch events for a specific resource:
kubectl get events -w --field-selector involvedObject.name=my-pod
```

## Performance Implications

| Factor | Impact | Mitigation |
|--------|--------|------------|
| Many watchers on same resource | API server fan-out overhead | Shared informers (client-side), watch cache (server-side) |
| Large objects (big ConfigMaps) | Bandwidth per event | Use field selectors, watch specific resources |
| High event rate | Queue buildup in DeltaFIFO | Rate-limited work queues, multiple workers |
| etcd compaction | 410 Gone forcing re-LIST | Bookmark events, longer compaction intervals |
| Reconnection storms | Many clients re-LIST simultaneously | Jittered backoff in reflectors |

## Quick Reference

```bash
# The list-watch pattern:
# 1. LIST /api/v1/pods → get all pods + resourceVersion
# 2. WATCH /api/v1/pods?watch=true&resourceVersion=X → stream changes

# Watch event types: ADDED, MODIFIED, DELETED, BOOKMARK, ERROR

# Key components (client-go):
# Reflector → DeltaFIFO → Indexer/Store → Event Handlers → Work Queue → Workers

# resourceVersion:
# - Opaque string (etcd revision)
# - Used to resume watches without re-listing
# - 410 Gone = RV too old, must re-LIST

# Controllers:
# - Read from local cache (informer.Lister())
# - Write to API server (client.Create/Update/Delete)
# - Level-triggered reconciliation (not edge-triggered)

# Shared informers:
# - One LIST+WATCH per resource type per process
# - Multiple controllers share the same cache
# - Registered via SharedInformerFactory
```
