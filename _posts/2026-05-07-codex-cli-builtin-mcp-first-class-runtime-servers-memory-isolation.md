---
title: "Codex CLI's Built-in MCPs Just Became First-Class Runtime Servers — What It Means for Memory and Plugins"
description: "On 7 May 2026, Codex CLI merged PR #21356, authored by jif-oai, which fundamentally rearchitects how built-in MCP servers (like the memories server) run."
date: 2026-05-07T00:00:00+00:00
last_modified_at: 2026-08-14T14:10:16+01:00
tags: [codex-cli, mcp, architecture, memory, plugins, v0.129]
status: draft
type: Technical Article
timestamp: 2026-05-07T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-05-07-codex-cli-builtin-mcp-first-class-runtime-servers-memory-isolation"
---
![Sketchnote diagram for: Codex CLI's Built-in MCPs Just Became First-Class Runtime Servers — What It Means for Memory and Plugins](/sketchnotes/articles/2026-05-07-codex-cli-builtin-mcp-first-class-runtime-servers-memory-isolation.png)


# Codex CLI's Built-in MCPs Just Became First-Class Runtime Servers

*Published: 7 May 2026. Source: [PR #21356](https://github.com/openai/codex/pull/21356) merged into v0.129 alpha.15.*

---

## The Change

On 7 May 2026, Codex CLI merged PR #21356, authored by **jif-oai**, which fundamentally rearchitects how built-in MCP servers (like the `memories` server) run inside the Codex process.

Previously, built-in MCPs were treated identically to user-configured servers — launched as stdio subprocesses via a hidden `codex builtin-mcp` re-exec path. The new architecture introduces two distinct types:

- **`BuiltinMcpServer`** — product-owned runtime capabilities that launch in-process
- **`EffectiveMcpServer`** — the merged runtime set combining built-ins and user-configured servers

## Why It Matters

### 1. Memory Mode Pollution Is Gone

The most consequential change: the built-in `memories` MCP is now classified as **local Codex state** rather than external context. Previously, any MCP call — including internal memory operations — could mark a thread as "memory mode polluted", affecting how compaction and context isolation decisions were made.

This distinction matters for long sessions. When the agent stores or retrieves memories, those operations no longer trigger the external-context flags that could alter summarisation behaviour during compaction cycles.

### 2. In-Process Transport

Built-in MCPs now use a reusable async transport running inside the Codex process, eliminating the overhead of spawning a subprocess for every built-in MCP interaction. For workloads with frequent memory operations (e.g. agentic pods with persistent state), this should reduce latency.

### 3. Clean Plugin Boundary

The separation between built-in and configured servers is a prerequisite for the emerging skills marketplace. CLI operations like `codex mcp list/get/login/logout` are now scoped to configured servers only. Built-ins merge into the effective runtime set transparently, meaning plugin authors don't need to account for internal capabilities polluting their server lists.

## Architectural Diagram

```
Before (flat model):
  Config Layer → [memories, user-mcp-1, user-mcp-2] → All stdio subprocesses

After (two-tier model):
  Built-in Layer → [memories] → In-process async transport (local state)
  Config Layer   → [user-mcp-1, user-mcp-2] → Stdio subprocesses (external context)

  Effective Runtime Set = Built-in ∪ Config (merged at launch)
```

## Related Changes in alpha.15

The same release includes **operation-backed turn diff tracking** (PR #21180), part of the CCA file system isolation push. Together, these PRs signal Codex is maturing its internal architecture for cloud execution environments where filesystem access may be virtualised and process isolation is stricter.

## Implications for Agentic Workflows

For teams running multi-agent Codex workflows:

1. **Memory operations are cheaper** — no subprocess spawn per built-in MCP call
2. **Context isolation is cleaner** — memory reads/writes don't affect external-context classification
3. **Plugin introspection improves** — PR #21447 (same release) makes plugin hooks visible in detail views, so the full surface of any plugin is inspectable

## What to Watch

- Whether additional built-in MCPs beyond `memories` adopt the in-process transport
- How the `EffectiveMcpServer` type evolves to support marketplace plugin discovery
- Impact on compaction behaviour now that memory calls are classified as local state
