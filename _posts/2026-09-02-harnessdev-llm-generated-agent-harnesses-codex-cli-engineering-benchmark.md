---
title: "HarnessDev: What Happens When an LLM Builds Its Own Agent Harness — and Why Codex CLI's Engineering Still Wins"
date: 2026-09-02T18:00:00+00:00
last_modified_at: 2026-09-04T00:13:32+01:00
tags: ["codex-cli", "harness-engineering", "benchmarks", "research", "architecture", "evaluation", "terminal-bench", "swe-bench"]
---

# HarnessDev: What Happens When an LLM Builds Its Own Agent Harness — and Why Codex CLI's Engineering Still Wins


A recurring assumption in the coding-agent field is that model quality is the dominant variable in agent performance. A September 2026 paper from Wu, Zhang, Shi, Lei et al. challenges this directly.[^1] HarnessDev is a benchmark that asks a fundamentally different question: not *can a model write code*, but *can a model engineer the execution infrastructure — the harness — that makes itself perform well*? The findings are instructive for anyone configuring Codex CLI: the 14-point performance gap between a weak harness and a strong one is not a model problem. It is an engineering problem.

## What HarnessDev Measures

The harness is everything that runs around the model: the execution loop, tool orchestration layer, context budget management, persistent state checkpointing, lifecycle recovery logic, and evaluation instrumentation. HarnessDev separates *creator* (the LLM asked to build the harness) from *executor* (the LLM that actually runs tasks inside the built harness), then measures both.[^1]

Each creator receives a minimal, policy-free seed containing only:

- Input/output contracts
- Passive tool primitives
- Audit logging hooks

No execution loop, no context management, no verification logic. From this seed, the creator must implement six functional modules:

```
execution control       → turn loop, step limits, abort conditions
tool orchestration      → dispatch, retry, parallel fan-out
context management      → budget tracking, summarisation triggers
persistent state        → cross-turn memory, checkpoint writes
lifecycle recovery      → crash restart, fork resumption
evaluation logic        → output verification, self-scoring
```

The benchmark ran six creator LLMs — Opus 4.8, GPT-5.5, Gemini 3.1 Pro, DeepSeek V4 Pro, Qwen 3.7 Max, and Seed 2.0 Pro — across four domains and 2,207 downstream task instances drawn from SWE-bench Pro (731 tasks), Terminal-Bench 2.1 (89 tasks), MLE-bench (75 tasks), EQ-Bench3 (46 tasks), and BrowseComp (1,266 tasks).[^1]

## Creation Results: Where the Gap Opens

Generated harnesses matched or exceeded human-engineered references on writing tasks and machine-learning experimentation. The gap opens sharply on code and search:

| Domain | Generated (best) | Human reference |
|---|---|---|
| SWE-bench Pro (code) | 69.3% | 80.0% |
| BrowseComp (search) | 52.4% | 92.2% |
| EQ-Bench3 (writing) | ~ref | ~ref |
| MLE-bench (ML) | ≥ref | ref |

The code gap — 10.7 percentage points — is structural. Generated harnesses consistently omitted persistent checkpointing (absent in 26,679 recorded trajectories), hardcoded executor-specific token limits that broke under different runtimes, and built verification logic that passed internal tests while failing on the held-out distribution.[^1]

Median generated code harness size was 698 lines of added code per artifact, ranging from 270 to 1,672 LOC. Edit volume had no predictive value: Gemini's minimalist changes (1,006 total LOC across three artifacts) produced the best Terminal-Bench score at 68.8%.[^1]

## Evolution: Feedback Gains That Do Not Transfer

HarnessDev also tests whether models can *improve* their own harness using downstream execution feedback — an appealing capability for self-tuning agents. The results are cautionary:

```mermaid
graph LR
    A[Feedback set gain\n+8.8 points] -->|held-out transfer| B[Held-out gain\n+2.7 points]
    style A fill:#4CAF50,color:#fff
    style B fill:#FF9800,color:#fff
```

Only Opus 4.8 improved reliably on held-out tasks under a fixed Gemini 3.1 Pro runtime. All other creators regressed or showed no gain when the executor changed.[^1] Dead code was common: mechanisms inserted to handle feedback-set failure modes simply never fired on different task distributions. Test count alone correlated weakly with success (r = 0.13–0.26), meaning the model's self-assessment of harness quality was unreliable.[^1]

## Cross-Executor Transfer Failure

The most operationally significant finding is that generated harnesses are not portable. A harness optimised for one executor model can actively harm performance under another:

- Qwen 3.7 Max harnesses gained **+17.6 percentage points** on BrowseComp when executed by Gemini 3.1 Pro rather than Qwen itself.
- Opus 4.8 harnesses *lost* **36.3 percentage points** on SWE-Pro under the same Gemini substitution.[^1]

The culprit is executor-specific configuration baked into generated harnesses: step limits, deduplication thresholds, and context truncation rules tuned to one model's verbosity profile break silently under a different executor.

## The Codex CLI Premium

HarnessDev references a measurement from a parallel harness-effects study:[^2] with identical GPT-5 weights, Terminal-Bench 2.1 resolve rates differ by **14.4 percentage points** depending on which harness surrounds the model:

| Harness | Terminal-Bench 2.1 |
|---|---|
| Terminus 2 | 35.2% |
| Codex CLI | 49.6% |

That gap is not attributable to prompt differences alone. Codex CLI contributes a purpose-built execution loop (Tokio async state machine), three-platform native sandboxing, a four-layer permission stack, per-model server-delivered prompt configuration, and an agent-maintained memory pipeline.[^3] These are precisely the six functional modules HarnessDev found models unable to reliably self-generate.

## What This Means for Codex CLI Configuration

The practical lesson is not "do not use AI-generated configuration". It is "do not *rely* on auto-generated harness logic for the modules that matter most". The six HarnessDev modules map directly to Codex CLI configuration surfaces:

### Execution Control — Step Limits and Turn Budgets

```toml
# config.toml
[agent]
max_turns = 40          # explicit ceiling; do not leave at default-∞ for long tasks
timeout_ms  = 300000    # per-turn wall-clock limit
```

Generated harnesses commonly omitted explicit ceilings. Long-horizon tasks without them burn the context budget on recovery loops rather than forward progress.

### Context Management — Memory Search Scope

```toml
[memory]
memory_search_upward = false   # prevents cross-project context bleed (v0.150.0+)
max_memory_items     = 60      # hard cap; prevents retrieval noise at session scale
```

HarnessDev found that generated context managers either over-retained (blowing budgets) or under-retained (losing working state). Both failure modes are addressed by explicit configuration rather than letting the model decide at runtime.

### Lifecycle Recovery — Hook Registration

```toml
[hooks]
on_interrupt = ["./hooks/checkpoint-state.sh"]   # Interrupt hook (v0.150.0+)
[[hooks.PreToolUse]]
command = "./hooks/verify-scope.sh"
```

The Interrupt hook fires before a turn abort, giving recovery logic a window to flush state.[^4] Generated harnesses in HarnessDev frequently wrote checkpoints *after* the abort, making them unreachable.

### Verification — PostToolUse Hooks

```toml
[[hooks.PostToolUse]]
command  = "./hooks/assert-output.sh"
timeout_ms = 5000
```

The PostToolUse hook is the correct place to enforce output contracts. Generated harnesses in HarnessDev embedded verification inside the execution loop, where it became entangled with retry logic and produced conflicting signals.

### Tool Orchestration — MCP Output Limits

```toml
[mcpServers.my-search-server.tools.web_search]
output_token_limit = 4096    # v0.152.0+; prevents context flooding from verbose tools
```

Per-tool output limits (introduced in v0.152.0) address the most common generated-harness failure in search tasks: uncapped tool output that consumed the entire context window before the model could act on results.[^5]

## The Limits of Self-Evolving Infrastructure

HarnessDev's evolution results are a measured counterargument to the idea that harnesses can be continuously self-improved by the model running them. The gains are real on the feedback distribution and fragile off it — exactly the failure mode that matters most in production, where the task distribution is unknown in advance.

This does not mean generated configuration has no place in Codex CLI workflows. Skill files (SKILLS.md) are a proven surface for LLM-generated domain knowledge, and AGENTS.md behavioural guidance benefits from model-assisted drafting. What HarnessDev demonstrates is that the *execution infrastructure* — loops, budgets, recovery, verification — benefits from deliberate engineering and explicit configuration rather than inference-time generation.

The 14.4-point premium Codex CLI holds over Terminus 2 on Terminal-Bench 2.1 was not delivered by a larger model. It was delivered by decisions made in code, configuration, and architecture that no amount of self-prompting reliably replicates.

## Citations

[^1]: Wu Y, Zhang J, Shi J, Lei X, et al. (2026). *HarnessDev: Can LLMs Create and Evolve Their Own Agent Harness?* arXiv:2609.01437. <https://arxiv.org/abs/2609.01437>

[^2]: Harness-Bench: Measuring Harness Effects across Models in Realistic Agent Workflows (2026). arXiv:2605.27922. <https://arxiv.org/abs/2605.27922>

[^3]: Barbaste M, Darrigol F, Vu T, Wiltberger J (2026). *Harness Engineering: Anatomy, Architecture, and Evolution of Coding Agents — A Source-Code Study of Eleven Systems.* arXiv:2609.00006. <https://arxiv.org/abs/2609.00006>

[^4]: OpenAI (2026). *Codex CLI v0.150.0 Release Notes — Interrupt Hook.* GitHub. <https://github.com/openai/codex/releases/tag/rust-v0.150.0>

[^5]: OpenAI (2026). *Codex CLI v0.152.0 Release Notes — MCP Tool Output Limits.* GitHub. <https://github.com/openai/codex/releases/tag/rust-v0.152.0>
