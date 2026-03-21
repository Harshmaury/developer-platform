# ADR-039 — Herald Migration: Replace Internal Nexus Collectors

**Status:** Accepted  
**Date:** 2026-03-21  
**Author:** Harsh Maury  
**Scope:** Observer, Metrics, Sentinel, Guardian, Navigator  
**Depends on:** ADR-033 (Accord), ADR-034 (Herald)

---

## Context

Five observer services each maintain their own `internal/collector/nexus.go`
with identical logic:

- Raw `http.Get` calls with inline anonymous structs
- Manual Canon header injection
- No retry on connection failure
- Inline JSON struct definitions that drift from the actual Nexus API schema

When the Event schema changes (ADR-037 adds `level`, `span_id`), all five
files need updating. This is the duplication problem Accord + Herald were
created to solve.

---

## Decision

Each observer service replaces its `internal/collector/nexus.go` with
herald client calls. Migration is per-service, one at a time.

### Before (guardian/internal/collector/nexus.go — 164 lines)

```go
func (c *NexusCollector) CollectServices(ctx context.Context, traceID string) []policy.ServiceRecord {
    resp, err := c.get(ctx, "/services", traceID)
    // ... manual decode, inline struct, error handling ...
}
```

### After (guardian/internal/collector/nexus.go — ~30 lines)

```go
import "github.com/Harshmaury/Herald/client"

type NexusCollector struct {
    c *client.Client
}

func NewNexusCollector(baseURL, serviceToken string) *NexusCollector {
    return &NexusCollector{
        c: client.New(baseURL, client.WithToken(serviceToken)),
    }
}

func (c *NexusCollector) CollectServices(ctx context.Context) []policy.ServiceRecord {
    svcs, err := c.c.Services().List(ctx)
    if err != nil { return nil }
    return toServiceRecords(svcs) // accord.ServiceDTO → policy.ServiceRecord
}
```

### Migration order

1. **Guardian** — simplest collector, 3 methods
2. **Observer** — events polling with cursor
3. **Metrics** — metrics + events
4. **Navigator** — atlas client only
5. **Sentinel** — most complex, 8 fetch methods

### go.mod changes per service

Add to `require`:
```
github.com/Harshmaury/Herald v0.1.0
github.com/Harshmaury/Accord v0.1.0
```

Remove `replace github.com/Harshmaury/Nexus => ../nexus` where present
(atlas, forge) — those services import Nexus for eventbus topics only.
Herald does not replace the eventbus import.

### Acceptance criteria

- Each migrated service builds cleanly: `go build ./...`
- Collector tests pass: `go test ./internal/collector/...`
- Service starts and collects data: `engx doctor` shows correct readings
- No inline struct definitions for Nexus API types remain in collector files

---

## Compliance

| ADR | Status |
|-----|--------|
| ADR-003 | ✅ Herald uses HTTP/JSON on 127.0.0.1 — protocol unchanged |
| ADR-008 | ✅ Herald passes X-Service-Token via WithToken() option |
| ADR-016 | ✅ Herald uses Canon header constants internally |
| ADR-033 | ✅ Accord types are the wire contract — changes propagate automatically |
