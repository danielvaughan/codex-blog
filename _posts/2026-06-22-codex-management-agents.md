---
title: "Codex Management Agents: Orchestrating Work Without a Human in Every Loop"
description: "As Codex agents grow more capable, the bottleneck shifts from doing the work to managing it. Management agents solve this by coordinating task assignment, progress tracking, and quality gates across fleets of worker agents."
parent: "Articles"
nav_order: 1186
date: 2026-06-22T12:00:00+00:00
last_modified_at: 2026-06-26T10:39:58+01:00
tags:
  - management-agents
  - orchestration
  - agents-sdk
  - symphony
  - multi-agent
  - automation
  - architecture
---

![Sketchnote diagram for: Codex Management Agents: Orchestrating Work Without a Human in Every Loop](/sketchnotes/articles/2026-06-22-codex-management-agents.png)

*Published 2026-06-22. Sources: OpenAI Agents SDK documentation, OpenAI Build Hour (May 2026), Symphony repository, Codex CLI changelog.*

---

A single Codex agent can write code, edit files, and run tests for hours without stopping. Two agents can split a task. But when you have ten agents working across ten branches on ten different issues, something has to hold the shape of the work together. That something is a management agent.

---

## The Problem: Agent Sprawl

Every team that scales past a handful of Codex sessions hits the same wall. The agents are productive — the problem is coordination. Who picks up the next issue? Which agent's output needs review first? When two agents touch the same module, who resolves the conflict? When one agent finishes early, does it sit idle or pull from a queue?

These are management problems, and they are older than software. What is new is that the manager can also be an agent.

---

## What a Management Agent Does

A management agent is not a smarter worker. It is a different kind of agent with a different job. Its tools are not code editors and test runners — they are task boards, status updates, assignment functions, and approval gates.

Concretely, a management agent:

1. **Watches a task queue** — pulling new issues from Linear, GitHub Issues, Jira, or a simple database table.
2. **Assigns work to worker agents** — choosing which agent (or agent type) handles each task, based on skills, current load, or domain.
3. **Monitors progress** — checking whether worker agents are stuck, running too long, or producing output that fails validation.
4. **Enforces quality gates** — requiring that tests pass, that a human approves a tool call, or that a second agent reviews the output before marking a task complete.
5. **Handles handoffs** — routing a task from one agent to another when the work crosses domains. A code agent finishes implementation; a documentation agent picks up the changelog entry.
6. **Reports status** — summarising what the fleet has done, what is in progress, and what is blocked, so a human can glance at a dashboard rather than watching every session.

None of these require the management agent to understand the content of the work. It manages the flow, not the code.

---

## The Agents SDK Pattern

OpenAI's Agents SDK, updated in May 2026, provides the primitives for building this.[^1] The key architectural idea is **splitting the harness from the compute**. The harness — the loop that calls the LLM, manages context, and routes tool calls — runs in your infrastructure. The compute — the sandbox where the agent writes and executes code — runs in an ephemeral container (Docker, Modal, E2B, Cloudflare, or others).

This split matters for management agents because:

- **The management agent runs in the harness layer.** It does not need a sandbox. It needs tools that talk to your task tracker, your database, and the sandbox control plane.
- **Worker agents run in sandboxes.** They are isolated, ephemeral, and cheap to spin up or tear down.
- **State survives sandbox death.** The Agents SDK snapshots the file system and conversation rollout as JSON. If a sandbox expires, the management agent can rehydrate it on a new container and resume from where it left off.[^1]

The management agent's tools are standard `FunctionTool` definitions — Python or TypeScript functions decorated so the SDK can translate them into API tool schemas. A `search_assignees` tool lists available agents. An `update_status` tool moves a task through stages. An `assign_task` tool hands a task to a specific worker.[^1]

```python
@function_tool
def assign_task(task_id: str, agent_name: str) -> str:
    """Assign a task to a worker agent and start its sandbox."""
    task = db.get_task(task_id)
    task.assignee = agent_name
    task.status = "in_progress"
    db.save(task)
    return f"Task {task_id} assigned to {agent_name}"
```

The management agent calls these tools through the same LLM loop as any other agent. The difference is that its instructions tell it to coordinate rather than implement.

---

## Handoffs: The Coordination Primitive

The Agents SDK supports agent handoffs natively.[^1] A worker agent can reassign a task to another agent when it finishes its part. In the Build Hour demo, a "program editor" agent refined content and then handed off to an "asset producer" agent to generate visual assets — all without human intervention.

For management agents, handoffs work in both directions:

- **Downward**: the management agent assigns a task to a worker.
- **Lateral**: one worker hands off to another (editor to asset producer, implementation to documentation).
- **Upward**: a worker signals completion back to the management agent, which then decides what happens next — mark done, request review, or assign a follow-up task.

This creates a natural hierarchy without rigid coupling. The management agent does not need to know the internal workings of each worker. It only needs to know their capabilities and their current status.

---

## Symphony: Management at Scale

OpenAI's Symphony project demonstrates this pattern at production scale.[^2] Symphony watches a Linear board for issues in the "Ready for Agent" state. When it finds one, it spawns an isolated Codex agent, assigns the issue, and monitors progress. The agent works autonomously — writing code, running tests, iterating on failures. When it believes the work is done, it opens a pull request. A human reviews the PR, not the agent session.

Symphony's management logic handles:

- **Queueing**: issues are picked up in priority order.
- **Isolation**: each agent works in its own sandbox with its own branch.
- **Proof of work**: agents must demonstrate passing tests before surfacing results.
- **Failure handling**: if an agent gets stuck after a defined number of iterations, the issue is returned to the board for human triage.

This is a management agent that happens to be implemented as an orchestration service rather than as an LLM agent. But the Agents SDK now makes it possible to build the same thing where the orchestrator itself is an LLM — capable of making judgement calls about priority, routing, and quality that a static rule engine cannot.

---

## Tool Call Approvals: The Human-in-the-Loop Gate

Not every decision should be autonomous. The Agents SDK supports **tool call approvals**, where certain tool invocations require human confirmation before executing.[^1] For management agents, this is how you keep humans in the loop without requiring them to watch every session.

```python
@function_tool(
    requires_approval=lambda args: args.get("status") == "done"
)
def update_status(task_id: str, status: str) -> str:
    """Update a task's status. Requires approval to mark as done."""
    ...
```

The management agent can assign tasks, monitor progress, and handle handoffs autonomously. But marking a task as truly complete — shipping it — requires a human to approve. This is a practical middle ground: the agent manages 95% of the workflow, and the human provides the final quality gate.

---

## Skills: Giving Workers Domain Knowledge

The Agents SDK's skills system lets you package domain knowledge as bundles of instructions and scripts that agents load at startup.[^1] A management agent can decide which skills a worker needs based on the task type.

Skills can be stored in a Git repository or uploaded via the Skills API, versioned, and referenced by name.[^1] A management agent that understands its fleet's skill inventory can make better assignment decisions: route a data-processing task to an agent loaded with the "data-pipeline" skill, and a frontend task to one with the "react-components" skill.

This is where management agents become more than glorified cron jobs. An LLM-based manager can read a task description, match it against skill descriptions, and make a judgement call about which agent is best suited — something a static routing table cannot do when tasks are described in natural language.

---

## Memory: Learning from Past Runs

The Agents SDK now includes agent memory, allowing tasks to improve over time.[^1] For management agents, memory serves a different purpose: it accumulates operational knowledge. Which types of tasks tend to get stuck? Which agents perform better on which domains? How long do certain task categories typically take?

A management agent with memory can adjust its behaviour — assigning more time for tasks that historically overrun, escalating to humans earlier for task types with high failure rates, and preferring certain agent configurations for certain domains.

---

## Practical Architecture

A minimal management agent system has three components:

1. **Task store**: a database or API that holds the task queue, statuses, and assignments. This could be Linear, GitHub Issues, or a simple PostgreSQL table.
2. **Management agent**: an Agents SDK agent with tools for reading the task store, assigning tasks, checking status, and enforcing gates. It runs in the harness layer with no sandbox.
3. **Worker agents**: sandbox agents with domain-specific skills, created on demand by the management agent. Each gets its own ephemeral container.

The management agent runs continuously or on a schedule. When it detects new tasks, it creates worker agents, assigns tasks, and monitors progress. When workers complete tasks, it handles the output — opening PRs, updating status, or triggering the next stage.

```
┌─────────────────────────────────────────┐
│           Management Agent              │
│  (harness layer, no sandbox)            │
│  Tools: assign, monitor, approve, route │
└─────────┬──────────┬──────────┬─────────┘
          │          │          │
    ┌─────▼───┐ ┌────▼────┐ ┌──▼──────┐
    │ Worker  │ │ Worker  │ │ Worker  │
    │ Agent A │ │ Agent B │ │ Agent C │
    │ (Modal) │ │ (E2B)   │ │ (Docker)│
    └─────────┘ └─────────┘ └─────────┘
```

The management agent can run on different infrastructure from the workers. It might run in a Temporal job or a long-lived server process, while workers spin up and down on Modal or E2B as demand requires.

---

## When You Don't Need a Management Agent

Not every team needs this. If you have one or two Codex sessions running at a time, a human can easily manage them. Management agents become valuable when:

- You have **more concurrent agent tasks than humans available to supervise them**.
- Tasks follow **predictable workflows** with clear stages and handoff points.
- You need **audit trails** of what was assigned, when, and what the outcome was.
- Worker agents operate on **different schedules** (some tasks take minutes, others take hours) and you need something to watch the clock.

If your agent usage is ad hoc and exploratory, the overhead of a management layer is not worth it. If your agent usage is systematic and recurring, it almost certainly is.

---

## Where This Is Heading

The Agents SDK team has signalled that multi-agent frameworks are coming in the near weeks and months.[^1] Today, building a management agent means writing the coordination logic yourself with function tools and handoffs. In the near future, expect higher-level abstractions — supervisor agents that can monitor hundreds of workers simultaneously, shared communication channels between agents, and built-in patterns for common workflows like code review pipelines and deployment chains.

The direction is clear: the unit of work in software is shifting from the pull request to the agent task. Management agents are how organisations will handle that shift without drowning in coordination overhead.

---

## References

[^1]: OpenAI, "Build Hour: Agents SDK," YouTube, May 28, 2026. Covers Agents SDK updates including sandbox agents, harness/compute split, skills, memory, tool call approvals, and handoff patterns.

[^2]: OpenAI, "Symphony," GitHub repository, github.com/openai/symphony. Autonomous agent orchestration system using Linear integration and isolated Codex agents.
