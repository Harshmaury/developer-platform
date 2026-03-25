// @engx-governance-project: engx-governance
// @engx-governance-path: CONTEXT.md
# CONTEXT.md
# @version: 1.0.0
# @updated: 2026-03-25

System-wide invariants and capability ownership. Falsifiable against code.

---

## Capability ownership

| Capability | Owner | Code location |
|---|---|---|
| Project registry | Nexus | `internal/state/db.go → projects table` |
| Service state (desired + actual) | Nexus | `internal/state/db.go → services table` |
| Runtime reconciliation | Nexus | `internal/daemon/reconciler.go` |
| Filesystem observation | Nexus | `internal/watcher/` |
| Platform event bus | Nexus | `internal/eventbus/bus.go` |
| Platform event log | Nexus | `internal/state/db.go → events table` |
| Workspace capability graph | Atlas | `internal/graph/builder.go` |
| `nexus.yaml` validation | Atlas | `internal/validator/nexus_yaml.go` |
| Project verification status | Atlas | `internal/store/db.go → status field` |
| Command execution | Forge | `internal/executor/engine.go` |
| Execution history | Forge | `internal/store/db.go → execution_history` |
| Workflow definitions | Forge | `internal/store/db.go → workflows` |
| Automation triggers | Forge | `internal/store/db.go → triggers` |
| Identity token issuance | Gate | `internal/identity/token.go` |
| Identity token validation | Gate | `internal/api/handler/validate.go` |
| Public tunnel registry | Relay | `internal/tunnel/registry.go` |
| Shared header/event constants | Canon | `identity/`, `events/`, `descriptor/` |
| API DTO types | Accord | `api/types.go`, `api/upstream.go` |
| HTTP client to Nexus | Herald | `client/` |
| Platform metrics snapshot | Metrics | in-memory `SnapshotStore` |
| Workspace topology graph | Navigator | in-memory `AtlasCollector` |
| Runtime policy findings | Guardian | in-memory `ReportStore` |
| Distributed trace ring buffer | Observer | in-memory `trace.Store` |
| AI diagnostic insights | Sentinel | in-memory `StateStore` |
| Packaging enforcement | Arbiter | `internal/engine/engine.go` |
| ZIP delivery | ZP | `internal/pack/zipper.go` |

---

## What Nexus data is guaranteed

- Project registry is append-only-by-design. Deregistration marks; it does not delete.
- `events` table is append-only. `GET /events?since=<id>` is a stable cursor contract.
- Service state is single-writer: only the reconciler goroutine writes desired/actual state.
- SQLite is the only persistent store. No other service writes to Nexus's DB.

---

## What observer data is NOT guaranteed

All five observer services (Metrics, Navigator, Guardian, Observer, Sentinel) hold derived, non-persistent, point-in-time data. Lost on restart. Not authoritative. Source of truth is always the upstream service.

---

## Thirteen rules (enforced)

| # | Rule | Enforcement |
|---|------|-------------|
| 1 | Nexus is the only project registry and filesystem observer | Arbiter A-A-003 |
| 2 | Canon is the only source of protocol constants | Arbiter A-C-001/002 |
| 3 | HTTP/JSON on 127.0.0.1 only | Arbiter A-S-001 |
| 4 | All Forge input becomes a Command before the executor | Arbiter A-A-002 |
| 5 | Forge instructs Nexus via `POST /projects/:id/start\|stop` only | Arbiter A-A-002 |
| 6 | All inter-service calls carry `X-Service-Token`. `/health` exempt | Auth middleware |
| 7 | Observer services (8083–8087) never call write endpoints | Arbiter A-A-001 |
| 8 | Sentinel AI called only on `GET /insights/explain` | `internal/api/handler/explain.go` |
| 9 | ADR committed before any new capability is implemented | Arbiter A-T-001 |
| 10 | Capability duplication is a design failure | Arbiter A-T-002 |
| 11 | `engx register` auto-registers project + service from `nexus.yaml` runtime section | `cmd/engx/cmd_onboard.go` |
| 12 | `engx platform start` resets fail counts before queuing | `cmd/engx/cmd_platform.go` |
| 13 | `arbiter verify` must pass before `zp` produces any package | `zp/internal/gate/arbiter.go` |

---

## v3 boundary rules (apply when v3 work begins)

- All v3→v2 calls: Herald only. No raw HTTP.
- All cross-boundary types: Accord only. No local anonymous structs.
- All header strings: Canon only. No literals.
- Gate is the only identity authority. Relay and Conduit validate via Gate.
- Conduit routes through Forge. Never calls Nexus start/stop directly.
- ADR-041 must be committed before any Relay code is written.
- ADR-046 must be committed before any Conduit code is written.
