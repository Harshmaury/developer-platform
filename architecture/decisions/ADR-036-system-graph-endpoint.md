# ADR-036 — GET /system/graph: Unified Topology Endpoint

**Status:** Accepted  
**Date:** 2026-03-21  
**Author:** Harsh Maury  
**Scope:** Nexus — new /system routes  
**Depends on:** ADR-001 (Nexus as registry), ADR-033 (deregister)

---

## Context

The Platform Backbone Proposal requires a single source of truth for system
topology. Currently callers must make three separate requests to reconstruct
the topology:

1. `GET /services` — service runtime state
2. `GET /projects` — registered projects
3. `GET /services/{id}` × N — per-service dependencies

No single response answers: "what is the current shape of the platform and
how do its parts relate?"

---

## Decision

Add two routes under `/system/` to Nexus:

```
GET  /system/graph     — full topology: services, projects, edges, agents
POST /system/validate  — pre-execution policy gate (ADR-038)
```

### GET /system/graph response

```json
{
  "ok": true,
  "data": {
    "services": [
      {
        "id": "atlas-daemon",
        "name": "atlas-daemon",
        "project": "atlas",
        "type": "process",
        "desired_state": "running",
        "actual_state": "running",
        "depends_on": [],
        "fail_count": 0
      }
    ],
    "projects": [
      {
        "id": "atlas",
        "name": "atlas",
        "language": "go",
        "type": "web-api",
        "services": ["atlas-daemon"]
      }
    ],
    "edges": [],
    "agents": ["local"]
  }
}
```

### Rules

1. `GET /system/graph` is **read-only** — never modifies state.
2. Dependencies are fetched per-service from `depends_on` column.
3. Agents are included as IDs only — no token exposure.
4. Response is always a complete snapshot — no pagination, no filtering.
   For large workspaces, cache at the consumer level.
5. Observer services consume this endpoint to replace their separate
   `/services` + `/projects` calls.

---

## Implementation

### New files
```
internal/api/handler/system.go   — SystemHandler with Graph() + Validate()
```

### Modified files
```
internal/api/server.go
  mux.HandleFunc("GET  /system/graph",     systemH.Graph)
  mux.HandleFunc("POST /system/validate",  systemH.Validate)
```

---

## Compliance

| ADR | Status |
|-----|--------|
| ADR-001 | ✅ Nexus remains the sole registry — graph reads from Nexus DB only |
| ADR-003 | ✅ HTTP/JSON on 127.0.0.1 — no new transport |
| ADR-020 | ✅ Read-only endpoint — observers call GET never POST |
