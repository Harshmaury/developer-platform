# ADR-026 — engxd System Service Install (Phase 18)

**Status:** Accepted
**Date:** 2026-03-19
**Author:** Harsh Maury
**Scope:** nexus (engxd daemon, engx CLI)

---

## Context

`engxd` runs as a background process started manually. On workstation reboot
or session login, the developer must manually restart it before any `engx`
commands work. This creates friction for everyday use and makes the platform
feel unreliable when the daemon is not running.

Registering `engxd` as a system-level user service solves this — it starts
automatically at login and restarts on crash without operator intervention.

---

## Decision

Add `engx platform install` and `engx platform uninstall` commands that
register and deregister `engxd` as a persistent user service.

### Platform targets

| OS    | Mechanism | File location |
|-------|-----------|---------------|
| macOS | launchd user agent | `~/Library/LaunchAgents/dev.engx.engxd.plist` |
| Linux | systemd user service | `~/.config/systemd/user/engxd.service` |

**No root required.** Both use per-user service managers.

### Commands

```bash
engx platform install      # register engxd as user service, enable autostart
engx platform uninstall    # remove registration, stop service
engx platform log          # tail the system service log
```

### Log path (system service)

```
~/.nexus/logs/engxd.log
```

Both the launchd plist and systemd unit file redirect stdout/stderr to this
path. Consistent with the existing log directory established in phase 19/20.

### Rules

1. `engx platform install` is idempotent — re-running when already installed
   reloads the service definition rather than erroring.
2. `engx platform uninstall` stops the running service before removing the
   registration file.
3. If the OS is neither macOS nor Linux, the command exits with a clear
   unsupported-platform message.
4. The engxd binary path written into the service file is resolved at install
   time via `os.Executable()` — never hardcoded.
5. `engx platform log` tails the log file directly — it does not call any
   system service API.

---

## Compliance

| ADR     | Status |
|---------|--------|
| ADR-003 | ✅ No new inter-service HTTP calls |
| ADR-008 | ✅ Service auth unchanged |
| ADR-025 | ✅ `platform` subcommand follows the cobra command pattern from ADR-025 |

---

## Next ADR

ADR-027 — Forge scheduled cron triggers
