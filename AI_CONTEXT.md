# AI_CONTEXT.md

Context document for AI systems working within this developer platform.
Read this file at the start of any session involving platform architecture,
new service design, code changes, or cross-project integration work.

**Updated:** 2026-03-21 | **Tag:** v1.7.0

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
Client     Herald  —       typed HTTP client — all engxd API calls go through here
```

This repository governs the platform. It contains no implementation code.

---

## 2. Current Platform State (2026-03-21)

| Service   | Version          | Phase    | Key State                              |
|-----------|------------------|----------|----------------------------------------|
| Nexus     | v1.7.0           | 1–22     | Wave 5+6+UX: system/graph, system/validate, engx run/ps/deregister |
| Atlas     | v0.5.0-phase3    | 1–4      | nexus.yaml contract, verified graph, logger-threaded handlers |
| Forge     | v0.5.0-phase5    | 1–5      | CronScheduler dedup fixed + wired, preflight snapshot |
| Metrics   | v0.2.0-phase2    | 1–2      | Prometheus /metrics/prometheus endpoint |
| Navigator | v0.1.0-phase1    | 1        | Canon headers, trace propagation       |
| Guardian  | v0.2.0-phase2    | 1–2      | Per-cycle trace ID, WARNING logs on failure |
| Observer  | v0.2.0-phase2    | 1–2      | Trace ring buffer 200 entries          |
| Sentinel  | v0.3.0-phase3    | 1–3      | engine_test S-001–S-008, actuator policy_test |
| Canon     | v0.3.0           | —        | identity constants, default addrs, descriptor package |
| ZP        | v2.0.0           | —        | packaging tool, LoadFromID dead code removed |
| Accord    | v0.1.0           | —        | shared API types, error codes, Response[T] |
| Herald    | v0.1.0           | —        | typed Nexus HTTP client, retry/backoff |

**All repos on `main` branch.**
**Tags:** v0.1.0-platform-working → v0.2.0-adr023-startup-grace → v0.3.0-cross-service-commands → v0.6.0-wave4 → v1.7.0

**Compiled binaries:** `~/bin/` — engxd, engx, engxa (installed via goreleaser pipeline)
**Service binaries:** `/tmp/bin/` — atlas, forge, metrics, navigator, guardian, observer, sentinel

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
| ADR-032 | platform start Must Persist desired=running | ✅ Accepted |
| ADR-033 | Accord — Shared API Types Module | ✅ Accepted |
| ADR-034 | Herald — Typed Nexus HTTP Client | ✅ Accepted |
| ADR-035 | engx deregister — Remove Ghost Projects | ✅ Accepted + Shipped (v1.7.0) |
| ADR-036 | GET /system/graph — Unified Topology Endpoint | ✅ Accepted + Shipped (v1.7.0) |
| ADR-037 | Signal System — Event Schema Enhancement | ✅ Accepted + Shipped (Nexus d34f03a, Canon f493f89) |
| ADR-038 | POST /system/validate — Pre-Execution Policy Gate | ✅ Accepted + Shipped (v1.7.0) |
| ADR-039 | Herald Migration — Replace Internal Nexus Collectors | ✅ Accepted — **not yet implemented** |
| ADR-040 | Outcome-Centric UX: Progressive Disclosure | ✅ Accepted + Shipped (v1.7.0) |

---

## 7. Architecture Files

```
engx-governance/
  standards/documentation.md            documentation system
  definitions/glossary.md               canonical term definitions
  architecture/decisions/               ADR-001 through ADR-040
  architecture/platform-capability-boundaries.md
  architecture/architecture-evolution-rules.md
```

Service repos: each contains `SERVICE-CONTRACT.md`, `nexus.yaml`, `.nexus.yaml`, `.gitignore`

---

## 8. Open Items

- ~~ADR-037~~: shipped — DB migration v6, Event struct, AppendEvent, Canon LevelInfo/Warn/Error
- **ADR-039**: Herald migration — replace `internal/collector/nexus.go` in Guardian, Observer, Metrics, Navigator, Sentinel — not started
- **ADR-032**: `engx platform start --register` flag — implementation pending (startup sequence still uses manual loop)
- `binary-versions` doctor check: version string not injected until goreleaser pipeline used for local builds
- Node.js 20 deprecation in GitHub Actions — forced to Node.js 24 from June 2 2026; set `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true` in workflow files to resolve now

---

## 9. Release Discipline (adopted 2026-03-21)

**`main` is stabilised first. Tags are cut only when the stability gate passes.**

Stability gate (must all pass before any tag):
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

# 2. First boot only — register all services
for svc in atlas forge guardian metrics navigator observer sentinel; do
  engx register ~/workspace/projects/engx/services/$svc
done

# 3. Set desired=running and start
for svc in atlas forge guardian metrics navigator observer sentinel; do
  engx project start $svc
done

# 4. Start agent
/tmp/bin/engxa --id local --server http://127.0.0.1:8080 \
  --token local-agent-token --addr 127.0.0.1:9090 &

# 5. Verify
engx doctor
```

After ADR-032 `--register` ships, steps 2+3 collapse to: `engx platform start --register`
