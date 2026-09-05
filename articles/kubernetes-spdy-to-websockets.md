# Kubernetes Is Migrating Streaming from SPDY to WebSockets

Four `kubectl` commands are different from the rest: `exec`, `attach`, `cp`, and
`port-forward`. Everything else (`get`, `apply`, `logs` without follow) is ordinary
request/response HTTP — send a request, stream a body, done. These four need a
**long-lived, bidirectional** connection that *both* ends can write to as data arrives:
your keystrokes going in, the container's stdout coming back, ports being tunneled both
ways.

For over a decade Kubernetes carried that traffic over **SPDY/3.1** — a protocol Google
deprecated in 2016, that never had real proxy/load-balancer support, and that isn't in any
standard HTTP library. **KEP-4006** replaces it with **WebSockets**, which is standardized
(RFC 6455), universally supported by proxies and browsers, and speaks the same HTTP
`Upgrade` handshake everything else already understands.

This article explains what actually changed on the wire, why it matters operationally, and
how to check and troubleshoot it. It draws on the KEP design, the Kubernetes 1.31 release
notes, and Henrique Cavarsan's excellent wire-level write-up at kftray (linked in Sources),
which traced the subprotocol naming by hand.

---

## 1. Why SPDY had to go

SPDY was the streaming transport for `exec`/`attach`/`cp`/`port-forward` since the early
days. The problems piled up:

- **Deprecated.** Google abandoned SPDY in 2016 in favor of HTTP/2. Kubernetes was carrying
  a dead protocol.
- **No ecosystem support.** SPDY's `Upgrade` isn't understood by standard HTTP proxies,
  ingress controllers, gateways, or client libraries. Anyone building tooling had to
  reimplement the SPDY codec themselves.
- **Proxy hostility.** Many reverse proxies and load balancers would mangle or drop the
  SPDY upgrade, which is why `kubectl exec` "sometimes works, sometimes hangs" behind
  certain ingress setups.

WebSockets fix all three: it's an IETF standard, every proxy and browser supports it, and
it uses the same `Connection: Upgrade` / `101 Switching Protocols` handshake as the rest of
the web.

---

## 2. The rollout timeline

The migration happened per-command, gated by feature flags, over several releases:

- **`exec` / `attach` / `cp`** moved first (the "RemoteCommand" subprotocol). Beta and on
  by default earlier in the 1.29–1.30 range.
- **Kubernetes 1.31:** by default, **`kubectl` uses WebSockets instead of SPDY for
  streaming**. This is the headline milestone. `port-forward` support arrived here too,
  behind flags.
- **`port-forward`** was the trickiest and trailed the others, gated by
  `PortForwardWebsockets` (server) and `KUBECTL_PORT_FORWARD_WEBSOCKETS=true` (client) while
  it stabilized.
- **WebSocket-to-kubelet** (the *upstream* hop, apiserver → kubelet) went **Beta in 1.36**
  and is on track for GA. Until that completes, the apiserver still talks SPDY to the
  kubelet even when the client talks WebSocket (see the proxy below).

The practical takeaway: on any modern cluster (1.31+), `kubectl exec`/`cp`/`attach` already
use WebSockets by default, and `port-forward` is following close behind.

---

## 3. What actually changed on the wire

The elegant part of KEP-4006 is how *little* changed above the transport.

### The Upgrade handshake

Both SPDY and WebSocket use HTTP's `Upgrade` mechanism. The client opens with
`Connection: Upgrade`, the server answers `101 Switching Protocols`, and from that point the
TCP connection *is* the new protocol.

**SPDY upgrade (old):**

```http
POST /api/v1/namespaces/default/pods/my-pod/portforward HTTP/1.1
Connection: Upgrade
Upgrade: SPDY/3.1
X-Stream-Protocol-Version: portforward.k8s.io
```

**WebSocket upgrade (new):** same shape, but with the standard WebSocket headers
(`Upgrade: websocket`, `Sec-WebSocket-Key`, and a `Sec-WebSocket-Protocol` naming the
subprotocol). The server replies `101 Switching Protocols` and echoes the chosen
subprotocol.

### The bytes inside didn't change — only the envelope

Here's the subtle bit: **the streams inside are still the same SPDY-style frames.** Moving
to WebSockets didn't change what the streams carry (data stream, error stream), how the
kubelet routes them, or how the container runtime terminates them. Only the **transport
wrapping the frames** changed — SPDY frames are now tunneled *inside* WebSocket payloads.

That's why the design kept the v1 protocol model and just gave the new tunneling shape a
second **label**. Three names you'll encounter, and what each means:

| Name | What it is |
|---|---|
| `portforward.k8s.io` | The original **v1** subprotocol name; still the model for what flows inside the streams. The code constant (`PortForwardProtocolV1Name`) is unchanged. |
| `v2.portforward.k8s.io` | The KEP's **label for the v2 transport** (SPDY frames tunneled inside WebSocket payloads). This is what `kubectl` logs. |
| `SPDY/3.1+portforward.k8s.io` | What the client advertises and the server echoes in **`Sec-WebSocket-Protocol`**. The `SPDY/3.1+` **prefix** is the switch the server uses to pick the WebSocket-tunneling path; the suffix is the unchanged v1 name. |

These look contradictory until you see the layering: the prefix selects the transport, the
suffix identifies the stream model, and the "v2" is just a name for "v1 frames over a
WebSocket."

---

## 4. The StreamTranslator / tunneling proxy

Because the client may speak WebSocket while the kubelet (until 1.36+) still speaks SPDY,
the API server sits in the middle as a **translating proxy**:

- **Client side** (`tunneling_dialer.go`): builds the v2 request by **prepending
  `SPDY/3.1+`** to the v1 subprotocol name before sending the upgrade.
- **API server** (`streamtunnel.go` / the StreamTranslator proxy): accepts only the
  **prefixed** subprotocols, **strips the `SPDY/3.1+`**, and rebuilds an **upstream SPDY**
  request to the kubelet using the unprefixed name. It translates WebSocket frames from the
  client to/from the SPDY connection to the runtime.

So a modern `kubectl exec` looks like:

```
kubectl  ──WebSocket──▶  kube-apiserver (StreamTranslator)  ──SPDY──▶  kubelet ──▶ runtime
```

Once WebSocket-to-kubelet (Beta in 1.36) is fully GA, that upstream SPDY hop disappears and
the whole path is WebSocket end to end, letting the SPDY codec finally be deleted.

---

## 5. Why this matters operationally

For most users the switch is invisible — that's the point. But it has real consequences:

- **Proxies and ingress now "just work" (if configured right).** WebSocket upgrades are
  understood by standard proxies, so `kubectl exec`/`port-forward` through an ingress or
  gateway is far more reliable — *provided* the proxy forwards the upgrade headers.
- **The #1 failure mode: a proxy stripping the upgrade headers.** WebSocket errors on
  `exec`/`attach` almost always trace to a reverse proxy dropping the `Upgrade: websocket`
  (and `Connection: Upgrade`) header before it reaches the API server. If your tooling or
  proxy doesn't handle the WebSocket upgrade, `kubectl` may **fall back to SPDY** — the
  command "succeeds" but over the deprecated path. That's a signal to fix the proxy, not to
  ignore it, because SPDY is going away.
- **RBAC change to be aware of.** Newer Kubernetes tightened authorization so that
  WebSocket `pods/exec`, `pods/attach`, and `pods/portforward` requests require the
  **`create`** verb (previously WebSocket only needed `get`). If a custom Role grants only
  `get` on these subresources, WebSocket-based `exec`/`attach`/`port-forward` will be
  denied. (This is called out in the 1.35 upgrade notes.)
- **Custom tooling must migrate.** Anything built on client-go streaming, or that
  reimplemented SPDY, needs to move to the WebSocket-capable dialers. SPDY-only clients are
  on borrowed time.

---

## 6. How to check what you're using

```bash
# Force kubectl to log the protocol negotiation. Look for the Sec-WebSocket-Protocol
# header advertising SPDY/3.1+... — that's the WebSocket-tunneling path.
kubectl exec -v=8 my-pod -- true 2>&1 | grep -iE 'websocket|spdy|upgrade|switching'

# port-forward: opt in explicitly on versions where it's still gated
KUBECTL_PORT_FORWARD_WEBSOCKETS=true kubectl port-forward -v=8 pod/my-pod 8080:80 \
  2>&1 | grep -iE 'websocket|spdy|Sec-WebSocket-Protocol'

# Confirm the server-side feature gate (on clusters where it's still a gate)
kubectl get --raw /metrics | grep -i 'feature.*PortForwardWebsockets' 2>/dev/null
```

Interpreting it:

- `Sec-WebSocket-Protocol: SPDY/3.1+portforward.k8s.io` (or `...+channel.k8s.io` for
  exec/attach) → you're on the **WebSocket** path.
- Plain `Upgrade: SPDY/3.1` with no WebSocket headers → still **SPDY** (either an old
  client/server, or a proxy stripped the upgrade and kubectl fell back).

If you see a fallback to SPDY, check the proxy/ingress in the path forwards the
`Upgrade`/`Connection` headers untouched.

---

## 7. Quick reference

| Item | Detail |
|---|---|
| KEP | [KEP-4006](https://github.com/kubernetes/enhancements/issues/4006): Transition streaming from SPDY/3.1 to WebSockets |
| Affected commands | `kubectl exec`, `attach`, `cp`, `port-forward` |
| Default WebSockets for kubectl | Kubernetes **1.31** |
| port-forward gates | server `PortForwardWebsockets`; client `KUBECTL_PORT_FORWARD_WEBSOCKETS=true` |
| WebSocket → kubelet (upstream hop) | Beta in **1.36**, heading to GA |
| Client subprotocol advertised | `SPDY/3.1+<v1-name>` in `Sec-WebSocket-Protocol` |
| Server proxy | StreamTranslator (`streamtunnel.go`): strips `SPDY/3.1+`, tunnels to kubelet |
| Inspect negotiation | `kubectl exec -v=8 ... 2>&1 \| grep -i websocket` |
| Common failure | reverse proxy stripping `Upgrade: websocket` → falls back to SPDY |
| RBAC note | WebSocket `pods/exec|attach|portforward` need the `create` verb |

---

## Summary

- Kubernetes is replacing **SPDY/3.1** (deprecated, unsupported by proxies) with
  **WebSockets** (RFC 6455, universally supported) for the four streaming commands:
  `exec`, `attach`, `cp`, `port-forward`. Default for `kubectl` since **1.31**.
- Only the **transport envelope** changed. The same SPDY-style stream frames still flow
  *inside* WebSocket payloads, which is why the v1 protocol model was kept and the new shape
  just got a label (`v2.portforward.k8s.io`, advertised as `SPDY/3.1+portforward.k8s.io`).
- The API server runs a **StreamTranslator** proxy that accepts the WebSocket connection,
  strips the `SPDY/3.1+` prefix, and (until WebSocket-to-kubelet is GA, Beta in 1.36) still
  talks SPDY upstream to the kubelet.
- Operationally: proxies now work if they forward the upgrade headers; the top failure is a
  proxy **stripping** them (causing a silent SPDY fallback); and note the RBAC change that
  WebSocket `exec`/`attach`/`port-forward` now require the `create` verb.

---

### Sources

- [KEP-4006: Transition from SPDY to WebSockets](https://github.com/kubernetes/enhancements/issues/4006) and its [design doc](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/4006-transition-spdy-to-websockets)
- [Kubernetes 1.31: Streaming Transitions from SPDY to WebSockets (Kubernetes Blog)](https://kubernetes.io/blog/2024/08/20/websockets-transition/)
- [Portforward over WebSockets — PR #120889 (StreamTranslator proxy)](https://github.com/kubernetes/kubernetes/pull/120889)
- [Tunnel SPDY through WebSockets — PR #123413 (port-forward feature flags)](https://github.com/kubernetes/kubernetes/pull/123413)
