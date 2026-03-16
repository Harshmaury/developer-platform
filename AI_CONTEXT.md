# AI_CONTEXT.md

Context document for AI systems working within this developer platform.
Read this file at the start of any session involving platform architecture,
new service design, or cross-project integration work.

Updated: 2026-03-16

---

## What This Platform Is

A local developer control plane built on three capability domains:

    Control    Nexus   coordinates the system   :8080
    Knowledge  Atlas   understands the system   :8081
    Execution  Forge   acts on the system       :8082

Each domain is a separate Go project with its own repository.
This repository governs the platform — it contains no implementation code.

---

## Platform Status (2026-03-16)

| Service | Phase | Tag                    | Build |
|---------|-------|------------------------|-------|
| Nexus   | 1–14 complete + ADR-002 | v1.0.0-fixes-complete | ✅ |
| Atlas   | Phase 1+2 complete      | v0.3.0-fixes-complete | ✅ |
| Forge   | Phase 1+2+3 complete    | v0.4.0-fixes-complete | ✅ |

All criticals and highs resolved as of 2026-03-16.

---

## Key Rules for AI Systems

**1. Capability ownership is fixed.**
Before suggesting a feature, check `architecture/platform-capability-boundaries.md`.
Every capability has exactly one owner. Duplication is a design failure.

**2. ADRs gate implementation.**
No new platform capability is built without an ADR in `architecture/decisions/`.
If a proposed change does not have an ADR, the ADR comes first.

**3. Projects are independent.**
Atlas and Forge do not import Nexus internal packages.
The only permitted cross-module import is `github.com/Harshmaury/Nexus/pkg/events`
for workspace topic constants — never `internal/eventbus`.
Integration is always HTTP API or event subscription.

**4. Nexus owns three things permanently.**
Project registry (ADR-001), filesystem observation (ADR-002),
and service runtime state. These never move to another service.

**5. Event topics are declared in one place.**
Topic constants live in Nexus `internal/eventbus/bus.go`.
External consumers import them from `pkg/events` — never redefine locally.

**6. Atlas phases are sequential.**
Phase 2 (graph, conflict detection) requires Phase 1 (index) to exist.

**7. Forge command schema is fixed.**
The five-field command object (id, intent, target, parameters, context)
is the ADR-004 contract. Suggest extensions additively, never breaking changes.

**8. Context enrichment runs once per workflow run.**
`ResolveContext` (Atlas + Nexus lookup) is called once before the step loop,
not once per step. All steps share the same base context. (ADR-006 rule 4)

**9. Trigger dispatch is bounded.**
Forge caps concurrent workflow goroutines at 8 via semaphore.
Triggers are best-effort — dropped under load with a WARNING log. (ADR-007)

**10. All migrations in one place.**
Every service keeps all schema migrations in a single ordered slice in `db.go`.
Never in `init()` functions in separate files.

---

## Platform Architecture Files

This repository:
  architecture/decisions/                 ADR-001 through ADR-007
  architecture/platform-capability-boundaries.md
  architecture/architecture-evolution-rules.md

Service repositories:
  github.com/Harshmaury/Nexus             v1.0.0-fixes-complete
  github.com/Harshmaury/Atlas             v0.3.0-fixes-complete
  github.com/Harshmaury/Forge             v0.4.0-fixes-complete

---

## Open Architectural Gaps

**Inter-service authentication (no ADR yet)**
Atlas→Nexus and Forge→Nexus/Atlas calls carry no token.
Any process on the machine can impersonate a service.
Must be resolved before any service is exposed beyond 127.0.0.1.
An ADR is required before implementation begins (per architecture-evolution-rules.md Rule 1).
