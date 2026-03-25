# ADR-046 — Conduit: Remote Execution Routing

**Status:** Accepted
**Date:** 2026-03-22
**Author:** Harsh Maury
**Scope:** Conduit service — remote Forge execution (v3 Phase 3)
**Depends on:** ADR-042 (Gate identity), ADR-041 (Relay tunnel)

---

## Context

v3 Phase 3 enables running Forge workflows on any machine from a single CLI.
A developer at machine A issues `engx run atlas` — Conduit routes the execution
to machine B where atlas is registered, executes it through Forge on B, and
streams the result back to the CLI on A.

Without a routing layer, remote execution would require the CLI to know machine
addresses directly — breaking the single-socket model and requiring manual
network configuration.

---

## Decision

Introduce **Conduit** (`github.com/Harshmaury/Conduit`) as the remote execution
router. Conduit sits between the CLI and remote Forge instances.

### Rules

1. Conduit validates `X-Identity-Token` (Gate JWT) on every inbound request.
   Anonymous requests are rejected — remote execution always has an actor.
2. Conduit routes by `project_id` — it queries the remote Nexus registry to find
   which machine hosts a given project.
3. Conduit uses Gate JWT validation (local + network) — remote execution requires
   revocation certainty.
4. Conduit never modifies execution results — it routes bytes, not semantics.
5. Sessions are evicted after inactivity (configurable TTL, default 30m).
6. Conduit port: 9092.

---

## Compliance

| ADR | Status |
|-----|--------|
| ADR-042 | ✅ Gate JWT required — no anonymous remote execution |
| ADR-041 | ✅ Uses Relay tunnel for the return stream to the CLI |
| ADR-020 | ✅ Conduit does not write to Nexus state — routing only |

---

## Next ADR

ADR-047 — Arbiter architectural enforcement engine
