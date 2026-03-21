# engx v3 — Distributed Platform Architecture & Evolution Roadmap

**Version:** 1.0  
**Date:** 2026-03-21  
**Author:** Harsh Maury  
**Status:** Planning — no v3 code until v2.0.0 is stable and tagged  
**Builds on:** STRATEGY.md §6, platform-capability-boundaries.md, architecture-evolution-rules.md

---

## 1. What v2 gives us (the foundation)

Before designing v3, understand what already exists. v2.0.0 ships with:

| Already built | What it enables in v3 |
|---|---|
| `GET /system/graph` — full topology in one endpoint | Web dashboard with zero extra API work |
| Structured events with `span_id`, `parent_span_id`, `level` | Distributed trace assembly across machines |
| `engxa` agent — connects remote machines | Multi-machine coordination already partially wired |
| Herald typed client + Accord contract types | Cross-machine API calls are type-safe from day one |
| Guardian G-rules + system/validate V-rules | Policy enforcement works at any scale |
| Forge cron + event triggers | Automation engine ready for remote execution |
| Sentinel AI reasoner | Insight generation works on any machine's data |

v3 is not a rewrite. It is a distribution layer on top of a complete local platform.

---

## 2. The v3 thesis

> v2 runs your local system.  
> v3 makes your local system public, shareable, and team-aware.

Three capabilities define v3:

1. **Expose** — give any local service a public URL instantly (`engx expose`)
2. **Team** — share a Nexus registry across a group of developers
3. **Remote** — run services on any machine from one CLI

Everything else in v3 is infrastructure that enables these three.

---

## 3. v3 phase plan

v3 is delivered in three sequential phases. Each phase is independently
useful — Phase 2 does not require Phase 3 to be valuable.

```
Phase 1 — Public Endpoints     engx expose → subdomain routing
Phase 2 — Team Workspace       shared registry + multi-user auth
Phase 3 — Remote Execution     run Forge on any machine from CLI
```

---

## 4. Phase 1 — Public Endpoints (engx expose)

### What it is

```bash
engx expose api
# → https://api.harsh.engx.dev
```

A local service running on port 8082 becomes reachable at a public HTTPS URL
in under 5 seconds. No DNS setup. No SSL configuration. No reverse proxy
to configure. One command.

### Why this matters

This is the highest-leverage v3 capability for user acquisition:
- Frictionless for individual developers (show your local work to teammates)
- Works on the existing engxd daemon — no new runtime required
- Creates the `*.engx.dev` subdomain surface that the team and cloud tiers build on
- Directly monetizable (paid plans get custom domains, higher bandwidth, persistence)

### Architecture

```
Developer machine              engx.dev infrastructure
─────────────────              ───────────────────────
engxd (port 8082)
     ↕ tunnel (TLS)
engxa (tunnel agent)  ───────→  Relay Server
                                     ↕
                               Routing Layer
                                     ↕ HTTPS
                       ←──────  api.harsh.engx.dev
                                     ↑
                               DNS wildcard
                               *.engx.dev → relay
```

**How it works:**
1. `engx expose api` tells engxd which service to expose
2. engxa opens a persistent TLS tunnel to the relay server
3. The relay assigns `api.harsh.engx.dev` and routes HTTPS traffic through the tunnel
4. The developer gets a stable URL as long as engxd is running

**Relay server:** A single Go process on a cheap VPS ($5–10/month Hetzner).
It multiplexes TLS tunnels — one connection per exposed service. No compute
cost per request — it is a pure TCP relay. The entire relay can be implemented
in ~300 lines of Go using `net.Conn` multiplexing.

**DNS:** One wildcard record: `*.engx.dev → relay IP`. Set once in Cloudflare.
SSL via Let's Encrypt wildcard cert, auto-renewed.

**Cost to operate:** $5–10/month for the relay VPS. Scales to thousands of
concurrent tunnels on a single machine (each tunnel is a long-lived TCP connection,
not a process).

### New service: Relay

```
relay/
  cmd/relay/main.go          — tunnel multiplexer, HTTP router
  internal/tunnel/           — TLS tunnel management, connection registry
  internal/router/           — subdomain → tunnel routing
  internal/auth/             — token validation (engxa connects with service token)
  internal/domain/           — subdomain assignment, availability check
```

Capability domain: **Control** (it is a runtime coordination service — routes traffic
to registered services). Owned by the relay service, not Nexus.

### New CLI command: engx expose

```bash
engx expose <service>              # expose a service
engx expose <service> --name api   # custom subdomain prefix
engx expose list                   # show all active tunnels
engx expose stop <service>         # stop exposing
```

Output:
```
  ✓ api.harsh.engx.dev
    → 127.0.0.1:8082 (my-api-daemon)
    → live — press Ctrl+C to stop
```

### engxa changes

engxa gains a `tunnel` mode alongside its existing agent mode:

```go
// existing
engxa --id local --server http://127.0.0.1:8080 --token ...

// new tunnel mode (internal, called by engx expose)
engxa tunnel --relay relay.engx.dev:9090 --local 127.0.0.1:8082 --subdomain api.harsh
```

engxa already exists and connects to remote servers. The tunnel capability
is a new connection type in the same binary — no new binary required.

### Monetization surface (Phase 1)

| Tier | Tunnels | Custom domain | Bandwidth | Persistence |
|------|---------|--------------|-----------|-------------|
| Free | 1 active | ✗ | 1 GB/month | Session only |
| Pro $9/mo | 5 active | ✓ | 20 GB/month | Persistent URL |
| Team $29/mo | 20 active | ✓ | 100 GB/month | Persistent URL |

**Persistence** means the subdomain is reserved for you permanently —
it does not change between sessions.

---

## 5. Phase 2 — Team Workspace

### What it is

A shared Nexus registry that multiple developers connect to. One developer's
`engx ps` shows all projects across the team, regardless of which machine
they run on.

```bash
engx team create my-team
engx team invite alice@company.com
engx team join https://team.engx.dev/my-team --token <invite>

# Alice's machine now syncs with the shared registry
engx ps
#  ✓ api         running   (harsh-machine)
#  ✓ worker      running   (alice-machine)
#  ✗ database    stopped   (alice-machine)
```

### Architecture

```
Developer A machine          Developer B machine
engxd (local)                engxd (local)
    ↕ sync                       ↕ sync
         Team Nexus (cloud)
         ─────────────────
         Shared project registry
         Shared event stream
         Multi-user auth (GitHub OAuth)
         Team-scoped service tokens
```

**Team Nexus** is the same Nexus codebase, deployed on a server, with:
- GitHub OAuth for user identity (no new auth system to build)
- Multi-tenant project namespacing (`team/project` instead of `project`)
- Read-only sync from local engxd to team registry (local state is source of truth)
- SSE stream subscription shared across all team members

**Key design principle:** Local engxd is still the source of truth for its own
services. Team Nexus is a read aggregator and coordination layer — it does not
control services on individual machines.

### New service: Gate (auth + identity)

```
gate/
  cmd/gate/main.go           — OAuth flow, token issuance
  internal/auth/             — GitHub OAuth, JWT tokens
  internal/team/             — team CRUD, membership
  internal/invite/           — invite link generation and redemption
```

Capability domain: **Control** (manages identity and team membership — a
coordination responsibility).

### Nexus changes for v3

Nexus gains a `--team` flag and a sync mode:

```bash
engxd --team https://team.engx.dev/my-team --token <team-token>
```

When `--team` is set:
- Local engxd registers its projects and service states with the team registry on a 5s push interval
- Team registry streams changes back to all connected local daemons
- No command-and-control — team registry cannot start/stop services on a remote machine

### Monetization surface (Phase 2)

| Tier | Team members | Shared registry | SSE stream | History |
|------|-------------|----------------|------------|---------|
| Free | — | ✗ | ✗ | ✗ |
| Pro | 1 (self) | ✗ | ✗ | ✗ |
| Team $29/mo | 5 | ✓ | ✓ | 7 days |
| Business $99/mo | 25 | ✓ | ✓ | 90 days |

---

## 6. Phase 3 — Remote Execution

### What it is

Run a Forge build or project start on any registered machine from your local CLI:

```bash
engx run api --on alice-machine
engx build api --on ci-server
engx platform start --on production-vps
```

This is the completion of what `engxa` was always designed for. The agent
already connects remote machines to the control plane. Phase 3 adds
the execution routing layer.

### Architecture

```
Developer CLI (engx)
    ↓
Local engxd
    ↓ (via team registry)
Target engxd (alice-machine)
    ↓
Forge on alice-machine
    ↓ output streams back
Developer CLI sees logs
```

**How it works:**
1. `engx run api --on alice-machine` sends `CmdProjectStart` to the team registry
2. Team registry routes the command to the engxd registered as `alice-machine`
3. That engxd executes locally and streams events back through the team SSE channel
4. Developer's CLI subscribes to those events and shows real-time output

**Key design principle:** Commands route through the team registry.
The target machine's engxd executes the command locally — there is no remote
code execution service. Forge on the remote machine does the actual work.

### Forge changes for v3

Forge gains a `--remote` flag in its intent execution pipeline:

```go
// ADR-005: Forge instructs Nexus via POST /projects/:id/start|stop
// v3 extension: Forge can target a remote Nexus via team registry routing
type ExecutionTarget struct {
    Local  bool
    Remote string  // machine ID, "" = local
}
```

No new binary. No new service. Forge's existing execution pipeline gains
a routing layer in front of the Nexus client call.

### New service: Conduit (command routing)

```
conduit/
  cmd/conduit/main.go        — command router, runs on team server
  internal/router/           — machine ID → engxd routing
  internal/stream/           — bidirectional command/event streaming
  internal/auth/             — team token validation
```

Capability domain: **Execution** (translates developer intent into
coordinated actions across machines — exactly Forge's domain at the
infrastructure level).

### Monetization surface (Phase 3)

| Tier | Remote machines | Concurrent remote runs | CI integration |
|------|----------------|----------------------|----------------|
| Free | — | — | ✗ |
| Pro | 1 | 1 | ✗ |
| Team $29/mo | 5 | 3 | ✓ |
| Business $99/mo | 25 | 10 | ✓ |

---

## 7. New services summary

| Service | Phase | Domain | Role |
|---------|-------|--------|------|
| Relay | 1 | Control | TLS tunnel multiplexer, subdomain routing |
| Gate | 2 | Control | GitHub OAuth, team membership, token issuance |
| Conduit | 3 | Execution | Remote command routing, event stream relay |

Three new services across three phases. Each is small (~300–500 lines core logic).
None of them replaces anything in v2 — they are additive.

---

## 8. Infrastructure requirements

| Phase | What | Provider | Cost |
|-------|------|----------|------|
| 1 | Relay VPS (2 vCPU, 2GB RAM) | Hetzner CX22 | ~$4/month |
| 1 | Wildcard DNS (`*.engx.dev`) | Cloudflare | Free |
| 1 | Wildcard SSL cert | Let's Encrypt | Free |
| 2 | Team Nexus server (same VPS or separate) | Hetzner | ~$4/month |
| 2 | PostgreSQL (team registry persistence) | Neon free tier | Free |
| 3 | Conduit (same server as team) | Existing VPS | $0 additional |

**Total infra cost at launch: ~$8–12/month.**

This is achievable before any paying user exists.

---

## 9. What does NOT change in v3

These v2 guarantees hold permanently:

- Local runtime is always free and always fully functional without a network connection
- `engxd` on your machine is always the source of truth for your services
- No cloud service can start or stop services on your machine without your explicit token
- The three capability domains (Control / Knowledge / Execution) do not change
- ADR-first rule applies to every new v3 capability
- No cross-service internal imports — everything through HTTP APIs

v3 extends the platform. It does not replace the local runtime or compromise
the local-first guarantee.

---

## 10. ADR sequence for v3

Each of the following requires a committed ADR before any implementation:

| ADR | Title | Phase | Unblocks |
|-----|-------|-------|---------|
| ADR-041 | `engx expose` — public endpoint tunneling | 1 | Relay service |
| ADR-042 | Relay service — TLS tunnel multiplexer | 1 | Phase 1 infra |
| ADR-043 | engxa tunnel mode | 1 | Expose command |
| ADR-044 | `*.engx.dev` subdomain system | 1 | DNS + SSL |
| ADR-045 | Gate service — GitHub OAuth + team identity | 2 | Team workspace |
| ADR-046 | Team Nexus — multi-user shared registry | 2 | Phase 2 |
| ADR-047 | Local↔Team sync protocol | 2 | Remote visibility |
| ADR-048 | Conduit service — remote command routing | 3 | Remote execution |
| ADR-049 | Forge remote execution target | 3 | `engx run --on` |
| ADR-050 | Billing layer — tier enforcement | All | Monetization |

---

## 11. Capability evolution diagram

```
v2 (now)                          v3 Phase 1         v3 Phase 2         v3 Phase 3
─────────────────────────────     ────────────────   ────────────────   ────────────────
Local Control Plane               + Public Endpoints  + Team Workspace   + Remote Exec
  engxd — registry, state           Relay service       Gate service       Conduit service
  Atlas — workspace graph           engxa tunnels       Team Nexus          Remote Forge
  Forge — build/run/deploy          engx expose         engx team           engx run --on
  Guardian — policy                 *.engx.dev DNS      GitHub OAuth        Command routing
  Sentinel — AI insights            SSL automation      Shared SSE stream   Event relay
  Observer — trace assembly
  Metrics — Prometheus
  Navigator — topology
  Herald — typed client
  Accord — contracts
  Canon — constants
```

---

## 12. The income path

```
v2.0.0 (now)    GitHub Sponsors — active, anyone can sponsor
                Landing page live — harshmaury.github.io/Nexus

Phase 1         engx expose free tier → Pro $9/mo
                First paying users. Infrastructure cost ~$8/mo.
                Break-even at 1 Pro subscriber.

Phase 2         Team tier $29/mo
                5 team members, shared registry, SSE stream.
                10 teams = $290/mo. 100 teams = $2,900/mo.

Phase 3         Business tier $99/mo
                Remote execution, CI integration, 25 members.
                10 businesses = $990/mo.
```

Phase 1 is the critical unlock. It is the first capability that
requires infrastructure, creates recurring cost, and justifies
a paid tier. Everything before Phase 1 is sponsorship-driven.

---

## 13. What to build first

The single most important next step is **ADR-041** (`engx expose`).

Not because it is technically complex — it is actually simple (~300 lines
for the relay, ~100 lines for the CLI command). But because:

1. It is the first user-facing v3 capability
2. It creates the `*.engx.dev` surface that Phase 2 and 3 build on
3. It is directly monetizable with a free/pro split
4. It is HN-friendly: "instant public URL for your local service" is a
   compelling demo in 10 seconds

Start with ADR-041. Write the ADR first. Code second.
