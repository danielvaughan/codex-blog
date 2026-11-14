---
title: "Codex CLI v0.167.2: Vertex AI Incremental Streaming, MCP OAuth Cancellation Safety, and Cloud Reconnect"
parent: "Articles"
nav_order: 1229
date: 2026-11-12T09:00:00+00:00
last_modified_at: 2026-11-14T10:09:22Z
tags: ["codex-cli", "release", "vertex-ai", "google-cloud", "mcp", "oauth", "cloud", "reliability", "streaming", "windows"]
copy_edited: true
---

# Codex CLI v0.167.2: Vertex AI Incremental Streaming, MCP OAuth Cancellation Safety, and Cloud Reconnect


---

Codex CLI v0.167.2 landed on 11 November 2026 at 22:44 UTC — the second stable patch on the v0.167.x line following v0.167.1 (10 November). It is a targeted reliability release with five fixes, the most significant of which improves streaming latency for Google Cloud Vertex AI users and hardens MCP OAuth credential persistence against cancellation races.

Install or upgrade:

```bash
npm install -g @openai/codex@0.167.2
```

---

## What Changed

### Incremental Vertex AI Tool Updates

The headline change: Codex CLI now sends tool call updates to Google Cloud Vertex AI incrementally by default, rather than batching them until a tool call completes.[^1]

For agentic workflows that chain multiple tool calls — file reads, shell commands, MCP calls — the previous batching behaviour introduced visible pauses: the Vertex AI endpoint received nothing while a long-running tool executed, then got the full payload at once. With incremental updates, partial results stream to Vertex AI in real time, reducing end-to-end latency for multi-step agent workflows running on Google Cloud.

If your workflow relies on the batched behaviour (for example, a downstream system that processes complete tool calls atomically), you can opt out via your `config.toml`:

```toml
[features]
incremental_tools_legacy_gcp = false
```

The opt-out flag is present from v0.167.2 onwards; prior to this release, batching was the only behaviour.

### Reactive MCP OAuth Refreshes Made Cancellation-Safe (PR #60981)

MCP servers that use HTTP transport with OAuth authentication occasionally trigger a reactive token refresh — a 401 response causes the client to pause the in-flight request, refresh the token, and retry. Before v0.167.2, if you cancelled the original MCP HTTP request during this reactive refresh window, the credential persistence step could be interrupted, leaving the token store in an inconsistent state.

v0.167.2 fixes this in two ways:[^1]

1. **Cancellation isolation** — the credential persistence step is now shielded from request-level cancellation. Cancelling the outer request no longer aborts the token write.
2. **Token reuse for concurrent 401s** — when multiple concurrent requests to the same MCP server each receive a 401 within the same refresh window, they now share the freshly rotated token rather than each attempting an independent refresh. This eliminates the redundant refresh race that could occur in multi-agent sessions where several subagents communicate with the same MCP server simultaneously.

In practice, this matters most in sessions with multiple agents connected to an MCP server that uses short-lived OAuth tokens (common in enterprise environments with strict token TTLs).

### Idle Cloud Conversation Reconnection

Long-running Codex Cloud sessions — remote sessions left running overnight or across a network interruption — can experience transient stream loss. Before v0.167.2, a lost stream left the Cloud conversation in a disconnected state requiring manual recovery.

v0.167.2 adds automatic reconnection for idle Cloud conversations after a transient stream loss.[^1] The session resumes transparently without user intervention. This is particularly useful for teams using Codex Cloud for multi-hour agent runs or overnight batch workflows.

### Windows Runtime Registration Reconciliation

On Windows, Codex CLI's background runtime service occasionally encounters privilege errors during registration — typically when running in environments with restricted service-creation permissions. These errors previously left the runtime in a partially registered state.

v0.167.2 adds a reconciliation step that detects and resolves inconsistent registration state after a service privilege error, restoring normal operation without requiring a manual reinstall or restart.[^1]

### Thread-Local Model ID State Preservation

During session replay (re-running a previous session's history) and context compaction (reducing context size for long sessions), the active model ID could be lost from thread-local state. This caused the replayed or compacted session to fall back to the default model rather than the model that was in use when the session was originally recorded.

v0.167.2 preserves thread-local model ID state across both replay and compaction.[^1] Sessions that use non-default models — GPT-6.1 Terra or Luna rather than Sol, or a named custom backend — will now correctly restore the intended model when resumed from history.

---

## Context: The v0.167.x Stable Line

v0.167.0 shipped on 6 November 2026 as a substantial feature release: inline patch diffs in approval prompts, network approval prompt lifecycle fixes, pending-approval thread prioritisation in the subagent picker, and a correction to `PreToolUse` hook execution order. v0.167.1 (10 November) followed with an MCP server lifecycle fix and a Guardian v3 policy edge case correction.

v0.167.2 completes the initial stabilisation of the line. All three patches together bring the v0.167.x series to production quality for Google Cloud Vertex AI deployments, multi-agent MCP sessions with OAuth-protected servers, and Windows enterprise environments.

The development track has already moved to v0.170.0-alpha (reliability cluster sprint), with v0.169.0-alpha still active on a parallel track. The pattern of v0.167.x receiving concurrent backport maintenance while two alpha series run in parallel is consistent with how v0.165.x and v0.166.x were stabilised.

---

## Upgrade Path

| From | To | Notes |
|------|----|-------|
| v0.167.1 | v0.167.2 | Drop-in. No config changes required. |
| v0.167.0 | v0.167.2 | Drop-in. Includes v0.167.1 fixes. |
| v0.166.x or earlier | v0.167.2 | Review v0.167.0 release notes for breaking changes (hook execution order, approval prompt lifecycle). |

```bash
# Verify after upgrade
codex --version
# Expected: 0.167.2
```

---

## Summary

v0.167.2 is a focused stability patch. If you are running Codex on Google Cloud Vertex AI, it is worth upgrading for the incremental streaming improvement alone — the latency reduction for multi-tool workflows is immediate and requires no configuration change. The MCP OAuth cancellation fix addresses a failure mode that is rare in single-agent sessions but reproducible in high-concurrency multi-agent environments with OAuth-protected MCP servers.

[^1]: Codex CLI v0.167.2 release notes, 11 November 2026. https://developers.openai.com/codex/changelog
