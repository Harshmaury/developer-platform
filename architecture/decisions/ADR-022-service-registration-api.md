# ADR-022 — Service Registration API

**Status:** Accepted
**Date:** 2026-03-19
**Author:** Harsh Maury
**Scope:** Nexus — new HTTP endpoint + engx CLI command
**Depends on:** ADR-001 (project registry), ADR-003 (HTTP/JSON protocol)

---

## Context

Service records in Nexus are the atomic unit of lifecycle management. The
reconciler reads desired state from the services table and engxa acts on it.
Without a service record, `engx project start` silently succeeds but nothing
happens — the project has no services to queue.

Currently the only way to create a service record is:
1. Direct SQLite write (requires stopping Nexus, bypasses all validation)
2. `POST /agents/:id/actual` (only updates existing records, never creates)

Every Nexus restart wipes the in-memory state from `go run` processes.
Services seeded via direct DB write persist across restarts, but there is
no supported API to create them. This was discovered during the first
full end-to-end platform run (2026-03-18) — every restart required a
manual seed script to restore service records.

This is the most critical operational gap in the platform.

---

## Decision

### 1. New endpoint: POST /services/register

Nexus adds a single service registration endpoint:

```
POST /services/register
```

Request body:
```json
{
  "id":       "atlas-daemon",
  "name":     "atlas-daemon",
  "project":  "atlas",
  "provider": "process",
  "config":   "{\"command\":\"go\",\"args\":[\"run\",\"./cmd/atlas/\"],\"dir\":\"/home/harsh/workspace/projects/apps/atlas\"}"
}
```

- `id` required — stable service identifier
- `name` required — human-readable name
- `project` required — must match a registered project ID
- `provider` required — one of: `process`, `docker`, `k8s`
- `config` required — provider-specific JSON config string

On success:
```json
{"ok": true, "data": {"id": "atlas-daemon", "name": "atlas-daemon"}}
```

### 2. Upsert semantics

`POST /services/register` uses upsert (INSERT OR REPLACE) semantics —
the same pattern as `POST /projects/register`. Re-registering a service
updates its config without resetting lifecycle state (desired_state,
actual_state, fail_count remain unchanged on conflict).

Initial registration sets:
- `desired_state = stopped`
- `actual_state  = stopped`
- `fail_count    = 0`

### 3. New engx CLI command: engx service register

```
engx service register <project-id> <service-id> \
  --provider process \
  --config '{"command":"go","args":["run","./cmd/atlas/"],"dir":"/path/to/atlas"}'
```

This wraps the HTTP endpoint. The `engx register <path>` command is
extended to also register the project's default service if the
`.nexus.yaml` contains a `runtime` section with `provider` and `command`.

### 4. Auto-registration from .nexus.yaml

When `engx register <path>` is called and `.nexus.yaml` contains:
```yaml
runtime:
  provider: process
  command: go
  args: [run, ./cmd/atlas/]
```

`engx register` automatically calls `POST /services/register` to create
the default service alongside the project. Service ID defaults to
`<project-id>-daemon`.

This eliminates the need for a separate seed step on every fresh install.

---

## What does NOT change

- `POST /projects/register` — unchanged, project and service remain separate
- `GET /services` — unchanged
- `POST /projects/:id/start|stop` — unchanged, still operates on services
- engxa sync protocol — unchanged, services still must exist before engxa manages them
- Service lifecycle state machine — unchanged

---

## Implementation scope — Nexus

### New file
```
internal/api/handler/services_register.go
    — Register() method on ServicesHandler
    — registerServiceRequest struct
```

### Modified files
```
internal/api/server.go
    — Add: mux.HandleFunc("POST /services/register", servicesH.Register)

cmd/engx/main.go
    — Extend readNexusManifest() to parse runtime.provider/command/args
    — Auto-call POST /services/register after project registration
    — Add: engx service register <project> <id> --provider --config
```

---

## Consequences

**Positive:**
- `engx register <path>` becomes a complete onboarding command — one call
  registers both project and service
- Services survive Nexus restarts — they persist in SQLite
- No more seed scripts or direct DB writes
- `engx project start` works immediately after `engx register`

**Negative:**
- `engx register` now makes two HTTP calls (project + service) — minor
- `.nexus.yaml` must have a `runtime` section for auto-registration
  (graceful: if absent, only project is registered with a clear log message)

---

## Compliance

| ADR | Status |
|-----|--------|
| ADR-001 | ✅ Nexus remains sole authority for project and service registry |
| ADR-003 | ✅ HTTP/JSON on 127.0.0.1 only |
| ADR-008 | ✅ X-Service-Token required (same as all other non-health routes) |

---

## Next ADR

ADR-023 — candidate: platform launcher (`engxd start-all`) that starts
all platform daemons in dependency order from a single command.
