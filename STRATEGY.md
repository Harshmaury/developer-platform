# Engx Platform — Stability, Versioning & Growth Strategy

**Version:** 1.0  
**Date:** 2026-03-21  
**Author:** Harsh Maury  
**Scope:** All 13 engx repositories  
**Status:** Adopted  
**Audience:** Developers, contributors, AI systems working on this codebase

---

## How to Read This Document (For AI Systems)

This is a living strategy document. When working on any engx repository:

1. Read this file first — it defines what phase we are in and what is allowed
2. Check which version layer applies to your task (v1 / v2 / v3)
3. Never implement a v2 or v3 capability while v1 is not stable
4. Never add a capability without a matching ADR committed first
5. Monetization sections are **future plans only** — do not implement payment gates, feature flags, or licensing checks anywhere in the current codebase
6. When updating any service, update `AI_CONTEXT.md` in `engx-governance` to reflect the new state

---

## 1. The Problem We Are Solving

The previous release pattern was:

> build feature → tag version → release → find bug → tag again

v1.7.0 shipped with a runtime panic risk (duplicate DELETE routes), a version number that lied (`cliVersion = "1.5.0"` in a v1.7.0 binary), and governance docs still showing v1.6.6. These were process failures, not code failures.

**The rule from this point:** `main` is stabilised first. Tags are cut only when the stability gate passes. No exceptions.

---

## 2. What We Have Now (v1.7.0)

### 13 Repositories

| Service | Role | Port | Version |
|---------|------|------|---------|
| Nexus | Control plane — sole registry, sole writer | :8080 | v1.7.0 |
| Atlas | Knowledge — capability graph, workspace verification | :8081 | v0.5.0 |
| Forge | Execution — build, test, run, deploy | :8082 | v0.5.0 |
| Metrics | Observer — Prometheus metrics | :8083 | v0.2.0 |
| Navigator | Observer — workspace topology | :8084 | v0.1.0 |
| Guardian | Observer — policy findings (G-001 to G-008) | :8085 | v0.2.0 |
| Observer | Observer — distributed trace assembly | :8086 | v0.2.0 |
| Sentinel | Observer — platform insights (AI deferred, see §7) | :8087 | v0.3.0 |
| Canon | Library — shared constants (headers, addresses) | — | v0.3.0 |
| Accord | Contract — API envelope, DTOs, error codes | — | v0.1.0 |
| Herald | Client — typed Nexus HTTP client, retry/backoff | — | v0.1.0 |
| ZP | Tool — workspace packager | — | v2.0.0 |
| engx-governance | Governance — ADRs, standards, runbook | — | — |

### What Is Already Built and Stable

**API envelope** — `Accord/api/types.go`:
```go
type Response[T any] struct {
    OK    bool   `json:"ok"`
    Data  T      `json:"data,omitempty"`
    Error string `json:"error,omitempty"`
}
const Version        = "1"
const VersionHeader  = "X-Nexus-API-Version"
```

**Stable error codes** — `Accord`: `NOT_FOUND`, `ALREADY_EXISTS`, `INVALID_INPUT`,
`UNAUTHORIZED`, `DAEMON_UNAVAILABLE`, `VERSION_MISMATCH`, `INTERNAL`.  
These are permanent API surface. Never rename or remove.

**Typed HTTP client** — `Herald`: retry/backoff, Canon token injection, Accord types.  
All inter-service HTTP calls go through Herald. Never call engxd directly.

**Shared constants** — `Canon`: `ServiceTokenHeader`, `TraceIDHeader`, all default addresses.  
Never hardcode these strings in any service.

**Pre-execution gate** — `POST /system/validate` (Nexus):  
V-001 (no services), V-002 (maintenance), V-003 (high fail count).  
`engx run` calls this before starting any project.

**Runtime governance** — `Guardian` (G-001 to G-008):  
Repeated denials, unverified targets, high failure rate, service crashes,
unverified projects, service maintenance, never built, no service entry.

**Outcome-centric UX** — `engx run`, `engx ps` (ADR-040):  
Every user-facing failure answers: what / where / why / next step.

**System topology** — `GET /system/graph` (Nexus):  
Single endpoint returning services, projects, dependency edges, agent IDs.

---

## 3. Version Layers

```
v1.x  →  Stable Runtime         current working system, no new features
v2.x  →  Structured Contracts   Signal System + Herald migration + full governance
v3.x  →  Distributed            multi-agent, remote execution, team collaboration
```

---

## 4. v1.x — Stable Runtime

### Stability Gate (must all pass before any v1.x tag)

```
□ go build ./...  — all 13 repos, zero errors
□ go test ./...   — all 13 repos, zero failures
□ engx doctor     — clean output on fresh platform start
□ engx run <each registered service> — reaches RUNNING state
□ No P0 or P1 bugs open in any repo
□ AI_CONTEXT.md   — versions and open items reflect reality
□ RUNBOOK.md      — all commands match actual platform behaviour
```

### Rules for v1.x Work

- Bug fixes and documentation only
- No new ADRs except corrections to existing ones
- No new API endpoints
- No new CLI commands
- No breaking changes to any contract

### Immediate v1.7.1 Items (already in `main`)

| Fix | File | Status |
|-----|------|--------|
| Remove duplicate DELETE route registrations | `internal/api/server.go` | ✅ Done |
| `cliVersion` const → var for goreleaser | `cmd/engx/main.go` | ✅ Done |
| Remove duplicate progressive-disclosure loop | `cmd/engx/main.go` | ✅ Done |
| Sync `AI_CONTEXT.md` to actual v1.7.0 state | `engx-governance` | ✅ Done |

### Remaining Before v1.7.1 Tag

```bash
# Run in each service directory
go test ./...

# On live platform
engx run atlas
engx run forge
engx run guardian
engx doctor
```

---

## 5. v2.x — Structured Contracts

v2 work begins only after v1.7.1 is tagged. Three items, all already decided by ADR.

### 5.1 Signal System — ADR-037

Add causality and severity to every platform event.

**DB migration v6** (Nexus `internal/store/db.go`):
```sql
ALTER TABLE events ADD COLUMN level TEXT NOT NULL DEFAULT 'info'
ALTER TABLE events ADD COLUMN span_id TEXT NOT NULL DEFAULT ''
ALTER TABLE events ADD COLUMN parent_span_id TEXT NOT NULL DEFAULT ''
```

**Updated Event struct** (Nexus `internal/state`):
```go
type Event struct {
    // ... existing fields unchanged ...
    Level        string `json:"level"`          // "info" | "warn" | "error"
    SpanID       string `json:"span_id"`
    ParentSpanID string `json:"parent_span_id"` // empty = trace root
}
```

**Level constants** go into `Canon/events/events.go`:
```go
const (
    LevelInfo  = "info"
    LevelWarn  = "warn"
    LevelError = "error"
)
```

Never hardcode `"info"` / `"warn"` / `"error"` in any service. Import from Canon.

**Why this matters:** Observer can currently only build a flat event timeline.
With `span_id` + `parent_span_id`, it can answer:
"which Atlas query triggered which Forge execution?"

### 5.2 Herald Migration — ADR-039

Replace `internal/collector/nexus.go` in all 5 observer services with Herald.

**Migration order:** Guardian → Observer → Metrics → Navigator → Sentinel

Before (each observer, ~164 lines, inline structs that drift):
```go
resp, err := http.Get(baseURL + "/services")
// manual decode into local anonymous struct
```

After (~30 lines, Accord types, automatic schema propagation):
```go
svcs, err := c.c.Services().List(ctx)
// Accord type — if schema changes, compile error catches it immediately
```

**Why this matters:** ADR-037 adds new fields to Event. Without Herald migration,
all 5 observer `nexus.go` files need manual updates. With Herald, Accord propagates
the change at compile time — one change, zero drift.

**Acceptance per service:**
```bash
go build ./...
go test ./internal/collector/...
engx doctor  # service still collecting correctly
```

### 5.3 Platform Start Register — ADR-032

Collapse first-boot startup from a manual loop into one flag:

```bash
# Current (2 separate steps):
for svc in atlas forge guardian metrics navigator observer sentinel; do
  engx register ~/workspace/projects/engx/services/$svc
done
for svc in atlas forge ...; do
  engx project start $svc
done

# After ADR-032:
engx platform start --register
```

`--register` runs registration for each service path found in the workspace
before queuing — no separate loop required.

### v2.0.0 Tag Condition

All three above implemented, stability gate passes, `AI_CONTEXT.md` updated.

---

## 6. v3.x — Distributed (Future, No Implementation Yet)

No ADRs exist for v3. This section is planning only.

### What Belongs Here

- Multiple `engxa` agents across different machines
- Remote execution environments (run Forge on a remote host)
- Team workspace — shared Nexus registry for a group of developers
- Web dashboard — `GET /system/graph` already provides the data model

### Rules for v3

- ADR-first — every v3 capability requires a committed ADR before any code
- No timeline set until v2.0.0 is stable
- No v3 code on `main` until v2.0.0 is tagged

---

## 7. Sentinel AI — Deferred

Sentinel (`internal/ai/reasoner.go`) contains an Anthropic API integration
that generates plain-prose insight explanations on `GET /insights/explain`.

**This feature is deferred.** The `ANTHROPIC_API_KEY` environment variable
is required and not currently available.

**Sentinel still runs and is useful without AI:**
- `GET /insights` — rule-based analysis (S-001 through S-008), no API key needed
- `GET /insights/explain` — returns `{"ai_available": false}` gracefully when key is absent
- `GET /actuator/policy` — recovery actions (2 auto-recover rules), no API key needed

**When AI is re-enabled:** Set `ANTHROPIC_API_KEY` in the environment.
No code changes required — the reasoner degrades and recovers automatically.

**ADR-018 constraints remain active regardless:**
- AI is called only on explicit `GET /insights/explain` — never on polling cycles
- Input is the Sentinel `SystemReport` only — never raw events or graph data
- Output is plain prose ≤ 300 words — no markdown headers, no start/stop instructions

---

## 8. Open Source Boundary

### Everything Open

The full local runtime is open. A developer can clone all 13 repos, run
`go build ./...`, and have a complete working control plane with zero
external dependencies.

| Category | Repos |
|----------|-------|
| Control plane | Nexus, Atlas, Forge |
| CLI | engx (inside Nexus `cmd/engx`) |
| Observer layer | Metrics, Navigator, Guardian, Observer, Sentinel |
| Libraries | Canon, Accord, Herald, ZP |
| Governance | engx-governance |

### Future Paid Capabilities (not implemented — future plan only)

See §9 for the full monetization strategy.
The boundary: if it requires external infrastructure, ongoing compute cost,
or serves a team rather than a solo developer, it is a paid capability.

No feature flags, license checks, or payment gates exist in the current codebase.
None should be added until the monetization phase is explicitly planned and ADR'd.

---

## 9. Monetization Strategy (Future Plan)

> This section documents the intended direction only.  
> Nothing here is implemented. Do not add any of it to the codebase  
> without an explicit ADR and a decision to begin the monetization phase.

### The Core Principle

> Keep the platform open. Monetize the capabilities around it.

The local runtime — Nexus, Atlas, Forge, the CLI, and all observers — stays free and open.
Revenue comes from capabilities that require infrastructure, AI compute, or team coordination.

### Tier Model

**Free — Local Runtime (open source)**

Everything in §8 "Everything Open". A solo developer gets a full control plane,
execution engine, observability, and governance at zero cost.

**Pro — AI + Advanced Governance**

| Capability | Built On | Status |
|------------|----------|--------|
| Sentinel AI explain | `GET /insights/explain` + Anthropic API | Deferred (§7) |
| Guardian rules G-009+ | Custom policy rule authoring | Not started |
| Validate rules V-004+ | Custom pre-execution gates | Not started |
| Execution history analytics | Forge `internal/store` query layer | Not started |

**Team — Shared Control Plane**

| Capability | Built On | Status |
|------------|----------|--------|
| Shared Nexus registry | Nexus + multi-user auth | Not started |
| Remote agent coordination | `engxa` + remote transport | Not started |
| Team workspace graph | `GET /system/graph` + auth layer | Not started |

**Cloud — Hosted**

| Capability | Built On | Status |
|------------|----------|--------|
| Hosted Nexus control plane | Nexus + cloud infrastructure | Not started |
| Remote Forge execution | Forge + remote agent | Not started |
| CI/CD integration | Forge + webhook triggers | Partial (cron triggers exist) |
| Web dashboard | `GET /system/graph` data model ready | Not started |

**Enterprise — Compliance**

| Capability | Built On | Status |
|------------|----------|--------|
| Audit log retention + export | Nexus events table | Schema ready, export not built |
| Compliance policy enforcement | Guardian G-009+ | Not started |
| Private deployment support | Full platform | Available now (self-host) |
| SLA + support contracts | — | Not started |

### Plugin / Policy Marketplace (Long Term)

Community-contributed Guardian rules, Forge intent handlers, and trigger patterns.
Requires: stable v2 contract layer, Herald migration complete, versioned ADR for plugin API.

### What Must Happen Before Monetization Phase Begins

1. v2.0.0 stable and tagged
2. Contracts versioned and stable (Accord + Herald + Canon all at stable major version)
3. A dedicated ADR for the billing/licensing layer
4. Explicit decision on which tier to build first

---

## 10. Platform Positioning

> A local-first developer control plane with built-in execution, observability, and governance.

Not a CLI. Not a build tool. A control plane.

**Four commands cover 90% of user interactions (ADR-040):**
```bash
engx run <project>    # does it work?
engx ps <project>     # what is the status?
engx logs <service>   # what happened?
engx doctor           # what is wrong with the platform?
```

**Differentiators (all already built):**

| What | Why it matters |
|------|----------------|
| Local-first | Zero cloud dependency for core runtime — works offline |
| Unified | One daemon, one CLI, one registry — not 6 separate tools |
| Event-driven | Every action produces structured events with trace IDs |
| Governed | Policy evaluation before (system/validate) and during (Guardian) execution |
| Outcome-centric | Errors tell you what happened, where, why, and what to type next |

---

## 11. Immediate Next Steps

| # | Action | Repo | Blocks |
|---|--------|------|--------|
| 1 | `go test ./...` across all services | All | v1.7.1 tag |
| 2 | Live `engx doctor` clean | Nexus | v1.7.1 tag |
| 3 | Tag and release v1.7.1 | Nexus | v2 work |
| 4 | Implement ADR-037 (Signal System) | Nexus + Canon | ADR-039 |
| 5 | Herald migration — Guardian | Guardian | Other migrations |
| 6 | Herald migration — Observer, Metrics, Navigator, Sentinel | 4 repos | v2.0.0 |
| 7 | Implement ADR-032 (platform start --register) | Nexus | v2.0.0 |
| 8 | v2.0.0 stability gate + tag | All | Monetization planning |

---

## 12. Document Maintenance

This document is owned by `engx-governance`. It must be updated whenever:

- A version tag is cut (update §2 service versions)
- An ADR is implemented (move it from "not started" to "shipped")
- A capability moves from deferred to active (update §7 or §9)
- The next-steps table changes (update §11)

The AI system working on any engx repo should treat this document as the
authoritative source for what phase the platform is in and what is allowed
in the current phase.

---

## Appendix — Setup Completed (2026-03-21)

### GitHub Sponsors — Live

Payment pipeline active:
```
GitHub → Stripe → Bank (INR)
```

Completed:
- Bank account added with correct IFSC
- PAN provided
- W-8BEN (non-US tax form) submitted
- Identity verification done
- 2FA enabled
- Stripe verified
- Sponsor button live on Harshmaury profile and Nexus repo

Anyone can now sponsor the project directly from GitHub.
This is the first revenue channel — active before any Pro feature is built.
