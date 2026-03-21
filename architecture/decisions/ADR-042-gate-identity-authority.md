# ADR-042 — Gate: Platform Identity Authority

Date: 2026-03-21
Status: Accepted
Domain: Platform-wide (v3 foundation)

---

## Context

The platform currently has one trust primitive: `X-Service-Token`, a
pre-shared static secret that proves a caller is a known platform
component (ADR-008). This is sufficient for service-to-service calls
on a local machine. It is not sufficient for the v3 capability set.

The v3 platform introduces remote agents (Conduit), external exposure
(Relay), and team collaboration. All three require the platform to know
not just *what type* of caller is acting, but *who*, *with what
authority*, *until when*, and *on whose behalf*. Without this, the
platform cannot:

- Distinguish a developer from a CI pipeline from a remote agent
- Attribute execution records to an actor
- Authorize Relay to expose endpoints to external callers
- Allow Conduit to route remote execution with bounded trust
- Support multi-developer or multi-team workspaces
- Enforce scope-limited delegation to agents

The existing service token model (ADR-008) is not replaced by this ADR.
It remains the intra-service mesh authentication mechanism. This ADR
introduces a complementary primitive: **actor identity**, which answers
a different question than service identity.

ADR-041 (Relay) and ADR-043 (Conduit) are blocked on this ADR.
No Relay or Conduit code may be written until ADR-042 is accepted and
Gate v1.0.0 is tagged.

---

## Decision

Introduce **Gate** as the sole identity authority for the platform.

Gate issues and validates cryptographically signed, self-contained
identity tokens. Services validate tokens locally using Gate's
distributed public key — no network call to Gate is required per
request. Gate going down does not interrupt authenticated operations
already in flight.

### Token model

**Algorithm:** Ed25519 (RFC 8037). Chosen over RS256/ES256 for:
- Smaller key and signature size (32-byte public key, 64-byte signature)
- Faster verification (critical for per-request validation)
- No parameter confusion attacks (unlike RSA/ECDSA variants)
- Native Go support (`crypto/ed25519`, no external dependency)

**Format:** JSON Web Token (JWT, RFC 7519) with the following claims:

```json
{
  "iss": "gate",
  "sub": "harsh@github",
  "jti": "<uuid>",
  "iat": 1742567400,
  "exp": 1742653800,
  "scp": ["execute", "observe", "register"]
}
```

| Claim | Type     | Meaning |
|-------|----------|---------|
| `iss` | string   | Always `"gate"` — rejects tokens from any other issuer |
| `sub` | string   | Actor identity — GitHub login, agent ID, or CI pipeline ID |
| `jti` | string   | UUID — unique token ID, used for revocation |
| `iat` | int64    | Issued-at Unix timestamp |
| `exp` | int64    | Expiry Unix timestamp |
| `scp` | []string | Scopes granted to this token (see scope model below) |

**Token lifetime defaults:**

| Actor type   | Default TTL | Rationale |
|--------------|-------------|-----------|
| Developer    | 24h         | Interactive session — long enough to be practical |
| Agent        | 1h          | Short-lived, renewable — principle of least privilege |
| CI pipeline  | 15m         | Per-run token — expires with the job |

TTLs are configurable via Gate environment variables. Defaults are
conservative. They may be increased by explicit configuration, never
silently.

### Scope model

Scopes are coarse-grained and additive. A token carries zero or more
scopes. A service checks whether the required scope is present.

| Scope       | Permits |
|-------------|---------|
| `execute`   | Forge command submission, project start/stop |
| `observe`   | Read access to events, metrics, history, topology |
| `register`  | Project and service registration (engx register, engx init) |
| `admin`     | Token revocation, Gate key rotation, Guardian rule management |

Scope absence is a hard denial — not a degraded response. A token
without `execute` that calls `POST /forge/run` receives HTTP 403 with
`ErrorCode: ErrForbidden` (new Accord constant, see below).

Scope inheritance is prohibited. `admin` does not imply `execute`.
Each scope is independent. Callers must request all scopes they need.

### Key distribution

Gate generates an Ed25519 keypair at first start. The private key
never leaves Gate. The public key is served at:

    GET /gate/public-key     → {"key": "<base64-encoded DER>", "alg": "Ed25519"}

This endpoint requires no authentication. It is called by every
service at startup to cache the public key. Services re-fetch the
public key if signature verification fails with `ErrInvalidSignature`
— this handles key rotation without restart.

Key rotation: Gate generates a new keypair on explicit admin command
(`engx gate rotate-key`). The old key remains valid for a grace period
equal to the longest active token TTL (24h by default). During the
grace period, Gate accepts tokens signed by either key.

### Token revocation

Gate maintains a revocation list in `~/.nexus/gate.db` (SQLite).
Revoked `jti` values are stored with their original expiry.

Services check revocation by calling:

    POST /gate/validate    Body: {"token": "<jwt>"}
    → {"valid": true, "claim": {...}}  or  {"valid": false, "reason": "..."}

This is the **only** network call in the validation path, and it is
**optional** — services may perform local signature + expiry validation
only (fail-open on revocation). The local path is always correct for
expiry. The network path adds revocation certainty.

**Default validation mode per service:**

| Service  | Validation mode | Rationale |
|----------|-----------------|-----------|
| Nexus    | Local + network | Write operations — revocation matters |
| Forge    | Local + network | Execution — revocation matters |
| Relay    | Local + network | External boundary — revocation mandatory |
| Atlas    | Local only      | Read-only — expiry sufficient |
| Observers| Local only      | Read-only — expiry sufficient |
| Conduit  | Local + network | Remote execution — revocation mandatory |

### Header

The identity token travels in a new header:

    X-Identity-Token: <jwt>

This header is defined in Canon as `identity.IdentityTokenHeader`.
Never hardcoded. Never confused with `X-Service-Token`.

Both headers may be present on a request:
- `X-Service-Token` — proves this is a known platform service (ADR-008)
- `X-Identity-Token` — proves who is acting through this service

They are independent trust dimensions. A request may carry one, both,
or neither. Missing `X-Identity-Token` is not a validation failure —
it means the actor is anonymous. What services do with anonymous actors
is a policy decision (see Guardian G-009, ADR-045).

### Identity provider — GitHub OAuth (v3.0)

Gate authenticates developers via GitHub OAuth 2.0:

1. `GET /gate/auth/github` → redirects to GitHub OAuth consent
2. GitHub redirects to `GET /gate/auth/github/callback?code=...`
3. Gate exchanges code for GitHub user profile
4. Gate issues a platform token with `sub = "<github_login>"`
5. Token returned to caller as JSON: `{"token": "<jwt>", "exp": ...}`

The `engx` CLI gains `engx login` which opens the OAuth flow in the
system browser and stores the returned token in `~/.nexus/identity`.

No username/password. No user database. Gate stores only: issued token
metadata (jti, sub, exp) for revocation tracking. It does not store
GitHub tokens, emails, or profile data beyond the login name.

### Agent identity

Agents (`engxa`) receive tokens from Gate, not from Nexus. Agent
registration flow:

1. `engxa` calls `POST /gate/tokens/agent` with its registered agent ID
   and the platform service token (ADR-008 — proves it is a known agent)
2. Gate issues a token with `sub = "agent:<agent-id>"`, `scp: ["execute"]`
3. `engxa` includes this token as `X-Identity-Token` on all requests to
   Nexus and Forge

This means every remote execution carries an attributable actor.
Guardian G-009 enforces this.

---

## Implementation — new service: Gate

**Binary:** `gate`
**Port:** `:8088`
**DB:** `~/.nexus/gate.db`
**Registered in:** Nexus (via `nexus.yaml`, standard ADR-022 flow)

### File structure

```
gate/
  cmd/gate/main.go
  internal/
    api/
      server.go
      handler/
        auth.go         — GET /gate/auth/github, callback
        tokens.go       — POST /gate/tokens/agent, POST /gate/tokens/ci
        validate.go     — POST /gate/validate
        publickey.go    — GET /gate/public-key
        revoke.go       — POST /gate/revoke (admin scope required)
    config/
      env.go
    identity/
      keypair.go        — Ed25519 keygen, load, rotate
      token.go          — issue, parse, verify (local)
      revocation.go     — revocation list (SQLite)
    provider/
      github.go         — GitHub OAuth client
    store/
      storer.go         — interface
      sqlite.go         — gate.db schema + queries
  nexus.yaml
  go.mod
```

### go.mod dependencies

```
github.com/Harshmaury/Canon
github.com/Harshmaury/Accord
github.com/golang-jwt/jwt/v5    — JWT parse/sign
github.com/mattn/go-sqlite3     — revocation store (same as Nexus, Forge)
```

No framework. Standard `net/http`. Same pattern as every other service.

### Accord additions (coordinate with Accord repo)

New types in `api/upstream.go`:

```go
// IdentityClaimDTO is the validated actor identity returned by Gate.
// Returned by POST /gate/validate and embedded in execution records.
type IdentityClaimDTO struct {
    Subject   string   `json:"sub"`
    Scopes    []string `json:"scp"`
    ExpiresAt int64    `json:"exp"`
    TokenID   string   `json:"jti"`
}

// GateValidateRequest is the body for POST /gate/validate.
type GateValidateRequest struct {
    Token string `json:"token"`
}

// GateValidateResponse is the response body for POST /gate/validate.
type GateValidateResponse struct {
    Valid  bool             `json:"valid"`
    Claim  *IdentityClaimDTO `json:"claim,omitempty"`
    Reason string           `json:"reason,omitempty"`
}

// GatePublicKeyDTO is the response body for GET /gate/public-key.
type GatePublicKeyDTO struct {
    Key string `json:"key"` // base64-encoded DER
    Alg string `json:"alg"` // always "Ed25519"
}
```

New error code in `api/types.go`:

```go
ErrForbidden ErrorCode = "FORBIDDEN" // 403 — valid identity, insufficient scope
```

### Canon additions (coordinate with Canon repo)

In `identity/identity.go`:

```go
// IdentityTokenHeader carries the Gate-issued actor identity token.
// Present on requests from human actors, agents, and CI pipelines.
// Distinct from ServiceTokenHeader — these are independent trust dimensions.
const IdentityTokenHeader = "X-Identity-Token"

// GateService is the canonical service name for Gate.
const GateService = "gate"

// DefaultGateAddr is the default Gate service address.
const DefaultGateAddr = "http://127.0.0.1:8088"
```

Scope constants in a new file `identity/scopes.go`:

```go
// Scope constants for Gate-issued tokens.
// Import from Canon — never hardcode scope strings in any service.
const (
    ScopeExecute  = "execute"
    ScopeObserve  = "observe"
    ScopeRegister = "register"
    ScopeAdmin    = "admin"
)
```

### Herald additions (coordinate with Herald repo)

New file `client/gate.go`:

```go
// GateClient provides typed access to the Gate identity API.
type GateClient struct{ c *Client }

func (g *GateClient) Validate(ctx context.Context, token string) (*accord.IdentityClaimDTO, error)
func (g *GateClient) PublicKey(ctx context.Context) (*accord.GatePublicKeyDTO, error)
func (g *GateClient) RevokeToken(ctx context.Context, jti string) error
```

`client.go` gains:

```go
func (c *Client) Gate() *GateClient { return &GateClient{c} }
```

### Platform integration points

These changes are additive (omitempty) and do not break existing behaviour:

**Nexus `internal/api/handler/events.go`:**
`EventDTO.Actor string json:"actor,omitempty"` — populated when
`X-Identity-Token` is present and valid on the originating request.

**Forge `internal/store/storer.go`:**
`ExecutionRecord.ActorSub string json:"actor_sub,omitempty"`
`ExecutionRecord.ActorScope []string json:"actor_scope,omitempty"`

**Forge `internal/api/handler/run.go`:**
Extract and validate `X-Identity-Token` before accepting a command.
If valid, attach claim to the execution record. If absent, record
`actor_sub = ""` — anonymous execution (Guardian G-009 handles policy).

---

## Startup integration

Gate starts as part of `engx platform start`:

```bash
engx platform start
# starts: atlas, forge, metrics, navigator, guardian, observer, sentinel, gate
```

`engx login` is available immediately after Gate starts:

```bash
engx login
# opens GitHub OAuth in system browser
# stores token in ~/.nexus/identity
# prints: ✓ logged in as harsh@github (expires in 24h)
```

All subsequent `engx` commands attach the stored token as
`X-Identity-Token` automatically if present. No token = anonymous.
Anonymous is valid — the system degrades gracefully, not refuses.

---

## Consequences

**Positive**
- Every execution is attributable to an actor — Guardian and Forge
  gain audit capability that was architecturally impossible before
- Relay and Conduit have a real trust foundation — not a workaround
- Remote agents carry bounded, expiring, revocable authority
- Token validation requires no network call in the common case —
  Gate going down does not interrupt operations
- Multi-developer workspaces become possible — each developer has
  a distinct `sub`
- The service token (ADR-008) is not disrupted — it remains the
  intra-service mesh primitive

**Negative**
- Gate is a new startup dependency — `engx platform start` is slightly
  more complex
- Key rotation requires a 24h grace period during which both keys
  must be accepted — operational complexity increase
- GitHub OAuth requires outbound network — local-only environments
  must configure an alternate provider (future ADR)
- JWT library adds one external dependency to Gate

**Constraints**
- Gate's private key never leaves `~/.nexus/gate.db`
- Gate never stores GitHub OAuth tokens, emails, or profile data
  beyond the login name used as `sub`
- `X-Identity-Token` is never required on `/health` — always exempt
- Services never implement their own token validation logic —
  local Ed25519 verification uses the shared Gate public key only
- ADR-041 (Relay) may not be drafted until this ADR is accepted
- ADR-043 (Conduit) may not be drafted until Gate v1.0.0 is tagged

---

## Alternatives Considered

**Opaque tokens + online validation only** — rejected. Gate becoming
unavailable would deny all authenticated requests. Unacceptable for a
platform where Forge may be executing long-running jobs. Self-contained
tokens with local verification eliminate this failure mode entirely.

**PASETO instead of JWT** — considered. PASETO (Platform-Agnostic
Security Tokens) has better defaults and no algorithm confusion attacks.
Rejected for v3.0 because Go JWT ecosystem (`golang-jwt/jwt`) is more
mature, Herald consumers are all Go, and Ed25519 with explicit algorithm
pinning closes the algorithm confusion vector. PASETO remains the
correct upgrade path if the token model is ever extended beyond Go.

**OAuth2 + OIDC full implementation** — rejected for v3.0. OIDC adds
discovery endpoints, JWKS rotation, claims mapping, and userinfo
endpoints. This is the correct long-term model but introduces a surface
area disproportionate to the current scale. Gate v1 implements the
minimum viable identity primitive. OIDC compliance is a future ADR.

**Shared symmetric signing (HMAC-SHA256)** — rejected. Symmetric keys
must be distributed to every verifying service, creating a secret
management problem identical to the current service token problem.
Ed25519 asymmetric signing means only Gate holds a secret.

**mTLS for service identity** — noted but out of scope. mTLS addresses
service-to-service identity. This ADR addresses actor identity. The two
are complementary, not alternatives.

---

## Versioning and rollout

Gate v1.0.0 ships when:
- `go build ./...` passes
- `go test ./...` passes — keypair, token issue/verify, revocation,
  scope checking, GitHub OAuth mock
- `engx login` completes a full OAuth round-trip in local dev
- `engx platform start` starts Gate alongside other services
- `engx doctor` reports Gate health
- Forge records `actor_sub` on executions with a valid identity token
- Guardian G-009 finding appears for anonymous executions when
  `GUARDIAN_REQUIRE_IDENTITY=true` (opt-in — not default in v3.0)

Other services (Relay, Conduit) are not blocked on the opt-in Guardian
rule — they are blocked on Gate being tagged and stable.
