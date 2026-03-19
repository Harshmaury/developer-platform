# Workflow Philosophy

The design philosophy governing how this developer platform is built
and how it evolves over time.

**Updated:** 2026-03-19

---

## Core Idea

The workstation is a system, not a collection of tools.

Tools are components. Components are orchestrated. The developer
interacts with a unified platform rather than running individual
commands manually.

---

## The Ten Constraints

These constraints govern every architectural decision on this platform.
A proposed change that violates any of these requires explicit justification
and an ADR before proceeding.

### 1. No Hard-Coded Project Structures

The architecture never assumes specific projects will permanently exist.
Projects may change, be replaced, be removed, or be rewritten.
Platform components treat projects as dynamic, not fixed.

### 2. Architecture Must Be Capability-Based

System descriptions are organised by capabilities, not implementations.
The three stable capability domains are:

```
Control    Nexus   coordinates the system
Knowledge  Atlas   understands the system
Execution  Forge   acts on the system
```

Observer services (Metrics, Navigator, Guardian, Observer, Sentinel) form a
fourth layer — they observe without acting. Specific projects implementing
these capabilities may change over time. The capability domains do not change.

### 3. Workflow Must Be Decoupled From Implementation

Workflows describe processes and responsibilities, not specific tools.
A workflow says what happens. It does not say which binary runs it.

### 4. Modular and Replaceable Components

Every subsystem is designed so it can be replaced without requiring
major changes to other components. Modules evolve, may be rewritten,
and may disappear. The system remains functional.

### 5. Independent Project Principle

Each project operates as an independent application.
Integration occurs through APIs, events, and CLI interfaces.
Projects never rely on internal implementation details of other projects.

### 6. Interface Contracts Over Direct Dependencies

Services communicate through documented HTTP/JSON contracts only.
No shared memory. No internal package imports across service boundaries.
The only permitted cross-module import is Canon (`github.com/Harshmaury/Canon`)
for shared constants — never internal packages.

### 7. Observer Isolation

Observer services (ports 8083–8087) are strictly read-only. They derive
signals from authoritative sources but never write to them. An observer
that calls a write endpoint is a design violation, not a bug.

### 8. ADR-First Evolution

No platform capability is built without an ADR committed first. The decision
record precedes the implementation — always. This is not process overhead;
it is the mechanism by which the platform stays coherent across sessions.

### 9. Canon as the Single Source of Shared Truth

Constants used across multiple services live in Canon. Services import from
Canon; they never redefine constants locally. A locally-defined constant that
already exists in Canon is a Canon violation.

### 10. Operational Correctness Over Theoretical Purity

The platform is a working system, not a research project. When a theoretical
concern conflicts with a confirmed operational need, the operational need wins —
provided the resolution is documented in an ADR. Runtime findings from actual
platform runs carry more weight than structural analysis of code snapshots.

---

## Platform Authority Model

Not all services have equal authority. The hierarchy is explicit:

**Tier 1 — Authoritative (writers)**
Nexus is the sole writer for project registry, service lifecycle, and events.
Forge is the sole writer for execution history and workflow state.
Atlas is the sole writer for the knowledge graph.

**Tier 2 — Reactive (CLI, agents)**
`engx` sends commands to Tier 1 services via HTTP.
`engxa` reconciles service state by starting and stopping processes.
Neither stores authoritative state.

**Tier 3 — Observational (read-only)**
Metrics, Navigator, Guardian, Observer, Sentinel derive signals from Tier 1.
They present, not decide. They report, not act.
Observer findings are advisory. Guardian findings are advisory.
Sentinel insights are advisory. None trigger platform actions.

**Canon — Cross-cutting**
Canon provides shared constants. It has no runtime component, no HTTP server,
no state. It is a library dependency, not a service.

---

## Workflows Replace Manual Commands

Instead of:
```
go build ./...
./nexus &
./atlas &
```

The developer uses:
```
engx build nexus
engx platform start
engx doctor
```

Forge executes the build. Nexus manages the lifecycle. Guardian and Sentinel
report what's happening. The developer expresses intent. The platform handles
coordination.

---

## The Platform Must Grow With the Developer

The system supports continuous evolution. New projects, new languages,
new tools, and new workflows are added without architectural redesign.
The capability domains are stable. The implementations within them evolve.

New capabilities follow this sequence:
1. Identify which capability domain it belongs to
2. Confirm no existing service already owns it
3. Write an ADR in `architecture/decisions/`
4. Implement in the correct service
5. Update `SERVICE-CONTRACT.md` and `AI_CONTEXT.md`
