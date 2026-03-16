# ADR-002 — Workspace Observation Ownership

Date: 2026-03-15
Status: Accepted
Updated: 2026-03-16 — consumer import path corrected; pkg/events re-export documented

---

## Context

Multiple platform components require awareness of filesystem changes
within the workspace. Atlas needs to update its index when files are
created or modified. Forge may trigger workflows on workspace events.
Diagnostic systems may monitor project activity.

Running independent watchers in each service duplicates infrastructure,
increases system load, and creates race conditions when multiple watchers
react to the same event simultaneously.

Nexus already runs a filesystem watcher (`internal/watcher/watcher.go`)
as part of the Drop Intelligence pipeline.

## Decision

Nexus owns filesystem observation for the entire platform.

Nexus extends its existing watcher to publish workspace change events
through the platform event bus alongside existing service events.

## Workspace Event Topics

Workspace event topic constants are declared in:

    internal/eventbus/bus.go

and re-exported for use by external services (Atlas, Forge) in:

    pkg/events/topics.go

Both files are in the Nexus repository. The re-export exists because
Go's module system prohibits external modules from importing `internal/`
packages. `internal/eventbus` is for Nexus-internal use only.

Topics:

    TopicWorkspaceFileCreated      "workspace.file.created"
    TopicWorkspaceFileModified     "workspace.file.modified"
    TopicWorkspaceFileDeleted      "workspace.file.deleted"
    TopicWorkspaceUpdated          "workspace.updated"
    TopicWorkspaceProjectDetected  "workspace.project.detected"

`TopicWorkspaceUpdated` is a debounced batch signal — Nexus publishes it
once after a burst of file events settles, not once per file. Consumers
that need to rebuild large data structures (graph edges, capability index)
should act on this topic rather than on individual file events.

## Consumer Rule

All consumers (Atlas, Forge, and any future services) must:

- Import topic constants from `github.com/Harshmaury/Nexus/pkg/events`
- Never import from `github.com/Harshmaury/Nexus/internal/eventbus`
- Never redefine topic strings locally in their own packages

Correct:

    import nexusevents "github.com/Harshmaury/Nexus/pkg/events"
    // use nexusevents.TopicWorkspaceFileCreated

Wrong:

    const myTopic = "workspace.file.created"   // silent decoupling

## Event Subscription Pattern

Atlas and Forge subscribe to workspace events by polling the Nexus HTTP
API: `GET /events?limit=50&since=<last_id>` every 3 seconds.

`since` is the highest event ID received so far. Consumers track this
value across poll cycles to avoid reprocessing events. If `since` is
omitted, Nexus returns the most recent events only.

Neither Atlas nor Forge subscribes to the internal Nexus event bus
directly. Neither runs a filesystem watcher of any kind.

## Implications

- Atlas subscribes to workspace topics to trigger index updates.
- Forge subscribes to workspace topics to trigger event-driven automation
  (Phase 3 of Forge evolution, ADR-007).
- No other component runs a filesystem watcher.
- The Nexus watcher configuration determines which directories are observed.

## Alternatives Considered

**Shared infrastructure watcher component** — rejected because it requires
a new binary and introduces a new failure point for a capability Nexus
already provides.

**Each service runs its own watcher** — rejected because it duplicates
kernel-level inotify resources and creates race conditions between
concurrent handlers for the same filesystem event.

## Consequences

Nexus becomes responsible for the reliability of workspace event delivery.
Atlas and Forge are event consumers only — they never observe the filesystem
directly.
