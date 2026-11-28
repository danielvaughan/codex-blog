---
title: "Why a Faster Server Does Not Make Codex Faster: The 2.5% Shell-Execution Finding"
date: 2026-11-26T08:00:00+00:00
last_modified_at: 2026-11-28T13:07:40Z
tags:
  - codex-cli
  - infrastructure
  - latency
  - llm-inference
  - performance
  - azure
  - cost-optimisation
  - agentic-engineering
rating: 4.3
depth: 4
originality: 5
accuracy: 4
substack: 4
copy_edited: true
---

A recent controlled experiment compared Codex CLI performance on two Azure instance types: a standard AMD VM and a Microsoft Cobalt 200 ARM chip. The Cobalt 200 is genuinely faster at running shell commands — by a factor of 1.61 to 2.87 depending on the workload. The build time for a Codex agent session was unchanged.

The reason is straightforward once you see it. Shell command execution was 2.5% of total agent build time. The other 97.5% was waiting for the language model to respond.

This article works through what that finding means for infrastructure decisions, prompt design, and the longer-term question of where the actual performance ceiling for agentic coding lives.

---

## What the Experiment Measured

The Coderlog ran a side-by-side Codex CLI build on two Azure instances:[^1]

- **AMD instance** (standard x86-64, common cloud baseline)
- **Microsoft Cobalt 200** (ARM-based, Azure's second-generation in-house Arm chip — successor to Cobalt 100 — reaching general availability in November 2026)

The shell-command benchmarks were clear. `make build` on Cobalt ran in 1.9 seconds against 4.9 seconds on AMD. File I/O throughput was 2.1× higher. Compiler invocations were 1.61× faster. These are real, measurable improvements.

The agent build — a Codex CLI session completing a multi-file refactoring task — took 4 minutes 12 seconds on AMD and 4 minutes 09 seconds on Cobalt. Within measurement noise.

The breakdown of where time went:

| Activity | Share of total session time |
|---|---|
| LLM inference (waiting for model response) | ~97.5% |
| Shell command execution | ~2.5% |
| File read / write I/O | < 0.5% |
| Other (parsing, formatting, MCP calls) | < 0.5% |

A 2× improvement in a 2.5% slice moves the total by roughly 1.25%. That is undetectable in wall-clock terms.

---

## Why This Is Easy to Get Wrong

The mental model most engineers carry when they think about "making a build faster" is rooted in traditional CI: faster machine → faster compilation → faster feedback loop. That model is correct for compiler-bound work.

Agentic Codex sessions are not compiler-bound. They are inference-bound.

Each agent turn follows roughly this sequence:

1. Collect context (tool results, file state, conversation history)
2. Send a prompt to the model
3. Wait for the model to return a response
4. Parse the response and execute any tool calls
5. Repeat

Steps 1, 4, and 5 happen locally. Step 3 — waiting — is what consumes nearly all the time. The model is running on OpenAI's infrastructure, not on the instance running the Codex CLI process. Upgrading the local CPU has no effect on inference latency.

This is not unique to Codex. Any tool that wraps an API-hosted model (Claude Code, Cursor, Copilot Workspace, Antigravity) has the same profile. The local machine is a thin client. It manages context, executes tool calls, and renders output. The heavy lifting is remote.

---

## What Does Move the Needle

If inference latency is 97.5% of session time, then the interventions that actually matter are those that reduce round-trips or reduce the cost of each round-trip.

**Reducing round-trips:**

- Write tighter, more precise prompts. A well-scoped task with explicit acceptance criteria gives the model enough information to complete a step in one pass rather than asking a clarifying question. Each clarification is a full round-trip.
- Use `AGENTS.md` to pre-answer the questions the model would otherwise ask. File structure conventions, naming rules, test-run commands — if the model can read these instead of discovering them through tool calls, it completes tasks with fewer turns.
- Chunk tasks at the right granularity. Very large tasks cause the model to hedge and ask mid-task questions. Very small tasks cause excessive session overhead. Empirically, tasks that complete in 8–15 turns tend to be the most efficient.

**Reducing per-round-trip cost:**

- Context length matters for inference latency as well as token cost. Long conversations accrue context. Using `/compact` at the right moment, or structuring multi-step workflows to start fresh sessions with summarised context, reduces the tokens the model processes on each call.
- Model selection is the highest-leverage infrastructure decision available. GPT-6.2 Sol (the current Codex CLI default as of v0.172.0, promoted to default in v0.171.0) is significantly faster at inference than its predecessor for comparable task quality. Choosing the right model for the task type — not always the most capable model — is a meaningful lever.
- Regional endpoint selection can affect inference latency. For European workloads, `api.openai.com` routes vary by load. Some teams report 10–20% session time improvement by explicitly routing to a region-pinned endpoint when available.

**The one local lever that does matter:**

Memory bandwidth, not CPU speed, is the constraint for local shell operations in Codex workflows. The Cobalt 200's performance advantage was most pronounced in memory-bandwidth-bound scenarios. For teams running genuinely large file I/O (processing large codebases, generating many small files), ARM-based instances may provide a measurable benefit on the 2.5% slice. Worth measuring, not worth optimising for first.

---

## The Inference-on-Chip Question

The finding points toward a future change that would shift the equation entirely: running inference locally.

Models small enough to run on-device at acceptable quality are improving rapidly. If a model capable of Codex-quality task completion could run on the same machine as the CLI process, inference latency would collapse from network-round-trip timescales (200–800 ms per call in typical conditions) to memory-bandwidth timescales (10–50 ms). The 97.5% slice would shrink, and local CPU/memory improvements would become meaningful again.

This is not available today for the class of tasks Codex handles. GPT-6.2 Sol is not a model that runs on a laptop. But the direction of the field — model distillation, speculative decoding, Apple Silicon inference, Qualcomm NPU acceleration — is toward closing that gap. The Cobalt 200 benchmark is a preview of what becomes important once it does.

For now, the practical implication is the opposite: cloud inference is the constraint, and the local machine is largely irrelevant to performance.

---

## Practical Infrastructure Guidance

For teams choosing instance types for Codex agent workloads:

**Choose based on cost, not performance.** Since inference latency dominates, the CPU speed of the local instance does not affect session time. Use the cheapest instance that provides reliable network connectivity and enough memory to hold the working context in RAM (typically 8–16 GB for mid-sized codebases).

**ARM instances may save money.** Cobalt 200 and Graviton 4 (AWS) instances are cost-competitive with equivalent x86 instances and consume less power. If the shell-execution work is memory-bandwidth-bound (large file trees, many small writes), ARM may also provide modest speedups on the 2.5% slice.

**Do not provision GPU instances for Codex CLI.** The GPU is not in the execution path for Codex CLI sessions (the model runs remotely). GPU instance costs are substantial and provide zero benefit for this workload.

**Measure your own session profile before optimising.** The 2.5% figure comes from one experiment on one task type. Teams running unusually compute-heavy test suites (integration tests, compilation of large projects) may have a higher local-execution share. Run a quick timing breakdown before concluding the finding applies directly to your workload.

---

## Summary

A faster server does not make Codex faster because Codex is not bottlenecked by the server. Shell command execution is roughly 2.5% of agent session time. LLM inference — waiting for the remote model to respond — is the other 97.5%.

The interventions that move the needle are those that reduce round-trips (better prompts, more informative `AGENTS.md`, right-sized task chunks) and reduce per-call latency (context management, model selection, endpoint routing).

Infrastructure spending on faster local CPUs for Codex workloads is largely wasted. The constraint is remote inference, and the lever for that is prompt engineering and model selection, not hardware procurement.

---

[^1]: The Coderlog, "Claude Code vs Codex on Microsoft's New Chip," YouTube, 2026-11-26. [youtube.com/watch?v=g5YEYTG3hLs](https://youtube.com/watch?v=g5YEYTG3hLs)
