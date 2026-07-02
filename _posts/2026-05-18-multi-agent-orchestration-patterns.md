---
title: "Multi-Agent Orchestration Patterns for Codex CLI"
description: "Published: 2026-05-18 Source: Addy Osmani — The Code Agent Orchestra ("
date: 2026-05-18T00:00:00+00:00
last_modified_at: 2026-07-02T08:14:08+01:00
---

![Sketchnote diagram for: Multi-Agent Orchestration Patterns for Codex CLI](/sketchnotes/articles/2026-05-18-multi-agent-orchestration-patterns.png)

# Multi-Agent Orchestration Patterns for Codex CLI

Published: 2026-05-18
Source: Addy Osmani — "The Code Agent Orchestra" (https://addyosmani.com/blog/code-agent-orchestra/)

## Summary

A synthesis of Addy Osmani's analysis of what makes multi-agent coding work, mapped to Codex CLI capabilities.

## Three Coordination Patterns

### 1. Subagents (Simple Delegation)
- Parent decomposes work, spawns focused children with specific briefs
- No peer messaging — parent collects all results
- Codex CLI native: use custom agents in `~/.codex/agents/` or `.codex/agents/`
- Cost: ~220k tokens for a three-agent task

### 2. Agent Teams (True Parallelism)
- Shared task list with dependency tracking
- Peer-to-peer messaging between teammates
- Sweet spot: 3-5 agents; costs scale linearly
- In Codex: combine subagents with git worktrees for isolation

### 3. Hierarchical Subagents
- Feature leads spawn their own specialists
- Mimics real org structure — prevents context fragmentation
- Codex: set `max_depth = 2` (carefully) to allow one level of sub-delegation

## Four Essential Quality Gates

1. **Plan approval** — agents write implementation plans before coding; review before code exists
2. **Loop guardrails** — set max iterations with forced reflection on failure
3. **Dedicated reviewer agent** — read-only tools, lint/test only, auto-triggered on task completion
4. **Lifecycle hooks** — automated checks: tests must pass, linting before merge

## Key Insight: The Bottleneck Shifted

> "The bottleneck is no longer generation. It's verification."

Agents produce impressive output at speed, but human confidence in correctness is the hard constraint. Quality gates aren't overhead — they're the safety system.

## Practical Recommendations

### AGENTS.md Discipline
- Human-curated only (LLM-generated versions show ~3% performance degradation)
- Document: style, gotchas, architecture decisions, test strategies
- Acts as institutional memory across sessions

### Multi-Model Routing
- Planning/architecture → cheaper models (GPT-5.4 mini)
- Implementation → GPT-5-codex
- Review → specialised reviewer agent with strict instructions

### The Factory Model
- View as a pipeline: Plan → Spawn → Monitor → Verify → Integrate → Retro
- WIP limits: 3-5 concurrent agents
- Kill criteria: stuck after 3+ iterations
- Granular commits on feature branches

## Mapping to Codex CLI

| Osmani's Pattern | Codex CLI Implementation |
|-----------------|--------------------------|
| Subagents | Custom agent TOML files + `spawn_agents_on_csv` |
| Agent teams | Multiple subagents + git worktrees |
| Hierarchical | `max_depth = 2` with feature-lead agents |
| Plan approval | Architect agent reviews before worker agents start |
| Reviewer agent | `sandbox_mode = "read-only"` custom agent |
| Lifecycle hooks | `/hooks` configuration in config.toml |
| Multi-model routing | Per-agent `model` override in TOML |
