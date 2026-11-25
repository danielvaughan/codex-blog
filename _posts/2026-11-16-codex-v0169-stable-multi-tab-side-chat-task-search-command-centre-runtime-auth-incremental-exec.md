---
title: "Codex CLI v0.169.0 Stable: Multi-Tab Side Chat, Task Search in Agent Command Centre, Runtime Auth Overhaul, and Incremental Exec Catalog"
date: 2026-11-16T08:00:00+00:00
last_modified_at: 2026-11-25T15:06:38Z
parent: "Articles"
nav_order: 2110
tags:
  - codex-cli
  - v0.169.0
  - release-notes
  - side-chat
  - agent-command-centre
  - runtime-auth
  - mcp
  - security
  - tui
  - code-mode
copy_edited: true
rating: 4.6
depth: 5
originality: 4
accuracy: 5
substack: 4
---

# Codex CLI v0.169.0 Stable: Multi-Tab Side Chat, Task Search in Agent Command Centre, Runtime Auth Overhaul, and Incremental Exec Catalog

**Release date:** 13 November 2026, 08:37 UTC
**Version:** v0.169.0 stable
**Previous stable:** v0.168.0 (13 November 2026, 03:54 UTC)
**Sprint duration:** ~7 days of alpha work across four sprint waves; stable promoted the same calendar day as v0.168.0 in a dual-track release
**Install:** `npm install -g @openai/codex@0.169.0`

---

v0.169.0 is the most substantive release of the November 2026 sprint cycle. Where v0.168.0 focused on TUI surface polish — model provider browsing, resizable side panes, audio attachments — v0.169.0 reworks what happens underneath: how the CLI authenticates every request, how Code Mode tool catalogs update without full session reinitialisation, and how MCP servers start up under load. The user-facing headlines are genuinely useful — multi-tab side chats that persist across restarts, task search across the Agent Command Centre history, and collapsible turn details in long transcripts — but the engineering significance of this release is the runtime authentication migration and the incremental exec tool catalog.

---

## What Shipped in Stable

### Multi-Tab Side Chat with Persistent History

The most visible change is to the sidebar. Side chats — the parallel conversation threads introduced in v0.167.0 — are now **persistently saved and tabbed**. Opening a side chat from the main workspace produces a tab in the sidebar. Tabs can be named, navigated by keyboard (`Shift-Tab` / `Tab`, `Up` / `Down`, `k` / `j`) and mouse (wheel over the tab strip), and closed with middle-click or `Ctrl-X W`. Closing a tab does not destroy the side chat — it remains in the sidebar list and can be reopened.

The persistence guarantee is new: when local workspace history is enabled, side chats survive a restart. Previously, side chats were ephemeral; a session restart wiped the entire sidebar. Now they are saved as attachments to their parent conversation. The `/rename` command applies to side chats, making a multi-chat workflow navigable across sessions rather than merely across minutes.

**Why this matters for teams:** Multi-tab side chat with persistence closes the feedback loop on the v0.167.0 launch. The community reception to side chats was broadly positive but tempered by two complaints — they were spatially cramped (addressed in v0.168.0's resizable panes) and they did not survive a restart (addressed here). Both issues are now resolved. A developer can open a side chat to sanity-check a dependency, rename it `dep-audit`, and return to it the following day without losing context.

### Task Search in the Agent Command Centre

The Agent Command Centre gains **full-text search across task history**. Operators running long multi-agent sessions can now type to filter the task list rather than scrolling through a chronological queue of completed and active tasks.

This is a fleet-management quality-of-life addition, but its impact scales non-linearly with session depth. A team running 20 concurrent tasks can find a specific debugging task instantly; a team running 200 cannot function without it. As agentic session depth increases — driven by features like automatic sub-task delegation in v0.167.0 and the queue-and-resume patterns in v0.168.0 — the command centre task list becomes the primary operational surface. Search makes it usable at scale.

### Collapsible Turn Details in Full-Screen Transcripts

Full-screen transcript view now supports **collapsible turn detail blocks**. Long agentic sessions produce extensive transcripts — multi-hundred-turn exchanges with tool calls, compaction events, and intermediate outputs that are rarely needed after the fact. The collapsible rendering reduces visual noise for sessions where the final output is what matters, while keeping the full detail accessible via expand.

This is a minor but high-frequency quality-of-life change for anyone auditing sessions post-run.

### Custom Model Provider from the Terminal

Custom model provider configurations — connections to self-hosted models, third-party gateways, or Amazon Bedrock — can now be **created directly at sign-in** without a prior settings visit. The sign-in picker includes "Create a custom provider or gateway", which opens the settings form inline. After saving, the picker returns and highlights the new provider, requiring explicit selection before it becomes the default.

Previously, first-run setup for a custom provider required visiting Settings first, saving a configuration, and then signing in. For enterprise deployments and teams using private gateway configurations, this flow is now collapsed into a single step. The form preserves entered fields after validation errors, which removes the frustration of re-entering a gateway URL after a misconfiguration.

Combined with the interactive provider browser that shipped in v0.168.0, provider selection is now a fully terminal-native workflow from first connection through ongoing management.

### Improved Cloud Connection on macOS and Linux

The changelog records unspecified improvements to Cloud connection behaviour on macOS and Linux. The alpha sprint notes point to three specific Cloud thread reliability fixes that likely underpin this entry:

- Sessions that ended in a Cloud failure without producing a thread item are now included in full transcript views (previously invisible)
- Stale stream disconnect events after thread reopening are suppressed
- Duplicate assistant messages on thread resumption are deduplicated

These are correctness fixes for common failure modes in unstable network conditions — the class of issues that surfaces in long overnight agent runs or in office environments with intermittent connectivity.

---

## Under the Hood: What This Release Actually Changed

The user-facing features above represent a fraction of the sprint's scope. Three engineering changes in v0.169.0 have long-term architectural significance.

### Runtime Authentication Migration

v0.169.0 completes a systematic migration of authentication across every major Codex subsystem. The change is conceptually simple: authentication tokens are now derived at the moment of each request rather than cached at startup or reused across sessions.

Before this sprint, several subsystems — Guardian Decisions, the memory quota checker, remote-control, Analytics, service-tier reads, and security reminders — held tokens that were acquired at session start and reused for the duration. This produced a class of silent failures (stale-token errors mid-session), created a correlated blast radius if a token was compromised, and prevented subsystems from handling auth denials independently.

The migration covers eight subsystems across PRs #60055–#60101:

| Subsystem | Change |
|-----------|--------|
| WebSocket | Re-authenticates before reusing an existing access token |
| Guardian / Memory | Thread-scoped runtime auth rather than process-level shared token |
| Remote-control | Per-request runtime auth |
| Service-tier | Async reads — no longer blocks the request pipeline |
| Budget | Async model request budget checks |
| Security reminders | Fetched through app-server, not cached credentials |
| Analytics | Per-request runtime auth |
| Auth error handling | Terminal denials short-circuit rather than entering a retry loop |

**Practical implications for operators:**

- **CI/CD pipelines** that previously relied on a session-start token surviving through long runs will now encounter per-request re-auth. Pipelines using short-lived tokens (GitHub OIDC, AWS IAM roles, Vault leases) benefit immediately — the system now requests a fresh token rather than failing silently with an expired one.
- **Multi-agent setups:** Guardian and memory quota auth is now thread-scoped. Each agent thread carries its own runtime auth context. This enables finer-grained access control in workflows where different agent threads should have different permission boundaries.
- **Non-interactive runs:** Combined with the explicit project trust requirement also introduced in this sprint (#60041), non-interactive Codex runs now fail fast and clearly on auth or trust issues rather than hanging. CI runs that lack an explicit trust declaration will exit with a clear error rather than blocking on an auth retry loop.

### Incremental Exec Tool Catalog for Code Mode

v0.169.0 introduces an incremental update mechanism for Code Mode's exec tool catalog. Prior to this sprint, any change to the tool definitions available in a Code Mode session required re-declaring the full `functions.exec` block — a costly operation in long sessions with dynamic tool sets (skills that add new nested tools mid-session, MCP server updates, plugin activations).

The incremental catalog (#60157, #60163) tracks stable entries for nested tool definitions and namespace instructions. When a tool is added or changed, it emits a targeted replacement. When removed, it emits an explicit removal notice. The existing full exec declaration is preserved as the foundation; incremental updates layer on top. If exec becomes unavailable (e.g. a network partition), entry hashes are preserved; the full catalog is restored on reconnection and removed entries are withdrawn.

The pipeline is opt-in behind `IncrementalTools` and `IncrementalExecTools` flags. When both are active, Code Mode tool catalog changes propagate without forcing a full session reinitialisation. The observability layer (#60177) instruments the pipeline with two new metrics: `codex.tools.exec_entry_updates` (tagged `added`, `schema_changed`, `removed`) and `codex.tools.exec_update_bytes`.

**Practical implications:** Teams running large multi-skill Code Mode sessions — particularly those where skill tool definitions change as work progresses — will see reduced context overhead and lower compaction pressure once the flags are enabled by default in a future release.

### MCP Lazy Startup

Stdio MCP servers can now defer their startup until the first tool call, behind an opt-in lazy flag. Three PRs (#60037–#60039) deliver the feature: deferred startup, clear error surfacing when a deferred server fails to start, and preservation of server capability sharing (tool sharing across providers) during deferral.

The motivation is cold-start latency. AGENTS.md configurations with many MCP server definitions previously incurred the full startup cost of every server on session initialisation, even for servers whose tools were never invoked in a given session. Lazy startup eliminates that cost for unused servers.

The opt-in nature is intentional — lazy servers that fail on first tool call surface errors at a potentially surprising point in the session. Teams should enable the flag for well-understood, stable servers and leave it disabled for servers where startup failure is a signal worth catching early.

---

## Security Changes Worth Noting

Two security-focused PRs merit explicit attention:

**Goal objectives as untrusted context (#60342):** Goal objective text is now classified as untrusted input within Guardian's security model, consistent with how user messages are treated. This closes a prompt-injection vector where a malicious goal objective string could have bypassed Guardian's trust boundary. Teams using goal-driven orchestration patterns should note that Guardian now applies the same scrutiny to objective content as to user messages — including content policy checks and trust-boundary enforcement.

**Sensitive process inputs omitted from spawn trace logs (#60034):** Process arguments passed to spawned subprocesses are no longer included in trace and debug logs. This prevents accidental credential leakage in verbose output — a risk in CI environments where debug logs are retained or forwarded to external systems.

**Explicit project trust in non-interactive runs (#60041):** Projects must now be explicitly trusted before Codex executes in non-interactive mode. This closes a vector where a CI runner could execute Codex against an untrusted workspace without an explicit trust declaration. Add `CODEX_TRUST=1` or the equivalent CLI flag to affected pipelines.

---

## Other Notable Changes

**Child agent model catalog inheritance (#60049):** When Codex spawns a child agent, the child now starts with the same model catalog as the parent. Previously, catalog divergence between parent and child was a subtle source of non-determinism in orchestrated workflows — a child agent could select a different model than intended because its catalog was populated independently. This is now fixed.

**Side-chat policy on first request (#60067):** A race condition in policy application meant the side-chat content policy was not guaranteed to apply to the very first model request in a new side-chat. Fixed. This matters for teams with strict content policies in regulated environments.

**MCP saved connection denials persist across restarts (#60046):** MCP server connection denials are now saved and survive a restart. This prevents repeatedly prompting for the same denied connection in persistent-session workflows. Teams that have denied a connection and want to re-enable it will need to clear saved denials explicitly.

**Model list warning for absent CLI model (#60270):** A new warning surfaces at session start when the model configured in `config.toml` is not present in the current ChatGPT model list — for example, after a model deprecation or when the configured identifier is not available on the user's subscription tier. Previously this surfaced as a silent degradation mid-session.

**Gateway OAuth model catalog discovery (#60414):** Enterprise deployments routing all API traffic through an OAuth-authenticated gateway (zero-trust architectures) previously saw an empty or stale model catalog because catalog discovery did not trigger the OAuth challenge. The catalog is now populated after the gateway auth handshake completes.

---

## Upgrade Notes

**Install:** `npm install -g @openai/codex@0.169.0`

**CI/CD pipelines using non-interactive mode:** Add `CODEX_TRUST=1` or the equivalent explicit trust flag if your pipeline does not already declare project trust. Non-interactive runs that lack explicit trust will now fail fast with a clear error rather than hanging.

**Teams with MCP server denial records:** Saved connection denials now persist across restarts. If you need to re-enable a previously denied server, clear the saved denial in settings.

**Plugin developers:** The 512-character starter prompt limit (#60048, alpha sprint) allows richer onboarding prompts. No breaking change; the previous shorter limit was simply a constraint that is now relaxed.

**Incremental exec catalog:** The opt-in flags (`IncrementalTools`, `IncrementalExecTools`) are not yet enabled by default. No action required; teams wanting to test the reduced context overhead in long Code Mode sessions can enable both flags.

---

## Context: The Dual-Track Release Pattern

v0.169.0 stable shipped on the same calendar day as v0.168.0 stable — a dual-track release pattern that reflects OpenAI's continued high velocity on the Codex CLI. At the time of the v0.169.0 stable promotion, the v0.171.0 alpha sprint was already at alpha.5. The numbering gap (v0.169.0 stable + v0.171.0-alpha.5 on the same day) reflects alphas cycling faster than stables are promoted, not skipped versions.

For teams managing upgrades: v0.169.0 is the recommended stable target. v0.170.0 stable has not yet shipped. v0.171.0 is expected to reach stable within days based on the alpha sprint velocity observed across 13–14 November.

---

## Summary

v0.169.0 is a significant release across two dimensions. For users, it closes the loop on v0.167.0's side-chat launch — persistent tabs and search make multi-chat workflows genuinely usable at scale. For operators and platform teams, the runtime auth migration and security hardening are the more important changes: they address a structural authentication fragility that accumulated as new subsystems were added without being migrated to per-request auth. The incremental exec catalog is a foundation for reducing context pressure in long Code Mode sessions; its impact will grow as the flags move toward default-on.

Teams running agentic workflows at scale — particularly those with large AGENTS.md configurations, gateway OAuth deployments, or CI pipelines in non-interactive mode — should treat this upgrade as high priority.
