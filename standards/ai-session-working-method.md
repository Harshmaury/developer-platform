# AI Session Working Method

**Version:** 1.0
**Date:** 2026-03-23

Defines the exact protocol for every AI-assisted development session.
Follow this precisely to get consistent, correct results every time.

---

## Session Protocol

### Before writing any code

1. Upload ZIPs for every service that will be touched. No partial uploads.
2. State the objective in one sentence.
3. The AI reads the ZIPs, reconstructs the system, confirms understanding.
4. The AI never acts from memory — it must read actual files first.

### During the session

5. One fix per script. Named: `<fix_id>_<description>_YYYYMMDD.sh`
6. Drop folder: `/mnt/c/Users/harsh/Downloads/engx-drop/`
   Run with: `bash /mnt/c/Users/harsh/Downloads/engx-drop/<script>.sh`
7. Build gate mandatory: `go build ./...` before every commit. Script aborts on failure.
8. Test gate mandatory: `go test ./... -count=1` after build. Script aborts on failure.
9. Commits are atomic: one problem = one commit per repo.
10. **Paste only the summary output** — the final block printed by the script.
    Never paste full terminal scroll unless there is an unexpected error.

### After each fix

11. The AI reads the summary and moves to the next fix or writes a targeted
    diagnostic script. It never guesses — it always diagnoses first.

---

## Mandatory Script Structure

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Pre-flight — verify dirs and modules
for dir in "$DIR1" "$DIR2"; do
  [ -d "$dir" ] || { echo "ERROR: missing $dir"; exit 1; }
done

# 2. Changes — complete file writes, never patch-on-patch
cat > "$TARGET_FILE" << 'CONTENT'
...complete file content...
CONTENT

# 3. Build gate
go build ./... || exit 1

# 4. Test gate
go test ./... -count=1 || exit 1

# 5. Commit
git add <files>
git commit -m "fix(<scope>): <what> (<fix-id>)

<why it was wrong>
<what changed>"

# 6. Summary block
echo "FIX COMPLETE — NEXT: <next>"
```

---

## File Modification Rules

**Always write complete files. Never patch on patch.**

- Read exact current content first (`cat -n file` or `git show HEAD~1:path`)
- Write the complete new file as a heredoc
- Never use `sed` for multi-line insertions — it mangles newlines in WSL
- Use Python string replacement only when the anchor is a single exact line
- If "anchor not found": the file was already modified — read current state and rewrite completely

---

## Mock Store Rule

When a `Storer` interface gains new methods, ALL mock stores in ALL test files must
receive stub implementations.

```bash
# Find all mocks in package
grep -rn "type mock.*struct\|func (m \*mock" ./internal/ --include="*_test.go"
```

Add as one-liners:
```go
func (m *mockStore) NewMethod(arg Type) (ReturnType, error) { return zero, nil }
```

Never use `sed` — write the complete test file.

---

## Contract Rules (Never Violate)

| Rule | What it means |
|------|---------------|
| ADR-016 | All types, constants, header names → Canon only |
| ADR-003 | `{ok, data, error}` envelope on all HTTP responses |
| ADR-008 | `X-Service-Token` on all non-health inter-service calls |
| ADR-045 | Workspace event payloads → Canon `events/payloads.go` |
| CW-4 | `EventDTO.Payload` consumers → `accord.DecodePayload[T]` |
| CW-5 | Forge commands with explicit ID → dedup checked on `POST /commands` |
| CW-1 | Set `ENGX_AUTH_REQUIRED=true` in production |
| CW-2 | Set `RELAY_GATE_ADDR` in production; never use `RELAY_TOKEN` in prod |

---

## Analysis Protocol (For Architecture Reviews)

1. Upload ZIPs for all services
2. Request system reconstruction before analysis
3. Mandatory dimensions: Architecture Integrity, Contract Design, Execution Flow,
   Failure Modes, Observability, Concurrency & State, Identity & Security,
   Transport Layer, Scalability, Evolution Readiness
4. Required output structure:
   🧠 System Reconstruction → ⚠️ Critical Weaknesses → ⚙️ Design Improvements →
   🧱 Missing Components → 🔮 Future Evolution Path → 🧪 Failure Simulation →
   🧭 Final Verdict
5. Fix order: highest blast radius first
6. Each fix = separate script, separate commit, separate summary paste

---

## Env Var Reference (Post-Hardening)

```bash
# Generate tokens once
python3 -c "
import uuid
for s in ['atlas','forge','observer','guardian','metrics','navigator','sentinel']:
    print(s, uuid.uuid4())
" > ~/.nexus/service-tokens && chmod 600 ~/.nexus/service-tokens

# Add to ~/.bashrc
for svc in atlas forge observer guardian metrics navigator sentinel; do
  upper=$(echo $svc | tr '[:lower:]' '[:upper:]')
  export ${upper}_SERVICE_TOKEN=$(awk "/^$svc/{print \$2}" ~/.nexus/service-tokens)
done
export ENGX_AUTH_REQUIRED=true      # strict mode — FATAL on missing token
export RELAY_GATE_ADDR=http://127.0.0.1:8088  # Gate JWT for tunnel auth
```

| Variable | Effect |
|----------|--------|
| `ENGX_AUTH_REQUIRED=true` | FATAL on missing token; 503 on missing middleware |
| `*_SERVICE_TOKEN` | Auth token per service |
| `RELAY_GATE_ADDR` | Gate address for JWT tunnel validation (production) |
| `RELAY_TOKEN` | Legacy pre-shared secret (dev fallback only) |

