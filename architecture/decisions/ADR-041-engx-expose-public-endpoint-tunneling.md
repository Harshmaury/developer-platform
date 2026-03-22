# ADR-041 — engx expose: Public Endpoint Tunneling

Date: 2026-03-22
Status: Accepted
Domain: Control — runtime connectivity (Relay)
Depends on: ADR-003 (HTTP/JSON), ADR-008 (service auth), ADR-042 (Gate identity),
            ADR-044 (runtime mode), ADR-020 (observer governance)

---

## Context

Every service running on the local platform is reachable only at
`127.0.0.1:<port>`. This is correct for local development. It is a
hard barrier for everything else a developer needs to do:

- Share a running API with a colleague for review
- Receive an incoming webhook from GitHub, Stripe, or Slack
- Demonstrate a working service without deploying to a cloud environment
- Let a CI pipeline trigger a Forge build on the developer's machine
- Connect a remote team member to the local platform in Phase 2

Currently, a developer must choose between manual solutions, each with
significant friction or vendor lock-in:

**Option A — ngrok.**
Works. $0 for one tunnel. Vendor dependency. Domain changes on every
restart unless paid. Produces an `ngrok.io` URL, not an `engx.dev`
URL. Not engx-native — requires a separate CLI, separate auth, no
integration with Nexus service registry.

**Option B — Cloudflare Tunnel (cloudflared).**
Works. Free. Still a vendor dependency. Still not engx-native. Domain
is `<random>.trycloudflare.com` on the free tier. Configuration is
separate from the platform — not registered in Nexus, not tracked in
events, not observable via `engx doctor`.

**Option C — SSH reverse tunnel (`ssh -R`).**
Works for developers comfortable with SSH. Requires a VPS already
running a jump host. No subdomain management. No TLS. No engx
integration. Not viable for non-technical users or as a team feature.

**Option D — Custom Relay service.**
Requires infrastructure investment upfront (~$4/month Hetzner VPS,
`engx.dev` domain). Returns: an engx-native URL, permanent subdomains
tied to identity, integration with the Nexus service registry, full
observability via engx events and doctor, and a monetizable platform
primitive for Phase 2 team features.

The decision is between accepting ongoing vendor dependency (A or B) or
owning the infrastructure and making it a platform primitive (D).

The v3 strategy document states this directly:

> "Why the custom Relay approach wins: no vendor dependency,
> monetizable, engx-native."

This ADR defines what `engx expose` does, what it does not do, which
domain owns it, and what it builds on. It is the gate that must be
committed before any Relay code is written.

---

## Decision

### What `engx expose` does

`engx expose` makes a locally running service reachable at a stable
public HTTPS URL. One command. No configuration file.

```bash
engx expose api
# ✓ api.harsh.engx.dev → 127.0.0.1:8082
# Press Ctrl+C to stop

engx expose forge --name build-server
# ✓ build-server.harsh.engx.dev → 127.0.0.1:8082
```

The URL format is: `https://<name>.<owner>.engx.dev`

- `name` defaults to the project ID. Override with `--name`.
- `owner` is the authenticated Gate subject (`sub` claim). For local
  dev without Gate, owner defaults to the OS username.
- `engx.dev` is the platform domain. TLS terminated at Cloudflare.
  Certificate issued by Let's Encrypt.

Requests arriving at the public URL are forwarded byte-for-byte to the
local service. The tunnel is transparent — headers, methods, bodies,
and response codes are not modified.

Two new CLI commands:

```bash
engx expose <project>              # expose project's primary service
engx expose <project> --name <n>   # custom subdomain prefix
engx expose list                   # list active tunnels with URLs
engx expose stop <project>         # stop tunnel for project
```

### What `engx expose` does NOT do

**Not persistent by default.** The tunnel exists only while `engx expose`
is running. The process exiting — for any reason — closes the tunnel.
A `--persistent` flag is a Phase 2 feature, not Phase 1.

**Not a VPN.** The tunnel routes HTTP traffic for a specific service.
It does not expose the local machine's network, file system, or any
other service. It cannot be used to reach `127.0.0.1:22`.

**Not a firewall bypass.** The tunnel routes through Relay's `9090`
port via an outbound TLS connection from engxa. No inbound port on the
developer's machine is opened. The developer's firewall is unchanged.

**Not a content delivery network.** Relay does not cache, compress, or
otherwise transform responses. It is a transparent byte forwarder.

**Not anonymous.** Every inbound request carries the Gate identity of
the requester in Phase 2. In Phase 1, requests are unauthenticated but
logged with a Relay-generated request ID.

**Not multi-protocol.** Phase 1 supports HTTP and HTTPS only. WebSocket
upgrade is supported (HTTP/1.1 upgrade mechanism). gRPC, raw TCP, and
UDP are not supported in Phase 1.

### Capability domain: Control

Relay is a Control-layer service. It coordinates runtime connectivity
— which services are reachable from where, under what identity, and
via which tunnel. This is the same domain as Nexus (which coordinates
service lifecycle) and Gate (which coordinates identity).

Relay does not own service state. Nexus owns service state.
Relay does not own execution authority. Forge owns execution.
Relay does not own identity. Gate owns identity.
Relay owns exactly one thing: tunnel registry and subdomain routing.

This domain assignment is not a close call. Connectivity coordination
is a Control responsibility. Observer services (8083–8087) are
read-only and do not coordinate runtime state. Relay is not an
observer — it actively manages tunnel connections.

### What `engx expose` builds on

**engxa** — the local agent already manages a persistent connection to
engxd. In Phase 1, engxa gains a `tunnel` subcommand. When engxd
receives an expose command, it instructs engxa to open a TLS tunnel
to `relay.engx.dev:9090`. engxa already handles the connection retry
and reconnect logic — the tunnel subcommand reuses this infrastructure.

**Nexus service registry** — `engx expose api` resolves the `api`
project to its registered service (e.g., `api-daemon` at
`127.0.0.1:8082`) using the existing Nexus project and service
registry. The expose command does not require the user to know the
port. Nexus already knows it.

**Canon headers** — all tunnel protocol headers use Canon constants.
`X-Relay-Token`, `X-Engx-Subdomain`, and `X-Engx-Owner` are added to
`Canon/identity/identity.go` before any Relay code is written.

**Accord DTOs** — `TunnelDTO` and `ExposeRequest` are added to
`Accord/api/types.go` before any Relay code is written.

**Herald client** — a `TunnelsClient` is added to Herald before any
Relay code is written. All expose-related service calls go through
Herald, not raw HTTP.

**GET /system/mode (ADR-044)** — Relay calls `GET /system/mode` on
Nexus before accepting any inbound connection. If `identity` capability
is `disabled`, Relay does not forward the request and returns:
`503 Service Unavailable — identity capability disabled`. This
prevents Relay from accidentally exposing an insecure local runtime.

This probe is non-blocking in Phase 1 (Gate is not required for local
dev). The identity probe is advisory: Relay logs a warning but still
forwards if `RELAY_REQUIRE_IDENTITY=false` (default in Phase 1).
In Phase 2 this default flips.

### Compliance with existing ADRs

**ADR-003 (HTTP/JSON on 127.0.0.1):** Relay on the developer's machine
binds to `127.0.0.1:9090` for local tunnel management. The tunnel
connection to the server is outbound — no inbound port opened. The
server-side Relay binds to `0.0.0.0:9090` and `0.0.0.0:9091` — this
is an explicit exception to ADR-003 for server deployments, documented
here.

**ADR-008 (X-Service-Token):** engxa presents a Relay service token
when opening a tunnel. The token is validated by Relay using
`crypto/subtle.ConstantTimeCompare`. The token is stored in
`~/.nexus/service-tokens` under the key `relay`.

**ADR-020 (observer read-only):** Relay is not an observer. It has
write authority over its own tunnel registry. It does not call write
endpoints on Nexus, Forge, Atlas, or any observer.

**ADR-042 (Gate identity):** In Phase 1, Gate integration is optional.
engxa tunnel mode attaches the stored Gate identity token
(`~/.nexus/identity`) to the tunnel registration if present. This
allows Relay to log the owner identity per tunnel without enforcing it.
Phase 2 makes this mandatory.

---

## Relay service specification

The Relay service is a new repository: `github.com/Harshmaury/Relay`.

### Structure

```
relay/
  cmd/relay/main.go
  internal/
    tunnel/
      registry.go       — in-memory tunnel map (sync.RWMutex)
      conn.go           — TLS tunnel connection lifecycle
      multiplexer.go    — HTTP request → correct tunnel
    router/
      subdomain.go      — subdomain → tunnel lookup
      http.go           — HTTP reverse proxy through tunnel
    auth/
      token.go          — service token validation (constant-time)
    config/
      env.go            — EnvOrDefault, all config from env
  nexus.yaml
  SERVICE-CONTRACT.md
  WORKFLOW-SESSION.md
  go.mod                — module github.com/Harshmaury/Relay
```

### Ports

```
9090  — tunnel listener (engxa connects here, outbound TLS from dev machine)
9091  — HTTP router (incoming public requests, behind Cloudflare)
```

### Connection sequence

```
1. Developer: engx expose api
2. engx → engxd (unix socket): CmdExposeService{project: "api"}
3. engxd → Nexus registry: resolve api → 127.0.0.1:8082
4. engxd → engxa: open tunnel to relay.engx.dev:9090
5. engxa → Relay :9090: TLS connect, present X-Relay-Token + X-Engx-Owner
6. Relay: validate token, assign subdomain api.harsh.engx.dev
7. Relay → engxa: SubdomainConfirmed{url: "https://api.harsh.engx.dev"}
8. engxa → engxd: tunnel active
9. engxd → engx CLI: ✓ api.harsh.engx.dev → 127.0.0.1:8082

10. Inbound: GET https://api.harsh.engx.dev/users
11. Cloudflare → Relay :9091
12. Relay router: subdomain "api.harsh" → tunnel connection
13. Relay → engxa (through tunnel): forward request bytes
14. engxa → 127.0.0.1:8082: forward to local service
15. Response: same path in reverse
```

### Infrastructure

```
Server:   Hetzner CX22 — 2 vCPU, 4 GB RAM (~$4/month)
DNS:      *.engx.dev CNAME → relay server IP (Cloudflare, proxied)
SSL:      Cloudflare Universal SSL (wildcard, automatic renewal)
Domain:   engx.dev (~$12/year)
```

---

## Changes to existing repos before Relay code

These changes must be committed and tagged before the Relay repo is
created. They are the contract foundation Relay depends on.

### Canon — new constants

```go
// Canon/identity/identity.go
const (
    DefaultRelayAddr = "relay.engx.dev:9090"
    RelayTokenHeader = "X-Relay-Token"     // service token for tunnel auth
    SubdomainHeader  = "X-Engx-Subdomain"  // subdomain prefix
    OwnerHeader      = "X-Engx-Owner"      // Gate subject or OS username
)
```

### Accord — new DTOs

```go
// Accord/api/types.go
type TunnelDTO struct {
    ID          string `json:"id"`
    ProjectID   string `json:"project_id"`
    ServiceID   string `json:"service_id"`
    Subdomain   string `json:"subdomain"`
    Owner       string `json:"owner"`
    PublicURL   string `json:"public_url"`
    LocalAddr   string `json:"local_addr"`
    ActiveSince string `json:"active_since"`
}

type ExposeRequest struct {
    ServiceID string `json:"service_id"`
    Name      string `json:"name,omitempty"`
}
```

### Herald — new client

```go
// Herald/client/tunnels.go
type TunnelsClient struct{ c *Client }

func (t *TunnelsClient) Register(ctx context.Context, req accord.ExposeRequest) (*accord.TunnelDTO, error)
func (t *TunnelsClient) List(ctx context.Context) ([]accord.TunnelDTO, error)
func (t *TunnelsClient) Delete(ctx context.Context, id string) error
```

### Nexus — new CLI commands and engxa tunnel mode

```
cmd/engx/cmd_expose.go    — engx expose / list / stop commands
cmd/engxa/tunnel.go       — engxa tunnel subcommand
internal/daemon/server.go — CmdExposeService dispatch
```

---

## Compliance checklist

An implementation satisfies this ADR when:

1. `engx expose <project>` starts a tunnel and prints the public URL.
2. The URL format is `https://<name>.<owner>.engx.dev`.
3. Relay calls `GET /system/mode` before accepting inbound connections.
4. engxa uses `X-Relay-Token` (from Canon) for tunnel authentication.
5. All tunnel types are defined in Accord, not locally.
6. Herald `TunnelsClient` is used for all Relay API calls.
7. `engx expose list` shows active tunnels with public URLs.
8. Ctrl-C or `engx expose stop` closes the tunnel cleanly.
9. Canon, Accord, and Herald changes are tagged before Relay repo is created.
10. Relay service implements all 15 policies in v3-strategy.md §8.

---

## Next ADR

ADR-045 — Observer cursor persistence (restart-safe `sinceID` for all
five observer collectors). This is a v2 stabilisation ADR that runs
in parallel with Relay infrastructure setup and must be complete before
Relay Phase 1 is tagged.
