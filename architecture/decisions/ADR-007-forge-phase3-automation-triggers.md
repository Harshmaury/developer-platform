# ADR-007 — Forge Phase 3 Automation Trigger Model

Date: 2026-03-15
Status: Accepted
Updated: 2026-03-16 — concurrency bound, pkg/events import path,
                       type-safe topic lookup, and since tracking documented

---

## Context

Forge Phase 2 introduced named workflow definitions stored in SQLite.
Phase 3 makes those workflows executable automatically in response to
platform events — specifically workspace change events published by
Nexus (ADR-002).

Before Phase 3 could be implemented, three questions needed answers:

1. What is the data model for a trigger — what maps an event to a workflow?
2. How does Forge receive workspace events — direct bus subscription
   or polling the Nexus HTTP API?
3. What filtering is available — should any workspace file change fire
   all triggers, or can triggers be scoped to specific extensions,
   directories, or projects?

## Decision

### Trigger Model

A trigger is a persistent record that maps one workspace event topic
to one stored workflow, with an optional filter.

```json
{
  "id":          "<uuid>",
  "event":       "workspace.file.modified",
  "workflow_id": "<uuid>",
  "filter": {
    "extension": ".go",
    "project":   "nexus"
  },
  "enabled": true
}
```

Fields:
- `event` — one of the five workspace topics declared in ADR-002
- `workflow_id` — must reference an existing workflow (Phase 2)
- `filter` — optional; all filter fields are AND-combined
- `enabled` — triggers can be disabled without deletion

### Event Subscription

Forge subscribes to workspace events using the same HTTP polling
pattern as Atlas: `GET /events?limit=50&since=<last_id>` every 3s.

The `since` parameter carries the highest event ID received so far.
The subscriber tracks this value across poll cycles so each event
is processed exactly once. Omitting `since` returns only the most
recent events — Forge always supplies it after the first poll.

Topic constants are imported from:

    github.com/Harshmaury/Nexus/pkg/events

This is the public re-export path for external consumers. Forge never
imports from `github.com/Harshmaury/Nexus/internal/eventbus` — that
package is Nexus-internal only (see ADR-002).

Topic lookup in the `SupportedEvents` map uses `nexusevents.Topic`
as the key type — the same named type as the constants. Event type
strings from the HTTP response are cast to `nexusevents.Topic` before
the map lookup to ensure type safety.

### Filter Semantics

An event matches a trigger if all supplied filter fields match:
- `extension` — file extension including dot (e.g. `.go`, `.md`)
- `project`   — Atlas project ID the file belongs to
- `directory` — absolute path prefix of the file

An empty filter matches all events of the given topic.

### Execution

When a workspace event matches a trigger, Forge executes the linked
workflow using the existing Phase 2 workflow executor. The base
`CommandContext` is populated with `workspace_root` from the event
payload and `requesting_agent` set to `"trigger"`.

Execution is asynchronous — the subscriber never blocks waiting for
a workflow to complete.

Concurrency is bounded by a buffered channel semaphore:

    maxConcurrentWorkflows = 8

Before spawning a goroutine, the subscriber performs a non-blocking
send on the semaphore channel. If all 8 slots are occupied, the trigger
is dropped and a WARNING is logged. Triggers are best-effort automation
— they are not guaranteed delivery under load.

The goroutine releases its semaphore slot via `defer` on completion.

## Supported Event Topics

Forge Phase 3 supports triggering on any of the five workspace topics
declared in Nexus `internal/eventbus/bus.go` and re-exported via
`pkg/events`:

    workspace.file.created
    workspace.file.modified
    workspace.file.deleted
    workspace.updated
    workspace.project.detected

## What Forge Must Never Do

- Subscribe to the Nexus internal event bus directly
- Import from `github.com/Harshmaury/Nexus/internal/eventbus`
- Redefine workspace topic strings locally
- Block the event polling loop waiting for workflow execution
- Spawn goroutines without acquiring a semaphore slot
- Execute triggers for disabled trigger records

## Alternatives Considered

**WebSocket or SSE push from Nexus** — rejected for Phase 3 because
the polling pattern is already proven in Atlas and consistent with
ADR-003. Push-based delivery can replace polling in a future phase
without changing the trigger model.

**One trigger can map to multiple workflows** — rejected to keep the
data model simple. Multiple triggers pointing to the same event achieve
the same result and are easier to manage individually.

**Unbounded goroutine dispatch** — rejected. A burst of file events
(e.g. `git checkout`) can produce 50+ matched triggers simultaneously.
Unbounded goroutine spawning would cause uncontrolled concurrent build
and test processes. The semaphore cap of 8 matches the typical number
of logical CPU cores on a developer machine.

**Trigger conditions beyond extension/project/directory** — deferred.
File content matching and regex patterns are possible future additions
but add complexity before the basic trigger model is proven.

## Consequences

Triggers are stored in the Forge SQLite database alongside workflows.
The Phase 3 subscriber runs as a background goroutine alongside the
existing HTTP server. Adding a trigger via the API takes effect
immediately — the subscriber reads the trigger registry on each
poll cycle.

The automation layer builds cleanly on top of Phase 2 without
modifying any existing Phase 1 or Phase 2 code.
