# ADR-033 — Accord: Shared API Types Module

**Status:** Accepted
**Date:** 2026-03-21
**Author:** Harsh Maury
**Scope:** All services — cross-service request/response types
**Canonical location:** `services/accord/architecture/decisions/ADR-033-accord-shared-api-types.md`

---

## Context

Services were defining anonymous structs for cross-service API shapes inline in
handler and collector files. When the Nexus API schema changed, all consumers
updated independently — drift was invisible at compile time.

---

## Decision

Introduce **Accord** (`github.com/Harshmaury/Accord`) as the sole source of all
cross-service request/response types (DTOs).

### Rules

1. No service defines its own struct for a cross-service API shape. Import from Accord.
2. All Nexus handler response types, Forge command types, and Atlas project types
   live in `accord/api/`.
3. `accord.DecodePayload[T]` is the only permitted decoder for `EventDTO.Payload`.
4. Accord version bumps (minor) when new types are added; patch for bug fixes.

---

## Compliance

| ADR | Status |
|-----|--------|
| ADR-016 | ✅ Accord complements Canon — Canon owns constants, Accord owns types |
| ADR-039 | ✅ Herald collectors use Accord types internally |

---

## Next ADR

ADR-034 — Herald typed HTTP client
