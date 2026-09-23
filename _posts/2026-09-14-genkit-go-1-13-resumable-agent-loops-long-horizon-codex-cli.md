---
title: "Resumable Agent Loops: What Genkit Go 1.13 Teaches Codex CLI Teams About Long-Horizon Task Design"
parent: "Articles"
nav_order: 1157
date: 2026-09-14T08:00:00+00:00
last_modified_at: 2026-09-23T18:09:02+01:00
tags: ["codex-cli", "multi-agent", "resumability", "long-running-agents", "genkit", "external-message", "hitl", "orchestration", "agentic"]
---

# Resumable Agent Loops: What Genkit Go 1.13 Teaches Codex CLI Teams About Long-Horizon Task Design


---

Two convergent releases from different parts of the AI tooling ecosystem arrived at the same conclusion in September 2026. Genkit Go 1.13 shipped on 3 September 2026 with resumable `generate` and agent loop primitives — the ability to pause mid-generation, persist state, and resume from exactly where execution stopped.[^1] On 10 September 2026, Codex CLI v0.154.0 introduced `ExternalMessage`, a structured HITL primitive that allows an orchestrator to inject human input or external data into a running agent turn without restarting the session.[^2] Neither team was building the same product. Both teams reached the same conclusion: resumability is not a convenience feature — it is a required primitive for any system that runs agents on tasks that exceed a single context window or a single human-attention span.

For Codex CLI teams building long-horizon workflows, the Genkit Go 1.13 design is worth studying not because you will use Genkit directly, but because it makes explicit the design decisions that `ExternalMessage` leaves implicit.

## Why Resumability Matters More Than Context Extension

The conventional response to long-horizon agentic tasks is to extend the context window. Models have grown from 8K to 128K to 1M tokens over a few years, and the temptation is to treat context extension as the solution to session continuity. It is not, for three reasons.

First, token cost scales with context length. A million-token context window does not cost the same as a 128K window, and the cost differential compounds across every tool call an agent makes within a session. For sustained overnight runs or multi-day workflows, the economics of unbounded context extension are unfavourable compared to structured checkpointing.

Second, compaction is lossy. Codex CLI's automatic context compaction — and the equivalent mechanisms in all frontier model APIs — is not lossless compression. It is a summarisation under uncertainty, and the accuracy of that summarisation degrades as the original context grows longer and more heterogeneous. An agent that has been compacted twice over a four-hour run may have a materially different understanding of its acceptance criteria than the agent that started the run.[^3]

Third, not all pauses are planned. Long-horizon tasks encounter interruptions: rate limits, network failures, human review checkpoints, infrastructure restarts. A system designed only for continuous execution fails at every interruption. A resumable system treats interruptions as first-class events and handles them without data loss.

Genkit Go 1.13's resumable `generate` primitive addresses all three problems by making persistence explicit. Instead of holding the entire session in memory and hoping nothing fails, a resumable agent serialises its state at defined checkpoints and loads from that serialised state on resume.[^1] The unit of work is not the session — it is the checkpoint interval.

## The Genkit Design: Serialisable State at Named Checkpoints

Genkit Go 1.13 exposes resumability through two mechanisms.[^1]

The first is resumable `generate`. A call to `generate` can now be annotated with a checkpoint name. If the call is interrupted — by a timeout, a resource constraint, or a planned pause — the partial state is written to a configurable backend (filesystem, Cloud Firestore, or a custom store). On the next invocation, the caller passes the checkpoint reference and execution resumes from the last serialised state rather than from the beginning.

The second is resumable agent loops. Agent loops in Genkit are sequences of tool calls and model turns. In the pre-1.13 design, a loop that failed at turn 17 of a 40-turn sequence would require restarting from turn 1. In 1.13, each completed turn is checkpointed, and a failed or interrupted loop can be resumed from the last committed turn. The loop's observable behaviour — from the perspective of an external caller or a human reviewer — is identical to a continuous run; the difference is entirely in the recovery path.

Both mechanisms share a design invariant that matters for correctness: the state serialised at a checkpoint is the complete input to the next turn, not a summary of prior turns. This means the model receiving a resumed prompt sees exactly what it would have seen in an uninterrupted run — the checkpoint does not introduce a lossy summarisation step.

## ExternalMessage: Codex CLI's Equivalent Primitive

Codex CLI's `ExternalMessage` (introduced in v0.154.0) is the injection side of the same design.[^2] Where Genkit's resumable loops provide the persistence mechanism, `ExternalMessage` provides the mechanism for injecting structured content into an already-running agent turn.

The canonical use case is human-in-the-loop review. An agent running a long task reaches a point requiring human judgement — a security decision, a deployment gate, a cost threshold. Rather than polling a file or restarting the session with a modified prompt, the orchestrating process calls the Codex CLI `run()` or `turn()` API with an `ExternalMessage` payload. The running agent receives the message at the injection point, incorporates it into its current turn context, and continues.

The two primitives are complementary rather than overlapping. `ExternalMessage` handles live injection into a running session. Genkit-style resumable loops handle recovery from interruption. A production long-horizon workflow might use both: `ExternalMessage` for mid-task human review, and a checkpoint-based persistence layer for recovery from unexpected failures.

Codex CLI does not yet ship a native checkpoint persistence layer equivalent to Genkit Go 1.13's backend. The gap is bridgeable with current tools — a PostToolUse hook that commits verified state to git after each passing harness run provides coarse-grained resumability, as documented in the Checkly overnight rewrite case study.[^4] But the Genkit design suggests the direction the ecosystem is moving: finer-grained, serialisable, backend-agnostic checkpoints that do not require the agent to have write access to a repository.

## Ecosystem Convergence on Resumability as a Primitive

The same September 2026 period that saw Genkit Go 1.13 and Codex CLI v0.154.0 ship also produced a practitioner post describing an agent team pattern for tasks that span multiple context windows.[^5] The pattern uses explicit handoff state — structured data passed between agent turns — to preserve task context across session boundaries. The handoff state is not a summary; it is the minimum representation required for the receiving agent to continue where the previous agent stopped.

These three signals — Genkit's resumable loops, Codex CLI's `ExternalMessage`, and the agent team handoff pattern — are converging on the same design insight: reliable long-horizon agentic work requires explicit state management at well-defined boundaries, not passive reliance on context windows and model memory.

The practical implication for Codex CLI teams is that "long-running" and "reliable" are increasingly achievable together, but only if the workflow is designed with persistence in mind from the start. The three dimensions of that design are:

**Checkpoint granularity.** How much work can be lost on failure without requiring a full restart? PostToolUse commit hooks provide session-level granularity. A Genkit-style backend provides turn-level granularity. The right choice depends on the task's rework cost per turn.

**Injection interface.** When does human input need to enter a running session? If the answer is "never," no injection mechanism is required. If the answer is "at specific decision points," `ExternalMessage` is the current Codex CLI primitive. If the answer is "whenever a threshold is crossed," a polling orchestrator that watches for threshold events and calls `run()` with `ExternalMessage` payloads is the pattern.

**State representation.** What does the receiving agent need to continue from a checkpoint? A git commit hash pointing to the last verified state is sufficient for code-generation tasks with a test harness. For multi-agent workflows involving non-code artefacts, a more explicit state schema — aligned with the handoff state pattern from the agent team literature — may be necessary.

## What This Means for Codex CLI Configuration

The AGENTS.md implications of a resumability-aware workflow are modest but specific.

The harness specification in AGENTS.md should describe what constitutes a valid checkpoint state, not just what constitutes a passing test. This is a small but important distinction: a test suite that passes is evidence of a valid state, but the checkpoint representation — the git ref, the output artefact, the structured handoff JSON — needs to be defined separately so the agent knows what to preserve.

The `writable_roots` configuration should include the checkpoint store if it lives within the repository. If it lives outside the repository (a Firestore instance, a file share), the `network` configuration needs to permit access to it. In either case, the agent should not have write access to the checkpoint store from which it is reading — this prevents a class of correctness failure where the agent modifies its own resumption context.

The PostToolUse hook that most Codex CLI teams use for test validation can double as a checkpoint trigger. A hook that runs after each successful harness pass and commits the current state provides the turn-level checkpointing that Genkit ships as a first-class feature. The commit message should include enough information for a resume operation to reconstruct the agent's intended next action — what the hook verified, what remains, and what the acceptance gate is.

## Summary

Genkit Go 1.13's resumable `generate` and agent loop primitives articulate, in explicit API form, the design pattern that the broader agentic tooling ecosystem is converging on: long-horizon tasks require first-class resumability, not just larger context windows.[^1] Codex CLI v0.154.0's `ExternalMessage` primitive provides the injection side of the same design — the ability to introduce structured human input into a running session without a restart.[^2] For Codex CLI teams, the actionable synthesis is: design checkpoints as part of the task specification, not as an afterthought; use PostToolUse commit hooks to materialise those checkpoints in git; define the state representation that makes a checkpoint self-sufficient; and treat `ExternalMessage` as the planned-interruption mechanism for human review points that checkpointing alone cannot handle. The tools are available now. The design pattern is no longer hypothetical.

---

## Citations

[^1]: Gill, C. (2026, September 3). *Genkit Go 1.13: Resumable generate and agent loops, async subagents, and A2UI.* <https://genkit.dev/blog/genkit-go-1-13/>. Highlighted by Seroter, R. (2026, September 11). *Daily Reading List #865.* <https://seroter.com/2026/09/11/daily-reading-list-september-11-2026-865/>

[^2]: OpenAI. (2026, September). *Codex CLI v0.154.0 Release Notes.* <https://github.com/openai/codex/releases/tag/v0.154.0> — `ExternalMessage` type for `run()` and `turn()` calls; `max` and `ultra` reasoning-effort values.

[^3]: See also: Vaughan, D. (2026, August 31). *The Compaction Cliff: How Context Compaction Silently Erodes Your AGENTS.md Safety Rules.* codex-resources/articles/2026-08-31-compaction-cliff-context-compaction-safety-rules-agents-md-knowledge-triage.md

[^4]: Janusevicius, E. (2026, September). *We Let AI Agents Rewrite a 92M-Message-a-Day Service in Go. Zero Incidents.* Checkly Engineering Blog. <https://www.checklyhq.com/blog/agentic-rewrite-nodejs-to-go/>

[^5]: Seroter, R. (2026, September 9). *Daily Reading List #863.* <https://seroter.com/2026/09/09/daily-reading-list-september-9-2026-863/> — agent teams for long-running tasks spanning multiple context windows.
