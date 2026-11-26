---
title: "Codex CLI v0.170.0 Stable: Realtime Voice Calls, OpenTelemetry Auth, Multi-Environment MCP, and Reliability Fixes"
date: 2026-11-18T08:00:00+00:00
last_modified_at: 2026-11-26T18:09:08Z
parent: "Articles"
nav_order: 2119
tags:
  - codex-cli
  - v0.170.0
  - release-notes
  - voice
  - webrtc
  - opentelemetry
  - mcp
  - enterprise
  - tui
  - reliability
  - windows
copy_edited: true
rating: 4.0
depth: 4
originality: 3
accuracy: 5
substack: 4
---

# Codex CLI v0.170.0 Stable: Realtime Voice Calls, OpenTelemetry Auth, Multi-Environment MCP, and Reliability Fixes

**Release date:** 18 November 2026, 03:40 UTC
**Version:** v0.170.0 stable
**Previous stable:** v0.169.0 (13 November 2026) / v0.169.1 patch (17 November 2026, 22:21 UTC)
**Sprint duration:** ~7 days of alpha work (alpha.1 opened 11 November; alpha.4.2 on 18 November before stable promotion)
**Install:** `npm install -g @openai/codex@0.170.0`

---

Five days after the dual-stable event that shipped v0.168.0 and v0.169.0 on the same calendar day, Codex CLI has promoted a third stable release in the same sprint window. v0.170.0 landed at 03:40 UTC on 18 November with a focused change set built around one significant new capability — realtime voice calls in the TUI — and a disciplined cluster of reliability fixes that address authentication edge cases, cloud reconnection, and platform parity on Windows.

Where v0.168.0 was a TUI-polish sprint and v0.169.0 was a deep infrastructure overhaul, v0.170.0 reads as a capability-plus-stability release: one headline feature that opens a new input modality, two infrastructure additions that extend Codex's production observability and MCP flexibility, and a half-dozen targeted bug fixes that make the system more resilient in conditions that controlled testing does not reliably reproduce.

---

## What Changed

### 1. Realtime Voice Calls in the TUI (#61172)

The headline addition in v0.170.0 is WebRTC negotiation for realtime voice calls within the terminal interface. The TUI now supports voice as a live interaction mode: rather than typing a prompt, a developer can speak directly to Codex and receive a real-time response in the terminal.

This is a meaningful expansion of the input surface. Voice input is not a novelty feature — it is a qualitatively different interaction mode suited to a different class of task. Describing a complex refactoring goal, explaining the intent behind a design decision, or narrating the behaviour of a bug as you observe it are all tasks where speech is faster and more natural than typed text. For developers working in environments where keyboard focus is split — debugging on one screen, monitoring on another — voice removes the context-switch cost entirely.

The WebRTC transport choice is deliberate. WebRTC provides low-latency, browser-compatible, encrypted peer-to-peer media negotiation. Using it for the TUI voice channel means the same transport layer could eventually serve browser-based Codex interfaces without protocol fragmentation. It also means the voice session inherits WebRTC's native NAT traversal properties, which matters for enterprise environments where outbound traffic passes through proxy layers.

The TUI voice feature builds on v0.168.0's audio attachment support, which introduced audio files as inputs to Codex tasks. v0.170.0 completes the logical arc: audio can now be provided as a live stream, not just a static file attachment.

### 2. OpenTelemetry HTTP Exporter Auth Header Refresh (#61028)

v0.170.0 adds automatic refresh of authentication headers for OpenTelemetry HTTP exporters. This is an infrastructure change aimed at teams running Codex in observability pipelines where telemetry is exported to backends — Grafana, Honeycomb, Datadog, or internal OTEL collectors — via authenticated HTTP endpoints.

The problem addressed: OpenTelemetry HTTP exporters that require authentication (bearer tokens, API keys) previously acquired those credentials at session start and cached them for the duration of the session. In long-running Codex sessions or in CI pipelines where token rotation is aggressive, the exporter's cached credentials could expire, causing telemetry exports to fail silently.

With v0.170.0, the exporter independently refreshes its authentication headers. Telemetry delivery no longer depends on credentials that were valid when the session started; it depends on credentials that are valid when the export fires.

The practical significance is highest for enterprise observability setups. Teams that have instrumented Codex with OTEL tracing to track agent behaviour, tool call patterns, and session costs will find that their telemetry pipelines become more reliable across long sessions — and that the tracing data they depend on for cost governance and debugging continues to flow even when token rotation policies are strict.

### 3. Experimental: Identically-Named MCP Servers Across Environments (#60905)

v0.170.0 ships experimental support for using identically-named MCP servers across different environments. Prior to this change, MCP server names were required to be globally unique within a Codex configuration. If you ran Codex in a development environment and a production environment, and both environments had an MCP server called `database`, the naming conflict required one to be renamed or configuration to be context-switched manually.

The experimental flag matters: this feature is not yet stable API and its behaviour may change in a future release. Teams should not build production workflows that depend on the precise semantics of this feature before it graduates to stable.

What it enables in practice: multi-environment MCP configurations — dev, staging, production — where the same logical tool surface (same server name, same tool descriptions) points to environment-appropriate backends. For agentic workflows that need to operate across environment boundaries without changing their tool invocation patterns, this is the enabler.

The feature complements the persistent MCP connection denial behaviour shipped in v0.169.0 (#60046). Together they give enterprise operators more precise control over which MCP connections are established and which server names are active in each context.

---

## Reliability and Bug Fix Cluster

The remaining changes in v0.170.0 are fixes rather than features, but several address failure modes that have been visible in community issue trackers and enterprise support channels since the v0.169.x line.

### WebSocket Oversized Request Fallback (#60840, #60841)

Oversized WebSocket requests now fall back to HTTP without entering retry loops. The previous behaviour — retrying a failed WebSocket send that exceeded the frame size limit — could cause pipeline stalls when the message was large and the retry budget was high. The fix adds a size check that routes oversized payloads to HTTP transport directly, bypassing the WebSocket path and the unnecessary retry cycle.

### MCP OAuth Credential Race Conditions (#60849, #60981)

Two related fixes address race conditions in MCP OAuth credential management during cancellation, login, and logout transitions. In setups where the user interrupts an OAuth flow or logs out while a credential refresh is in flight, the previous code could leave the credential store in an inconsistent state — authenticated for some operations but not others. The fixes serialise the credential lifecycle transitions to prevent these races.

### Cloud Conversation Reconnection (#60969, #61014)

Idle cloud conversations now reconnect automatically after brief stream interruptions, and active recovery turns survive client reconnections without being dropped. These two fixes address the most common form of cloud session disruption: a network hiccup or backend scaling event that briefly drops the streaming connection. Previously, recovery required the user to manually restart the session. With v0.170.0, the reconnection is handled transparently for both idle and mid-turn scenarios.

### Windows File Replacement and Sandbox Permissions (#60872, #60943, #60944)

Three related fixes address Windows-specific sandbox issues: file replacement operations that were failing silently in sandboxed contexts, sandbox permission settings that were not being correctly applied during Windows file I/O, and runtime registration edge cases introduced by the interaction between Windows path handling and the sandbox policy layer. Windows support in Codex has been an ongoing stabilisation effort; this cluster continues the work.

### Sandbox Permission Inheritance Fix (#60936, #61134)

A subtle but important fix: permission updates were dropping existing sandbox restrictions from child threads. When a permission update was applied to a session, the update logic was rebuilding the child thread permission set rather than merging with it — with the effect that restrictions established at thread creation could be silently removed. The fix ensures permission updates are applied additively: they can tighten or extend existing restrictions but cannot implicitly clear them.

### Duplicate File Search Results (#60845, #60846)

Two fixes address file search quality in the TUI: duplicate results were appearing in file search output under certain indexing conditions, and thread search snippets were rendering incorrectly when the matched text contained leading or trailing whitespace. Both are polish fixes; neither is a regression introduced in v0.170.0 — they address long-standing issues that became more visible as file search was used more heavily in multi-agent workflows.

---

## Release Context

v0.170.0 is a coherent release that balances a genuinely new capability (voice calls) with the kind of infrastructure work that keeps Codex reliable at production scale. It does not carry the architectural depth of v0.169.0's runtime auth migration, but it is not attempting to. What it delivers is a voice input channel that extends how developers can interact with the terminal, two backend infrastructure improvements that matter to teams running Codex in observed, multi-environment production contexts, and a set of targeted fixes that address the highest-friction edge cases reported since the v0.169.x stabilisation.

The sprint opened on 11 November with alpha.1 and promoted through alpha.4.2 before the stable cut on 18 November. The dual-track alpha maintenance pattern continues: v0.170.0-alpha.4.2 landed at 01:52 UTC on 18 November — less than two hours before the stable promotion — suggesting last-minute confidence verification rather than any significant late change.

---

## What Is Already in Flight: v0.171.0-alpha.13

The v0.171.0 sprint is well underway. Alpha.13 shipped at 05:47 UTC on 18 November — just two hours after v0.170.0 stable was cut. The sprint has been running since 13 November and shows no signs of slowing: 13 alphas in five days matches the velocity that produced v0.169.0 and v0.168.0.

Sprint theme for v0.171.0 is not yet fully visible from release note patterns, but the community is watching for SPIFFE JWT-SVID authentication (enterprise workload identity for MCP) and improvements to the Windows AppX packaging. If the cadence holds, v0.171.0 stable could arrive by mid-to-late this week.

---

## Upgrading

```bash
npm install -g @openai/codex@0.170.0
```

No breaking changes are documented in v0.170.0. The realtime voice feature is additive — no configuration changes are required to access it, and existing text-based workflows are unaffected. The experimental MCP server naming feature is opt-in. The reliability fixes are transparent to users.

If you are running Codex in CI with pinned versions, the v0.169.1 patch (17 November) addressed minor bug fixes; v0.170.0 is the recommended upgrade target for production stable deployments as of today.

---

## Summary

| Change | PR(s) | Category |
|---|---|---|
| Realtime voice calls via WebRTC negotiation in TUI | #61172 | Capability |
| OpenTelemetry HTTP exporter auth header auto-refresh | #61028 | Observability |
| Experimental identically-named MCP server support across environments | #60905 | MCP |
| Oversized WebSocket request fallback to HTTP | #60840, #60841 | Reliability |
| MCP OAuth credential race condition fixes | #60849, #60981 | Authentication |
| Cloud conversation auto-reconnection | #60969, #61014 | Reliability |
| Windows file replacement and sandbox permission fixes | #60872, #60943, #60944 | Platform |
| Sandbox permission inheritance (child threads) | #60936, #61134 | Security |
| Duplicate file search result and snippet fixes | #60845, #60846 | Polish |

v0.170.0 opens a voice channel that changes how developers interact with the terminal, adds observability infrastructure that makes Codex more trustworthy in production pipelines, and clears a backlog of reliability issues across authentication, cloud connectivity, and Windows platform support.

---

*Source material: openai/codex GitHub releases and official Codex changelog (Nov 18 2026). Fact-checked against primary source on 2026-11-19.*
