// @engx-governance-project: engx-governance
// @engx-governance-path: CONTRACTS.md
# CONTRACTS.md
# @version: 1.0.0
# @updated: 2026-03-25

---

## Shared invariants

These are enforced in code. Violation fails either `arbiter verify` (static) or runtime auth middleware.

| ID | Invariant | Enforcement point |
|----|-----------|-------------------|
| I-01 | All inter-service calls carry `X-Service-Token`. `/health` exempt. | `internal/api/middleware/service_auth.go` in every service |
| I-02 | All header strings imported from `Canon/identity`. No literals. | Arbiter A-C-001 |
| I-03 | All event type strings imported from `Canon/events`. No literals. | Arbiter A-C-002 |
| I-04 | `nexus.yaml` type field must match `Canon/descriptor.ValidTypes`. | Atlas `internal/validator/nexus_yaml.go` |
| I-05 | All cross-service request/response types use Accord. No local anonymous structs. | Arbiter A-C-003 |
| I-06 | Observer services (ports 8083–8087) never call write endpoints. | Arbiter A-A-001 |
| I-07 | Forge instructs Nexus via `POST /projects/:id/start\|stop` only. | ADR-005, Arbiter A-A-002 |
| I-08 | All Forge input becomes `Command{id, intent, target, parameters, context}` before executor. | `internal/command/model.go` |
| I-09 | Nexus is the only service that watches the filesystem. | ADR-002, Arbiter A-A-003 |
| I-10 | Token comparison uses `crypto/subtle.ConstantTimeCompare`. | All auth middleware |
| I-11 | `ENGX_AUTH_REQUIRED=true` → missing token = `FATAL` at startup, `503` at middleware. | `internal/config/env.go` in every service |

---

## Service discovery

Services discover each other by environment variable address, defaulting to Canon constants:

```go
// Canon/identity
DefaultNexusAddr     = "http://127.0.0.1:8080"
DefaultAtlasAddr     = "http://127.0.0.1:8081"
DefaultForgeAddr     = "http://127.0.0.1:8082"
DefaultMetricsAddr   = "http://127.0.0.1:8083"
DefaultNavigatorAddr = "http://127.0.0.1:8084"
DefaultGuardianAddr  = "http://127.0.0.1:8085"
DefaultObserverAddr  = "http://127.0.0.1:8086"
DefaultSentinelAddr  = "http://127.0.0.1:8087"
DefaultGateAddr      = "http://127.0.0.1:8088"
```

No service hardcodes these strings. All calls go through Herald.

---

## State versions

State versioning is not yet implemented (planned ADR-050). Until then: no service depends on state ordering across service restarts.

---

## Conflict resolution

Not implemented. All write authority is centralised in Nexus. Conflict resolution is planned for Conduit (ADR-053) using `intent_priority` + `state_ref` ordering.

---

## Event payload schema registry

Defined in `Accord/api/payload.go → EventPayloadType`. Switch on `EventDTO.Type`, then call `accord.DecodePayload[T]`.

| Event type | Payload Go type |
|------------|-----------------|
| `SYSTEM_ALERT` | `Canon/events.SystemAlertPayload` |
| `PLAN_STEP` | `Accord/api.PlanSpanDTO` |
| `workspace.file.created\|modified\|deleted` | `Canon/events.WorkspaceFilePayload` |
| `workspace.updated` | `Canon/events.WorkspaceUpdatedPayload` |
| `workspace.project.detected` | `Canon/events.WorkspaceProjectPayload` |
| `SERVICE_STARTED\|STOPPED\|CRASHED\|HEALED\|STATE_CHANGED` | *(empty payload)* |

---

## Versioning rules

| Change class | Accord bump | Coordination required |
|---|---|---|
| Add optional field (`omitempty`) | patch | Accord only |
| Add new type | minor | Accord only |
| Rename field / change json tag | **major** | Nexus + Herald + Accord simultaneously |
| Remove field | **major** | Nexus + Herald + Accord simultaneously |
| Rename `ErrorCode` | **major** | All callers switch on `Code` |

Major version bump requires an ADR before any code change.

---

## Integration checklist for new services

A new service is compliant when it satisfies all of the following. `arbiter verify ./...` validates items marked (A).

- [ ] `nexus.yaml` present at repo root with valid `type` from `Canon/descriptor.ValidTypes` (A)
- [ ] `SERVICE-CONTRACT.md` present (A)
- [ ] `Canon/identity` imported; no header string literals (A)
- [ ] `Canon/events` imported; no event type string literals (A)
- [ ] All cross-service HTTP calls go through Herald (A)
- [ ] All cross-boundary types from Accord (A)
- [ ] `GET /health` returns `{"ok":true,"status":"healthy","service":"<n>"}`, no auth required
- [ ] All other endpoints require `X-Service-Token` via middleware
- [ ] `X-Trace-ID` middleware active on all responses
- [ ] Graceful shutdown: SIGINT/SIGTERM → 10s context → server shutdown
- [ ] Config from env only via `EnvOrDefault` / `ExpandHome`
- [ ] Token comparison via `crypto/subtle.ConstantTimeCompare`
- [ ] ADR committed before any new capability is implemented (A)
