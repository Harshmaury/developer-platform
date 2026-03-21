# ADR-037 — Signal System: Event Schema Enhancement

**Status:** Accepted  
**Date:** 2026-03-21  
**Author:** Harsh Maury  
**Scope:** Nexus (state), Canon (events) — all services consume  
**Depends on:** ADR-014 (trace propagation), ADR-016 (Canon)

---

## Context

The Platform Backbone Proposal defines a Signal System where all system
activity is represented as structured events. The current Event schema is
missing three fields required for full observability:

| Missing field | Why needed |
|---|---|
| `level` | Severity routing — consumers filter info/warn/error without parsing payload |
| `span_id` | Causality — identifies this specific operation within a trace |
| `parent_span_id` | Causality tree — links this operation to its cause |

Without `span_id` + `parent_span_id`, Observer can only build a flat timeline.
It cannot answer: "which Atlas query triggered which Forge execution?"

---

## Decision

### 1. DB migration v6 — add three columns to events table

```sql
ALTER TABLE events ADD COLUMN level TEXT NOT NULL DEFAULT 'info'
ALTER TABLE events ADD COLUMN span_id TEXT NOT NULL DEFAULT ''
ALTER TABLE events ADD COLUMN parent_span_id TEXT NOT NULL DEFAULT ''
```

Defaults preserve backwards compatibility — existing rows get `level='info'`,
empty span IDs.

### 2. Updated Event struct

```go
type Event struct {
    // ... existing fields ...
    SpanID       string `json:"span_id"`        // this operation's span
    ParentSpanID string `json:"parent_span_id"` // causing span (empty = trace root)
    Level        string `json:"level"`           // "info" | "warn" | "error"
}
```

### 3. Level constants in Canon

```go
// canon/events/events.go additions
const (
    LevelInfo  = "info"
    LevelWarn  = "warn"
    LevelError = "error"
)
```

### 4. Span ID generation

Span IDs are 8-char hex generated at the call site (same as TraceID).
`ParentSpanID` is passed in context via `X-Span-ID` header alongside `X-Trace-ID`.

### 5. AppendEvent signature update

Current:
```go
AppendEvent(serviceID, eventType, source, traceID, component, outcome, payload string) error
```

New:
```go
AppendEvent(serviceID, eventType, source, traceID, spanID, parentSpanID, component, level, outcome, payload string) error
```

All existing call sites pass `""` for spanID/parentSpanID and `"info"` for
level — fully backwards compatible at the call site.

---

## Migration

All 12 call sites to `AppendEvent` in Nexus are updated to pass the new
parameters. Existing callers outside Nexus (none currently) are unaffected
because AppendEvent is only called internally.

---

## Compliance

| ADR | Status |
|-----|--------|
| ADR-014 | ✅ Extends existing trace propagation — same X-Trace-ID, adds X-Span-ID |
| ADR-016 | ✅ Level constants added to Canon — never hardcoded locally |
