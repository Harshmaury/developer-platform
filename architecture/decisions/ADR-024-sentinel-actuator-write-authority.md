# ADR-024 — Sentinel Actuator: Write Authority for Observer

**Status:** Accepted
**Date:** 2026-03-19
**Author:** Harsh Maury
**Scope:** sentinel, nexus

---

## Context

All observer services are strictly read-only (ADR-020). However, the fail-safe
model requires an automated recovery path for services stuck in maintenance
state — a condition where the operator is not present and no other platform
component has the authority to act.

Sentinel is the only observer with the full picture: it correlates health
signals, tracks recovery history, and can reason about whether a restart
attempt is appropriate. Granting it bounded write authority completes the
autonomous recovery loop without violating the spirit of ADR-020.

---

## Decision

Sentinel gains a new internal component — the **Actuator** — with strictly
bounded write authority. All other observers remain read-only.

### Allowed actions

```
POST /services/:id/reset    — clear maintenance state
POST /projects/:id/start    — request service restart
```

Both calls target Nexus only. No other service is writable by the Actuator.

### Three safety constraints (all must pass before any action)

| Constraint | Rule |
|------------|------|
| **Cooldown** | 5 minutes between reset attempts per service |
| **Max attempts** | 3 per service per hour — escalate if exceeded |
| **Verify recovery** | Check service state on next collection cycle before counting success |

### Escalation path

When max attempts is reached for a service, the Actuator stops acting and
raises a G-006 finding (Guardian) and an S-006 insight (Sentinel). The operator
is notified and must intervene manually.

### Rules

1. The Actuator is the **only** observer component with write authority.
   All other Sentinel components (engine, AI reasoner, collectors) remain read-only.
2. Actuator actions are logged to `~/.nexus/recovery.log` as append-only JSON
   lines (phase 3 — ADR-031 by extension via recovery log).
3. The Actuator never acts on an insight that recommends a start/stop for
   reasons other than maintenance-state recovery.
4. The AI system prompt explicitly prohibits start/stop recommendations —
   AI reasoning is advisory only.

---

## Compliance

| ADR     | Status |
|---------|--------|
| ADR-017 | ✅ Sentinel phase 2+ — Actuator is a phase 3 addition |
| ADR-020 | ⚠️ Modified — Sentinel Actuator is the sole exception to the read-only rule |
| ADR-023 | ✅ Uses the reset endpoint defined in ADR-023 |

---

## Next ADR

ADR-025 — `engx` automation commands (Phase 17)
