---
title: "The codex agents Dashboard and codex queue: Orchestrating Parallel Sessions Without the Terminal-Tab Overhead"
parent: "Articles"
nav_order: 1131
tags: ["codex-cli", "multi-agent", "session-management", "orchestration", "v0.149.0", "v0.150.0", "workflow"]
---

# The `codex agents` Dashboard and `codex queue`: Orchestrating Parallel Sessions Without the Terminal-Tab Overhead


Codex CLI v0.149.0, released 20 August 2026, shipped two primitives that change how you manage concurrent agent work: an interactive `codex agents` dashboard for surveying and controlling all running sessions at a glance, and a `codex queue` command for injecting messages into any session from outside it.[^1] Together they address a friction that anyone juggling multiple Codex tasks in parallel will recognise — the terminal-tab archaeology problem, where finding and instructing the right session means hunting through a forest of interchangeable windows.

This article covers the mechanics of both features, the working-directory commands that landed alongside them, the v0.150.0 `@` task mention that extended cross-session referencing one release later, and the practical orchestration patterns all of this enables.

## Why Multi-Session Management Was Broken Before v0.149.0

Codex has supported multiple concurrent local sessions since its app-server architecture matured (roughly v0.119.0). What it lacked was any first-class way to navigate between them. If you launched four sessions — one refactoring a monorepo module, one tailing logs, one running a test harness, one generating documentation — you had four terminal tabs, no shared index, and no way to send a follow-up message to session three without `cd`-ing into its window and typing.

The 0.149.0 release fixes all three pain points: a dashboard gives you the index, `codex queue` gives you the send-from-outside mechanism, and `/cd`/`/pwd`/`/cwd` give you working-directory control without restarting a session.[^2]

## The `codex agents` Dashboard

### Launching and Navigation

```bash
# From your shell — opens the full-screen dashboard
codex agents

# From inside any active Codex TUI session
/agents
```

The default keybinding to enter the dashboard from within a session is `Alt+A`.[^3] Once inside, the dashboard shows all sessions loaded by the shared local app-server: their names, status (`Need input`, `Working`, `Ready`), working directories, and active branch. Sub-agent activity is visible inline under each root session.

Keyboard controls:

| Key | Action |
|-----|--------|
| `↑` / `↓` | Navigate session list |
| `Enter` | Open selected session |
| `Ctrl+F` | Search sessions |
| `Ctrl+S` | Cycle grouping (by project / by status) |
| `Ctrl+R` | Rename selected session |
| `Ctrl+X` | Stop selected session |

All of these are remappable via the `tui.keymap` system introduced in v0.128.0.[^4] The dashboard shortcut itself sits at `tui.keymap.global.open_agents_dashboard`.

### Implementation Scope

The dashboard is a view over the **local** app-server. It does not federate remote sessions by default — a remote session is only visible after you connect to it explicitly with `--remote wss://host:port`. This matches the app-server's trust model: the dashboard never opens an unauthenticated cross-host view.

PRs: #39094 (core dashboard TUI), #39112 (status aggregation), #39114 (search and grouping), #39142 (configurable shortcuts).[^1]

## Working-Directory Commands: `/cd`, `/pwd`, `/cwd`

Shipped in the same release (PR #38894), these three slash commands remove the need to restart a session when your focus shifts within a repository.[^1]

```
# Show the current working directory of this session
/pwd
/cwd          # alias — identical behaviour

# Change working directory mid-session
/cd src/billing
/cd ../services/auth
```

For monorepos the pattern is common: a session launched at the root needs to drill into a specific package directory for a focused task, then return. Previously that required either spawning a new session or relying on the model to prefix every shell command with `cd`. With `/cd` the session's file-system context changes immediately and persists for subsequent tool calls.

## `codex queue`: Sending Instructions Without Switching Windows

### Command Syntax

```bash
# Named local session
codex queue --session my-refactor \
  "The migration tests are green. Proceed with the schema change."

# Read message from a file (useful for long prompts or CI output)
codex queue --session my-refactor \
  --text-file /tmp/ci-report.txt

# Remote session over WebSocket
codex queue --session my-refactor \
  --remote wss://workstation.local:4040
```

PR #39092.[^1]

`--session` accepts either the session's exact name or its UUID. Ambiguous names (two sessions sharing the same name) are resolved to the most-recently-active one — a deliberate tie-break rather than an error, though using UUIDs in automation scripts eliminates the ambiguity entirely.

### Delivery Guarantees

Three reliability properties were built into the initial release:[^5]

1. **Idle wake** — a message queued to a session that has completed its previous task and is awaiting input wakes it immediately. You do not need to open the TUI to restart it.
2. **Name-collision resolution** — as above, most-recently-active wins.
3. **Permission-profile restoration** — if the target session was resumed or forked, its permission profile is restored before the queued message is processed. A session that launched in `untrusted` mode stays in `untrusted` mode for queued instructions.

Queued messages inherit the target session's permission profile and cannot escalate privileges. Remote sessions authenticate via short-lived server tokens, not long-lived credentials.[^5]

### What `codex queue` Is Not

The semantics are fire-and-forget. There is no built-in acknowledgement, no response channel, and no structured message schema. If you need bidirectional coordination, you currently compose it yourself — typically by having the receiving session write results to a shared file or git branch, and the sending side polling that artefact.

This is a deliberate scope constraint: the primitive is intentionally minimal so it composes cleanly with existing shell tooling, hooks, and CI pipelines.

## Orchestration Patterns

### Pattern 1: CI-Driven Session Injection

A GitHub Actions workflow runs tests and, on failure, queues a triage message to a standing Codex session:

{% raw %}
```yaml
- name: Inject failure context into Codex session
  if: failure()
  run: |
    codex queue \
      --session "fix-${{ github.run_id }}" \
      --text-file test-results/summary.txt
```
{% endraw %}

The session was started before the CI run. It receives the failure summary and continues its triage loop without manual intervention.[^5]

### Pattern 2: Fork-and-Diverge with `codex exec fork`

`codex exec fork` (v0.148.0) branches a session at a decision point. `codex queue` then steers each branch independently:

```bash
codex exec fork --session my-task --fork-name variant-a
codex exec fork --session my-task --fork-name variant-b

codex queue --session variant-a "Implement using the adapter pattern"
codex queue --session variant-b "Implement using the strategy pattern"
```

The dashboard lets you watch both forks progress simultaneously and stop the losing approach early.

### Pattern 3: Metrics Drip-Feed via Cron

```bash
# In crontab — feed live metrics to a monitoring session every 5 minutes
*/5 * * * * codex queue --session monitor \
  "Current error rate: $(curl -s metrics.internal/error-rate | jq .rate)"
```

### Pattern 4: Cross-Session Context with `@` Mentions (v0.150.0)

v0.150.0 (26 August 2026) added task `@` mentions — the ability to reference another running session by name inside a message.[^6] When a session receives a message containing `@variant-a`, it can read that session's recent output, create a new task, or queue a message into it:

```
@variant-a What was the final approach you settled on?
```

This closes a gap in the fire-and-forget model: the receiving session can now pull context from a peer rather than waiting for a human to relay it. PRs #40308 and #40315.[^6]

## Session Lifecycle Flow

```mermaid
flowchart TD
    A[Developer: codex agents] --> B{Dashboard}
    B --> C[New task field]
    B --> D[Existing session]
    C --> E[codex run — new session]
    D --> F[Enter: open TUI]
    D --> G[Ctrl+X: stop]
    
    H[Shell / CI / Cron] -->|codex queue --session NAME msg| I[App Server]
    I -->|idle wake| J[Session resumes]
    I -->|active| K[Message queued — delivered after current turn]
    
    J --> L[Model processes message]
    K --> L
    L --> M{@ mention?}
    M -->|yes| N[Read peer session output]
    M -->|no| O[Normal tool loop]
```

## Configuration Reference

```toml
# ~/.codex/config.toml

[tui.keymap.global]
open_agents_dashboard = "alt-a"   # default; remap as needed

[tui.keymap.agents_dashboard]
search           = "ctrl-f"
cycle_grouping   = "ctrl-s"
rename_session   = "ctrl-r"
stop_session     = "ctrl-x"
```

Use `/keymap` inside a session to inspect and live-edit bindings. Codex writes changes back to config.toml.[^4]

## Companion v0.150.0 Improvements

Two v0.150.0 additions are worth noting alongside the dashboard and queue:

**Automatic descriptive titles** (PR #40492) — sessions without explicit names receive machine-generated titles derived from the first exchange. This makes the dashboard meaningful even for ad-hoc sessions.[^6]

**Interrupt hooks** (PR #40511) — commands registered under `[hooks.interrupt]` fire when an active top-level turn is interrupted. Combined with `codex queue`, this enables a clean handover pattern: the interrupt hook records state, and a queued message picks up from that state in a fresh session.[^6]

## What This Means for Workflow Design

Before v0.149.0, parallelising work in Codex was technically possible but ergonomically expensive. You needed to track sessions yourself, navigate between terminal windows, and manually copy context between them.

The dashboard, `codex queue`, and `@` mentions collectively shift Codex from a single-session tool to a session fleet that one developer can manage from a single command. The cap on concurrent sub-agents (six per root session as of v0.149.0) applies within a session tree; `codex queue` operates across independent sessions, so the practical ceiling on parallelism is now the token budget and the developer's ability to synthesise outputs — not an architectural limit.

The fire-and-forget semantics are the right call for a v1 primitive. They keep the tool composable with shell, cron, CI, and hooks without introducing a protocol surface that would need its own versioning and error model. If you need acknowledgement, write to a shared file. If you need structured bidirectional messaging, that is what MCP servers are for.

## Citations

[^1]: OpenAI, "Codex v0.149.0 Release Notes," GitHub, 20 August 2026. <https://github.com/openai/codex/releases/tag/rust-v0.149.0>

[^2]: Vladislav Guzey, "OpenAI Codex Agents Dashboard and Codex Queue Tutorial," proflead.dev, August 2026. <https://proflead.dev/posts/openai-codex-agents-dashboard-codex-queue/>

[^3]: The Forge / Hammer Automation, "OpenAI Codex 0.149.0 release notes: agent dashboard and codex queue," hammerautomation.ai, 21 August 2026. <https://hammerautomation.ai/en/forge/openai-codex-release-notes-2026-08-21>

[^4]: Codex Knowledge Base, "Configurable TUI Keymaps in Codex CLI: Custom Keyboard Shortcuts for Every Context," codex.danielvaughan.com, 6 May 2026. <https://codex.danielvaughan.com/2026/05/06/codex-cli-configurable-tui-keymaps-custom-keyboard-shortcuts/>

[^5]: Codex Knowledge Base, "codex queue and Inter-Session Messaging: What v0.149.0's New Primitive Means for Orchestration and Automation," codex.danielvaughan.com, 21 August 2026. <https://codex.danielvaughan.com/2026/08/21/codex-queue-inter-session-messaging-codex-cli-v0149-orchestration-automation-agent-to-agent/>

[^6]: OpenAI, "Codex v0.150.0 Release Notes," GitHub, 26 August 2026. <https://github.com/openai/codex/releases/tag/rust-v0.150.0>
