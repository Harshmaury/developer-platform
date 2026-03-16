# ADR-008 — Inter-Service Authentication

Date: 2026-03-16
Status: Accepted
Domain: Platform-wide

---

## Context

Atlas polls Nexus `GET /events` and `GET /projects` every 3 seconds.
Forge calls Nexus `GET /projects/:id` and Atlas `GET /workspace/project/:id`
on every workflow run. Currently none of these calls carry any authentication
token — any process on the developer's machine can impersonate Atlas or Forge
against Nexus, or impersonate Forge against Atlas.

Nexus already has a per-agent token mechanism (`X-Nexus-Token` header,
stored in the agents SQLite table, validated with
`crypto/subtle.ConstantTimeCompare`). The same pattern — static token in
header, constant-time comparison on receipt — is sufficient for
service-to-service calls on a local machine where the threat model is
process isolation, not network interception.

This ADR was identified as an open gap in AI_CONTEXT.md (2026-03-16)
and governs implementation. No implementation begins before this ADR
is accepted per architecture-evolution-rules.md Rule 1.

---

## Decision

Each platform service authenticates inbound HTTP calls from sibling
services using a pre-shared static token passed in the `X-Service-Token`
header.

### Token storage

Tokens are stored in `~/.nexus/service-tokens` — a flat file, not in
source control, not in SQLite, not in env vars.

Format (one entry per line):

    atlas  <uuid>
    forge  <uuid>

Nexus reads this file at startup. If the file does not exist, Nexus
logs a WARNING and operates in unauthenticated mode. Unauthenticated
mode is acceptable on a single-developer local machine and must be
explicitly disabled before any service is exposed beyond 127.0.0.1.

Tokens are generated once per environment:

    python3 -c "import uuid; print(uuid.uuid4())"

### Request header

Callers set:

    X-Service-Token: <token>

The header name is fixed. It must not vary between services.

### Validation — Nexus

Nexus validates inbound service calls in a middleware layer
(`internal/api/middleware/service_auth.go`), applied to all routes
except `/health`.

The middleware:
1. Reads `X-Service-Token` from the request header
2. Identifies the caller by matching the token against the loaded
   service-tokens table
3. Returns HTTP 401 if the token is absent or unrecognised
4. Uses `crypto/subtle.ConstantTimeCompare` — same as agent token validation
5. Passes through to the handler on success

The token table is loaded once at startup and held in memory. File
changes require a Nexus restart to take effect. This is intentional —
there is no hot-reload surface to exploit.

### Validation — Atlas

Atlas validates inbound Forge calls on all routes except `/health`.
Same middleware pattern as Nexus. Token is stored in
`~/.nexus/service-tokens` (same file, same format — Atlas reads the
`forge` entry).

Atlas does not receive calls from Nexus — Nexus publishes events,
Atlas polls. No inbound auth is required on the event polling path.

### Callers

| Caller | Target  | Header token source         |
|--------|---------|-----------------------------|
| Atlas  | Nexus   | `ATLAS_SERVICE_TOKEN` env   |
| Forge  | Nexus   | `FORGE_SERVICE_TOKEN` env   |
| Forge  | Atlas   | `FORGE_SERVICE_TOKEN` env   |

Callers read their outbound token from an env var. The env var value
must match the entry in `~/.nexus/service-tokens` on the receiving end.

### Exempt routes

`GET /health` is always unauthenticated on all services. Health checks
must be callable by monitoring tools and the CLI without token management.

---

## Implementation — files to create or modify

### Nexus

| File | Change |
|------|--------|
| `internal/config/service_tokens.go` | `LoadServiceTokens(path) (map[string]string, error)` — reads `~/.nexus/service-tokens`, returns `{"atlas": "<token>", "forge": "<token>"}` |
| `internal/api/middleware/service_auth.go` | `ServiceAuth(tokens map[string]string, logger) http.Handler` middleware |
| `internal/api/server.go` | `ServerConfig.ServiceTokens map[string]string`; wire middleware in `newRouter`, exempt `/health` |
| `cmd/engxd/main.go` | Load tokens at step 1 (after config), pass into `api.ServerConfig` |

### Atlas

| File | Change |
|------|--------|
| `internal/api/middleware/service_auth.go` | Same middleware (copy — Atlas has its own middleware package) |
| `internal/api/server.go` | Wire middleware, exempt `/health`; `ServerConfig.ServiceToken string` |
| `internal/nexus/client.go` | Add `X-Service-Token` header to all outbound requests |
| `cmd/atlas/main.go` | Read `ATLAS_SERVICE_TOKEN` env; pass into `ServerConfig` and `nexus.Client` |

### Forge

| File | Change |
|------|--------|
| `internal/nexus/client.go` | Add `X-Service-Token` header to all outbound requests |
| `internal/atlas/client.go` | Add `X-Service-Token` header to all outbound requests |
| `cmd/forge/main.go` | Read `FORGE_SERVICE_TOKEN` env; pass into both clients |

---

## Consequences

**Positive**
- Eliminates unauthenticated service-to-service calls
- Reuses existing token validation infrastructure — no new protocol
- Single token file — one place to rotate credentials
- Zero impact on `/health` — monitoring and CLI unaffected

**Negative**
- Unauthenticated fallback mode remains for local dev convenience
- Token rotation requires Nexus and Atlas restart
- Static tokens are not suitable beyond a single-developer machine —
  a future ADR must address this before multi-user or remote deployment

**Constraint**
This ADR must be implemented before any service binds to `0.0.0.0`
or is exposed via a reverse proxy to any network beyond localhost.

---

## Alternatives Considered

**mTLS** — rejected. Requires certificate management infrastructure
(CA, cert rotation, tooling) disproportionate to a local developer
platform. Can be adopted in a future phase for remote deployment.

**JWT** — rejected. Adds a signing key management problem and token
expiry handling. Static tokens with constant-time comparison are
sufficient for the current threat model.

**Per-endpoint API keys (different key per route)** — rejected.
Unnecessary complexity. One token per service identity is enough.

**Nexus-issued tokens (dynamic)** — rejected for Phase 1. Would require
Nexus to issue tokens to Atlas and Forge at startup, adding a bootstrap
sequencing dependency. Static pre-shared tokens start working immediately
regardless of startup order.
