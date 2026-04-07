---
title: "Codex CLI as an MCP Server: Orchestrating Agents with the OpenAI Agents SDK"
layout: single
date: 2026-03-26
tags: [agents-sdk, mcp, orchestration, multi-agent, agentic-pod]
---

![Sketchnote](/sketchnotes/articles/2026-03-26-codex-cli-agents-sdk-integration.png)

# Codex CLI as an MCP Server: Orchestrating Agents with the OpenAI Agents SDK

*Published 2026-03-26. Based on the [OpenAI Cookbook tutorial](https://developers.openai.com/cookbook/examples/codex/codex_mcp_agents_sdk/building_consistent_workflows_codex_cli_agents_sdk) — "Building Consistent Workflows with Codex CLI & Agents SDK".*

---

## The Pattern

Expose Codex CLI as a long-running **MCP server**, then orchestrate it from the OpenAI Agents SDK. This unlocks:

- **Deterministic code generation** routed through specialised agent roles
- **Full traceability** via SDK Traces (prompts, tool calls, handoffs, timelines)
- **Scalable multi-agent pipelines** with gating logic between stages

```bash
# Codex CLI starts as an MCP server — the SDK connects to it
codex mcp-server
```

The MCP server exposes two tools:

- `codex()` — initiate a new Codex conversation
- `codex-reply()` — continue an existing conversation

---

## Single-Agent Architecture

The simplest pattern: one Designer Agent, one Developer Agent, sequential handoff.

```python
from agents import Agent, MCPServerStdio
import asyncio

async def main():
    async with MCPServerStdio(
        command="codex",
        args=["mcp-server"],
        client_session_timeout_seconds=360000  # long timeout for agentic tasks
    ) as codex_mcp:

        designer = Agent(
            name="Designer",
            instructions="Brainstorm a specification for the requested feature.",
            mcp_servers=[codex_mcp]
        )

        developer = Agent(
            name="Developer",
            instructions="""Implement the spec provided.
            Use approval-policy: never and sandbox: workspace-write.
            Write all files before returning.""",
            mcp_servers=[codex_mcp]
        )

        # Designer → spec → Developer → implementation
        spec = await designer.run("Build a REST API for user authentication")
        await developer.run(spec.output)
```

**Key config for the Developer agent:**

```python
codex_params = {
    "approval-policy": "never",
    "sandbox": "workspace-write"
}
```

---

## Multi-Agent Orchestration with Gating Logic

For complex projects, a **Project Manager agent** coordinates specialised roles and enforces quality gates before advancing.

```python
project_manager = Agent(
    name="Project Manager",
    instructions="""
    You coordinate a team of specialist agents:
    - Designer: product spec + architecture
    - Frontend Developer: React/TypeScript components
    - Backend Developer: API + database layer
    - Tester: unit and integration tests

    GATING RULES:
    1. Do not advance to Frontend until design artefacts are present in /design/
    2. Do not advance to Backend until API contract is defined in /api/schema.json
    3. Do not advance to Tester until both Frontend and Backend hand off
    4. Do not return final output until all tests pass

    Each specialist receives: their role, the overall spec, and only the files relevant to their work.
    """,
    agents=[designer, frontend_dev, backend_dev, tester]
)
```

### The `RECOMMENDED_PROMPT_PREFIX` Pattern

Each specialist agent should receive scoped context to reduce hallucination and improve handoffs:

```python
RECOMMENDED_PROMPT_PREFIX = """
You are {role}.
Project context: {spec_summary}
Your input artefacts: {relevant_files}
Your output artefacts: {expected_outputs}
Do not modify files outside your scope.
"""
```

This is the single biggest quality improvement in multi-agent Codex pipelines — agents with scoped context produce tighter, more consistent output than agents with full project context.

---

## Observability with SDK Traces

The SDK automatically captures a full trace for every run:

| Trace field | What it contains |
|-------------|-----------------|
| Prompts | All agent instructions + user messages |
| Tool invocations | Every `codex()` / `codex-reply()` call |
| Handoffs | When and what was passed between agents |
| Execution timeline | Duration per step |
| File artefacts | Files created/modified by each agent |

Access traces:

```python
from agents import trace

with trace("feature-build"):
    result = await project_manager.run("Build user auth feature")
    # Full trace available in OpenAI dashboard and locally
```

**Why traces matter for Daniel's agentic pod:** When a multi-agent run fails mid-way (the Developer's code doesn't compile, the Tester finds regressions), traces let you identify exactly which agent made the bad decision and what context it had.

---

## Main Use Cases

| Use case | Agents involved | Gating |
|----------|----------------|--------|
| **Large-scale refactoring** | PM → Developer(s) → Tester | Tests pass before merge |
| **Feature rollout across repos** | PM → N Developer agents (parallel) | Each repo passes its own test suite |
| **Framework migration** | Analyser → Developer → Tester | Checkpoint per module |
| **Parallel frontend/backend dev** | PM → Frontend + Backend (concurrent) → Tester | API contract must exist before both start |

---

## Connection to Daniel's Agentic Pod

This pattern is the architecture foundation for the "agentic pod" — multiple Codex agents with roles, coordinated by an orchestrator. Key mapping:

| Agentic Pod Role | SDK Agent |
|-----------------|-----------|
| Planner / Tech Lead | Project Manager agent with gating rules |
| Feature Developer | Developer agent with scoped MCP tools |
| QA | Tester agent, output-gated |
| DevOps | CI agent using `codex exec` (non-interactive) |
| Reviewer | Designer/Architect agent pre-handoff |

The `SKILL.md` files in your repo map to the `instructions` field in each SDK Agent — you're already building in this direction.

---

## See Also

- [Official Cookbook tutorial](https://developers.openai.com/cookbook/examples/codex/codex_mcp_agents_sdk/building_consistent_workflows_codex_cli_agents_sdk)
- [Codex CLI MCP Integration](/codex-resources/articles/2026-03-26-codex-cli-mcp-integration/)
- [Codex CLI Subagents: TOML Format & Parallelism](/codex-resources/articles/2026-03-26-codex-cli-subagents-toml-parallelism/)
- [Agentic Pod Roles and Codex](/codex-resources/articles/2026-03-26-agentic-pod-roles-and-codex/)
