# ADR-034 — Herald: Typed Nexus HTTP Client

**Status:** Accepted
**Date:** 2026-03-21
**Author:** Harsh Maury
**Scope:** All observer services — inter-service HTTP calls
**Canonical location:** `services/herald/architecture/decisions/ADR-034-herald-typed-http-client.md`

---

## Context

Five observer services each maintained their own `internal/collector/nexus.go`
with identical logic: raw `http.Get` calls, manual Canon header injection, no retry,
inline JSON struct definitions that drifted from the actual Nexus API schema.

---

## Decision

Introduce **Herald** (`github.com/Harshmaury/Herald`) as the typed HTTP client
for all inter-service calls.

### Rules

1. No service uses raw `http.Get` or `http.NewRequest` to call another platform service.
   All calls go through Herald.
2. Herald injects `X-Service-Token` and `X-Trace-ID` on every request.
3. Herald uses Accord types for all request/response shapes — never anonymous structs.
4. Herald is fail-open: connection errors return nil + log WARNING, never panic.

---

## Compliance

| ADR | Status |
|-----|--------|
| ADR-008 | ✅ Herald passes X-Service-Token via WithToken() option |
| ADR-033 | ✅ Herald uses Accord types as wire contract |
| ADR-039 | ✅ This ADR is the foundation for the Herald migration |

---

## Next ADR

ADR-035 — engx deregister
