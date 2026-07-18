---
title: "Codex CLI State of Play — May 2026"
description: "OpenAIs Codex CLI has matured rapidly in 2026. The ecosystem now boasts 280+ community tools across 20 categories, a maturing Python SDK, and enterprise."
date: 2026-05-22T00:00:00+00:00
last_modified_at: 2026-07-18T10:16:12+01:00
type: Technical Article
timestamp: 2026-05-22T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-05-22-codex-cli-state-of-play"
---
![Sketchnote diagram for: Codex CLI State of Play — May 2026](/sketchnotes/articles/2026-05-22-codex-cli-state-of-play.png)

# Codex CLI State of Play — May 2026

Published: 2026-05-22

## The Landscape

OpenAI's Codex CLI has matured rapidly in 2026. The ecosystem now boasts 280+ community tools across 20 categories, a maturing Python SDK, and enterprise adoption by companies including Cisco, Virgin Atlantic, Vanta, and Duolingo.

## What's New (v0.131–v0.133)

The past week alone has seen three significant releases:

- **Goal mode GA** (v0.133) — No longer experimental. Goals are persisted workflow objects that can drive toward objectives for hours or days. Now enabled by default with dedicated storage.
- **Permission profile inheritance** (v0.133) — List APIs and inheritance support for enterprise governance.
- **Python SDK auth** (v0.132) — First-class authentication, richer turn APIs, and concurrent turn routing.
- **codex doctor** (v0.131) — Support-ready diagnostics across runtime, auth, terminal, network, config, and local state.
- **Unified @ picker** (v0.131) — Search files, directories, and plugins from a single interface.
- **Plugin marketplace** (v0.131) — Marketplace CLI commands, version-aware sharing, and default-enabled plugin hooks.

## Agentic Workflows

The most important development for enterprise adoption is the maturation of three agentic patterns:

### 1. Built-in Subagents
Codex ships with three agent types (default, worker, explorer) and supports custom agents via TOML configuration. The `spawn_agents_on_csv` tool enables batch processing at scale.

### 2. Agents SDK Integration
Running Codex as an MCP server and orchestrating via the Agents SDK enables deterministic, reviewable multi-agent pipelines. The cookbook example demonstrates hierarchical coordination with project manager, designer, developer, and tester agents.

### 3. Goal Mode for Autonomous Work
Goal mode lets Codex pursue objectives over extended periods, with worktree integration for isolation. Combined with `codex exec` for CI/CD, this enables fully autonomous development workflows.

## Ecosystem Growth

The ecosystem trajectory mirrors npm's early years:
- 280+ tools across 20 categories
- skills.sh emerging as the de facto distribution channel
- Cross-model orchestration tools (mco) gaining adoption
- Enterprise plugin registries supporting private, org-scoped registries

## Key Tools for the Agentic Pod

| Tool | Why It Matters |
|------|---------------|
| mco-org/mco | Neutral orchestration across Codex, Claude Code, Gemini CLI |
| VoltAgent/awesome-codex-subagents | 136+ ready-made agents |
| ComposioHQ/agent-orchestrator | Parallel coordination with CI/CD |
| basilisk-labs/codex-swarm | Swarm intelligence for large refactors |
| codex exec + GitHub Action | Non-interactive CI/CD automation |

## What to Watch

1. **Plugin marketplace maturation** — Version-aware sharing and discovery
2. **Python SDK evolution** — Moving toward feature parity with CLI
3. **Remote workflows** — Daemon-managed `codex remote-control` for distributed teams
4. **GPT-5-Codex** — Now available via Responses API, expect more Codex-optimised models
