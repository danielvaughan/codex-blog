---
title: "Codex Management Agents: Orchestration Patterns for Multi-Agent Systems"
parent: "Articles"
nav_order: 1541
date: 2026-06-22T00:00:00+00:00
last_modified_at: 2026-08-13T00:10:34+01:00
author: Daniel Vaughan
description: "How OpenAI's Agents SDK enables management agents that coordinate, delegate, and supervise specialist workers — and why the patterns matter more than the framework."
tags: [codex, agents-sdk, multi-agent, orchestration, management, handoffs]
type: Technical Article
timestamp: 2026-06-22
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-06-22-codex-management-agents-orchestration-patterns"
---
# Codex Management Agents: Orchestration Patterns for Multi-Agent Systems

A single agent can do useful work. But production systems rarely need just one. They need a manager — something that triages incoming tasks, delegates to specialists, monitors progress, and decides when the job is done. OpenAI's Agents SDK now provides first-class support for these management agents, drawing heavily from the patterns that power Codex itself.

This article examines the two primary orchestration patterns available today, when each one fits, and the practical trade-offs that determine which you should reach for.

## The Problem Management Agents Solve

Consider a conference planning system. Tasks arrive: refine a speaker's abstract, produce programme assets, upload content to a website. Each task requires different skills and different tools. A monolithic agent armed with every tool and every instruction would drown in context — its prompt would bloat, its tool-selection accuracy would drop, and its failure modes would become unpredictable.

The alternative is specialisation. Give each agent a narrow mandate — one edits text, another generates images, a third manages the website — and then place a management agent above them to route work. This is not a new idea. It is how organisations work, and it is how Codex itself operates internally, spinning up sub-agents for discrete tasks while a central loop maintains oversight.[^1]

The Agents SDK makes this pattern explicit with two mechanisms: **handoffs** and **agents-as-tools**.

## Pattern One: Handoffs (Decentralised Delegation)

A handoff transfers conversational control from one agent to another. The originating agent steps aside; the receiving agent takes over the conversation entirely. Think of it as passing a baton in a relay race — only one runner moves at a time.

```python
from agents import Agent, handoff

editor_agent = Agent(
    name="Programme editor",
    instructions="Edit documents for clarity and house style."
)

asset_agent = Agent(
    name="Asset producer",
    instructions="Generate programme assets from edited content."
)

triage_agent = Agent(
    name="Triage agent",
    instructions=(
        "You are a conference planning coordinator. "
        "Route editing tasks to the programme editor. "
        "Route asset creation to the asset producer."
    ),
    handoffs=[editor_agent, asset_agent]
)
```

Under the hood, each handoff is represented as a tool. When the triage agent decides to hand off to the editor, it calls `transfer_to_programme_editor`. The SDK handles the plumbing: swapping the active agent, optionally filtering conversation history, and executing any `on_handoff` callback you provide.[^2]

### When handoffs fit

Handoffs work well when tasks are **sequential and self-contained**. Agent A finishes its portion, then Agent B takes over with full conversational context. The pattern suits customer service flows (triage, then billing, then escalation) and document pipelines (draft, then edit, then publish).

### When handoffs struggle

Handoffs are less natural when you need the manager to **retain control** throughout. Once a handoff occurs, the original agent loses the thread. If you need a coordinator that dispatches work to three specialists simultaneously and then synthesises their outputs, handoffs alone will not get you there.

## Pattern Two: Agents as Tools (Centralised Management)

The second pattern keeps the manager in charge. Instead of handing off control, the manager **invokes specialist agents as tools** — calling them like functions, receiving their output, and deciding what to do next.

```python
customer_agent = Agent(
    name="Customer-facing agent",
    instructions="Handle all direct user communication. Call tools when needed.",
    tools=[
        editor_agent.as_tool(
            tool_name="programme_editor",
            tool_description="Edits documents for clarity and style."
        ),
        asset_agent.as_tool(
            tool_name="asset_producer",
            tool_description="Generates programme assets from content."
        )
    ]
)
```

Here, the customer-facing agent remains the primary decision-maker. It can call the programme editor, inspect the result, then call the asset producer with the edited content — all within a single conversational turn. The manager never relinquishes control.[^3]

### When agents-as-tools fit

This pattern suits **fan-out/fan-in** workflows. A research agent that queries three data sources in parallel, then synthesises findings. A code review agent that runs linting, security scanning, and test coverage checks, then produces a unified report. Any scenario where a central intelligence needs to coordinate multiple results.

### When agents-as-tools struggle

The overhead is higher. Every sub-agent call is a tool invocation, which means the manager's context window grows with each result. For simple linear pipelines where Agent A's output feeds directly into Agent B, the handoff pattern is leaner.

## Combining Both Patterns

The two patterns are not mutually exclusive. A well-designed system often layers them:

```python
# A triage agent hands off to a manager
# The manager uses specialists as tools

specialist_a = Agent(name="Data analyst", ...)
specialist_b = Agent(name="Report writer", ...)

manager = Agent(
    name="Project manager",
    instructions="Coordinate analysis and reporting.",
    tools=[
        specialist_a.as_tool(...),
        specialist_b.as_tool(...)
    ]
)

triage = Agent(
    name="Triage",
    instructions="Route project requests to the project manager.",
    handoffs=[manager]
)
```

The triage agent uses handoffs for coarse routing. The project manager uses agents-as-tools for fine-grained coordination. This mirrors how organisations actually work: a receptionist routes your call, then the project lead manages the specialists.

## The Sandbox Dimension

Management agents become considerably more powerful when combined with the SDK's sandbox capabilities. Each specialist can operate in an isolated container — a `SandboxAgent` with its own file system, shell access, and installed packages. The manager dispatches work; each specialist executes in its own environment.[^4]

This matters for three reasons:

1. **Security isolation.** Secrets stay on the harness side. The sandbox sees only the files it needs. A prompt injection attack on one specialist cannot exfiltrate credentials from the manager.

2. **State independence.** Each sandbox snapshots its file system when a task pauses. If a container dies, the manager can rehydrate a new one from the snapshot and resume without data loss.

3. **Parallel execution.** Multiple sandboxes can run simultaneously across different providers — Docker locally, Modal or E2B in production — while the manager coordinates from a single orchestration loop.

The architectural insight here is the **separation of harness from compute**. The management agent runs the orchestration loop (the "harness"), while specialist agents execute in ephemeral containers (the "compute"). This split, borrowed directly from Codex's own architecture, means the manager is never at risk from a sandbox failure.[^5]

## Skills: Giving Specialists Expertise

A management agent is only as good as its specialists, and specialists are only as good as their instructions. The SDK's skills system addresses this by packaging domain expertise into portable bundles.

A skill is a directory containing a `SKILL.md` file (instructions in markdown with YAML frontmatter) plus any supporting scripts or resources. Skills can be loaded from a Git repository, the OpenAI Skills API, or local files:

```python
from agents import SandboxAgent, SkillsCapability

agent = SandboxAgent(
    name="Tax preparer",
    capabilities=[
        SkillsCapability(
            source="git",
            repo="myorg/tax-skills",
            branch="main"
        )
    ]
)
```

For management agents, this means each specialist can be equipped with versioned, reviewable expertise. The programme editor loads its house style guide from Git. The asset producer loads its brand guidelines. When the style guide changes, you update the skill file and the next task picks it up — no agent code changes required.[^6]

## Human-in-the-Loop: Tool Call Approvals

Not every decision should be automated. The SDK provides a `requires_approval` parameter on function tools that gates execution on human review:

```python
@function_tool(requires_approval=lambda ctx, args:
    args.get("status") == "done"
)
def update_status(task_id: str, status: str):
    """Update a task's status."""
    ...
```

In a management agent context, this creates a natural escalation path. The manager coordinates routine work autonomously but pauses for human sign-off on high-stakes actions — publishing to production, closing a customer case, approving a financial transaction. The agent does not proceed until approval arrives.[^7]

## Practical Considerations

### Context window management

Management agents accumulate context from every specialist interaction. The SDK's compaction capability addresses this by allowing the agent to summarise and compress its conversation history, enabling runs that span hours or days without hitting token limits.

### Choosing a sandbox provider

For local development, Docker is the path of least resistance. For production, Modal, E2B, Cloudflare Workers, and Vercel are all supported with first-class sandbox clients. The choice depends on your existing infrastructure — the SDK abstracts the provider behind a common interface.

### Session persistence

The SDK stores both the conversation rollout (a JSON object) and the file system snapshot. Together, these allow a management agent to pause, persist its state to a database, and resume on a different node. For multi-tenant systems processing many users concurrently, this is essential.

### Debugging multi-agent flows

The SDK includes built-in tracing that captures every tool call, handoff, and LLM interaction. When a specialist produces unexpected output, the trace shows exactly what the manager sent, what the specialist received, and where the flow diverged.

## What This Means for Codex Users

If you have used Codex, you have already experienced management agents in action. Codex's internal architecture — a central loop that dispatches sub-agents, manages context compaction, snapshots file system state, and resumes across sessions — is precisely what the Agents SDK now exposes as a programmable surface.

The difference is that Codex is a coding agent. The Agents SDK lets you build management agents for any domain: legal document review, customer support triage, data pipeline orchestration, conference planning. The patterns are the same; the domain knowledge changes.

The models are getting better at long-horizon work — internally, OpenAI has run Codex tasks for days.[^5] Management agents are how you harness that capability at scale: not by building one enormous agent, but by building a system of specialists coordinated by a manager that knows when to delegate, when to intervene, and when to step back.

---

[^1]: OpenAI, "Agents SDK Documentation," 2026. https://openai.github.io/openai-agents-python/
[^2]: OpenAI, "Handoffs — Agents SDK," 2026. https://openai.github.io/openai-agents-python/handoffs/
[^3]: OpenAI, "Agents — Agents SDK," 2026. https://openai.github.io/openai-agents-python/agents/
[^4]: OpenAI, "Agents SDK Guide," 2026. https://developers.openai.com/api/docs/guides/agents-sdk
[^5]: OpenAI, "Build Hour: Agents SDK," YouTube, May 28, 2026. https://www.youtube.com/watch?v=tK32trvj_b4
[^6]: OpenAI, "Skills — Agents SDK," 2026. https://openai.github.io/openai-agents-python/
[^7]: OpenAI, "Build Hour: Agents SDK," YouTube, May 28, 2026. https://www.youtube.com/watch?v=tK32trvj_b4
