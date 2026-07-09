---
title: "The Official Codex Subagents Documentation — Architecture, Patterns, and CSV Batch Processing"
description: "OpenAI's official subagents documentation page has matured into a comprehensive reference for multi-agent Codex workflows."
date: 2026-05-06T00:00:00+00:00
last_modified_at: 2026-07-09T20:12:50+01:00
category: reference
tags: [subagents, multi-agent, csv-batch, orchestration, official-docs]
source: https://developers.openai.com/codex/subagents
parent: "Articles"
nav_order: 930
type: Technical Article
timestamp: 2026-05-06T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-05-06-codex-subagents-official-docs-reference-patterns-csv-batch"
---
![Sketchnote diagram for: The Official Codex Subagents Documentation — Architecture, Patterns, and CSV Batch Processing](/sketchnotes/articles/2026-05-06-codex-subagents-official-docs-reference-patterns-csv-batch.png)

# The Official Codex Subagents Documentation

OpenAI's official subagents documentation page has matured into a comprehensive reference for multi-agent Codex workflows. This article captures the key patterns and configuration details as of May 2026.

## Architecture

Codex spawns subagents **only on explicit request** — there is no automatic decomposition. The system manages:
- Spawning new agents with specific roles
- Routing follow-up instructions to the correct thread
- Waiting for all results before consolidating a response
- Closing agent threads on completion

## Built-in Agent Types

Three agents ship by default:

| Agent | Purpose |
|-------|---------|
| `default` | General-purpose fallback |
| `worker` | Execution-focused — implementation and fixes |
| `explorer` | Read-heavy codebase exploration |

## Custom Agent Configuration

Custom agents are TOML files in `~/.codex/agents/` (personal) or `.codex/agents/` (project-scoped).

**Required fields:**
- `name` — agent identifier
- `description` — when Codex should deploy this agent
- `developer_instructions` — core behavioural guidelines

**Optional fields** (inherit from parent if omitted):
- `model`, `model_reasoning_effort`
- `sandbox_mode` (e.g. `"read-only"` for explorers)
- `mcp_servers`
- `skills.config`
- `nickname_candidates` — readable labels for multiple instances

## Global Orchestration Settings

The `[agents]` section in `config.toml` controls:

| Setting | Default | Purpose |
|---------|---------|---------|
| `max_threads` | 6 | Maximum concurrent open threads |
| `max_depth` | 1 | Nesting depth for spawned agents |
| `job_max_runtime_seconds` | — | Timeout per worker for CSV jobs |

**Important:** Raising `max_depth` beyond 1 multiplies resource costs through recursive delegation. Most workflows should stay at depth 1.

## Sandbox and Approval Inheritance

Subagents inherit the parent session's sandbox policy. During interactive sessions:
- Approval requests from inactive threads surface with source labels
- Users press `o` to inspect the thread before approving
- Live runtime overrides (e.g. `/approvals` changes) propagate to child agents
- Individual agents can override sandbox (e.g. `sandbox_mode = "read-only"`)

## CSV Batch Processing (Experimental)

The `spawn_agents_on_csv` tool automates repetitive work across data rows:

```
spawn_agents_on_csv(
  csv_path="tasks.csv",
  instruction="Review {file_path} for security issues and report findings",
  id_column="task_id",
  output_schema={"severity": "string", "findings": "string"},
  output_csv_path="results.csv",
  max_concurrency=4,
  max_runtime_seconds=300
)
```

Workers call `report_agent_job_result` to submit results. The system collects all results into a combined CSV with metadata.

## Documented Use-Case Patterns

### PR Review (Three-Agent Split)
- `pr_explorer` (read-only) — maps codebase, gathers evidence
- `reviewer` — checks correctness, security, test coverage
- `docs_researcher` — validates framework APIs via MCP server

### Frontend Debugging
- `code_mapper` (read-only) — locates relevant code paths
- `browser_debugger` — reproduces issues, captures evidence
- `ui_fixer` — implements targeted fixes

## Thread Management

- CLI: `/agent` to switch threads, inspect ongoing work
- Codex App and CLI: full activity visibility
- IDE Extension: coming soon
- Direct steering: ask Codex to control, stop, or close running subagents

## Token Implications

Each subagent performs independent model and tool work. Token consumption scales linearly with agent count. The docs explicitly note this as a cost consideration.

## What This Means for Agentic Pod Architecture

The official docs now validate several patterns from Daniel's agentic pod research:
1. **Role specialisation** — read-only explorers, execution workers, domain experts
2. **Shallow orchestration** — depth 1 is the recommended default
3. **Project-scoped agents** — `.codex/agents/` enables per-repo agent teams
4. **CSV fan-out** — batch processing pattern for audit/migration workflows

The missing pieces remain: cross-session state sharing, agent-to-agent direct communication, and durable task queues. These gaps are where tools like oh-my-codex, Symphony, and the agent graph store (PR #20689) fill in.

## Cross-references

- [Custom agent TOML definitions](#)
- [CSV fan-out pattern](#)
- [Subagent design divergence: Codex vs Claude](#)
- [Agent graph store PR #20689](#)
