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
