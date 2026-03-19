# ADR-027 — Forge Scheduled Cron Triggers (Phase 5)

**Status:** Accepted
**Date:** 2026-03-19
**Author:** Harsh Maury
**Scope:** forge

---

## Context

Forge already supports event-driven triggers (ADR-007): a Nexus event fires a
workflow. However, some workflows need to run on a schedule — nightly builds,
periodic health checks, timed deployments. These are not driven by events but
by wall-clock time.

Adding cron support to Forge's existing trigger mechanism is the natural
extension. It reuses the same workflow execution path without adding a new
capability domain.

---

## Decision

Add a `CronScheduler` to Forge that fires registered triggers on schedule.
Cron triggers share the existing execution infrastructure — they produce the
same `Command` object and run through the same executor path as event triggers.

### Supported schedule formats

```
@every <duration>    — e.g. @every 30m, @every 2h
@hourly              — shorthand for @every 1h
@daily               — shorthand for @every 24h
```

**Minimum interval: 1 minute.** Shorter durations are rejected at registration
time with a clear error. This prevents accidental load from misconfigured triggers.

### Implementation

```
internal/trigger/scheduler.go    — CronScheduler struct, tick loop, store poll
```

`CronScheduler` is wired in `cmd/forge/main.go` alongside the existing
`Subscriber`. Both share the **same execution semaphore** — a cron trigger
and an event trigger can never execute the same workflow concurrently.

SQLite migration v5 adds a `schedule TEXT` column to the `triggers` table.
Triggers with a non-empty `schedule` field are cron triggers. Triggers with
an empty `schedule` remain event-driven (backward compatible).

The scheduler re-polls the store every 60 seconds — new cron triggers are
picked up without a restart.

### Command context for cron-fired executions

```go
Command.context["requesting_agent"] = "scheduler"
```

This distinguishes cron-fired executions in the history log from CLI or
event-triggered runs.

### Rules

1. `CronScheduler` and `Subscriber` share the existing Forge execution
   semaphore. New semaphores are never created.
2. `@every` durations shorter than 1 minute are rejected at registration —
   not at fire time.
3. No new Go module dependencies — tick loop uses `time.NewTicker` from stdlib.
4. Migration v5 is append-only — existing event triggers are unaffected.
5. `CronScheduler` gracefully shuts down on context cancellation — no goroutine
   leaks on `engx platform stop` or daemon restart.

---

## Compliance

| ADR     | Status |
|---------|--------|
| ADR-004 | ✅ Cron-fired commands become Command objects before execution |
| ADR-007 | ✅ Cron triggers coexist with event triggers — same execution path |
| ADR-020 | ✅ No observer involvement — Forge owns its own scheduler |

---

## Next ADR

ADR-028 — `engx upgrade` self-upgrade protocol
