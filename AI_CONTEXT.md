# AI_CONTEXT.md

Context document for AI systems working within this developer platform.
Read this file at the start of any session involving platform architecture,
new service design, or cross-project integration work.

Updated: 2026-03-18

---

## What This Platform Is

A local developer control plane built on three capability domains plus
five read-only observer services, a shared types library, and a packaging tool:

    Control    Nexus     coordinates the system   :8080
    Knowledge  Atlas     understands the system   :8081
    Execution  Forge     acts on the system       :8082
    Observer   Metrics   measures the system      :8083
    Observer   Navigator maps the system          :8084
    Observer   Guardian  audits the system        :8085
    Observer   Observer  traces the system        :8086
    Observer   Sentinel  reasons about the system :8087
    Library    Canon     shared types + constants  (no port)
    Tool       zp        packages the system       (no port)

Each service is a separate Go project with its own repository.
This repository governs the platform — it contains no implementation code.

---

## Platform Status (2026-03-18)

| Service   | Phase    | Tag               | Build |
|-----------|----------|-------------------|-------|
| Nexus     | 1–16     | v1.2.0-phase16    | ✅    |
| Atlas     | 1–3      | v0.5.0-phase3     | ✅    |
| Forge     | 1–4      | v0.5.0-phase4     | ✅    |
| Metrics   | 1        | v0.1.0-phase1     | ✅    |
| Navigator | 1        | v0.1.0-phase1     | ✅    |
| Guardian  | 1        | v0.1.0-phase1     | ✅    |
| Observer  | 1        | v0.1.0-phase1     | ✅    |
| Sentinel  | 2        | v0.2.0-phase2     | ✅    |
| Canon     | —        | v0.1.0            | ✅    |
| zp        | —        | v2.0.0            | ✅    |

All services build clean as of 2026-03-18.

---

## Key Rules for AI Systems

**1. Capability ownership is fixed.**
Before suggesting a feature, check `architecture/platform-capability-boundaries.md`.
Every capability has exactly one owner. Duplication is a design failure.

**2. ADRs gate implementation.**
No new platform capability is built without an ADR in `architecture/decisions/`.
If a proposed change does not have an ADR, the ADR comes first.
Next ADR: ADR-022.

**3. Projects are independent.**
Atlas and Forge do not import Nexus internal packages.
The only permitted cross-module import is `github.com/Harshmaury/Nexus/pkg/events`
for workspace topic constants — never `internal/eventbus`.
Integration is always HTTP API or event subscription.

**4. Nexus owns three things permanently.**
Project registry (ADR-001), filesystem observation (ADR-002),
and service runtime state. These never move to another service.

**5. Import from Canon — never redefine.**
All platform-wide constants (header names, event types, service names,
nexus.yaml descriptor schema) live in `github.com/Harshmaury/Canon`.
Import Canon — never hardcode `"X-Service-Token"`, `"X-Trace-ID"`, or
event type strings locally in any service.

**6. Event topics are declared in one place.**
Topic constants live in Nexus `internal/eventbus/bus.go`,
re-exported via `pkg/events`. External consumers import from `pkg/events` only.

**7. Forge command schema is fixed.**
The five-field command object (id, intent, target, parameters, context)
is the ADR-004 contract. Suggest extensions additively, never breaking changes.

**8. Context enrichment runs once per workflow run.**
`ResolveContext` (Atlas + Nexus lookup) is called once before the step loop,
not once per step. All steps share the same base context. (ADR-006 Rule 4)

**9. Trigger dispatch is bounded.**
Forge caps concurrent workflow goroutines at 8 via semaphore.
Triggers are best-effort — dropped under load with a WARNING log. (ADR-007)

**10. All migrations in one place.**
Every service keeps all schema migrations in a single ordered slice in `db.go`.
Never in `init()` functions in separate files.

**11. Observer services are strictly read-only.**
Metrics, Navigator, Guardian, Observer, Sentinel (ports 8083–8087) must never
call POST /projects/:id/start|stop, POST /commands, or any write endpoint.
Governed by ADR-020. See each service's SERVICE-CONTRACT.md.

**12. Sentinel AI is on-demand only.**
The Anthropic API (claude-sonnet-4-6) is called ONLY when the developer
explicitly requests GET /insights/explain. Never on background polling cycles.
ANTHROPIC_API_KEY env var. Graceful degradation if absent. (ADR-018)

**13. PreflightSnapshot is immutable.**
Forge captures the Atlas graph state at preflight check time and passes it
by value through the execution pipeline. Never re-queried between check and
history log. Stored in execution_history.preflight_snapshot_json. (ADR-021)

**14. Platform authority hierarchy.**
Sentinel suggests → Guardian flags → Developer decides → Forge executes → Nexus controls.
The Developer node is mandatory and cannot be bypassed by automation.
See `workflow-philosophy.md` Platform Authority Model section.

---

## Platform Architecture Files

This repository:
```
architecture/decisions/          ADR-001 through ADR-021
architecture/platform-capability-boundaries.md
architecture/architecture-evolution-rules.md
definitions/glossary.md          Canonical term definitions
standards/navigation.md          File-level map of all services
workflow-philosophy.md           Design philosophy + authority model
```

Service repositories:
```
github.com/Harshmaury/Nexus      v1.2.0-phase16
github.com/Harshmaury/Atlas      v0.5.0-phase3
github.com/Harshmaury/Forge      v0.5.0-phase4
github.com/Harshmaury/Metrics    v0.1.0-phase1
github.com/Harshmaury/Navigator  v0.1.0-phase1
github.com/Harshmaury/Guardian   v0.1.0-phase1
github.com/Harshmaury/Observer   v0.1.0-phase1
github.com/Harshmaury/Sentinel   v0.2.0-phase2
github.com/Harshmaury/Canon      v0.1.0
github.com/Harshmaury/ZP         v2.0.0
```

Each service repo contains a `SERVICE-CONTRACT.md` at its root.
Read it before writing any code for that service.

---

## Open Architectural Gaps

None at this time.

All previously identified gaps have been resolved:
- Inter-service authentication → ADR-008 (X-Service-Token, all services)
- Observer concurrency → fixed 2026-03-18 (sync.RWMutex, atomic snapshots)
- Canon header constants → all services import identity.ServiceTokenHeader
  and identity.TraceIDHeader — no hardcoded strings remain
- Forge execution context race → ADR-021, PreflightSnapshot implemented

Next candidate: ADR-022 — Forge Phase 5 capability-gated execution
(intent → required capability mapping, currently deferred from ADR-010).
No implementation until ADR is written and accepted.
