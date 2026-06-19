---
title: "The Code Agent Orchestra — Key Insights from Addy Osmani"
description: "Source: Captured: 2026-05-14"
date: 2026-05-14T00:00:00+00:00
last_modified_at: 2026-06-19T08:13:13+01:00
---

![Sketchnote diagram for: The Code Agent Orchestra — Key Insights from Addy Osmani](/sketchnotes/articles/2026-05-14-addy-osmani-code-agent-orchestra.png)

# The Code Agent Orchestra — Key Insights from Addy Osmani

Source: https://addyosmani.com/blog/code-agent-orchestra/
Captured: 2026-05-14

## Summary

Addy Osmani's deep analysis of what makes multi-agent coding work. Essential reading for anyone building agentic pods.

## Core Patterns

### 1. Subagents (Task Decomposition)
Parent orchestrator spawns focused child agents with specific briefs and file ownership. Good for simple decomposition.

### 2. Agent Teams (True Parallelism)
- Shared task list with dependency tracking
- Peer-to-peer messaging between teammates
- File locking to prevent conflicts
- **Optimal team size: 3-5 agents**

### 3. Hierarchical Delegation
Feature leads spawn their own specialists — 3x deeper decomposition without exploding context windows.

## Critical Quality Gates

1. **Plan Approval** — require written plans before implementation. Cheaper to catch architecture bugs before code exists.
2. **Hooks** — automated checks on lifecycle events (tests, linting) that block progression
3. **AGENTS.md** — human-curated institutional memory. LLM-generated AGENTS.md files offer no benefit and can reduce success rates.

## Anti-Patterns

- **Context Overload** — single agents lose important details as conversations grow
- **Unbounded Loops** — set `MAX_ITERATIONS=8` with forced reflection steps
- **Verification Neglect** — generation is easy; verification is the bottleneck

## Architecture Rules

- **One File, One Owner** — never let two agents edit the same file. Git worktrees for isolation.
- **Multi-Model Routing** — planning to cheap models, implementation to mid-tier, review to specialised
- **Self-Improving Sessions** — each session reads AGENTS.md; approved learnings merge back

## The Human Role

> "Strong engineers gain *more* leverage from these tools, not less."

Keep for yourself: architecture/API design, deciding what NOT to build, reviewing agent output with full system context. Spec quality directly multiplies across the entire fleet — vague requirements propagate across parallel runs, each failing differently.
