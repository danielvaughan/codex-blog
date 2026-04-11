---
title: "Background Agent Streaming: From Fire-and-Forget to Observable Multi-Agent UX"
date: 2026-04-11T14:00:00+01:00
tags: ["multi-agent", "background-agent", "realtime-v2", "streaming", "v0.120", "UX", "delegation", "agentic-pod"]
---

# Background Agent Streaming: From Fire-and-Forget to Observable Multi-Agent UX


---

Until v0.120.0, Codex CLI's multi-agent delegation followed a **fire-and-forget** pattern: the orchestrator spawned a subagent, waited for it to finish, and then received the final result. For quick tasks this worked fine. For long-running delegated work — large refactors, cross-repo migrations, complex test generation — the foreground agent (and the human) sat in silence, unable to see progress or intervene.

v0.120.0 changes this with **background agent progress streaming**, marking the shift from opaque delegation to **observable multi-agent orchestration**.

## What Changed

The internal "Realtime V2" tool has been renamed to `background_agent` (PR #17278), signaling that multi-agent delegation is now a first-class named concept rather than an experimental extension of the realtime voice pipeline.

Key capabilities in v0.120.0:

- **Progress streaming while running** — background agents stream incremental progress updates back to the foreground agent and TUI while work is still in progress
- **Queued follow-up responses** — if the background agent produces output while the foreground agent is still responding, updates queue rather than being lost
- **Live TUI rendering** — running hooks and background agent activity now display separately from completed output, with 300ms reveal timing and collapsible parallel hooks to reduce noise

## Why This Matters for Agentic Pods

In Daniel's agentic pod architecture — where multiple specialised agents handle different aspects of a task (code generation, testing, review, documentation) — observable delegation solves three problems:

### 1. Progress Visibility

Previously, delegating a 10-minute test generation task to a subagent meant 10 minutes of silence. Now the orchestrator (and human) can see incremental progress: "Generated 3/12 test files", "Running test suite", "Fixing 2 failing tests". This changes the human's role from waiting to **monitoring**.

### 2. Early Intervention

With streaming progress visible, the orchestrator or human can detect when a background agent is going down the wrong path and intervene before it burns through tokens on wasted work. This directly addresses the "token drain trust gap" — one of the top friction points in multi-agent workflows.

### 3. Parallel Coordination

When multiple background agents run concurrently (e.g., one generating code while another writes tests), their progress streams can be observed side by side. The collapsible parallel hooks in the TUI prevent information overload while keeping everything accessible.

## Architectural Pattern: Observable Delegation

The background agent streaming model suggests a new delegation pattern for agentic pod design:

```
Orchestrator Agent
  ├── spawn background_agent "generate-tests" → streams progress
  ├── spawn background_agent "update-docs" → streams progress
  └── monitor progress, intervene if needed, merge results
```

This is architecturally closer to how Kubernetes manages pods with readiness probes and health checks than to the traditional "call a function and wait" model.

## Comparison with Claude Code Agent Teams

Claude Code's agent teams use **inter-agent messaging** — agents communicate directly with each other in real time via `SendMessage`. Codex's background agent streaming is a different model: progress flows **upward** from background agent to orchestrator, not laterally between peers.

| Aspect | Codex Background Agent | Claude Code Agent Teams |
|--------|----------------------|------------------------|
| Communication | Upward (agent → orchestrator) | Lateral (agent ↔ agent) |
| Visibility | Streamed to TUI | Via SendMessage |
| Coordination | Orchestrator-mediated | Peer-to-peer |
| Best for | Hierarchical delegation | Collaborative tasks |

Both patterns have their place. Hierarchical delegation with streaming is better for tasks with clear ownership (one agent owns test generation). Peer-to-peer messaging is better for tasks requiring negotiation (e.g., API contract design between frontend and backend agents).

## Configuration

Background agent streaming works automatically in v0.120.0+ when using `codex exec` or the TUI. No additional configuration required. The TUI rendering improvements (collapsible hooks, 300ms reveal timing) are also automatic.

## What's Next

The `background_agent` rename and streaming support suggest Codex is building toward a more structured multi-agent runtime. Combined with guardian review IDs (also in v0.120.0), this means background agents can be both observed and governed — the two prerequisites for enterprise multi-agent deployment.

---

*Source: [Codex CLI v0.120.0 Release](https://github.com/openai/codex/releases) | PRs #17278, #17264 | Added 2026-04-11*
