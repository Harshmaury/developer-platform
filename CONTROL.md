// @engx-governance-project: engx-governance
// @engx-governance-path: CONTROL.md
# CONTROL.md
# @version: 1.0.0
# @updated: 2026-03-25

Runtime behavior across all services. Derived from execution paths. No aspirational content.

---

## Nexus — runtime behavior

**Daemon loop** (`internal/daemon/`):
- Reconciler ticks every 5s: reads desired state → compares actual state → instructs runtime provider
- Crash detector: `fail_count ≥ threshold` → desired state → `maintenance`
- Recovery controller: `maintenance` projects polled every 30s for manual reset

**Event bus** (`internal/eventbus/bus.go`):
- All cross-component communication is in-process pub/sub
- Topics declared in `Canon/events` — never locally
- External consumers (Atlas, Forge) subscribe via HTTP polling `GET /events?since=<id>`

**Filesystem watcher** (`internal/watcher/`):
- Two modes: `WatchModeDropFolder` and `WatchModeWorkspace`
- Publishes `TopicWorkspaceFile*` and `TopicWorkspaceUpdated` to the bus
- Nexus is the only watcher. All other services subscribe via Nexus events.

**SSE broker** (`GET /events/stream`):
- Per-client goroutine with 5s send timeout
- Slow clients evicted without blocking the bus

---

## Atlas — runtime behavior

**Startup**: full workspace scan → index all `nexus.yaml` files → HTTP server starts.

**Polling loop** (`internal/nexus/subscriber.go`):
- `GET /events?since=<lastID>` every 3s
- On `workspace.project.detected`: re-index project
- On `workspace.file.*`: re-index affected source files

**Verification** (`internal/validator/nexus_yaml.go`):
- `status=verified` iff `nexus.yaml` parses cleanly AND `type` ∈ `Canon/descriptor.ValidTypes`
- Parse errors → `status=unverified`, no crash

**Graph build** (`internal/graph/builder.go`):
- Triggered after each indexing pass
- `depends_on` fields from `nexus.yaml` → directed edges

---

## Forge — runtime behavior

**Command lifecycle** (`internal/command/`):
1. `POST /commands` → `validator.Validate()` → `translator.Translate()` → `Command` struct
2. Dedup check: if same `command_id` within 300s TTL → `HTTP 409` + original result
3. Preflight: `Atlas GET /graph/services` → verified project check (fail-open)
4. `executor.Execute()` → intent handler (`run/build/deploy/test`)
5. Intent handler calls `Nexus POST /projects/:id/start|stop`
6. Result logged to `execution_history`

**Trigger loop** (`internal/trigger/scheduler.go`):
- Cron: ticks matching schedules, fires workflow
- Event: polls `GET /events?since=<id>` every 3s, matches against registered triggers
- Semaphore: max 8 concurrent workflow goroutines; dropped triggers logged at WARNING

**Dedup table** (`command_dedup`, migration v6):
- TTL: 300s
- Duplicate within TTL: `HTTP 409`, body contains original result
- No `command_id` supplied: UUID generated — never a duplicate

---

## Gate — runtime behavior

**Key lifecycle**:
- Ed25519 keypair loaded from `GATE_KEY_PATH` at startup; generated if absent
- Public key served at `GET /gate/public-key` — no auth required

**Token issuance** (`POST /gate/tokens/*`):
- Requires `X-Service-Token`
- Returns `{token, sub, exp}`; `sub` format: `<github_login>@github`, `agent:<id>`, `ci:<id>`

**Validation** (`POST /gate/validate`):
- No auth required — services must be able to validate without a service token
- Checks: signature, `iss == "gate"`, `exp`, revocation list
- Returns `{valid, claim}` or `{valid: false, reason}`

**GitHub OAuth**:
- `GET /gate/auth/github` → redirect to GitHub
- `GET /gate/auth/github/callback` → exchange code → issue developer token
- GitHub OAuth tokens and emails are never stored

---

## Observer services — common runtime behavior

Applies to: Metrics (:8083), Navigator (:8084), Guardian (:8085), Observer (:8086), Sentinel (:8087).

- Single polling goroutine owns all collection and store writes
- HTTP handlers are read-only — never touch collection state directly
- `sync.RWMutex` on all shared store structs: `Set()` write lock, `Get()` read lock
- Per-cycle trace ID: `<prefix>-<hex>` on all outbound calls
- One full collection pass completes before HTTP server starts (ADR-020 Rule 6)
- Graceful degradation: upstream unavailability → stale data served + WARNING logged

---

## Guardian — policy evaluation

Evaluation runs against a point-in-time `PolicyInput` snapshot assembled from all three collectors before `Evaluate()` is called. Findings are deterministic: same input → same output.

Active rules:

| Rule | Trigger condition | Severity |
|------|------------------|----------|
| G-001 | Same target denied 3+ times in 10 min | warning |
| G-002 | Forge executed against unverified Atlas project | warning |
| G-003 | >50% failure rate for target in 20 min | error |
| G-004 | `SERVICE_CRASHED` events in last 5 min | error |
| G-005 | Topology node `status=unverified` | warning |

Guardian never blocks execution. It is an audit layer.

---

## Sentinel — AI reasoning constraint

`GET /insights/explain` is the only endpoint that calls the Anthropic API.
Background collection never calls the AI. The Reasoner receives only the structured `SystemReport` — not raw events or graph data. Prompt explicitly prohibits start/stop recommendations.

Active Phase 1 rules: S-001 (cascade), S-002 (deploy correlation), S-003 (dependency risk), S-004 (stale project), S-005 (high denial rate).

---

## Relay — tunnel lifecycle

1. engxa opens TLS connection to `:9090` with JSON handshake `{token, owner, name}`
2. Relay validates token (Gate JWT if `RELAY_GATE_ADDR` set; HMAC fallback if not)
3. Relay assigns subdomain → responds `{ok, tunnel_id, subdomain, public_url}`
4. Incoming HTTPS to `*.<owner>.engx.dev` → Relay `:9091` → subdomain lookup → tunnel → engxa → local port
5. Reconnecting engxa for same subdomain closes previous connection

---

## Arbiter — enforcement gates

Two enforcement points:

| Gate | Trigger | What runs |
|------|---------|-----------|
| Packaging | `zp` before writing any ZIP | `VerifyPackaging(dir)` — static rules only |
| Execution | `engx run` Enforce step | `VerifyExecution(nexusAddr, token, dir)` — static + dynamic |

`arbiter verify ./...` runs all static rules standalone (no platform required).

Exit codes: `0` clean · `1` violations · `2` internal error.
