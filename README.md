# engx Developer Platform

Local developer control plane. Runs on your machine. Manages your projects.

**Updated:** 2026-03-19 | **Tag:** v0.3.0-cross-service-commands

---

## 1. What It Does

```
engxd              daemon — coordinates everything
engx               CLI — how you talk to the platform
engxa              agent — starts and stops your services
```

One boot sequence:

```bash
/tmp/bin/engxd &
for proj in nexus atlas forge metrics navigator guardian observer sentinel; do
  /tmp/bin/engx register ~/workspace/projects/apps/$proj
done
/tmp/bin/engxa --id local --server http://127.0.0.1:8080 --token local-agent-token --addr 127.0.0.1:9090 &
sleep 30 && /tmp/bin/engx platform start
```

---

## 2. Ten Services

| Service   | Port | Role                        | Layer     |
|-----------|------|-----------------------------|-----------|
| Nexus     | 8080 | Registry + lifecycle        | Control   |
| Atlas     | 8081 | Workspace knowledge         | Knowledge |
| Forge     | 8082 | Build + run + deploy        | Execution |
| Metrics   | 8083 | Platform signal aggregator  | Observer  |
| Navigator | 8084 | Dependency graph            | Observer  |
| Guardian  | 8085 | Policy findings             | Observer  |
| Observer  | 8086 | Trace assembler             | Observer  |
| Sentinel  | 8087 | AI health synthesis         | Observer  |
| Canon     | —    | Shared types library        | Library   |
| ZP        | —    | Packaging tool              | Tool      |

---

## 3. Essential Commands

```bash
engx doctor                          # full platform health — start here
engx platform start                  # start all services
engx platform status                 # show running/stopped
engx build <project> --path <dir>    # build via Forge
engx check <project>                 # Atlas + Guardian for one project
engx run <project> --wait            # start + wait until running
engx logs <service-id>               # tail service log
engx services reset <id>             # clear maintenance state
```

---

## 4. Repository Map

```
developer-platform/          ← you are here
  README.md                  platform overview (this file)
  AI_CONTEXT.md              rules for AI systems
  workflow-philosophy.md     design constraints
  standards/
    documentation.md         documentation system (12 rules, 4C, 3 levels)
  definitions/
    glossary.md              canonical term definitions
  architecture/
    decisions/               ADR-001 through ADR-023
    platform-capability-boundaries.md
    architecture-evolution-rules.md
```

Each service repo contains:
```
  SERVICE-CONTRACT.md        what this service does and does not do
  nexus.yaml                 Atlas capability descriptor
  .gitignore                 standard platform gitignore
```

---

## 5. Governance

- Every new capability → ADR first, code second
- Every term → one definition in `definitions/glossary.md`
- Every service → one `SERVICE-CONTRACT.md`
- Every document → passes 4C: Clear, Correct, Concise, Current

**Full rules:** `standards/documentation.md`
