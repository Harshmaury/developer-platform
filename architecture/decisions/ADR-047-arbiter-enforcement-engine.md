# ADR-047 — Arbiter: Architectural Enforcement Engine

**Status:** Accepted
**Date:** 2026-03-22
**Author:** Harsh Maury
**Scope:** Arbiter — static architectural rule enforcement
**Canonical location:** `services/arbiter/architecture/decisions/ADR-047-arbiter-enforcement-engine.md`

---

## Context

Architectural rules (Canon constants, observer read-only, Forge file access via Atlas)
were documented in ADRs but not enforced in code. Violations accumulated silently
and were only caught during code review.

---

## Decision

Introduce **Arbiter** (`github.com/Harshmaury/Arbiter`) as the architectural
enforcement engine. Arbiter verifies rules against source code before execution.

### Rule Categories

| Category | Prefix | Examples |
|----------|--------|---------|
| Authority | A | Observers never call write endpoints |
| Contract | C | Canon constants used, no string literals |
| Spatial | S | No service imports another's `internal/` |
| Temporal | T | Startup order, migration sequence |

### Rules

1. `engx run <project>` calls `arbiter.VerifyExecution()` as a `KindEnforce` plan step.
2. `--skip-enforce` bypasses the gate and emits a `SYSTEM_ALERT` to Nexus (audited).
3. Arbiter is fail-open — if unreachable, the step skips with a warning. Never blocks execution.
4. Rule violations return `HTTP 422` with structured violation list.
5. `arbiter verify <path>` is available as a standalone CLI command.

---

## Compliance

| ADR | Status |
|-----|--------|
| ADR-043 | ✅ Arbiter gate inserted as KindEnforce step in buildRunPlan |
| ADR-016 | ✅ Arbiter enforces Canon constant usage (rule A-C-001, A-C-002) |
| ADR-020 | ✅ Arbiter enforces observer read-only rule (A-A-001) |

---

## Next ADR

ADR-048 — Arbiter HTTP service
