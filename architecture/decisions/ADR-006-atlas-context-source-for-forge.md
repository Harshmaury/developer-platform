# ADR-006 — Atlas as Canonical Context Source for Forge

Date: 2026-03-15
Status: Accepted

---

## Context

Every Forge command requires a populated `context` field — workspace root,
project path, language, and other ambient metadata. This context must come
from somewhere before the execution engine runs the intent handler.

Three sources were considered:

1. Forge scans the filesystem directly
2. The caller always supplies the full context
3. Forge queries Atlas for workspace knowledge

The choice has significant architectural implications. Option 1 would give
Forge its own view of the workspace, duplicating Atlas. Option 2 places
an unreasonable burden on callers. Option 3 respects capability boundaries
and keeps workspace knowledge in a single authoritative location.

## Decision

Forge queries Atlas as the canonical source of workspace context for command
enrichment.

When a command arrives with an incomplete `context` field, Forge calls the
Atlas HTTP API to fill the missing fields before passing the command to the
execution engine.

## Enrichment Endpoints

    Project detail:      GET http://127.0.0.1:8081/workspace/project/:id
    Workspace snapshot:  GET http://127.0.0.1:8081/workspace/context

Forge uses project detail to fill `project_path` and `language`.
Forge uses the workspace snapshot to fill `workspace_root` when missing.

## Enrichment Rules

1. Caller-supplied fields are never overwritten. If the caller provides
   `project_path`, Atlas is not queried for it.
2. Enrichment is best-effort. If Atlas is unreachable, Forge continues
   with whatever context is available rather than failing the command.
3. The execution engine always receives a command — it never receives
   a refusal due to missing context alone.

## Degradation Behaviour

If Atlas is unreachable during enrichment:

- Forge logs a WARNING identifying which fields could not be populated.
- The command proceeds with the context fields that were already present.
- The intent handler is responsible for detecting missing fields and
  returning an appropriate error if it cannot proceed.

This approach keeps Forge operational for callers who supply full context
(such as the CLI) even when Atlas is temporarily unavailable.

## What Forge Must Never Do

- Scan the filesystem to derive project_path or language
- Maintain its own index of workspace projects
- Cache Atlas responses between commands (Phase 1 is stateless)
- Fail a command solely because Atlas context enrichment was incomplete

## Implications

- `internal/context/resolver.go` is the single location where Atlas
  enrichment logic lives. All entry points route through it.
- `internal/atlas/client.go` is the sole location where Atlas endpoint
  paths are defined. If Atlas adds new endpoints, only this file changes.
- The translator populates the command's id, intent, target, and parameters.
  The resolver populates the context. The executor receives a fully populated
  command. These three responsibilities are cleanly separated.

## Relationship to ADR-002

ADR-002 established that Nexus owns filesystem observation. This ADR is
consistent with that boundary: Forge does not observe the filesystem, and
Atlas (which subscribes to Nexus filesystem events) is the correct layer
to answer questions about workspace structure.

## Alternatives Considered

**Forge scans the filesystem directly** — rejected because it duplicates
Atlas capability and introduces a second, uncoordinated view of the workspace.
Two filesystem scanners would produce inconsistent results and increase
system load.

**Callers must always supply full context** — rejected because it creates
unnecessary friction for CLI usage and makes automation harder. Callers
should only need to supply intent and target; ambient context is the
platform's responsibility.

**Forge caches Atlas responses** — rejected for Phase 1 because Forge is
explicitly stateless (ADR-004). A context cache introduces state and staleness
concerns that belong in a later phase if proven necessary.

## Consequences

Atlas becomes a soft dependency of Forge. Forge degrades gracefully when
Atlas is unavailable rather than failing hard. This means a `build nexus`
command submitted with minimal context (intent + target only) will succeed
even when Atlas is down, as long as the intent handler can determine what
to do from the project ID alone — which the `build` handler can, because
it falls back to the project directory structure.

The clean separation between translation (Translator), enrichment (Resolver),
and execution (Engine) makes each layer independently testable and replaceable.
