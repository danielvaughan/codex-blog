---
title: "Gemini CLI Subagents: Multi-Agent Orchestration Arrives"
description: "Source: Google Developers Blog Author: Google Published: 2026-04-15 Content age: Current — announced April 15, 2026 Tags: gemini-cli, subagents,."
date: 2026-04-15T00:00:00+00:00
last_modified_at: 2026-06-18T10:15:57+01:00
tags: ["gemini-cli", "subagents", "multi-agent", "competitor-update", "google"]
---

# Gemini CLI Subagents: Multi-Agent Orchestration Arrives

**Source:** [Google Developers Blog](https://developers.googleblog.com/en/subagents-have-arrived-in-gemini-cli/)
**Author:** Google
**Date saved:** 2026-04-16
**Content age:** Current — announced April 15, 2026

---

## Summary

Google has introduced subagents to Gemini CLI, enabling the primary agent to delegate complex tasks to specialized expert agents that operate in isolated context windows. Each subagent runs with its own system instructions and curated tool set, consolidating potentially dozens of tool calls into a single summary response back to the main agent — preventing context pollution and keeping the orchestrator focused on high-level decision-making. The feature includes three built-in subagents, custom agent definitions via Markdown files with YAML frontmatter, and parallel execution of multiple subagents simultaneously.

---

![Sketchnote diagram for: Gemini CLI Subagents: Multi-Agent Orchestration Arrives](/sketchnotes/articles/2026-04-15-gemini-cli-subagents-launch.png)

## Key Points

- **@agent invocation syntax** — subagents are invoked with `@agent-name` inline (e.g. `@frontend-specialist review our app`), not a separate API call
- **Isolated context windows** — each subagent gets its own context window, custom system instructions, and curated tool set, preventing context rot in the main session
- **Parallel execution** — multiple subagents can run simultaneously (e.g. "Run the frontend-specialist on each package in parallel"), drastically reducing total completion time; caveat: risk of code conflicts and faster rate-limit consumption
- **Summarised responses** — subagents consolidate entire multi-step executions (dozens of tool calls, file searches, test runs) into a single concise response back to the orchestrator
- **Custom agent definitions** — defined as Markdown files with YAML frontmatter; stored globally in `~/.gemini/agents` or per-project in `.gemini/agents`; can also be bundled in Gemini CLI extensions via an `agents/` directory
- **Built-in subagents** — `generalist` (general-purpose with all tools for batch refactoring), `cli_help` (Gemini CLI documentation expert), `codebase_investigator` (codebase exploration and architectural mapping)
- **Agent discovery** — `/agents` command lists all configured subagents
- **Extension bundling** — subagents can ship inside Gemini CLI extensions, enabling team-wide distribution of specialised agents

---

## Competitive Implications for Codex CLI

Codex CLI has had multi-agent capabilities through MCP parallel-call since v0.121.0 and cross-provider bridging via `codex-plugin-cc`, but it does not yet have native subagent spawning with isolated context windows. Key differences:

| Capability | Gemini CLI (April 2026) | Codex CLI (v0.121) |
|---|---|---|
| Native subagent spawn | Yes — `@agent` syntax | No — uses MCP tool calls |
| Isolated context | Yes — per-subagent window | Partial — MCP servers are separate processes but not agent-level isolation |
| Parallel execution | Yes — multiple `@agent` calls | Yes — MCP parallel-call |
| Custom definitions | Markdown + YAML frontmatter | TOML subagent config / Skills |
| Cross-provider | No (Gemini models only) | Yes — `codex-plugin-cc` bridges to Claude, Gemini |
| Built-in agents | 3 bundled specialists | Templates (orchestrator, reviewer) |

Codex CLI's MCP-based approach is more protocol-standard and cross-provider, but Gemini's native subagent UX is more ergonomic with the `@agent` invocation pattern and automatic context isolation. This is the clearest signal yet that native subagent spawning (not just MCP delegation) is becoming table stakes.

---

## Book/Article Impact

- **ch05 (Competing Tools)** needs updating — Gemini CLI now has native subagent orchestration, not just basic tool use
- **Premium article 06 (Three Terminals)** should note this — the existing bridging section mentions Gemini CLI but predates subagent support
- **Premium article 05 (Agentic Pod)** — Gemini's subagent model maps naturally to pod roles (frontend-specialist, codebase_investigator, etc.), worth a comparison sidebar

---

## Personal Notes

Google matching Codex's multi-agent story. All three major CLIs now have some form of multi-agent capability within a week of each other. The convergence is striking: Claude Code has managed agents and task delegation, Codex CLI has MCP parallel-call and subagent TOML, and now Gemini CLI has native `@agent` spawning. The implementation details differ but the user-facing promise — "delegate subtasks to specialist agents" — is identical across all three.
