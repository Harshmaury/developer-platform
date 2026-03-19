# Documentation Standard

The method governing all documentation on the engx developer platform.
Every document produced for this platform must conform to this standard.

**Updated:** 2026-03-19

---

## The 4C Test

Before publishing any document, ask four questions. If any answer is no, fix it first.

| C | Question | Pass condition |
|---|----------|----------------|
| **Clear** | Does every term have exactly one meaning? | No ambiguous or locally-redefined terms |
| **Correct** | Does it match the current code and ADRs? | No stale versions, no superseded decisions |
| **Concise** | Is every sentence load-bearing? | No padding, no repetition, no preamble |
| **Current** | Was it updated at the last relevant commit? | Date stamp matches last change |

A document that fails 4C is worse than no document — it misleads.

---

## Three Levels — Quick Scan Structure

Every subject is documented at exactly the level that matches its scope.
Never mix levels in one document.

### Level 1 — Platform (30-second scan)

**What:** What the whole system does. No implementation detail.
**Who reads it:** New contributors, AI systems, users evaluating the platform.
**Length:** One screen. Never more than two pages.
**Files:** `README.md`, `AI_CONTEXT.md`, `workflow-philosophy.md`
**Rule:** If someone needs to read past the fold to understand what the platform does, it is too long.

### Level 2 — Service (5-minute scan)

**What:** What one service does, what it owns, what it does not do.
**Who reads it:** Developers working on or integrating with that service.
**Length:** One page per service. Fixed sections, fixed order.
**Files:** `SERVICE-CONTRACT.md` in each service repo
**Rule:** If a sentence describes another service's behaviour, it belongs in that service's contract.

**Required sections (in this order):**
```
1. Identity        name, port, version, module
2. Responsibilities   what this service owns — bullet list, present tense
3. Non-Responsibilities   what this service explicitly does not do
4. API Surface     endpoints — method, path, auth, one-line description
5. Outbound Calls  which services it calls and why
6. Event Topics    which events it emits or consumes
7. Config          env vars, defaults, files read at startup
8. Failure Mode    what happens when this service is unavailable
```

### Level 3 — Component (point reference)

**What:** What one function, endpoint, or type does.
**Who reads it:** Developers reading or modifying the code.
**Length:** One paragraph maximum. Usually one sentence.
**Location:** In-code comments, ADR implementation notes.
**Rule:** If it requires more than one paragraph, the component is too complex — split it.

---

## Twelve Rules

These rules govern the creation and maintenance of every document on this platform.

**Rule 1 — One definition per term.**
Every term used across multiple documents has exactly one canonical definition in `definitions/glossary.md`. Documents reference the glossary term; they never redefine it locally.

**Rule 2 — One document per scope.**
Platform-level decisions live in `developer-platform`. Service-level decisions live in the service repo. Component-level decisions live in code. Never duplicate across levels.

**Rule 3 — Date every document.**
Every document carries `Updated: YYYY-MM-DD` at the top. If a document is changed, the date changes. If the date has not changed in 30 days and the system has changed, the document is stale.

**Rule 4 — ADR-first for architecture.**
No architectural decision is implemented before its ADR is committed to `architecture/decisions/`. The ADR precedes the code, not the other way around.

**Rule 5 — Non-Responsibilities are mandatory.**
Every `SERVICE-CONTRACT.md` must have a Non-Responsibilities section. A service boundary only means something if what falls outside it is explicitly stated.

**Rule 6 — No passive voice for ownership.**
"Events are consumed by..." is wrong. "Nexus emits. Metrics consumes." is right. Ownership statements use active voice and name the owner.

**Rule 7 — No forward references without links.**
If a document references an ADR, it links or cites the ADR number. If it references a function, it names the file. Bare references ("as described elsewhere") are not permitted.

**Rule 8 — Correct before complete.**
A document with one accurate sentence is better than ten pages with one stale fact. When in doubt, remove rather than leave stale content.

**Rule 9 — The 4C test runs at every commit.**
Any commit that changes platform behaviour must also update the relevant document. A code change without a documentation update is an incomplete commit.

**Rule 10 — Service contracts do not describe other services.**
If `SERVICE-CONTRACT.md` for Forge mentions Atlas, it names only the endpoint it calls and why. It does not describe Atlas's internal behaviour. Atlas owns its own contract.

**Rule 11 — Glossary terms are platform-wide.**
Once a term is in the glossary it is used consistently everywhere. If the code uses a different name for the same concept, the code is wrong — not the glossary.

**Rule 12 — Documentation is code.**
Pull requests that add or change platform capabilities require documentation changes in the same PR. Documentation is not optional and is not a follow-up task.

---

## File Map

```
developer-platform/
  README.md                        Level 1 — platform overview
  AI_CONTEXT.md                    Level 1 — rules for AI systems (12 rules, ADR status)
  workflow-philosophy.md           Level 1 — design constraints (Ten Constraints)
  standards/
    documentation.md               this file — documentation system
  definitions/
    glossary.md                    canonical term definitions (Level 3 point reference)
  architecture/
    decisions/                     ADR-001 through current
    platform-capability-boundaries.md
    architecture-evolution-rules.md

<service-repo>/
  SERVICE-CONTRACT.md              Level 2 — service boundary document
  nexus.yaml                       Atlas capability descriptor
```

---

## Maintenance Cadence

| Trigger | Document to update |
|---|---|
| New ADR committed | `AI_CONTEXT.md` ADR table, relevant `SERVICE-CONTRACT.md` |
| New service phase | `AI_CONTEXT.md` phase table, `SERVICE-CONTRACT.md` |
| New CLI command | `README.md` commands section, `AI_CONTEXT.md` open items |
| New term introduced | `definitions/glossary.md` |
| Bug fix with behaviour change | Relevant `SERVICE-CONTRACT.md` |
| New tag released | `README.md` tag, `AI_CONTEXT.md` tag |

---

## Anti-Patterns

These appear frequently and are always wrong:

| Anti-pattern | Why wrong | Fix |
|---|---|---|
| "See the code for details" | Code is not documentation | Write the detail in Level 3 comment |
| Duplicate term definitions | Creates contradictions over time | One definition in glossary only |
| "TODO: document this" | TODOs never become documentation | Write it now or delete the feature |
| Version numbers in prose | Goes stale immediately | Use tag names and ADR numbers |
| Past tense for current state | Implies it changed | Present tense for what is true now |
| "As mentioned above" | Forces reading sequence | Repeat the key fact inline |
