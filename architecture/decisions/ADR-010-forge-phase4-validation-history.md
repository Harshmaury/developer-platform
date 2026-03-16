# ADR-010 — Forge Phase 4: Pre-Execution Validation + Workflow History

**Status:** Accepted  
**Date:** 2026-03-17  
**Author:** Harsh Maury  
**Scope:** Forge execution service  
**Depends on:** ADR-009 (Atlas Phase 3) — requires GET /graph/services to be live

---

## Context

Forge Phase 1-3 translates developer intent into commands and executes them
via Nexus. However, commands are executed against any target string without
verifying that the project exists in the Atlas workspace graph, that its
declared capabilities match the requested action, or that the runtime
provider is permitted. This means a typo in a project name or a command
against an unregistered project silently fails at the Nexus level rather
than being caught early by Forge with a clear error.

Additionally, Forge has no queryable execution history. Workflows run and
their results exist only in memory during execution. There is no way for
the developer or future observer services (Guardian, Observer) to audit
what ran, when it ran, and what the outcome was.

Atlas Phase 3 (ADR-009) has stabilised GET /graph/services as a contract
endpoint returning verified projects with capabilities. This is the fact
surface Forge Phase 4 will use for pre-execution validation.

---

## Decision

### 1. Pre-execution validation middleware in the command pipeline

Before the executor sees any command, Forge will run a validation step that:

1. Queries Atlas `GET /graph/services` for the command target project
2. Checks the project exists and has `status: verified`
3. Checks the requested intent maps to a declared capability (if capabilities are declared)
4. Makes the permit/deny decision internally — Atlas provides facts only

**Capability mapping:**

| Intent  | Required capability (if declared) |
|---------|----------------------------------|
| build   | (no capability gate — always permitted for verified projects) |
| test    | (no capability gate) |
| run     | (no capability gate) |
| deploy  | (no capability gate) |

Phase 4 does not gate on capabilities — that is Phase 5 territory. Phase 4
only enforces that the target project is `verified` in Atlas. Unverified
projects are rejected with a clear error message.

**ADR-006 constraint:** Atlas provides facts. Forge decides policy.
Atlas never returns permit/deny — it returns project data. Forge's
`PreflightChecker` reads that data and makes the decision.

### 2. Workflow execution history

Every command execution is persisted to a new `execution_history` table
in `forge.db`. Each record contains:

- `id` — UUID
- `command_id` — the Command.ID
- `intent`, `target` — from the command
- `trace_id` — X-Trace-ID from the request context
- `status` — `running` | `success` | `failure`
- `output`, `error` — execution result
- `duration_ms` — wall-clock milliseconds
- `started_at`, `finished_at`

Two new endpoints expose this history:

- `GET /history` — paginated list, most recent first
- `GET /history/:trace_id` — all executions for a trace ID

### 3. X-Trace-ID propagation

Forge wires `middleware.TraceID` into its HTTP server (same pattern as
Nexus Phase 15 and Atlas Phase 3). All outbound calls to Nexus and Atlas
forward the trace ID via `X-Trace-ID` header.

---

## Implementation — Forge Phase 4 scope

### New files
- `internal/preflight/checker.go` — Atlas query + permit/deny logic
- `internal/preflight/checker_test.go` — table-driven tests
- `internal/api/middleware/traceid.go` — X-Trace-ID middleware

### Modified files
- `internal/store/db.go` — migration v3: execution_history table
- `internal/store/storer.go` — LogExecution, GetHistory, GetHistoryByTrace
- `internal/atlas/client.go` — GetVerifiedServices() using /graph/services
- `internal/api/handler/commands.go` — preflight check before execution
- `internal/api/handler/history.go` — new handler for GET /history endpoints
- `internal/api/server.go` — register history routes + TraceID middleware

---

## Consequences

**Positive:**
- Commands against non-existent or unverified projects are rejected early
- Full audit trail of every execution queryable by trace ID
- Observer services (Guardian) can poll /history for policy auditing
- X-Trace-ID connects Forge executions to Nexus event log entries

**Negative:**
- Forge now has a hard runtime dependency on Atlas being available
  (previously soft — Atlas was queried for context enrichment only)
- Projects without nexus.yaml are blocked from execution until they
  add a descriptor (acceptable trade-off per ADR-009)

**Mitigation:** If Atlas is unreachable, Forge logs a WARNING and falls
back to permitting execution (fail-open). This preserves local dev
usability when Atlas is not running.

---

## Compliance

| ADR | Status |
|-----|--------|
| ADR-003 | ✅ HTTP/JSON only — no direct Atlas DB access |
| ADR-004 | ✅ All input still becomes a Command before executor sees it |
| ADR-005 | ✅ Forge still instructs Nexus via POST /projects/:id/start\|stop only |
| ADR-006 | ✅ Atlas provides facts. Forge decides policy. |
| ADR-008 | ✅ X-Service-Token on all inter-service calls |

---

## Next ADR

ADR-011 — Metrics observer service (port 8083).
Depends on Forge Phase 4 history endpoint being tagged and stable.
