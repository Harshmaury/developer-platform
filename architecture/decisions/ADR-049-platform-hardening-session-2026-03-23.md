# ADR-049 — Platform Hardening Session: Six Critical Weaknesses Fixed

**Date:** 2026-03-23
**Status:** Accepted
**Author:** Harsh Maury
**Scope:** Platform-wide — Canon, Accord, Arbiter, Nexus, Atlas, Forge, Gate, Relay, all observers

---

## Context

A full system analysis was performed across all 17 platform modules using the
ENGX System Analysis Protocol v1. Six critical architectural weaknesses were
identified, ranked by blast radius, and fixed in a single session.

---

## Weaknesses Fixed

### CW-7 — Atlas Descriptor Duplicated (ADR-016 Violation)
**Repos:** Canon, Atlas | **Commits:** Canon `7ad3123`, Atlas `0054683`

`atlas/internal/validator` defined its own `Descriptor`, `Runtime`, `ValidTypes` —
exact duplicates of `canon/descriptor`. Canon's `ValidTypes` was also missing
`"governance"` and `"tool"` which Atlas had added locally.

**Fix:**
- Canon `descriptor/descriptor.go`: added `"governance"` and `"tool"` to `ValidTypes`
- Atlas `internal/validator/nexus_yaml.go`: deleted local types, imported `canon/descriptor`
- Canon bumped to v1.0.1, Atlas local replace dropped

**Invariant restored:** ADR-016 — Canon is the sole source of all types and constants.

---

### CW-5 — Forge Command Idempotency Missing
**Repo:** Forge | **Commit:** `0cd3b2f`

`POST /commands` had no duplicate detection. A CLI timeout that retried would
execute the same command twice silently.

**Fix:**
- Migration v6: `command_dedup` SQLite table (command_id PK, result_json, expires_at)
- `DedupTTL = 300s`, `SetDedupRecord` / `GetDedupRecord` / `purgeExpiredDedup`
- `Submit()`: dedup check before `Translate()` when caller supplies explicit ID
- Duplicate within TTL: **HTTP 409 + original result in body** (never silent)
- 4 new tests: set+get, not-found, expired, upsert

```
First call (id="cmd-abc"):   200 OK   + result
Retry within 5min (same id): 409      + {ok:false, data:<original result>}
No id supplied:              200 OK   (UUID generated — never a duplicate)
```

---

### CW-4 — Untyped EventDTO.Payload
**Repos:** Canon, Accord, Arbiter | **Commit:** Arbiter `1707660`

`EventDTO.Payload` was an untyped `string`. Schema drift was invisible at compile time.

**Fix:**
- Canon `events/payloads.go`: `SystemAlertPayload{Rule, ProjectID, Message}` +
  `AlertRuleSkipEnforce = "skip-enforce"` constant
- Accord `api/payload.go` (new): `DecodePayload[T any]` generic decoder +
  `EventPayloadType` registry map
- Arbiter `internal/probe/nexus.go`: `EmitSkipEnforceAlert` marshals typed struct

```go
// Usage going forward — all consumers
alert, err := accord.DecodePayload[canonevents.SystemAlertPayload](event.Payload)
span,  err := accord.DecodePayload[accord.PlanSpanDTO](event.Payload)
```

---

### CW-6 — Relay Mux RequestID Collision Risk
**Repo:** Relay | **Commit:** `da99fc7`

`nextID()` used `math/rand.Uint32()`. Birthday-paradox collision risk under
concurrent load — two goroutines could get the same ID, corrupting both HTTP
streams and collapsing the tunnel.

**Fix:**
- `counter atomic.Uint32` field added to `Mux` struct
- `nextID()` returns `m.counter.Add(1)` — monotonically increasing, zero collision risk
- `math/rand` import removed
- Bonus: `DefaultTunnelListenAddr` + `DefaultHTTPListenAddr` constants added
  (were referenced but never defined — pre-existing build break)

---

### CW-1 — Auth Fail-Open Cascade (Systemic)
**Repos:** Nexus, Atlas, Forge + 5 observers | **Commits:** `4a45c64`→`4cbfc6e`

All 7 services degraded to unauthenticated mode when token env vars were missing.
No production enforcement path existed.

**Fix — `ENGX_AUTH_REQUIRED=true` strict mode:**

| Condition | Default (local dev) | `ENGX_AUTH_REQUIRED=true` |
|-----------|--------------------|-----------------------------|
| Token present | ✅ normal | ✅ normal |
| Token missing — startup | WARNING (continue) | **FATAL** (exit 1) |
| Token missing — middleware | pass-through | **HTTP 503** |

WARNING messages now include exact fix command. Zero behavior change when tokens present.

```bash
# Production: one env var enforces all 7 services simultaneously
export ENGX_AUTH_REQUIRED=true
```

---

### CW-2 — Relay Pre-Shared Token Bypasses Gate
**Repos:** Canon, Gate, Relay | **Commit:** Relay `e0a77b1`

`RELAY_TOKEN` was a static pre-shared secret — no revocation, no attribution,
no scope control. Contradiction with ADR-042 (Gate = sole identity authority).

**Fix:**
- Canon `identity/scopes.go`: `ScopeTunnel = "tunnel"`
- Gate: new `POST /gate/tokens/tunnel` endpoint (service-token protected)
- Relay `tunnel/conn.go`: `validateToken()` has two modes:
  - **Production** (`RELAY_GATE_ADDR` set): Gate JWT validation, owner from `sub`
    claim — **cannot be spoofed by request field**
  - **Dev fallback** (`RELAY_GATE_ADDR` unset): legacy `RELAY_TOKEN` comparison

```bash
# Production
export RELAY_GATE_ADDR=http://127.0.0.1:8088
# engxa: POST /gate/tokens/tunnel {"agent_id":"my-agent"} → JWT scope=tunnel
# Relay: validates JWT, owner = sub claim (not from request)

# Dev fallback (unchanged)
export RELAY_TOKEN=my-secret
```

---

## Remaining Items (Future ADRs Required)

| ID | Description | Priority |
|----|-------------|----------|
| M-1 | SQLite migration framework (golang-migrate) | Medium |
| M-2 | Nexus orphan process detector on startup | High |
| M-3 | Gate token refresh endpoint | Medium |
| M-4 | Distributed revocation list (v3 multi-machine) | Low |
| M-5 | Forge workflow DLQ for failed step compensation | Medium |
| M-6 | Platform health aggregator endpoint | Low |

