---
title: "The Overnight Agent: What Checkly's Zero-Incident Rewrite Teaches Codex CLI Teams About Long-Running Tasks"
parent: "Articles"
nav_order: 1149
date: 2026-09-10T08:00:00+00:00
last_modified_at: 2026-09-10T11:46:37+01:00
tags: ["codex-cli", "long-running-agents", "test-harness", "agentic-rewrite", "production", "code-generation"]
---

# The Overnight Agent: What Checkly's Zero-Incident Rewrite Teaches Codex CLI Teams About Long-Running Tasks


When Checkly decided to rewrite a Node.js service handling 92 million messages a day into Go, they handed the task to a single Claude Code agent running overnight.[^1] By morning, the agent had produced 13,000 lines of deployable application code. The rollout proceeded in phases — free accounts first, then paid, then enterprise — and ended without a single production incident.

This is not a story about multi-agent orchestration or sophisticated handoff protocols. It is a story about one well-constrained agent doing a week's worth of engineering work while its owners slept. For Codex CLI teams thinking about how to structure long-horizon tasks, the methodology behind that overnight run is more instructive than the headline number.

## The Context Problem in Long-Running Agentic Work

The central challenge of any long-running agentic task is not intelligence — it is feedback. A human engineer rewriting a service accumulates a mental model of what the new code must do, tests it incrementally, and self-corrects based on a continuous stream of signals. An agent working inside a context window has to do the same, but context windows are finite, compaction is lossy, and an agent that drifts from the acceptance criteria halfway through a 13,000-line task has no reliable way to recover.

The conventional response to this problem is multi-session orchestration: break the task into segments, hand state between sessions, use external memory or structured checkpoints. That works, but it introduces coordination overhead and state-transfer risk. Checkly took a different approach: they made the feedback signal so cheap and deterministic that the agent never needed human intervention to know whether it was on track.

## Test Harness as Context Anchor

The key decision Checkly made before writing a single line of Go was to build a complete test harness in isolation from the target language.[^1] The harness operated as a black box: it sent inputs through the same queue topology the production service would use, captured outputs, and compared them against golden files that encoded the expected behaviour of the legacy Node.js implementation down to the byte level.

This design had a consequence that mattered enormously for agent operation. Because the harness was language-agnostic and deterministic, it could give the agent a binary pass/fail signal on every iteration without any human in the loop. The agent did not need to infer whether its Go implementation was correct by reasoning about the codebase — it could run the harness, read the result, and continue. That deterministic loop replaced human code review as the primary feedback mechanism.

Three principles governed the harness design and are directly portable to Codex CLI sessions:

**Boundary exhaustiveness.** The harness ran real PostgreSQL, a custom SQS emulator, and Toxiproxy for network fault injection — not mocks of those systems. Checkly discovered later that their initial harness modelled three queues where production used eighteen per region, and this gap allowed the agent to produce incorrect retry logic.[^1] The lesson: every infrastructure assumption that is left implicit becomes a potential source of silent deviation.

**Zero coupling to implementation.** The harness tested the service's external contract, not its internal structure. This meant the agent could refactor freely without invalidating the feedback signal. A harness that reaches into implementation details rewards preservation of structure rather than correctness of behaviour.

**CI enforcement on both sides.** Once the Go service passed the harness, Checkly added the harness to CI with gates against both the legacy Node.js service and the new Go daemon simultaneously. Any pull request that broke either would be blocked. This preserved the harness as a living contract rather than a one-time acceptance test.

## The Trust Problem for Long-Running Agents

A related challenge surfaces when long-running agents need access to production infrastructure during development. Checkly's overnight run required access to queue infrastructure, a database, and real service boundaries. The agent could not operate purely in a sandbox if the golden files were going to reflect production behaviour.

A concurrent case study from the same week describes the trust model some teams are adopting for exactly this situation.[^2] The pattern uses a CLI with a secret vault so agents can retrieve credentials by name at runtime without the values ever appearing in the agent's context window. Containers provide isolation between concurrent agents, though the author is explicit that container boundaries are not the security mechanism — the frontier model's improved ability to distinguish legitimate instructions from injected ones is. The model training improvements in Claude Code are cited as the practical basis for extending production access.

For Codex CLI teams, the practical synthesis of these two approaches is:

1. Credentials live in the vault and are retrieved by reference at execution time — they never appear in AGENTS.md, context, or logs.
2. The agent's writable surface is constrained to the repository under test and explicitly enumerated infrastructure endpoints.
3. The test harness, not human review, is the acceptance gate — which means the agent can iterate without pausing for approval, and the human's review occurs once at the boundary between passing and deployment.

## Mapping to Codex CLI Configuration

The Checkly methodology translates directly to a set of Codex CLI configuration choices for overnight or long-horizon tasks.

**AGENTS.md as harness specification.** The most important document in a long-running Codex CLI session is not the task description — it is the specification of the acceptance gate. AGENTS.md should describe exactly how to run the test harness, what a passing output looks like, and what to do when the harness fails. An agent that knows its acceptance criteria can iterate without requesting guidance.

```markdown
# Acceptance Gate
Run `make harness` from the repo root. A passing run prints `ALL GOLDEN FILES MATCHED`
to stdout and exits 0. On failure, the diff between actual and expected output is written
to `harness/failures/`. Do not mark a task complete until `make harness` exits 0.
```

**`writable_roots` scoped to the work surface.** For a rewrite task, the agent needs write access to the target directory and nothing else. Locking the writable surface prevents the agent from modifying the harness itself — which would allow it to pass by changing the acceptance criteria rather than the implementation.

```toml
[sandbox]
writable_roots = ["src/go-daemon/", "harness/failures/"]
network = "off"
```

**`output_token_limit` per tool call, not per session.** For a 13,000-line generation task, session token limits will be reached and compaction will occur. Setting a per-tool limit on code-writing calls keeps individual outputs within the range where the model produces coherent Go rather than truncated fragments.

**Checkpoint commits.** Checkly's agent ran until morning without explicit checkpointing, but for Codex CLI sessions where compaction risk is higher, a PostToolUse hook that commits passing harness states to git provides recovery points:

```bash
# .codex/hooks/post-tool-use.sh
if make harness > /dev/null 2>&1; then
  git add src/go-daemon/
  git commit -m "checkpoint: harness passing at $(date -u +%Y%m%dT%H%M%SZ)"
fi
```

This ensures that even if compaction degrades the agent's model of what it has done, the work product is preserved in git at every verified state.

## What the Zero-Incident Rollout Reveals

Checkly's phased rollout — free accounts, paid accounts, enterprise — is notable not because it is novel but because it was possible. The feature flag routing (`GO_DAEMON=true`) and parallel operation of both services during transition are standard deployment practice. What made the practice applicable here was confidence in the output: the harness had validated byte-level parity against the legacy service across the full queue topology, so the rollout was a traffic migration, not a gamble.

This is the underappreciated second-order benefit of test-harness-first methodology for agentic work. The harness does not just control the agent — it produces evidence that satisfies the deployment gate. The artefact that guided the overnight run is the same artefact that justified moving 92 million daily messages to a new implementation without a staged experimental period.

For Codex CLI teams, the implication is that investment in acceptance infrastructure pays twice: once as agent guidance, once as deployment evidence. A harness that can be run in CI against both old and new implementations is the minimum viable structure for any agentic rewrite task with production consequences.

---

## Summary

Checkly's zero-incident Node.js to Go rewrite (September 2026) demonstrates that long-running agentic tasks are solvable without complex multi-session orchestration, given a sufficiently deterministic feedback loop.[^1] The test-harness-first approach — language-agnostic, boundary-exhaustive, CI-enforced on both the legacy and new implementations — replaced human review as the primary acceptance gate and allowed a single Claude Code agent to produce 13,000 lines of deployable code overnight. For Codex CLI teams, the methodology maps directly to AGENTS.md harness specification, scoped `writable_roots`, PostToolUse checkpoint commits, and secret-vault credential patterns for production-adjacent agent sessions.[^2] The resource gains (70% pod reduction, 60% fewer database sessions, 15% lower database CPU) followed from correctness confidence, not from the agent's sophistication.

---

## Citations

[^1]: Janusevicius, E. (2026, September). *We Let AI Agents Rewrite a 92M-Message-a-Day Service in Go. Zero Incidents.* Checkly Engineering Blog. <https://www.checklyhq.com/blog/agentic-rewrite-nodejs-to-go/>

[^2]: Gold, J. (2026, September). *I trust my coding agents with production secrets now.* <https://jacob.gold/posts/i-trust-my-coding-agents-with-production-secrets-now>

[^3]: Codex CLI AGENTS.md Reference — OpenAI Codex Documentation. <https://github.com/openai/codex/blob/main/codex-rs/docs/agents-md.md>

[^4]: Codex CLI Sandbox Configuration: `writable_roots`, `network`, and `shell`. <https://github.com/openai/codex/blob/main/codex-rs/docs/configuration.md>

[^5]: OpenAI Codex CLI Hooks Reference: PreToolUse and PostToolUse. <https://github.com/openai/codex/blob/main/codex-rs/docs/hooks.md>
