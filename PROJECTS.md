# Platform Projects
# @version: 2.1.0
# @updated: 2026-03-15

Complete registry of all developer platform repositories.

---

## Core Platform Triangle

| Project | Domain    | Purpose                              | Port  | Repository                            | Status |
|---------|-----------|--------------------------------------|-------|---------------------------------------|--------|
| Nexus   | Control   | System coordination, service runtime | 8080  | https://github.com/Harshmaury/Nexus   | ✅ Phases 1–14 complete |
| Atlas   | Knowledge | Workspace awareness, source indexing | 8081  | https://github.com/Harshmaury/Atlas   | ✅ Phase 2 complete (v0.2.0) |
| Forge   | Execution | Intent execution, workflow engine    | 8082  | https://github.com/Harshmaury/Forge   | ⏳ Phase 1 ready to start |

---

## Governance

| Repository         | Purpose                                      |
|--------------------|----------------------------------------------|
| developer-platform | Platform architecture, ADRs, design rules    |

https://github.com/Harshmaury/developer-platform

---

## Local Workspace Paths

```
~/workspace/projects/apps/nexus/
~/workspace/projects/apps/atlas/
~/workspace/projects/apps/forge/
```

---

## Ports

```
Nexus   127.0.0.1:8080   NEXUS_HTTP_ADDR
Atlas   127.0.0.1:8081   ATLAS_HTTP_ADDR
Forge   127.0.0.1:8082   FORGE_HTTP_ADDR
```

Override any port via environment variable before starting the service.

---

## Binaries

```
~/bin/engxd    Nexus daemon
~/bin/engx     Nexus CLI
~/bin/engxa    Nexus remote agent
~/bin/atlas    Atlas knowledge service    ✅ Phase 2 complete
~/bin/forge    Forge execution engine     ⏳ Phase 1 not yet built
```

---

## Phase Dependency Chain

```
Nexus Phases 1–14 ✅  →  Atlas Phase 1 ✅  →  Atlas Phase 2 ✅  →  Forge Phase 1  →  Forge Phase 2  →  Forge Phase 3
```

Forge Phase 1 requires Atlas Phase 1 running (context enrichment via Atlas HTTP API).

---

## Atlas API Reference (for Forge client)

```
Phase 1:
  GET  /health
  GET  /workspace
  GET  /workspace/projects
  GET  /workspace/project/:id
  GET  /workspace/search?q=
  GET  /workspace/context

Phase 2:
  GET  /workspace/capabilities
  GET  /workspace/conflicts
  GET  /workspace/graph
```

---

## Tags

```
Nexus   v0.14-stable           Phases 1–14
Atlas   v0.2.0-phase2-complete Phase 2 — capability model + graph + conflict detection
```

---

## Adding a New Project

Before creating a new repository:

1. Identify which capability domain it belongs to.
2. Confirm it does not duplicate an existing capability.
3. Create an ADR in `architecture/decisions/`.
4. Add it to this registry.
5. Create the repository following the project scaffold pattern.

Capability boundaries: `architecture/platform-capability-boundaries.md`
