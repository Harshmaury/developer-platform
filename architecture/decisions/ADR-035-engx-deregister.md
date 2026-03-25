# ADR-035 — engx deregister: Remove Ghost Projects

**Status:** Accepted
**Date:** 2026-03-21
**Author:** Harsh Maury
**Scope:** Nexus — project and service deregistration

---

## Context

Projects registered with Nexus but no longer on disk accumulated as ghost entries.
`engx platform start` would attempt to start them, fail, and pollute `engx status`.
There was no command to cleanly remove a project and all its associated services.

---

## Decision

Add `engx deregister <project-id>` command and `POST /projects/:id/deregister`
endpoint to Nexus.

### Rules

1. Deregistration removes the project record and all service records associated with it.
2. Running services are stopped before deregistration — never orphan a live process.
3. `engx deregister` is idempotent — deregistering an already-absent project returns success.
4. Deregistration is logged as a `StateChanged` event with `to: "deregistered"`.
5. The daemon socket handles `CmdProjectDeregister` — the CLI never calls HTTP directly.

---

## Compliance

| ADR | Status |
|-----|--------|
| ADR-001 | ✅ Nexus remains the canonical registry — deregister is a write operation on Nexus only |
| ADR-005 | ✅ Services are stopped before project is removed |
| ADR-003 | ✅ HTTP/JSON — new POST endpoint follows existing envelope |

---

## Next ADR

ADR-036 — GET /system/graph unified topology endpoint
