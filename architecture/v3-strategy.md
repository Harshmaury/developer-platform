# engx v3 — Full Strategy Document
# Relay · Gate · Conduit

**Version:** 1.0  
**Date:** 2026-03-21  
**Author:** Harsh Maury  
**Status:** Adopted — planning phase  
**Prerequisites:** v2.0.0 stable and tagged ✅  
**Audience:** All AI systems and contributors working on engx v3

---

## How to read this document

This document is the authoritative reference for v3 architecture.
Read it completely before writing any ADR, any code, or any design proposal
for Relay, Gate, or Conduit.

Rules for AI systems:
1. Never implement v3 capabilities without a committed ADR
2. Never violate the inter-service contracts in §7
3. Never add capabilities that belong to existing services — extend via API only
4. This document supersedes any v3 speculation in STRATEGY.md §6

---

## 1. What v3 is

v2 runs your local system.
v3 makes your local system public, shareable, and team-aware.

Three capabilities define v3:

```
engx expose api         → https://api.harsh.engx.dev     (Phase 1)
engx team join          → shared registry across a team   (Phase 2)
engx run api --on alice → Forge executes on alice machine  (Phase 3)
```

Three new services deliver these capabilities:

```
Relay     — TLS tunnel multiplexer + subdomain routing    (Phase 1)
Gate      — GitHub OAuth + team identity + token issuance  (Phase 2)
Conduit   — remote command routing + event stream relay    (Phase 3)
```

Nothing in v2 is replaced. Everything in v3 is additive.

---

## 2. What does NOT change in v3

These guarantees are permanent and apply to every v3 service:

| Guarantee | Rule |
|-----------|------|
| Local-first | engxd on your machine is always the source of truth |
| No remote control | No cloud service can start/stop local services without explicit token |
| Offline capable | All v2 capabilities work with no network connection |
| ADR-first | Every v3 capability requires a committed ADR before any code |
| Domain boundaries | Control / Knowledge / Execution — no new domains |
| No cross-imports | v3 services talk to v2 services via Herald + HTTP only |
| Canon constants | All header strings and event types imported from Canon |
| Accord envelope | All HTTP responses use `{ ok, data, error }` |

---

## 3. Phase sequence

Phases are strictly sequential. Phase 2 does not begin until Phase 1 is
stable and tagged. Phase 3 does not begin until Phase 2 is stable and tagged.

```
Phase 1 — Public Endpoints      Relay service + engx expose
Phase 2 — Team Workspace        Gate service + Team Nexus mode
Phase 3 — Remote Execution      Conduit service + Forge remote target
```

Each phase delivers a capability that is independently useful.
A developer on Phase 1 gets full value without Phase 2 or 3 existing.

---

## 4. Relay — Phase 1

### Purpose

Relay gives any local service a public HTTPS URL in one command.

```bash
engx expose api
# → https://api.harsh.engx.dev
```

### Capability domain

**Control** — Relay coordinates runtime connectivity. It manages
which services are reachable, from where, and under what identity.
This is a runtime coordination responsibility, the same domain as Nexus.
Relay does not own service state — Nexus still owns that.

### What Relay owns

- Tunnel registry: which engxa tunnel maps to which subdomain
- Subdomain assignment and availability
- TLS termination for `*.engx.dev`
- HTTP routing: incoming request → correct tunnel → local service
- Connection lifecycle: open, keepalive, close, reconnect

### What Relay does NOT own

- Service state (Nexus owns this)
- User identity (Gate owns this in Phase 2)
- Command execution (Forge/Conduit own this)
- Workspace knowledge (Atlas owns this)
- Any write authority over engxd

### Repository

```
github.com/Harshmaury/Relay
~/workspace/projects/engx/services/relay
```

### Structure

```
relay/
  cmd/relay/main.go                 — startup, config, goroutine lifecycle
  internal/
    tunnel/
      registry.go                   — in-memory tunnel registry (sync.RWMutex)
      conn.go                       — TLS tunnel connection lifecycle
      multiplexer.go                — TCP connection multiplexing
    router/
      subdomain.go                  — subdomain → tunnel lookup
      http.go                       — HTTP reverse proxy to tunnel
    auth/
      token.go                      — engxa service token validation
    domain/
      assign.go                     — subdomain assignment logic
      availability.go               — check subdomain not already taken
    config/
      env.go                        — EnvOrDefault, all config from env vars
  nexus.yaml                        — type: platform-daemon, port: 9090
  SERVICE-CONTRACT.md
  WORKFLOW-SESSION.md
  go.mod                            — module github.com/Harshmaury/Relay
```

### Port assignment

```
Relay tunnel listener:  :9090  (engxa connects here — TLS)
Relay HTTP router:      :9091  (incoming public HTTPS — behind Cloudflare)
```

### How it works

```
1. engx expose api
     ↓
2. engx CLI tells engxd: expose service "my-api-daemon" on port 8082
     ↓
3. engxd instructs engxa (local): open tunnel to relay.engx.dev:9090
     ↓
4. engxa opens persistent TLS connection to Relay
5. Relay assigns subdomain: api.harsh.engx.dev
6. Relay responds to engxa: subdomain confirmed
     ↓
7. engxa reports back to engxd
8. engxd reports to engx CLI: ✓ api.harsh.engx.dev → 127.0.0.1:8082
     ↓
9. Incoming HTTPS request to api.harsh.engx.dev
10. Cloudflare terminates TLS → forwards to Relay :9091
11. Relay router: subdomain lookup → tunnel connection
12. Relay forwards request bytes through tunnel to engxa
13. engxa forwards to 127.0.0.1:8082
14. Response travels back the same path
```

### engxa changes (in Nexus repo)

engxa gains a `tunnel` subcommand. No new binary — same `engxa` binary:

```bash
# existing mode (unchanged)
engxa --id local --server http://127.0.0.1:8080 --token local-agent-token

# new tunnel mode (invoked by engxd when engx expose runs)
engxa tunnel \
  --relay relay.engx.dev:9090 \
  --local 127.0.0.1:8082 \
  --subdomain api \
  --owner harsh \
  --token <relay-service-token>
```

engxa tunnel mode is an internal implementation detail.
Users only ever run `engx expose` — engxd manages engxa tunnel invocation.

### New CLI commands (in Nexus repo, cmd/engx/)

```bash
engx expose <project>              # expose default service of project
engx expose <project> --name <n>   # custom subdomain prefix
engx expose list                   # show all active tunnels
engx expose stop <project>         # stop tunnel for project
```

These live in a new file: `cmd/engx/cmd_expose.go`
Pattern: identical to existing command files in Nexus.

### Accord changes

New DTOs in `Accord/api/types.go`:

```go
// TunnelDTO — active tunnel registration
type TunnelDTO struct {
    ID         string `json:"id"`
    ProjectID  string `json:"project_id"`
    ServiceID  string `json:"service_id"`
    Subdomain  string `json:"subdomain"`      // e.g. "api"
    Owner      string `json:"owner"`           // e.g. "harsh"
    PublicURL  string `json:"public_url"`      // e.g. "https://api.harsh.engx.dev"
    LocalAddr  string `json:"local_addr"`      // e.g. "127.0.0.1:8082"
    ActiveSince string `json:"active_since"`   // RFC3339
}

// ExposeRequest — body for POST /expose
type ExposeRequest struct {
    ServiceID string `json:"service_id"`
    Name      string `json:"name,omitempty"` // subdomain prefix, defaults to service ID
}
```

### Canon changes

New constants in `Canon/identity/identity.go`:

```go
const (
    DefaultRelayAddr   = "relay.engx.dev:9090"
    RelayTokenHeader   = "X-Relay-Token"
    SubdomainHeader    = "X-Engx-Subdomain"
    OwnerHeader        = "X-Engx-Owner"
)
```

### Herald changes

New `TunnelsClient` in `Herald/client/tunnels.go`:

```go
// TunnelsClient provides typed access to the Relay /tunnels API.
type TunnelsClient struct{ c *Client }

func (t *TunnelsClient) Register(ctx context.Context, req accord.ExposeRequest) (*accord.TunnelDTO, error)
func (t *TunnelsClient) List(ctx context.Context) ([]accord.TunnelDTO, error)
func (t *TunnelsClient) Delete(ctx context.Context, id string) error
```

### Infrastructure

```
Server:     Hetzner CX22 — 2 vCPU, 4 GB RAM, 40 GB SSD — ~$4/month
DNS:        *.engx.dev A record → relay server IP (Cloudflare, free)
SSL:        Let's Encrypt wildcard cert (certbot, auto-renew)
Domain:     engx.dev (register once, ~$12/year)
```

### ADRs required

```
ADR-041   engx expose — public endpoint tunneling
ADR-042   Relay service — TLS tunnel multiplexer architecture
ADR-043   engxa tunnel mode — new connection type
ADR-044   *.engx.dev subdomain system — DNS + SSL + routing
```

---

## 5. Gate — Phase 2

### Purpose

Gate provides user identity and team membership for the engx platform.
It is the authentication and authorization layer that Phase 2 and 3 require.

```bash
engx team create my-team
engx team invite alice@company.com
engx team join https://gate.engx.dev/join/abc123
```

### Capability domain

**Control** — Gate manages who has access to what. This is a runtime
coordination responsibility. Gate does not own service state or execution
authority — it owns identity and team membership only.

### What Gate owns

- User identity (GitHub OAuth — no password storage)
- Team creation, membership, and invitation
- Service token issuance for team-scoped access
- Team-scoped subdomain prefixes (Phase 2 extends Phase 1 subdomains)
- Session management (JWT, short-lived tokens)

### What Gate does NOT own

- Service state (Nexus)
- Tunnel connections (Relay)
- Command execution (Forge/Conduit)
- Workspace knowledge (Atlas)
- Any ability to start/stop services

### Repository

```
github.com/Harshmaury/Gate
~/workspace/projects/engx/services/gate
```

### Structure

```
gate/
  cmd/gate/main.go                  — startup, OAuth flow, token server
  internal/
    auth/
      github.go                     — GitHub OAuth flow
      jwt.go                        — JWT issuance and validation
      token.go                      — service token generation (team-scoped)
    team/
      store.go                      — team CRUD (PostgreSQL via Neon)
      model.go                      — Team, Member, Invitation types
    invite/
      generate.go                   — invite link generation
      redeem.go                     — invite redemption
    api/
      handler/
        auth.go                     — GET /auth/github, GET /auth/callback
        teams.go                    — CRUD /teams/:id
        members.go                  — POST /teams/:id/members
        tokens.go                   — POST /tokens (issue team service token)
      server.go                     — routes, middleware
      middleware/
        jwt.go                      — JWT validation middleware
  nexus.yaml
  SERVICE-CONTRACT.md
  go.mod                            — module github.com/Harshmaury/Gate
```

### Port assignment

```
Gate HTTP API:  :9092  (public, behind Cloudflare)
```

### How team sync works

When a developer joins a team:

```
1. engx team join <url> --token <invite>
     ↓
2. Gate validates invite, issues team service token
3. engx stores team token in ~/.nexus/team-token
     ↓
4. engxd starts syncing to Team Nexus every 5 seconds:
   PUT /team/sync { projects, service_states }
     ↓
5. Team Nexus broadcasts sync to all team members via SSE
     ↓
6. Alice's engx ps shows both her services and yours
```

### Team Nexus mode

Nexus itself gains a `--team` flag. When set, Nexus operates in
team-sync mode in addition to its normal local mode:

```bash
engxd --team https://nexus.engx.dev/teams/my-team --token <team-token>
```

This is a **configuration change to Nexus**, not a new service.
The team registry is a separate deployed Nexus instance on the server.

Team Nexus instance is the same binary (`engxd`) deployed on the Hetzner
server with a PostgreSQL backend instead of SQLite.

### Accord changes

```go
// TeamDTO
type TeamDTO struct {
    ID      string `json:"id"`
    Name    string `json:"name"`
    Members []string `json:"members"` // GitHub usernames
}

// TeamSyncRequest — body for PUT /team/sync
type TeamSyncRequest struct {
    MachineID string        `json:"machine_id"`
    Projects  []ProjectDTO  `json:"projects"`
    Services  []ServiceDTO  `json:"services"`
}

// InviteDTO
type InviteDTO struct {
    URL       string `json:"url"`
    ExpiresAt string `json:"expires_at"`
}
```

### Infrastructure additions (Phase 2)

```
PostgreSQL:   Neon free tier (5 GB, serverless) — $0
Gate server:  Same Hetzner VPS as Relay (separate process)
Team Nexus:   Same VPS, separate engxd instance, port 8090
```

### ADRs required

```
ADR-045   Gate service — GitHub OAuth + team identity
ADR-046   Team Nexus — multi-user shared registry mode
ADR-047   Local↔Team sync protocol — push interval, conflict rules
ADR-048   Team-scoped subdomain prefixes (extends ADR-044)
```

---

## 6. Conduit — Phase 3

### Purpose

Conduit routes commands from one machine's Forge to another machine's
engxd, and streams events back. It is the execution routing layer
that enables `engx run --on`.

```bash
engx run api --on alice-machine
engx build api --on ci-server
engx platform start --on staging-vps
```

### Capability domain

**Execution** — Conduit translates developer intent (run this on that machine)
into coordinated actions across machines. This is exactly the Execution
domain's responsibility. Conduit is the distributed extension of Forge's
execution pipeline.

### What Conduit owns

- Command routing: source machine → target machine
- Bidirectional event streaming: target machine → source CLI
- Machine identity resolution: name → engxd connection
- Execution receipts: command ID, status, target machine

### What Conduit does NOT own

- Command execution (Forge on the target machine owns this)
- Service state (Nexus on the target machine owns this)
- User identity (Gate owns this)
- Tunnel connections (Relay owns this)

### Repository

```
github.com/Harshmaury/Conduit
~/workspace/projects/engx/services/conduit
```

### Structure

```
conduit/
  cmd/conduit/main.go               — startup, connection registry
  internal/
    router/
      registry.go                   — machine ID → engxd connection
      dispatch.go                   — route command to target machine
    stream/
      relay.go                      — bidirectional command/event streaming
      subscription.go               — SSE fan-out to waiting CLI clients
    auth/
      token.go                      — team token validation (calls Gate)
    receipt/
      store.go                      — execution receipt persistence
      model.go                      — CommandReceipt type
    api/
      handler/
        dispatch.go                 — POST /dispatch
        stream.go                   — GET /stream/:receipt_id (SSE)
        machines.go                 — GET /machines (registered machines)
      server.go
  nexus.yaml
  SERVICE-CONTRACT.md
  go.mod                            — module github.com/Harshmaury/Conduit
```

### Port assignment

```
Conduit HTTP API:  :9093  (internal, accessible only to team members)
```

### How remote execution works

```
1. engx run api --on alice-machine
     ↓
2. engx CLI sends POST /dispatch to Conduit:
   { command: "project_start", target: "api", machine: "alice-machine" }
     ↓
3. Conduit validates team token via Gate
4. Conduit looks up "alice-machine" in registry → engxd connection
5. Conduit forwards CmdProjectStart to alice's engxd via persistent connection
     ↓
6. Alice's engxd executes command locally
7. Alice's engxd streams events back through the Conduit connection
     ↓
8. Conduit fans out events to the waiting CLI via SSE
9. Developer's terminal shows real-time output from alice's machine
```

### Forge changes (in Forge repo)

Forge gains an optional `--remote` flag in its execution context:

```go
// internal/executor/context.go
type ExecutionContext struct {
    // ... existing fields ...
    RemoteMachine string  // "" = local, "alice-machine" = remote via Conduit
    ConduitAddr   string  // from config, only used when RemoteMachine != ""
}
```

When `RemoteMachine` is set, Forge routes the command through Conduit
instead of calling Nexus directly. This is a ~50-line change in
`internal/executor/intent/runner.go`.

### Accord changes

```go
// DispatchRequest — body for POST /dispatch
type DispatchRequest struct {
    Command   string            `json:"command"`    // "project_start" | "project_stop" | "build"
    ProjectID string            `json:"project_id"`
    Machine   string            `json:"machine"`    // target machine ID
    Params    map[string]string `json:"params,omitempty"`
}

// CommandReceipt — response from POST /dispatch
type CommandReceipt struct {
    ID        string `json:"id"`        // stream ID for SSE subscription
    Machine   string `json:"machine"`
    ProjectID string `json:"project_id"`
    Status    string `json:"status"`    // "dispatched" | "executing" | "complete" | "failed"
    StreamURL string `json:"stream_url"` // GET /stream/:id
}
```

### ADRs required

```
ADR-049   Conduit service — remote command routing architecture
ADR-050   Forge remote execution target — ExecutionContext extension
ADR-051   Remote event streaming — SSE fan-out pattern for multi-machine
```

---

## 7. Inter-service communication rules for v3

These rules are absolute. Every v3 service must follow them.

### Rule 1 — Herald only, no raw HTTP

All v3 service-to-v2-service calls use Herald.
No raw `http.Get` or `http.Post` in any v3 service.

```go
// WRONG
resp, err := http.Get("http://127.0.0.1:8080/projects")

// CORRECT
client := herald.New(canon.DefaultNexusAddr, herald.WithToken(token))
projects, err := client.Projects().List(ctx)
```

### Rule 2 — Accord types only, no local DTOs

All request/response types that cross service boundaries use Accord.
No locally-defined anonymous structs for API shapes.

```go
// WRONG
var result struct {
    ID   string `json:"id"`
    Name string `json:"name"`
}

// CORRECT
var result accord.ProjectDTO
```

### Rule 3 — Canon constants only, no string literals

All header names, event type strings, and service addresses use Canon.

```go
// WRONG
req.Header.Set("X-Service-Token", token)

// CORRECT
req.Header.Set(canon.ServiceTokenHeader, token)
```

### Rule 4 — No v3 service writes to v2 service state

Relay, Gate, and Conduit are read-only consumers of v2 service state.

| v3 Service | May call | May NOT call |
|------------|----------|--------------|
| Relay | GET /projects, GET /services (Nexus) | POST /projects/:id/start, POST /projects/:id/stop |
| Gate | GET /projects (Nexus) | Any write endpoint on any service |
| Conduit | GET /projects, GET /services (Nexus) | POST start/stop directly — routes through Forge only |

### Rule 5 — Conduit routes through Forge, not Nexus

Conduit never calls Nexus start/stop directly.
It sends commands to the target machine's Forge via the
established connection. Forge on the target machine calls Nexus.
The execution chain is preserved: CLI → Forge → Nexus, even remotely.

### Rule 6 — Gate is the only identity authority

Relay and Conduit validate team tokens by calling Gate.
Neither Relay nor Conduit maintains its own user database or token store.

```go
// In Relay auth validation
ok, err := gateClient.ValidateToken(ctx, token)

// In Conduit auth validation
membership, err := gateClient.GetTeamMembership(ctx, token, teamID)
```

### Rule 7 — All v3 services bind to 127.0.0.1 locally

When running on a developer's machine, v3 services bind to 127.0.0.1.
Only the server-side deployments (relay.engx.dev etc.) bind to 0.0.0.0.
This matches ADR-003.

---

## 8. Service policy — what every v3 service must implement

Every new v3 service must implement all of the following before its
first tag. No exceptions.

### 8.1 File header (every .go file)

```go
// @relay-project: relay
// @relay-path: internal/tunnel/registry.go
// Package tunnel manages active TLS tunnel connections.
package tunnel
```

### 8.2 Health endpoint

```
GET /health → { "ok": true, "status": "healthy", "service": "<name>" }
No auth required. Always first route registered.
```

### 8.3 Response envelope

All endpoints use Accord `Response[T]`:

```go
respondOK(w, data)       // { ok: true, data: ... }
respondErr(w, status, err) // { ok: false, error: "..." }
```

### 8.4 Structured logging

```go
logger.Printf("INFO: tunnel registered subdomain=%s owner=%s", sub, owner)
logger.Printf("WARNING: tunnel disconnected subdomain=%s reason=%v", sub, err)
logger.Printf("ERROR: relay routing failed subdomain=%s: %v", sub, err)
```

Level prefixes: `INFO:`, `WARNING:`, `ERROR:` — always present.

### 8.5 Graceful shutdown

```go
// In every cmd/<service>/main.go
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit
logger.Printf("INFO: shutting down")
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
server.Shutdown(ctx)
```

### 8.6 Context propagation

Every outbound HTTP call passes context and sets X-Trace-ID:

```go
req, err := http.NewRequestWithContext(ctx, method, url, body)
req.Header.Set(canon.TraceIDHeader, traceID)
```

### 8.7 Per-cycle trace IDs (for polling services)

If a service polls upstream services (like Gate polling Nexus):

```go
traceID := fmt.Sprintf("relay-%x", rand8bytes())
// pass traceID to all calls in this cycle
```

### 8.8 Config from environment only

```go
// internal/config/env.go — required in every v3 service
func EnvOrDefault(key, def string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return def
}

func ExpandHome(path string) string {
    if strings.HasPrefix(path, "~/") {
        return filepath.Join(os.Getenv("HOME"), path[2:])
    }
    return path
}
```

Never use `os.Getenv("HOME")` directly outside of `ExpandHome`.

### 8.9 Token comparison (constant time)

```go
// ALWAYS — timing attack prevention
if subtle.ConstantTimeCompare([]byte(got), []byte(expected)) != 1 {
    respondErr(w, http.StatusUnauthorized, errors.New("unauthorized"))
    return
}
```

### 8.10 nexus.yaml

Every v3 service has a `nexus.yaml` at repo root:

```yaml
name: relay
id: relay
type: platform-daemon
language: go
version: 0.1.0
keywords: [tunnel, proxy, subdomain, engx]
capabilities:
  - tunnel-registry
  - subdomain-routing
  - tls-termination
depends_on:
  - nexus   # reads project/service state for token validation
runtime:
  provider: process
  command: go
  args: [run, ./cmd/relay/]
  port: 9090
```

### 8.11 SERVICE-CONTRACT.md

Every v3 service has a `SERVICE-CONTRACT.md` that states:
- Role (one sentence)
- What it owns
- What it explicitly does NOT own
- Inputs (which endpoints it calls on other services)
- Outputs (which endpoints it exposes)
- Guarantees (what callers can depend on)

### 8.12 Function size limit

Max 40 lines per function. No exceptions.
Split into named helpers at every logical boundary.

### 8.13 No package-level mutable state

All dependencies injected at construction time.
No global variables that hold mutable state.

### 8.14 Table-driven tests for every new component

```go
tests := []struct {
    name    string
    input   X
    want    Y
    wantErr bool
}{...}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) { ... })
}
```

### 8.15 WORKFLOW-SESSION.md in every ZIP

Every delivery ZIP contains an updated WORKFLOW-SESSION.md.

---

## 9. Delivery workflow for v3 services

Same workflow as all engx services. No exceptions.

```
ADR committed → Code → Tests → WORKFLOW-SESSION.md → ZIP → Drop → Apply → Build → Commit
```

### ZIP naming

```
relay-phase1-tunnel-registry-20260322-0900.zip
gate-phase1-oauth-team-20260401-1400.zip
conduit-phase1-command-routing-20260415-1000.zip
```

### Apply command template

```bash
# New service first delivery
mkdir -p ~/workspace/projects/engx/services/<service>
cd ~/workspace/projects/engx/services/<service> && \
unzip -o /mnt/c/Users/harsh/Downloads/engx-drop/<ZIP>.zip -d . && \
go mod tidy && go build ./...

# Subsequent deliveries
cd ~/workspace/projects/engx/services/<service> && \
unzip -o /mnt/c/Users/harsh/Downloads/engx-drop/<ZIP>.zip -d . && \
go build ./...
```

### Commit format

```
feat(phase1): tunnel registry + subdomain assignment (ADR-042)
feat(phase1): GitHub OAuth flow + JWT issuance (ADR-045)
feat(phase1): command dispatch + SSE streaming (ADR-049)
```

---

## 10. ADR sequence

All of the following require committed ADRs before any code is written.
Numbers are provisional — next available is ADR-041.

### Phase 1 — Relay

| ADR | Title | Must precede |
|-----|-------|-------------|
| ADR-041 | `engx expose` — public endpoint tunneling | All Phase 1 code |
| ADR-042 | Relay service — TLS tunnel multiplexer | Relay repo creation |
| ADR-043 | engxa tunnel mode — new connection type | engxa changes |
| ADR-044 | `*.engx.dev` subdomain system | DNS + SSL setup |

### Phase 2 — Gate

| ADR | Title | Must precede |
|-----|-------|-------------|
| ADR-045 | Gate service — GitHub OAuth + team identity | Gate repo creation |
| ADR-046 | Team Nexus — multi-user shared registry mode | Team sync code |
| ADR-047 | Local↔Team sync protocol | engxd team flag |
| ADR-048 | Team-scoped subdomain prefixes | Phase 2 expose extension |

### Phase 3 — Conduit

| ADR | Title | Must precede |
|-----|-------|-------------|
| ADR-049 | Conduit service — remote command routing | Conduit repo creation |
| ADR-050 | Forge remote execution target | Forge changes |
| ADR-051 | Remote event streaming — multi-machine SSE | Conduit streaming |
| ADR-052 | Billing layer — tier enforcement | Any paid feature gate |

---

## 11. Capability ownership matrix (v2 + v3)

| Capability | Owner | v3 Note |
|---|---|---|
| Project registry | Nexus | Unchanged |
| Service state | Nexus | Unchanged |
| Runtime reconciliation | Nexus | Unchanged |
| Filesystem observation | Nexus | Unchanged |
| Event log | Nexus | Unchanged |
| Workspace knowledge | Atlas | Unchanged |
| Command execution | Forge | Gains remote target in Phase 3 |
| Policy findings | Guardian | Unchanged |
| AI insights | Sentinel | Unchanged |
| Platform metrics | Metrics | Unchanged |
| Trace assembly | Observer | Unchanged |
| Workspace topology | Navigator | Unchanged |
| Shared type constants | Canon | Gains relay/team constants |
| API contract types | Accord | Gains tunnel/team/dispatch DTOs |
| Nexus HTTP client | Herald | Gains TunnelsClient, GateClient |
| **Tunnel registry** | **Relay** | New — Phase 1 |
| **Subdomain routing** | **Relay** | New — Phase 1 |
| **User identity** | **Gate** | New — Phase 2 |
| **Team membership** | **Gate** | New — Phase 2 |
| **Token issuance** | **Gate** | New — Phase 2 |
| **Remote command routing** | **Conduit** | New — Phase 3 |
| **Multi-machine event streaming** | **Conduit** | New — Phase 3 |

---

## 12. Infrastructure summary

| Phase | Component | Provider | Cost |
|-------|-----------|----------|------|
| 1 | VPS (Relay + future services) | Hetzner CX22 | ~$4/month |
| 1 | Wildcard DNS `*.engx.dev` | Cloudflare | Free |
| 1 | Wildcard SSL cert | Let's Encrypt | Free |
| 1 | Domain `engx.dev` | Namecheap/Cloudflare | ~$12/year |
| 2 | PostgreSQL (Gate + Team Nexus) | Neon free tier | Free |
| 2 | Gate process (same VPS) | Hetzner | $0 additional |
| 2 | Team Nexus process (same VPS) | Hetzner | $0 additional |
| 3 | Conduit process (same VPS) | Hetzner | $0 additional |

Total recurring cost at full v3: ~$5/month + $12/year.
Break-even: 1 Pro subscriber at $9/month.

---

## 13. First step

Write ADR-041.

Not the code. Not the repo. Not the DNS. The ADR first.

ADR-041 must answer:
- What does `engx expose` do from the user's perspective
- What does it NOT do (not persistent by default, not a VPN, not a firewall bypass)
- Which domain owns the expose capability (Control — Relay)
- What alternatives were considered (ngrok wrapper, Cloudflare Tunnel, raw SSH -R)
- Why the custom Relay approach wins (no vendor dependency, monetizable, engx-native)
- What existing capabilities it builds on (engxa, Nexus service registry, Canon headers)
- Compliance with ADR-003 (HTTP/JSON), ADR-008 (service tokens), ADR-020 (observer rules)

Commit ADR-041 to `engx-governance/architecture/decisions/`.
Then and only then: create the Relay repo and start Phase 1 code.
