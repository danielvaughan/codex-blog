---
title: "Daemon Auto-Start Is Now the Stable Default in Codex CLI v0.157.0"
parent: "Articles"
nav_order: 1167
date: 2026-09-25T13:00:00+00:00
last_modified_at: 2026-09-26T11:40:43+01:00
tags: ["codex-cli", "daemon", "exec-server", "v0.157.0", "architecture", "always-on", "session-management", "background-server"]
---

# Daemon Auto-Start Is Now the Stable Default in Codex CLI v0.157.0


---

Codex CLI v0.157.0, released 25 September 2026, confirmed daemon auto-start as a stable default behaviour.[^1] The feature was introduced as opt-in in v0.156.0 and promoted to default during the v0.157 alpha cycle. With the stable release, every new interactive Codex session automatically starts a background server process unless one is already running. Codex is now, in practical terms, an always-on tool.

This is a quiet but consequential architectural shift. It changes the startup contract, the recovery UX, and the mental model for teams running Codex in multi-terminal workflows.

## What Daemon Auto-Start Does

Before v0.157.0, Codex CLI operated as a stateless per-session process. Each `codex` invocation launched the TUI, ran the session, and exited cleanly. The background server (`codex exec-server`) was an opt-in component that power users could enable explicitly.

With daemon auto-start as the default, the behaviour changes:

1. **On first launch**, Codex spawns a background server process (`codex exec-server`) automatically. The TUI connects to it rather than running inline.
2. **On subsequent launches**, Codex detects the running server and reconnects to it. Startup is faster because the server is already warm.
3. **On incompatible settings**, Codex offers a recovery choice: use the existing server (ignoring the conflicting settings), restart the server with the new settings, or exit and resolve the conflict manually.

The practical result: the server process outlives individual CLI sessions. It persists in the background between invocations, holding state that would otherwise be torn down on exit.

## Why This Matters

### Faster cold starts

The daemon approach eliminates the per-session initialisation overhead. Sandbox provisioning, model connection negotiation, and workspace indexing happen once at server start rather than once per invocation. On projects with large working trees, this can shave several seconds from every session launch.

### Persistent execution context

The background server holds execution state between sessions. A long-running background agent kicked off in one terminal window continues uninterrupted when you close that window and open a new one. This was possible before with explicit exec-server configuration but required deliberate setup; it now happens automatically.

### Multi-terminal workflows

Teams that keep Codex open in multiple terminal tabs simultaneously benefit from a shared server. All tabs connect to the same background process, sharing a consistent view of running agents and queued tasks. The v0.157.0 release includes the `f` shortcut to fork a conversation open in another app while preserving drafts and queued prompts — a feature that requires the shared server model to work.

### Recovery UX for conflicting settings

The recovery dialogue is new in v0.157.0. It surfaces when a session is launched with settings (model, sandbox profile, network policy) that differ from the running server's configuration. Rather than silently ignoring the conflict or crashing, Codex presents three options:

- **Use existing server** — proceed with the server's current settings; the session-level override is ignored.
- **Restart server** — stop the server, apply the new settings, restart, and reconnect. Interrupts any in-flight agents.
- **Exit** — leave Codex without connecting; resolve the conflict outside the tool (for example, by killing the server manually).

This is a meaningful UX improvement for workflows where different projects or tasks require different configurations.

## Configuration

The daemon auto-start behaviour is controlled by the `daemon` key in `~/.codex/config.toml`:

```toml
[daemon]
auto_start = true      # default from v0.157.0; set false to disable
port = 40022           # default port; change if you run multiple isolated environments
restart_on_conflict = false  # if true, restarts automatically without prompting
```

To revert to the pre-v0.157.0 per-session behaviour, set `auto_start = false`. The exec-server is still available for explicit invocation with `codex exec-server`; the only change is the default.

For CI/CD pipelines that use `codex exec` for headless runs, daemon auto-start does not apply — `codex exec` remains fully stateless. The daemon change affects interactive TUI sessions only.

### Checking server status

```bash
# Check whether the background server is running
codex server status

# Stop the background server manually
codex server stop

# Start the background server without opening a TUI session
codex server start
```

## Impact on Team Workflows

### Single-developer setups

For a solo developer using a single machine, daemon auto-start is transparent. The server starts, the TUI connects, and the only observable difference is faster subsequent launches and the ability to resume agents across terminal sessions.

### Multi-machine and remote setups

For teams using Codex on remote machines via SSH, the persistent daemon means agents keep running after an SSH session disconnects — previously, disconnecting would kill the Codex process. The same applies to VS Code Remote and JetBrains Gateway sessions. This is particularly useful for long-running background agents kicked off before leaving for the day.

### Docker and containerised environments

Container users should be aware that the daemon persists for the lifetime of the container, not the terminal session. In short-lived CI containers, the difference is negligible. In long-running development containers, the daemon accumulates state across sessions, which is usually desirable but worth monitoring for memory use on resource-constrained containers.

### Enterprise environments

Enterprise deployments with strict process isolation policies may need to disable daemon auto-start explicitly. The configuration key (`auto_start = false`) provides a clean opt-out. Administrators managing fleet deployments via managed config should add the `[daemon]` stanza to their baseline `config.toml` template.

## Relationship to the Exec-Server Architecture

The daemon model is the productised form of the `codex exec-server` subcommand introduced in v0.117.0 (March 2026).[^2] The exec-server was always the long-term architectural direction — a JSON-RPC execution backend that any frontend (TUI, IDE plugin, CI runner, remote SSH client) could connect to. Daemon auto-start is the moment that architecture became the default rather than the advanced configuration.

The implication for teams building tooling around Codex: the exec-server API is now a first-class interface, not an experimental one. Automation that connects to the exec-server over the JSON-RPC protocol is connecting to the same backend that interactive users hit.

## What Has Not Changed

- **`codex exec` is unaffected.** Non-interactive headless runs remain stateless.
- **The TUI experience is unchanged.** The connection to the background server is transparent; the TUI looks and behaves identically.
- **Agent definitions and `AGENTS.md` files are unaffected.** The daemon change is infrastructure-level; it does not alter how agents are defined, invoked, or scoped.
- **Model selection is independent.** The three-tier GPT-6 routing introduced in v0.156.1–v0.157.0 operates at the model layer, not the server layer. Configure model routing and daemon settings independently.

## Summary

Daemon auto-start becoming the stable default in v0.157.0 is a low-friction change with meaningful upside: faster sessions, persistent execution context, and better multi-terminal support. The recovery dialogue for conflicting settings adds a safety net that was missing from earlier opt-in configurations. For most developers, the right response is to do nothing — the default works well. Teams with specific isolation or CI requirements have a clear opt-out path via `config.toml`.

The architectural trajectory is clear: Codex is moving from a CLI tool you invoke to a persistent local service you interact with. Daemon auto-start is the stable milestone that marks that transition.

---

[^1]: OpenAI, "Codex CLI v0.157.0 release notes," 25 September 2026. Daemon auto-start confirmed stable default; recovery dialogue for incompatible server settings; fullscreen transcripts stable. https://developers.openai.com/codex/changelog
[^2]: OpenAI, "Codex CLI v0.117.0 — exec-server subcommand," March 2026. Introduction of `codex exec-server` as a standalone JSON-RPC execution backend. https://developers.openai.com/codex/changelog
