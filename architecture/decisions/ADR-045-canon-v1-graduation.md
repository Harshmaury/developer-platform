# ADR-045 — Canon v1.0.0: Graduation, Relay Constants, Payload Migration

Date: 2026-03-22
Status: Accepted
Domain: Canon (library) — platform-wide contract
Depends on: ADR-016 (Canon origin), ADR-041 (Relay — requires relay constants), ADR-002 (workspace topics)
Unblocks: Relay repo creation, ADR-041 Phase 1 implementation

---

## Context

Canon is the single source of shared constants across the platform
(ADR-016). It currently holds:
- Header constants (TraceIDHeader, ServiceTokenHeader, IdentityTokenHeader)
- Service name and default address constants
- Event type and component constants
- Workspace topic constants (TopicWorkspaceFileCreated etc.)
- Gate scope constants
- Descriptor types

Canon is at `v0.4.1`. The `0.x` versioning signals "contract still
forming." Per the versioning standard (versioning.md v2.0, Rule 5),
`v1.0.0` is earned when all planned fields for the current capability
set are present and no breaking changes are planned next session.

Three items block Canon graduation:

**Item 1 — Relay constants are missing.**
ADR-041 defines `RelayTokenHeader`, `SubdomainHeader`, `OwnerHeader`,
`DefaultRelayTunnelAddr`, and `ServiceRelay`. These must exist in Canon
before the Relay repo is created. Without them, Relay would either
define its own header literals (Canon violation) or wait indefinitely.

**Item 2 — Workspace payload types still live in `nexus/pkg/events`.**
The three payload types (`WorkspaceFilePayload`, `WorkspaceUpdatedPayload`,
`WorkspaceProjectPayload`) are defined in `github.com/Harshmaury/Nexus/pkg/events`.
Atlas and Forge import this package to decode workspace event payloads.
This creates a cross-module dependency on Nexus for types that have
nothing to do with Nexus's runtime behavior. It also means any service
that needs workspace payloads must take a dependency on the entire
Nexus module.

**Item 3 — `nexus/pkg/events` is a leaking abstraction.**
The package exists because Go's module system prevents external modules
from importing `internal/`. It was a bridge — workspace topics would
live in Canon, but payloads had nowhere to go yet. That bridge is now
the wrong place. Canon is the right place.

---

## Decision

Graduate Canon to `v1.0.0` with three additions:

### 1. Relay constants (identity/identity.go)

```go
// Relay tunnel service name and default address.
ServiceRelay           = "relay"
DefaultRelayTunnelAddr = "relay.engx.dev:9090"

// Relay HTTP header constants (ADR-041).
RelayTokenHeader = "X-Relay-Token"   // authenticates engxa tunnel connection
SubdomainHeader  = "X-Engx-Subdomain" // subdomain assigned to this tunnel
OwnerHeader      = "X-Engx-Owner"    // owner identifier (e.g. "harsh")
```

### 2. Workspace payload types (events/payloads.go — new file)

Migrate from `nexus/pkg/events` to a new file `Canon/events/payloads.go`:

```go
// WorkspaceFilePayload is the payload for file created/modified/deleted events.
type WorkspaceFilePayload struct {
    Path      string    `json:"path"`
    Name      string    `json:"name"`
    Extension string    `json:"extension"`
    SizeBytes int64     `json:"size_bytes"`
    EventAt   time.Time `json:"event_at"`
}

// WorkspaceUpdatedPayload is the payload for workspace.updated batch events.
type WorkspaceUpdatedPayload struct {
    WatchDir string    `json:"watch_dir"`
    EventAt  time.Time `json:"event_at"`
}

// WorkspaceProjectPayload is the payload for workspace.project.detected.
type WorkspaceProjectPayload struct {
    Path       string    `json:"path"`
    Name       string    `json:"name"`
    DetectedBy string    `json:"detected_by"`
    DetectedAt time.Time `json:"detected_at"`
}
```

### 3. nexus/pkg/events consumer migration

Atlas and Forge switch from:
```go
import nexusevents "github.com/Harshmaury/Nexus/pkg/events"
```
to:
```go
import canonevents "github.com/Harshmaury/Canon/events"
```

`nexus/pkg/events/topics.go` is retained but marked deprecated with
a comment pointing to Canon. It is not deleted in this session —
deletion happens after all consumers have migrated and the next
Nexus minor is tagged.

---

## Graduation criteria satisfied

Per versioning.md v2.0 Rule 5, all three criteria are met:

| Criterion | Status |
|---|---|
| All planned fields and types present | ✅ — relay constants + payload types complete the set |
| No breaking changes planned next session | ✅ — Relay adds constants, never removes |
| At least two services depend on it in production | ✅ — all 7 platform services import Canon |

Canon `v1.0.0` signals: **this contract is stable. Consumers can depend on it.**

After graduation, Canon increments conservatively:
- New constant or type → MINOR (`v1.1.0`)
- Bug in existing type → PATCH (`v1.0.1`)
- Remove or rename anything → MAJOR (`v2.0.0`)

---

## Compliance

A Canon implementation satisfies this ADR when:

1. `identity/identity.go` contains `ServiceRelay`, `DefaultRelayTunnelAddr`,
   `RelayTokenHeader`, `SubdomainHeader`, `OwnerHeader`.

2. `events/payloads.go` exists and contains `WorkspaceFilePayload`,
   `WorkspaceUpdatedPayload`, `WorkspaceProjectPayload`.

3. The Canon module is tagged `v1.0.0`.

4. Atlas and Forge import payload types from `Canon/events`, not
   `Nexus/pkg/events`.

5. `nexus/pkg/events/topics.go` carries a deprecation notice.

---

## Next ADR

ADR-046 — Accord v0.2.0: `TunnelDTO` and `ExposeRequest` types
required by the Relay service (ADR-041 §Changes to existing services).
