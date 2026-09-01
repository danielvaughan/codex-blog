---
title: "Codex CLI: State of Play — May 2026"
description: "OpenAIs Codex CLI has evolved from a simple terminal coding assistant into a full agentic platform. As of v0.133.0 (May 21, 2026), it includes:"
date: 2026-05-23T00:00:00+00:00
last_modified_at: 2026-09-01T22:09:28+01:00
type: Technical Article
timestamp: 2026-05-23T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-05-23-codex-cli-state-of-play"
tags:
  - codex-cli
  - state-of-play
  - releases
---
![Sketchnote diagram for: Codex CLI: State of Play — May 2026](/sketchnotes/articles/2026-05-23-codex-cli-state-of-play.png)

# Codex CLI: State of Play — May 2026

*Date: 2026-05-23*

## Where Codex CLI Stands Today

OpenAI's Codex CLI has evolved from a simple terminal coding assistant into a full agentic platform. As of v0.133.0 (May 21, 2026), it includes:

- **Subagents** for parallel, specialised work
- **Goal mode** for persistent, long-running objectives
- **Plugins and marketplace** for distributable workflows
- **MCP integration** for connecting to external tools
- **Hooks** for governance and audit logging
- **Python SDK** (`openai-codex`) for programmatic access

## The Agentic Stack

Codex's agentic capabilities form a layered stack:

1. **Foundation**: Rust-based CLI with bubblewrap sandboxing (Linux) and Docker devcontainer support
2. **Agent Layer**: Built-in agents (default, worker, explorer) + custom TOML-defined agents
3. **Orchestration**: Subagent spawning with configurable parallelism (`max_threads`) and depth limits
4. **Persistence**: Goal mode for multi-session, long-running objectives
5. **Extension**: Plugins bundling skills, apps, and MCP servers
6. **Integration**: MCP client/server, Agents SDK compatibility, AGENTS.md cross-agent standard

## Key Numbers

- 150+ ecosystem tools catalogued in awesome-codex-cli
- 136+ community subagents (VoltAgent collection)
- 1,340+ community skills (antigravity-awesome-skills)
- 20+ official launch plugins (Slack, Figma, Notion, Sentry)
- 60,000+ projects using agents.md standard

## Enterprise Angle

Thibault Sottiaux (OpenAI Head of Codex Product) positions Codex as "becoming the standard agent" for enterprise. Key enterprise features:

- Permission profiles with inheritance
- Hook-based audit logging and governance
- Managed `requirements.toml` for reproducible environments
- Plugin marketplace with enterprise controls
- OAuth support for MCP servers

## What's Missing / Coming

- Self-serve plugin publishing (announced "coming soon")
- Windows native support (expected late 2026)
- PreToolUse hooks for non-Bash tools (current gap)
- Deeper CI/CD integration patterns

## For the Agentic Pod

The most relevant areas for Daniel's agentic pod work:

1. **Subagent orchestration patterns** — PR review pipelines, CSV batch processing
2. **Goal mode for long-running automation** — migration tasks, documentation generation
3. **MCP server composition** — chaining multiple MCP servers for complex workflows
4. **Cross-agent standards** — agents.md for projects that use both Codex and Claude Code
5. **Codex as MCP server** — letting other agents (Claude Code) call Codex agents
