# ADR-040 — Outcome-Centric UX: Progressive Disclosure

**Status:** Accepted
**Date:** 2026-03-21
**Author:** Harsh Maury
**Scope:** engx CLI — user-facing commands
**Depends on:** ADR-038 (system/validate), ADR-036 (system/graph)

---

## Context

The current engx CLI is system-centric. Commands expose internal structure
(sentinel, guardian, events, workflows) before users need it. Error messages
report what failed technically but not what the user should do next.

The UX Proposal defines a shift:

> A user cares about: "Did my project run? If not, why? What should I do next?"

Three specific problems:

1. **Cognitive overload** — `engx help` shows 24 commands including
   `sentinel`, `guard`, `workflow`, `trigger`, `on`. New users see system
   internals before they can run anything.

2. **Non-actionable errors** — `daemon not running`, `HTTP 400`,
   `json unmarshal error` tell what failed, not what to do.

3. **Fragmented failure diagnosis** — logs, doctor, sentinel, guardian
   must all be checked manually. No single command explains a failure.

---

## Decision

### 1. engx run — validate + start + monitor + outcome in one command

```
engx run <project>
```

```
Project: atlas

  Validating... ✓
  Starting...   ✓
  Waiting...    ···✓

  Status:     RUNNING
  Services:   1/1 running
```

On failure:

```
Project: atlas

  Validating... ✓
  Starting...   ✓
  Waiting...    · ✗

  Status:     FAILED

  What:       1/2 services started
  Where:      atlas-daemon (maintenance)
  Why:        service exceeded restart threshold — automatically paused
  Next step:  engx services reset atlas-daemon
```

### 2. engx ps [project] — project-level status

```
engx ps           # all projects
engx ps atlas     # one project with cause + next step
```

Replaces `engx project status --all` with a user-friendly view.
Does not show service IDs — shows project outcomes.

### 3. Structured UserError type

Every user-facing error uses `UserError` with four fields:
- **What** — what happened
- **Where** — which component or step
- **Why** — plain language root cause
- **Next step** — exactly what to type

### 4. Progressive disclosure — hide advanced commands

Default `engx help` shows only user-facing commands.
Advanced commands (`sentinel`, `events`, `workflow`, `trigger`, etc.)
are hidden but fully functional.

Users discover advanced mode when they need it:
```
engx doctor        # when ps shows issues
engx sentinel      # when doctor shows incidents
engx events        # when sentinel needs context
```

---

## What Does NOT Change

- All existing commands remain — hidden ≠ removed
- `engx doctor`, `engx logs`, `engx trace` remain visible (debugging tools)
- `engx platform start/stop` remains visible (operator commands)
- No changes to daemon, API, or any service

---

## Mental Model Shift

**Before:** User must understand nexus → project → service → daemon → state machine

**After:**
```
engx run <project>     — does it work?
engx ps <project>      — what's the status?
engx logs <service>    — what happened?
engx doctor            — what's wrong with the platform?
```

Four commands cover 90% of user interactions.

---

## Compliance

| ADR | Status |
|-----|--------|
| ADR-038 | ✅ engx run calls POST /system/validate before starting |
| ADR-003 | ✅ No new inter-service calls — reads from Nexus only |
