# ADR-048 — Arbiter HTTP Service

**Status:** Accepted
**Date:** 2026-03-22
**Author:** Harsh Maury
**Scope:** Arbiter — HTTP API for rule verification
**Canonical location:** `services/arbiter/architecture/decisions/ADR-048-arbiter-http-service.md`
**Depends on:** ADR-047 (Arbiter engine)

---

## Context

ADR-047 defined the Arbiter engine as a Go library (`arbiter.VerifyExecution()`).
The engx CLI calls this directly. But CI pipelines, the web dashboard, and other
tools need rule verification over HTTP without embedding the Go library.

---

## Decision

Expose Arbiter as an HTTP service on port 9093.

### Endpoints

```
POST /verify          — verify a project directory against all rules
GET  /rules           — list all registered rules with metadata
GET  /rules/:id       — single rule detail
GET  /health          — standard health check
```

### Rules

1. Arbiter HTTP service runs alongside the Go library API — they share the same engine.
2. Port 9093 — registered in Nexus via `nexus.yaml`.
3. Auth: `X-Service-Token` required (ADR-008). `/health` exempt.
4. `POST /verify` body: `{"path": "<absolute-project-dir>"}`.
5. Response includes `passed []RuleResult`, `violations []RuleResult`, `ok bool`.
6. Nexus emits `SystemAlertPayload` with `rule: "skip-enforce"` when `--skip-enforce` is used.

---

## Compliance

| ADR | Status |
|-----|--------|
| ADR-003 | ✅ HTTP/JSON on 127.0.0.1 |
| ADR-008 | ✅ X-Service-Token required |
| ADR-047 | ✅ HTTP service wraps the same engine — no logic duplication |

---

## Next ADR

ADR-049 — Platform hardening session
