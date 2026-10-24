---
title: "Persistent Cloud Sessions: What 'Tasks Run After the Laptop Closes' Actually Means"
date: 2026-10-21T08:00:00+00:00
last_modified_at: 2026-10-24T07:06:46+01:00
parent: "Articles"
nav_order: 1194
tags:
  - codex-cli
  - codex-cloud
  - persistent-sessions
  - agents
  - supervision
  - agents-md
  - hooks
  - safety
copy_edited: true
---

# Persistent Cloud Sessions: What "Tasks Run After the Laptop Closes" Actually Means

![Sketchnote diagram for: Persistent Cloud Sessions: What 'Tasks Run After the Laptop Closes' Actually Means](/sketchnotes/articles/2026-10-21-persistent-cloud-sessions-tasks-run-after-laptop-closes-codex-cli.png)

**Release track:** Codex Cloud (generally available)
**Related feature:** v0.164.0-alpha observation hooks, Codex Dots, 12-hour prompt caching (API, 20 October 2026)
**Impact:** All users running long-horizon tasks; anyone designing unattended Codex workflows

---

In "Cooking with Codex: Ep. 1," OpenAI presents persistent cloud environments as a casual convenience feature — your Codex task keeps running after the laptop lid goes down.[^1] The framing is deliberately low-key: close the screen, come back later, the work is done. For a mainstream audience, that is an accurate and appealing summary. For a practitioner designing production workflows, it raises several questions that the episode does not answer.

This article answers them. What exactly persists in a Codex Cloud session? What is the cost model? What can and cannot an unsupervised Codex agent do while you are away? What happens when a task hits an unexpected branch? And how do you configure AGENTS.md and hooks so that unattended sessions fail safely rather than silently?

---

## What "persistence" actually means

A Codex Cloud session is not a running process on your laptop. When you start a task in Codex Cloud, the execution environment is provisioned on OpenAI's infrastructure. Your local Codex CLI client connects to it, displays output, and accepts input — but the computation happens remotely. Closing your laptop disconnects the client. The remote environment keeps running.

Three things persist across the disconnect:

**The execution environment.** The sandbox — including the cloned repository, installed dependencies, and any files written during the session — remains intact. When you reconnect, the working tree is exactly as it was when you closed the lid. No state is lost unless the session hits its configured timeout or a hard resource limit.

**The task context.** Codex retains the conversation history, tool call log, and any in-progress plan for the duration of the session. On reconnect, the agent resumes from where it stopped. It does not restart; it continues. This is distinct from the `--continue` flag in local CLI, which reattaches to the most recent local session — Codex Cloud handles continuation automatically.

**Queued tool calls.** If Codex was mid-operation when you disconnected — running a test suite, waiting for a shell command to complete — it continues waiting. The tool call is not abandoned. You may reconnect to find the agent several steps further through its plan than when you left.

What does *not* persist: your local file system. Any files written by the cloud session exist in the remote sandbox. You pull them back by committing and pushing, or by using `codex pull` when the session completes. The session is not a mirror of your local environment; it is a separate working copy.

---

## The cost model

Local Codex CLI incurs token costs for model calls and, in some configurations, tool execution costs for MCP servers. There is no persistent compute charge — when the CLI exits, nothing is running.

Codex Cloud adds a compute dimension. The remote sandbox is billed for the duration of the session, not just for active model calls. A 6-hour session running overnight carries 6 hours of compute at the applicable Codex Cloud rate, regardless of whether the agent was actively generating tokens for all of that time.

The practical implication: always scope tasks before starting a cloud session. A task that "explores the codebase and suggests improvements" has an unbounded runtime. A task that "adds error handling to the three functions listed in TASK.md, writes a test for each, and opens a draft PR" has a defined endpoint. The second task terminates; the first may run until the session timeout (currently 24 hours by default for Codex Cloud tasks).

The 12-hour prompt caching introduced on 20 October 2026 intersects here.[^2] For long-running sessions that repeatedly reference a large, stable context — a long AGENTS.md, a shared architecture document, a multi-thousand-line specification — enabling 12-hour caching for the static portion of the prompt reduces the cached-input rate for that content. Mixing 12-hour caching for instructions with 30-minute defaults for dynamic content (tool outputs, intermediate results) is explicitly supported and reduces total session cost on extended tasks.

---

## The supervision gap

When you close the laptop, you cede real-time oversight. Codex will continue to:

- Read and write files in the remote sandbox
- Execute shell commands (within the sandbox's permissions)
- Make model calls
- Call any MCP tools configured for the session
- Commit to local branches within the sandbox
- Open or update pull requests if configured to do so

It will not, by default, push directly to protected branches or deploy to production environments — but "by default" is doing significant work in that sentence. If your AGENTS.md includes a `post-task` hook that runs `git push origin main`, or if your MCP server exposes a deployment tool, and you have not constrained them explicitly, an unattended agent will use them.

The supervision gap is not a bug. It is the feature. The design intent is that Codex completes the task. The responsibility for defining what "completes the task" safely falls to the practitioner who writes the AGENTS.md and configures the session.

---

## Failure modes

Three failure modes are worth designing against explicitly.

**Unexpected decision branches.** Codex encounters a condition not anticipated in the task description — an ambiguous test failure, a merge conflict, a missing dependency — and must choose between continuing with a best-guess resolution or stopping for clarification. Without a configured `on-uncertainty` policy, many agents default to the best-guess path. On a supervised session, you catch this in real time. Overnight, a best-guess resolution that is subtly wrong can propagate through 20 subsequent steps before you reconnect.

**Silent tool failures.** A shell command exits with a non-zero code. An MCP server returns an error response. Depending on how the tool call is handled, the agent may log the failure, note it in its plan, and continue — treating the failed step as recoverable context rather than a stopping condition. If no hook validates the output of critical steps, failure is silent from the practitioner's perspective.

**Session timeout without completion.** A 24-hour session timeout sounds generous, but under-scoped tasks and recursive loops can exhaust it. When a session hits its timeout, the sandbox is snapshotted and the session terminates. Work completed to that point is preserved; work not yet committed is in the snapshot but requires explicit recovery. There is no automatic notification to your email or Slack unless you have configured a `post-session` webhook.

---

## Designing for persistence: AGENTS.md patterns

The AGENTS.md file is the primary tool for making unattended sessions safe. Four patterns apply directly to persistent sessions.

**Define a completion criterion, not just a goal.** Instead of:

```text
Goal: Refactor the authentication module.
```

Write:

```text
Goal: Refactor the authentication module.
Completion criterion: All tests in auth/__tests__/ pass with `npm test -- --testPathPattern=auth`.
  No new TypeScript errors in `tsc --noEmit`. PR opened as draft against `main`.
Stopping condition: Stop and leave a note in TASK-STATUS.md if any test was passing
  before the refactor and is now failing after two resolution attempts.
```

The stopping condition is the critical addition. It defines a condition under which the agent should stop and preserve state rather than continuing to a potentially broken commit.

**Scope tool permissions explicitly.** If the task does not require MCP server access, disable it for the session:

```yaml
tools:
  mcp: false
  shell: true
  editor: true
```

If the task requires specific MCP tools, list them:

```yaml
tools:
  mcp:
    allowed: [filesystem, github]
    denied: [deploy, production-db]
```

Permission minimisation applies to unattended sessions at least as forcefully as it applies to supervised ones.

**Write task status to a file at each major step.** Hooks can write structured progress to a file that you can inspect on reconnect:

```bash
# post-step hook example
echo "{\"step\": \"$CODEX_STEP_NAME\", \"status\": \"$CODEX_STEP_STATUS\", \"ts\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}" >> TASK-LOG.jsonl
```

When you reconnect after 8 hours, `cat TASK-LOG.jsonl` gives you a structured account of what happened. Without this, you reconstruct the session from the agent's prose narration, which is less reliable.

**Set a conservative token budget per step.** Recursive or exploratory tasks can burn tokens on intermediate work that contributes little to the goal. Setting a per-step budget in AGENTS.md forces the agent to be economical:

```yaml
max_tokens_per_step: 4000
on_budget_exceeded: stop_and_summarise
```

This is a blunt instrument — some steps legitimately require more — but it is a useful backstop against runaway exploration.

---

## Observation hooks in v0.164.0

The observation hooks introduced in v0.164.0-alpha (alpha.5, 19 October 2026) add a read-only interception layer at each tool call.[^3] An observation hook receives the tool call context — tool name, arguments, prior output — and can log it, send it to a monitoring endpoint, or trigger an alert. It cannot block or modify the call (that is validation hooks territory), but it gives you a real-time audit trail without requiring reconnection.

For persistent sessions, the practical pattern is a lightweight observation hook that writes structured logs to a GCS bucket or a monitoring webhook. When something looks wrong — a file was deleted that should not have been, a command was run outside the expected path — you have a timestamped record from the hook log rather than a reconstructed narrative.

A minimal observation hook in AGENTS.md:

```bash
# hooks.observation
curl -s -X POST "$MONITORING_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d "{\"tool\": \"$CODEX_TOOL_NAME\", \"args\": $CODEX_TOOL_ARGS_JSON, \"session\": \"$CODEX_SESSION_ID\", \"ts\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}" \
  > /dev/null
```

This adds minimal latency (one outbound HTTP call per tool use) and gives you a queryable log of every action taken while you were away.

---

## The practical checklist

Before starting a Codex Cloud session you intend to leave running:

1. **Task has a completion criterion**, not just a goal
2. **Task has a stopping condition** for at least one foreseeable ambiguity
3. **Tool permissions are explicitly scoped** — no unnecessary MCP servers enabled
4. **A post-step logging hook is configured** — TASK-LOG.jsonl or equivalent
5. **The task is bounded** — a time estimate exists; tasks expected to run longer than 4 hours have intermediate checkpoints
6. **Prompt caching is enabled** for any static context longer than ~2,000 tokens
7. **A completion notification is configured** — webhook, email, or Slack message on session end

The "laptop closed" framing in "Cooking with Codex: Ep. 1" is accurate. The missing context is that it works well when the task is well-defined, the permissions are constrained, and the failure surface is auditable. Without those three conditions, "tasks run after the laptop closes" can mean "Codex spent 6 hours making confident mistakes with no one watching."

---

## References

[^1]: OpenAI. "Cooking with Codex: Ep. 1." YouTube, 19 October 2026. <https://www.youtube.com/watch?v=DQHtmW9ZlIs>
[^2]: OpenAI. "12-Hour Prompt Caching in the API." OpenAI Developer Community, 20 October 2026. 28 Days of Shipping Day 16.
[^3]: Codex CLI v0.164.0-alpha.5, PR #55144 — "Add observation-only model request lifecycle hooks." GitHub, 18 October 2026.
