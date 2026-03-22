# AI_CONTEXT.md

Context document for AI systems working within this developer platform.
Read this file at the start of any session involving platform architecture,
new service design, code changes, or cross-project integration work.

**Updated:** 2026-03-21 | **Tag:** v2.0.0-candidate

---

## 1. What This Platform Is

A local developer control plane. Three capability layers, thirteen services, one CLI.

```
Control    Nexus   :8080   coordinates everything — sole registry, sole writer
Knowledge  Atlas   :8081   understands the workspace — capability graph, verified projects
Execution  Forge   :8082   acts on the workspace — build, test, run, deploy
Observer   5 svcs  :8083–:8087   read-only — never call write endpoints
Library    Canon   —       shared types — ServiceTokenHeader, TraceIDHeader, defaults
Tool       ZP      v2.0.0  packaging — builds ZIPs from nexus.yaml
Contract   Accord  —       shared types — API DTOs, ErrorCode, Response[T]
Client     Herald  —       typed HTTP client — all inter-service calls go through here
```

This repository governs the platform. It contains no implementation code.

---

## 2. Current Platform State (2026-03-21)

| Service   | Version          | Phase    | Key State                              |
|-----------|------------------|----------|----------------------------------------|
| Nexus     | v1.7.3           | 1–22     | Wave 5+6+UX: system/graph, system/validate, engx run/ps/deregister |
| Atlas     | v0.5.0-phase3    | 1–4      | nexus.yaml contract, verified graph, logger-threaded handlers |
| Forge     | v0.5.0-phase5    | 1–5      | CronScheduler dedup fixed + wired, preflight snapshot |
| Metrics   | v0.2.0-phase2    | 1–2      | Prometheus /metrics/prometheus endpoint. Herald v1.5 collectors |
| Navigator | v0.1.0-phase1    | 1        | Herald v1.5 collector. Canon headers, trace propagation |
| Guardian  | v0.2.0-phase2    | 1–2      | Herald v1.5 collectors. Per-cycle trace ID, WARNING logs |
| Observer  | v0.2.0-phase2    | 1–2      | Herald v1.5 collector. Trace ring buffer 200 entries |
| Sentinel  | v0.3.0-phase3    | 1–3      | Herald v1.5 collector. Race fix (T2-A). engine_test S-001–S-008 |
| Canon     | v0.3.0           | —        | identity constants, default addrs, descriptor package |
| ZP        | v2.0.0           | —        | packaging tool, LoadFromID dead code removed |
| Arbiter   | v0.1.0           | —        | Enforcement — ADR-047, packaging + execution gate |
| Relay     | v0.1.0           | —        | Control: public tunnel :9090/:9091, ADR-041 |
| Conduit   | v0.1.0           | —        | Control: remote agent bridge :9092/:9093, ADR-042 |
| Accord    | v0.1.2           | —        | shared API types + upstream DTOs (Atlas, Forge, Guardian, NexusMetrics) |
| Herald    | v0.1.5           | —        | typed client — Nexus + Atlas + Forge + Guardian + NexusMetrics |

**All repos on `main` branch.**
**Tags:** v0.1.0-platform-working → v0.2.0-adr023-startup-grace → v0.3.0-cross-service-commands → v0.6.0-wave4 → v1.7.0 → v1.7.3

**Compiled binaries:** `~/bin/` — engxd, engx, engxa (installed via goreleaser pipeline)
**Service binaries:** `/tmp/bin/` — atlas, forge, metrics, navigator, guardian, observer, sentinel
**Service repos:** `~/workspace/projects/engx/services/<name>`

---

## 3. Thirteen Rules — Never Violate

| # | Rule | ADR |
|---|------|-----|
| 1 | Nexus is the only canonical project registry and filesystem observer | ADR-001, ADR-002 |
| 2 | Import `identity.ServiceTokenHeader` and `identity.TraceIDHeader` from Canon only — never redefine | ADR-016 |
| 3 | HTTP/JSON on 127.0.0.1 only — no gRPC, shared memory, message queues | ADR-003 |
| 4 | All Forge input becomes a Command object before the executor sees it | ADR-004 |
| 5 | Forge instructs Nexus via `POST /projects/:id/start\|stop` only | ADR-005 |
| 6 | All inter-service calls carry `X-Service-Token`. `/health` always exempt | ADR-008 |
| 7 | Observer services (8083–8087) are strictly read-only — never call write endpoints | ADR-020 |
| 8 | Sentinel AI called only on explicit `GET /insights/explain` — never on polling | ADR-018 |
| 9 | ADR-first — any new capability requires an ADR committed before implementation | Evo rules |
| 10 | Capability duplication is a design failure — check capability matrix before building | Cap boundaries |
| 11 | `engx register` auto-registers project + service from `.nexus.yaml` runtime section | ADR-022 |
| 12 | `engx platform start` resets fail counts before queuing — never start without reset | ADR-023 |
| 13 | `engx platform start` requires services to be registered — use `--register` on first boot | ADR-032 |
| 14 | `arbiter verify` must pass before `zp` produces any package — Canon violations block packaging | ADR-047 |

---

## 4. Service Token

```
Service token (forge / inter-service): 7d5fcbe4-44b9-4a8f-8b79-f80925c1330e
Atlas token (in service-tokens file):  f36150fa-a2a3-451b-b4d9-126027d07eb5
Agent token (local dev):               local-agent-token
```

For local dev: `~/.nexus/service-tokens` must be absent (move to `.bak`).
ServiceAuth is disabled when file is absent — engxa can then connect.

---

## 5. Canon Import Pattern

```go
import canon "github.com/Harshmaury/Canon/identity"

req.Header.Set(canon.ServiceTokenHeader, token)  // "X-Service-Token"
req.Header.Set(canon.TraceIDHeader, traceID)      // "X-Trace-ID"
```

Canon is in `go.mod` for all 8 service repos. Never hardcode header strings.

## 5b. Herald Usage Pattern

```go
// Nexus (events, services, agents, projects, metrics)
c := herald.New(nexusAddr, herald.WithToken(serviceToken))
svcs, err := c.Services().List(ctx)
evts, err := c.Events().Since(ctx, sinceID, 100)
m,   err := c.NexusMetrics().Get(ctx)

// Non-Nexus upstreams (Atlas, Forge, Guardian)
ac := herald.NewForService(atlasAddr, serviceToken)
projs, err := ac.Atlas().Projects(ctx)

fc := herald.NewForService(forgeAddr, serviceToken)
hist, err := fc.Forge().History(ctx, 100)

gc := herald.NewForService(guardianAddr, serviceToken)
report, err := gc.Guardian().Findings(ctx)
```

Never use raw `http.NewRequestWithContext` in observer collectors — always Herald.

---

## 6. ADR Status

| ADR | Title | Status |
|-----|-------|--------|
| ADR-001 | Project Registry Authority | ✅ Accepted |
| ADR-002 | Workspace Observation Ownership | ✅ Accepted |
| ADR-003 | Service Communication Protocol | ✅ Accepted |
| ADR-004 | Forge Intent Model | ✅ Accepted |
| ADR-005 | Forge → Nexus Lifecycle Protocol | ✅ Accepted |
| ADR-006 | Atlas as Context Source | ✅ Accepted |
| ADR-007 | Forge Automation Triggers | ✅ Accepted |
| ADR-008 | Inter-Service Authentication | ✅ Accepted |
| ADR-009 | Atlas Phase 3 nexus.yaml Contract | ✅ Accepted |
| ADR-010 | Forge Preflight Check | ✅ Accepted |
| ADR-011 | Metrics Observer | ✅ Accepted |
| ADR-012 | Navigator Observer | ✅ Accepted |
| ADR-013 | Guardian Observer | ✅ Accepted |
| ADR-014 | Observer Tracing | ✅ Accepted |
| ADR-015 | SSE Streaming | ✅ Accepted |
| ADR-016 | Platform Shared Types (Canon) | ✅ Accepted |
| ADR-017 | Sentinel Observer | ✅ Accepted |
| ADR-018 | Sentinel AI Reasoning | ✅ Accepted |
| ADR-019 | ZP Developer Packaging Tool | ✅ Accepted |
| ADR-020 | Observer Governance | ✅ Accepted |
| ADR-021 | PreflightSnapshot in Execution History | ✅ Accepted |
| ADR-022 | Service Registration API | ✅ Accepted |
| ADR-023 | Platform Startup Grace (Reset) | ✅ Accepted |
| ADR-024 | Sentinel Actuator Write Authority | ✅ Accepted |
| ADR-025 | engx init — nexus.yaml Generation | ✅ Accepted |
| ADR-026 | engxd System Service Install | ✅ Accepted |
| ADR-027 | Forge Scheduled Cron Triggers | ✅ Accepted |
| ADR-028 | engx Self-Upgrade Protocol | ✅ Accepted |
| ADR-029 | Doctor Extended Checks | ✅ Accepted |
| ADR-030 | goreleaser Release Pipeline | ✅ Accepted |
| ADR-031 | scripts/install.sh Zero-to-Running | ✅ Accepted |
| ADR-032 | platform start --register flag | ✅ Accepted + Shipped |
| ADR-033 | Accord — Shared API Types Module | ✅ Accepted |
| ADR-034 | Herald — Typed Nexus HTTP Client | ✅ Accepted |
| ADR-035 | engx deregister — Remove Ghost Projects | ✅ Accepted + Shipped (v1.7.0) |
| ADR-036 | GET /system/graph — Unified Topology Endpoint | ✅ Accepted + Shipped (v1.7.0) |
| ADR-037 | Signal System — Event Schema Enhancement | ✅ Accepted + Shipped |
| ADR-038 | POST /system/validate — Pre-Execution Policy Gate | ✅ Accepted + Shipped (v1.7.0) |
| ADR-039 | Herald Migration — Replace Internal Collectors | ✅ Accepted + **Shipped (2026-03-21)** |
| ADR-040 | Outcome-Centric UX: Progressive Disclosure | ✅ Accepted + Shipped (v1.7.0) |
| ADR-041 | Relay — engxa expose + tunnel architecture | ⏳ Future — ADR required before any code |
| ADR-042 | Gate — Platform Identity Authority (Ed25519 JWT) | ✅ Accepted + Shipped (Gate v1.0.0) |
| ADR-043 | Plan — Command Execution Model | ✅ Accepted + Shipped (Nexus v1.8.0) |
| ADR-046 | Conduit — remote execution routing | ⏳ Future — after ADR-041 |

---

## 7. Architecture Files

```
engx-governance/
  standards/documentation.md            documentation system
  definitions/glossary.md               canonical term definitions
  architecture/decisions/               ADR-001 through ADR-043
  architecture/v3-strategy.md           authoritative v3 spec (Relay → Gate → Conduit)
  architecture/platform-capability-boundaries.md
  architecture/architecture-evolution-rules.md
```

Service repos: each contains `SERVICE-CONTRACT.md`, `nexus.yaml`, `.nexus.yaml`, `.gitignore`

---

## 8. Open Items

**Remaining before v2.0.0 tag:**
- `go test ./...` across all services — must pass clean
- `engx doctor` clean on fresh platform start
- `platform status` command uses raw HTTP + anonymous struct in `cmd_platform.go` — minor, not a blocker but should be fixed

**Known non-blocking items:**
- `binary-versions` doctor check: version string not injected until goreleaser pipeline used for local builds
- Node.js 20 deprecation in GitHub Actions — forced to Node.js 24 from June 2 2026; set `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` in workflow files to resolve now
- Workspace topic constants (`workspace.file.created` etc.) live in `nexus/pkg/events` — should migrate to Canon before v3. Not a v2.0.0 blocker.

**Completed this session (2026-03-21):**
- ✅ ADR-039: Full Herald migration across all 5 observer collectors
- ✅ Accord v0.1.2: upstream DTOs for Atlas, Forge, Guardian, NexusMetrics
- ✅ Herald v0.1.5: NewForService + Atlas/Forge/Guardian/NexusMetrics typed clients
- ✅ T2-A: Sentinel race condition fixed (lastEventID mutex + DeployRisk uses StateStore)
- ✅ ADR-032: `engx platform start --register` confirmed shipped

---

## 9. Release Discipline (adopted 2026-03-21)

**`main` is stabilised first. Tags are cut only when the stability gate passes.**

Stability gate (must all pass before v2.0.0 tag):
```
□ go build ./... passes — all services
□ go test ./...  passes — all services
□ engx doctor clean on a fresh start
□ engx run <each service> succeeds
□ No known P0/P1 bugs open
□ AI_CONTEXT.md reflects actual state
□ RUNBOOK.md reflects actual commands
```

Fix commits carry no version bump. One tag when the gate passes.

---

## 10. Startup Sequence (current correct procedure)

```bash
# 1. Start daemon
engxd &
sleep 2

# 2. First boot — register + start in one command (ADR-032 shipped)
engx platform start --register

# 3. Start agent
/tmp/bin/engxa --id local --server http://127.0.0.1:8080 \
  --token local-agent-token --addr 127.0.0.1:9090 &

# 4. Verify
engx doctor
```

---

## 11. v3 Architecture (future — not started. Read v3-strategy.md before any v3 work)

v3 is an additive expansion layer over v2. Nothing in v2 is replaced.
**No v3 code exists. No v3 ADRs are drafted. v3 begins only after this section is activated.**

```
Gate      — identity authority          ✅ ALREADY BUILT (v1.0.0, ADR-042)
Phase 1 — Relay    engxa expose → public HTTPS endpoint (*.engx.dev)   ⏳ future
Phase 2 — Conduit  engxa run --on <machine> remote execution routing    ⏳ future
```

**Gate is already built as a v2 platform primitive (ADR-042, Gate v1.0.0).**
Relay and Conduit are not started. Two new services remaining. No existing service modified.

Inter-service rules for v3 (absolute — apply when v3 work begins):
- All v3 → v2 calls use Herald. No raw HTTP.
- All cross-boundary types use Accord. No local anonymous structs.
- All header strings from Canon. No literals.
- Gate is the only identity authority. Relay and Conduit validate via Gate (already operational).
- Conduit routes through Forge, never calls Nexus start/stop directly.
- ADR-first: ADR-041 must be committed before any Relay code is written.

---

## 12. v3 Robustness Framework (adopted 2026-03-21)

Before ADRs are written for v3 services, the following areas require hypothesis-driven
exploration and validation. No exploration result becomes permanent without an ADR.
No ADR is written until validation is complete.

### Exploration domains

**Network & Transport**
- Tunnel resilience: auto-reconnect, backpressure handling in Relay
- Herald timeout policy review: existing 10s/3-retry baseline, evaluate per-service tuning
- Circuit breaker patterns per upstream client

**Security & Trust**
- TLS enforcement at Relay external boundary
- Token expiry + rotation model for Gate-issued tokens
- Replay attack prevention (nonce or timestamp window)
- Request signing between Relay ↔ Gate

**Execution Resilience**
- Idempotent execution design for Conduit dispatch
- Failure classification: transient vs terminal in remote execution
- Dead-letter handling for failed remote executions
- Graceful degradation when target machine is unreachable

**Observability**
- Correlation ID propagation through Relay → Gate → Conduit (via Herald X-Trace-ID)
- Distributed trace assembly across machine boundaries
- Structured logging consistency in v3 services

**Operational**
- Health check depth: liveness vs readiness vs dependency health
- Graceful shutdown guarantees under active tunnel connections
- Configuration validation on startup

### Process per exploration
1. Define hypothesis: problem + approach + expected impact
2. Implement in isolation — no cross-service coupling
3. Stress test: network failure, partial unavailability, malformed input
4. Evaluate: determinism improvement, operational risk, complexity cost
5. Decision: accept → ADR, reject → document reasoning, defer → revisit

### Current exploration candidates (ordered by v3 phase dependency)
| # | Area | Phase | Status |
|---|------|-------|--------|
| 1 | Tunnel resilience — Relay reconnection model | Phase 1 | Not started |
| 2 | Herald timeout policy — review existing 10s/3-retry baseline | All | Not started |
| 3 | Token validation + expiry model | Gate (v2) | ✅ Shipped — Ed25519 JWT, ADR-042 |
| 4 | Correlation ID propagation across v3 boundary | All | Not started |
| 5 | Graceful shutdown under active tunnel connections | Phase 1 | Not started |
