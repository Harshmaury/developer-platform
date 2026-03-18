# Platform Glossary

Canonical definitions for terms used across the engx developer platform.
One meaning per term. All documents and ADRs reference these definitions.
Do not redefine these terms locally in service documentation.

Updated: 2026-03-18

---

## Event

A structured signal emitted by Nexus when something changes in the platform
or workspace. Consumed by all observer services via polling `GET /events?since=<id>`
or streaming `GET /events/stream`.

**Delivery model:** at-least-once via cursor-based polling. Ordering: by Nexus event ID (monotonically increasing integer). Consumers must be idempotent — duplicate delivery is possible if a poll fails mid-cycle.

**Owner:** Nexus (ADR-001, ADR-002)
**Code:** `github.com/Harshmaury/Nexus/pkg/events/topics.go` — topic constants
**Canon:** `github.com/Harshmaury/Canon/events/events.go` — EventType constants

---

## Intent

A developer-expressed desired action before it is structured into a Command.
Raw strings, CLI input, automation triggers — all begin as intent and are
translated into a Command object before the executor sees them.

**Owner:** Forge (ADR-004)
**Code:** `github.com/Harshmaury/Forge/internal/command/model.go`

---

## Command

The structured five-field object (id, intent, target, parameters, context)
that is the only input Forge's executor accepts. All intent paths — CLI,
workflow, automation trigger — produce a Command before execution.
The schema is fixed. Extensions are additive only.

**Owner:** Forge (ADR-004)
**Code:** `github.com/Harshmaury/Forge/internal/command/model.go`

---

## Context

Resolved workspace data from Atlas used by Forge at execution time.
Captured once per workflow run before the step loop begins — not re-queried
per step. Atlas provides context facts; Forge uses them to enrich Commands.

**Owner:** Atlas produces it. Forge consumes it. (ADR-006)
**Code:** `github.com/Harshmaury/Forge/internal/context/resolver.go`

---

## Finding

A read-only policy evaluation result produced by Guardian. Findings are
non-blocking and carry no execution authority. Guardian never starts,
stops, or triggers any platform action. Findings are audit outputs only.

**Owner:** Guardian (ADR-013, ADR-020)
**Code:** `github.com/Harshmaury/Guardian/internal/policy/model.go`

---

## Trace

A correlated timeline of platform activity identified by a single X-Trace-ID
value. A trace connects Nexus events, Forge execution records, and observer
collection cycles that share the same trace ID. Assembled on demand by Observer.

**Owner:** Observer assembles. Nexus, Atlas, Forge propagate. (ADR-014, ADR-015)
**Code:** `github.com/Harshmaury/Observer/internal/trace/model.go`
**Header constant:** `github.com/Harshmaury/Canon/identity/identity.go` — `TraceIDHeader`

---

## Metric

An aggregated quantitative snapshot of platform health at a point in time.
Derived from Nexus runtime counters, Forge execution history, and Atlas
workspace state. Non-authoritative — reflects collected data, not system truth.

**Relationship to events:** events are discrete state transitions; metrics are continuous aggregations derived from event streams and runtime counters. They describe different views of the same system — both are correct, neither supersedes the other.

**Owner:** Metrics (ADR-011)
**Code:** `github.com/Harshmaury/Metrics/internal/snapshot/model.go`

---

## State

The derived workspace topology model built from the Atlas project graph.
Non-authoritative — Navigator computes state from Atlas data. The source
of truth for workspace topology is Atlas. The source of truth for service
runtime state is Nexus.

**Owner:** Navigator (ADR-012)
**Code:** `github.com/Harshmaury/Navigator/internal/topology/model.go`

---

## PreflightSnapshot

An immutable record of the Atlas graph state at the moment Forge authorizes
a command execution. Captured once by `preflight.Checker.Check()` and passed
by value through the execution pipeline — never re-queried between check and
history log. Stored in `execution_history.preflight_snapshot_json` (ADR-021).

**Owner:** Forge (ADR-021)
**Code:** `github.com/Harshmaury/Forge/internal/preflight/checker.go`
