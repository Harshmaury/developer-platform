# ADR-015 — SSE Streaming from Nexus

**Status:** Accepted
**Date:** 2026-03-17
**Author:** Harsh Maury
**Scope:** Nexus — new streaming endpoint
**Depends on:** Nexus Phase 15, all observer services (Metrics, Navigator, Guardian, Observer)

---

## Context

ADR-003 mandates HTTP/JSON on localhost. All four observer services
currently poll GET /events?since=<id> every 3–30 seconds to discover
new platform activity. This works but introduces:
- Latency between an event occurring and an observer reacting
- Redundant HTTP round-trips when nothing has changed
- N×M polling load as the observer ecosystem grows

Now that all planned observers exist (Metrics, Navigator, Guardian,
Observer), the polling overhead is measurable and the case for
streaming is real.

Server-Sent Events (SSE) is still HTTP. It uses the same TCP connection
as normal HTTP, the same X-Service-Token auth, and the same JSON event
format. It does not introduce a new protocol — it extends HTTP with
a persistent response stream. This is within the spirit of ADR-003
(HTTP/JSON) even if the letter says "request/response".

---

## Decision

### 1. SSE is permitted for read-only event streaming only

ADR-003 is amended to read:
> HTTP/JSON is the default protocol. SSE (Server-Sent Events) is
> permitted for read-only, server-to-client event streaming where
> the server is Nexus and the consumers are observer services.
> All other ADR-003 constraints remain unchanged.

### 2. New endpoint: GET /events/stream

Nexus adds a single SSE endpoint:

```
GET /events/stream
```

- Requires `X-Service-Token` (ADR-008 — /health exempt, /events/stream is NOT)
- Streams events as `data: <json>\n\n` in the SSE format
- Each event is the same JSON shape as GET /events response items
- Connection kept alive with `: keepalive\n\n` comment every 15s
- Client disconnects are detected via context cancellation

### 3. What does NOT change

- GET /events and GET /events?since=<id> remain — polling is still supported
- Atlas subscriber continues polling (changing it is a future Atlas phase)
- Forge trigger subscriber continues polling (ADR-007 constraint)
- No SSE on Atlas, Forge, or observer services
- No WebSockets anywhere
- No gRPC anywhere

### 4. Who may use /events/stream

Observer services only:
- Metrics (future phase — switch from polling to SSE)
- Navigator (future phase)
- Guardian (future phase)
- Observer (future phase)

Atlas and Forge must NOT use SSE — they have ADR-mandated polling patterns.

### 5. Fan-out model

Nexus maintains an in-memory subscriber registry. When an event is
written to the store, it is also broadcast to all connected SSE clients.
Each SSE client gets its own goroutine. Slow clients are dropped after
a 5s send timeout rather than blocking the event bus.

---

## Implementation — Nexus Phase 16

### New files
- `internal/api/handler/stream.go` — SSE handler
- `internal/sse/broker.go`         — fan-out broker

### Modified files
- `internal/api/server.go` — register GET /events/stream
- `internal/state/events.go` — EventWriter notifies broker after write

---

## Compliance

| ADR | Status |
|-----|--------|
| ADR-003 | ✅ Amended — SSE is still HTTP, read-only only |
| ADR-007 | ✅ Forge trigger subscriber continues polling |
| ADR-008 | ✅ X-Service-Token required on /events/stream |

---

## Next ADR

ADR-016 — Platform shared types module.
