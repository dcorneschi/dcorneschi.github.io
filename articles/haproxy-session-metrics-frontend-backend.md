# HAProxy Session Metrics: Frontend vs Backend

## Overview

HAProxy defines a session as being composed of two connections: one from the client to HAProxy, and the other from HAProxy to the appropriate backend server. The `session.current` metrics (also known as `scur` on the HAProxy stats page) track how many of these sessions are active at any given moment.

## Metrics

### haproxy.frontend.session.current

The number of active sessions on the frontend right now. This represents how many client connections are currently established **to** HAProxy. Each client TCP connection (or HTTP request in HTTP mode) that's been accepted counts as a frontend session.

### haproxy.backend.session.current

The number of active sessions on the backend right now. This represents how many connections HAProxy currently has open **to** your backend servers. It's the sum across all servers in that backend.

## Key Difference

| Metric | Measures | Direction |
|--------|----------|-----------|
| `frontend.session.current` | Client → HAProxy connections | Inbound |
| `backend.session.current` | HAProxy → Server connections | Outbound |

## Why They Can Differ

| Scenario | Meaning |
|----------|---------|
| Frontend > Backend | HAProxy is queuing requests (backends full or slow to accept), or keep-alive connections are idle on the client side with no active backend connection yet |
| Backend > Frontend | Rare, but possible with connection multiplexing or health checks counted separately |
| Frontend ≈ Backend | Normal in simple proxy mode (1:1 mapping of client to server connections) |

## Session Flow Architecture

```
                    CLIENTS                              BACKEND SERVERS
              ┌──────────────┐                        ┌──────────────────┐
              │  Client A    │                        │  Server 1        │
              │  (browser)   │                        │  (app instance)  │
              └──────┬───────┘                        └────────▲─────────┘
                     │                                         │
              ┌──────┴───────┐                        ┌────────┴─────────┐
              │  Client B    │                        │  Server 2        │
              │  (mobile)    │                        │  (app instance)  │
              └──────┬───────┘                        └────────▲─────────┘
                     │                                         │
              ┌──────┴───────┐                        ┌────────┴─────────┐
              │  Client C    │                        │  Server 3        │
              │  (API call)  │                        │  (app instance)  │
              └──────┬───────┘                        └────────▲─────────┘
                     │                                         │
                     │  INBOUND                      OUTBOUND  │
                     │  connections               connections  │
                     │                                         │
         ┌───────────▼─────────────────────────────────────────┴───────────┐
         │                                                                 │
         │                         HAProxy                                 │
         │                                                                 │
         │  ┌─────────────────────┐         ┌──────────────────────┐       │
         │  │      FRONTEND       │         │       BACKEND        │       │
         │  │                     │         │                      │       │
         │  │  session.current: 3 │────────▶│  session.current: 3  │       │
         │  │                     │         │                      │       │
         │  │  Metric:            │         │  Metric:             │       │
         │  │  haproxy.frontend.  │         │  haproxy.backend.    │       │
         │  │  session.current    │         │  session.current     │       │
         │  │                     │         │                      │       │
         │  │  Stats: scur = 3    │         │  Stats: scur = 3     │       │
         │  └─────────────────────┘         └──────────────────────┘       │
         │                                                                 │
         └─────────────────────────────────────────────────────────────────┘
```

## Scenario: Frontend > Backend (Queuing / Keep-Alive Idle)

```
              CLIENTS                                BACKEND SERVERS
         ┌──────────────┐                         ┌──────────────┐
         │  Client A ●──┼──── active ────┐        │  Server 1    │
         └──────────────┘                │        └──────▲───────┘
         ┌──────────────┐                │               │
         │  Client B ●──┼──── active ────┤        ┌──────┴───────┐
         └──────────────┘                │        │  Server 2    │
         ┌──────────────┐                │        └──────────────┘
         │  Client C ●──┼──── idle ──────┤
         └──────────────┘                │
         ┌──────────────┐                │
         │  Client D ●──┼──── queued ────┤
         └──────────────┘                │
                                         ▼
                               ┌──────────────────┐
                               │     HAProxy      │
                               │                  │
                               │  Frontend: 4     │  ← 4 client connections
                               │  Backend:  2     │  ← only 2 forwarded
                               │                  │
                               │  Queue: 2        │  ← waiting/idle
                               └──────────────────┘

  Why: Backends are full or slow, or keep-alive clients are idle
       with no active backend connection yet.
```

## Scenario: Frontend ≈ Backend (Normal 1:1 Proxy)

```
         Client A ●────────┐         ┌────────● Server 1
                           │         │
         Client B ●─────── HAProxy ──┼────────● Server 2
                           │         │
         Client C ●────────┘         └────────● Server 3

                    Frontend: 3    Backend: 3
                         (1:1 mapping - simple proxy mode)
```

## Metric Relationship Summary

```
                        ┌─────────────────────┐
                        │   CLIENT REQUEST    │
                        └──────────┬──────────┘
                                   │
                                   ▼
                ┌──────────────────────────────────────┐
                │           FRONTEND                   │
                │                                      │
                │   scur++ (frontend.session.current)  │
                │   stot++ (total counter)             │
                │   rate recalculated                  │
                └──────────────────┬───────────────────┘
                                   │
                          ┌────────┴────────┐
                          │                 │
                          ▼                 ▼
                   ┌────────────┐    ┌────────────┐
                   │  FORWARD   │    │   QUEUE    │
                   │  to backend│    │  (if full) │
                   └─────┬──────┘    └────────────┘
                         │
                         ▼
                ┌──────────────────────────────────────┐
                │            BACKEND                   │
                │                                      │
                │   scur++ (backend.session.current)   │
                │   Connection to upstream server      │
                └──────────────────┬───────────────────┘
                                   │
                                   ▼
                        ┌─────────────────────┐
                        │   BACKEND SERVER    │
                        │  processes request  │
                        └──────────┬──────────┘
                                   │
                                   ▼
                        ┌─────────────────────┐
                        │   RESPONSE SENT     │
                        │                     │
                        │   backend.scur--    │
                        │   frontend.scur--   │
                        └─────────────────────┘
```

## Stats Page Session Metrics

```
┌──────────────────────────────────────────────────────────────────────┐
│  SESSION LIFECYCLE                                                   │
│                                                                      │
│  New connection ──▶ scur++ ──▶ request processed ──▶ scur--          │
│       │                                                              │
│       ▼                                                              │
│    stot++                                                            │
│    rate recalculated                                                 │
│                                                                      │
│  If scur > smax → smax = scur (new high water mark)                  │
│  If scur ≥ slim → new connections REJECTED                           │
└──────────────────────────────────────────────────────────────────────┘
```

## Related Stats Page Fields

| Field | Description |
|-------|-------------|
| `scur` | Current sessions (same as `session.current`) |
| `smax` | Maximum concurrent sessions observed |
| `slim` | Configured session limit (`maxconn`) |
| `stot` | Total number of sessions since HAProxy started |
| `rate` | Number of new sessions per second |

## Troubleshooting Scenarios

### High Frontend Sessions, Low Backend Sessions

**Symptom:** `frontend.session.current` >> `backend.session.current`

**Diagnosis:** HAProxy is rejecting or filtering requests before they reach backends.

**Causes:**
- Rate limiting rules blocking traffic
- ACL denying requests
- Authentication failures
- Request size limits exceeded
- Backend queue full (`maxconn` on backend reached)

### Equal Sessions, High Backend Response Time

**Symptom:** Frontend/backend sessions match, but `backend.response_time` is high.

**Diagnosis:** Application server performance issue — all requests get through, but responses are slow.

**Causes:**
- Database slowdowns
- CPU/memory constraints on app servers
- Network latency to backends
- Upstream dependency timeouts

### High Backend Queue

**Symptom:** `backend.queue.current` is consistently high.

**Diagnosis:** Backends can't keep up — need more capacity or faster servers.

**Causes:**
- Insufficient backend server count
- Backend servers too slow to process requests
- Load balancing algorithm sending too much to one server
- `maxconn` per server too low

### High Frontend Denied Requests

**Symptom:** High `frontend.denied_req` or frontend 4xx errors.

**Diagnosis:** Client-side issues or HAProxy ACL/config problems.

**Causes:**
- Malformed client requests
- Invalid SSL certificates
- IP blacklisting or geo-blocking
- Incorrect HAProxy ACL rules

### High Backend 5xx Responses

**Symptom:** High `backend.http_responses_5xx`.

**Diagnosis:** Application servers are failing.

**Causes:**
- Application crashes or exceptions
- Database connection pool exhaustion
- Resource limits reached (memory, file descriptors)
- Deployment issues (bad release)

## Performance Ratios

### Request Acceptance Rate

```
Backend Sessions / Frontend Sessions ≈ 1.0 (healthy)
```

Lower values indicate request filtering or rejection. Investigate ACLs, rate limiting, or backend saturation.

### Error Source Identification

```
High frontend errors → client or HAProxy config issues
High backend errors  → application server issues
```

Compare `frontend.denied_req` vs `backend.http_responses_5xx` to locate the problem layer.

### Capacity Pressure

```
Backend Queue / Backend Sessions = capacity pressure indicator
```

Higher ratios mean backends are saturated. Scale horizontally or optimize application performance.

### Monitoring Strategy

| Focus | Key Metrics |
|-------|-------------|
| Frontend health | `session_rate`, `denied_req`, `bytes_in_rate`, connection errors |
| Backend health | `response_time`, `servers_up/down`, `queue_current`, `http_responses_5xx` |
| Combined | Frontend/backend session ratio, end-to-end latency, error distribution |
