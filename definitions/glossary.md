# Platform Glossary

Canonical definitions for terms used across the engx developer platform.
One meaning per term. All documents and ADRs reference these definitions.
Do not redefine these terms locally in service documentation.

**Updated:** 2026-03-19

---

## Command

The structured five-field object (`id`, `intent`, `target`, `parameters`, `context`)
that is the only input Forge's executor accepts. All intent paths — CLI, workflow,
automation trigger — produce a Command before execution. The schema is fixed.
Extensions are additive only.

**Owner:** Forge (ADR-004)
**Code:** `github.com/Harshmaury/Forge/internal/command/model.go`

---

## Context

Resolved workspace data from Atlas used by Forge at execution time.
Captured once per command before execution — not re-queried mid-execution.
Atlas provides context facts; Forge uses them to enrich Commands.

**Owner:** Atlas produces it. Forge consumes it. (ADR-006)
**Code:** `github.com/Harshmaury/Forge/internal/context/resolver.go`

---

## Event

A structured signal emitted by Nexus when something changes in the platform
or workspace. Immutable once written. Consumed by all observer services via
cursor-based polling: `GET /events?since=<lastEventID>&limit=<n>`.

Events are not metrics. Events are discrete state transitions; metrics are
continuous aggregations. Both describe the same system — neither supersedes the other.

**Owner:** Nexus emits. Observers consume. (ADR-001, ADR-002)
**Code:** `github.com/Harshmaury/Nexus/internal/state/db.go` — `Event` struct
**Topics:** `github.com/Harshmaury/Nexus/pkg/events/topics.go`

---

## Finding

A read-only policy evaluation result produced by Guardian. Findings are
non-blocking and carry no execution authority. Guardian never starts,
stops, or triggers any platform action. Findings are audit outputs only.

**Owner:** Guardian (ADR-013, ADR-020)
**Code:** `github.com/Harshmaury/Guardian/internal/policy/model.go`

---

## Intent

A developer-expressed desired action before it is structured into a Command.
Raw strings, CLI input, automation triggers — all begin as intent and are
translated into a Command object before the executor sees them.

**Owner:** Forge (ADR-004)
**Code:** `github.com/Harshmaury/Forge/internal/command/model.go`

---

## Metric

An aggregated quantitative snapshot of platform health at a point in time.
Derived from Nexus runtime counters, Forge execution history, and Atlas
workspace state. Non-authoritative — reflects collected data, not system truth.

**Owner:** Metrics (ADR-011)
**Code:** `github.com/Harshmaury/Metrics/internal/snapshot/model.go`

---

## PreflightSnapshot

An immutable record of the Atlas graph state at the moment Forge authorizes
a command execution. Captured once by `preflight.Checker.Check()` before
execution begins — never re-queried between check and history log.
Stored in `execution_history.preflight_snapshot_json` (ADR-021).

**Owner:** Forge (ADR-021)
**Code:** `github.com/Harshmaury/Forge/internal/preflight/checker.go`

---

## Project

A registered workspace directory with a known `nexus.yaml` descriptor.
Nexus is the sole registry. Atlas indexes projects and determines
`verified`/`unverified` status from `nexus.yaml` validity.
A project is not a service — one project may have zero or more services.

**Owner:** Nexus registers. Atlas verifies. (ADR-001, ADR-009)
**Code:** `github.com/Harshmaury/Nexus/internal/state/db.go` — `Project` struct

---

## Service

A managed process associated with a project. Nexus tracks desired and actual
state. engxa reconciles desired state by starting or stopping the process.
Services persist in SQLite across Nexus restarts. Registered via
`POST /services/register` (ADR-022).

**Owner:** Nexus (ADR-022, ADR-023)
**Code:** `github.com/Harshmaury/Nexus/internal/state/db.go` — `Service` struct

---

## State

The derived workspace topology model built from the Atlas project graph.
Non-authoritative — Navigator computes state from Atlas data. The source
of truth for workspace topology is Atlas. The source of truth for service
runtime state is Nexus.

**Owner:** Navigator (ADR-012)
**Code:** `github.com/Harshmaury/Navigator/internal/topology/model.go`

---

## Trace

A correlated timeline of platform activity identified by a single `X-Trace-ID`
value. A trace connects Nexus events, Forge execution records, and observer
collection cycles that share the same trace ID. Assembled on demand by Observer.

Each observer generates a per-cycle trace ID with a service prefix:
`mt-` Metrics, `nv-` Navigator, `gd-` Guardian, `st-` Sentinel.

**Owner:** Observer assembles. Nexus, Atlas, Forge propagate. (ADR-014, ADR-015)
**Code:** `github.com/Harshmaury/Observer/internal/trace/model.go`
**Header constant:** `github.com/Harshmaury/Canon/identity/identity.go` — `TraceIDHeader`

---

## Event Delivery Model

Events are delivered at-least-once on successful polls. Successful polls
advance the `lastEventID` cursor — each event is fetched exactly once per
consumer. Failed polls replay from the last successful cursor position.
No deduplication is needed on success; the cursor is the deduplication mechanism.
