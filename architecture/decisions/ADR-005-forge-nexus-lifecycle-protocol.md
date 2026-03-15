# ADR-005 — Forge-to-Nexus Service Lifecycle Protocol

Date: 2026-03-15
Status: Accepted

---

## Context

Forge is the execution domain. When a developer submits a `run` or `deploy`
command, Forge must cause a service to start or stop. Nexus is the control
plane and owns service lifecycle authority (ADR-001). Forge must not manage
service state directly.

Two questions needed to be settled before Forge Phase 1 could be implemented:

1. Exactly which Nexus HTTP endpoints does Forge call to start and stop a service?
2. What does Forge do when Nexus is unreachable at the time of the call?

Without recording these decisions, future contributors could introduce direct
provider calls inside Forge, bypassing Nexus and breaking the capability
boundary.

## Decision

Forge requests service lifecycle changes exclusively through the Nexus HTTP API
using the following endpoints:

    Start a service:  POST http://127.0.0.1:8080/projects/:id/start
    Stop a service:   POST http://127.0.0.1:8080/projects/:id/stop

Both calls use the standard JSON request body and the standard response envelope
defined in ADR-003.

Forge treats a 200 or 202 response as success. Any other status code is treated
as a failure and reported in the command result. Forge does not retry — the
caller is responsible for resubmitting if needed.

## Degradation Behaviour

If Nexus is unreachable when Forge attempts a lifecycle call, the intent handler
returns a failed result immediately. Forge does not queue the request or attempt
a delayed retry. The command result clearly identifies that the failure was due
to Nexus being unreachable so the caller can take corrective action.

This behaviour is consistent with Forge's stateless Phase 1 design (ADR-004):
Forge does not maintain a queue of pending lifecycle requests.

## What Forge Must Never Do

- Call runtime providers (Docker, K8s, Process) directly
- Write to the Nexus SQLite database
- Maintain its own record of service runtime state
- Assume a service is running without confirming through Nexus

## Implications

- The `run` and `deploy` intent handlers in `internal/executor/intent/`
  call only the two endpoints above.
- Forge's Nexus client (`internal/nexus/client.go`) is the sole location
  where these endpoint paths are defined.
- If Nexus adds authentication to these endpoints in a future phase,
  only `internal/nexus/client.go` needs to change.

## Alternatives Considered

**Forge calls providers directly** — rejected because it duplicates Nexus
reconciliation logic and breaks the capability boundary. Two systems would
then manage the same service state, producing conflicts.

**Forge queues lifecycle requests when Nexus is down** — rejected because
Phase 1 is explicitly stateless (ADR-004). Queuing requires persistence
and retry logic that belongs in Phase 2 or later.

**Forge polls Nexus for confirmation that the service started** — rejected
for Phase 1. Nexus reconciles state asynchronously. Polling adds latency
and complexity without clear benefit at this stage. Can be revisited in
Phase 2 if workflow steps need to wait for service health before proceeding.

## Consequences

Service lifecycle from Forge always flows through Nexus. This makes Nexus
the single point of control for service state, which is exactly the
capability boundary ADR-001 establishes. Forge execution results accurately
reflect whether the Nexus instruction was accepted, not whether the service
is actually running — that distinction is visible through `engx services`.
