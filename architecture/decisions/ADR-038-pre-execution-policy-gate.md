# ADR-038 — POST /system/validate: Pre-Execution Policy Gate

**Status:** Accepted  
**Date:** 2026-03-21  
**Author:** Harsh Maury  
**Scope:** Nexus — new POST /system/validate endpoint  
**Depends on:** ADR-036 (system routes), ADR-023 (startup grace)

---

## Context

The Platform Backbone Proposal requires governance to operate at two points:

1. **Pre-execution** — block unsafe operations before they happen
2. **Runtime** — react to violations as they occur (Guardian already does this)

Currently nothing blocks `engx project start atlas` if atlas has unresolved
policy violations. Guardian warns, but does not deny. The platform has no
enforcement point before a project is queued.

---

## Decision

Add `POST /system/validate` to Nexus. This is the pre-execution policy gate.

### Request

```json
{
  "project_id": "atlas",
  "intent": "start"
}
```

### Response

```json
{
  "ok": true,
  "data": {
    "project_id": "atlas",
    "allowed": true,
    "violations": []
  }
}
```

Or on deny:

```json
{
  "ok": true,
  "data": {
    "project_id": "forge",
    "allowed": false,
    "violations": [
      {
        "rule_id": "V-001",
        "message": "project 'forge' has no services — run: engx register <path>",
        "action": "deny"
      }
    ]
  }
}
```

### Validation Rules

| Rule | Scope | Action | Condition |
|---|---|---|---|
| V-001 | start | deny | project has zero registered services |
| V-002 | start | warn | any service in maintenance state |
| V-003 | start | warn | any service with fail_count ≥ 5 |

Rules are evaluated by Nexus using local DB state — no call to Guardian.
Guardian rules (G-001 through G-008) operate at the runtime layer. V-rules
operate at the structural layer.

### Integration with engx platform start

`platformStartCmd` calls `POST /system/validate` for each project before
queuing. On `allowed=false`, the project is skipped with a clear error.
On warnings, the output is shown but the project is still started.

```
Starting platform services...
  ✗ forge: validation denied — no services registered
  ✓ atlas: started (1 service)
  ! guardian: warning — atlas-daemon has 2 failures
  ✓ sentinel: started (1 service)
```

### Rules for the validate endpoint

1. `POST /system/validate` is **fast** — only reads from local DB, no outbound calls.
2. `allowed=false` MUST block the operation in `platform start` and `engx run`.
3. Warnings MUST be shown but MUST NOT block.
4. The endpoint is idempotent — calling it multiple times has no side effects.
5. Future rules (V-004+) require an ADR and a version bump.

---

## Compliance

| ADR | Status |
|-----|--------|
| ADR-003 | ✅ HTTP/JSON on 127.0.0.1 — no new transport |
| ADR-023 | ✅ validate runs after reset, before queue — ADR-023 reset is unchanged |
| ADR-032 | ✅ --register flag runs before validate in platform start |
