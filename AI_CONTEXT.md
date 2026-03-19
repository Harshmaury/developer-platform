# AI_CONTEXT.md

Context document for AI systems working within this developer platform.
Read this file at the start of any session involving platform architecture,
new service design, code changes, or cross-project integration work.

**Updated:** 2026-03-19 | **Tag:** v0.3.0-cross-service-commands

---

## 1. What This Platform Is

A local developer control plane. Three capability layers, ten services, one CLI.

```
Control    Nexus   :8080   coordinates everything — sole registry, sole writer
Knowledge  Atlas   :8081   understands the workspace — capability graph, verified projects
Execution  Forge   :8082   acts on the workspace — build, test, run, deploy
Observer   5 svcs  :8083–:8087   read-only — never call write endpoints
Library    Canon   —       shared types — ServiceTokenHeader, TraceIDHeader, defaults
Tool       ZP      —       packaging — builds ZIPs from nexus.yaml
```

This repository governs the platform. It contains no implementation code.

---

## 2. Current Platform State (2026-03-19)

| Service   | Version          | Phase    | Key State                              |
|-----------|------------------|----------|----------------------------------------|
| Nexus     | v1.2.0-phase16   | 1–16     | ADR-022 service register, ADR-023 reset |
| Atlas     | v0.5.0-phase3    | 1–3      | nexus.yaml contract, verified graph    |
| Forge     | v0.5.0-phase4    | 1–4      | PreflightSnapshot, execution history   |
| Metrics   | v0.1.0-phase1    | 1        | Canon headers, event limit 500         |
| Navigator | v0.1.0-phase1    | 1        | Canon headers, trace propagation       |
| Guardian  | v0.1.0-phase1    | 1        | Canon headers, 5 policy rules          |
| Observer  | v0.1.0-phase1    | 1        | Canon v0.3.0, trace assembler          |
| Sentinel  | v0.2.0-phase2    | 1–2      | AI on-demand, lastEventID mutex        |
| Canon     | v0.3.0           | —        | identity constants, default addrs      |
| ZP        | v2.0.0           | —        | packaging tool                         |

**All repos on `main` branch. Tags:** v0.1.0-platform-working → v0.2.0-adr023-startup-grace → v0.3.0-cross-service-commands

**Compiled binaries:** `/tmp/bin/` — engxd, engx, engxa, atlas, forge, metrics, navigator, guardian, observer, sentinel

---

## 3. Twelve Rules — Never Violate

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
| ADR-024 | engx init — Project Onboarding | 🔜 Next |

---

## 7. Architecture Files

```
developer-platform/
  standards/documentation.md            documentation system
  definitions/glossary.md               canonical term definitions
  architecture/decisions/               ADR-001 through ADR-023
  architecture/platform-capability-boundaries.md
  architecture/architecture-evolution-rules.md
```

Service repos: each contains `SERVICE-CONTRACT.md`, `nexus.yaml`, `.gitignore`

---

## 8. Open Items

- ADR-024: `engx init` — user project onboarding command (generates `.nexus.yaml`)
- `start.sh` — single boot script for distribution
- Atlas auth: engx `check` command requires `--token` flag until Atlas auth is normalised
- `engx build` requires `--path` flag until Atlas is reliably reachable
